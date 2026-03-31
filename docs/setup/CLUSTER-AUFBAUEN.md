# Cluster aufbauen — Komplettes Setup von Null auf Produktion

> Diese Dokumentation beschreibt den **korrekten Weg** ein Kubernetes-Cluster
> mit ArgoCD, Traefik und cert-manager von Grund auf aufzubauen.
> Kein Flickwerk, kein "ins laufende System eingreifen" — sauber von Anfang bis Ende.
>
> Zielgruppe: Jemand mit einem frischen Server und leichtem Vorwissen.

---

## Inhaltsverzeichnis

1. [Was wir bauen und warum](#1-was-wir-bauen-und-warum)
2. [Voraussetzungen](#2-voraussetzungen)
3. [Phase 1 — Server vorbereiten](#3-phase-1--server-vorbereiten)
4. [Phase 2 — Kubernetes installieren](#4-phase-2--kubernetes-installieren)
5. [Phase 3 — Git Repository aufbauen](#5-phase-3--git-repository-aufbauen)
6. [Phase 4 — ArgoCD installieren](#6-phase-4--argocd-installieren)
7. [Phase 5 — Helm Charts vorbereiten](#7-phase-5--helm-charts-vorbereiten)
8. [Phase 6 — GitOps Bootstrap](#8-phase-6--gitops-bootstrap)
9. [Phase 7 — ArgoCD per HTTPS erreichbar machen](#9-phase-7--argocd-per-https-erreichbar-machen)
10. [Phase 8 — Verifizierung](#10-phase-8--verifizierung)
11. [Reihenfolge und warum sie wichtig ist](#11-reihenfolge-und-warum-sie-wichtig-ist)
12. [Die komplette Dateistruktur](#12-die-komplette-dateistruktur)

---

## 1. Was wir bauen und warum

### Das Ziel

Am Ende dieses Guides hast du:

```
https://deine-domain.de/argocd  ← ArgoCD Web UI, gesichert mit echtem TLS
        │
        ├── Traefik verwaltet den eingehenden Traffic
        ├── cert-manager stellt TLS-Zertifikat automatisch aus
        └── ArgoCD deployt alles aus deinem GitHub-Repo
```

Und wenn du danach eine neue Anwendung deployen willst:
```bash
# Das ist alles was du tun musst:
vim apps/meine-neue-app.yaml
git add . && git commit -m "add neue-app" && git push
# → ArgoCD deployed automatisch. Fertig.
```

### Die Komponenten und ihre Rollen

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Repo                                            │
│  → Einzige Quelle der Wahrheit                          │
│  → Alle Konfiguration als YAML                          │
└──────────────────────┬──────────────────────────────────┘
                       │ ArgoCD liest von hier
                       ▼
┌─────────────────────────────────────────────────────────┐
│  ArgoCD                                                 │
│  → Vergleicht GitHub mit Cluster                        │
│  → Deployed automatisch was in GitHub steht             │
└──────────────────────┬──────────────────────────────────┘
                       │ deployed
                       ▼
┌──────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                      │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │   Traefik   │  │ cert-manager │  │  Deine Apps    │  │
│  │  (Routing)  │  │    (TLS)     │  │                │  │
│  └─────────────┘  └──────────────┘  └────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Voraussetzungen

### Was du brauchst

| Was | Warum |
|-----|-------|
| Einen Server (VPS/Bare Metal) | Kubernetes läuft darauf |
| Ubuntu 22.04 oder Debian 12 | Bewährte Basis für K8s |
| Mindestens 2 CPU Cores, 4GB RAM | K8s Control Plane braucht Ressourcen |
| Eine öffentliche IP | Damit deine Domain erreichbar ist |
| Eine Domain | Für HTTPS (z.B. goava.ai) |
| GitHub Account | Für das GitOps-Repository |
| SSH-Key | Für GitHub-Authentifizierung |

### SSH-Key erstellen (falls nicht vorhanden)

```bash
# Auf deinem lokalen Rechner:
ssh-keygen -t ed25519 -C "dein@email.de" -f ~/.ssh/id_ed25519_github

# Public Key anzeigen → diesen bei GitHub hinterlegen
cat ~/.ssh/id_ed25519_github.pub
```

Bei GitHub: Settings → SSH and GPG keys → New SSH key → Public Key einfügen

### GitHub Repository erstellen

1. GitHub.com → New Repository
2. Name: `K8s-Helm` (oder was du möchtest)
3. Private oder Public (beides funktioniert)
4. **Keine README**, **kein .gitignore** — wir starten leer

```bash
# Lokal:
mkdir K8s && cd K8s
git init
git remote add origin git@github.com:DEIN-USERNAME/K8s-Helm.git
```

---

## 3. Phase 1 — Server vorbereiten

Alle Befehle auf dem Server ausführen (per SSH einloggen).

### System aktualisieren

```bash
apt update && apt upgrade -y
```

### Swap deaktivieren

Kubernetes funktioniert nicht korrekt mit Swap. Swap ist ein Bereich auf der
Festplatte der als zusätzlicher RAM genutzt wird — Kubernetes braucht verlässliche
Speicherangaben und kann mit Swap nicht korrekt kalkulieren.

```bash
# Swap sofort deaktivieren
swapoff -a

# Swap dauerhaft deaktivieren (überlebt Neustart)
sed -i '/swap/d' /etc/fstab

# Prüfen: Output sollte leer sein
swapon --show
```

### Kernel-Module und Netzwerk-Parameter

Kubernetes braucht bestimmte Kernel-Einstellungen:

```bash
# Module laden die Kubernetes für Netzwerk braucht
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay       # Container-Dateisystem (OverlayFS)
br_netfilter  # Bridge-Netzwerk-Filterung
EOF

modprobe overlay
modprobe br_netfilter

# Netzwerk-Parameter setzen
# ip_forward: Erlaubt dem Server Traffic zwischen Interfaces weiterzuleiten
# bridge-nf: Ermöglicht iptables-Regeln für Bridge-Traffic (Service-Routing)
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

**Warum `ip_forward`?** Ohne das leitet der Linux-Kernel keinen Traffic zwischen
verschiedenen Netzwerk-Interfaces weiter. Kubernetes-Pods kommunizieren über
virtuelle Interfaces — ohne `ip_forward` kein Pod-zu-Pod-Traffic.

### containerd installieren

containerd ist die Container-Runtime — das Programm das Docker-Images tatsächlich
startet. Kubernetes nutzt es über die CRI (Container Runtime Interface):

```bash
apt install -y containerd

# Standard-Konfiguration erstellen
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# SystemdCgroup aktivieren — wichtig für korrekte Ressourcenverwaltung
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
```

**Warum `SystemdCgroup = true`?** Kubernetes und systemd müssen sich einig sein
wer die cgroup-Hierarchie verwaltet. Ohne diese Einstellung kann es zu
Ressourcen-Konflikten kommen.

---

## 4. Phase 2 — Kubernetes installieren

### kubeadm, kubelet und kubectl installieren

```bash
# Kubernetes APT-Repository hinzufügen
apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubeadm kubelet kubectl

# Version einfrieren — automatische Updates könnten den Cluster brechen
apt-mark hold kubeadm kubelet kubectl
```

**Warum Version einfrieren?** Ein unbeabsichtigtes `apt upgrade` könnte Kubernetes
auf eine inkompatible Version hochziehen. Kubernetes-Updates sollten immer bewusst
und geplant durchgeführt werden.

**Die drei Komponenten:**
- `kubeadm` — Tool um Kubernetes zu initialisieren
- `kubelet` — Agent der auf dem Node läuft und Container verwaltet
- `kubectl` — CLI um mit dem Cluster zu kommunizieren

### Kubernetes initialisieren

```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \    # IP-Bereich für Pods
  --apiserver-advertise-address=DEINE-SERVER-IP
```

**`--pod-network-cidr`**: Jeder Pod bekommt eine IP aus diesem Bereich.
`10.244.0.0/16` passt zum Flannel-CNI das wir gleich installieren.
Dieser Bereich ist nur intern sichtbar, nicht im Internet.

Nach Abschluss zeigt kubeadm einen `kubeadm join` Befehl — den für
spätere Worker-Nodes merken/speichern (für Single-Node nicht benötigt).

```bash
# kubectl Konfiguration einrichten
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Prüfen ob API-Server antwortet:
kubectl cluster-info
# → Kubernetes control plane is running at https://...
```

### Auf Single-Node: Control Plane auch als Worker nutzen

Standardmäßig werden auf dem Control Plane Node keine normalen Pods gescheduled
(Taint). Für Single-Node-Setup den Taint entfernen:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
#                                                               ↑ Minus = entfernen
```

### CNI installieren — Flannel

CNI (Container Network Interface) ermöglicht Pods miteinander zu kommunizieren.
Ohne CNI bleiben alle Pods im Status `Pending`:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel erstellt ein **Overlay-Netzwerk**: Jeder Node bekommt ein Subnetz aus
dem Pod-CIDR. Pods auf verschiedenen Nodes kommunizieren über verschlüsselte
Tunnel (VXLAN).

```bash
# Warten bis alle System-Pods laufen (kann 1-2 Minuten dauern)
kubectl get pods -n kube-system -w

# Cluster-Status prüfen:
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# k8s-main   Ready    control-plane   2m    v1.31.x
# → "Ready" = CNI funktioniert, Node ist bereit
```

### Helm installieren

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Prüfen:
helm version
# → version.BuildInfo{Version:"v3.x.x", ...}
```

---

## 5. Phase 3 — Git Repository aufbauen

**Jetzt verlassen wir den Server und arbeiten lokal.**

Diese Struktur legen wir an:

```
K8s/
├── bootstrap/
│   └── root-app.yaml
├── apps/
│   ├── traefik.yaml
│   ├── cert-manager.yaml
│   └── manifests.yaml
├── values/
│   ├── traefik/
│   │   └── values.yaml
│   └── cert-manager/
│       └── values.yaml
└── manifests/
    ├── argocd/
    │   └── ingress.yaml
    └── cert-manager/
        └── clusterissuer.yaml
```

### bootstrap/root-app.yaml

Das ist die einzige Datei die wir jemals manuell `kubectl apply` werden.
Sie startet den kompletten GitOps-Prozess:

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
    repoURL: git@github.com:DEIN-USERNAME/K8s-Helm.git
    targetRevision: HEAD
    path: apps                  # Überwacht den apps/ Ordner
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true               # Lösche was in Git entfernt wurde
      selfHeal: true            # Korrigiere manuelle Änderungen im Cluster
    syncOptions:
      - CreateNamespace=true
```

### apps/traefik.yaml

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
    # Quelle 1: Helm Chart direkt vom Traefik-Repository
    - repoURL: https://helm.traefik.io/traefik
      chart: traefik
      targetRevision: "33.x"
      helm:
        releaseName: traefik
        valueFiles:
          - $values/values/traefik/values.yaml   # Referenziert Quelle 2
    # Quelle 2: Unser Git-Repo mit den Values-Dateien
    - repoURL: git@github.com:DEIN-USERNAME/K8s-Helm.git
      targetRevision: HEAD
      ref: values               # Name für die Referenz: $values
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

### apps/cert-manager.yaml

```yaml
# apps/cert-manager.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://charts.jetstack.io
      chart: cert-manager
      targetRevision: "v1.16.x"    # Version wählen die zu deinem K8s passt
      helm:
        releaseName: cert-manager
        valueFiles:
          - $values/values/cert-manager/values.yaml
    - repoURL: git@github.com:DEIN-USERNAME/K8s-Helm.git
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
      - ServerSideApply=true        # Für große CRDs empfohlen
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

**Warum `ignoreDifferences` von Anfang an?**
cert-manager CRDs haben immer Felder die Kubernetes nach dem Apply modifiziert.
Das führt zu dauerhaftem OutOfSync. Von Anfang an korrekt konfigurieren spart Debugging.

### apps/manifests.yaml

```yaml
# apps/manifests.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: manifests
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:DEIN-USERNAME/K8s-Helm.git
    targetRevision: HEAD
    path: manifests
    directory:
      recurse: true             # Alle Unterordner einschließen
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

### values/traefik/values.yaml

```yaml
# values/traefik/values.yaml
deployment:
  replicas: 1

service:
  type: LoadBalancer            # Bekommt externe IP (MetalLB oder Cloud)

ports:
  web:
    port: 8000
    expose:
      default: true
    exposedPort: 80
    protocol: TCP
  websecure:
    port: 8443
    expose:
      default: true
    exposedPort: 443
    protocol: TCP

ingressRoute:
  dashboard:
    enabled: false              # Dashboard nicht öffentlich

providers:
  kubernetesCRD:
    enabled: true               # IngressRoute CRDs verarbeiten
  kubernetesIngress:
    enabled: true               # Standard Kubernetes Ingress verarbeiten

logs:
  general:
    level: INFO
  access:
    enabled: true
```

### values/cert-manager/values.yaml

```yaml
# values/cert-manager/values.yaml
crds:
  enabled: false                # CRDs separat installieren (siehe Phase 6)

replicaCount: 1

global:
  leaderElection:
    namespace: cert-manager
```

**Warum `crds.enabled: false`?**
cert-manager CRDs sind cluster-weite Ressourcen. Wenn Helm sie verwaltet:
- Droppen bei `helm uninstall` NICHT mit (Helm-Limitation bei CRDs)
- Erzeugen dauerhaften Drift in ArgoCD (verschiedene K8s-Versionen unterstützen
  verschiedene CRD-Features)
- Können bei Helm-Upgrades andere Releases beeinflussen

Besser: CRDs einmalig direkt installieren, Helm kümmert sich nur um die Deployments.

### manifests/cert-manager/clusterissuer.yaml

```yaml
# manifests/cert-manager/clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: deine@email.de       # ← Anpassen! Für Ablauf-Warnungen
    privateKeySecretRef:
      name: letsencrypt-prod    # ACME Account-Key (wird automatisch erstellt)
    solvers:
      - http01:
          ingress:
            class: traefik      # Traefik löst HTTP-01 Challenges
```

### manifests/argocd/ingress.yaml

```yaml
# manifests/argocd/ingress.yaml

# Schritt 1: TLS-Zertifikat anfordern
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
    - argocd.deine-domain.de    # ← Anpassen!

---
# Schritt 2: HTTPS-Route
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`argocd.deine-domain.de`)   # ← Anpassen!
      kind: Rule
      services:
        - name: argocd-server
          port: 80
      middlewares:
        - name: argocd-headers
  tls:
    secretName: argocd-tls

---
# Schritt 3: HTTP → HTTPS Weiterleitung
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-http
  namespace: argocd
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`argocd.deine-domain.de`)   # ← Anpassen!
      kind: Rule
      middlewares:
        - name: redirect-https
      services:
        - name: argocd-server
          port: 80

---
# Middleware: HTTP auf HTTPS weiterleiten
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true             # 301-Redirect (permanent)

---
# Middleware: ArgoCD mitteilen dass HTTPS aktiv ist
# Ohne diesen Header: ArgoCD redirectet selbst → Endlosschleife
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

### Alles committen und pushen

```bash
# Alle Dateien zur Git-Versionskontrolle hinzufügen
git add .
git commit -m "Initial GitOps structure: ArgoCD, Traefik, cert-manager"
git push origin main
```

---

## 6. Phase 4 — ArgoCD installieren

**Zurück auf dem Server.**

Das ist der einzige Dienst den wir manuell installieren.
Danach übernimmt ArgoCD alles andere.

### cert-manager CRDs vorab installieren

Bevor wir cert-manager über Helm/ArgoCD installieren, installieren wir die
CRDs direkt. So vermeiden wir Drift-Probleme von Anfang an:

```bash
# CRDs für die cert-manager Version die wir nutzen wollen
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.crds.yaml
```

```bash
# Prüfen:
kubectl get crds | grep cert-manager
# certificates.cert-manager.io
# certificaterequests.cert-manager.io
# challenges.acme.cert-manager.io
# clusterissuers.cert-manager.io
# issuers.cert-manager.io
# orders.cert-manager.io
```

Diese CRDs bleiben jetzt dauerhaft im Cluster. Helm (und damit ArgoCD) fassen
sie nicht an (`crds.enabled: false` in unseren Values).

### ArgoCD Namespace erstellen

```bash
kubectl create namespace argocd
```

### ArgoCD per Helm installieren

**Warum Helm statt `kubectl apply -f install.yaml`?**

Das offizielle ArgoCD-Manifest (`kubectl apply -f install.yaml`) ist gut für
schnelle Tests, hat aber Nachteile:
- Kein einfaches Upgrade-Management
- Labels/Selektoren könnten sich zwischen Versionen ändern
- Helm verwaltet alle Ressourcen konsistent als eine Einheit

```bash
# ArgoCD Helm Repository hinzufügen
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# ArgoCD installieren
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.x.x \            # Aktuelle stabile Version prüfen: helm search repo argo/argo-cd
  --set configs.params."server\.insecure"=true
  # ↑ ArgoCD läuft intern auf HTTP (Port 80)
  # TLS macht Traefik — ArgoCD muss nicht selbst TLS machen
```

**`server.insecure=true`** bedeutet: ArgoCD-Server läuft auf Port 80 (HTTP) statt 8080/443.
Das klingt unsicher, ist es aber nicht — Traefik macht das TLS davor.
Wenn ArgoCD selbst TLS machen würde UND Traefik TLS macht, gibt es einen TLS-in-TLS
Konflikt. Deshalb: Traefik = TLS, ArgoCD = plain HTTP hinter Traefik.

```bash
# Warten bis ArgoCD bereit ist
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# Prüfen ob alle Pods laufen:
kubectl get pods -n argocd
# NAME                                                READY   STATUS    RESTARTS
# argocd-application-controller-0                     1/1     Running   0
# argocd-applicationset-controller-...                1/1     Running   0
# argocd-dex-server-...                               1/1     Running   0
# argocd-notifications-controller-...                 1/1     Running   0
# argocd-redis-...                                    1/1     Running   0
# argocd-repo-server-...                              1/1     Running   0
# argocd-server-...                                   1/1     Running   0
```

### SSH-Key für ArgoCD hinterlegen

ArgoCD muss auf dein GitHub-Repo zugreifen. Dafür braucht es einen SSH-Key:

```bash
# SSH-Key auf dem Server erstellen (speziell für ArgoCD)
ssh-keygen -t ed25519 -C "argocd@dein-cluster" -f /tmp/argocd_deploy_key -N ""

# Public Key anzeigen → bei GitHub als Deploy Key hinterlegen
cat /tmp/argocd_deploy_key.pub
```

Bei GitHub: Repository → Settings → Deploy Keys → Add deploy key
- Title: "ArgoCD Cluster"
- Key: Inhalt von `argocd_deploy_key.pub`
- Allow write access: Nein (ArgoCD braucht nur Lesezugriff)

```bash
# Private Key als Kubernetes Secret für ArgoCD speichern
kubectl create secret generic argocd-repo-key \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:DEIN-USERNAME/K8s-Helm.git \
  --from-file=sshPrivateKey=/tmp/argocd_deploy_key

# ArgoCD mitteilen dass das ein Repo-Secret ist
kubectl label secret argocd-repo-key \
  argocd.argoproj.io/secret-type=repository \
  -n argocd

# Temporäre Schlüsseldateien löschen
rm /tmp/argocd_deploy_key /tmp/argocd_deploy_key.pub
```

### Initial-Passwort auslesen

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Das Passwort merken — wir brauchen es gleich.

---

## 7. Phase 5 — Helm Charts vorbereiten

Bevor ArgoCD Traefik und cert-manager deployen kann, müssen die Helm Repositories
bekannt sein. ArgoCD lädt Charts direkt — wir müssen aber sicherstellen dass
die Chart-Versionen existieren:

```bash
# Auf dem Server: Helm Repos hinzufügen und verfügbare Versionen prüfen
helm repo add traefik https://helm.traefik.io/traefik
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Aktuelle Traefik-Chart-Version prüfen
helm search repo traefik/traefik
# NAME             CHART VERSION   APP VERSION
# traefik/traefik  33.x.x          v3.x.x       ← Diese Version in apps/traefik.yaml nutzen

# Aktuelle cert-manager-Version prüfen
helm search repo jetstack/cert-manager
# NAME                    CHART VERSION   APP VERSION
# jetstack/cert-manager   v1.16.x         v1.16.x    ← Diese in apps/cert-manager.yaml nutzen
```

**Wichtig:** Die `targetRevision` in den ArgoCD Application-YAMLs muss mit
einer tatsächlich existierenden Chart-Version übereinstimmen.

---

## 8. Phase 6 — GitOps Bootstrap

**Das ist der Moment wo alles zusammenkommt.**

Wir wenden die Root-Application an — das ist der einzige manuelle Schritt nach
der ArgoCD-Installation. Danach verwaltet sich das System selbst.

### DNS-Eintrag setzen

Bevor wir bootstrappen, muss die Domain auf unsere Server-IP zeigen.
Sonst kann Let's Encrypt die HTTP-01 Challenge nicht lösen.

Bei deinem DNS-Provider:
```
Typ:   A
Name:  argocd.deine-domain.de   (oder @, *.deine-domain.de)
Wert:  DEINE-SERVER-IP
TTL:   300 (5 Minuten — klein für schnelle Propagation beim Setup)
```

Propagation prüfen (kann 1-30 Minuten dauern):
```bash
dig argocd.deine-domain.de +short
# → DEINE-SERVER-IP    ← Wenn das erscheint, ist DNS propagiert
```

### LoadBalancer-IP für Traefik vorbereiten

Auf einem Cloud-Server (Hetzner, DigitalOcean, etc.) wird `type: LoadBalancer`
automatisch eine externe IP zuweisen. Auf einem Bare-Metal-Server brauchen wir
**MetalLB**:

```bash
# Nur für Bare-Metal/VPS ohne Cloud-LoadBalancer:
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.x/config/manifests/metallb-native.yaml

# MetalLB konfigurieren — nutzt deine Server-IP als LoadBalancer-IP
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - DEINE-SERVER-IP/32       # ← Anpassen! Nur diese eine IP
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
EOF
```

**Was MetalLB macht:** In der Cloud gibt es einen externen Load Balancer der
Services vom Typ `LoadBalancer` eine externe IP gibt. Auf Bare Metal gibt es
das nicht. MetalLB emuliert das — es weist Services die Server-IP zu.

### Root-App anwenden

```bash
kubectl apply -f bootstrap/root-app.yaml
```

Ab jetzt läuft alles automatisch. Beobachten:

```bash
# In einem separaten Terminal — live beobachten:
watch kubectl get applications -n argocd

# Was du sehen wirst (über die nächsten Minuten):
# NAME           SYNC STATUS   HEALTH STATUS
# root           Synced        Healthy          ← Sofort
# traefik        OutOfSync     Progressing      ← Helm Chart wird geladen
# cert-manager   OutOfSync     Progressing      ← Helm Chart wird geladen
# manifests      OutOfSync     Progressing      ← YAML wird geladen

# Dann (nach 2-5 Minuten):
# NAME           SYNC STATUS   HEALTH STATUS
# root           Synced        Healthy
# traefik        Synced        Healthy          ← Traefik läuft!
# cert-manager   Synced        Healthy          ← cert-manager läuft!
# manifests      Synced        Healthy          ← IngressRoute & Certificate angelegt
```

### Was passiert in diesen Minuten (im Detail)

```
kubectl apply -f bootstrap/root-app.yaml
  │
  ▼
ArgoCD Application "root" erstellt
  │
  ▼
ArgoCD application-controller bemerkt neue Application
  │ Beauftragt repo-server: "Render mir apps/ aus GitHub"
  ▼
repo-server klont GitHub (SSH Key aus Secret)
  │ Findet: traefik.yaml, cert-manager.yaml, manifests.yaml
  ▼
application-controller: "Diese 3 Applications existieren noch nicht → Sync"
  │ kubectl apply für alle 3 Applications
  ▼
3 neue ArgoCD Applications entstehen im Cluster
  │
  ├──► Application "traefik":
  │      repo-server: Helm Chart von helm.traefik.io laden
  │      Values aus GitHub lesen
  │      Helm template rendern
  │      kubectl apply → Traefik Deployment, Service, CRDs...
  │      Traefik Pod startet
  │      LoadBalancer-Service bekommt externe IP
  │
  ├──► Application "cert-manager":
  │      repo-server: Helm Chart von charts.jetstack.io laden
  │      Values aus GitHub lesen (crds.enabled: false)
  │      Helm template rendern
  │      kubectl apply → cert-manager Deployment, Service...
  │      cert-manager Pod startet
  │
  └──► Application "manifests":
         repo-server: YAML aus GitHub lesen
         kubectl apply → ClusterIssuer, Certificate, IngressRoutes, Middlewares
         cert-manager sieht Certificate-Ressource → startet ACME-Prozess
         Traefik sieht IngressRoute → konfiguriert Routing
         Let's Encrypt Challenge → Traefik antwortet → Zertifikat ausgestellt
         Secret "argocd-tls" wird erstellt
         Traefik nutzt Secret für TLS
```

---

## 9. Phase 7 — ArgoCD per HTTPS erreichbar machen

Nach Phase 6 sollte theoretisch alles laufen. Prüfen:

```bash
# Zertifikat-Status
kubectl get certificate argocd-tls -n argocd
# NAME         READY   SECRET       AGE
# argocd-tls   True    argocd-tls   2m   ← "True" = Zertifikat ausgestellt ✓

# IngressRoutes
kubectl get ingressroute.traefik.io -n argocd
# NAME          AGE
# argocd        2m
# argocd-http   2m

# Traefik Service — hat er eine externe IP?
kubectl get svc -n traefik
# NAME      TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# traefik   LoadBalancer   10.x.x.x      DEINE-SERVER-IP  80:xxx/TCP,443:xxx/TCP

# Erreichbarkeit testen
curl -s -o /dev/null -w "%{http_code}" https://argocd.deine-domain.de
# → 200   ← ArgoCD UI erreichbar ✓
```

### Einloggen

```bash
# Passwort nochmal auslesen falls vergessen
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Browser: `https://argocd.deine-domain.de`
- Username: `admin`
- Password: Ausgabe des Befehls oben

### Initial-Passwort nach erstem Login ändern

Das Initial-Passwort ist automatisch generiert und sollte geändert werden:

In der ArgoCD UI: User Info (oben rechts) → Update Password

Oder per CLI:
```bash
# ArgoCD CLI installieren
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Einloggen
argocd login argocd.deine-domain.de \
  --username admin \
  --password DEIN-PASSWORT

# Passwort ändern
argocd account update-password

# Initial-Secret löschen (enthält das alte Passwort, nicht mehr gebraucht)
kubectl delete secret argocd-initial-admin-secret -n argocd
```

---

## 10. Phase 8 — Verifizierung

Alles prüfen bevor man es "fertig" nennt.

### Cluster-Gesundheit

```bash
# Alle Nodes Ready?
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# k8s-main   Ready    control-plane   30m   v1.31.x   ← Ready ✓

# Alle System-Pods OK?
kubectl get pods -n kube-system
# Alle sollten "Running" sein

# Alle ArgoCD Pods OK?
kubectl get pods -n argocd
# Alle sollten "Running" sein, READY = 1/1

# Alle Traefik Pods OK?
kubectl get pods -n traefik
# traefik-xxx   1/1   Running   0   ← ✓

# Alle cert-manager Pods OK?
kubectl get pods -n cert-manager
# cert-manager-xxx          1/1   Running   0   ← ✓
# cert-manager-cainjector   1/1   Running   0   ← ✓
# cert-manager-webhook      1/1   Running   0   ← ✓
```

### ArgoCD Applications

```bash
kubectl get applications -n argocd
# NAME           SYNC STATUS   HEALTH STATUS
# cert-manager   Synced        Healthy        ✓
# manifests      Synced        Healthy        ✓
# root           Synced        Healthy        ✓
# traefik        Synced        Healthy        ✓
```

Alle müssen `Synced` und `Healthy` sein.

### TLS-Zertifikat

```bash
# cert-manager Zertifikat-Status
kubectl get certificate -n argocd
# NAME         READY   SECRET       AGE
# argocd-tls   True    argocd-tls   10m   ← True ✓

# TLS-Secret vorhanden?
kubectl get secret argocd-tls -n argocd
# NAME         TYPE                DATA   AGE
# argocd-tls   kubernetes.io/tls   2      10m   ← 2 = tls.crt und tls.key ✓

# Zertifikat-Details prüfen (Ablaufdatum, Domain)
kubectl get secret argocd-tls -n argocd \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | \
  openssl x509 -noout -text | \
  grep -A2 "Subject:\|DNS:\|Not After"
```

### End-to-End Test

```bash
# HTTP → HTTPS Redirect funktioniert?
curl -v http://argocd.deine-domain.de 2>&1 | grep "Location:"
# → Location: https://argocd.deine-domain.de/   ← 301 Redirect ✓

# HTTPS funktioniert?
curl -s -o /dev/null -w "%{http_code}" https://argocd.deine-domain.de
# → 200   ✓

# TLS-Zertifikat gültig?
curl -v https://argocd.deine-domain.de 2>&1 | grep "SSL certificate verify"
# → SSL certificate verify ok.   ✓
```

### GitOps funktioniert?

Den kompletten Kreislauf testen:

```bash
# Lokal: Eine neue Test-Application erstellen
cat <<EOF > apps/test-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: "18.x"
    helm:
      releaseName: test-nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

git add apps/test-app.yaml
git commit -m "test: add nginx test app"
git push

# Warten und prüfen (max 3 Minuten):
kubectl get applications -n argocd -w
# → test   Synced   Healthy   ← GitOps funktioniert ✓

# Test-App wieder entfernen
git rm apps/test-app.yaml
git commit -m "test: remove nginx test app"
git push

# ArgoCD löscht die App automatisch (prune: true)
kubectl get applications -n argocd
# → "test" ist weg ✓
```

---

## 11. Reihenfolge und warum sie wichtig ist

Die Reihenfolge der Schritte ist nicht zufällig. Jede Phase hängt von der
vorherigen ab:

```
Phase 1 (Server)
  │ Muss vor K8s sein: Kernel-Module, Swap, containerd
  ▼
Phase 2 (Kubernetes)
  │ Muss vor allem anderen sein: CNI, Node Ready
  ▼
Phase 3 (Git Repo)
  │ Muss vor Bootstrap sein: ArgoCD braucht das Repo um sich zu konfigurieren
  ▼
Phase 4 (ArgoCD)
  │ Muss vor cert-manager CRDs? NEIN — aber vor dem Bootstrap
  │ cert-manager CRDs können parallel installiert werden
  ▼
Phase 5 (Helm Repos prüfen)
  │ Optional aber empfohlen: Versionen verifizieren bevor ArgoCD sie lädt
  ▼
Phase 6 (Bootstrap)
  │ DNS muss propagiert sein: cert-manager braucht erreichbare Domain
  │ Internes: root-app → traefik + cert-manager + manifests
  ▼
Phase 7 (Verifikation)
  └── Alles prüfen
```

**Was passiert wenn man die Reihenfolge ignoriert:**

| Falsche Reihenfolge | Fehler |
|---------------------|--------|
| K8s ohne Swap-Deaktivierung | Kubelet startet nicht |
| K8s ohne CNI | Alle Pods bleiben "Pending" |
| Bootstrap ohne DNS | cert-manager Challenge schlägt fehl |
| ArgoCD ohne SSH-Key | repo-server kann GitHub nicht erreichen |
| cert-manager ohne CRDs | Certificate-Ressource nicht erkannt |

---

## 12. Die komplette Dateistruktur

```
K8s/
│
├── bootstrap/
│   └── root-app.yaml
│       → Einmalig manuell anwenden
│       → Startet den gesamten GitOps-Prozess
│       → Überwacht: apps/
│
├── apps/
│   ├── traefik.yaml
│   │   → ArgoCD Application für Traefik Helm Chart
│   │   → Multi-Source: helm.traefik.io + $values
│   │   → Namespace: traefik
│   │
│   ├── cert-manager.yaml
│   │   → ArgoCD Application für cert-manager Helm Chart
│   │   → Multi-Source: charts.jetstack.io + $values
│   │   → Namespace: cert-manager
│   │   → ignoreDifferences für CRD-Drift
│   │
│   └── manifests.yaml
│       → ArgoCD Application für eigene YAML-Dateien
│       → Source: manifests/ Ordner im Repo
│       → Namespace: argocd (Ziel für ArgoCD-eigene Ressourcen)
│
├── values/
│   ├── traefik/
│   │   └── values.yaml
│   │       → Helm Values für Traefik
│   │       → Referenziert in apps/traefik.yaml als $values
│   │
│   └── cert-manager/
│       └── values.yaml
│           → Helm Values für cert-manager
│           → crds.enabled: false ← wichtig!
│
├── manifests/
│   ├── argocd/
│   │   └── ingress.yaml
│   │       → Certificate (TLS für ArgoCD UI)
│   │       → IngressRoute HTTPS
│   │       → IngressRoute HTTP (Redirect)
│   │       → Middleware: redirect-https
│   │       → Middleware: argocd-headers
│   │
│   └── cert-manager/
│       └── clusterissuer.yaml
│           → ClusterIssuer: letsencrypt-prod
│           → HTTP-01 Challenge via Traefik
│
└── docs/
    ├── SETUP-DOKU.md          → Grundlagen und Konzepte
    ├── FEHLER-DOKU.md         → Was schiefgehen kann
    ├── WIE-ALLES-ZUSAMMENHAENGT.md → Prozessketten
    └── HOW-TO-CREATE.md       → Diese Datei
```

---

## Kurzreferenz: Alle Befehle in Reihenfolge

```bash
# === PHASE 1: Server ===
swapoff -a && sed -i '/swap/d' /etc/fstab
modprobe overlay br_netfilter
apt install -y containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

# === PHASE 2: Kubernetes ===
apt install -y kubeadm kubelet kubectl
apt-mark hold kubeadm kubelet kubectl
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=SERVER-IP
mkdir -p ~/.kube && cp /etc/kubernetes/admin.conf ~/.kube/config
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# === PHASE 3: Git ===
# Lokal: Dateien erstellen, committen, pushen (siehe Abschnitt 5)

# === PHASE 4: ArgoCD ===
# cert-manager CRDs vorab:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.crds.yaml
# ArgoCD:
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set configs.params."server\.insecure"=true
# SSH-Key für Repo-Zugriff:
ssh-keygen -t ed25519 -f /tmp/argocd_key -N ""
# → Public Key bei GitHub als Deploy Key hinterlegen
kubectl create secret generic argocd-repo-key \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:USERNAME/K8s-Helm.git \
  --from-file=sshPrivateKey=/tmp/argocd_key
kubectl label secret argocd-repo-key argocd.argoproj.io/secret-type=repository -n argocd
rm /tmp/argocd_key /tmp/argocd_key.pub

# === PHASE 5: DNS setzen ===
# Beim DNS-Provider: A-Record für argocd.deine-domain.de → SERVER-IP
# Prüfen: dig argocd.deine-domain.de +short

# === PHASE 6: Bootstrap ===
kubectl apply -f bootstrap/root-app.yaml
watch kubectl get applications -n argocd
# → Warten bis alle Synced + Healthy

# === PHASE 7: Verifizierung ===
kubectl get nodes
kubectl get pods -A
kubectl get applications -n argocd
kubectl get certificate argocd-tls -n argocd
curl -s -o /dev/null -w "%{http_code}" https://argocd.deine-domain.de
# → 200 ✓
```

---

*Diese Dokumentation beschreibt das ideale Setup ohne vorhandene Systeme oder
Altlasten. Abweichungen (bestehende Cluster, andere K8s-Versionen, Cloud-Provider)
können einzelne Schritte ändern — die Prinzipien bleiben gleich.*
