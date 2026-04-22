# Kubernetes Konzepte — Objekte, Prozesse und Netzwerk verstehen

> Diese Dokumentation erklärt Kubernetes von Grund auf — alle Objekte, alle Prozesse,
> alle Netzwerkkonzepte und wie alles miteinander zusammenläuft.
> Ziel: Du sollst nicht nur wissen WAS du tippst, sondern WARUM es funktioniert.

---

## Inhaltsverzeichnis

1. [Was ist Kubernetes überhaupt?](#1-was-ist-kubernetes-überhaupt)
2. [Die Control Plane — Das Gehirn](#2-die-control-plane--das-gehirn)
3. [Die Worker Nodes — Die Muskeln](#3-die-worker-nodes--die-muskeln)
4. [Workload-Objekte — Was läuft im Cluster](#4-workload-objekte--was-läuft-im-cluster)
5. [Netzwerk in Kubernetes — Vollständig erklärt](#5-netzwerk-in-kubernetes--vollständig-erklärt)
6. [Services — Wie Traffic zu Pods kommt](#6-services--wie-traffic-zu-pods-kommt)
7. [Ingress & IngressRoute — Traffic von außen](#7-ingress--ingressroute--traffic-von-außen)
8. [Konfiguration & Secrets](#8-konfiguration--secrets)
9. [Storage — Persistenter Speicher](#9-storage--persistenter-speicher)
10. [RBAC — Wer darf was](#10-rbac--wer-darf-was)
11. [ArgoCD — GitOps erklärt](#11-argocd--gitops-erklärt)
12. [Traefik — Der Ingress Controller](#12-traefik--der-ingress-controller)
13. [cert-manager — Automatische TLS-Zertifikate](#13-cert-manager--automatische-tls-zertifikate)
14. [Die komplette Prozesskette](#14-die-komplette-prozesskette)
15. [Schnellreferenz & Befehle](#15-schnellreferenz--befehle)

---

## 1. Was ist Kubernetes überhaupt?

Kubernetes (kurz: **k8s**) ist ein **Container-Orchestrierungssystem**.

### Das Problem das Kubernetes löst

Stell dir vor du hast eine Anwendung die in einem Docker-Container läuft.
Was passiert wenn:
- Der Container abstürzt?
- Du 10x mehr Traffic bekommst?
- Du ein Update einspielst ohne Downtime?
- Du 20 verschiedene Anwendungen verwalten musst?

Manuell ist das nicht zu managen. **Kubernetes übernimmt genau diese Aufgaben automatisch.**

### Das Grundprinzip: Desired State

Kubernetes arbeitet nach dem Prinzip des **Desired State** (gewünschter Zustand):

```
Du sagst:  "Ich will 3 Pods von Anwendung X laufen haben"
K8s macht: Stellt sicher dass IMMER 3 Pods laufen — egal was passiert
           Fällt einer aus → startet sofort einen neuen
           Node geht down → verschiebt Pods auf andere Node
```

Du beschreibst **was** du willst (in YAML), Kubernetes kümmert sich um das **wie**.

```yaml
# Beispiel: Ich will 3 Pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app
spec:
  replicas: 3        # ← Das ist der Desired State
  selector:
    matchLabels:
      app: meine-app
  template:
    spec:
      containers:
      - name: meine-app
        image: nginx:1.25
```

### Der Reconciliation Loop

Das Herzstück von Kubernetes ist eine Endlosschleife:

```
┌─────────────────────────────────────────┐
│           Reconciliation Loop           │
│                                         │
│  1. Was ist der gewünschte Zustand?     │
│     (aus etcd lesen)                    │
│           ↓                             │
│  2. Was ist der aktuelle Zustand?       │
│     (Cluster beobachten)                │
│           ↓                             │
│  3. Gibt es einen Unterschied?          │
│     Ja → Aktionen ausführen             │
│     Nein → Warten                       │
│           ↓                             │
│  4. Zurück zu Schritt 1                 │
└─────────────────────────────────────────┘
```

Diese Loop läuft für **jedes Kubernetes-Objekt** — Deployments, Services, Pods, alles.

---

## 2. Die Control Plane — Das Gehirn

Die Control Plane ist die "Verwaltungszentrale" des Clusters. Sie läuft normalerweise
auf einem oder mehreren **Master Nodes**.

```
┌──────────────────────────────────────────────────────┐
│                    Control Plane                      │
│                                                      │
│  ┌─────────────┐  ┌──────┐  ┌──────────────────┐   │
│  │ kube-api-   │  │ etcd │  │ kube-controller- │   │
│  │ server      │  │      │  │ manager          │   │
│  └─────────────┘  └──────┘  └──────────────────┘   │
│                                                      │
│  ┌─────────────────┐                                │
│  │ kube-scheduler  │                                │
│  └─────────────────┘                                │
└──────────────────────────────────────────────────────┘
```

### kube-apiserver

Der **einzige Eintrittspunkt** in den Cluster. Jedes kubectl-Kommando, jede interne
Kommunikation läuft über den API-Server.

```
kubectl apply -f deployment.yaml
      ↓
kube-apiserver (REST API auf Port 6443)
      ↓
Validierung (Ist das YAML gültig? Hat der User die Rechte?)
      ↓
Speichern in etcd
      ↓
Benachrichtigung an Controller Manager + Scheduler
```

**Wichtig:** Der API-Server speichert selbst nichts — er ist nur die Schnittstelle.
Alles landet in **etcd**.

### etcd

**etcd** ist eine verteilte Key-Value-Datenbank — das "Gedächtnis" des Clusters.

```
/registry/deployments/default/meine-app  →  { komplettes Deployment-Objekt als JSON }
/registry/pods/default/meine-app-abc123  →  { komplettes Pod-Objekt als JSON }
/registry/services/default/meine-app     →  { komplettes Service-Objekt als JSON }
```

Wenn etcd stirbt, verliert Kubernetes seinen kompletten Zustand.
Deshalb: **etcd immer backupen!**

etcd ist ein **Raft-Konsensus-System** — bei 3 Nodes muss die Mehrheit (2) einig sein
bevor eine Änderung gespeichert wird. Das verhindert Split-Brain.

### kube-scheduler

Der Scheduler entscheidet **auf welcher Node** ein neuer Pod läuft.

```
Neuer Pod ohne zugewiesene Node
        ↓
Scheduler analysiert alle verfügbaren Nodes:
  ├── Hat die Node genug CPU?        (Resource Requests)
  ├── Hat die Node genug RAM?
  ├── Passt die Node zu Taints/Tolerations?
  ├── Passt die Node zu Node Selectors/Affinity?
  └── Ist die Node gesund?
        ↓
Scoring: Welche Node ist am besten geeignet?
        ↓
Pod wird der Node zugewiesen (Binding)
        ↓
kubelet auf der Node übernimmt
```

### kube-controller-manager

Führt viele **Controller** in einem Prozess aus. Jeder Controller ist eine
Reconciliation Loop für einen bestimmten Ressourcentyp:

| Controller | Aufgabe |
|------------|---------|
| **Deployment Controller** | Erstellt/updated ReplicaSets wenn Deployment sich ändert |
| **ReplicaSet Controller** | Stellt sicher dass N Pods laufen |
| **Node Controller** | Erkennt wenn Nodes ausfallen, markiert Pods als terminated |
| **Service Controller** | Erstellt LoadBalancer bei Cloud-Providern |
| **Endpoint Controller** | Updated Endpoints-Objekte wenn Pods starten/stoppen |

---

## 3. Die Worker Nodes — Die Muskeln

Auf jeder Worker Node laufen 3 Prozesse:

```
┌─────────────────────────────────────────┐
│              Worker Node                │
│                                         │
│  ┌─────────┐  ┌───────────┐  ┌───────┐ │
│  │ kubelet │  │ kube-proxy│  │contai-│ │
│  │         │  │           │  │nerd   │ │
│  └─────────┘  └───────────┘  └───────┘ │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Pod  │  Pod  │  Pod  │  Pod   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### kubelet

Das kubelet ist der **Agent** auf jeder Node — es ist die direkte Verbindung zwischen
Control Plane und Node.

**Was kubelet macht:**
1. Registriert die Node beim API-Server ("Ich bin da, ich habe X CPU und Y RAM")
2. Beobachtet den API-Server: "Welche Pods soll ICH auf DIESER Node ausführen?"
3. Startet Container über die Container Runtime
4. Überwacht laufende Pods (Health Checks)
5. Meldet Status zurück an den API-Server

```
kube-apiserver: "Node worker-1 soll Pod nginx-abc123 ausführen"
      ↓
kubelet auf worker-1 empfängt diese Anweisung
      ↓
kubelet ruft containerd auf: "Starte Container nginx:1.25"
      ↓
containerd pulled das Image (falls nicht lokal)
      ↓
Container läuft
      ↓
kubelet meldet: "Pod nginx-abc123 ist Running" → zurück an API-Server
```

### kube-proxy

kube-proxy läuft auf jeder Node und ist für das **Service-Networking** zuständig.

**Das Problem das kube-proxy löst:**
- Ein Service hat eine virtuelle IP (ClusterIP) — z.B. `10.96.0.1`
- Hinter dieser IP stecken echte Pod-IPs — z.B. `10.244.1.5`, `10.244.2.3`
- Wenn ein Request an `10.96.0.1` kommt, muss er zu einer der Pod-IPs gehen

kube-proxy löst das mit **iptables-Regeln**:

```
Eingehender Traffic an 10.96.0.1:80
      ↓
iptables PREROUTING Chain auf der Node
      ↓
Regel: "Wenn Ziel 10.96.0.1:80 → wähle zufällig eine dieser IPs:"
         - 10.244.1.5:8080  (33% Chance)
         - 10.244.2.3:8080  (33% Chance)
         - 10.244.3.7:8080  (33% Chance)
      ↓
Paket wird DNAT'd (Destination NAT) zur gewählten Pod-IP
      ↓
Paket landet beim Pod
```

kube-proxy beobachtet den API-Server und **updated iptables-Regeln automatisch**
wenn Pods starten oder sterben.

### Container Runtime (containerd)

containerd ist das Programm das Container **wirklich startet und stoppt**.

```
kubelet → (CRI Interface) → containerd → Container
```

**CRI** = Container Runtime Interface — ein Standard damit kubelet mit verschiedenen
Runtimes sprechen kann (containerd, CRI-O, früher Docker).

Was containerd macht:
1. Container-Image aus Registry pullen (Docker Hub, GHCR, etc.)
2. Image entpacken und Filesystem aufbauen (overlayfs)
3. Namespaces und cgroups erstellen (Isolation!)
4. Prozess im Container starten

---

## 4. Workload-Objekte — Was läuft im Cluster

### Pod

Der **Pod** ist die kleinste deploybare Einheit in Kubernetes.

Ein Pod enthält einen oder mehrere Container die:
- **dasselbe Netzwerk teilen** (gleiche IP, können via localhost kommunizieren)
- **denselben Storage teilen** können (Volumes)
- **immer zusammen** auf einer Node laufen

```
┌─────────────────────────────────┐
│              Pod                │
│  IP: 10.244.1.5                 │
│                                 │
│  ┌──────────┐  ┌─────────────┐ │
│  │ Container│  │  Sidecar    │ │
│  │  (App)   │  │ (z.B. Proxy)│ │
│  │ Port 8080│  │  Port 9090  │ │
│  └──────────┘  └─────────────┘ │
│                                 │
│  Volume: /data  (geteilt)       │
└─────────────────────────────────┘
```

**Pods sind ephemeral** — sie werden gelöscht und neu erstellt, nie "repariert".
Eine neue Pod-Instanz bekommt eine neue IP. Deshalb braucht man Services.

### Deployment

Ein Deployment verwaltet **zustandslose** Anwendungen.

```
Deployment "meine-app" (replicas: 3)
      │
      ▼
ReplicaSet "meine-app-7d6f8b9c4"
      │
      ├── Pod "meine-app-7d6f8b9c4-xk2p9"  (Node: worker-1)
      ├── Pod "meine-app-7d6f8b9c4-m8n3j"  (Node: worker-2)
      └── Pod "meine-app-7d6f8b9c4-p4q7l"  (Node: worker-1)
```

**Rolling Update** — wie ein Deployment updated wird ohne Downtime:
```
Altes ReplicaSet: 3 Pods (v1)
Neues ReplicaSet: 0 Pods (v2)

Schritt 1: Starte 1 neuen Pod (v2) → 3 alt + 1 neu
Schritt 2: Stoppe 1 alten Pod (v1) → 2 alt + 1 neu
Schritt 3: Starte 1 neuen Pod (v2) → 2 alt + 2 neu
Schritt 4: Stoppe 1 alten Pod (v1) → 1 alt + 2 neu
Schritt 5: Starte 1 neuen Pod (v2) → 1 alt + 3 neu
Schritt 6: Stoppe 1 alten Pod (v1) → 0 alt + 3 neu ✓
```

Bei Fehler: `kubectl rollout undo deployment/meine-app` macht alles rückgängig.

### ReplicaSet

ReplicaSet stellt sicher dass immer N Pods laufen. Es wird normalerweise
vom Deployment verwaltet — man erstellt selten ein ReplicaSet direkt.

**Selector-Mechanismus:** Ein ReplicaSet "weiß" welche Pods zu ihm gehören
über **Labels**:

```yaml
selector:
  matchLabels:
    app: meine-app    # ← "Alle Pods mit diesem Label gehören zu mir"
```

### StatefulSet

Für **zustandsbehaftete** Anwendungen (Datenbanken, Message Queues).

Unterschied zum Deployment:
- Pods haben **stabile, vorhersehbare Namen**: `db-0`, `db-1`, `db-2`
- Pods starten und stoppen in **definierter Reihenfolge** (db-0 zuerst)
- Jeder Pod bekommt seinen **eigenen PersistentVolume** der bleibt wenn Pod neustartet
- Pods haben **stabile DNS-Namen**: `db-0.db-service.namespace.svc.cluster.local`

```
StatefulSet "db"
  ├── Pod "db-0"  ← PVC "data-db-0"  (10GB)
  ├── Pod "db-1"  ← PVC "data-db-1"  (10GB)
  └── Pod "db-2"  ← PVC "data-db-2"  (10GB)

Wenn db-1 neustartet: Gleicher Pod-Name, gleiche IP (durch stable network identity),
                       SELBES Volume "data-db-1" wird wieder gemountet
```

### DaemonSet

Ein DaemonSet stellt sicher dass auf **jeder Node genau 1 Pod** läuft.

Verwendung:
- Log-Collector (Fluentd, Promtail) — muss auf jeder Node Logs sammeln
- Monitoring-Agent (Node Exporter) — muss auf jeder Node Metrics sammeln
- Netzwerk-Plugin (Calico, Flannel) — muss auf jeder Node für Netzwerk sorgen
- kube-proxy selbst läuft als DaemonSet

```
Node worker-1  →  kube-proxy Pod
Node worker-2  →  kube-proxy Pod
Node worker-3  →  kube-proxy Pod
Node worker-4  →  kube-proxy Pod  (neu hinzugekommen → DaemonSet startet sofort Pod)
```

### Job & CronJob

**Job:** Führt einen Pod aus bis er **erfolgreich abgeschlossen** ist.
```
Job "datenbank-migration"
  → Startet Pod
  → Pod führt Migration aus
  → Pod beendet sich mit Exit Code 0
  → Job ist Complete ✓

  Wenn Pod mit Fehler endet → Job startet neuen Pod (retry)
```

**CronJob:** Job auf Zeitplan — wie Linux-cron aber in Kubernetes.
```yaml
schedule: "0 2 * * *"  # Täglich um 02:00 Uhr
```

---

## 5. Netzwerk in Kubernetes — Vollständig erklärt

Das Kubernetes-Netzwerk ist der komplexeste Teil. Hier eine vollständige Erklärung.

### Das Grundproblem: Container-Networking

Jeder Pod bekommt eine eigene IP. In einem Cluster mit 100 Nodes und 1000 Pods
müssen alle Pods miteinander kommunizieren können — egal auf welcher Node sie sind.

```
Pod A (10.244.1.5) auf Node 1  →  kann Pod B (10.244.2.8) auf Node 2 erreichen
```

Das funktioniert NICHT von alleine — dafür braucht es ein **CNI Plugin**.

### CNI — Container Network Interface

CNI (Container Network Interface) ist ein Standard für Netzwerk-Plugins.
Bekannte CNI-Plugins: **Flannel, Calico, Cilium, WeaveNet**

**Wie Flannel Pod-zu-Pod-Networking ermöglicht:**

```
Cluster-Netzwerk: 10.244.0.0/16

Node 1 bekommt:  10.244.1.0/24  (alle Pods auf Node 1 bekommen IPs aus diesem Bereich)
Node 2 bekommt:  10.244.2.0/24
Node 3 bekommt:  10.244.3.0/24

Pod A auf Node 1: 10.244.1.5
Pod B auf Node 2: 10.244.2.8
```

**Kommunikation Pod A → Pod B (verschiedene Nodes):**

```
Pod A schickt Paket:
  Source:      10.244.1.5  (Pod A)
  Destination: 10.244.2.8  (Pod B)
        ↓
Flannel auf Node 1 sieht: "10.244.2.x → das ist Node 2"
        ↓
VXLAN-Tunnel: Paket wird in UDP-Paket verpackt:
  Outer Source:      192.168.1.10  (Node 1 echte IP)
  Outer Destination: 192.168.1.11  (Node 2 echte IP)
  Inner Payload:     Original Pod-zu-Pod-Paket
        ↓
Paket reist durchs echte Netzwerk zu Node 2
        ↓
Flannel auf Node 2 entpackt VXLAN
        ↓
Paket landet bei Pod B (10.244.2.8) ✓
```

### Die 3 Netzwerk-Ebenen in Kubernetes

```
┌─────────────────────────────────────────────────────────┐
│  Ebene 1: Node-Netzwerk (echte IPs)                     │
│  192.168.1.10, 192.168.1.11, 192.168.1.12              │
│  → Das physische/virtuelle Netzwerk deiner Server        │
├─────────────────────────────────────────────────────────┤
│  Ebene 2: Pod-Netzwerk (virtuelle IPs)                  │
│  10.244.0.0/16                                          │
│  → Jeder Pod bekommt eine IP, alle Pods können sich     │
│    gegenseitig erreichen (via CNI-Plugin)               │
├─────────────────────────────────────────────────────────┤
│  Ebene 3: Service-Netzwerk (virtuelle ClusterIPs)       │
│  10.96.0.0/12                                           │
│  → Stabile IPs für Services — dahinter stecken Pod-IPs  │
│  → Verwaltet von kube-proxy via iptables               │
└─────────────────────────────────────────────────────────┘
```

### DNS in Kubernetes (CoreDNS)

Kubernetes betreibt einen eigenen DNS-Server: **CoreDNS**.

Jeder Service bekommt automatisch einen DNS-Namen:
```
<service-name>.<namespace>.svc.cluster.local

Beispiele:
  meine-app.default.svc.cluster.local
  argocd-server.argocd.svc.cluster.local
  traefik.traefik.svc.cluster.local
```

**Kurzformen** (funktionieren innerhalb desselben Namespaces):
```
meine-app                          # Im gleichen Namespace
meine-app.default                  # Mit Namespace
meine-app.default.svc              # Mit svc
meine-app.default.svc.cluster.local  # Vollständig (FQDN)
```

**Wie DNS-Auflösung funktioniert:**

```
Pod fragt: "Was ist die IP von argocd-server.argocd?"
      ↓
/etc/resolv.conf im Pod:
  nameserver 10.96.0.10   ← CoreDNS Service-IP
  search default.svc.cluster.local svc.cluster.local cluster.local
      ↓
CoreDNS empfängt Anfrage
      ↓
CoreDNS schaut in Kubernetes API: "Welchen Service gibt es mit dem Namen?"
      ↓
Antwort: "argocd-server.argocd.svc.cluster.local → 10.96.14.123"
      ↓
Pod kennt die ClusterIP und kann kommunizieren
```

### Ports und Protokolle

```
Container Port:  Der Port auf dem der Prozess im Container lauscht
                 (nginx lauscht auf 80)

Pod Port:        Identisch mit Container Port — Pods haben keine eigene
                 Port-Mapping-Schicht

Service Port:    Der Port des Services (kann anders sein als Container Port)
                 z.B. Service Port 80 → Container Port 8080

NodePort:        Port auf jeder Node (30000-32767)

HostPort:        Port direkt auf der Node (wie docker -p) — selten genutzt
```

---

## 6. Services — Wie Traffic zu Pods kommt

### Das Problem ohne Services

Pods sterben und werden neu erstellt — dabei ändert sich die IP:
```
Pod "meine-app-abc" hatte IP: 10.244.1.5  → stirbt
Pod "meine-app-xyz" hat  IP: 10.244.2.9  → neu
```

Andere Pods können nicht hardcoded auf `10.244.1.5` zeigen — die IP ist weg.
**Services lösen das Problem** mit einer stabilen virtuellen IP.

### Wie Services funktionieren

Ein Service ist eine **stabile virtuelle IP + DNS-Name** der Traffic zu einer
Gruppe von Pods weiterleitet.

**Der Selector-Mechanismus:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: meine-app
spec:
  selector:
    app: meine-app    # ← "Schick Traffic an alle Pods mit diesem Label"
  ports:
  - port: 80          # ← Service lauscht auf Port 80
    targetPort: 8080  # ← Pod-Container lauscht auf Port 8080
```

**Das Endpoints-Objekt** wird automatisch erstellt und enthält die echten Pod-IPs:
```
Service "meine-app" → ClusterIP: 10.96.42.100
                       Port: 80

Endpoints "meine-app":
  - 10.244.1.5:8080   (Pod auf Node 1, läuft)
  - 10.244.2.9:8080   (Pod auf Node 2, läuft)
  - 10.244.3.2:8080   (Pod auf Node 3, läuft)
```

Wenn ein Pod stirbt: **Endpoint Controller** entfernt die IP sofort aus dem Endpoints-Objekt.
kube-proxy updated iptables → kein Traffic mehr an tote Pod-IP.

### Service-Typen im Detail

#### ClusterIP (Standard)

```
┌────────────────────────────────────────┐
│              Kubernetes Cluster        │
│                                        │
│  Pod A ─────► Service "db"            │
│               ClusterIP: 10.96.5.100  │
│               Port: 5432              │
│                    │                  │
│                    ▼                  │
│               Pod "db-0" (10.244.1.5) │
└────────────────────────────────────────┘
```

- Nur **innerhalb des Clusters** erreichbar
- Kein Zugriff von außen
- Standard für interne Kommunikation

#### NodePort

```
Internet
    │
    ▼
Node (192.168.1.10:30080)   ← Jede Node öffnet diesen Port
    │
    ▼
Service "meine-app" (ClusterIP: 10.96.5.200)
    │
    ▼
Pods
```

- Öffnet einen Port (30000-32767) auf **jeder Node**
- Von außen erreichbar über `<jede-node-ip>:<nodeport>`
- Unpraktisch für Produktion (seltsame Ports, keine TLS, kein Host-based Routing)

#### LoadBalancer

```
Internet
    │
    ▼
Externe IP (z.B. 89.167.116.105)  ← Vom Cloud-Provider bereitgestellt
    │
    ▼
Cloud Load Balancer
    │
    ▼
Service "traefik" (ClusterIP: 10.96.0.50)
    │
    ▼
Traefik Pods
```

- Für Cloud-Umgebungen (AWS, GCP, Hetzner Cloud, etc.)
- Cloud-Provider erstellt automatisch einen echten Load Balancer
- Bekommt eine öffentliche IP
- **In diesem Setup:** Traefik hat einen LoadBalancer Service

#### Headless Service (ClusterIP: None)

```yaml
spec:
  clusterIP: None   # ← Headless
```

- Kein Load Balancing
- DNS gibt direkt **alle Pod-IPs** zurück
- Pod kann selbst entscheiden mit welchem anderen Pod es redet
- Wichtig für StatefulSets (db-0 direkt ansprechen)

---

## 7. Ingress & IngressRoute — Traffic von außen

### Das Problem

Du hast 10 verschiedene Anwendungen im Cluster:
- `app-a.beispiel.de`
- `app-b.beispiel.de`
- `api.beispiel.de`

Du willst nicht für jede App einen eigenen LoadBalancer (→ teure externe IPs).
Du willst einen **einzigen Eintrittspunkt** der den Traffic nach Hostname/Pfad verteilt.

**Das ist der Job des Ingress Controllers.**

### Ingress Controller

```
Internet
    │
    ▼
Externe IP :443/:80
    │
    ▼
┌──────────────────────────────────────────────┐
│           Ingress Controller (Traefik)        │
│                                              │
│  Regel 1: app-a.beispiel.de → Service app-a │
│  Regel 2: app-b.beispiel.de → Service app-b │
│  Regel 3: api.beispiel.de   → Service api   │
└──────────────────────────────────────────────┘
    │         │         │
    ▼         ▼         ▼
Service    Service    Service
 app-a      app-b      api
    │         │         │
    ▼         ▼         ▼
  Pods      Pods      Pods
```

### Standard Ingress vs. Traefik IngressRoute

**Standard Kubernetes Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: meine-app
spec:
  rules:
  - host: app.beispiel.de
    http:
      paths:
      - path: /
        backend:
          service:
            name: meine-app
            port:
              number: 80
```

**Traefik IngressRoute (CRD):**
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: meine-app
spec:
  entryPoints:
    - websecure    # HTTPS
  routes:
  - match: Host(`app.beispiel.de`)
    kind: Rule
    services:
    - name: meine-app
      port: 80
  tls:
    secretName: meine-app-tls   # TLS-Zertifikat aus Kubernetes Secret
```

Traefik IngressRoute ist mächtiger: Middleware, TCP-Routing, gewichtetes Load Balancing, etc.

---

## 8. Konfiguration & Secrets

### ConfigMap

Konfigurationsdaten die Pods nutzen können:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgresql://db:5432/mydb"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      debug: false
```

**In Pod einbinden als Umgebungsvariable:**
```yaml
env:
- name: DATABASE_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DATABASE_URL
```

**In Pod einbinden als Datei:**
```yaml
volumes:
- name: config
  configMap:
    name: app-config
volumeMounts:
- name: config
  mountPath: /etc/config
# → /etc/config/config.yaml existiert im Container
```

### Secret

Wie ConfigMap aber für **sensitive Daten**. Werte sind base64-kodiert (nicht verschlüsselt!).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: bXlzZWNyZXRwYXNzd29yZA==   # base64("mysecretpassword")
```

**Wichtige Secret-Typen:**
```
Opaque              → Allgemeine Secrets (Passwörter, API-Keys)
kubernetes.io/tls   → TLS-Zertifikate (tls.crt + tls.key)
kubernetes.io/dockerconfigjson → Registry-Zugangsdaten für Image Pull
```

**Achtung:** Secrets sind standardmäßig nicht wirklich verschlüsselt — nur base64.
Für echte Verschlüsselung: **Sealed Secrets** oder **Vault**.

---

## 9. Storage — Persistenter Speicher

Container-Dateisysteme sind ephemeral — alles geht verloren wenn der Container stirbt.
Für Datenbanken und persistente Daten braucht man Volumes.

### Die Storage-Schichten

```
┌─────────────────────────────────────────────────┐
│  StorageClass                                   │
│  "Wie wird Speicher bereitgestellt?"            │
│  (NFS-Provisioner, Longhorn, AWS EBS, etc.)     │
├─────────────────────────────────────────────────┤
│  PersistentVolume (PV)                          │
│  "Ein konkretes Stück Speicher"                 │
│  (z.B. 50GB auf einem bestimmten Server)        │
├─────────────────────────────────────────────────┤
│  PersistentVolumeClaim (PVC)                    │
│  "Ein Pod fordert Speicher an"                  │
│  (z.B. "Ich brauche 10GB ReadWriteOnce")        │
├─────────────────────────────────────────────────┤
│  Volume Mount im Pod                            │
│  "Der Pod nutzt den Speicher"                   │
└─────────────────────────────────────────────────┘
```

**Ablauf:**
```
1. Admin erstellt StorageClass (oder sie ist schon da)
2. Developer erstellt PVC: "Ich brauche 10GB"
3. K8s findet passendes PV oder erstellt eines (Dynamic Provisioning)
4. PVC wird an PV gebunden (Bound)
5. Pod referenziert PVC → bekommt Zugriff auf den Speicher
```

### Access Modes

```
ReadWriteOnce (RWO)   → Nur eine Node kann lesen+schreiben (z.B. Datenbank)
ReadOnlyMany (ROX)    → Viele Nodes können lesen
ReadWriteMany (RWX)   → Viele Nodes können lesen+schreiben (z.B. NFS, CephFS)
```

---

## 10. RBAC — Wer darf was

RBAC (Role-Based Access Control) kontrolliert wer im Cluster was machen darf.

### Die 4 Objekte

```
ServiceAccount  →  Eine Identität (für Pods oder Menschen)
Role            →  Berechtigungen in einem Namespace
ClusterRole     →  Berechtigungen clusterweit
RoleBinding     →  Verbindet ServiceAccount mit Role
ClusterRoleBinding → Verbindet ServiceAccount mit ClusterRole
```

**Beispiel: ArgoCD darf alles lesen und Deployments patchen:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-binding
subjects:
- kind: ServiceAccount
  name: argocd-application-controller
  namespace: argocd
roleRef:
  kind: ClusterRole
  name: argocd-role
```

---

## 11. ArgoCD — GitOps erklärt

### Was ist GitOps?

GitOps bedeutet: **Git ist die einzige Quelle der Wahrheit.**

```
Traditionell:
  Entwickler → kubectl apply → Cluster
  Problem: Was läuft wirklich im Cluster? Wer hat was deployed? Rollback wie?

GitOps:
  Entwickler → Git Push → ArgoCD → kubectl apply → Cluster
  Vorteile:
    - Git ist Audit-Log ("Wer hat was wann deployed?")
    - Pull Request = Review-Prozess für Deployments
    - Rollback = git revert
    - Cluster kann aus Git komplett neu aufgebaut werden
```

### ArgoCD Architektur

```
┌──────────────────────────────────────────────────────┐
│                      ArgoCD                          │
│                                                      │
│  ┌─────────────────┐    ┌──────────────────────┐    │
│  │ argocd-server   │    │ application-         │    │
│  │ (Web UI + API)  │    │ controller           │    │
│  └─────────────────┘    │ (Reconciliation Loop)│    │
│                         └──────────────────────┘    │
│  ┌─────────────────┐    ┌──────────────────────┐    │
│  │ repo-server     │    │ redis                │    │
│  │ (Git Clone +    │    │ (Cache)              │    │
│  │  Render)        │    └──────────────────────┘    │
│  └─────────────────┘                                │
└──────────────────────────────────────────────────────┘
```

- **argocd-server:** Web UI und API — hier loggst du dich ein
- **application-controller:** Das Herzstück — vergleicht Git mit Cluster
- **repo-server:** Cloned Git-Repos, rendert Helm-Charts zu YAML
- **redis:** Cached Git-Inhalte und Cluster-Status

### Application-Objekt

Eine ArgoCD Application definiert: "Nimm diesen Git-Pfad und deploy es in diesen Namespace."

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
    targetRevision: HEAD          # Welcher Branch/Tag/Commit
    path: apps/meine-app          # Welcher Ordner im Repo
    # Oder für Helm:
    chart: traefik
    helm:
      valueFiles:
      - values/production.yaml
  destination:
    server: https://kubernetes.default.svc   # Dieser Cluster
    namespace: meine-app
  syncPolicy:
    automated:
      prune: true      # Lösche Objekte die nicht mehr im Git sind
      selfHeal: true   # Überschreibe manuelle kubectl-Änderungen
    syncOptions:
      - CreateNamespace=true
```

### App-of-Apps Pattern

Das mächtigste ArgoCD-Pattern — wie in diesem Setup verwendet:

```
bootstrap/root-app.yaml (manuell einmalig anwenden)
      │
      ▼
ArgoCD Application "root"
      │  überwacht Ordner apps/
      │
      ├── apps/traefik.yaml      → ArgoCD Application "traefik"
      ├── apps/cert-manager.yaml → ArgoCD Application "cert-manager"
      ├── apps/manifests.yaml    → ArgoCD Application "manifests"
      └── apps/...               → ArgoCD Application "..."
```

**Warum:** Ein einziger `kubectl apply -f bootstrap/root-app.yaml` bootstrapped
den gesamten Cluster. ArgoCD deployed sich dabei quasi selbst und alle anderen Apps.

### Sync-Zyklus im Detail

```
1. ArgoCD repo-server cloned Git-Repo (alle 3 Minuten oder via Webhook)
        ↓
2. repo-server rendert Manifeste:
   - Roh-YAML: direkt übernehmen
   - Helm Chart: helm template ausführen → YAML
   - Kustomize: kustomize build → YAML
        ↓
3. application-controller vergleicht:
   Desired State (Git-YAML) vs. Live State (kubectl get -o json)
        ↓
4. Unterschied gefunden (OutOfSync)?
   → Bei automated sync: sofort kubectl apply
   → Ohne automated: warte auf manuellen Sync
        ↓
5. Nach apply: warte bis Pods Running sind (Health Check)
        ↓
6. Status: Synced + Healthy ✓
```

### Sync-Status & Health

```
Sync Status:
  Synced     → Git und Cluster sind identisch
  OutOfSync  → Unterschied zwischen Git und Cluster
  Unknown    → ArgoCD konnte nicht vergleichen

Health Status:
  Healthy    → Alle Pods laufen, alle Checks grün
  Progressing → Deployment läuft gerade
  Degraded   → Etwas stimmt nicht (CrashLoopBackOff, etc.)
  Missing    → Ressource existiert nicht im Cluster
  Suspended  → Ressource ist pausiert
```

### selfHeal und prune erklärt

```
selfHeal: true
  Szenario: Jemand macht manuell kubectl edit deployment/meine-app
            und ändert die Replica-Anzahl von 3 auf 5
  Ohne selfHeal: Diese Änderung bleibt
  Mit selfHeal:  ArgoCD erkennt den Unterschied zu Git und setzt zurück auf 3
  Grund: Git ist die einzige Wahrheit — manuelle Änderungen sind verboten

prune: true
  Szenario: Jemand löscht eine YAML-Datei aus Git
            Das Kubernetes-Objekt existiert noch im Cluster
  Ohne prune: Objekt bleibt im Cluster (Datenmüll)
  Mit prune:  ArgoCD löscht das Objekt automatisch
```

---

## 12. Traefik — Der Ingress Controller

### Was Traefik macht

Traefik ist ein **Reverse Proxy und Load Balancer** der als Kubernetes Ingress Controller läuft.

```
Internet → Traefik → richtige App im Cluster
```

Traefik beobachtet den Kubernetes API-Server und konfiguriert sich **automatisch**
wenn neue IngressRoutes erstellt werden.

### Traefik Konzepte

```
EntryPoints:  Auf welchen Ports lauscht Traefik?
              web:       Port 80  (HTTP)
              websecure: Port 443 (HTTPS)

Routers:      Welche Requests gehen wohin?
              Rule: Host(`meine-app.beispiel.de`) → Service meine-app

Middlewares:  Was passiert mit dem Request unterwegs?
              - Redirect HTTP → HTTPS
              - Basic Auth
              - Rate Limiting
              - Header manipulation

Services:     Wohin geht der Traffic?
              → Kubernetes Service → Pods
```

### Traefik als DaemonSet vs. Deployment

In diesem Setup läuft Traefik als **Deployment** mit einem **LoadBalancer Service**.

```
Extern: 89.167.116.105:443  (LoadBalancer IP)
          ↓
LoadBalancer Service "traefik" (K8s)
          ↓
Traefik Pod(s) (Deployment)
          ↓
IngressRoute Rules auswerten
          ↓
Service im Cluster
          ↓
Pod
```

### Automatische Service Discovery

Traefik liest **alle Namespaces** nach IngressRoutes:
```
Kubernetes API: "Gibt es neue/geänderte IngressRoutes?"
      ↓
Neue IngressRoute gefunden für "app.beispiel.de"
      ↓
Traefik erstellt intern Router-Konfiguration
      ↓
Ab sofort: Requests für "app.beispiel.de" gehen an richtigen Service
      ↓
Keine Traefik-Restart nötig — dynamische Konfiguration
```

---

## 13. cert-manager — Automatische TLS-Zertifikate

### Das Problem

HTTPS braucht ein TLS-Zertifikat. Früher: manuell beantragen, manuell erneuern,
alle 90 Tage → Stress.

cert-manager automatisiert den kompletten Prozess.

### Die Objekte

**ClusterIssuer:** "Wie besorgen wir Zertifikate?" — einmalig konfigurieren
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@beispiel.de
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: traefik
```

**Certificate:** "Ich brauche ein Zertifikat für diese Domain"
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: meine-app-tls
spec:
  secretName: meine-app-tls-secret   # Hier wird das Cert gespeichert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - meine-app.beispiel.de
```

### Der ACME-Prozess (HTTP-01 Challenge)

```
1. cert-manager erstellt CertificateRequest
        ↓
2. cert-manager kontaktiert Let's Encrypt API:
   "Ich will ein Zertifikat für meine-app.beispiel.de"
        ↓
3. Let's Encrypt antwortet:
   "Beweise, dass du meine-app.beispiel.de kontrollierst.
    Mach folgenden Inhalt erreichbar:
    http://meine-app.beispiel.de/.well-known/acme-challenge/TOKEN"
        ↓
4. cert-manager erstellt temporären Pod + Service + Ingress
   der genau diesen TOKEN ausliefert
        ↓
5. Let's Encrypt ruft URL ab und prüft TOKEN ✓
        ↓
6. Let's Encrypt stellt Zertifikat aus
        ↓
7. cert-manager speichert Zertifikat in Kubernetes Secret:
   Secret "meine-app-tls-secret":
     tls.crt: -----BEGIN CERTIFICATE-----...
     tls.key: -----BEGIN PRIVATE KEY-----...
        ↓
8. Traefik liest Secret und nutzt Zertifikat für TLS ✓
        ↓
9. 30 Tage vor Ablauf: cert-manager erneuert automatisch
```

---

## 14. Die komplette Prozesskette

### Prozesskette 1: Git Push → App läuft im Cluster

```
Entwickler pusht YAML-Änderung zu GitHub
        ↓
ArgoCD repo-server pollt GitHub alle 3 Minuten
(oder Webhook → sofort)
        ↓
Unterschied erkannt: Desired State (Git) ≠ Live State (Cluster)
        ↓
application-controller führt Sync aus:
  repo-server rendert Helm/YAML zu Kubernetes-Manifesten
        ↓
kubectl apply der Manifeste an kube-apiserver
        ↓
kube-apiserver validiert und speichert in etcd
        ↓
Deployment Controller erkennt: "Neues Deployment oder Änderung"
  → Erstellt neues ReplicaSet
        ↓
ReplicaSet Controller: "Ich brauche 3 Pods, habe 0"
  → Erstellt 3 Pod-Objekte (ohne Node-Zuweisung)
        ↓
kube-scheduler: "Diese Pods brauchen eine Node"
  → Analysiert verfügbare Nodes (CPU, RAM, Affinities)
  → Weist jeden Pod einer Node zu
        ↓
kubelet auf jeweiliger Node: "Ich soll Pod X ausführen"
  → Ruft containerd auf
  → containerd pulled Image (falls nicht gecacht)
  → Container startet
        ↓
kubelet meldet: Pod Status = Running
        ↓
Endpoint Controller: "Pod-IP hat sich geändert"
  → Updated Endpoints-Objekt des Services
        ↓
kube-proxy: "Endpoints haben sich geändert"
  → Updated iptables-Regeln auf allen Nodes
        ↓
ArgoCD: Health Check → Alle Pods Running → Status: Synced + Healthy ✓
```

### Prozesskette 2: HTTPS-Request von Browser zu Pod

```
Nutzer tippt: https://meine-app.beispiel.de
        ↓
1. DNS-Auflösung:
   Browser fragt DNS-Resolver: "Wo ist meine-app.beispiel.de?"
   DNS-Resolver schaut nach A-Record beim Domain-Provider
   Antwort: 89.167.116.105 (LoadBalancer IP)
        ↓
2. TCP Three-Way Handshake:
   Browser → SYN → 89.167.116.105:443
   Server  → SYN-ACK
   Browser → ACK
   TCP-Verbindung steht
        ↓
3. TLS Handshake:
   Browser: "Client Hello" (unterstützte TLS-Versionen, Cipher Suites)
   Traefik: "Server Hello" + Zertifikat (aus K8s Secret, ausgestellt von Let's Encrypt)
   Browser: Prüft Zertifikat (CA-Kette, Ablaufdatum, Domain)
   Schlüsselaustausch (Diffie-Hellman) → Session Keys
   Ab jetzt: alles verschlüsselt
        ↓
4. HTTP-Request (verschlüsselt):
   GET / HTTP/2
   Host: meine-app.beispiel.de
        ↓
5. LoadBalancer Service leitet weiter:
   89.167.116.105:443 → Traefik Pod (10.244.x.x:8443)
   (via kube-proxy iptables DNAT)
        ↓
6. Traefik wertet IngressRoute aus:
   Host: meine-app.beispiel.de → Match mit IngressRoute-Regel
   Middlewares anwenden (z.B. Header setzen)
   Ziel: Service "meine-app" Port 80
        ↓
7. Traefik macht HTTP-Request zum Service:
   → CoreDNS löst "meine-app.namespace.svc.cluster.local" auf → ClusterIP
   → kube-proxy DNAT: ClusterIP:80 → Pod-IP:8080
        ↓
8. Request landet beim Pod:
   Pod verarbeitet Request
   Antwort zurück durch denselben Weg
        ↓
9. Browser zeigt Seite an ✓
```

### Prozesskette 3: TLS-Zertifikat ausstellen

```
IngressRoute mit tls.secretName erstellt
        ↓
cert-manager Certificate-Objekt erstellt
        ↓
cert-manager: "Secret existiert nicht → Zertifikat beantragen"
        ↓
ACME-Prozess bei Let's Encrypt starten
        ↓
Challenge-Pod, Service, Ingress erstellen
        ↓
Let's Encrypt validiert Domain ✓
        ↓
Zertifikat wird ausgestellt
        ↓
cert-manager speichert in Kubernetes Secret
        ↓
Traefik liest Secret → TLS aktiv ✓
        ↓
(nach 60 Tagen) cert-manager erneuert automatisch
```

### Vollständiges Zusammenspiel als Übersicht

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub (Git Repo)                           │
│  apps/     bootstrap/    values/    manifests/                      │
└─────────────────────┬───────────────────────────────────────────────┘
                      │ Poll / Webhook
                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           ArgoCD                                    │
│  ┌──────────────┐  ┌─────────────────┐  ┌─────────────────────┐   │
│  │ repo-server  │  │ app-controller  │  │ argocd-server       │   │
│  │ (Clone+Render│  │ (Reconcile Loop)│  │ (UI + API)          │   │
│  └──────────────┘  └────────┬────────┘  └─────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ kubectl apply
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Kubernetes Control Plane                       │
│  ┌────────────┐  ┌──────┐  ┌──────────────────┐  ┌─────────────┐  │
│  │ API-Server │  │ etcd │  │ Controller Manager│  │  Scheduler  │  │
│  └────────────┘  └──────┘  └──────────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
          │                              │
          │                              │ Pod-Zuweisung
          ▼                              ▼
┌─────────────────────┐       ┌─────────────────────┐
│     Worker Node 1   │       │     Worker Node 2   │
│  ┌───────────────┐  │       │  ┌───────────────┐  │
│  │    kubelet    │  │       │  │    kubelet    │  │
│  │  kube-proxy   │  │       │  │  kube-proxy   │  │
│  │  containerd   │  │       │  │  containerd   │  │
│  │               │  │       │  │               │  │
│  │  ┌─────────┐  │  │       │  │  ┌─────────┐  │  │
│  │  │ Traefik │  │  │       │  │  │  App    │  │  │
│  │  │  Pod    │  │  │       │  │  │  Pod    │  │  │
│  │  └─────────┘  │  │       │  │  └─────────┘  │  │
│  └───────────────┘  │       │  └───────────────┘  │
└─────────────────────┘       └─────────────────────┘
          │                              │
          │  Netzwerk (CNI Plugin)        │
          └──────────────────────────────┘
          │
          │ Traffic von außen
          ▼
  DNS → LoadBalancer IP → Traefik → Service → Pod
```

---

## 15. Schnellreferenz & Befehle

### Cluster-Überblick

```bash
# Alle Nodes anzeigen
kubectl get nodes -o wide

# Alle Pods in allen Namespaces
kubectl get pods -A

# Alle Services
kubectl get services -A

# Alle ArgoCD Applications
kubectl get applications -n argocd
```

### Debugging

```bash
# Pod-Logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # Letzter abgestürzter Container

# Pod-Details (Events sind oft hilfreich!)
kubectl describe pod <pod-name> -n <namespace>

# In Pod einloggen
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Service Endpoints prüfen (sind Pods hinter dem Service?)
kubectl get endpoints <service-name> -n <namespace>

# DNS im Cluster testen
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup argocd-server.argocd
```

### ArgoCD

```bash
# App-Status
kubectl get applications -n argocd

# App manuell synchronisieren
argocd app sync <app-name>

# App-Details
argocd app get <app-name>

# Alle OutOfSync Apps
argocd app list | grep OutOfSync
```

### Netzwerk & TLS

```bash
# Zertifikat-Status prüfen
kubectl get certificates -A
kubectl describe certificate <cert-name> -n <namespace>

# Challenge-Status (während Ausstellung)
kubectl get challenges -A

# Traefik IngressRoutes
kubectl get ingressroutes -A

# TLS-Zertifikat eines Endpoints prüfen
openssl s_client -connect meine-app.beispiel.de:443 -servername meine-app.beispiel.de
```

### Nützliche Flags

```bash
-n <namespace>      # Namespace angeben
-A / --all-namespaces  # Alle Namespaces
-o wide             # Mehr Details (z.B. Node-Name, IPs)
-o yaml             # Als YAML ausgeben
-o json             # Als JSON ausgeben
--watch / -w        # Live-Updates
-f <datei>          # YAML-Datei angeben
```

---

*Diese Dokumentation beschreibt das konkrete Setup mit ArgoCD, Traefik und cert-manager
auf einem selbstverwalteten Kubernetes-Cluster.*
