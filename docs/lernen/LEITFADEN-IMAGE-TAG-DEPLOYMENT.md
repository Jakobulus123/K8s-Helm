# Leitfaden: Von Code zu laufender App — das ganze Thema

> Ein pädagogischer Durchlauf vom Quellcode bis zur UI im Browser.
> Ergänzt [`IMAGE-UND-DEPLOYMENT.md`](IMAGE-UND-DEPLOYMENT.md) (Kurz-Überblick)
> und [`DEBUG-IMAGE-TAG-PROBLEM.md`](DEBUG-IMAGE-TAG-PROBLEM.md) (konkreter Bug).
> Ziel hier: die Kette komplett verstehen, nicht nur den Einzelfall.

---

## Das große Bild

Stell dir die Reise deines Codes vor:

```
[dein Editor] ──git push──> [GitHub Repo (Quellcode)]
                                 │
                                 ▼  GitHub Actions: docker build + push
                          [Registry (GHCR) — Images liegen hier]
                                 │
                     ╔═══════════╧═══════════╗
                     ║                       ║
           Tag-Bump im Chart          Cluster-Node pullt Image
           ins K8s-Helm Repo           wenn Pod startet
                     │                       ▲
                     ▼                       │
           [GitHub (Manifeste)] ──ArgoCD──> [Kubernetes Cluster]
                                                │
                                                ├── Pod + Deployment + Service
                                                ├── Traefik IngressRoute
                                                ├── cert-manager / Let's Encrypt
                                                └── DNS (OVH) ──> extern
```

Drei verschiedene Git-Repos können im Spiel sein:

1. **App-Code-Repo** — dein Quellcode (z.B. `Jakobulus123/miniapp`)
2. **Chart-Repo** — Kubernetes-Manifeste & Helm-Charts (hier: `K8s-Helm`)
3. **Registry** — kein Git, aber auch „Content mit Adressen": GHCR

ArgoCD beobachtet nur das Chart-Repo. Dass das App-Repo auch existiert, ist
ArgoCD völlig egal. Das ist später wichtig fürs Verständnis.

---

## Teil A — Das Image

### Was ist ein Image eigentlich?

Ein Container-Image ist ein **versiegelter Paketschnack**: Code + Laufzeit +
Libs + System-Files, alles in einem reproduzierbaren Archiv. Aus einem Image
werden zur Laufzeit ein oder mehrere **Container** instanziiert.

Analogie: Image = Klassenplan, Container = Instanz der Klasse.

### Wie sieht ein Image intern aus?

Ein Image besteht aus drei Teilen:

1. **Layer**: Stapel aus Tar-Archiven (je Layer eine Dockerfile-Instruktion).
   Jeder Layer hat seinen eigenen SHA256. Layer können zwischen Images
   geteilt werden — deshalb ziehen Node-Pulls oft nur Differenzen.
2. **Config**: JSON mit Metadaten — Entrypoint, Env-Vars, User,
   exposed Ports, Labels.
3. **Manifest**: Liste aller Layer + Referenz auf die Config. Der SHA256
   *dieses Manifests* ist der **Image-Digest**.

Das Schöne: Der Digest ist content-addressed. Wenn ich denselben Digest habe,
habe ich byte-identischen Inhalt. Das ist die Basis für alles weitere.

### Woraus wird das Image gebaut?

Aus einem `Dockerfile` im App-Repo. Typisch für Node:

```dockerfile
FROM node:22-alpine       # Basis-Layer
WORKDIR /app              # neuer Layer
COPY package*.json ./     # neuer Layer
RUN npm ci                # neuer Layer (installiert Deps)
COPY . .                  # neuer Layer (dein Code)
CMD ["node", "server.js"] # kein Layer, nur Config
```

`docker build` führt das aus, erzeugt die Layer, speichert sie lokal.
`docker push` lädt sie in die Registry.

### Wer macht das bei uns?

GitHub Actions im App-Repo. Grober Ablauf:

```yaml
on:
  push:
    branches: [main]
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/jakobulus123/miniapp:latest
            ghcr.io/jakobulus123/miniapp:${{ github.sha }}
```

Nach jedem Push auf `main` liegt in GHCR dann:
- `miniapp:latest` (überschrieben)
- `miniapp:<commit-sha>` (neu)

