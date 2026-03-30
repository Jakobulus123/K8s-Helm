# Komplette Setup-Dokumentation: Kubernetes + ArgoCD + Traefik + cert-manager

> Diese Datei liegt im `docs/` Ordner. ArgoCD überwacht nur `apps/` und `manifests/`,
> daher wird diese Datei von keiner ArgoCD-Application erfasst und nie deployed.

---

## Inhaltsverzeichnis

1. [Grundlagen: Was ist Kubernetes?](#1-grundlagen-was-ist-kubernetes)
2. [Kubernetes-Architektur im Detail](#2-kubernetes-architektur-im-detail)
3. [Wichtige Kubernetes-Konzepte und Ressourcen](#3-wichtige-kubernetes-konzepte-und-ressourcen)
4. [Netzwerk in Kubernetes](#4-netzwerk-in-kubernetes)
5. [Was ist Helm?](#5-was-ist-helm)
6. [Was ist GitOps und ArgoCD?](#6-was-ist-gitops-und-argocd)
7. [Was ist Traefik?](#7-was-ist-traefik)
8. [Was ist cert-manager?](#8-was-ist-cert-manager)
9. [Das Setup: Schritt für Schritt](#9-das-setup-schritt-für-schritt)
10. [Ordnerstruktur erklärt](#10-ordnerstruktur-erklärt)
11. [Fehlersuche und was schiefgelaufen ist](#11-fehlersuche-und-was-schiefgelaufen-ist)
12. [Wichtige kubectl-Befehle Referenz](#12-wichtige-kubectl-befehle-referenz)

---

## 1. Grundlagen: Was ist Kubernetes?

### Die Ausgangssituation: Warum braucht man Kubernetes?

Stell dir vor, du betreibst eine Webanwendung auf einem einzelnen Server. Du hast:
- Eine Datenbank
- Ein Backend
- Ein Frontend
- Einen Reverse Proxy (z.B. nginx)

Das funktioniert, hat aber Probleme:
- Fällt der Server aus, ist alles weg
- Willst du mehr Traffic abfangen, musst du manuell skalieren
- Updates erfordern Downtime
- Unterschiedliche Anwendungen teilen sich Ressourcen und können sich gegenseitig beeinflussen

**Kubernetes** (kurz: K8s — weil zwischen K und s 8 Buchstaben sind) löst diese Probleme.

### Was Kubernetes im Kern macht

Kubernetes ist ein **Container-Orchestrierungssystem**. Es verwaltet automatisch:
- **Wo** Container laufen (auf welchem Server/Node)
- **Wie viele** Instanzen einer Anwendung laufen
- **Neustart** bei Abstürzen
- **Netzwerk** zwischen Containern
- **Updates** ohne Downtime (Rolling Updates)
- **Ressourcenverteilung** (CPU, RAM)

### Container kurz erklärt

Ein Container ist wie eine "Box", die eine Anwendung mit allem enthält, was sie braucht:
- Den Anwendungscode
- Alle Bibliotheken/Dependencies
- Die Laufzeitumgebung

Das Basisformat ist **Docker**. Ein Container-Image ist die Vorlage, ein laufender Container ist die Instanz.

```
Image (Vorlage)  →  Container (laufende Instanz)
Wie eine Klasse  →  Wie ein Objekt (OOP-Analogie)
```

---

## 2. Kubernetes-Architektur im Detail

### Control Plane vs. Worker Nodes

Ein Kubernetes-Cluster besteht aus mindestens zwei Typen von Maschinen:

```
┌─────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                    │
│                                                          │
│  ┌──────────────────┐     ┌──────────────────────────┐  │
│  │   Control Plane  │     │      Worker Nodes        │  │
│  │   (k8s-master)   │     │                          │  │
│  │                  │     │  ┌────────┐  ┌────────┐  │  │
│  │  API-Server      │────►│  │ Node 1 │  │ Node 2 │  │  │
│  │  etcd            │     │  │        │  │        │  │  │
│  │  Scheduler       │     │  │ Pods   │  │ Pods   │  │  │
│  │  Controller Mgr  │     │  └────────┘  └────────┘  │  │
│  └──────────────────┘     └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Control Plane** — das "Gehirn":
- **API-Server**: Einziger Einstiegspunkt. Alle Kommunikation geht über die REST-API. `kubectl` spricht mit dem API-Server.
- **etcd**: Eine Key-Value Datenbank. Speichert den gesamten Cluster-Zustand. Wenn etcd weg ist, ist der Cluster "blind".
- **Scheduler**: Entscheidet, auf welchem Node ein Pod landet (basierend auf Ressourcen, Labels, Affinität etc.)
- **Controller Manager**: Überwacht den Cluster und sorgt dafür, dass der gewünschte Zustand (desired state) dem tatsächlichen Zustand (actual state) entspricht.

**Worker Nodes** — die "Muskeln":
- **kubelet**: Agent auf jedem Node, führt Container-Instruktionen aus
- **kube-proxy**: Netzwerk-Routing auf jedem Node (mehr dazu in Abschnitt 4)
- **Container Runtime**: Führt Container aus (containerd, CRI-O, früher Docker)

### In unserem Fall

Unser Cluster hat einen einzelnen Node, der sowohl Control Plane als auch Worker ist:
```
Server: 89.167.116.105
Node:   k8s-main
```

Das ist für Entwicklung/kleines Homelab OK, für Produktion würde man trennen.

---

## 3. Wichtige Kubernetes-Konzepte und Ressourcen

### Pod

Der kleinste deploybare Baustein in Kubernetes. Ein Pod enthält einen oder mehrere Container, die:
- Sich einen Netzwerk-Namespace teilen (gleiche IP)
- Sich Storage teilen können
- Immer zusammen deployed werden

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mein-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

**Wichtig**: Pods sind kurzlebig ("ephemeral"). Stirbt ein Pod, ist er weg. Kubernetes managed Pods nicht direkt — das macht man über Deployments.

### Deployment

Ein Deployment verwaltet einen **ReplicaSet**, der wiederum Pods verwaltet:

```
Deployment
  └── ReplicaSet
        ├── Pod 1
        ├── Pod 2
        └── Pod 3
```

Ein Deployment sorgt dafür:
- Immer X Replicas laufen
- Rolling Updates (schrittweise ersetzen)
- Rollback möglich

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3                    # Wie viele Pods
  selector:
    matchLabels:
      app: nginx                 # Welche Pods gehören dazu
  template:
    metadata:
      labels:
        app: nginx               # Label auf den Pods
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### Service

Pods haben wechselnde IPs. Ein Service gibt einem Satz von Pods eine **stabile IP und DNS-Name**.

```
Client → Service (stabile IP: 10.96.0.1) → Pod 1
                                          → Pod 2
                                          → Pod 3
```

Service-Typen:
- **ClusterIP** (Standard): Nur intern erreichbar, innerhalb des Clusters
- **NodePort**: Öffnet einen Port auf jedem Node (30000-32767)
- **LoadBalancer**: Fordert externe IP an (in Cloud-Umgebungen oder mit MetalLB on-prem)

Der Service findet "seine" Pods über **Label-Selektoren**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx        # Findet alle Pods mit diesem Label
  ports:
    - port: 80        # Port des Service
      targetPort: 80  # Port des Pods (kann auch Name sein!)
  type: ClusterIP
```

**Named Ports** — ein wichtiges Detail:
```yaml
# Im Pod/Deployment:
ports:
  - name: http        # Port bekommt einen Namen
    containerPort: 80

# Im Service:
ports:
  - targetPort: http  # Referenziert den Namen statt Nummer
```

Named Ports sind flexibler aber fehleranfälliger: Stimmt der Name nicht überein, hat der Service **keine Endpoints** und der Traffic kommt nicht an. Das war ein echtes Problem in diesem Setup (mehr dazu in Abschnitt 11).

### Namespace

Namespaces sind virtuelle Cluster innerhalb eines Clusters. Sie isolieren Ressourcen:

```
Cluster
  ├── Namespace: default       (Standard)
  ├── Namespace: kube-system   (Kubernetes-interne Komponenten)
  ├── Namespace: argocd        (ArgoCD)
  ├── Namespace: traefik       (Traefik)
  └── Namespace: cert-manager  (cert-manager)
```

Pods in verschiedenen Namespaces können miteinander kommunizieren (sofern keine NetworkPolicies das verhindern), aber mit vollqualifiziertem DNS-Namen:
```
argocd-repo-server.argocd.svc.cluster.local
              │        │    │       │
          Service   Namespace  Type  Cluster-Domain
```

### ConfigMap und Secret

**ConfigMap**: Nicht-sensible Konfigurationsdaten
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: meine-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "info"
```

**Secret**: Sensible Daten (Base64-kodiert, nicht verschlüsselt!):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mein-secret
type: Opaque
data:
  password: Q2ZpbFlOUzdweEk0OWhreg==  # base64 von "CfilYNS7pxI49hkx"
```

Base64 ist kein Schutz — es ist nur Kodierung! Für echte Verschlüsselung braucht man Sealed Secrets, External Secrets, oder Vault.

### CRD — Custom Resource Definition

Das ist ein zentrales Konzept! Kubernetes hat eingebaute Ressourcen (Pod, Service, Deployment...), aber man kann eigene **definieren**.

Eine CRD registriert einen neuen Ressourcentyp beim API-Server:

```yaml
# Dies definiert einen neuen Typ "Database"
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
```

Danach kann man "Databases" erstellen:
```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: meine-db
spec:
  engine: postgres
  version: "15"
```

**Wer nutzt CRDs?**
- **ArgoCD**: Registriert `Application`, `AppProject`, `ApplicationSet`
- **cert-manager**: Registriert `Certificate`, `ClusterIssuer`, `Issuer`, `CertificateRequest`
- **Traefik**: Registriert `IngressRoute`, `Middleware`, `TLSStore`

Ein **Operator** ist ein Controller, der auf CRDs reagiert und etwas tut. cert-manager ist ein Operator: Er beobachtet `Certificate`-Ressourcen und holt automatisch TLS-Zertifikate.

### StatefulSet vs Deployment

- **Deployment**: Für zustandslose Apps. Pods sind austauschbar.
- **StatefulSet**: Für zustandsbehaftete Apps (Datenbanken). Pods bekommen feste Namen (pod-0, pod-1) und eigene Storage.

ArgoCD's `argocd-application-controller` läuft als StatefulSet (pod-0).

### Endpoints und EndpointSlices

Wenn ein Service Pods findet (via Label-Selector), trägt Kubernetes diese in **Endpoints** ein:
```
Service argocd-repo-server
  └── Endpoints: 10.244.0.245:8081
```

Moderne Kubernetes-Versionen nutzen **EndpointSlices** (effizienter bei vielen Pods). Kube-proxy liest EndpointSlices und programmiert Netzwerkregeln (iptables/ipvs).

Wenn Endpoints **leer** sind (`<none>`), kommt kein Traffic beim Pod an — 503 oder "connection refused".

---

## 4. Netzwerk in Kubernetes

### Das Netzwerk-Modell

Kubernetes hat ein flaches Netzwerkmodell:
- **Jeder Pod bekommt eine eigene IP** (aus dem Pod-CIDR, z.B. `10.244.0.0/16`)
- Alle Pods können direkt miteinander kommunizieren (ohne NAT)
- Nodes haben ebenfalls IPs

In unserem Setup:
```
Node IP:  89.167.116.105   (öffentlich erreichbar)
Pod IPs:  10.244.0.x       (nur intern)
Svc IPs:  10.96.0.x / 10.111.x.x   (nur intern, ClusterIPs)
```

### CNI — Container Network Interface

Das Netzwerk-Plugin, das die Pod-IPs verwaltet. Bekannte CNIs:
- **Flannel**: Einfach, flaches Overlay-Netzwerk
- **Calico**: Erweitert, mit NetworkPolicies, BGP
- **Cilium**: Modern, eBPF-basiert, sehr performant

Ohne CNI können Pods nicht kommunizieren.

### kube-proxy und Service-Routing

kube-proxy läuft auf jedem Node und verwaltet **iptables-Regeln** (oder IPVS-Regeln):

```
Anfrage an Service-IP 10.111.117.63:8081
    ↓
iptables DNAT-Regel
    ↓
Pod-IP 10.244.0.245:8081
```

kube-proxy liest EndpointSlices, um zu wissen, welche Pods hinter einem Service stehen.

**Wenn EndpointSlices keine Ports haben** (`ports: null`), weiß kube-proxy nicht, welchen Pod-Port er nutzen soll → Connection refused. Das war ein Problem in diesem Setup.

### NetworkPolicy

Kubernetes hat eine "alles erlaubt"-Default-Policy. NetworkPolicies können den Traffic einschränken:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nur-argocd-darf-rein
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: argocd-application-controller
      ports:
        - port: 8081
```

ArgoCD installiert NetworkPolicies für alle seine Komponenten — nur spezifische Pods dürfen miteinander reden.

### DNS in Kubernetes

CoreDNS ist der DNS-Server im Cluster. Er löst auf:
```
argocd-server.argocd.svc.cluster.local → 10.111.178.108
```

Pods können innerhalb desselben Namespaces Kurzformen nutzen:
```
argocd-server → argocd-server.argocd.svc.cluster.local
```

### Ingress vs IngressRoute

**Ingress** ist die native Kubernetes-Ressource für HTTP(S)-Routing:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mein-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: mein-service
                port:
                  number: 80
```

Ein **Ingress Controller** (z.B. nginx-ingress, Traefik) interpretiert diese Ressourcen.

**IngressRoute** ist Traefiiks eigenes CRD — mächtiger als Standard-Ingress:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: mein-route
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: mein-service
          port: 80
  tls:
    secretName: mein-tls-cert
```

### TLS / HTTPS kurz erklärt

**TLS** (Transport Layer Security) verschlüsselt den Traffic zwischen Browser und Server.

Ablauf:
1. Browser verbindet sich mit Server
2. Server schickt **Zertifikat** (enthält Public Key + Domain-Name + Ablaufdatum)
3. Browser prüft: Ist das Zertifikat von einer vertrauenswürdigen CA signiert?
4. Beide einigen sich auf einen Session-Key (via Diffie-Hellman)
5. Ab jetzt ist alles verschlüsselt

**CA** = Certificate Authority = Organisation, die Zertifikate signiert und damit bestätigt: "Ja, diese Domain gehört diesem Server"

**Let's Encrypt** ist eine kostenlose, automatisierte CA. Sie nutzt das **ACME-Protokoll**:
1. Du beantragst ein Zertifikat für `example.com`
2. Let's Encrypt schickt eine Challenge: "Beweise, dass du example.com kontrollierst"
3. **HTTP-01 Challenge**: Platziere eine bestimmte Datei unter `http://example.com/.well-known/acme-challenge/TOKEN`
4. Let's Encrypt ruft diese URL ab und verifiziert
5. Zertifikat wird ausgestellt (gültig 90 Tage, auto-renewal möglich)

---

## 5. Was ist Helm?

### Das Problem ohne Helm

Kubernetes-Ressourcen sind YAML-Dateien. Eine echte Anwendung besteht aus:
- Deployment
- Service
- ConfigMap
- Secret
- Ingress
- ServiceAccount
- RBAC-Regeln
- ...

Das sind schnell 10-20 YAML-Dateien. Für verschiedene Umgebungen (dev/staging/prod) muss man diese Dateien kopieren und anpassen — fehleranfällig und mühsam.

### Helm = Package Manager für Kubernetes

Helm ist wie `apt` oder `npm`, aber für Kubernetes-Anwendungen.

**Helm Chart**: Ein Paket mit Templates und Default-Werten
**Values**: Konfiguration, die die Templates befüllt
**Release**: Eine installierte Instanz eines Charts

```
Chart (Template)  +  values.yaml  →  Kubernetes-Manifeste
```

### Chart-Struktur

```
mein-chart/
├── Chart.yaml          # Metadaten: Name, Version, Description
├── values.yaml         # Default-Werte
├── templates/
│   ├── deployment.yaml # Template mit {{ .Values.xxx }} Platzhaltern
│   ├── service.yaml
│   └── ingress.yaml
└── charts/             # Abhängige Charts (subcharts)
```

### Helm-Templating

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app    # Wird ersetzt mit Release-Name
spec:
  replicas: {{ .Values.replicas }}  # Aus values.yaml
  template:
    spec:
      containers:
        - image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

```yaml
# values.yaml
replicas: 2
image:
  repository: nginx
  tag: "1.25"
```

### Wichtige Helm-Befehle

```bash
# Repo hinzufügen
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

# Chart installieren
helm install mein-release traefik/traefik --namespace traefik --create-namespace

# Mit eigenen Values
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --values my-values.yaml

# Aktualisieren
helm upgrade cert-manager jetstack/cert-manager --namespace cert-manager

# Was ist installiert?
helm list -A

# Template rendern (ohne installieren)
helm template cert-manager jetstack/cert-manager --set crds.enabled=true

# Chart deinstallieren
helm uninstall cert-manager -n cert-manager
```

### Helm Repositories

Helm lädt Charts aus Repositories (wie apt-Quellen):
- `https://helm.traefik.io/traefik` — Traefik offiziell
- `https://charts.jetstack.io` — cert-manager offiziell
- `https://argoproj.github.io/argo-helm` — ArgoCD offiziell
- `https://artifacthub.io` — Öffentlicher Hub für alle Charts

---

## 6. Was ist GitOps und ArgoCD?

### Das Problem: "Wo läuft was?"

Ohne GitOps: Jemand macht `kubectl apply -f ...` oder `helm install ...` manuell. Der Cluster-Zustand lebt nur im Cluster selbst und in den Köpfen der Admins. Niemand weiß genau:
- Wann wurde was deployed?
- Wer hat die Änderung gemacht?
- Warum wurde es geändert?
- Wie sieht die Konfiguration aus?

### GitOps-Prinzip

**Git ist die einzige Quelle der Wahrheit (Single Source of Truth).**

```
Developer → Git Push → GitHub/GitLab → ArgoCD liest → Kubernetes
                                         ↑
                          Vergleicht Git-State mit Cluster-State
                          und korrigiert Abweichungen automatisch
```

Vorteile:
- **Audit-Trail**: Git-History zeigt wer wann was geändert hat
- **Rollback**: `git revert` rollt die Infrastruktur zurück
- **Review-Prozess**: Pull Requests für Infrastruktur-Änderungen
- **Self-Healing**: ArgoCD korrigiert manuelle Änderungen im Cluster

### ArgoCD-Architektur

```
┌──────────────────────────────────────────────────────┐
│                      ArgoCD                          │
│                                                       │
│  ┌─────────────────┐    ┌──────────────────────────┐ │
│  │  argocd-server  │    │ argocd-repo-server       │ │
│  │  (Web UI + API) │    │ (Klont Repos, rendert    │ │
│  └─────────────────┘    │  Helm/Kustomize)         │ │
│                         └──────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │  argocd-application-controller                   │ │
│  │  (Vergleicht Git-State mit Cluster-State,        │ │
│  │   führt Sync durch)                              │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │  argocd-dex  │  │ argocd-redis │                  │
│  │  (Auth/SSO)  │  │  (Cache)     │                  │
│  └──────────────┘  └──────────────┘                  │
└──────────────────────────────────────────────────────┘
```

**argocd-repo-server**: Klont das Git-Repository, rendert Helm-Charts und Kustomize-Manifeste zu fertigen Kubernetes-YAML. Läuft auf Port 8081.

**argocd-application-controller**: Der Kern. Er vergleicht kontinuierlich:
- **Desired State**: Was steht im Git-Repo (gerendert vom repo-server)
- **Live State**: Was läuft im Cluster

Wenn es Abweichungen gibt → `OutOfSync`.

### ArgoCD Application Ressource

Eine `Application` ist ein ArgoCD CRD. Sie definiert:
- Wo der Source-Code liegt (Git-Repo, Pfad, Branch)
- Wohin deployed wird (Cluster, Namespace)
- Wie deployed wird (Helm, Kustomize, plain YAML)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: meine-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:user/repo.git
    targetRevision: HEAD          # Branch, Tag, oder Commit-Hash
    path: apps/meine-app          # Pfad im Repo
  destination:
    server: https://kubernetes.default.svc   # Dieser Cluster
    namespace: meine-app
  syncPolicy:
    automated:
      prune: true       # Lösche Ressourcen die im Git nicht mehr sind
      selfHeal: true    # Korrigiere manuelle Änderungen im Cluster
    syncOptions:
      - CreateNamespace=true   # Erstelle Namespace wenn nicht vorhanden
```

### App of Apps Pattern

Das mächtige Muster in diesem Setup: Eine ArgoCD-Application verwaltet andere ArgoCD-Applications.

```
bootstrap/root-app.yaml
  │ (einmalig manuell applied)
  ▼
ArgoCD Application "root"
  │ (beobachtet apps/ Ordner im Git)
  ▼
  ├── apps/traefik.yaml     → Application "traefik"
  ├── apps/cert-manager.yaml → Application "cert-manager"
  └── apps/manifests.yaml   → Application "manifests"
```

**Vorteil**: Man muss nur die `root` Application einmal manuell anlegen. Danach verwaltet ArgoCD sich selbst — füge eine neue YAML in `apps/` hinzu und ArgoCD deployed automatisch.

### Multi-Source Applications

ArgoCD unterstützt mehrere Sources für eine Application. Das nutzen wir für Helm Charts mit externen Values:

```yaml
spec:
  sources:
    # Source 1: Das Helm Chart (extern, von Helm Repo)
    - repoURL: https://helm.traefik.io/traefik
      chart: traefik
      targetRevision: "33.x"
      helm:
        valueFiles:
          - $values/values/traefik/values.yaml  # Referenz auf Source 2

    # Source 2: Unser Git Repo (mit den Values-Dateien)
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values    # Gibt dieser Source den Namen "$values"
```

**Warum?** Wir wollen den Helm Chart *unverändert* nutzen, aber *unsere eigenen Values* aus unserem Git-Repo verwenden. Das trennt "was deployen wir" von "wie konfigurieren wir es".

### Sync-Status Erklärung

| Status | Bedeutung |
|--------|-----------|
| `Synced` | Git-State = Cluster-State. Alles gut. |
| `OutOfSync` | Unterschied zwischen Git und Cluster. |
| `Unknown` | ArgoCD kann nicht vergleichen (oft Verbindungsproblem zum repo-server). |

| Health | Bedeutung |
|--------|-----------|
| `Healthy` | Alle Ressourcen laufen korrekt. |
| `Progressing` | Gerade am Deployen (Pods starten). |
| `Degraded` | Etwas läuft nicht wie erwartet. |
| `Missing` | Ressource existiert nicht im Cluster. |

### ArgoCD SSH-Authentifizierung

ArgoCD klont Git-Repos. Für private Repos (SSH):

```bash
# ArgoCD Secret für SSH-Key
kubectl create secret generic repo-jakob \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:Jakobulus123/K8s-Helm.git \
  --from-file=sshPrivateKey=/path/to/id_rsa
kubectl label secret repo-jakob argocd.argoproj.io/secret-type=repository -n argocd
```

In unserem Fall war dieser Secret (`repo-727441422`) bereits vorhanden.

---

## 7. Was ist Traefik?

### Reverse Proxy & Ingress Controller

Traefik ist ein moderner **Reverse Proxy** und **Load Balancer**, der auch als Kubernetes **Ingress Controller** fungiert.

```
Internet
    │
    ▼
┌──────────┐
│  Traefik │  ← Lauscht auf Port 80 (HTTP) und 443 (HTTPS)
│          │
│ Routing- │  Host(`app.example.com`) → Service app-service:80
│  Regeln  │  Host(`api.example.com`) → Service api-service:8080
└──────────┘
    │             │
    ▼             ▼
 Service A    Service B
    │             │
    ▼             ▼
 Pod A-1      Pod B-1
 Pod A-2      Pod B-2
```

### Traefik EntryPoints

EntryPoints sind die "Türen", an denen Traefik lauscht:
- **web**: Port 80 (HTTP)
- **websecure**: Port 443 (HTTPS)
- **metrics**: Port 9100 (Prometheus-Metriken)

### IngressRoute (Traefik CRD)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure           # Nur HTTPS
  routes:
    - match: Host(`jakob-argocd.goava.ai`)   # Domain-Regel
      kind: Rule
      services:
        - name: argocd-server    # Kubernetes Service
          port: 80
      middlewares:
        - name: argocd-headers   # Zusätzliche Verarbeitung
  tls:
    secretName: argocd-tls      # TLS-Zertifikat aus Secret
```

**Rule-Syntax**:
```
Host(`example.com`)                    # Exakte Domain
Host(`api.example.com`) && PathPrefix(`/v2`)  # Domain + Pfad
HeadersRegexp(`User-Agent`, `.*bot.*`)  # Header-Match
```

### Middleware

Middlewares transformieren Requests/Responses:

```yaml
# HTTP → HTTPS Redirect
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true    # 301 statt 302

# Custom Headers setzen
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-headers
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: https   # ArgoCD braucht diesen Header
```

**Warum `X-Forwarded-Proto: https`?** ArgoCD erkennt, ob es hinter einem Proxy läuft, anhand dieses Headers. Ohne ihn würde ArgoCD selbst auf HTTPS redirecten → Endlosschleife.

### Traefik API-Gruppen: traefik.io vs traefik.containo.us

Traefik hat historisch zwei API-Gruppen:
- `traefik.containo.us` — Alte API (v2.x)
- `traefik.io` — Neue API (v3.x, ab Traefik 2.10)

Beide CRDs können im Cluster existieren. Wichtig: `kubectl get ingressroute` findet keine Ressourcen, weil es im Cluster beide Gruppen gibt und kubectl die falsche erwischt. Richtig:
```bash
kubectl get ingressroute.traefik.io -n argocd
# ODER
kubectl get ingressroute.traefik.containo.us -n argocd
```

---

## 8. Was ist cert-manager?

### Das Problem: TLS-Zertifikate manuell verwalten

Ohne cert-manager:
1. Manuell Zertifikat bei Let's Encrypt anfordern
2. In Kubernetes Secret ablegen
3. Nach 90 Tagen erneuern
4. Wenn du's vergisst → HTTPS kaputt

### cert-manager als Operator

cert-manager ist ein Kubernetes-Operator. Er beobachtet `Certificate`-Ressourcen und handelt automatisch:

```
Certificate CRD angelegt
    ↓
cert-manager erkennt es
    ↓
HTTP-01 Challenge mit Let's Encrypt
    ↓
Zertifikat wird ausgestellt
    ↓
In Kubernetes Secret gespeichert
    ↓
Traefik nutzt das Secret für TLS
    ↓
Automatische Erneuerung nach ~60 Tagen
```

### ClusterIssuer vs Issuer

**Issuer**: Stellt Zertifikate nur in einem Namespace aus
**ClusterIssuer**: Cluster-weit, stellt in jedem Namespace Zertifikate aus

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@goava.ai           # Let's Encrypt schickt Ablauf-Warnungen hierhin
    privateKeySecretRef:
      name: letsencrypt-prod        # Speichert den ACME-Account-Key
    solvers:
      - http01:
          ingress:
            class: traefik          # Traefik löst die HTTP-01 Challenge
```

**Staging vs Production**:
```
https://acme-v02.api.letsencrypt.org/directory    ← Produktion (echte Zertifikate)
https://acme-staging-v02.api.letsencrypt.org/...  ← Staging (Fake-Zerts zum Testen)
```

Let's Encrypt hat Rate-Limits (5 Zertifikate pro Domain pro Woche). Beim Testen immer Staging nutzen!

### Certificate Ressource

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd
spec:
  secretName: argocd-tls           # Wohin das Zertifikat gespeichert wird
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - jakob-argocd.goava.ai       # Für welche Domain(s)
```

Nach erfolgreicher Ausstellung:
```bash
kubectl get certificate argocd-tls -n argocd
# NAME         READY   SECRET       AGE
# argocd-tls   True    argocd-tls   5m
```

`READY: True` = Zertifikat ist gültig und im Secret gespeichert.

### CRD-Drift Problem mit ArgoCD

cert-manager CRDs sind sehr groß (riesige OpenAPI-Schemas für Validierung). Das führt zu einem bekannten Problem:

**Was passiert:**
1. Helm rendert CRD mit `spec.versions[0].selectableFields: []`
2. ArgoCD applied das in den Cluster
3. Kubernetes streicht `selectableFields` (Feature zu neu, Cluster-Version zu alt)
4. ArgoCD vergleicht: Helm will `selectableFields`, Cluster hat es nicht → `OutOfSync`
5. ArgoCD applied nochmal → Kubernetes streicht wieder → Endlosschleife

**Lösung**: CRDs nicht über Helm verwalten:
```yaml
# values/cert-manager/values.yaml
crds:
  enabled: false   # ArgoCD soll CRDs nicht anfassen
```

Die CRDs sind bereits im Cluster und funktionieren. Wir brauchen ArgoCD nicht, um sie zu verwalten.

---

## 9. Das Setup: Schritt für Schritt

### Voraussetzungen

- Kubernetes-Cluster läuft (`kubectl cluster-info` funktioniert)
- Git-Repository auf GitHub vorhanden
- SSH-Key für GitHub konfiguriert
- ArgoCD installiert (in unserem Fall bereits vorhanden)

### Schritt 1: ArgoCD installieren (falls nicht vorhanden)

```bash
# Namespace erstellen
kubectl create namespace argocd

# Manifeste anwenden (latest stable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Warten bis alles läuft
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Initial-Passwort auslesen
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### Schritt 2: Git-Repository struktur anlegen

```
K8s/
├── bootstrap/
│   └── root-app.yaml          # Einmalig anwenden → startet alles
├── apps/
│   ├── traefik.yaml           # ArgoCD App für Traefik
│   ├── cert-manager.yaml      # ArgoCD App für cert-manager
│   └── manifests.yaml         # ArgoCD App für eigene Manifeste
├── values/
│   ├── traefik/
│   │   └── values.yaml        # Helm Values für Traefik
│   └── cert-manager/
│       └── values.yaml        # Helm Values für cert-manager
├── manifests/
│   ├── argocd/
│   │   └── ingress.yaml       # IngressRoute + Certificate für ArgoCD UI
│   └── cert-manager/
│       └── letsencrypt-clusterissuer.yaml
└── docs/
    └── SETUP-DOKU.md          # Diese Datei (wird von ArgoCD ignoriert)
```

**Warum wird `docs/` ignoriert?** ArgoCD überwacht nur explizit konfigurierte Pfade. Die `root` App überwacht `apps/`, die `manifests` App überwacht `manifests/`. Der `docs/` Ordner liegt außerhalb.

### Schritt 3: Root Application erstellen

```yaml
# bootstrap/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    path: apps              # Überwacht den apps/ Ordner
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# Einmalig anwenden — danach synct ArgoCD alles selbst
kubectl apply -f bootstrap/root-app.yaml
```

### Schritt 4: Traefik Application

```yaml
# apps/traefik.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    # Quelle 1: Helm Chart vom offiziellen Traefik Repo
    - repoURL: https://helm.traefik.io/traefik
      chart: traefik
      targetRevision: "33.x"
      helm:
        releaseName: traefik
        valueFiles:
          - $values/values/traefik/values.yaml
    # Quelle 2: Unser Git Repo mit den Values
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: traefik
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Schritt 5: cert-manager Application

```yaml
# apps/cert-manager.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://charts.jetstack.io
      chart: cert-manager
      targetRevision: "v1.20.0"
      helm:
        releaseName: cert-manager
        valueFiles:
          - $values/values/cert-manager/values.yaml
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
  ignoreDifferences:
    - group: apiextensions.k8s.io
      kind: CustomResourceDefinition
      jqPathExpressions:
        - .spec.conversion
        - .spec.preserveUnknownFields
        - .spec.versions[].selectableFields
        - .status
```

**`ServerSideApply=true`**: Nutzt Server-Side Apply statt Client-Side Apply. Unterschied:
- Client-Side: ArgoCD berechnet Diff lokal, schickt fertiges YAML
- Server-Side: ArgoCD schickt "gewünschte Felder", Kubernetes mergt selbst (besser bei großen CRDs)

**`RespectIgnoreDifferences=true`**: Die ignorierten Felder werden auch beim Sync nicht angefasst.

### Schritt 6: ArgoCD per HTTPS erreichbar machen

```yaml
# manifests/argocd/ingress.yaml

# TLS-Zertifikat anfordern
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd
spec:
  secretName: argocd-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - jakob-argocd.goava.ai

---
# HTTPS Route
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`jakob-argocd.goava.ai`)
      kind: Rule
      services:
        - name: argocd-server
          port: 80
      middlewares:
        - name: argocd-headers
  tls:
    secretName: argocd-tls

---
# HTTP → HTTPS Redirect
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-http
  namespace: argocd
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`jakob-argocd.goava.ai`)
      kind: Rule
      middlewares:
        - name: redirect-https
      services:
        - name: argocd-server
          port: 80

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-headers
  namespace: argocd
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: https
```

### Schritt 7: Git Push und Bootstrap

```bash
# Alle Dateien committen und pushen
git add .
git commit -m "Initial ArgoCD GitOps Setup"
git push origin main

# Root App einmalig anwenden
kubectl apply -f bootstrap/root-app.yaml

# Status beobachten
kubectl get applications -n argocd -w
```

---

## 10. Ordnerstruktur erklärt

```
K8s/
│
├── bootstrap/                    ← EINMALIG MANUELL ANWENDEN
│   └── root-app.yaml             Startet den gesamten GitOps-Prozess
│                                 Danach verwaltet ArgoCD alles selbst
│
├── apps/                         ← VON "ROOT" APP ÜBERWACHT
│   ├── traefik.yaml              ArgoCD Application für Traefik
│   ├── cert-manager.yaml         ArgoCD Application für cert-manager
│   └── manifests.yaml            ArgoCD Application für eigene Ressourcen
│
│   ↑ Neue App hinzufügen = neue YAML hier ablegen = ArgoCD deployed automatisch
│
├── values/                       ← HELM VALUES (referenziert von apps/)
│   ├── traefik/
│   │   └── values.yaml           Konfiguration für Traefik
│   └── cert-manager/
│       └── values.yaml           Konfiguration für cert-manager
│
├── manifests/                    ← VON "MANIFESTS" APP ÜBERWACHT
│   ├── argocd/
│   │   └── ingress.yaml          IngressRoute, Certificate, Middlewares für ArgoCD UI
│   └── cert-manager/
│       └── letsencrypt-clusterissuer.yaml
│
│   ↑ Neue K8s Ressourcen hier ablegen = ArgoCD deployed automatisch
│
└── docs/                         ← WIRD VON ARGO IGNORIERT
    └── SETUP-DOKU.md             Diese Datei
```

**Warum diese Struktur?**

- `bootstrap/` — Außerhalb aller Apps, nur Startpunkt
- `apps/` — Nur ArgoCD Application CRDs, keine eigentlichen Workloads
- `values/` — Helm Konfiguration, von Multi-Source Apps referenziert
- `manifests/` — Alle eigenen Kubernetes Ressourcen (nicht Helm)
- `docs/` — Dokumentation, nicht von ArgoCD überwacht

---

## 11. Fehlersuche und was schiefgelaufen ist

Das sind die tatsächlichen Probleme die während dieses Setups aufgetreten sind — sehr lehrreich!

### Problem 1: ArgoCD Application-Controller kann repo-server nicht erreichen

**Symptom:**
```
Failed to load target state: failed to generate manifest for source 1 of 1:
rpc error: code = Unavailable desc = connection error:
dial tcp 10.111.117.63:8081: connect: connection refused
```

**Diagnose:**
```bash
# Service hat keine Endpoints!
kubectl get endpoints argocd-repo-server -n argocd
# NAME                 ENDPOINTS   AGE
# argocd-repo-server   <none>      2d20h

# EndpointSlice zeigt zwar eine IP, aber KEINE Ports!
kubectl get endpointslice -n argocd | grep repo
# argocd-repo-server-rhbp9   IPv4   <unset>   10.244.0.243   2d20h
#                                    ↑ Das ist das Problem!

# Service nutzt Named Port:
kubectl get svc argocd-repo-server -n argocd -o jsonpath='{.spec.ports}'
# targetPort: "repo-server"  ← Named Port

# Pod hat KEINEN Namen für den Port:
kubectl get pod argocd-repo-server-... -o jsonpath='{.spec.containers[0].ports}'
# [{"containerPort":8081,"protocol":"TCP"}]  ← Kein Name!
```

**Ursache:** Service referenziert Named Port `repo-server`, aber Container-Ports haben keinen Namen. Kubernetes kann den Named Port nicht auflösen → EndpointSlice mit `ports: null`.

**Fix:**
```bash
kubectl patch deployment argocd-repo-server -n argocd --type=json \
  -p='[
    {"op":"replace","path":"/spec/template/spec/containers/0/ports/0",
     "value":{"containerPort":8081,"name":"repo-server","protocol":"TCP"}},
    {"op":"replace","path":"/spec/template/spec/containers/0/ports/1",
     "value":{"containerPort":8084,"name":"metrics","protocol":"TCP"}}
  ]'
```

Danach: `ports: null` → `ports: [{port: 8081}]` im EndpointSlice.

### Problem 2: ArgoCD UI 503 "no available server"

**Symptom:** Traefik antwortet mit HTTP 503 für `https://jakob-argocd.goava.ai`

**Diagnose:**
```bash
# argocd-server Service hat auch keine Endpoints!
kubectl get endpoints argocd-server -n argocd
# NAME            ENDPOINTS   AGE
# argocd-server   <none>      2d21h

# Vergleich Service-Selector vs Pod-Labels:
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.selector}'
# {"app.kubernetes.io/instance":"argocd","app.kubernetes.io/name":"argocd-server"}
#  ↑ Benötigt BEIDE Labels

kubectl get pod argocd-server-... -n argocd -o jsonpath='{.metadata.labels}'
# {"app.kubernetes.io/name":"argocd-server","pod-template-hash":"..."}
#  ↑ Fehlt: app.kubernetes.io/instance: argocd
```

**Ursache:** ArgoCD wurde mit Helm installiert. Der Service-Selector erwartet `app.kubernetes.io/instance: argocd`, aber die Pods haben dieses Label nicht. Wahrscheinlich war das Helm-Chart beim initialen Install anders konfiguriert oder Pods wurden ersetzt.

**Fix für alle betroffenen ArgoCD-Deployments:**
```bash
for deploy in argocd-server argocd-repo-server argocd-applicationset-controller \
              argocd-notifications-controller argocd-dex-server; do
  kubectl patch deployment $deploy -n argocd --type=json \
    -p='[{"op":"add",
          "path":"/spec/template/metadata/labels/app.kubernetes.io~1instance",
          "value":"argocd"}]'
done
```

**Wichtig:** In JSON Patch muss `/` in Schlüsseln als `~1` kodiert werden.
Also `app.kubernetes.io/instance` → `app.kubernetes.io~1instance`

### Problem 3: cert-manager OutOfSync wegen CRD-Drift

**Symptom:**
```bash
kubectl get application cert-manager -n argocd
# NAME           SYNC STATUS   HEALTH STATUS
# cert-manager   OutOfSync     Healthy

# Nur CRDs sind OutOfSync:
# CustomResourceDefinition certificates.cert-manager.io - OutOfSync
# CustomResourceDefinition challenges.acme.cert-manager.io - OutOfSync
```

**Diagnose:**
```bash
# Was rendert Helm?
helm template cert-manager jetstack/cert-manager --version v1.20.0 \
  --set crds.enabled=true | \
  grep -A5 "selectableFields"
# selectableFields: []   ← Helm will das

# Was hat der Cluster?
kubectl get crd certificates.cert-manager.io -o jsonpath='{.spec.versions[0]}' \
  | python3 -m json.tool | grep selectableFields
# (nichts) ← Cluster hat es nicht!
```

**Ursache:** Chart v1.20.0 fügt `selectableFields` zu CRDs hinzu. Das ist ein Kubernetes-Feature ab v1.30+. Der Cluster hat eine ältere Kubernetes-Version → Kubernetes streicht das Feld beim Apply. ArgoCD sieht ständig eine Differenz.

**Fix-Versuche in Reihenfolge:**
1. `ServerSideApply=true` hinzugefügt → kein Effekt
2. `ignoreDifferences` mit `jsonPointers: ["/spec/conversion"]` → kein Effekt
3. `ignoreDifferences` mit `jqPathExpressions` → kein Effekt
4. **Finale Lösung**: `crds.enabled: false` in values.yaml

**Lesson learned:** CRDs von cert-manager und anderen großen Helm Charts sollte man **nicht von Helm verwalten lassen**, wenn man ArgoCD nutzt. CRDs separat installieren (z.B. via `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.0/cert-manager.crds.yaml`) und dann in den Chart-Values deaktivieren.

---

## 12. Wichtige kubectl-Befehle Referenz

### Cluster-Überblick

```bash
# Cluster-Info
kubectl cluster-info

# Alle Nodes
kubectl get nodes -o wide

# Alle Pods in allen Namespaces
kubectl get pods -A

# Ressourcen-Verbrauch
kubectl top nodes
kubectl top pods -A
```

### Debugging

```bash
# Pod-Logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # Vorheriger Container-Run
kubectl logs <pod-name> -n <namespace> -f           # Follow (live)

# Pod beschreiben (Events sind wichtig!)
kubectl describe pod <pod-name> -n <namespace>

# In Pod einsteigen (falls Shell vorhanden)
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Service und Endpoints
kubectl get endpoints <service-name> -n <namespace>
kubectl get endpointslice -n <namespace>

# Events im Namespace
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Ressourcen verwalten

```bash
# Ressource anwenden (erstellen oder updaten)
kubectl apply -f manifest.yaml

# Ressource löschen
kubectl delete -f manifest.yaml
kubectl delete pod <pod-name> -n <namespace>

# Ressource patchen (JSON Patch)
kubectl patch deployment <name> -n <namespace> --type=json \
  -p='[{"op":"add","path":"/spec/replicas","value":3}]'

# Deployment neu starten (rollingUpdate)
kubectl rollout restart deployment <name> -n <namespace>

# Rollout-Status beobachten
kubectl rollout status deployment <name> -n <namespace>
```

### Labels und Selektoren

```bash
# Pods mit bestimmtem Label
kubectl get pods -l app=nginx -n default

# Label zu Pod hinzufügen
kubectl label pod <pod-name> environment=prod -n <namespace>

# Label entfernen
kubectl label pod <pod-name> environment- -n <namespace>
```

### ArgoCD spezifisch

```bash
# Alle ArgoCD Applications
kubectl get applications -n argocd

# Application Details
kubectl describe application <name> -n argocd

# Hard Refresh erzwingen (neu vom Git holen)
kubectl annotate application <name> -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite

# Manueller Sync
kubectl patch application <name> -n argocd --type=merge \
  -p '{"operation":{"sync":{}}}'

# ArgoCD Admin Passwort auslesen
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### Netzwerk-Debugging

```bash
# Service-Konfiguration
kubectl get svc <name> -n <namespace> -o yaml

# Endpoints prüfen
kubectl get endpoints <service-name> -n <namespace>

# DNS auflösen (von einem Pod aus)
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup argocd-server.argocd.svc.cluster.local

# HTTP-Request testen (von einem Pod aus)
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://argocd-server.argocd.svc.cluster.local

# NetworkPolicies
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
```

---

## Abschluss: Der gesamte Datenfluss

Wenn du jetzt `https://jakob-argocd.goava.ai` aufrufst, passiert folgendes:

```
Browser
  │ DNS-Lookup: jakob-argocd.goava.ai → 89.167.116.105
  ▼
89.167.116.105:443 (Traefik LoadBalancer Service)
  │ Kubernetes NodePort: 443 → Traefik Pod
  ▼
Traefik Pod
  │ TLS Terminierung mit Zertifikat aus Secret argocd-tls
  │ SNI: jakob-argocd.goava.ai → IngressRoute "argocd"
  │ Middleware "argocd-headers": X-Forwarded-Proto: https hinzufügen
  ▼
argocd-server Service (ClusterIP 10.111.178.108:80)
  │ kube-proxy: DNAT 10.111.178.108:80 → Pod-IP:8080
  ▼
argocd-server Pod (Port 8080)
  │ ArgoCD UI wird gerendert
  ▼
Browser zeigt ArgoCD UI ✓
```

Und wenn du eine YAML-Datei in `apps/` pushst:

```
git push origin main
  │
  ▼
GitHub Repository
  │ ArgoCD pollt alle 3 Minuten (oder via Webhook sofort)
  ▼
argocd-repo-server
  │ Klont Repo via SSH
  │ Rendert YAML / Helm-Templates
  ▼
argocd-application-controller
  │ Vergleicht: Was will Git? Was hat der Cluster?
  │ Unterschied gefunden → Sync
  ▼
kubectl apply (intern)
  │
  ▼
Kubernetes API-Server
  │ Validiert, speichert in etcd
  ▼
Kubelet auf Node
  │ Startet Container
  ▼
Neue Ressource läuft ✓
```

---

*Erstellt während des initialen K8s + ArgoCD + Traefik + cert-manager Setups auf dem Cluster 89.167.116.105*
