# Wie alles zusammenhängt — Die komplette Prozesskette

> Diese Dokumentation erklärt WARUM jede Komponente existiert, WIE sie mit den
> anderen kommuniziert und WAS passiert wenn eine Komponente fehlt oder kaputt ist.
> Geschrieben für Anfänger die nicht nur wissen wollen WAS sie tippen,
> sondern WARUM es funktioniert.

---

## Inhaltsverzeichnis

1. [Das große Bild — Bevor wir ins Detail gehen](#1-das-große-bild)
2. [Die Reise eines Requests von Browser bis Pod](#2-die-reise-eines-requests)
3. [Die Reise eines Git-Commits bis zum laufenden Container](#3-die-reise-eines-git-commits)
4. [Wie Kubernetes intern kommuniziert](#4-wie-kubernetes-intern-kommuniziert)
5. [Wie ArgoCD intern funktioniert](#5-wie-argocd-intern-funktioniert)
6. [Wie Traefik Traffic verteilt](#6-wie-traefik-traffic-verteilt)
7. [Wie cert-manager TLS-Zertifikate holt](#7-wie-cert-manager-tls-zertifikate-holt)
8. [Warum jede Komponente die andere braucht](#8-warum-jede-komponente-die-andere-braucht)
9. [Was passiert wenn eine Komponente ausfällt](#9-was-passiert-wenn-eine-komponente-ausfällt)
10. [Das vollständige Zusammenspiel als Diagramm](#10-das-vollständige-zusammenspiel)

---

## 1. Das große Bild

Bevor wir in Details gehen, hier die Frage die du dir stellen solltest:

> **"Was wollen wir eigentlich erreichen?"**

Wir wollen, dass:
1. Jemand eine YAML-Datei in GitHub pusht
2. Eine Anwendung automatisch im Cluster deployed wird
3. Die Anwendung per HTTPS erreichbar ist
4. Das TLS-Zertifikat automatisch ausgestellt und erneuert wird
5. Das alles ohne manuelles Eingreifen passiert

Das klingt einfach. Aber dahinter stecken viele Systeme die zusammenarbeiten müssen.

### Die Akteure

```
┌─────────────────────────────────────────────────────────────────┐
│                         Unser System                            │
│                                                                 │
│  GitHub ──► ArgoCD ──► Kubernetes ──► Traefik ──► cert-manager │
│                                                                 │
│  "Wahrheit"  "Wächter"  "Betriebssystem"  "Türsteher"  "Notar" │
└─────────────────────────────────────────────────────────────────┘
```

Jede Komponente hat eine klare Rolle:

| Komponente | Rolle | Analogie |
|------------|-------|----------|
| **GitHub** | Speichert den gewünschten Zustand | Bauplan eines Hauses |
| **ArgoCD** | Vergleicht Bauplan mit echtem Haus und baut nach | Bauleiter |
| **Kubernetes** | Führt aus, verwaltet Container | Betriebssystem |
| **Traefik** | Verteilt eingehenden Traffic | Empfangsdame im Hotel |
| **cert-manager** | Besorgt und erneuert TLS-Zertifikate | Notar der Echtheit bestätigt |

---

## 2. Die Reise eines Requests

Wenn du `https://jakob-argocd.goava.ai` im Browser eingibst, passiert folgendes —
und jeder Schritt ist wichtig:

### Schritt 1: DNS-Auflösung

```
Browser: "Wo ist jakob-argocd.goava.ai?"
    │
    ▼
DNS-Resolver (z.B. 8.8.8.8)
    │ Sucht im DNS-System nach dem A-Record
    ▼
Antwort: "89.167.116.105"
    │
    ▼
Browser: "OK, ich verbinde mich mit 89.167.116.105"
```

**DNS** (Domain Name System) ist das Telefonbuch des Internets.
Jede Domain hat einen **A-Record** der auf eine IP zeigt:
```
jakob-argocd.goava.ai.  IN  A  89.167.116.105
```

Ohne diesen Eintrag beim DNS-Provider kann der Browser die IP nicht finden.
Das ist außerhalb von Kubernetes — das muss man manuell beim Domain-Anbieter setzen.

---

### Schritt 2: TCP-Verbindung + TLS Handshake

```
Browser verbindet sich mit 89.167.116.105:443
    │
    ▼
TCP Three-Way Handshake:
    Browser: "SYN" (Ich will reden)
    Server:  "SYN-ACK" (OK, ich höre)
    Browser: "ACK" (Verstanden)
    │
    ▼
TLS Handshake:
    Browser: "Client Hello" (Welche TLS-Versionen kannst du? Ich kann 1.2, 1.3)
    Server:  "Server Hello" + Zertifikat (Ich kann 1.3, hier ist mein Ausweis)
    Browser: Prüft Zertifikat:
             - Ist die Domain korrekt? (jakob-argocd.goava.ai ✓)
             - Ist es noch gültig? (Ablaufdatum ✓)
             - Ist es von einer CA signiert die ich kenne? (Let's Encrypt ✓)
    Browser: "Finished" (Alles gut, ab jetzt verschlüsseln wir)
```

**Wer hört auf Port 443?** Das ist der **Traefik-Pod** über den LoadBalancer-Service.

Kubernetes hat einen Service vom Typ `LoadBalancer` der externe Anfragen auf Port 443
an den Traefik-Pod weiterleitet:
```
Außenwelt:443 → LoadBalancer Service → Traefik Pod:8443
```

**Das Zertifikat** das der Server schickt, kommt aus einem Kubernetes Secret:
```
Secret "argocd-tls" im Namespace "argocd"
  ├── tls.crt  (das öffentliche Zertifikat)
  └── tls.key  (der private Schlüssel)
```

Dieses Secret wurde von **cert-manager** erstellt. Dazu später mehr.

---

### Schritt 3: Traefik entscheidet wohin der Traffic geht

```
Traefik erhält HTTPS-Request für jakob-argocd.goava.ai
    │
    ▼
Traefik liest alle IngressRoutes im Cluster
    │
    ▼
Vergleich der Host-Regeln:
    IngressRoute "argocd": Host(`jakob-argocd.goava.ai`) → MATCH!
    │
    ▼
Middleware "argocd-headers" wird angewendet:
    X-Forwarded-Proto: https wird hinzugefügt
    │
    ▼
Traefik sucht Service "argocd-server" im Namespace "argocd" Port 80
    │
    ▼
Service "argocd-server" hat Endpoints: 10.244.0.253:8080
    │
    ▼
Traefik schickt Request an 10.244.0.253:8080 (ArgoCD Pod)
```

**Warum der Header `X-Forwarded-Proto: https`?**

ArgoCD selbst kann auch HTTPS. Wenn es nicht weiß dass Traefik schon TLS macht,
denkt es: "Der Browser kommt per HTTP an, ich muss auf HTTPS redirecten."
→ Browser landet bei `https://jakob-argocd.goava.ai`
→ Traefik leitet weiter an ArgoCD Port 80 (HTTP)
→ ArgoCD redirectet wieder auf HTTPS
→ Endlosschleife (Redirect Loop)

Mit dem Header sagt Traefik ArgoCD: "Hey, der Browser hat schon HTTPS genutzt,
du musst nicht mehr redirecten."

---

### Schritt 4: ArgoCD antwortet

```
ArgoCD Pod empfängt Request
    │
    ▼
ArgoCD Server (Go-Anwendung auf Port 8080)
    │
    ▼
Sendet HTML/CSS/JS der Web-UI
    │
    ▼
Traefik bekommt Response
    │
    ▼
Traefik schickt Response zurück an Browser (verschlüsselt über TLS)
    │
    ▼
Browser rendert ArgoCD UI ✓
```

### Die vollständige Request-Kette auf einen Blick

```
Browser
  │ HTTPS GET https://jakob-argocd.goava.ai/
  ▼
DNS → 89.167.116.105
  │
  ▼
Node (89.167.116.105)
  │ iptables DNAT: Port 443 → Traefik Service ClusterIP
  ▼
kube-proxy Routing
  │ Service → EndpointSlice → Traefik Pod IP
  ▼
Traefik Pod (10.244.0.x:8443)
  │ TLS terminieren (Secret argocd-tls)
  │ IngressRoute matchen
  │ Middleware anwenden
  ▼
kube-proxy Routing
  │ Service argocd-server → ArgoCD Pod IP
  ▼
ArgoCD Pod (10.244.0.x:8080)
  │ HTTP Response
  ▼
(zurück durch die Kette)
  ▼
Browser zeigt ArgoCD ✓
```

---

## 3. Die Reise eines Git-Commits

Wenn du eine neue Datei in `apps/` pushst, passiert folgendes:

### Schritt 1: Git Push

```bash
git push origin main
```

```
Dein Rechner
  │ SSH-Verbindung zu github.com:22
  ▼
GitHub
  │ Nimmt Commit entgegen
  │ Speichert in Repository
  ▼
Repository aktualisiert ✓
```

**SSH-Key**: GitHub authentifiziert dich per SSH-Key (nicht Passwort).
ArgoCD nutzt ebenfalls einen SSH-Key um das Repository zu lesen.
Das ist sicherer als Passwörter — Private Key bleibt bei dir, Public Key ist bei GitHub.

---

### Schritt 2: ArgoCD bemerkt die Änderung

```
ArgoCD-Application-Controller läuft im Cluster
  │ Pollt alle 3 Minuten: "Hat sich etwas im Git-Repo geändert?"
  │
  │ (Optional: Webhook für sofortige Benachrichtigung)
  ▼
argocd-repo-server wird beauftragt:
  │ "Klon das Repo und rendere den Inhalt von apps/"
  ▼
argocd-repo-server:
  1. SSH-Verbindung zu github.com
  2. git clone / git fetch
  3. Liest YAML-Dateien aus apps/
  4. Das sind ArgoCD Application CRDs → gibt sie zurück
  ▼
argocd-application-controller:
  │ Vergleicht: Was ist jetzt in Git vs was ist im Cluster?
  │ Neue Datei "apps/meine-neue-app.yaml" gefunden!
  │ Diese Application existiert noch nicht im Cluster.
  ▼
Sync wird ausgelöst (weil automated: true)
```

---

### Schritt 3: ArgoCD synct die neue Application

```
ArgoCD führt kubectl apply aus (intern):
  kubectl apply -f <gerendertes YAML>
    │
    ▼
Kubernetes API-Server empfängt die neue Application-Ressource
    │ Validiert das YAML (Schema-Check)
    │ Speichert in etcd
    ▼
ArgoCD-Application-Controller bemerkt neue Application
    │ "Es gibt jetzt eine Application 'meine-neue-app'"
    │ Was steht drin? source.path = "charts/meine-app"
    ▼
argocd-repo-server wird beauftragt:
    │ "Klon das Repo, geh in charts/meine-app/, rendere Helm/YAML"
    ▼
argocd-repo-server:
    1. SSH nach GitHub
    2. git fetch (schon gecacht)
    3. Helm template rendern ODER YAML lesen
    4. Fertige Kubernetes-Manifeste zurückgeben
    ▼
argocd-application-controller:
    │ Vergleicht Manifeste mit Cluster-Zustand
    │ Neue Deployments, Services etc. gefunden
    │ → Sync ausführen
    ▼
kubectl apply für alle Ressourcen der neuen App
    ▼
Kubernetes erstellt Deployment, Service, etc.
    ▼
kubelet auf dem Node startet Container
    ▼
Anwendung läuft ✓
```

### Warum dauert das manchmal länger?

ArgoCD pollt standardmäßig alle **3 Minuten**. Das heißt: Nach einem `git push`
kann es bis zu 3 Minuten dauern bis ArgoCD die Änderung bemerkt.

Für schnellere Reaktion kann man einen **Git Webhook** einrichten:
GitHub benachrichtigt ArgoCD sofort nach jedem Push.

```
git push
  │
  ▼ (sofort, statt bis zu 3 Minuten)
GitHub sendet POST-Request an ArgoCD Webhook-URL
  │
  ▼
ArgoCD startet sofort Refresh
```

---

## 4. Wie Kubernetes intern kommuniziert

Das ist das Fundament. Alles andere baut darauf auf.

### Der API-Server als zentraler Hub

```
┌─────────────────────────────────────────────────────────────────┐
│                         etcd (Datenbankschicht)                 │
│              speichert ALLES: Pods, Services, Secrets...        │
└───────────────────────────┬─────────────────────────────────────┘
                            │ lesen/schreiben
┌───────────────────────────▼─────────────────────────────────────┐
│                      API-Server (REST API)                      │
│         Einziger Zugang zu allen Kubernetes-Ressourcen          │
└──┬──────────┬─────────────┬──────────────┬───────────────┬──────┘
   │          │             │              │               │
kubectl   ArgoCD      Controller-      Scheduler      kubelet
          repo-server   Manager                       (auf Node)
```

**Kein Komponente kommuniziert direkt mit einer anderen.**
Alles läuft über den API-Server. Das ist das Kernprinzip von Kubernetes.

Wenn ArgoCD ein Deployment erstellen will:
```
ArgoCD → POST /apis/apps/v1/namespaces/traefik/deployments → API-Server
                                                                │
                                                           speichert in etcd
                                                                │
                                                     Deployment Controller bemerkt
                                                     neues Deployment → erstellt ReplicaSet
                                                                │
                                                     ReplicaSet Controller bemerkt
                                                     kein Pod existiert → erstellt Pod
                                                                │
                                                     Scheduler bemerkt
                                                     Pod ohne Node → weist Node zu
                                                                │
                                                     kubelet auf Node bemerkt
                                                     Pod für seinen Node → startet Container
```

Jeder Schritt ist **deklarativ**: Ein Controller sieht "Desired State ≠ Actual State"
und handelt. Niemand gibt direkte Befehle an jemand anderen.

### Watch-Mechanismus

Controller schlafen nicht und fragen nicht ständig nach. Sie **watchen** den API-Server:

```bash
# Intern macht jeder Controller sowas:
GET /apis/apps/v1/deployments?watch=true
# → Offene HTTP-Verbindung
# → API-Server schickt Events wenn sich etwas ändert
```

Das ist effizient: Kein Polling, sofortige Reaktion.

### Label-Selektoren: Das Bindeglied

Labels sind Key-Value-Paare auf Kubernetes-Ressourcen. Sie sind das Herzstück
der losen Kopplung in Kubernetes.

```yaml
# Deployment gibt Pods diese Labels:
metadata:
  labels:
    app: argocd-server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/instance: argocd
```

```yaml
# Service sucht Pods mit diesen Labels:
spec:
  selector:
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/instance: argocd   ← BEIDE müssen matchen
```

Wenn ein Pod ein Label verliert (durch fehlerhaften Patch), verschwindet er
aus dem Service — der Service findet ihn nicht mehr. Das war **Fehler 1 und 2**
in diesem Setup.

**Warum lose Kopplung?**
Service und Pod kennen sich nicht direkt. Der Service fragt nicht:
"Wo ist Pod argocd-server-8486b55865-xjl2z?"
Er fragt: "Welche Pods haben Label X und Y?"

Das bedeutet: Pods können ausgetauscht werden (Updates, Neustarts) ohne den Service
ändern zu müssen. Solange die Labels stimmen, findet der Service sie.

---

## 5. Wie ArgoCD intern funktioniert

### Die drei Kernkomponenten

```
┌────────────────────────────────────────────────────────────────┐
│                          ArgoCD                                │
│                                                                │
│  ┌─────────────────┐                                          │
│  │  argocd-server  │  Web UI + REST API + gRPC API            │
│  │  Port 8080/8083 │  Authentifizierung, Autorisierung         │
│  └────────┬────────┘  Was der Benutzer sieht                  │
│           │ spricht mit                                        │
│  ┌────────▼───────────────────────────────────────────────┐   │
│  │         argocd-application-controller                   │   │
│  │  StatefulSet (pod-0)                                    │   │
│  │                                                         │   │
│  │  - Watcht alle Application CRDs im Cluster              │   │
│  │  - Berechnet Diff: Git-State vs Cluster-State           │   │
│  │  - Führt Sync durch (kubectl apply intern)              │   │
│  │  - Setzt Health-Status der Applications                 │   │
│  └────────┬───────────────────────────────────────────────┘   │
│           │ beauftragt                                         │
│  ┌────────▼───────────────────────────────────────────────┐   │
│  │              argocd-repo-server                         │   │
│  │  Port 8081 (gRPC)                                       │   │
│  │                                                         │   │
│  │  - Klont Git-Repositories                               │   │
│  │  - Rendert Helm Charts (helm template)                  │   │
│  │  - Rendert Kustomize                                    │   │
│  │  - Gibt fertige Kubernetes-YAML zurück                  │   │
│  │  - Cached Repos lokal                                   │   │
│  └────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### Der Sync-Prozess im Detail

```
Trigger: git push ODER 3-Minuten-Poll ODER manueller Refresh
  │
  ▼
application-controller → repo-server:
  "Render mir den Inhalt von:
   Repo: git@github.com:Jakobulus123/K8s-Helm.git
   Branch: HEAD
   Path: apps/"
  │
  ▼
repo-server:
  1. SSH-Verbindung zu GitHub (nutzt Secret 'repo-727441422')
  2. git fetch (oder git clone beim ersten Mal)
  3. git checkout HEAD
  4. Liest alle .yaml Dateien in apps/
  5. Parst YAML → Kubernetes-Objekte
  6. Gibt zurück: [Application traefik, Application cert-manager, ...]
  │
  ▼
application-controller hat jetzt den "Desired State" (aus Git)
  │
  ▼
application-controller fragt Kubernetes API-Server:
  "Was existiert aktuell im Cluster?" → "Live State"
  │
  ▼
Diff-Berechnung:
  Desired (Git)          Live (Cluster)
  ─────────────          ──────────────
  traefik App ✓    vs    traefik App ✓   → Synced
  cert-manager ✓   vs    cert-manager ✓  → Synced
  meine-neue-app   vs    (existiert nicht) → OutOfSync!
  │
  ▼
automated: true → Sync wird ausgeführt
  │
  ▼
application-controller macht intern:
  kubectl apply -f <rendered manifest>
  │
  ▼
Kubernetes erstellt die Ressource
  │
  ▼
application-controller watcht den Status der erstellten Ressource
  │ Ist das Deployment ready? Laufen die Pods?
  ▼
Setzt Health-Status: Progressing → Healthy
```

### Warum ist der repo-server getrennt vom application-controller?

Sicherheit und Isolation. Der repo-server:
- Führt fremden Code aus (Helm-Templates können Scripting enthalten)
- Hat Zugriff auf Git-Credentials
- Muss von der Außenwelt isoliert sein

Der application-controller:
- Hat Zugriff auf die Kubernetes API (kann alles im Cluster tun)
- Sollte keine direkte Git-Credentials haben

Durch die Trennung gilt: Selbst wenn ein Angreifer den repo-server kompromittiert,
hat er keinen direkten Cluster-Zugriff. Das application-controller hat Cluster-Zugriff
aber keinen Git-Credentials-Zugriff.

### gRPC statt HTTP

Der application-controller kommuniziert mit dem repo-server per **gRPC** (Port 8081),
nicht per HTTP. gRPC ist:
- Schneller (binäres Protokoll statt Text)
- Typsicher (Protocol Buffers)
- Unterstützt Streaming

Das war relevant für **Fehler 1**: Der Fehler "connection refused" kam weil der
gRPC-Port 8081 nicht erreichbar war.

---

## 6. Wie Traefik Traffic verteilt

### Traefik als Edge Proxy

```
Internet
   │
   ▼
Traefik Pod (der einzige Pod mit direktem Außen-Zugriff)
   │
   ├── jakob-argocd.goava.ai → ArgoCD
   ├── andere-app.goava.ai   → Andere App
   └── api.goava.ai          → API Service
```

Traefik ist der **einzige Eintrittspunkt** ins Cluster für HTTP/HTTPS-Traffic.
Kein anderer Pod ist direkt von außen erreichbar (außer über NodePort/LoadBalancer,
was man vermeiden sollte).

### Wie Traefik seine Konfiguration bekommt

Das ist ein wesentlicher Unterschied zu klassischem nginx:
**Traefik konfiguriert sich selbst dynamisch.**

```
Traefik Pod läuft
  │
  │ Watcht Kubernetes API-Server:
  │   - Alle IngressRoute CRDs
  │   - Alle Middleware CRDs
  │   - Alle Services und Endpoints
  ▼
Jemand erstellt eine neue IngressRoute:
  kubectl apply -f meine-ingressroute.yaml
  │
  ▼
Kubernetes API-Server speichert sie in etcd
  │
  ▼
Traefik bemerkt sofort (Watch) die neue IngressRoute
  │
  ▼
Traefik aktualisiert intern seine Routing-Tabelle:
  Host(`meine-app.example.com`) → Service meine-app:80
  │
  ▼
Ohne Restart, ohne Config-Reload — sofort aktiv ✓
```

Das ist **Kubernetes-nativ**: Traefik nutzt die Kubernetes API als
Konfigurationsquelle statt statischer Config-Dateien.

### Das Routing im Detail

```
Incoming Request: GET https://jakob-argocd.goava.ai/settings
  │
  ▼
Traefik: TLS terminieren
  Öffnet Secret "argocd-tls" aus Namespace "argocd"
  Entschlüsselt mit tls.key
  Präsentiert tls.crt
  │
  ▼
Traefik: EntryPoint bestimmen
  Port 443 → EntryPoint "websecure"
  │
  ▼
Traefik: Welche IngressRoute passt?
  Durchsucht alle IngressRoutes mit entryPoints: [websecure]
  Findet: Host(`jakob-argocd.goava.ai`) → MATCH
  │
  ▼
Traefik: Middlewares ausführen (in Reihenfolge)
  argocd-headers:
    Fügt Header hinzu: X-Forwarded-Proto: https
  │
  ▼
Traefik: Service auflösen
  Service: argocd-server, Port: 80
  │
  Fragt Kubernetes API: "Welche Endpoints hat argocd-server?"
  Antwort: 10.244.0.253:8080
  │
  ▼
Traefik: Load Balancing (falls mehrere Endpoints)
  Nur einer → nimmt ihn direkt
  Mehrere → Round Robin
  │
  ▼
Traefik: HTTP Request weiterleiten
  POST http://10.244.0.253:8080/settings
  Headers: X-Forwarded-Proto: https, X-Forwarded-For: <Browser-IP>
  │
  ▼
ArgoCD Pod antwortet
  │
  ▼
Traefik: Response zurück an Browser (verschlüsselt)
```

### Warum IngressRoute statt nativen Kubernetes Ingress?

Kubernetes hat eine eingebaute `Ingress`-Ressource:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

Das ist aber sehr limitiert:
- Nur HTTP/HTTPS, kein TCP/UDP
- Keine komplexen Routing-Regeln
- Keine Middlewares
- TLS-Konfiguration eingeschränkt

Traefiiks `IngressRoute` CRD ist mächtiger:
```yaml
# Kann das nativer Ingress nicht:
match: Host(`api.example.com`) && PathPrefix(`/v2`) && Headers(`X-API-Version`, `2`)
```

Der Nachteil: Traefik-spezifisch. Willst du zu nginx wechseln, musst du alle
IngressRoutes neu schreiben. Dafür hast du volle Kontrolle.

---

## 7. Wie cert-manager TLS-Zertifikate holt

### Das ACME-Protokoll Schritt für Schritt

ACME (Automatic Certificate Management Environment) ist das Protokoll das
cert-manager mit Let's Encrypt spricht. Ziel: Beweise dass du eine Domain kontrollierst.

```
1. cert-manager findet Certificate-Ressource:
   dnsNames: [jakob-argocd.goava.ai]
   │
   ▼
2. cert-manager kontaktiert Let's Encrypt:
   POST https://acme-v02.api.letsencrypt.org/acme/new-order
   Body: {"identifiers": [{"type":"dns","value":"jakob-argocd.goava.ai"}]}
   │
   ▼
3. Let's Encrypt antwortet:
   "OK, ich brauche einen Beweis. Lege diese Datei ab:"
   URL:     http://jakob-argocd.goava.ai/.well-known/acme-challenge/TOKEN123
   Inhalt:  TOKEN123.ACCOUNT_KEY_THUMBPRINT
   │
   ▼
4. cert-manager erstellt automatisch:
   - Einen temporären Kubernetes Ingress (oder IngressRoute)
   - Der antwortet auf /.well-known/acme-challenge/TOKEN123
   │
   ▼
5. Traefik leitet Anfragen an /.well-known/acme-challenge/* an cert-manager weiter
   │
   ▼
6. Let's Encrypt ruft http://jakob-argocd.goava.ai/.well-known/acme-challenge/TOKEN123 ab
   → Bekommt den erwarteten Inhalt
   → Domain-Kontrolle bestätigt ✓
   │
   ▼
7. cert-manager sendet Certificate Signing Request (CSR) an Let's Encrypt
   CSR enthält: Public Key + Domain-Name
   │
   ▼
8. Let's Encrypt signiert das Zertifikat und schickt es zurück
   │
   ▼
9. cert-manager speichert Zertifikat in Kubernetes Secret:
   kubectl create secret tls argocd-tls \
     --cert=tls.crt \
     --key=tls.key \
     -n argocd
   │
   ▼
10. Traefik bemerkt neues Secret "argocd-tls" → nutzt es für TLS ✓
```

### Der ClusterIssuer als Konfiguration

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    #        ↑ Let's Encrypt Produktionsserver
    email: admin@goava.ai
    #       ↑ Wird benachrichtigt wenn Zertifikat bald abläuft
    privateKeySecretRef:
      name: letsencrypt-prod
    #   ↑ ACME-Account-Key (nicht das TLS-Zertifikat!)
    #     Identifiziert uns bei Let's Encrypt
    solvers:
      - http01:
          ingress:
            class: traefik
    #         ↑ Welcher Ingress-Controller löst die Challenge?
```

Der `ClusterIssuer` ist eine Vorlage. Mehrere `Certificate`-Ressourcen können
denselben Issuer nutzen.

### Automatische Erneuerung

Let's Encrypt Zertifikate laufen nach **90 Tagen** ab. cert-manager erneuert
automatisch nach ca. 60 Tagen (30 Tage vor Ablauf):

```
Zertifikat ausgestellt: Tag 0
cert-manager prüft täglich: "Wann läuft es ab?"
  │
  ▼
Tag 60: "Noch 30 Tage — Zeit für Erneuerung"
  │ Startet ACME-Prozess erneut
  ▼
Neues Zertifikat → Secret wird aktualisiert
  │
  ▼
Traefik bemerkt Secret-Update → lädt neues Zertifikat
  │
  ▼
Kein Downtime, nahtloser Wechsel ✓
```

Ohne cert-manager müsste man das **manuell** machen — und wenn man es vergisst,
ist die Seite plötzlich mit "Not Secure" markiert oder komplett nicht erreichbar.

---

## 8. Warum jede Komponente die andere braucht

### Die Abhängigkeitskette

```
GitHub
  ↓ (liefert Konfiguration)
ArgoCD
  ↓ (deployed Ressourcen)
cert-manager
  ↓ (stellt Zertifikat aus, speichert in Secret)
Traefik
  ↓ (liest Secret, terminiert TLS, routet Traffic)
ArgoCD UI
  ↓ (erreichbar per HTTPS)
Du (Browser)
```

Das ist eine **kreisförmige Abhängigkeit** bei der Bootstrap-Frage:

> "ArgoCD managed Traefik und cert-manager.
> Traefik macht ArgoCD per HTTPS erreichbar.
> cert-manager macht TLS für ArgoCD.
> Aber ArgoCD deployed cert-manager und Traefik..."

**Wie löst man das?** Boot-Reihenfolge:

```
1. ArgoCD manuell installieren (einmalig, ohne Helm/ArgoCD)
2. root-app.yaml manuell anwenden (einmalig)
3. ArgoCD deployed Traefik (aus apps/traefik.yaml)
4. ArgoCD deployed cert-manager (aus apps/cert-manager.yaml)
5. ArgoCD deployed manifests (IngressRoute + Certificate)
6. cert-manager holt Zertifikat via Traefik HTTP-Challenge
7. Traefik nutzt Zertifikat für HTTPS
8. ArgoCD UI erreichbar per HTTPS
```

Nach Schritt 2 ist alles **self-managing**. Nur Schritt 1 und 2 sind manuell.

### Was passiert wenn man die Reihenfolge falsch macht?

**Szenario: cert-manager deployed, aber Traefik noch nicht**

cert-manager will HTTP-01 Challenge lösen:
```
Let's Encrypt → http://jakob-argocd.goava.ai/.well-known/acme-challenge/TOKEN
                     │
                     ▼
               Traefik ist noch nicht da → Connection refused / Timeout
                     │
                     ▼
          Let's Encrypt: Challenge fehlgeschlagen
                     │
                     ▼
          cert-manager: Retry in 1h
```

Das Zertifikat wird nicht ausgestellt. Aber das ist OK — cert-manager versucht
es automatisch erneut wenn Traefik dann läuft.

**Szenario: Traefik deployed, cert-manager noch nicht**

```
IngressRoute für jakob-argocd.goava.ai ist da
Secret "argocd-tls" existiert noch nicht
  │
  ▼
Traefik: "Ich brauche Secret argocd-tls für TLS... existiert nicht."
  │
  ▼
IngressRoute ist da aber TLS funktioniert nicht
  │
  ▼
Browser: SSL-Fehler
```

Auch das heilt sich selbst: Sobald cert-manager das Zertifikat ausstellt und
das Secret erstellt, bemerkt Traefik es und TLS funktioniert.

---

## 9. Was passiert wenn eine Komponente ausfällt

### ArgoCD fällt aus

```
Symptom: ArgoCD UI nicht erreichbar, git push hat keine Wirkung mehr
  │
Was passiert:
  - Bereits laufende Anwendungen (Traefik, etc.) laufen weiter
  - Keine neuen Deployments möglich
  - Kubernetes self-healing funktioniert noch
    (Pods die abstürzen werden von Kubernetes neu gestartet)
  │
Was NICHT passiert:
  - Traefik hört nicht auf zu funktionieren
  - cert-manager erneuert keine neuen Zertifikate mehr
    (aber bestehende Zertifikate laufen weiter bis zum Ablauf)
  │
Fix:
  kubectl get pods -n argocd
  kubectl delete pod argocd-server-... -n argocd  (Neustart erzwingen)
```

### Traefik fällt aus

```
Symptom: Alle Websites nicht erreichbar (503/Connection refused)
  │
Was passiert:
  - Alle HTTP/HTTPS-Anfragen von außen kommen nicht mehr an
  - Das Cluster selbst läuft weiter (ArgoCD, Pods, etc.)
  - kubectl funktioniert weiterhin (andere API)
  │
Fix:
  kubectl get pods -n traefik
  kubectl describe pod <traefik-pod> -n traefik
  kubectl rollout restart deployment traefik -n traefik
```

### cert-manager fällt aus

```
Symptom: Neue Zertifikate werden nicht ausgestellt, bestehende nicht erneuert
  │
Was passiert sofort:
  - Bestehende Zertifikate funktionieren weiter (sie sind im Secret gespeichert)
  - TLS läuft weiter bis das Zertifikat abläuft
  │
Was passiert nach 90 Tagen:
  - Zertifikat läuft ab
  - Browser zeigt "Not Secure" / Fehler
  - Wenn cert-manager bis dahin nicht repariert wurde: Website per HTTPS kaputt
  │
Fix:
  kubectl get pods -n cert-manager
  kubectl rollout restart deployment cert-manager -n cert-manager
```

### etcd fällt aus

```
Symptom: Kubernetes API-Server antwortet nicht mehr, alles bricht zusammen
  │
Was passiert:
  - Kein kubectl möglich
  - Kein ArgoCD, kein Traefik, kein cert-manager
  - Bereits laufende Container laufen noch kurz weiter
    (kubelet hat sie gestartet, braucht etcd nicht zum Laufen)
  - Aber: Keine neuen Container, keine Updates, kein Healing
  │
Das ist der schlimmste Ausfall. etcd ist Single Point of Failure
in Single-Node-Setups wie unserem.
  │
Fix in Produktion:
  etcd als Cluster mit 3 oder 5 Nodes betreiben (Quorum)
```

### Ein Node fällt aus (in Multi-Node-Cluster)

```
Symptom: Einige Pods nicht erreichbar
  │
Was Kubernetes macht:
  1. Node-Controller bemerkt: Node antwortet nicht mehr (nach 40s)
  2. Pods auf diesem Node werden als "Unknown" markiert
  3. Nach 5 Minuten: Pods werden auf anderen Nodes neu gestartet
  │
Warum dauert das so lang?
  Kubernetes wartet absichtlich — vielleicht ist der Node nur kurz überlastet
  oder hat Netzwerkprobleme. Sofortiges Verschieben wäre schlechter.
  │
In unserem Setup (Single Node):
  Wenn der einzige Node ausfällt, ist alles weg.
  → Dafür gibt es Backups und Disaster Recovery
```

---

## 10. Das vollständige Zusammenspiel

### Alle Komponenten und ihre Verbindungen

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTPS:443 / HTTP:80
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NODE (89.167.116.105)                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    KUBERNETES CLUSTER                                │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ kube-proxy (iptables)                                        │    │   │
│  │  │ :443 → traefik-svc → 10.244.0.x:8443                       │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │  Namespace: traefik                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  Traefik Pod                                                │     │   │
│  │  │  - Liest IngressRoutes (Watch auf K8s API)                 │     │   │
│  │  │  - Liest Secrets für TLS                                   │     │   │
│  │  │  - Routet Traffic                                          │     │   │
│  │  └────────────┬───────────────────────────────────────────────┘     │   │
│  │               │ routes to                                            │   │
│  │               ▼                                                      │   │
│  │  Namespace: argocd                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  argocd-server Pod                                          │     │   │
│  │  │  ← Traefik leitet HTTPS-Traffic hierhin                    │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  argocd-application-controller Pod (StatefulSet pod-0)      │     │   │
│  │  │  ← Vergleicht Git-State mit Cluster-State                  │     │   │
│  │  │  → Führt kubectl apply durch                               │     │   │
│  │  └──────────────────┬─────────────────────────────────────────┘     │   │
│  │                     │ gRPC:8081                                      │   │
│  │  ┌──────────────────▼─────────────────────────────────────────┐     │   │
│  │  │  argocd-repo-server Pod                                      │     │   │
│  │  │  ← application-controller schickt Render-Aufträge           │     │   │
│  │  │  → Klont GitHub via SSH                                     │     │   │
│  │  │  → Rendert Helm/YAML                                        │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  Secret: argocd-tls                                         │     │   │
│  │  │  ← cert-manager schreibt hier das TLS-Zertifikat rein      │     │   │
│  │  │  → Traefik liest es für HTTPS                              │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  │                                                                      │   │
│  │  Namespace: cert-manager                                             │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  cert-manager Pod                                           │     │   │
│  │  │  ← Watcht Certificate CRDs                                 │     │   │
│  │  │  → Kontaktiert Let's Encrypt                               │     │   │
│  │  │  → Erstellt temporäre Challenge-Endpoints via Traefik      │     │   │
│  │  │  → Schreibt fertiges Zertifikat in Secret                  │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  │                                                                      │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │  Kubernetes API-Server (kube-apiserver)                      │     │   │
│  │  │  ← Alle Komponenten sprechen hierhin                        │     │   │
│  │  │  → Speichert alles in etcd                                 │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
         ↕ SSH (git clone/fetch)
┌────────────────────────────────┐
│    GitHub                      │
│    Jakobulus123/K8s-Helm       │
│    ← Du pushst YAML hier       │
│    → ArgoCD liest von hier     │
└────────────────────────────────┘
         ↕ HTTPS (ACME-Protokoll)
┌────────────────────────────────┐
│    Let's Encrypt               │
│    ACME v2 API                 │
│    → Stellt TLS-Zertifikate    │
│      aus nach Validation       │
└────────────────────────────────┘
```

### Ein Tag im Leben des Systems

```
06:00 Uhr:
  cert-manager: "Zertifikat argocd-tls läuft in 28 Tagen ab → erneuern"
  cert-manager → Let's Encrypt: "Neues Zertifikat für jakob-argocd.goava.ai"
  Let's Encrypt → cert-manager: "HTTP-01 Challenge: leg TOKEN an .well-known/"
  cert-manager erstellt temporären Ingress via Traefik
  Let's Encrypt prüft → Erfolg
  cert-manager schreibt neues Zertifikat in Secret "argocd-tls"
  Traefik bemerkt Secret-Update → lädt neues Zertifikat
  Zertifikat erneuert ✓ (kein Mensch hat etwas getan)

09:00 Uhr:
  Du pushst neue Datei: apps/neue-datenbank.yaml
  ArgoCD bemerkt nach max. 3 Minuten die Änderung
  argocd-repo-server klont Repo, liest neue Application
  argocd-application-controller: "neue-datenbank existiert nicht → Sync"
  Kubernetes API-Server bekommt kubectl apply für die Datenbank
  Pod startet auf dem Node
  neue-datenbank läuft ✓ (kein Mensch hat kubectl ausgeführt)

14:00 Uhr:
  Traefik-Pod stürzt ab (hypothetisch, OutOfMemory)
  Kubernetes: "ReplicaSet sagt 1 Replica, ich sehe 0 → starte neu"
  Neuer Traefik-Pod startet
  Traefik lädt Konfiguration aus Kubernetes API
  Service läuft wieder ✓ (kein Mensch hat etwas getan, Downtime: ~10s)

17:00 Uhr:
  ArgoCD bemerkt: Jemand hat manuell kubectl edit deployment traefik gemacht
  und replicas auf 3 gesetzt
  ArgoCD: "Desired State (Git): replicas=1. Live State: replicas=3. OutOfSync!"
  selfHeal: true → ArgoCD setzt replicas zurück auf 1
  Git ist die einzige Wahrheit ✓
```

---

## Fazit: Warum dieses Setup Sinn macht

Das Setup klingt komplex — ArgoCD, Helm, Traefik, cert-manager, GitOps —
aber jede Schicht löst ein konkretes Problem:

| Problem | Lösung |
|---------|--------|
| "Wo ist dokumentiert was deployed ist?" | GitHub — alles als YAML |
| "Wer hat wann was geändert?" | Git-History |
| "Wie deploye ich ohne Fehler?" | ArgoCD — declarative, automatisch |
| "Wie kommt Traffic ans Cluster?" | Traefik — dynamisches Routing |
| "Wie manage ich HTTPS-Zertifikate?" | cert-manager — vollautomatisch |
| "Was wenn ein Pod abstürzt?" | Kubernetes — self-healing |
| "Wie skaliere ich?" | Kubernetes — replicas erhöhen |

Das Ergebnis: Ein System das sich **selbst heilt**, **selbst konfiguriert** und
**vollständig in Git dokumentiert** ist. Du musst nie wieder direkt im Cluster
"herumschrauben" — jede Änderung geht durch Git, und ArgoCD sorgt dafür dass
der Cluster dem Git-Zustand folgt.

---

*Erstellt als Ergänzung zur SETUP-DOKU.md und FEHLER-DOKU.md*
*Alle drei Dokumente zusammen bilden eine vollständige Referenz für dieses Setup.*