Genau deshalb haben wir in der Registry Tags wie
`6445eabc66585128a1d8ee187bbc01a80b52bd6f` — das sind Commit-SHAs.

---

## Teil B — Registry und Tags

### Was ist eine Registry?

Ein Dienst, der Images hostet. GHCR (`ghcr.io`) ist GitHubs Registry;
Docker Hub, quay.io, ECR, GCR sind andere. Pro **Repository**
(= Namensraum, z.B. `jakobulus123/miniapp`) liegen dort viele Images
mit verschiedenen Digests, und eine **Tag-Tabelle**, die Labels auf Digests
abbildet.

### Tag = Label, Digest = Inhalt

Das ist der wichtigste Satz dieses Dokuments:

> **Ein Tag ist nur ein Zeiger. Der Digest ist die Wahrheit.**

```
ghcr.io/jakobulus123/miniapp
├── @sha256:2f2bbff4…        ← Image-Inhalt A
│   ├── :latest               ← Tag A1
│   ├── :6445eabc…            ← Tag A2 (Commit-SHA)
│   └── :v1.2.3               ← Tag A3 (Semver)
│
└── @sha256:f4540d65…        ← Image-Inhalt B
    └── :c1e93196…            ← Tag B1
```

Wenn jemand `docker push :latest` ausführt:
- neuer Digest wird hochgeladen
- `:latest` zeigt jetzt auf den neuen Digest
- der alte Digest bleibt abrufbar (bis niemand mehr drauf verweist)

Das heißt: **Tags sind veränderbar, Digests nicht.**

### Wann Tag, wann Digest?

| Tag-Typ | Beispiel | Stabil? | Gut für | Weniger gut für |
|---|---|---|---|---|
| `:latest` | `:latest` | nein | lokale Experimente | Prod, GitOps |
| Umgebung | `:stable`, `:prod` | nein | manuelle Promotion | Audit, Rollback |
| Branch | `:main`, `:dev` | nein | Preview-Deploys | deterministische Deploys |
| Datum | `:2026-04-22` | nein¹ | Nightly-Builds | mehrere Builds/Tag |
| Semver | `:v1.2.3` | konventionell ja | Releases, Humans | Hotfix-Varianten |
| Commit-SHA | `:6445eab…` | de-facto ja | CI, GitOps | hübsche URLs |
| Digest | `@sha256:…` | garantiert ja | kritische Prod | Lesbarkeit |

¹ *Alle Tags lassen sich technisch überschreiben; die Stabilität ist
nur Konvention.*

### Der `:latest`-Mythos

`:latest` ist **nicht** der neueste Push — es ist nur das Tag, auf das
Docker/OCI defaultmäßig zeigt, wenn du gar kein Tag angibst (`docker pull
miniapp` = `docker pull miniapp:latest`).

Du kannst in der Registry ein Repository haben, das **nie** `:latest` gesehen
hat — perfekt valide, wenn dein Build-Prozess ausschließlich Semver-Tags
oder SHA-Tags schreibt.

