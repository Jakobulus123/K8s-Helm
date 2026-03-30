# Fehler-Dokumentation: Was schiefgelaufen ist und warum

> Alle Fehler die während des initialen Setups aufgetreten sind.
> Jeder Fehler mit: Symptom → Diagnose → Ursache → Fix → Lesson Learned

---

## Inhaltsverzeichnis

1. [Fehler 1: ArgoCD kann repo-server nicht erreichen](#fehler-1-argocd-kann-repo-server-nicht-erreichen)
2. [Fehler 2: ArgoCD UI zeigt "no available server" (503)](#fehler-2-argocd-ui-zeigt-no-available-server-503)
3. [Fehler 3: cert-manager dauerhaft OutOfSync](#fehler-3-cert-manager-dauerhaft-outofsync)

---

## Fehler 1: ArgoCD kann repo-server nicht erreichen

### Symptom

Die Root-Application wurde angelegt, aber ArgoCD konnte keinen Sync durchführen.
Status war dauerhaft `Unknown`:

```
kubectl get applications -n argocd
# NAME   SYNC STATUS   HEALTH STATUS
# root   Unknown       Healthy
```

In den Application-Details stand:

```
Failed to load target state: failed to generate manifest for source 1 of 1:
rpc error: code = Unavailable
desc = "transport: Error while dialing: dial tcp 10.111.117.63:8081: connect: connection refused"
```

Das bedeutet: Der `argocd-application-controller` konnte den `argocd-repo-server`
auf Port 8081 nicht erreichen — obwohl der Pod lief und healthy war.

---

### Diagnose

**Schritt 1 — Sind Endpoints vorhanden?**
```bash
kubectl get endpoints argocd-repo-server -n argocd
# NAME                 ENDPOINTS   AGE
# argocd-repo-server   <none>      2d20h
```
`<none>` bedeutet: Der Service hat keinen Pod gefunden, an den er Traffic schicken kann.

**Schritt 2 — EndpointSlice prüfen:**
```bash
kubectl get endpointslice -n argocd | grep repo
# argocd-repo-server-rhbp9   IPv4   <unset>   10.244.0.243   2d20h
#                                    ↑ Ports sind "unset"!
```
Der EndpointSlice hat die Pod-IP aber **keine Ports** (`<unset>`). Das heißt kube-proxy
weiß nicht, welchen Port er auf dem Pod ansprechen soll → Traffic kann nicht ankommen.

**Schritt 3 — Warum fehlen die Ports?**

Der Service nutzt einen **Named Port**:
```bash
kubectl get svc argocd-repo-server -n argocd -o jsonpath='{.spec.ports}'
# [{"name":"https-repo-server","port":8081,"targetPort":"repo-server"}]
#                                                         ↑ Named Port!
```

Der Pod hat aber **keinen Namen** auf seinem Port:
```bash
kubectl get pod argocd-repo-server-... -n argocd \
  -o jsonpath='{.spec.containers[0].ports}'
# [{"containerPort":8081,"protocol":"TCP"}]
#   ↑ Kein "name" Feld!
```

Kubernetes kann `targetPort: "repo-server"` nicht auflösen,
weil kein Container-Port diesen Namen hat.
→ EndpointSlice bekommt `ports: null`
→ kube-proxy programmiert keine iptables-Regeln
→ Connection refused

---

### Ursache

Das ArgoCD Deployment war offenbar in einem inkonsistenten Zustand.
Wahrscheinlich wurde ArgoCD manuell oder teilweise neu installiert,
wobei die Container-Port-Namen verloren gingen.

Für Named Ports gilt in Kubernetes:
```
Service.spec.ports[].targetPort = "repo-server"
                                       ↕ muss übereinstimmen
Pod.spec.containers[].ports[].name = "repo-server"
```

Wenn diese Verbindung fehlt, hat der Service keine Endpoints.

---

### Fix

Named Ports im Deployment nachtragen:

```bash
kubectl patch deployment argocd-repo-server -n argocd --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/ports/0",
      "value": {"containerPort": 8081, "name": "repo-server", "protocol": "TCP"}
    },
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/ports/1",
      "value": {"containerPort": 8084, "name": "metrics", "protocol": "TCP"}
    }
  ]'
```

Danach rollte Kubernetes das Deployment neu aus. Der neue Pod hatte die korrekten
Port-Namen und Kubernetes konnte den EndpointSlice befüllen:

```bash
kubectl get endpointslice -n argocd | grep repo
# argocd-repo-server-rhbp9   IPv4   8081   10.244.0.245   2d20h
#                                    ↑ Port ist jetzt gesetzt!
```

Danach Application-Controller neu starten damit er die Verbindung neu aufbaut:
```bash
kubectl delete pod argocd-application-controller-0 -n argocd
```

---

### Lesson Learned

**Named Ports sind fehleranfällig.** Wenn ein Service `targetPort: "name"` (String statt
Zahl) nutzt, muss der Container-Port diesen Namen explizit haben. Fehlt der Name, gibt
es keinerlei Fehlermeldung beim Anlegen des Services — der Service existiert einfach
ohne Endpoints, still und leise.

Debugging-Reihenfolge bei "connection refused" zu einem Service:
```
1. kubectl get endpoints <service> -n <namespace>
2. kubectl get endpointslice -n <namespace>  → Ports gesetzt?
3. kubectl get svc <service> -o yaml         → targetPort: String oder Zahl?
4. kubectl get pod <pod> -o jsonpath='{.spec.containers[0].ports}'
5. Vergleichen: Service-Selector vs Pod-Labels
```

---

## Fehler 2: ArgoCD UI zeigt "no available server" (503)

### Symptom

Traefik war deployed, das TLS-Zertifikat war `READY: True`, die IngressRoute war
angelegt — aber beim Aufruf von `https://jakob-argocd.goava.ai` kam:

```
HTTP/2 503
no available server
```

TLS hat funktioniert (Zertifikat wurde korrekt präsentiert), aber Traefik
konnte keinen Backend-Pod erreichen.

---

### Diagnose

**Schritt 1 — argocd-server Endpoints prüfen:**
```bash
kubectl get endpoints argocd-server -n argocd
# NAME            ENDPOINTS   AGE
# argocd-server   <none>      2d21h
```
Wieder keine Endpoints — gleiche Symptomatik wie Fehler 1.

**Schritt 2 — Diesmal aber anderer Grund. Service-Selector vs Pod-Labels:**
```bash
# Was der Service erwartet:
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.selector}'
# {"app.kubernetes.io/instance":"argocd","app.kubernetes.io/name":"argocd-server"}
#   ↑ BEIDE Labels müssen auf dem Pod sein

# Was der Pod tatsächlich hat:
kubectl get pod argocd-server-... -n argocd -o jsonpath='{.metadata.labels}'
# {"app.kubernetes.io/name":"argocd-server","pod-template-hash":"8486b55865"}
#   ↑ "app.kubernetes.io/instance: argocd" FEHLT!
```

Der Service findet den Pod nicht, weil ein Label fehlt.
Alle Services in ArgoCD haben diesen Selector mit beiden Labels,
aber die Pods hatten nur eines davon.

---

### Ursache

ArgoCD wurde mit Helm installiert. Das Helm-Chart setzt `app.kubernetes.io/instance: argocd`
als Label auf alle Pods. Offenbar wurde ArgoCD zu einem früheren Zeitpunkt mit einer
anderen Konfiguration installiert oder das Label ging beim manuellen Eingriff verloren.

Das betraf nicht nur `argocd-server` sondern alle ArgoCD Deployments:
- `argocd-server`
- `argocd-repo-server` (hatte denselben Fehler, Fehler 1)
- `argocd-applicationset-controller`
- `argocd-notifications-controller`
- `argocd-dex-server`

---

### Fix

Fehlendes Label zu allen betroffenen Deployments hinzufügen:

```bash
for deploy in argocd-server argocd-repo-server \
              argocd-applicationset-controller \
              argocd-notifications-controller \
              argocd-dex-server; do
  kubectl patch deployment $deploy -n argocd --type=json \
    -p='[{
      "op": "add",
      "path": "/spec/template/metadata/labels/app.kubernetes.io~1instance",
      "value": "argocd"
    }]'
done
```

**Wichtig:** In JSON Patch muss `/` in Schlüsseln als `~1` kodiert werden.
`app.kubernetes.io/instance` → Pfad: `app.kubernetes.io~1instance`

Nach dem Patch rollte Kubernetes neue Pods aus, diesmal mit beiden Labels.
Die Services fanden ihre Pods und Endpoints wurden befüllt.

---

### Lesson Learned

Kubernetes Services matchen Pods über **Label-Selektoren**. Fehlt auch nur ein Label,
ist der Service leer — ohne Fehlermeldung.

Ein Service mit Selector `{a: x, b: y}` findet **nur** Pods die **beide** Labels haben.
Ein Pod mit nur `{a: x}` wird ignoriert.

Debugging-Reihenfolge bei 503 von Traefik/Ingress:
```
1. Traefik-Logs: kubectl logs -n traefik <traefik-pod>
2. Service-Endpoints: kubectl get endpoints <service> -n <namespace>
3. Service-Selector: kubectl get svc <service> -o jsonpath='{.spec.selector}'
4. Pod-Labels: kubectl get pod <pod> -o jsonpath='{.metadata.labels}'
5. Vergleich: Decken sich Selector und Labels?
```

Schneller Check:
```bash
# Zeigt direkt welche Pods ein Service findet
kubectl get pods -n argocd -l \
  "app.kubernetes.io/name=argocd-server,app.kubernetes.io/instance=argocd"
```

---

## Fehler 3: cert-manager dauerhaft OutOfSync

### Symptom

Nach dem initialen Sync war cert-manager `Healthy` (alle Pods liefen), aber der
Sync-Status blieb dauerhaft `OutOfSync`:

```bash
kubectl get application cert-manager -n argocd
# NAME           SYNC STATUS   HEALTH STATUS
# cert-manager   OutOfSync     Healthy
```

ArgoCD sync'd immer wieder, direkt danach war es wieder OutOfSync.
Ein Endloskreislauf ohne Fehler in den Pods selbst.

---

### Diagnose

**Schritt 1 — Welche Ressourcen sind OutOfSync?**
```bash
kubectl get application cert-manager -n argocd -o json | python3 -c "
import sys, json
d = json.load(sys.stdin)
for r in d['status']['resources']:
    if r.get('status') == 'OutOfSync':
        print(r['kind'], r['name'])
"
# CustomResourceDefinition certificaterequests.cert-manager.io
# CustomResourceDefinition certificates.cert-manager.io
# CustomResourceDefinition challenges.acme.cert-manager.io
# CustomResourceDefinition orders.acme.cert-manager.io
```

Nur die 4 CRDs sind betroffen, nicht die Deployments oder Services.

**Schritt 2 — Was ist der konkrete Unterschied?**

Was Helm rendert (was ArgoCD deployen will):
```bash
helm template cert-manager jetstack/cert-manager --version v1.20.0 \
  --set crds.enabled=true | grep -A3 "selectableFields"
# selectableFields: []   ← Helm fügt dieses Feld hinzu
```

Was der Cluster nach dem Apply tatsächlich hat:
```bash
kubectl get crd certificates.cert-manager.io \
  -o jsonpath='{.spec.versions[0]}' | python3 -m json.tool | grep selectable
# (kein Output) ← Cluster hat das Feld nicht!
```

**Schritt 3 — Warum streicht Kubernetes das Feld?**

`selectableFields` ist ein Kubernetes-Feature das ab Version 1.30 eingeführt wurde.
Unser Cluster läuft auf einer älteren Version → Kubernetes kennt das Feld nicht →
es wird beim Apply automatisch entfernt.

ArgoCD sieht:
- Git/Helm: CRD hat `selectableFields`
- Cluster: CRD hat kein `selectableFields`
- → OutOfSync → Sync → Kubernetes entfernt wieder → OutOfSync → ...

---

### Fix-Versuche (was nicht funktioniert hat)

**Versuch 1: `ServerSideApply=true`**
```yaml
syncOptions:
  - ServerSideApply=true
```
Kein Effekt. Server-Side Apply ändert nichts daran, dass Kubernetes das
Feld entfernt.

**Versuch 2: `ignoreDifferences` mit `jsonPointers`**
```yaml
ignoreDifferences:
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jsonPointers:
      - /spec/conversion
      - /spec/preserveUnknownFields
```
Kein Effekt, weil das konkrete Feld `selectableFields` tiefer liegt
(`/spec/versions/0/selectableFields`) und wir es nicht korrekt adressiert hatten.

**Versuch 3: `jqPathExpressions`**
```yaml
ignoreDifferences:
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jqPathExpressions:
      - .spec.versions[].selectableFields
```
Hat nicht funktioniert weil ArgoCD einen Hard-Refresh brauchte und die
App noch durch die root-App auf die alte Version zurückgesetzt wurde.

---

### Echter Fix

CRDs **gar nicht von Helm verwalten lassen**:

```yaml
# values/cert-manager/values.yaml
crds:
  enabled: false   # ← CRDs werden nicht durch Helm deployed/verwaltet
```

Die CRDs sind bereits im Cluster und funktionieren korrekt. Helm und damit ArgoCD
fassen sie nicht mehr an → kein Vergleich → kein OutOfSync.

Nach dem Push und Hard-Refresh:
```bash
kubectl annotate application cert-manager -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite

kubectl get application cert-manager -n argocd
# NAME           SYNC STATUS   HEALTH STATUS
# cert-manager   Synced        Healthy ✓
```

---

### Ursache (tieferes Verständnis)

Das ist ein bekanntes Problem in der ArgoCD + cert-manager Community.
cert-manager CRDs enthalten riesige OpenAPI-Validierungsschemas und nutzen
neue Kubernetes-Features sobald sie verfügbar sind.

Das Grundproblem: **Helm Chart-Version und Kubernetes-Version sind nicht kompatibel
für dieses spezifische Feature** (`selectableFields` requires k8s >= 1.30).

Wenn man das nicht lösen kann (Cluster-Version nicht updaten), gibt es zwei Optionen:

**Option A**: CRDs separat installieren (außerhalb von Helm):
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.0/cert-manager.crds.yaml
```
Dann `crds.enabled: false` im Helm Chart.

**Option B**: Ältere Chart-Version nutzen die `selectableFields` noch nicht verwendet.

Wir haben Option A gewählt.

---

### Lesson Learned

**CRDs von großen Charts (cert-manager, Prometheus-Operator, etc.) sollte man
grundsätzlich nicht über Helm verwalten wenn man ArgoCD nutzt.**

Gründe:
1. CRDs sind cluster-weite Ressourcen — ein Helm Upgrade in einem Namespace
   kann CRDs aller anderen Helm Releases brechen
2. Kubernetes modifiziert CRDs nach dem Apply (Schema-Normalisierung, Defaults,
   feature-gate-abhängige Felder) → dauerhafter Drift
3. Helm kann CRDs nicht löschen (`helm uninstall` löscht CRDs nicht)

Best Practice:
```
CRDs separat installieren (kubectl apply oder eigene ArgoCD App)
Helm Chart mit crds.enabled: false deployen
```

---

## Zusammenfassung aller Fehler

| # | Fehler | Ursache | Gefährlichkeit | Fix |
|---|--------|---------|----------------|-----|
| 1 | ArgoCD repo-server nicht erreichbar | Named Port ohne Namen auf Pod | Kritisch — ArgoCD komplett funktionslos | `kubectl patch deployment` um Port-Namen hinzuzufügen |
| 2 | ArgoCD UI 503 | Fehlendes Label `app.kubernetes.io/instance` auf Pods | Kritisch — UI nicht erreichbar | Label auf alle ArgoCD Deployments patchen |
| 3 | cert-manager OutOfSync | `selectableFields` CRD-Feature erfordert k8s >= 1.30 | Minor — Pods laufen, nur Drift | `crds.enabled: false` in Helm Values |

Fehler 1 und 2 hatten dieselbe Grundursache: **inkonsistenter Cluster-Zustand**
durch eine frühere Installation die nicht vollständig mit dem Helm-Chart übereinstimmte.

Fehler 3 ist ein **Versions-Inkompatibilitäts-Problem** zwischen Helm Chart und
Kubernetes-Version — ein sehr häufiges Problem in der Praxis.

---

*Dokumentiert während des initialen Setups auf Cluster 89.167.116.105*
