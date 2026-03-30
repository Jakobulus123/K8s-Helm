# YAML & Helm Charts lernen — Von Null bis eigene App deployen

> Diese Dokumentation erklärt YAML und Helm von Grund auf.
> Alle Beispiele kommen aus diesem echten Setup — nichts ist erfunden.
> Am Ende kannst du selbst eine neue App anlegen und in den Cluster deployen.

---

## Inhaltsverzeichnis

1. [YAML — Die Sprache von Kubernetes](#1-yaml--die-sprache-von-kubernetes)
2. [Kubernetes YAML — Wie ein Manifest aufgebaut ist](#2-kubernetes-yaml--wie-ein-manifest-aufgebaut-ist)
3. [Die wichtigsten Kubernetes-Objekte in YAML](#3-die-wichtigsten-kubernetes-objekte-in-yaml)
4. [Was ist Helm?](#4-was-ist-helm)
5. [Helm Chart Struktur — Was ist wo und warum](#5-helm-chart-struktur--was-ist-wo-und-warum)
6. [values.yaml — Das Herzstück der Konfiguration](#6-valuesyaml--das-herzstück-der-konfiguration)
7. [ArgoCD Application — Der Kleber zwischen Git und Cluster](#7-argocd-application--der-kleber-zwischen-git-und-cluster)
8. [Das App-of-Apps Pattern in diesem Setup](#8-das-app-of-apps-pattern-in-diesem-setup)
9. [Schritt für Schritt: Neue App anlegen](#9-schritt-für-schritt-neue-app-anlegen)
10. [Echte Beispiele aus diesem Repo erklärt](#10-echte-beispiele-aus-diesem-repo-erklärt)
11. [Häufige Fehler und wie du sie erkennst](#11-häufige-fehler-und-wie-du-sie-erkennst)
12. [Spickzettel — Das Wichtigste auf einen Blick](#12-spickzettel--das-wichtigste-auf-einen-blick)

---

## 1. YAML — Die Sprache von Kubernetes

### Was ist YAML?

YAML steht für "YAML Ain't Markup Language". Es ist ein Dateiformat um
**strukturierte Daten als lesbaren Text** darzustellen — ähnlich wie JSON,
aber viel leserfreundlicher für Menschen.

Kubernetes nutzt YAML um zu beschreiben was im Cluster laufen soll.

### Die 3 Grundbausteine von YAML

#### Baustein 1: Key-Value Paare (Schlüssel-Wert)

Das ist das einfachste — ein Name und ein Wert, getrennt durch Doppelpunkt + Leerzeichen:

```yaml
name: meine-app
version: 1.0
replicas: 3
aktiv: true
```

**Wichtige Regel:** Nach dem Doppelpunkt MUSS ein Leerzeichen kommen.
`name:meine-app` ist FALSCH. `name: meine-app` ist RICHTIG.

#### Baustein 2: Einrückung (Hierarchie)

YAML nutzt Leerzeichen um zu zeigen was zu was gehört.
**Immer 2 Leerzeichen** (niemals Tabs!):

```yaml
person:          # "person" ist eine Gruppe
  name: Jakob    # gehört zu "person"
  alter: 25      # gehört zu "person"
  adresse:       # eine Untergruppe von "person"
    stadt: Berlin    # gehört zu "adresse"
    land: Deutschland  # gehört zu "adresse"
```

Das ist wie eine Baumstruktur:
```
person
├── name: Jakob
├── alter: 25
└── adresse
    ├── stadt: Berlin
    └── land: Deutschland
```

#### Baustein 3: Listen (mit Bindestrich)

Wenn mehrere Dinge zur selben Gruppe gehören, nutzt du einen Bindestrich:

```yaml
früchte:
  - Apfel
  - Banane
  - Orange

# Oder Listen von Gruppen:
container:
  - name: meine-app        # Erstes Element der Liste
    image: nginx:1.25
    port: 8080
  - name: sidecar          # Zweites Element der Liste
    image: busybox:latest
    port: 9090
```

### Datentypen in YAML

```yaml
# Text (String)
name: "meine-app"         # Mit Anführungszeichen
name: meine-app           # Ohne Anführungszeichen — beides funktioniert
beschreibung: "Hallo Welt"  # Mit Anführungszeichen wenn Sonderzeichen drin

# Zahlen
replicas: 3               # Ganze Zahl (Integer)
version: 1.5              # Kommazahl (Float)
port: 8080

# Wahrheitswerte (Boolean)
aktiv: true
debug: false

# Null (nichts)
wert: null
wert: ~           # Alternative Schreibweise für null

# Mehrzeiliger Text
beschreibung: |
  Diese Beschreibung geht
  über mehrere Zeilen.
  Zeilenumbrüche bleiben erhalten.

kurztext: >
  Diese Zeilen werden
  zu einer einzigen Zeile
  zusammengefasst.
```

### YAML-Fallen die Anfänger treffen

```yaml
# FEHLER 1: Tabs statt Leerzeichen
spec:
	replicas: 3    # ← Tab! YAML mag keine Tabs → Fehler

# RICHTIG:
spec:
  replicas: 3    # ← 2 Leerzeichen

# FEHLER 2: Kein Leerzeichen nach Doppelpunkt
name:meine-app   # ← FALSCH
name: meine-app  # ← RICHTIG

# FEHLER 3: Falsche Einrückung
spec:
  replicas: 3
   containers:    # ← 3 Leerzeichen statt 2 → Fehler
  - name: app

# RICHTIG:
spec:
  replicas: 3
  containers:
  - name: app

# FEHLER 4: Anführungszeichen vergessen bei Sonderzeichen
version: 1.0.0   # YAML interpretiert das als Zahl — Fehler!
version: "1.0.0" # ← RICHTIG: als Text kennzeichnen

# FEHLER 5: Boolesche Falle
aktiv: yes       # YAML interpretiert "yes" als true — Vorsicht!
aktiv: "yes"     # ← Als Text, nicht als Boolean
```

### Mehrere Dokumente in einer Datei

Mit `---` kannst du mehrere YAML-Dokumente in einer Datei trennen.
Das nutzt Kubernetes oft um mehrere Objekte in einer Datei zu definieren:

```yaml
---
# Erstes Objekt
apiVersion: v1
kind: Service
metadata:
  name: meine-app
---
# Zweites Objekt
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app
```

---

## 2. Kubernetes YAML — Wie ein Manifest aufgebaut ist

Jedes Kubernetes-Objekt — egal ob Pod, Deployment, Service oder was auch immer —
hat **immer dieselbe Grundstruktur**:

```yaml
apiVersion: ...    # Welche API-Version?
kind: ...          # Was für ein Objekt?
metadata:          # Informationen ÜBER das Objekt
  name: ...
  namespace: ...
  labels: ...
spec:              # Was das Objekt tun soll (der eigentliche Inhalt)
  ...
```

Diese 4 Felder sind **immer vorhanden**. Lass uns jedes einzeln verstehen.

### apiVersion — Welche API-Version?

Kubernetes hat viele APIs. `apiVersion` sagt welche Gruppe und Version du nutzt:

```yaml
apiVersion: v1                    # Kern-Objekte: Pod, Service, ConfigMap, Secret
apiVersion: apps/v1               # App-Objekte: Deployment, StatefulSet, DaemonSet
apiVersion: networking.k8s.io/v1  # Netzwerk-Objekte: Ingress, NetworkPolicy
apiVersion: argoproj.io/v1alpha1  # ArgoCD-Objekte: Application
apiVersion: cert-manager.io/v1    # cert-manager Objekte: Certificate, ClusterIssuer
apiVersion: traefik.io/v1alpha1   # Traefik Objekte: IngressRoute, Middleware
```

**Woher weiß ich die richtige apiVersion?**
- Kubernetes-Doku oder `kubectl api-resources` im Terminal
- In diesem Setup: schau einfach in bestehende Dateien

### kind — Was für ein Objekt?

```yaml
kind: Pod              # Ein einzelner Container-Gruppe
kind: Deployment       # Verwaltet mehrere Pod-Kopien
kind: Service          # Macht Pods erreichbar
kind: ConfigMap        # Konfigurationsdaten
kind: Secret           # Geheime Daten (Passwörter, Certs)
kind: Namespace        # Logische Trennung im Cluster
kind: Application      # ArgoCD-spezifisch
kind: IngressRoute     # Traefik-spezifisch
kind: Certificate      # cert-manager-spezifisch
```

### metadata — Informationen über das Objekt

```yaml
metadata:
  name: meine-app           # PFLICHT: Name des Objekts (eindeutig im Namespace)
  namespace: production     # In welchem Namespace (Standard: "default")
  labels:                   # Schlüssel-Wert Tags zum Gruppieren/Finden
    app: meine-app
    version: "1.0"
    umgebung: production
  annotations:              # Wie Labels, aber für Metadaten — nicht zum Filtern
    beschreibung: "Meine Hauptanwendung"
    kontakt: "jakob@beispiel.de"
```

**Labels vs. Annotations:**
- **Labels** = für Kubernetes selbst — Services finden Pods über Labels, Selektoren nutzen Labels
- **Annotations** = für Menschen und externe Tools — Notizen, URLs, Versionsinformationen

### spec — Was das Objekt tun soll

Das ist der Hauptteil und ist **je nach `kind` völlig unterschiedlich**.
Ein Deployment-spec sieht anders aus als ein Service-spec.
Das lernst du am besten durch Beispiele — kommt in Abschnitt 3.

### status — Was das Objekt gerade macht (automatisch)

`status` wirst du in YAML-Dateien nie selbst schreiben.
Kubernetes füllt das automatisch aus um den aktuellen Zustand zu beschreiben:

```yaml
status:
  readyReplicas: 3      # 3 Pods laufen gerade
  availableReplicas: 3
  conditions:
  - type: Available
    status: "True"
```

Du siehst `status` wenn du `kubectl get deployment meine-app -o yaml` ausführst.

---

## 3. Die wichtigsten Kubernetes-Objekte in YAML

### Deployment — Die häufigste Art eine App zu deployen

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app           # Name des Deployments
  namespace: meine-app      # In diesem Namespace
  labels:
    app: meine-app
spec:
  replicas: 2               # Wie viele Pod-Kopien sollen laufen?
  selector:                 # Welche Pods "gehören" zu diesem Deployment?
    matchLabels:
      app: meine-app        # Pods mit diesem Label werden verwaltet
  template:                 # Vorlage für jeden Pod der erstellt wird
    metadata:
      labels:
        app: meine-app      # MUSS zum selector.matchLabels passen!
    spec:                   # Was soll im Pod laufen?
      containers:
      - name: meine-app     # Name des Containers (beliebig)
        image: nginx:1.25   # Welches Docker-Image?
        ports:
        - containerPort: 80 # Auf welchem Port lauscht die App?
        resources:          # Wie viel CPU und RAM darf der Container nutzen?
          requests:         # Mindestens reserviert
            memory: "64Mi"  # 64 Megabyte RAM
            cpu: "50m"      # 50 Millicores (= 0.05 CPU-Kerne)
          limits:           # Maximal erlaubt
            memory: "128Mi"
            cpu: "200m"
        env:                # Umgebungsvariablen für den Container
        - name: APP_ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:   # Wert aus einem Secret holen
              name: db-secret
              key: password
```

**Warum `selector` und `template.metadata.labels` identisch sein müssen:**
Das Deployment nutzt den Selector um zu wissen welche Pods es verwaltet.
Wenn die Labels nicht übereinstimmen, "sieht" das Deployment seine eigenen Pods nicht.

### Service — Macht Pods erreichbar

```yaml
apiVersion: v1
kind: Service
metadata:
  name: meine-app         # Name des Services (wird auch DNS-Name!)
  namespace: meine-app
spec:
  selector:
    app: meine-app        # Traffic geht an alle Pods mit diesem Label
  ports:
  - port: 80              # Auf diesem Port ist der SERVICE erreichbar
    targetPort: 80        # An diesen Port im POD wird weitergeleitet
    protocol: TCP
  type: ClusterIP         # Nur intern erreichbar (Standard)
```

**Der Selector ist der Schlüssel:** Der Service schickt Traffic an alle Pods
deren Labels mit dem Selector übereinstimmen. Wenn du einen neuen Pod startest
mit `app: meine-app` bekommt er sofort Traffic — automatisch.

### Namespace — Logische Trennung

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: meine-app
```

Das war es schon. Namespaces sind einfach — sie sind nur Container für andere Objekte.

### ConfigMap — Konfigurationsdaten

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: meine-app-config
  namespace: meine-app
data:
  # Einfache Werte
  LOG_LEVEL: "info"
  APP_PORT: "8080"
  # Ganze Dateien als Wert
  config.yaml: |
    server:
      port: 8080
    database:
      host: db-service
      port: 5432
```

### Secret — Geheime Daten

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: meine-app-secrets
  namespace: meine-app
type: Opaque
data:
  # Werte MÜSSEN base64-kodiert sein!
  # echo -n "meinPasswort" | base64  → bWVpblBhc3N3b3J0
  DB_PASSWORD: bWVpblBhc3N3b3J0
  API_KEY: c2VocmdlaGVpbWVyc2NobHVzc2Vs
```

**Tipp:** Um einen Wert zu kodieren:
```bash
echo -n "meinPasswort" | base64
# Ausgabe: bWVpblBhc3N3b3J0
```

Um einen kodierten Wert zu lesen:
```bash
echo "bWVpblBhc3N3b3J0" | base64 -d
# Ausgabe: meinPasswort
```

### Alles zusammen — Komplettes Beispiel

So sieht eine vollständige App-Konfiguration in einer Datei aus:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: meine-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app
  namespace: meine-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: meine-app
  template:
    metadata:
      labels:
        app: meine-app
    spec:
      containers:
      - name: meine-app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: meine-app
  namespace: meine-app
spec:
  selector:
    app: meine-app
  ports:
  - port: 80
    targetPort: 80
```

---

## 4. Was ist Helm?

### Das Problem ohne Helm

Stell dir vor du willst dieselbe App in 3 verschiedenen Umgebungen deployen:
- **Development** (1 Replica, wenig RAM, kein TLS)
- **Staging** (2 Replicas, mittleres RAM, TLS)
- **Production** (5 Replicas, viel RAM, TLS, Monitoring)

Ohne Helm: Du kopierst die YAML-Dateien 3x und änderst manuell alle Werte.
Wenn du jetzt den Image-Namen ändern willst: 3x ändern, 3x vergessen kann.

**Helm löst das** mit Templates und Variablen.

### Was Helm ist

Helm ist der **Paket-Manager für Kubernetes** — wie apt für Ubuntu oder brew für Mac.

```
apt install nginx           → installiert nginx auf Linux
helm install traefik ...    → installiert Traefik im Kubernetes-Cluster
```

Ein Helm **Chart** ist ein Paket das:
1. Die YAML-Templates enthält (mit Variablen statt fester Werte)
2. Eine `values.yaml` Datei hat (die Standard-Werte für alle Variablen)
3. Metadaten über das Paket enthält (Name, Version, Beschreibung)

### Der Unterschied: Chart vs. Release

```
Chart  = Das Paket (die Vorlage)        → wie eine DVD mit Software
Release = Eine installierte Instanz     → wie die installierte Software

Beispiel:
  helm install traefik-prod traefik/traefik --values production.yaml
              │              │               │
              Release-Name   Chart           Deine Werte
```

Du kannst denselben Chart mehrmals installieren mit verschiedenen Werten:
```
helm install traefik-dev  traefik/traefik --values dev.yaml
helm install traefik-prod traefik/traefik --values prod.yaml
```
→ Zwei verschiedene Traefik-Instanzen aus demselben Chart.

### Was Helm im Hintergrund macht

```
Du gibst Helm:
  1. Chart (die Templates)
  2. Deine values.yaml (deine Werte)

Helm macht:
  1. Nimmt jedes Template
  2. Fügt deine Werte ein (wo {{ .Values.xxx }} steht)
  3. Rendert fertiges YAML
  4. Führt kubectl apply auf das fertige YAML aus

Ergebnis: Kubernetes-Objekte laufen im Cluster
```

---

## 5. Helm Chart Struktur — Was ist wo und warum

Ein Helm Chart ist ein Ordner mit dieser Struktur:

```
mein-chart/
│
├── Chart.yaml          ← Metadaten (Name, Version, Beschreibung)
├── values.yaml         ← Standard-Werte (die du überschreiben kannst)
│
├── templates/          ← Die YAML-Templates mit Variablen
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    ← Wiederverwendbare Template-Teile
│   └── NOTES.txt       ← Text der nach Installation angezeigt wird
│
└── charts/             ← Abhängige Charts (Sub-Charts)
    └── postgresql/     ← z.B. PostgreSQL als Abhängigkeit
```

### Chart.yaml — Die Visitenkarte des Charts

```yaml
apiVersion: v2              # Helm-Version (immer v2 für Helm 3)
name: meine-app             # Name des Charts
description: Meine tolle Anwendung
type: application           # "application" oder "library"
version: 0.1.0              # Chart-Version (du selbst vergibst das)
appVersion: "1.25.0"        # Version der App die deployed wird (z.B. nginx Version)

# Optionale Felder:
keywords:
  - webserver
  - nginx
maintainers:
  - name: Jakob
    email: jakob@beispiel.de
```

### templates/ — Die Template-Dateien

Hier liegt das eigentliche YAML — aber mit **Variablen** statt fester Werte:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}          # ← Variable: Name des Helm-Releases
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Values.app.name }}      # ← Variable aus values.yaml
spec:
  replicas: {{ .Values.replicas }}   # ← Variable aus values.yaml
  template:
    spec:
      containers:
      - name: {{ .Values.app.name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
```

**Variablen-Syntax in Helm-Templates:**
```
{{ .Release.Name }}         → Name des Helm-Releases (z.B. "traefik")
{{ .Release.Namespace }}    → Namespace wo deployed wird
{{ .Chart.Name }}           → Name des Charts
{{ .Chart.Version }}        → Version des Charts
{{ .Values.irgendwas }}     → Wert aus values.yaml
```

### _helpers.tpl — Wiederverwendbare Teile

Dateinamen die mit `_` anfangen werden nicht als Kubernetes-Manifest behandelt.
`_helpers.tpl` enthält Template-Funktionen die du überall nutzen kannst:

```
{{- define "meine-app.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}
```

Dann im Deployment: `name: {{ include "meine-app.fullname" . }}`

Das brauchst du am Anfang nicht verstehen — nur wissen dass es diese Datei gibt.

---

## 6. values.yaml — Das Herzstück der Konfiguration

Die `values.yaml` ist **die wichtigste Datei** wenn du mit fremden Charts arbeitest.
Du änderst fast nie die Templates — du änderst nur die Werte.

### Standard values.yaml eines Charts

```yaml
# Das sind die Default-Werte des Charts
# Du kannst jeden Wert in deiner eigenen values.yaml überschreiben

replicas: 1                 # Standard: 1 Kopie

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent  # Image nur pullen wenn nicht lokal vorhanden

service:
  type: ClusterIP           # Standard: nur intern erreichbar
  port: 80

ingress:
  enabled: false            # Standard: kein Ingress

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "200m"

env: {}                     # Keine Umgebungsvariablen standardmäßig
```

### Deine eigene values.yaml (überschreibt nur was du änderst)

Du musst NICHT alle Werte kopieren. Du schreibst nur was du anders haben willst:

```yaml
# Meine values.yaml für Production
# Alles was hier nicht steht bleibt beim Standard aus dem Chart

replicas: 3             # Überschreibe: ich will 3 statt 1

service:
  type: LoadBalancer    # Überschreibe: ich will einen LoadBalancer

resources:
  requests:
    memory: "256Mi"     # Überschreibe: ich brauche mehr RAM
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# replicas, image, ingress, env etc. nutzen die Chart-Defaults
```

Helm macht dann: Chart-Defaults + Deine Werte = finale Konfiguration.

### Echtes Beispiel: Traefik values.yaml aus diesem Repo

```yaml
# values/traefik/values.yaml — das ist deine echte Datei!

deployment:
  replicas: 1              # 1 Traefik-Pod soll laufen

service:
  type: LoadBalancer       # Traefik braucht eine externe IP

ports:
  web:
    port: 8000             # Intern lauscht Traefik auf Port 8000
    expose:
      default: true
    exposedPort: 80        # Nach außen ist es Port 80 (HTTP)
    protocol: TCP
  websecure:
    port: 8443             # Intern Port 8443
    expose:
      default: true
    exposedPort: 443       # Nach außen Port 443 (HTTPS)
    protocol: TCP
    tls:
      enabled: true        # TLS aktivieren

ingressRoute:
  dashboard:
    enabled: false         # Dashboard nicht öffentlich zugänglich

providers:
  kubernetesCRD:
    enabled: true          # Traefik liest IngressRoute-Objekte
  kubernetesIngress:
    enabled: true          # Traefik liest auch Standard-Ingress-Objekte

logs:
  general:
    level: INFO
  access:
    enabled: true          # Access-Logs aktivieren

metrics:
  prometheus:
    enabled: true          # Prometheus-Metriken aktivieren
```

**Was du hier siehst:** Du musst nicht wissen wie Traefik intern funktioniert.
Der Chart-Entwickler hat alles vorbereitet. Du sagst nur: "1 Replica, LoadBalancer,
TLS an, Dashboard aus." Der Chart kümmert sich um den Rest.

### Wie findest du heraus welche Werte ein Chart hat?

**Methode 1:** Offizielle Dokumentation des Charts (immer erste Wahl)

**Methode 2:** Die default values.yaml des Charts anzeigen:
```bash
helm show values traefik/traefik
helm show values jetstack/cert-manager
```

**Methode 3:** Auf Artifact Hub suchen — artifacthub.io ist das Verzeichnis
aller öffentlichen Helm Charts.

---

## 7. ArgoCD Application — Der Kleber zwischen Git und Cluster

Bisher haben wir gelernt: YAML beschreibt was wir wollen, Helm rendert es.
Aber wer deployt es eigentlich? In diesem Setup: **ArgoCD**.

ArgoCD hat sein eigenes Kubernetes-Objekt: die **Application**.
Diese Application sagt ArgoCD: "Schau hier in Git nach, deploy das dort hin."

### Aufbau einer ArgoCD Application — Zeile für Zeile

```yaml
apiVersion: argoproj.io/v1alpha1   # ArgoCD API
kind: Application                  # ArgoCD Application-Objekt
metadata:
  name: meine-app                  # Name dieser Application in ArgoCD
  namespace: argocd                # IMMER "argocd" — hier lebt ArgoCD
  finalizers:
    - resources-finalizer.argocd.argoproj.io
    # ↑ Wenn diese Application gelöscht wird,
    #   soll ArgoCD auch alle deployte Ressourcen löschen
spec:
  project: default                 # ArgoCD-Projekt (für RBAC, Standard: "default")

  source:                          # WO kommen die Manifeste her?
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD           # Welcher Branch? HEAD = main/master
    path: manifests/meine-app      # Welcher Ordner im Git-Repo?

  destination:                     # WOHIN wird deployed?
    server: https://kubernetes.default.svc   # Dieser Cluster
    namespace: meine-app           # In diesen Namespace

  syncPolicy:
    automated:
      prune: true                  # Lösche K8s-Objekte die nicht mehr in Git sind
      selfHeal: true               # Überschreibe manuelle kubectl-Änderungen
    syncOptions:
      - CreateNamespace=true       # Erstelle den Namespace falls er nicht existiert
```

### Source: Git-Ordner vs. Helm-Chart

Es gibt zwei Hauptvarianten:

**Variante 1: Rohes YAML aus Git-Ordner**
```yaml
source:
  repoURL: git@github.com:Jakobulus123/K8s-Helm.git
  targetRevision: HEAD
  path: manifests/meine-app    # Ordner mit YAML-Dateien
```
→ ArgoCD nimmt alle `.yaml` Dateien in diesem Ordner und wendet sie an.

**Variante 2: Helm-Chart aus Chart-Repository + eigene values**
```yaml
sources:                       # ← "sources" (Plural!) für mehrere Quellen
  - repoURL: https://helm.traefik.io/traefik
    chart: traefik             # Chart-Name
    targetRevision: "33.x"     # Chart-Version
    helm:
      releaseName: traefik
      valueFiles:
        - $values/values/traefik/values.yaml   # ← kommt aus der zweiten Quelle!
  - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    ref: values                # ← Diese Quelle wird als "$values" Referenz
```

**Warum 2 Quellen?** Das Chart kommt von `helm.traefik.io` (externes Repo),
aber deine values.yaml liegt in deinem GitHub-Repo. ArgoCD muss beides zusammenführen.

### Echte Beispiele aus diesem Repo

**cert-manager (Helm-Chart mit eigenen Values):**
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
      targetRevision: "v1.20.0"          # Exakte Version — wichtig für Stabilität
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
```

**manifests (rohes YAML aus Git-Ordner):**
```yaml
# apps/manifests.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: manifests
  namespace: argocd
spec:
  source:
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    path: manifests              # Gesamter manifests/-Ordner
    directory:
      recurse: true              # Alle Unterordner auch durchsuchen
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

---

## 8. Das App-of-Apps Pattern in diesem Setup

### Wie dieses Repo organisiert ist

```
K8s-Helm/
│
├── bootstrap/
│   └── root-app.yaml          ← Einmalig manuell anwenden (einmal!)
│
├── apps/                      ← Jede Datei hier = eine ArgoCD Application
│   ├── traefik.yaml
│   ├── cert-manager.yaml
│   └── manifests.yaml
│
├── values/                    ← Deine Helm-Values
│   ├── traefik/
│   │   └── values.yaml
│   └── cert-manager/
│       └── values.yaml
│
└── manifests/                 ← Rohes YAML (kein Helm)
    └── argocd/
        └── ingress.yaml
```

### Die Kette die das alles antreibt

```
1. Du führst EINMALIG aus:
   kubectl apply -f bootstrap/root-app.yaml

2. Das erstellt die ArgoCD Application "root"
   Diese Application überwacht: apps/ Ordner

3. ArgoCD sieht in apps/:
   - traefik.yaml     → erstellt ArgoCD Application "traefik"
   - cert-manager.yaml → erstellt ArgoCD Application "cert-manager"
   - manifests.yaml   → erstellt ArgoCD Application "manifests"

4. Jede dieser Applications deployed ihre Inhalte:
   - "traefik"     → installiert Traefik Helm Chart
   - "cert-manager" → installiert cert-manager Helm Chart
   - "manifests"   → deployed alle YAMLs aus manifests/

5. Neue App hinzufügen:
   → Neue Datei in apps/ erstellen
   → Git Push
   → ArgoCD entdeckt die neue Datei automatisch
   → Neue App wird deployed ✓
```

**Das Geniale:** Du musst nie wieder direkt `kubectl apply` ausführen.
Du pushst YAML in Git → ArgoCD übernimmt alles.

---

## 9. Schritt für Schritt: Neue App anlegen

Nehmen wir an du willst eine neue App deployen: **Uptime Kuma**
(ein Monitoring-Dashboard).

### Was du brauchst

1. Einen Helm-Chart oder YAML-Manifeste für die App
2. Einen Eintrag in `apps/`
3. Falls Helm: eine `values.yaml` in `values/`
4. Falls HTTPS: ein Certificate und IngressRoute in `manifests/`

### Schritt 1: Prüfen ob ein Helm Chart existiert

Gehe auf `artifacthub.io` und suche nach "uptime-kuma".
Falls ja: du benutzt Variante "Helm Chart".
Falls nein: du schreibst eigenes YAML (Variante "Rohes YAML").

### Schritt 2a: Neue App mit Helm Chart

**Datei erstellen: `apps/uptime-kuma.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://helm.irsigler.cloud          # Chart-Repo URL (von Artifact Hub)
      chart: uptime-kuma                            # Chart-Name
      targetRevision: "2.x"                         # Version (x = neueste 2.x)
      helm:
        releaseName: uptime-kuma
        valueFiles:
          - $values/values/uptime-kuma/values.yaml  # Deine Values
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: uptime-kuma                          # Eigener Namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true                        # Namespace automatisch erstellen
```

**Datei erstellen: `values/uptime-kuma/values.yaml`**

```yaml
# Schau zuerst: helm show values <repo>/<chart>
# Dann überschreibe nur was du brauchst

replicaCount: 1

service:
  type: ClusterIP    # Kein direkter externer Zugriff — Traefik übernimmt das

persistence:
  enabled: true      # Daten sollen gespeichert bleiben wenn Pod neustartet
  size: 1Gi          # 1 Gigabyte Speicher

resources:
  requests:
    memory: "128Mi"
    cpu: "50m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### Schritt 2b: Neue App mit eigenem YAML (ohne Helm)

Wenn es keinen Helm Chart gibt, schreibst du das YAML selbst:

**Ordner und Datei erstellen: `manifests/uptime-kuma/deployment.yaml`**

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: uptime-kuma
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma:1
        ports:
        - containerPort: 3001
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: data
          mountPath: /app/data       # Wo die App Daten speichert
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: uptime-kuma-pvc # Verweist auf PVC unten
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-pvc
  namespace: uptime-kuma
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  selector:
    app: uptime-kuma
  ports:
  - port: 3001
    targetPort: 3001
```

Dann noch eine ArgoCD Application anlegen die auf diesen Ordner zeigt:

**Datei erstellen: `apps/uptime-kuma.yaml`** (für rohes YAML):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    path: manifests/uptime-kuma    # Zeigt auf deinen Ordner
  destination:
    server: https://kubernetes.default.svc
    namespace: uptime-kuma
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Schritt 3: HTTPS einrichten (Certificate + IngressRoute)

Egal ob Helm oder rohes YAML — für HTTPS brauchst du immer diese 3 Objekte.
Das gehört in `manifests/uptime-kuma/ingress.yaml`:

```yaml
---
# 1. TLS-Zertifikat von Let's Encrypt anfordern
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: uptime-kuma-tls
  namespace: uptime-kuma
spec:
  secretName: uptime-kuma-tls          # Name des Secrets das erstellt wird
  issuerRef:
    name: letsencrypt-prod             # Nutze den bestehenden ClusterIssuer
    kind: ClusterIssuer
  dnsNames:
    - status.deine-domain.de           # DEINE Domain hier eintragen
---
# 2. HTTPS-Route: Traefik leitet Traffic zur App
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  entryPoints:
    - websecure                        # HTTPS (Port 443)
  routes:
    - match: Host(`status.deine-domain.de`)   # Deine Domain
      kind: Rule
      services:
        - name: uptime-kuma            # Name des Services oben
          port: 3001                   # Port des Services
  tls:
    secretName: uptime-kuma-tls        # Das Secret vom Certificate oben
---
# 3. HTTP → HTTPS Redirect
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: uptime-kuma-http
  namespace: uptime-kuma
spec:
  entryPoints:
    - web                              # HTTP (Port 80)
  routes:
    - match: Host(`status.deine-domain.de`)
      kind: Rule
      middlewares:
        - name: redirect-https
          namespace: argocd            # Diese Middleware existiert schon!
      services:
        - name: uptime-kuma
          port: 3001
```

### Schritt 4: Git Push

```bash
git add apps/uptime-kuma.yaml
git add values/uptime-kuma/values.yaml    # Falls Helm
git add manifests/uptime-kuma/            # Falls rohes YAML
git commit -m "Add uptime-kuma application"
git push
```

**Fertig.** ArgoCD erkennt die neue Datei in `apps/` innerhalb von 3 Minuten
und deployed alles automatisch.

### Was danach passiert (automatisch)

```
1. ArgoCD erkennt neue Datei in apps/uptime-kuma.yaml
2. Erstellt neue ArgoCD Application "uptime-kuma"
3. Application deployed: Deployment, Service, PVC
4. cert-manager sieht das Certificate-Objekt
5. Stellt TLS-Zertifikat bei Let's Encrypt aus
6. Zertifikat landet in Secret "uptime-kuma-tls"
7. Traefik sieht IngressRoute + Secret
8. App ist per HTTPS erreichbar ✓
```

---

## 10. Echte Beispiele aus diesem Repo erklärt

### Das ArgoCD-Ingress Manifest (manifests/argocd/ingress.yaml)

```yaml
---
# Fordert ein TLS-Zertifikat für die ArgoCD-Domain an
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd              # Im argocd-Namespace, weil ArgoCD dort läuft
spec:
  secretName: argocd-tls         # Secret wird "argocd-tls" heißen
  issuerRef:
    name: letsencrypt-prod       # Nutzt den ClusterIssuer
    kind: ClusterIssuer
  dnsNames:
    - jakob-argocd.goava.ai      # Die Domain für ArgoCD

---
# Traefik leitet HTTPS-Traffic zu ArgoCD
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure                  # Nur HTTPS
  routes:
    - match: Host(`jakob-argocd.goava.ai`)   # Diese Domain...
      kind: Rule
      services:
        - name: argocd-server   # ...geht zum argocd-server Service
          port: 80
      middlewares:
        - name: argocd-headers  # Header-Middleware (weiter unten definiert)
  tls:
    secretName: argocd-tls      # Nutzt das Zertifikat von oben

---
# HTTP auf HTTPS umleiten
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-http
  namespace: argocd
spec:
  entryPoints:
    - web                        # HTTP (Port 80)
  routes:
    - match: Host(`jakob-argocd.goava.ai`)
      kind: Rule
      middlewares:
        - name: redirect-https   # Leite um zu HTTPS
      services:
        - name: argocd-server
          port: 80

---
# Middleware: leitet HTTP auf HTTPS um
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true              # 308 Permanent Redirect

---
# Middleware: setzt Header damit ArgoCD weiß es läuft hinter einem Proxy
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-headers
  namespace: argocd
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: https   # Sagt ArgoCD: "Du läufst hinter HTTPS"
```

**Warum der X-Forwarded-Proto Header?**
ArgoCD läuft intern als HTTP. Traefik terminiert TLS und leitet HTTP intern weiter.
Ohne diesen Header denkt ArgoCD "ich bin HTTP" und macht self-redirects auf HTTP.
Mit dem Header weiß ArgoCD: "Der Proxy vor mir macht TLS, ich bin OK."

### Die traefik values.yaml verstehen

```yaml
deployment:
  replicas: 1              # Nur 1 Traefik-Instanz (genug für kleinen Cluster)

service:
  type: LoadBalancer       # Hetzner Cloud erstellt automatisch eine externe IP
                           # Diese IP zeigt auf alle Traefik-Pods

ports:
  web:
    port: 8000             # Traefik-Container lauscht intern auf 8000
    exposedPort: 80        # Nach außen (LoadBalancer) ist es Port 80
                           # Mapping: extern 80 → intern 8000
  websecure:
    port: 8443             # Intern 8443
    exposedPort: 443       # Extern 443
    tls:
      enabled: true        # Traefik erwartet TLS-Traffic auf diesem Port

providers:
  kubernetesCRD:
    enabled: true          # Traefik liest IngressRoute, Middleware etc.
  kubernetesIngress:
    enabled: true          # Traefik liest auch Standard k8s Ingress-Objekte
```

---

## 11. Häufige Fehler und wie du sie erkennst

### YAML-Fehler

**Einrückungsfehler:**
```yaml
# FALSCH — "containers" ist auf der falschen Ebene
spec:
  replicas: 1
    containers:            # ← 4 Leerzeichen statt 2
    - name: app

# RICHTIG
spec:
  replicas: 1
  containers:              # ← 2 Leerzeichen
  - name: app
```

**Fehlende Leerzeichen:**
```yaml
name:meine-app             # FALSCH
name: meine-app            # RICHTIG
```

**Falscher Datentyp:**
```yaml
targetRevision: 33.x       # YAML liest das als Zahl → Fehler
targetRevision: "33.x"     # RICHTIG: als String in Anführungszeichen
```

### Kubernetes-Fehler

**Labels stimmen nicht überein:**
```yaml
# Das Deployment "sieht" seine Pods nicht!
spec:
  selector:
    matchLabels:
      app: meine-app       # ← Selector sagt "meine-app"

  template:
    metadata:
      labels:
        app: meine-App     # ← Label sagt "meine-App" (A groß!) → Mismatch!
```

**Falscher Namespace:**
```yaml
# Service und Deployment sind in verschiedenen Namespaces → können sich nicht finden
---
kind: Deployment
metadata:
  namespace: app-v1        # ← Namespace 1

---
kind: Service
metadata:
  namespace: app-v2        # ← Namespace 2 → Service findet keine Pods!
```

**Port-Mismatch:**
```yaml
# Service schickt Traffic an Port 80, Container lauscht auf 8080 → Connection Refused
kind: Service
spec:
  ports:
  - port: 80
    targetPort: 80         # ← Sagt: schick an Port 80 im Pod

---
kind: Deployment
spec:
  template:
    spec:
      containers:
      - containerPort: 8080  # ← Pod lauscht auf 8080!
      # Lösung: targetPort: 8080 im Service
```

### ArgoCD-Fehler

**OutOfSync aber kein Auto-Sync:**
```yaml
syncPolicy:
  automated:               # ← Dieses Feld fehlt → kein auto sync
    prune: true
```

**Namespace existiert nicht:**
```yaml
syncOptions:
  - CreateNamespace=true   # ← Vergessen → ArgoCD kann nicht deployen
```

**Falscher Repo-Pfad:**
```yaml
source:
  path: apps/meine-app     # ← Dieser Ordner existiert nicht im Repo
  path: manifests/meine-app  # ← RICHTIG: Ordner muss wirklich existieren
```

### Debugging-Befehle

```bash
# ArgoCD Application Status prüfen
kubectl get applications -n argocd

# Details einer Application (zeigt Sync-Fehler)
kubectl describe application meine-app -n argocd

# Alle Pods in einem Namespace
kubectl get pods -n meine-app

# Pod-Logs lesen
kubectl logs -n meine-app deployment/meine-app

# Pod-Fehler analysieren (Events sind oft entscheidend!)
kubectl describe pod <pod-name> -n meine-app

# Service und Endpoints prüfen (sind Pods hinter dem Service?)
kubectl get endpoints -n meine-app

# Certificate-Status prüfen
kubectl get certificate -n meine-app
kubectl describe certificate meine-app-tls -n meine-app
```

---

## 12. Spickzettel — Das Wichtigste auf einen Blick

### Neue App Checkliste

```
□ Helm Chart oder rohes YAML?
  → Helm:  apps/<app>.yaml  +  values/<app>/values.yaml
  → YAML:  apps/<app>.yaml  +  manifests/<app>/*.yaml

□ ArgoCD Application erstellen (apps/<app>.yaml)
  □ Name eindeutig?
  □ namespace: argocd (immer!)
  □ Richtige repoURL?
  □ Richtiger path/chart?
  □ Richtiger Ziel-Namespace?
  □ CreateNamespace=true?
  □ automated.prune: true?
  □ automated.selfHeal: true?

□ Falls Helm: values/<app>/values.yaml erstellen
  □ Nur Werte die von Defaults abweichen

□ Falls HTTPS gewünscht: manifests/<app>/ingress.yaml
  □ Certificate (mit richtiger Domain und secretName)
  □ IngressRoute websecure (mit richtiger Domain und richtigem Service+Port)
  □ IngressRoute web → redirect-https Middleware
  □ DNS-Eintrag beim Domain-Anbieter setzen!

□ Git add, commit, push
□ In ArgoCD prüfen ob sync erfolgreich
```

### Dateistruktur Referenz

```
K8s-Helm/
├── apps/
│   └── <app-name>.yaml          ← ArgoCD Application Definition
│
├── values/
│   └── <app-name>/
│       └── values.yaml          ← Deine Helm Values
│
└── manifests/
    └── <app-name>/
        ├── deployment.yaml      ← Falls kein Helm
        ├── service.yaml
        └── ingress.yaml         ← Certificate + IngressRoute
```

### YAML Grundregeln

```
✓ Immer 2 Leerzeichen einrücken (niemals Tabs)
✓ Leerzeichen nach jedem Doppelpunkt: key: value
✓ Strings mit Sonderzeichen in Anführungszeichen: "33.x"
✓ Listen mit Bindestrich: - item
✓ Mehrere Dokumente trennen: ---
✓ Kommentare mit #
```

### ArgoCD Application Minimal-Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: DEIN-APP-NAME
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:                          # Für rohes YAML aus Git
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    path: manifests/DEIN-APP-NAME
  destination:
    server: https://kubernetes.default.svc
    namespace: DEIN-APP-NAME
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Certificate + IngressRoute Minimal-Template

```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: DEIN-APP-NAME-tls
  namespace: DEIN-APP-NAME
spec:
  secretName: DEIN-APP-NAME-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - DEINE-DOMAIN.DE
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: DEIN-APP-NAME
  namespace: DEIN-APP-NAME
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`DEINE-DOMAIN.DE`)
      kind: Rule
      services:
        - name: DEIN-SERVICE-NAME
          port: DEIN-PORT
  tls:
    secretName: DEIN-APP-NAME-tls
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: DEIN-APP-NAME-http
  namespace: DEIN-APP-NAME
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`DEINE-DOMAIN.DE`)
      kind: Rule
      middlewares:
        - name: redirect-https
          namespace: argocd
      services:
        - name: DEIN-SERVICE-NAME
          port: DEIN-PORT
```

---

*Faustregel: Wenn etwas nicht funktioniert — zuerst `kubectl describe` auf das
betroffene Objekt. Die Events am Ende der Ausgabe erklären meistens genau was fehlt.*