`:latest` ist bequem, aber trügerisch:
- kein Hinweis, wann es zuletzt gepusht wurde
- kein Rollback-Pfad (welcher Digest war „gestern"?)
- zwei Nodes können unterschiedliche gecachte Digests hinter derselben
  `:latest`-Referenz haben

---

## Teil C — Wie Kubernetes das Image bekommt

### Die image-Referenz im Pod

Irgendwo im YAML steht etwas wie:

```yaml
spec:
  containers:
    - name: app
      image: ghcr.io/jakobulus123/miniapp:6445eabc…
      imagePullPolicy: IfNotPresent
```

Zwei Dinge hier:
1. Die **Image-Referenz** — wo holen wir das Image?
2. Die **Pull-Policy** — wie entscheiden wir beim Pod-Start?

### imagePullPolicy im Detail

Die Policy sitzt **auf dem Container**, nicht auf dem Image. Drei Werte:

| Wert | Verhalten |
|---|---|
| `Always` | Beim Pod-Start Registry fragen: „hat sich der Digest für dieses Tag geändert?" Wenn ja, neu pullen. |
| `IfNotPresent` | Nur pullen, falls Node das Image nicht im lokalen Cache hat. |
| `Never` | Nie pullen. Muss schon da sein (für pre-loaded Images). |

Default ist **tag-abhängig**:

- Tag `:latest` oder kein Tag → default `Always`
- Jeder andere Tag → default `IfNotPresent`

Die Logik: K8s geht davon aus, dass `:latest` sich jederzeit bewegt, also
lieber immer prüfen. SHA-Tags gelten per Konvention als immutable — da reicht
der Cache.

### Was `Always` *nicht* macht

`Always` prüft beim **Pod-Start**, nicht im laufenden Pod. Ein 43h alter Pod
kümmert sich nicht um neue Pushes in die Registry. Das war genau die Falle
im Debug-Fall (siehe [`DEBUG-IMAGE-TAG-PROBLEM.md`](DEBUG-IMAGE-TAG-PROBLEM.md)).

Um einen Re-Pull zu erzwingen, braucht man einen Pod-Neustart. Das passiert
bei Kubernetes natürlich, wenn sich das `spec.template` des Deployments
ändert — z.B. neuer Tag.

---

## Teil D — Die Kubernetes-Objekte

Zum Verständnis: Wenn ArgoCD `kubectl apply` macht, entstehen *Objekte* in
etcd. Jedes Objekt ist ein YAML-Dokument mit `kind`. Die wichtigen für
unseren Flow:

### Pod

Kleinste ausführbare Einheit. Enthält 1-n Container (meistens 1). Bekommt
eine IP. Wird neu geboren, wenn es crasht oder ersetzt wird.

### Deployment

Managed Pod-Kopien („Replicas") und Rolling-Updates. Du spezifizierst
„ich will 2 Pods mit dieser Spec", es sorgt dafür. Änderst du die Spec,
erzeugt es neue Pods und tötet alte — ohne Downtime (wenn Readiness-Probes
stimmen).

Im Deployment steht das `spec.template` — das ist die Pod-Vorlage. Jede
Änderung hier triggert ein neues ReplicaSet und einen Rolling-Update.

### Service

Stabiler Netzwerk-Endpunkt im Cluster. Pods kommen und gehen (neue IPs jedes
Mal) — ein Service hat eine konstante virtuelle ClusterIP und leitet an
passende Pods (per Label-Selector).

Bei uns:
```yaml
service:
  type: ClusterIP      # nur cluster-intern erreichbar
  port: 80             # Port des Service
  targetPort: 3000     # Port im Container
```

Ein ClusterIP-Service ist von außen nicht erreichbar — dafür braucht's einen
Ingress oder port-forward.

### Ingress vs. Traefik IngressRoute

Das war in unserem Debug eine Stolperfalle. Es gibt *zwei* Konzepte:

1. **`Ingress`** — Standard-Kubernetes-Ressource, `networking.k8s.io/v1`.
   Ein Ingress-Controller (Nginx, Traefik, HAProxy) implementiert sie.
2. **`IngressRoute`** — Traefik-eigene CRD, `traefik.io/v1alpha1`.
   Reicher als Standard-Ingress (Middlewares, Match-DSL, TLS-Logik).

Wir nutzen IngressRoute, weil Traefik der installierte Ingress-Controller
ist. Darum zeigt `kubectl get ingress` bei uns nichts, aber
`kubectl get ingressroutes.traefik.io` zeigt alles.

### cert-manager

Ein Operator, der Certificate-Ressourcen beobachtet und automatisch bei
Let's Encrypt oder einer anderen ACME-CA ein Zertifikat holt. Speichert es
als Secret, das dann von IngressRoute über `tls.secretName` referenziert
wird. Die HTTP-01-Challenge läuft: CA schickt Request an `http://<host>/
.well-known/acme-challenge/<token>` → Traefik routet zum Solver-Pod →
Solver antwortet → CA stellt Zert aus.

Deshalb war DNS in unserer Session wichtig: ohne korrekten A-Record zeigt
`jakob-miniapp.goava.ai` woanders hin, und die Challenge schlägt fehl.

---

## Teil E — Helm als Templating

Ein Helm-Chart besteht aus:

- **`Chart.yaml`** — Metadaten (Name, Version).
- **`values.yaml`** — Default-Werte (Parameter).
- **`templates/`** — YAML-Dateien mit Go-Template-Syntax (`{{ … }}`).

Helm macht daraus nach `helm template` konkrete K8s-Manifeste. ArgoCD
ruft das intern auf.

### Der Clou: values als einzige Datenquelle

In `charts/miniapp/templates/deployment.yaml`:
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Und in `values.yaml`:
```yaml
image:
  repository: ghcr.io/jakobulus123/miniapp
  tag: "6445eabc…"
```

Ergebnis: `image: "ghcr.io/jakobulus123/miniapp:6445eabc…"` im gerenderten
Deployment.

Warum ist das gut? **Ein Ort für Änderungen.** Wenn du das Tag bumpst,
änderst du genau eine Zeile — und das gesamte Deployment wird konsistent
neu gerendert.

---

## Teil F — GitOps mit ArgoCD

### Was macht ArgoCD überhaupt?

ArgoCD ist ein Controller im Cluster, der eine `Application`-Custom-Resource
liest und zyklisch abgleicht:

```
DESIRED STATE = helm template <chart>  (aus Git)
LIVE STATE    = kubectl get …          (aus dem Cluster)

ArgoCD diffed beides und macht kubectl apply, bis sie übereinstimmen.
```

Wichtig: **Desired State kommt aus Git.** Die Registry ist nicht Teil der
Gleichung.

### Die Application-Resource bei uns

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: miniapp
  namespace: argocd
spec:
  destination:
    namespace: miniapp
    server: https://kubernetes.default.svc
  sources:
  - path: charts/miniapp
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true      # lösche, was nicht mehr in Git ist
      selfHeal: true   # revidere manuelle Cluster-Änderungen
```

### Refresh-Interval

ArgoCD pollt Git *nicht* ständig. Default alle 3 Minuten. Sofort-Sync per:
```bash
kubectl annotate app -n argocd miniapp argocd.argoproj.io/refresh=hard --overwrite
```

### selfHeal — Segen und Fluch

`selfHeal: true` heißt: Wenn im Cluster etwas vom Git-Stand abweicht, macht
ArgoCD es rückgängig. Vorteil: keine „wilden" Änderungen per `kubectl edit`.
Nachteil: `kubectl rollout restart` oder ähnliche Ad-hoc-Eingriffe sind
fragil, weil selfHeal sie potenziell zurückdreht.

**Regel:** Änderungen gehören in den Git-Repo. `kubectl` ist Read-only-Tool.

### Warum `:latest` + GitOps sich beißt

Das war der Kern des Debug-Falls. Noch mal explizit:

1. `values.yaml` sagt `tag: "latest"`.
2. Helm rendert `image: ghcr.io/…/miniapp:latest`.
3. ArgoCD vergleicht: Cluster hat Deployment mit `image:
   ghcr.io/…/miniapp:latest`. **Identisch** → `Synced`.
4. Dass hinter `:latest` in der Registry inzwischen ein anderer Digest liegt,
   interessiert den Vergleich nicht.

**Fix:** Konkreter Tag → jeder neue Build braucht einen Git-Commit → ArgoCD
erkennt die Manifest-Änderung → Deployment bekommt neues
`spec.template.containers[0].image` → neue Pods werden gestartet → frisches
Image wird gepullt (weil der Tag unbekannt ist).

### Optionen jenseits des manuellen Tag-Bumps

- **ArgoCD Image Updater**: Extra-Controller, der die Registry pollt,
  neue Tags findet und den Tag-Bump automatisch als Git-Commit macht.
  Setup: <https://argocd-image-updater.readthedocs.io/>
- **CI-seitiger Commit**: Die Build-Pipeline pusht nach erfolgreichem
  Image-Push einen Commit ins Chart-Repo, der den Tag updated.
  (Vorsicht: Schreibrechte auf Chart-Repo nötig.)
- **Semver-Release**: Du triggerst Deploys bewusst über
  Versionen, nicht über jeden Commit.

---

## Teil G — Von außen erreichbar

Damit ein Request vom Browser beim Pod landet, müssen etliche Hops stimmen:

```
Browser
  │ DNS-Lookup: jakob-miniapp.goava.ai → ???
  │ (bei uns aktuell: 80.158.91.8 ← falsch)
  │ (sollte sein:     89.167.116.105)
  ▼
Router / Node-IP mit MetalLB
  │
  ▼
Traefik (Ingress-Controller, hört auf Port 80/443)
  │ matched Host(`jakob-miniapp.goava.ai`) aus IngressRoute
  │
  ▼
Service miniapp (ClusterIP, Port 80)
  │ LoadBalanced auf Pods per Label-Selector
  │
  ▼
Pod (Port 3000, Node-Prozess)
  │
  ▼
HTTP-Response
```

**Fehlstellen in der Kette:**

- **DNS falsch** → Request kommt nie bei Traefik an
- **Kein Ingress/IngressRoute** → Traefik weiß nichts vom Host, 404
- **Ingress-Class fehlt** → Ingress wird ignoriert
- **Service-Selector passt nicht zum Pod-Label** → Traefik findet keine
  Pods hinter dem Service
- **Pod nicht Ready** → Service leitet trotzdem nicht (Readiness-Probe)
- **Zertifikat nicht valide** → Browser blockt, obwohl technisch alles
  läuft

Für lokales Testen umgeht man die ganze Kette mit Port-Forward:
```bash
kubectl port-forward -n miniapp svc/miniapp 9090:80
```
→ dein lokaler Port 9090 → direkt zum Service → Pod. DNS, Traefik,
Zertifikat sind alle nicht beteiligt.

---

## Teil H — Die Kette im Zusammenspiel (Beispiel: neue Version ausrollen)

Angenommen: Du hast im App-Repo eine neue Feature-Zeile gepusht. Was passiert
konkret?

### Schritt 1 — App-Repo

```
git commit -m "add UI" && git push
```

→ GitHub Actions feuert. Baut Image. Pusht nach GHCR:
```
ghcr.io/jakobulus123/miniapp:latest         ← überschrieben
ghcr.io/jakobulus123/miniapp:<neuer-sha>    ← neu
```

### Schritt 2 — Chart-Repo (hier K8s-Helm)

Du änderst `charts/miniapp/values.yaml`:
```yaml
image:
  tag: "<neuer-sha>"   # vorher: alter SHA
```

`git commit && git push`.

### Schritt 3 — ArgoCD

Wenn Refresh-Zyklus läuft (oder du `refresh=hard` annotierst):
- Liest Git HEAD.
- Helm-rendert das Deployment.
- Neue Image-Zeile → Manifest unterscheidet sich vom Cluster-Stand.
- ArgoCD macht `kubectl apply`.

### Schritt 4 — Kubernetes

- Deployment-Controller sieht geändertes `spec.template`.
- Neues ReplicaSet wird erzeugt.
- Neuer Pod wird geschedult auf einem Node.
- Node-kubelet ruft Container-Runtime: „zieh Image X".
- Runtime pullt (oder nimmt Cache).
- Container startet.
- Readiness-Probe wird grün.
- Service-Endpoint-Controller fügt Pod zum Service-Endpoint hinzu.
- Alter Pod wird terminiert.

### Schritt 5 — Browser

- Browser macht Request.
- DNS → Node-IP.
- Traefik matched Host → Service → neuer Pod.
- 200 OK, neue UI.

**Mind-Map**: Jeder dieser Schritte hat seine eigene Observability.

- App-Repo: `git log`, Actions-Tab
- Registry: `curl .../v2/<repo>/tags/list`
- Chart-Repo: `git log`
- ArgoCD: `kubectl get app -n argocd miniapp -o yaml`, Argo-UI
- Deployment: `kubectl describe deploy -n miniapp miniapp`
- Pod: `kubectl describe pod …`, `kubectl logs …`
- Service: `kubectl get endpoints -n miniapp miniapp`
- Traefik: Traefik-Logs, IngressRoute-Objekt
- DNS: `dig +short`
- Browser: DevTools Network-Tab

---

## Teil I — Debug-Checkliste

Wenn „irgendwas stimmt nicht", geh die Kette von hinten nach vorne:

### 0. Was siehst du eigentlich?

- HTTP-Fehler? Code = Traefik, 500 = App, 502 = Service findet keinen Pod.
- `Cannot GET /` = App läuft, aber kein Route-Handler → App-Logik
- `Bad Gateway` = Traefik erreicht Service nicht
- DNS-Fehler im Browser = Host-Auflösung kaputt
- SSL-Warnung = Zertifikat nicht valide

### 1. Pod

```bash
kubectl get pods -n <ns>
kubectl describe pod -n <ns> <pod>
kubectl logs -n <ns> <pod> --tail=50
```

- Ist Status `Running`? Wie oft restart?
- Welcher Image-Digest läuft?
  ```bash
  kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].imageID}{"\n"}{end}'
  ```
- Wie alt ist der Pod?

### 2. Deployment / Image

- Welches Image steht im Deployment-Spec?
- Ist das der erwartete Tag?
- Passt Tag zu dem, was in Git steht?

### 3. Git ↔ Cluster

- Welcher Git-Commit ist laut ArgoCD deployed?
  ```bash
  kubectl get app -n argocd <app> -o jsonpath='{.status.sync.revisions}'
  ```
- Ist das der aktuelle HEAD von origin/main?
- Sync-Status: `Synced`? Health: `Healthy`?

### 4. Registry

- Existiert das Image mit diesem Tag?
  ```bash
  # Token holen, Tag-Liste abfragen
  TOKEN=$(curl -s 'https://ghcr.io/token?scope=repository:<repo>:pull' | jq -r .token)
  curl -s -H "Authorization: Bearer $TOKEN" https://ghcr.io/v2/<repo>/tags/list
  ```
- Zeigt der Tag auf einen Digest, den du erwartest?

### 5. Service / Netzwerk

- `kubectl get endpoints -n <ns> <service>` — sind Pod-IPs drin?
- `kubectl port-forward -n <ns> svc/<service> 9090:<port>` → `curl localhost:9090`
- Wenn port-forward funktioniert aber Browser nicht: Problem liegt zwischen
  Service und Browser (Ingress, DNS, TLS).

### 6. Ingress

- Standard-Ingress oder Traefik-IngressRoute?
  ```bash
  kubectl get ingress,ingressroutes.traefik.io -A
  ```
- Matched der Host-Eintrag dem gewünschten DNS-Namen?
- Steht TLS-Secret drin und existiert das Secret?

### 7. DNS

- `dig +short <host>` — zeigt das auf die richtige IP?
- Browser-Cache? Host-File?

### 8. cert-manager

```bash
kubectl get certificates,challenges,orders -n <ns>
```
- `READY=False` bei Certificate → was sagen die Events?
- Pending Challenge → DNS oder Ingress-Routing stimmen nicht.

---

## Teil J — Anti-Patterns (aus dieser Session gelernt)

1. **`:latest` in GitOps-Manifesten.**
   ArgoCD sieht keine Änderung, während die Registry sich bewegt.
2. **`kubectl rollout restart` als Fix.**
   In Clustern mit selfHeal kann das revidert werden — außerdem kein Audit.
3. **`kubectl get ingress` und „ist nichts da" als Antwort akzeptieren.**
   Bei Traefik-CRD-Setups muss man gezielt nach `ingressroutes.traefik.io`
   fragen.
4. **DNS und Image-Problem vermischen.**
   Zwei unabhängige Probleme. Lokales Testen via port-forward entkoppelt
   beide.
5. **Annahme, ArgoCD synct sofort.**
   Default 3 Minuten. `refresh=hard` annotation triggert direkt.
6. **Image-Pull-Policy `Always` als Garantie für Aktualität.**
   Greift nur beim Pod-Start, nicht im laufenden Pod.

---

## Weiterlesen

- [`IMAGE-UND-DEPLOYMENT.md`](IMAGE-UND-DEPLOYMENT.md) — kompakter Überblick
- [`DEBUG-IMAGE-TAG-PROBLEM.md`](DEBUG-IMAGE-TAG-PROBLEM.md) — konkrete
  Debug-Session zu `:latest`
- [`ARCHITEKTUR.md`](ARCHITEKTUR.md) — Gesamt-Cluster-Aufbau
- [`KUBERNETES-KONZEPTE.md`](KUBERNETES-KONZEPTE.md) — Objekt-Referenz
- [`YAML-UND-HELM.md`](YAML-UND-HELM.md) — Helm-Templating vertieft
- [`PROZESS-KETTEN.md`](PROZESS-KETTEN.md) — weitere End-to-End-Abläufe
