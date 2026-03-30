# YAML & Helm Charts lernen — Von Null bis eigene App deployen

> Diese Dokumentation erklärt YAML und Helm von Grund auf — für absolute Anfänger.
> Alle Beispiele kommen aus diesem echten Setup — nichts ist erfunden oder theoretisch.
> Am Ende kannst du selbst eine neue App anlegen, konfigurieren und in den Cluster deployen.
> Lies es von oben nach unten durch. Jedes Kapitel baut auf dem vorherigen auf.

---

## Inhaltsverzeichnis

1. [YAML — Die Sprache von Kubernetes](#1-yaml--die-sprache-von-kubernetes)
2. [Kubernetes YAML — Wie ein Manifest aufgebaut ist](#2-kubernetes-yaml--wie-ein-manifest-aufgebaut-ist)
3. [Die wichtigsten Kubernetes-Objekte in YAML](#3-die-wichtigsten-kubernetes-objekte-in-yaml)
4. [Wie Kubernetes-Objekte zusammenarbeiten](#4-wie-kubernetes-objekte-zusammenarbeiten)
5. [Was ist Helm und warum braucht man es?](#5-was-ist-helm-und-warum-braucht-man-es)
6. [Helm Chart Struktur — Was ist wo und warum](#6-helm-chart-struktur--was-ist-wo-und-warum)
7. [Helm Templates — Wie Variablen funktionieren](#7-helm-templates--wie-variablen-funktionieren)
8. [values.yaml — Das Herzstück der Konfiguration](#8-valuesyaml--das-herzstück-der-konfiguration)
9. [ArgoCD Application — Der Kleber zwischen Git und Cluster](#9-argocd-application--der-kleber-zwischen-git-und-cluster)
10. [Das App-of-Apps Pattern in diesem Setup](#10-das-app-of-apps-pattern-in-diesem-setup)
11. [Schritt für Schritt: Neue App anlegen](#11-schritt-für-schritt-neue-app-anlegen)
12. [Echte Beispiele aus diesem Repo — vollständig erklärt](#12-echte-beispiele-aus-diesem-repo--vollständig-erklärt)
13. [Häufige Fehler, ihre Ursachen und Lösungen](#13-häufige-fehler-ihre-ursachen-und-lösungen)
14. [Wie du unbekannte Charts liest und verstehst](#14-wie-du-unbekannte-charts-liest-und-verstehst)
15. [Spickzettel — Das Wichtigste auf einen Blick](#15-spickzettel--das-wichtigste-auf-einen-blick)

---

## 1. YAML — Die Sprache von Kubernetes

### Was ist YAML und warum nutzt Kubernetes es?

YAML steht für "YAML Ain't Markup Language" — ein selbstreferenzieller Witz aus der
Programmierwelt. Aber der eigentliche Grund warum YAML existiert ist simpel:
Computer brauchen strukturierte Daten, und Menschen sollen diese Daten lesen und
schreiben können, ohne zu leiden.

Vergleich mit anderen Formaten die dasselbe ausdrücken:

**JSON** — maschinenfreundlich, aber für Menschen anstrengend zu schreiben:
```json
{
  "name": "meine-app",
  "replicas": 3,
  "ports": [80, 443],
  "config": {
    "debug": false,
    "logLevel": "info"
  }
}
```

**YAML** — exakt dieselben Daten, viel lesbarer:
```yaml
name: meine-app
replicas: 3
ports:
  - 80
  - 443
config:
  debug: false
  logLevel: info
```

Kubernetes hat YAML gewählt weil du täglich viele Konfigurationsdateien schreiben
und lesen musst. XML wäre noch unlesbarer als JSON, JSON braucht überall Anführungszeichen
und Kommas. YAML ist der Kompromiss der sich in der Praxis durchgesetzt hat.

**Wichtig zu verstehen:** YAML ist nur ein Format um Daten zu speichern — wie ein
leeres Formular. Kubernetes liest dieses Formular aus und tut dann etwas damit.
YAML selbst "tut" nichts.

---

### Die 3 Grundbausteine von YAML

YAML hat genau drei Strukturen. Alles andere ist eine Kombination davon.

---

#### Baustein 1: Key-Value Paare (Schlüssel-Wert)

Das ist die absolut einfachste Form. Ein Name (Key) und ein Wert (Value),
getrennt durch einen Doppelpunkt und ein Leerzeichen:

```yaml
name: meine-app
version: 1.0
replicas: 3
aktiv: true
```

Stell es dir wie eine Tabelle mit zwei Spalten vor:

| Key | Value |
|-----|-------|
| name | meine-app |
| version | 1.0 |
| replicas | 3 |
| aktiv | true |

**Die wichtigste Regel von YAML:** Nach dem Doppelpunkt MUSS mindestens ein
Leerzeichen kommen. Nicht optional — Pflicht.

```yaml
name:meine-app   # FALSCH — kein Leerzeichen → YAML-Fehler
name: meine-app  # RICHTIG
name:  meine-app # Auch OK — mehrere Leerzeichen sind erlaubt
```

---

#### Baustein 2: Verschachtelung durch Einrückung (Hierarchie)

Das ist das Konzept das Anfänger am häufigsten falsch machen.
YAML nutzt **Leerzeichen** um zu zeigen was zu was gehört — wie Einrückung in
einer Gliederung.

**Die eiserne Regel: Immer genau 2 Leerzeichen pro Ebene. Niemals Tabs.**

```yaml
# Ebene 0: keine Einrückung
deployment:

  # Ebene 1: 2 Leerzeichen
  name: meine-app
  replicas: 3
  container:

    # Ebene 2: 4 Leerzeichen (2 × 2)
    image: nginx:1.25
    ressourcen:

      # Ebene 3: 6 Leerzeichen (3 × 2)
      cpu: "100m"
      memory: "128Mi"
```

Stell es dir wie eine Gliederung vor:

```
deployment
├── name: meine-app
├── replicas: 3
└── container
    ├── image: nginx:1.25
    └── ressourcen
        ├── cpu: "100m"
        └── memory: "128Mi"
```

Alles was gleich weit eingerückt ist, gehört zur selben Gruppe.
Alles was weiter eingerückt ist, ist ein "Kind" der Zeile darüber.

**Warum keine Tabs?** Tabs können in verschiedenen Editoren unterschiedlich
dargestellt werden (2, 4 oder 8 Zeichen breit). YAML hat deshalb Tabs komplett
verboten um Verwirrung zu vermeiden. Konfiguriere deinen Editor so dass er bei
Tab-Taste automatisch Leerzeichen einfügt.

---

#### Baustein 3: Listen (mit Bindestrich)

Wenn mehrere gleichwertige Dinge aufgelistet werden sollen, nutzt du einen
Bindestrich gefolgt von einem Leerzeichen:

```yaml
# Einfache Liste von Werten
früchte:
  - Apfel
  - Banane
  - Orange

# Liste von Objekten — jedes Listenelement kann mehrere Felder haben
# Der Bindestrich markiert den Anfang eines neuen Listenelements
container:
  - name: meine-app          # ← Erstes Element beginnt mit Bindestrich
    image: nginx:1.25        # ← Zweites Feld des ersten Elements (kein Bindestrich!)
    port: 8080               # ← Drittes Feld des ersten Elements
  - name: sidecar            # ← Zweites Element beginnt mit neuem Bindestrich
    image: busybox:latest
    port: 9090
```

Das ist ein häufiger Stolperstein: Der Bindestrich steht NUR beim ersten Feld
jedes Listenelements. Die folgenden Felder desselben Elements haben keinen Bindestrich,
aber sind auf derselben Einrückungsebene wie der Name nach dem Bindestrich.

```yaml
# Wie es in Kubernetes aussieht — Ports-Liste im Service:
ports:
  - port: 80           # ← Bindestrich: neues Element
    targetPort: 8080   # ← Kein Bindestrich: gehört zum selben Element wie port: 80
    protocol: TCP      # ← Kein Bindestrich: gehört zum selben Element
  - port: 443          # ← Bindestrich: neues Element
    targetPort: 8443
    protocol: TCP
```

---

### Alle Datentypen in YAML

YAML erkennt automatisch welchen Typ ein Wert hat. Das kann manchmal
zu unerwarteten Ergebnissen führen:

```yaml
# ──────────────────────────────────────────────
# STRINGS (Text)
# ──────────────────────────────────────────────
name: meine-app               # Ohne Anführungszeichen — funktioniert
name: "meine-app"             # Mit Anführungszeichen — auch OK
name: 'meine-app'             # Einfache Anführungszeichen — auch OK

# Wann MÜSSEN Anführungszeichen?
version: "1.0.0"              # YAML würde 1.0.0 als Fehler parsen (kein gültiger Float)
wert: "true"                  # Ohne Anführungszeichen → wird als Boolean interpretiert!
wert: "null"                  # Ohne Anführungszeichen → wird als null interpretiert!
pfad: "C:\\Users\\Jakob"      # Backslash braucht Anführungszeichen
text: "Hallo: Welt"           # Doppelpunkt im Wert → Anführungszeichen nötig
text: "Hallo #Welt"           # Raute im Wert → Anführungszeichen nötig

# ──────────────────────────────────────────────
# ZAHLEN
# ──────────────────────────────────────────────
replicas: 3                   # Integer (ganze Zahl)
gewicht: 1.5                  # Float (Kommazahl)
port: 8080                    # Integer
speicher: 1024                # Integer

# ──────────────────────────────────────────────
# BOOLEAN (Wahrheitswerte)
# ──────────────────────────────────────────────
aktiv: true                   # true/false (kleingeschrieben)
debug: false
# ACHTUNG: Diese werden AUCH als true/false interpretiert:
# yes, no, on, off, True, False, TRUE, FALSE
# → Immer true/false nutzen um Verwirrung zu vermeiden

# ──────────────────────────────────────────────
# NULL (leer / kein Wert)
# ──────────────────────────────────────────────
wert: null                    # Explizit null
wert: ~                       # Alternative für null
wert:                         # Leerer Wert = auch null

# ──────────────────────────────────────────────
# MEHRZEILIGER TEXT
# ──────────────────────────────────────────────

# Pipe (|) — Zeilenumbrüche werden ERHALTEN
skript: |
  #!/bin/bash
  echo "Hallo"
  echo "Welt"
  # Diese drei Zeilen bleiben drei separate Zeilen

# Größer-als (>) — Zeilenumbrüche werden zu LEERZEICHEN
beschreibung: >
  Das ist ein langer Text der
  eigentlich eine einzige Zeile
  sein soll.
  # Ergebnis: "Das ist ein langer Text der eigentlich eine einzige Zeile sein soll."
```

**Praxistipp:** Im Zweifel immer Anführungszeichen setzen. Es ist sicherer
als zu riskieren dass YAML den Wert falsch interpretiert.

---

### Kommentare in YAML

Mit `#` kannst du Kommentare schreiben — alles nach `#` wird ignoriert:

```yaml
# Das ist ein Kommentar — wird von Kubernetes komplett ignoriert
replicas: 3    # Kommentar am Zeilenende — auch OK
# replicas: 5  # Diese Zeile ist auskommentiert — hat keinen Effekt
```

Kommentare sind extrem nützlich um zu erklären WARUM etwas so konfiguriert ist.
Schreib Kommentare nicht für das WAS (das sieht man im Code) sondern für das WARUM.

---

### Mehrere Dokumente in einer Datei

Mit `---` (drei Bindestriche) trennst du mehrere YAML-Dokumente in einer Datei.
Kubernetes nutzt das um mehrere Objekte in einer einzigen Datei zu definieren:

```yaml
---
# Erstes Kubernetes-Objekt: der Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: meine-app
---
# Zweites Kubernetes-Objekt: das Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app
  namespace: meine-app
---
# Drittes Kubernetes-Objekt: der Service
apiVersion: v1
kind: Service
metadata:
  name: meine-app
  namespace: meine-app
```

Das ist sehr praktisch weil du alles was zusammengehört in einer Datei halten kannst.
Wenn du `kubectl apply -f datei.yaml` ausführst, werden alle drei Objekte auf einmal erstellt.

---

### YAML validieren bevor du appliest

Bevor du etwas an Kubernetes schickst, kannst du YAML-Fehler prüfen:

```bash
# YAML-Syntax prüfen (braucht python3)
python3 -c "import yaml; yaml.safe_load(open('datei.yaml'))" && echo "OK"

# Kubernetes-Manifest validieren ohne anzuwenden (dry-run)
kubectl apply -f datei.yaml --dry-run=client

# Noch detailliertere Validierung
kubectl apply -f datei.yaml --dry-run=server
```

---

## 2. Kubernetes YAML — Wie ein Manifest aufgebaut ist

### Was ist ein Manifest?

In Kubernetes nennt man eine YAML-Datei die ein Kubernetes-Objekt beschreibt
ein **Manifest**. Das Wort kommt von "manifestieren" — du machst sichtbar was
du willst, und Kubernetes sorgt dafür dass es Wirklichkeit wird.

Jedes Kubernetes-Manifest — egal was es beschreibt — hat **immer dieselbe
Grundstruktur** mit denselben 4 Pflichtfeldern:

```yaml
apiVersion: ...    # Welche API-Version verwende ich?
kind: ...          # Was für ein Objekt will ich erstellen?
metadata:          # Informationen ÜBER das Objekt (Name, Tags, etc.)
  name: ...
spec:              # Die eigentliche Konfiguration — was soll das Objekt tun?
  ...
```

Das ist wie ein Formular: Oben stehen immer "Art des Formulars" und "Name des Antragstellers",
dann kommt der eigentliche Inhalt. Diese Struktur kennst du nach einer Woche auswendig.

---

### apiVersion — Welche API benutze ich?

Kubernetes ist kein monolithisches System — es besteht aus vielen verschiedenen
API-Gruppen, die unterschiedliche Objekte verwalten. `apiVersion` sagt Kubernetes
welche dieser APIs du ansprechen willst.

```yaml
# Kern-APIs (die ältesten, stabilsten)
apiVersion: v1
# → Für: Pod, Service, ConfigMap, Secret, Namespace, PersistentVolume,
#        PersistentVolumeClaim, ServiceAccount, Endpoints

# Apps-API (für Workload-Objekte)
apiVersion: apps/v1
# → Für: Deployment, ReplicaSet, StatefulSet, DaemonSet

# Batch-API (für einmalige Aufgaben)
apiVersion: batch/v1
# → Für: Job, CronJob

# Netzwerk-API
apiVersion: networking.k8s.io/v1
# → Für: Ingress, NetworkPolicy, IngressClass

# RBAC-API (Berechtigungen)
apiVersion: rbac.authorization.k8s.io/v1
# → Für: Role, ClusterRole, RoleBinding, ClusterRoleBinding

# Erweiterungen durch installierte Tools (CRDs)
apiVersion: argoproj.io/v1alpha1    # ArgoCD: Application, AppProject
apiVersion: cert-manager.io/v1     # cert-manager: Certificate, ClusterIssuer
apiVersion: traefik.io/v1alpha1    # Traefik: IngressRoute, Middleware
```

**Was bedeutet das `v1alpha1` oder `v1beta1`?**

Kubernetes kennzeichnet APIs mit ihrem Reifegrad:
- `v1` = Stabil, produktionsreif, wird nicht ohne Ankündigung geändert
- `v1beta1` = Fast stabil, kleine Änderungen möglich
- `v1alpha1` = Experimentell, kann sich jederzeit ändern

Für ArgoCD-Objekte nutzt du `argoproj.io/v1alpha1` — das ist die ArgoCD-API,
die ArgoCD selbst definiert hat (als sogenannte "Custom Resource Definition").

**Woher weiß ich die richtige apiVersion?**

```bash
# Zeigt alle verfügbaren API-Ressourcen mit ihren apiVersions
kubectl api-resources

# Ausgabe (Ausschnitt):
# NAME                SHORTNAMES  APIVERSION                 KIND
# pods                po          v1                         Pod
# deployments         deploy      apps/v1                    Deployment
# services            svc         v1                         Service
# ingresses           ing         networking.k8s.io/v1       Ingress
# applications                    argoproj.io/v1alpha1       Application
```

---

### kind — Was für ein Objekt?

`kind` sagt Kubernetes was du erstellen willst. Der Wert ist immer
**GroßCamelCase** (erster Buchstabe groß, kein Bindestrich):

```yaml
# Workloads (was läuft)
kind: Pod               # Direkt einen Pod erstellen (selten)
kind: Deployment        # Mehrere Pod-Kopien, mit Rolling Update
kind: StatefulSet       # Wie Deployment, aber für Datenbanken mit stabiler Identität
kind: DaemonSet         # Genau 1 Pod pro Node
kind: Job               # Pod der einmal läuft und fertig ist
kind: CronJob           # Job auf Zeitplan

# Netzwerk
kind: Service           # Macht Pods intern oder extern erreichbar
kind: Ingress           # HTTP-Routing (Standard-Kubernetes)
kind: IngressRoute      # HTTP-Routing (Traefik-spezifisch, mächtiger)
kind: Middleware        # Traefik: Modifikationen am Traffic
kind: NetworkPolicy     # Firewall-Regeln zwischen Pods

# Konfiguration
kind: ConfigMap         # Konfigurationsdaten (nicht geheim)
kind: Secret            # Geheime Daten (Passwörter, Zertifikate)

# Storage
kind: PersistentVolumeClaim   # Anforderung für persistenten Speicher
kind: PersistentVolume        # Tatsächlicher Speicher (meist auto-erstellt)
kind: StorageClass            # Wie Speicher bereitgestellt wird

# Cluster-Organisation
kind: Namespace         # Logische Trennung im Cluster
kind: ServiceAccount    # Identität für Pods

# Berechtigungen
kind: Role              # Berechtigungen in einem Namespace
kind: ClusterRole       # Berechtigungen clusterweit
kind: RoleBinding       # Verbindet ServiceAccount mit Role
kind: ClusterRoleBinding # Verbindet ServiceAccount mit ClusterRole

# ArgoCD (Custom Resources)
kind: Application       # Eine deployete App in ArgoCD
kind: AppProject        # Gruppierung von Applications mit RBAC

# cert-manager (Custom Resources)
kind: Certificate       # Anforderung eines TLS-Zertifikats
kind: ClusterIssuer     # Wie Zertifikate ausgestellt werden (Let's Encrypt etc.)
```

---

### metadata — Informationen über das Objekt

`metadata` enthält alles was das Objekt identifiziert und klassifiziert:

```yaml
metadata:
  # ── PFLICHTFELDER ──────────────────────────────────────────────
  name: meine-app
  # Der Name muss innerhalb des Namespaces eindeutig sein.
  # Erlaubt: Kleinbuchstaben, Zahlen, Bindestriche.
  # Verboten: Großbuchstaben, Unterstriche, Punkte (meistens).
  # Maximal 253 Zeichen.

  # ── OPTIONALE FELDER ───────────────────────────────────────────
  namespace: meine-app
  # In welchem Namespace lebt dieses Objekt?
  # Wenn nicht angegeben: "default"
  # Ausnahme: ClusterRole, ClusterIssuer etc. sind cluster-weit → kein Namespace

  labels:
  # Labels sind Schlüssel-Wert-Paare die du frei vergeben kannst.
  # Kubernetes nutzt sie um Objekte zu finden und zu gruppieren.
  # Services finden ihre Pods über Labels!
    app: meine-app          # Standard-Label: welche App ist das?
    version: "2.1"          # Welche Version?
    umgebung: production    # Welche Umgebung?
    team: backend           # Welches Team ist verantwortlich?

  annotations:
  # Annotations sind auch Schlüssel-Wert-Paare, aber für Metadaten die
  # Kubernetes selbst NICHT für Filtering/Selection nutzt.
  # Gut für: Dokumentation, Tool-Konfiguration, externe Systeme
    kubernetes.io/description: "Hauptanwendung des Backend-Teams"
    deployment-date: "2025-03-30"
    contact: "jakob@beispiel.de"
    # cert-manager nutzt Annotations um IngressRoutes zu konfigurieren:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

**Der Unterschied zwischen Labels und Annotations — ein Beispiel:**

Labels sind wie Kategorien in einem Datei-System — du kannst danach filtern und suchen:
```bash
kubectl get pods -l app=meine-app          # Alle Pods mit label app=meine-app
kubectl get pods -l umgebung=production    # Alle Production-Pods
kubectl get pods -l app=meine-app,version=2.1  # Mehrere Labels kombinieren
```

Annotations kannst du nicht filtern — sie sind nur zum Lesen:
```bash
kubectl get pod meine-app -o yaml | grep annotations  # Zeigt alle annotations
```

---

### spec — Das Herzstück des Manifests

`spec` (kurz für "specification") ist der Hauptteil jedes Manifests.
Hier beschreibst du was das Objekt tun soll.

Das Wichtigste zu verstehen: **spec ist je nach `kind` völlig anders.**
Ein Deployment-spec sieht komplett anders aus als ein Service-spec oder ein
Certificate-spec. Es gibt keine universelle spec-Struktur.

Du lernst die verschiedenen spec-Strukturen am besten durch Beispiele — die
kommen in Kapitel 3.

---

### status — Der aktuelle Zustand (von Kubernetes geschrieben)

`status` schreibst du nie selbst. Kubernetes füllt dieses Feld automatisch
aus um den aktuellen Zustand des Objekts zu dokumentieren:

```yaml
# Was du schreibst (spec = was du willst):
spec:
  replicas: 3

# Was Kubernetes ausfüllt (status = was gerade ist):
status:
  replicas: 3           # Wie viele Pods existieren
  readyReplicas: 2      # Wie viele sind bereit (einer startet gerade noch)
  updatedReplicas: 3    # Wie viele haben die neue Version
  conditions:
  - type: Available
    status: "True"
    message: "Deployment has minimum availability."
  - type: Progressing
    status: "True"
    message: "ReplicaSet meine-app-7d6f has successfully progressed."
```

Du siehst den `status` wenn du:
```bash
kubectl get deployment meine-app -o yaml
kubectl describe deployment meine-app
```

---

## 3. Die wichtigsten Kubernetes-Objekte in YAML

Jetzt kommen die konkreten Objekte. Wir schauen uns jedes Feld genau an
und erklären warum es da ist.

---

### Deployment — Die häufigste Methode eine App zu betreiben

Ein Deployment ist das Standard-Objekt für Anwendungen die:
- Mehrfach laufen sollen (mehrere Replicas für Ausfallsicherheit)
- Keine persistente Identität brauchen (Pods sind austauschbar)
- Updates ohne Downtime bekommen sollen

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meine-app
  namespace: meine-app
  labels:
    app: meine-app    # Label am Deployment selbst (für kubectl-Filterung)
spec:

  # ── WIE VIELE PODS? ────────────────────────────────────────────
  replicas: 2
  # Kubernetes stellt sicher dass IMMER genau 2 Pods laufen.
  # Stirbt einer → neuer wird sofort gestartet.
  # Node fällt aus → Pods werden auf andere Nodes verschoben.

  # ── WELCHE PODS GEHÖREN ZU DIESEM DEPLOYMENT? ──────────────────
  selector:
    matchLabels:
      app: meine-app
  # Das Deployment muss wissen welche Pods es verwaltet.
  # Es "sucht" nach Pods mit exakt diesen Labels.
  # WICHTIG: Muss identisch mit template.metadata.labels sein!

  # ── STRATEGIE FÜR UPDATES ──────────────────────────────────────
  strategy:
    type: RollingUpdate      # Standard: alte Pods nach und nach ersetzen
    rollingUpdate:
      maxUnavailable: 0      # Nie weniger als 2 Pods verfügbar haben
      maxSurge: 1            # Maximal 1 extra Pod während Update erlaubt
  # Alternativ: type: Recreate → erst alle alten töten, dann neue starten
  # (→ kurzer Downtime, aber sauber — gut für inkompatible DB-Migrationen)

  # ── VORLAGE FÜR JEDEN POD ──────────────────────────────────────
  template:
    metadata:
      labels:
        app: meine-app     # MUSS mit selector.matchLabels übereinstimmen!
        version: "2.1"     # Zusätzliche Labels sind erlaubt
    spec:                  # Hier beginnt die Pod-Spezifikation
      containers:
        # Ein Pod kann mehrere Container haben.
        # Sie teilen sich das Netzwerk (gleiche IP) und Volumes.

      - name: meine-app         # Name des Containers (innerhalb des Pods eindeutig)
        image: nginx:1.25       # Docker-Image Name:Tag

        # ── PORTS ──────────────────────────────────────────────
        ports:
        - containerPort: 80     # Auf welchem Port lauscht dieser Container?
          name: http            # Optionaler Name (kann im Service referenziert werden)
          protocol: TCP         # TCP (Standard) oder UDP

        # ── RESSOURCEN ──────────────────────────────────────────
        resources:
          requests:
            # Was der Container MINDESTENS braucht.
            # Kubernetes reserviert diese Menge auf der Node.
            # Wenn keine Node genug hat → Pod bleibt Pending.
            memory: "64Mi"    # 64 Mebibyte RAM (Mi = Mebibyte, M = Megabyte)
            cpu: "50m"        # 50 Millicores = 0,05 CPU-Kerne
          limits:
            # Das MAXIMUM das der Container nutzen darf.
            # RAM-Limit überschritten → Container wird sofort gekillt (OOMKilled)
            # CPU-Limit überschritten → Container wird gedrosselt (throttled)
            memory: "128Mi"
            cpu: "200m"       # 200 Millicores = 0,2 CPU-Kerne

        # ── UMGEBUNGSVARIABLEN ──────────────────────────────────
        env:
        - name: APP_ENV         # Variablenname im Container
          value: "production"   # Direkter Wert

        - name: DB_HOST
          value: "postgres-service.datenbank.svc.cluster.local"
          # DNS-Name des PostgreSQL-Services im Namespace "datenbank"

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:       # Wert kommt aus einem Secret
              name: db-secret   # Name des Secrets
              key: password     # Welcher Key im Secret

        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:    # Wert kommt aus einer ConfigMap
              name: app-config
              key: LOG_LEVEL

        # ── HEALTH CHECKS ──────────────────────────────────────
        livenessProbe:
          # Kubernetes prüft ob der Container noch "lebt".
          # Schlägt es fehl → Container wird neugestartet.
          httpGet:
            path: /healthz      # Dieser HTTP-Endpunkt muss 200 zurückgeben
            port: 80
          initialDelaySeconds: 30  # Erst nach 30s prüfen (Container braucht Zeit zum Starten)
          periodSeconds: 10        # Alle 10 Sekunden prüfen
          failureThreshold: 3      # 3 Fehlschläge → Neustart

        readinessProbe:
          # Kubernetes prüft ob der Container bereit ist Traffic zu empfangen.
          # Schlägt es fehl → Pod bekommt KEINEN Traffic (aber wird nicht neugestartet).
          # Nützlich wenn App beim Start Initialisierung braucht.
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5

        # ── VOLUMES EINBINDEN ───────────────────────────────────
        volumeMounts:
        - name: config-volume   # Muss mit volumes[].name übereinstimmen
          mountPath: /etc/config  # Wo im Container-Dateisystem

        - name: data-volume
          mountPath: /data

      # Volumes die dieser Pod nutzen kann (am Pod-Level, nicht Container-Level)
      volumes:
      - name: config-volume
        configMap:
          name: meine-app-config   # Die ConfigMap mit diesem Namen

      - name: data-volume
        persistentVolumeClaim:
          claimName: meine-app-pvc  # Der PVC mit diesem Namen
```

**Das Zusammenspiel von selector und template.labels:**

Das ist das häufigste Missverständnis für Anfänger. Warum müssen beide Labels
identisch sein? Weil das Deployment so seine "eigenen" Pods erkennt:

```
Deployment erstellt Pod mit labels: {app: meine-app}
Deployment's selector:              {app: meine-app}  ← sucht danach

Kubernetes:
"Welche Pods gehören zu diesem Deployment?"
→ Alle Pods wo labels.app == 'meine-app'
→ Findet die Pods die das Deployment selbst erstellt hat ✓

Wenn Labels nicht passen:
→ Deployment findet seine eigenen Pods nicht
→ Denkt: "Ich habe 0 Pods, ich brauche 2" → erstellt endlos neue Pods
→ Chaos
```

---

### Service — Stabiler Endpunkt für Pods

Ein Service löst ein fundamentales Problem: Pods kommen und gehen, ihre IPs
ändern sich ständig. Andere Pods können nicht auf wechselnde IPs zeigen.
Der Service ist die stabile Adresse die immer gleich bleibt.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: meine-app          # Der Name wird automatisch zum DNS-Namen!
  namespace: meine-app     # Im gleichen Namespace wie das Deployment
spec:

  # ── WELCHE PODS BEKOMMT TRAFFIC? ───────────────────────────────
  selector:
    app: meine-app
  # Der Service schickt Traffic an alle Pods mit diesem Label.
  # Er prüft automatisch Pods die starten oder stoppen.
  # → Kein manuelles Update nötig wenn Pods neugestartet werden.

  # ── WELCHE PORTS WERDEN ANGEBOTEN? ─────────────────────────────
  ports:
  - name: http             # Optionaler Name für diesen Port
    port: 80               # Auf diesem Port ist der SERVICE erreichbar
    targetPort: 8080       # An diesen Port im POD wird Traffic weitergeleitet
    protocol: TCP

  # ← Wichtig: port und targetPort MÜSSEN nicht identisch sein!
  # Service lauscht auf 80, Container lauscht auf 8080.
  # Das ist nützlich wenn du den Service-Port nicht ändern kannst,
  # aber der Container auf einem anderen Port läuft.

  # ── SERVICE-TYP ─────────────────────────────────────────────────
  type: ClusterIP          # Standard: nur im Cluster erreichbar
  # Weitere Typen:
  # NodePort:      Öffnet Port auf jeder Node (30000-32767) — von außen erreichbar
  # LoadBalancer:  Fordert externe IP beim Cloud-Provider an
  # ExternalName:  Leitet auf externen DNS-Namen weiter (kein Selector)
```

**Wie der DNS-Name funktioniert:**

Der Service-Name `meine-app` im Namespace `meine-app` ist automatisch erreichbar unter:
```
meine-app                              # Aus dem gleichen Namespace
meine-app.meine-app                    # Aus einem anderen Namespace
meine-app.meine-app.svc               # Mit svc-Suffix
meine-app.meine-app.svc.cluster.local  # Vollständiger Name (FQDN)
```

Pods im selben Namespace können einfach `meine-app` als Hostname nutzen.
Pods in anderen Namespaces müssen `meine-app.meine-app` nutzen.

**Das Endpoints-Objekt — automatisch im Hintergrund:**

Wenn du einen Service erstellst, erstellt Kubernetes automatisch ein
Endpoints-Objekt das die echten Pod-IPs enthält:

```bash
kubectl get endpoints meine-app -n meine-app
# NAME        ENDPOINTS                          AGE
# meine-app   10.244.1.5:8080,10.244.2.9:8080   5m
```

Wenn Pods starten oder sterben, updated Kubernetes dieses Objekt automatisch.
kube-proxy liest dieses Objekt und passt die iptables-Regeln an.

---

### Namespace — Logische Trennung im Cluster

Ein Namespace ist wie ein Ordner in einem Dateisystem. Er trennt Objekte
voneinander und ermöglicht:
- Verschiedene Teams können denselben Objektnamen nutzen ohne Konflikte
  (`meine-app` in Namespace `team-a` und `meine-app` in Namespace `team-b`)
- RBAC: Team A kann nur in Namespace `team-a` Änderungen machen
- Ressourcen-Limits pro Namespace (ResourceQuota)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: meine-app
  labels:
    # Labels an Namespaces werden z.B. für NetworkPolicies genutzt
    kubernetes.io/metadata.name: meine-app
    umgebung: production
```

Ein Namespace selbst hat kaum Konfiguration — er ist nur ein Container.

**Standard-Namespaces in Kubernetes:**
```
default      → Wenn du keinen Namespace angibst, landet alles hier
kube-system  → Kubernetes-interne Komponenten (kube-proxy, CoreDNS, etc.)
kube-public  → Öffentlich lesbare Daten (ohne Authentifizierung)
kube-node-lease → Node-Heartbeats
```

**Namespaces in diesem Setup:**
```
argocd       → ArgoCD selbst
traefik      → Traefik Ingress Controller
cert-manager → cert-manager
```

---

### ConfigMap — Konfigurationsdaten

Eine ConfigMap speichert Konfigurationsdaten als Schlüssel-Wert-Paare.
Sie ist für alles gedacht was NICHT geheim ist — Datenbankhost, Log-Level,
Feature-Flags, Konfigurationsdateien.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: meine-app-config
  namespace: meine-app
data:
  # ── EINFACHE SCHLÜSSEL-WERT-PAARE ──────────────────────────────
  LOG_LEVEL: "info"
  APP_PORT: "8080"
  DATABASE_HOST: "postgres.datenbank.svc.cluster.local"
  DATABASE_NAME: "myapp_prod"
  FEATURE_FLAG_NEUE_UI: "true"

  # ── GANZE DATEIEN ALS WERT ──────────────────────────────────────
  # Der Key ist der Dateiname, der Wert ist der Dateiinhalt
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }

  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      host: postgres.datenbank.svc.cluster.local
      port: 5432
      name: myapp_prod
      maxConnections: 10
    logging:
      level: info
      format: json
```

**ConfigMap im Pod nutzen:**

```yaml
# Methode 1: Als einzelne Umgebungsvariable
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: meine-app-config
      key: LOG_LEVEL

# Methode 2: Alle Keys als Umgebungsvariablen auf einmal
envFrom:
- configMapRef:
    name: meine-app-config
# → Erstellt ENV-Vars für ALLE Keys: LOG_LEVEL, APP_PORT, etc.

# Methode 3: Als Datei einbinden (Volume Mount)
volumes:
- name: config
  configMap:
    name: meine-app-config
    items:           # Optional: nur bestimmte Keys
    - key: nginx.conf
      path: nginx.conf    # Dateiname im Volume
    - key: config.yaml
      path: app/config.yaml

volumeMounts:
- name: config
  mountPath: /etc/nginx    # nginx.conf landet unter /etc/nginx/nginx.conf
```

---

### Secret — Geheime Daten

Secrets funktionieren fast identisch zu ConfigMaps, aber für sensitive Daten
wie Passwörter, API-Keys und TLS-Zertifikate.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: meine-app-secrets
  namespace: meine-app
type: Opaque    # Standard-Typ für allgemeine Secrets
                # Andere Typen: kubernetes.io/tls, kubernetes.io/dockerconfigjson
data:
  # ALLE WERTE MÜSSEN BASE64-KODIERT SEIN!
  # echo -n "meinPasswort123" | base64  →  bWVpblBhc3N3b3J0MTIz
  DB_PASSWORD: bWVpblBhc3N3b3J0MTIz
  API_KEY: c2VocmdlaGVpbWVyQVBJS2V5

  # Für TLS-Secrets (type: kubernetes.io/tls):
  # tls.crt: <base64-kodiertes Zertifikat>
  # tls.key: <base64-kodierter privater Schlüssel>
```

**Achtung — Base64 ist KEINE Verschlüsselung:**

Base64 ist nur eine Kodierung — jeder kann es dekodieren:
```bash
# Kodieren:
echo -n "meinPasswort" | base64
# Ausgabe: bWVpblBhc3N3b3J0

# Dekodieren — jeder kann das:
echo "bWVpblBhc3N3b3J0" | base64 -d
# Ausgabe: meinPasswort
```

Kubernetes-Secrets sind standardmäßig unverschlüsselt in etcd gespeichert.
Für echte Sicherheit: **Sealed Secrets** oder **External Secrets Operator** mit
einem Vault. Für dieses Setup ist Base64 ausreichend, da der Cluster nicht
öffentlich zugänglich ist.

**Secret im Pod nutzen:**

```yaml
# Als Umgebungsvariable (häufigste Methode)
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: meine-app-secrets
      key: DB_PASSWORD

# Alle Keys als Umgebungsvariablen
envFrom:
- secretRef:
    name: meine-app-secrets

# Als Datei (für TLS-Certs, Konfigurationsdateien mit Secrets)
volumes:
- name: secrets
  secret:
    secretName: meine-app-secrets
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true    # Best Practice: Secrets nur lesbar einbinden
```

---

### PersistentVolumeClaim — Persistenter Speicher

Container haben ein ephemeres (kurzlebiges) Dateisystem. Wenn der Container
stirbt oder neustartet, sind alle Daten weg. Für Datenbanken und andere
zustandsbehaftete Anwendungen brauchst du persistenten Speicher.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: meine-app-daten
  namespace: meine-app
spec:
  accessModes:
    - ReadWriteOnce      # Nur eine Node kann gleichzeitig lesen+schreiben
    # Alternativen:
    # ReadOnlyMany   → Viele Nodes können lesen (aber nicht schreiben)
    # ReadWriteMany  → Viele Nodes können lesen und schreiben (braucht NFS/CephFS)

  storageClassName: local-path   # Welcher Speichertyp? (abhängig vom Cluster)
  # Wenn nicht angegeben: Standard-StorageClass des Clusters wird genutzt

  resources:
    requests:
      storage: 5Gi       # 5 Gibibyte Speicher anfordern
```

**Speicher-Größen in Kubernetes:**
```
Ki = Kibibyte  (1024 Bytes)    — nicht zu verwechseln mit KB (1000 Bytes)
Mi = Mebibyte  (1024 Ki)       — nicht zu verwechseln mit MB
Gi = Gibibyte  (1024 Mi)       — nicht zu verwechseln mit GB
Ti = Tebibyte  (1024 Gi)

In der Praxis: 1Gi ≈ 1.07 GB — der Unterschied ist meistens egal.
```

---

## 4. Wie Kubernetes-Objekte zusammenarbeiten

Jetzt wo wir die einzelnen Objekte kennen, schauen wir wie sie miteinander
verbunden sind. Das ist der Schlüssel um zu verstehen warum Kubernetes so
konfiguriert wird wie es ist.

### Das Label-System — Die universelle Verbindung

Labels sind der Klebstoff der alles zusammenhält. Objekte finden einander
durch Labels — nicht durch direkte Referenzen auf Namen.

```
Deployment
  selector.matchLabels:
    app: meine-app        ─────────────────────────────────────┐
                                                               │ "Verwalte alle
  template.metadata.labels:                                    │  Pods mit diesen
    app: meine-app        ─── erstellt Pods mit diesem Label   │  Labels"
                                                               │
Service                                                        │
  spec.selector:                                               │
    app: meine-app        ─────────────────────────────────────┘
                          "Schicke Traffic an alle Pods mit diesem Label"
```

**Warum Labels statt direkter Namen?**

Stell dir vor du hast 3 Pods namens `meine-app-abc123`, `meine-app-def456`,
`meine-app-ghi789`. Pods haben zufällige Suffixes — du kannst nicht vorhersagen
wie sie heißen werden. Labels lösen das: alle drei haben das Label `app: meine-app`
und der Service findet alle drei — automatisch, ohne die exakten Namen zu kennen.

### Der komplette Fluss: Request kommt an

```
Internet-Request für https://meine-app.beispiel.de
        ↓
DNS: beispiel.de → 89.167.116.105 (LoadBalancer-IP)
        ↓
Traefik Pod (hat LoadBalancer Service)
  → Liest IngressRoute: Host(meine-app.beispiel.de) → Service meine-app
        ↓
Service "meine-app" (ClusterIP: 10.96.42.100)
  → Endpoints: [10.244.1.5:8080, 10.244.2.9:8080]
  → kube-proxy wählt zufällig einen Pod
        ↓
Pod (10.244.1.5) mit Container (Port 8080)
  → Container verarbeitet Request
        ↓
Antwort zurück durch denselben Weg
```

### Abhängigkeiten beim Erstellen von Objekten

Reihenfolge ist wichtig wenn du Objekte erstellst:

```
1. Namespace  ← muss zuerst existieren
2. ConfigMap und Secret  ← müssen vor Pods existieren (Pods referenzieren sie)
3. PersistentVolumeClaim  ← muss vor Pod existieren
4. Deployment  ← kann erstellt werden wenn Namespace, Config, Secrets existieren
5. Service  ← kann parallel zum Deployment erstellt werden
6. Certificate  ← kann erstellt werden wenn ClusterIssuer existiert
7. IngressRoute  ← kann erstellt werden wenn Service und Certificate existieren
```

In diesem Setup übernimmt ArgoCD die Reihenfolge automatisch durch seine
Sync-Waves und Abhängigkeitserkennung.

---

## 5. Was ist Helm und warum braucht man es?

### Das Problem das Helm löst

Nehmen wir an du willst Traefik in deinem Cluster installieren.
Traefik braucht:
- 1 Deployment
- 1 ServiceAccount
- 1 ClusterRole
- 1 ClusterRoleBinding
- 1 Service (LoadBalancer)
- 1 ConfigMap
- Mehrere CustomResourceDefinitions (IngressRoute, Middleware, etc.)

Das sind ca. 500-1000 Zeilen YAML. Willst du das alles selbst schreiben?
Und was wenn du ein Update von Traefik 32.x auf 33.x machen willst?

**Helm löst genau das.** Der Traefik-Chart enthält alle diese Dateien fertig
vorbereitet. Du sagst nur: "Ich will 1 Replica, LoadBalancer-Service, TLS an."
Helm kümmert sich um den Rest.

### Was Helm konkret tut

```
Du hast:
  1. Einen Helm Chart (die Templates)
  2. Deine values.yaml (deine Konfiguration)

Helm macht:
  1. Lädt den Chart (entweder lokal oder aus einem Chart-Repository)
  2. Nimmt jedes Template
  3. Ersetzt alle {{ .Values.xyz }} durch Werte aus deiner values.yaml
  4. Wo du keinen Wert angegeben hast: nutzt den Default-Wert des Charts
  5. Rendert fertiges YAML
  6. Wendet es auf Kubernetes an (kubectl apply)

Ergebnis:
  Alle Kubernetes-Objekte sind im Cluster — ohne dass du 500 Zeilen YAML
  selbst schreiben musstest.
```

### Helm Chart Repositories

Helm Charts werden in **Repositories** gespeichert — wie ein App Store für
Kubernetes-Pakete. Bekannte Repositories:

```bash
# Traefik's offizielles Repository
https://helm.traefik.io/traefik

# cert-manager Repository
https://charts.jetstack.io

# Bitnami (viele populäre Apps: PostgreSQL, Redis, WordPress, etc.)
https://charts.bitnami.com/bitnami

# Artifact Hub — Verzeichnis aller öffentlichen Charts
https://artifacthub.io
```

Um einen Chart zu nutzen musst du das Repository kennen. Du findest es immer
in der Dokumentation der jeweiligen Software.

### Chart-Version vs. App-Version

Helm unterscheidet zwischen zwei Versionen:

```yaml
# In Chart.yaml:
version: 33.2.1      # Die Chart-Version (Versionierung der Helm-Dateien)
appVersion: "v3.1.2" # Die Version der eigentlichen App (z.B. Traefik v3.1.2)
```

In der ArgoCD Application gibst du die **Chart-Version** an:
```yaml
targetRevision: "33.x"   # Chart-Version 33.x (x = neueste 33er Version)
targetRevision: "33.2.1" # Exakte Chart-Version
```

Nutze exakte Versionen in Production — `33.x` kann auf eine inkompatible
Version updaten. Bewusste Updates sind besser als automatische.

---

## 6. Helm Chart Struktur — Was ist wo und warum

Ein Helm Chart ist ein Ordner mit einer definierten Verzeichnisstruktur.
Wenn du einen Chart herunterlädst oder lokal erstellst, sieht er immer so aus:

```
mein-chart/
│
├── Chart.yaml          ← PFLICHT: Metadaten des Charts
├── values.yaml         ← PFLICHT: Standard-Konfigurationswerte
│
├── templates/          ← PFLICHT: Die YAML-Templates
│   ├── deployment.yaml      Kubernetes Deployment Template
│   ├── service.yaml         Kubernetes Service Template
│   ├── ingress.yaml         Kubernetes Ingress Template (optional)
│   ├── configmap.yaml       Kubernetes ConfigMap Template (optional)
│   ├── serviceaccount.yaml  ServiceAccount Template (optional)
│   ├── hpa.yaml             HorizontalPodAutoscaler (optional)
│   ├── _helpers.tpl         Wiederverwendbare Template-Teile (kein K8s-Objekt)
│   └── NOTES.txt            Text der nach Installation angezeigt wird
│
├── charts/             ← OPTIONAL: Abhängige Sub-Charts
│   └── postgresql/         z.B. PostgreSQL als Abhängigkeit
│
├── .helmignore         ← OPTIONAL: Welche Dateien ignoriert werden
└── README.md           ← OPTIONAL: Dokumentation des Charts
```

### Chart.yaml — Die Visitenkarte

```yaml
apiVersion: v2              # Helm-API-Version (immer v2 für Helm 3)
name: meine-app             # Name des Charts — muss mit Ordnernamen übereinstimmen
description: |
  Eine Kubernetes-Anwendung für meine tolle App.
  Stellt einen nginx-Server mit konfigurierbaren Replicas bereit.

type: application           # "application" (deployt was) oder "library" (nur Helfer)

version: 0.1.0              # VERSION DES CHARTS — du vergibst das
                            # Erhöhe diese Version wenn du den Chart änderst
                            # Nutze Semantic Versioning: Major.Minor.Patch

appVersion: "1.25.0"        # VERSION DER APP die deployed wird
                            # Informativ — wird oft als Default-Image-Tag genutzt
                            # Ändere das wenn du auf eine neue App-Version updatest

# Optionale aber nützliche Felder:
keywords:
  - webserver
  - nginx
  - production

home: https://github.com/Jakobulus123/K8s-Helm

sources:
  - https://github.com/nginx/nginx

maintainers:
  - name: Jakob
    email: jakob@beispiel.de

dependencies:
  # Wenn dein Chart andere Charts braucht (Sub-Charts):
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled   # Nur wenn postgresql.enabled: true
```

### templates/ — Die eigentliche Magie

Dateien im `templates/` Ordner sind **normale YAML-Dateien**, aber mit
Helm-Template-Syntax (doppelte geschweifte Klammern) angereichert.

Helm verarbeitet jede Datei in `templates/` und ersetzt die Template-Ausdrücke.
Das Ergebnis ist reguläres Kubernetes-YAML.

**Ausnahmen:**
- Dateien die mit `_` beginnen (z.B. `_helpers.tpl`) werden nicht als
  Kubernetes-Objekte behandelt — sie enthalten nur Template-Funktionen
- `NOTES.txt` ist Text der nach `helm install` angezeigt wird

---

## 7. Helm Templates — Wie Variablen funktionieren

### Die Template-Syntax

Helm nutzt die **Go-Template-Syntax** — doppelte geschweifte Klammern für alles
was dynamisch ist:

```yaml
# Festes YAML (kein Template):
name: meine-app

# Template-Ausdruck — wird durch einen Wert ersetzt:
name: {{ .Values.appName }}

# Mit Leerzeichen-Kontrolle (Bindestrich entfernt Whitespace):
name: {{- .Values.appName -}}
```

### Die wichtigsten Template-Variablen

```yaml
# ── WERTE AUS DER values.yaml ──────────────────────────────────
{{ .Values.replicas }}              # values.yaml: replicas: 3
{{ .Values.image.repository }}      # values.yaml: image: \n  repository: nginx
{{ .Values.service.port }}          # values.yaml: service: \n  port: 80

# ── INFORMATIONEN ÜBER DIE HELM-INSTALLATION ───────────────────
{{ .Release.Name }}       # Name des Helm Releases (z.B. "traefik")
{{ .Release.Namespace }}  # Namespace wo deployed wird (z.B. "traefik")
{{ .Release.IsInstall }}  # true wenn erste Installation (nicht Update)
{{ .Release.IsUpgrade }}  # true wenn Update einer bestehenden Installation

# ── INFORMATIONEN ÜBER DEN CHART ───────────────────────────────
{{ .Chart.Name }}         # Name aus Chart.yaml
{{ .Chart.Version }}      # Version aus Chart.yaml
{{ .Chart.AppVersion }}   # AppVersion aus Chart.yaml
```

### Ein echtes Template-Beispiel

So sieht ein typisches Deployment-Template aus:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  # Wenn Release "myapp" und Chart "webapp" → Name: "myapp-webapp"
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicas | default 1 }}
  # "| default 1" bedeutet: wenn .Values.replicas nicht gesetzt ist → nutze 1
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        # Image: nginx:1.25 — oder wenn kein tag in values: nutze AppVersion aus Chart.yaml
        ports:
        - containerPort: {{ .Values.service.targetPort | default 80 }}
        {{- if .Values.resources }}
        # "if" — dieser Block erscheint nur wenn .Values.resources definiert ist
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
          # toYaml: konvertiert das Objekt zu YAML-Text
          # nindent 10: fügt 10 Leerzeichen Einrückung hinzu
        {{- end }}
```

**Du musst Templates nicht selbst schreiben.** Du nutzt fertige Charts von
anderen und konfigurierst sie nur mit deiner `values.yaml`. Das Verstehen
der Template-Syntax hilft dir aber wenn du Fehler debuggst oder den Chart
anpassen musst.

### Hilfreiche Template-Funktionen

```yaml
# default — Fallback-Wert
{{ .Values.replicas | default 1 }}        # Wenn nicht gesetzt → 1
{{ .Values.image.tag | default "latest" }} # Wenn nicht gesetzt → "latest"

# quote — Anführungszeichen hinzufügen
{{ .Values.appName | quote }}             # "meine-app" (mit Anführungszeichen)

# upper / lower — Groß-/Kleinschreibung
{{ .Values.umgebung | upper }}            # "PRODUCTION"
{{ .Values.umgebung | lower }}            # "production"

# if/else — Bedingungen
{{- if .Values.ingress.enabled }}
# Dieser Block erscheint nur wenn ingress.enabled: true
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# range — Schleifen (für Listen)
{{- range .Values.extraEnv }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}

# toYaml — Objekt zu YAML konvertieren
resources:
  {{- toYaml .Values.resources | nindent 10 }}
```

---

## 8. values.yaml — Das Herzstück der Konfiguration

### Wie values.yaml wirklich funktioniert

Jeder Helm Chart hat eine eingebaute `values.yaml` mit Standard-Werten.
Du schreibst deine eigene `values.yaml` die nur die Werte enthält die du
anders haben willst. Helm **merged** beide zusammen:

```
Chart-eigene values.yaml (Standard-Werte):
  replicas: 1
  image:
    repository: nginx
    tag: "1.25"
  service:
    type: ClusterIP
    port: 80
  resources:
    requests:
      memory: "64Mi"
      cpu: "50m"
  ingress:
    enabled: false

Deine values.yaml (nur was du änderst):
  replicas: 3
  service:
    type: LoadBalancer

Ergebnis nach dem Merge (was Helm wirklich nutzt):
  replicas: 3              ← von dir überschrieben
  image:
    repository: nginx      ← Standard (von dir nicht geändert)
    tag: "1.25"            ← Standard
  service:
    type: LoadBalancer     ← von dir überschrieben
    port: 80               ← Standard (du hast nur type geändert, port bleibt)
  resources:
    requests:
      memory: "64Mi"       ← Standard
      cpu: "50m"           ← Standard
  ingress:
    enabled: false         ← Standard
```

**Das ist sehr mächtig:** Du kannst einen 500-Zeilen-Chart mit einer
10-Zeilen values.yaml konfigurieren. Du schreibst nur was abweicht.

### Wie du die values.yaml eines fremden Charts liest

Wenn du einen Chart nutzen willst, schau dir zuerst dessen values.yaml an:

```bash
# Zeigt die komplette default values.yaml des Charts
helm show values traefik/traefik
helm show values jetstack/cert-manager

# In eine Datei speichern um sie bequem zu lesen
helm show values traefik/traefik > /tmp/traefik-defaults.yaml
```

Die default values.yaml eines Charts ist gleichzeitig die beste Dokumentation:
Sie zeigt alle verfügbaren Optionen mit ihren Standard-Werten und oft mit
Kommentaren die erklären was jede Option macht.

### Struktur einer guten values.yaml

```yaml
# ── ALLGEMEIN ──────────────────────────────────────────────────
# Anzahl der Pods
replicaCount: 1

# ── IMAGE ──────────────────────────────────────────────────────
image:
  repository: nginx         # Welches Docker-Image?
  tag: "1.25"               # Immer als String (mit Anführungszeichen!)
  pullPolicy: IfNotPresent  # Wann soll das Image gepullt werden?
  # IfNotPresent: nur wenn nicht lokal vorhanden (am häufigsten)
  # Always: immer beim Start (gut für :latest Tags)
  # Never: niemals (Image muss auf der Node vorhanden sein)

# ── SERVICE ────────────────────────────────────────────────────
service:
  type: ClusterIP    # ClusterIP / NodePort / LoadBalancer
  port: 80

# ── INGRESS ────────────────────────────────────────────────────
ingress:
  enabled: false     # true = Ingress wird erstellt, false = nicht
  # Dieses Pattern (enabled: true/false) ist in Charts sehr verbreitet.
  # Viele optionale Funktionen haben so einen Schalter.

# ── RESSOURCEN ─────────────────────────────────────────────────
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "200m"

# ── STORAGE ────────────────────────────────────────────────────
persistence:
  enabled: false      # Kein persistenter Speicher standardmäßig
  size: "1Gi"
  storageClass: ""    # Leerer String = Standard-StorageClass nutzen

# ── UMGEBUNGSVARIABLEN ─────────────────────────────────────────
env: {}               # Keine Umgebungsvariablen standardmäßig
# So kannst du welche hinzufügen:
# env:
#   LOG_LEVEL: "info"
#   APP_ENV: "production"

# ── EXTRA-KONFIGURATION ────────────────────────────────────────
nodeSelector: {}      # Auf welchen Nodes soll der Pod laufen? (leer = überall)
tolerations: []       # Welche Node-Taints werden toleriert?
affinity: {}          # Komplexere Node-Auswahl-Regeln
```

### Die echte traefik/values.yaml aus diesem Repo erklärt

```yaml
# values/traefik/values.yaml — deine echte Konfiguration

deployment:
  replicas: 1
  # Warum 1? Ein kleiner Cluster braucht nur einen Traefik-Pod.
  # Bei mehr Last: auf 2-3 erhöhen für Ausfallsicherheit.

service:
  type: LoadBalancer
  # Traefik ist der einzige Eintrittspunkt von außen.
  # LoadBalancer → Hetzner Cloud erstellt automatisch eine externe IP.
  # Diese IP muss in DNS eingetragen sein (→ dein Domain-Provider).

ports:
  web:
    port: 8000          # Intern: Traefik-Container lauscht auf 8000
    expose:
      default: true     # Dieser Port soll nach außen geöffnet werden
    exposedPort: 80     # Nach außen: Port 80 (HTTP)
    protocol: TCP
    # Mapping: Anfragen auf Port 80 von außen → Port 8000 im Container
    # (Der LoadBalancer-Service macht dieses Mapping)

  websecure:
    port: 8443          # Intern: Port 8443
    expose:
      default: true
    exposedPort: 443    # Nach außen: Port 443 (HTTPS)
    protocol: TCP
    tls:
      enabled: true     # Traefik erwartet TLS-Traffic auf diesem EntryPoint

ingressRoute:
  dashboard:
    enabled: false
    # Das Traefik-Dashboard (Übersicht aller Routes) ist deaktiviert.
    # Warum? Sicherheit — das Dashboard soll nicht öffentlich sein.
    # Zum Debuggen: kubectl port-forward -n traefik svc/traefik 9000:9000

providers:
  kubernetesCRD:
    enabled: true
    # Traefik soll IngressRoute, Middleware, etc. Objekte aus K8s lesen.
    # Ohne das: Traefik ignoriert alle IngressRoute-Dateien!

  kubernetesIngress:
    enabled: true
    # Traefik soll auch Standard-Kubernetes-Ingress-Objekte lesen.
    # Gut für Kompatibilität mit Charts die Standard-Ingress nutzen.

logs:
  general:
    level: INFO         # Logging-Level: DEBUG, INFO, WARN, ERROR
  access:
    enabled: true       # Access-Logs: jeder HTTP-Request wird geloggt

metrics:
  prometheus:
    enabled: true       # Metriken für Prometheus aktivieren
    entryPoint: metrics # Auf welchem EntryPoint sind Metriken erreichbar
```

---

## 9. ArgoCD Application — Der Kleber zwischen Git und Cluster

### Warum braucht man ArgoCD?

Ohne ArgoCD: Du änderst eine YAML-Datei und musst manuell `kubectl apply -f datei.yaml`
ausführen. Du musst selbst daran denken. Es gibt kein Audit-Log. Rollback ist
manuell und fehleranfällig.

Mit ArgoCD: Du pushst die Änderung nach Git. ArgoCD erkennt sie, deployt sie
automatisch, und du siehst in der UI ob alles funktioniert hat. Git ist der
Audit-Log. Rollback = git revert.

### Aufbau einer ArgoCD Application — jedes Feld erklärt

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: meine-app
  # Name der Application in ArgoCD. Erscheint in der UI.
  # Muss im argocd-Namespace eindeutig sein.
  # Gut: beschreibend und kurz — "traefik", "cert-manager", "monitoring"

  namespace: argocd
  # IMMER "argocd". Application-Objekte leben im argocd-Namespace.
  # Das ist eine Eigenheit von ArgoCD — egal was deployed wird.

  finalizers:
    - resources-finalizer.argocd.argoproj.io
  # Was passiert wenn diese Application gelöscht wird?
  # Mit Finalizer: ArgoCD löscht AUCH alle deployten Kubernetes-Objekte.
  # Ohne Finalizer: Application wird gelöscht, K8s-Objekte bleiben.
  #
  # Für dieses Setup: Finalizer an lassen. Wenn du eine App entfernst,
  # soll auch alles sauber aufgeräumt werden.

spec:
  project: default
  # ArgoCD-Projects sind wie Ordner für Applications — mit eigenem RBAC.
  # "default" ist das Standard-Project — gut genug für dieses Setup.
  # Mehrere Projects brauchst du nur wenn mehrere Teams mit unterschiedlichen
  # Berechtigungen auf denselben ArgoCD-Server zugreifen.

  # ────────────────────────────────────────────────────────────────
  # SOURCE — Wo kommen die Manifeste her?
  # ────────────────────────────────────────────────────────────────

  # VARIANTE A: Rohes YAML aus einem Git-Ordner
  source:
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    # Das Git-Repository. Muss in ArgoCD als Repository registriert sein
    # (unter Settings → Repositories in der ArgoCD UI).
    # SSH-Format: git@github.com:USER/REPO.git
    # HTTPS-Format: https://github.com/USER/REPO.git

    targetRevision: HEAD
    # Welcher Stand des Repos? HEAD = neuester Commit im Standard-Branch.
    # Alternativen:
    # "main"        → Branch "main"
    # "develop"     → Branch "develop"
    # "v1.2.3"      → Git-Tag (exakte Version — gut für Stabilität)
    # "a1b2c3d4"    → Exakter Commit-Hash (maximale Kontrolle)

    path: manifests/meine-app
    # Welcher Ordner im Repo enthält die YAML-Dateien?
    # ArgoCD wendet ALLE .yaml Dateien in diesem Ordner an.

    directory:
      recurse: true
      # Auch Unterordner durchsuchen? true = ja.
      # Gut wenn du Dateien in Unterordnern organisierst.

  # VARIANTE B: Helm Chart aus Chart-Repository
  source:
    repoURL: https://helm.traefik.io/traefik
    # URL des Helm Chart Repositories (nicht dein GitHub-Repo!)

    chart: traefik
    # Name des Charts im Repository.

    targetRevision: "33.2.1"
    # Chart-Version. Exakte Version für Stabilität empfohlen.
    # "33.x" = neueste 33er Version (kann unerwartet updaten)

    helm:
      releaseName: traefik
      # Name des Helm Releases — erscheint in "helm list"
      # Wird oft in Objektnamen verwendet (z.B. "traefik-deployment")

      values: |
        # Direkt hier YAML-Werte angeben (für kleine Konfigurationen)
        replicas: 1
        service:
          type: LoadBalancer

      valueFiles:
        - values.yaml
        # Werte aus einer Datei die im selben Repo wie der Chart liegt.
        # Da der Chart extern ist, geht das nur mit der zweiten Quelle (unten).

  # VARIANTE C: Helm Chart + eigene Values aus separatem Git-Repo
  # (Das ist was in diesem Setup genutzt wird!)
  sources:          # ← "sources" mit "s" — Plural für mehrere Quellen!
    - repoURL: https://helm.traefik.io/traefik
      chart: traefik
      targetRevision: "33.x"
      helm:
        releaseName: traefik
        valueFiles:
          - $values/values/traefik/values.yaml
          # "$values" ist eine Referenz auf die zweite Quelle (ref: values)
          # Der Pfad ist relativ zum Root des referenzierten Repos.

    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
      # "ref: values" macht diesen Repo-Eintrag zur "$values" Referenz.
      # ArgoCD cloned dieses Repo und stellt es als "$values" bereit.
      # So können valueFiles auf Dateien in DEINEM Repo zeigen,
      # obwohl der Chart aus einem anderen Repo kommt.

  # ────────────────────────────────────────────────────────────────
  # DESTINATION — Wohin wird deployed?
  # ────────────────────────────────────────────────────────────────
  destination:
    server: https://kubernetes.default.svc
    # In welchen Kubernetes-Cluster? Dieser Wert bedeutet "der lokale Cluster"
    # (wo ArgoCD selbst läuft). Für externe Cluster: URL des API-Servers.

    namespace: meine-app
    # In welchen Namespace werden die Objekte deployed?
    # Wenn der Namespace nicht existiert und CreateNamespace=true gesetzt:
    # ArgoCD erstellt ihn automatisch.

  # ────────────────────────────────────────────────────────────────
  # SYNC POLICY — Wann und wie wird synchronisiert?
  # ────────────────────────────────────────────────────────────────
  syncPolicy:
    automated:
      prune: true
      # Was passiert wenn ein Objekt in Git gelöscht wird?
      # prune: true  → ArgoCD löscht das K8s-Objekt automatisch
      # prune: false → Objekt bleibt im Cluster (muss manuell gelöscht werden)
      # Empfehlung: true — sonst sammeln sich verwaiste Objekte an

      selfHeal: true
      # Was passiert wenn jemand direkt kubectl apply/edit nutzt?
      # selfHeal: true  → ArgoCD überschreibt die manuelle Änderung (zurück zu Git)
      # selfHeal: false → Manuelle Änderungen bleiben
      # Empfehlung: true — Git ist die Wahrheit, manuelle Änderungen sind verboten

    syncOptions:
      - CreateNamespace=true
      # Namespace automatisch erstellen wenn er nicht existiert.
      # Ohne das: ArgoCD schlägt fehl wenn Namespace fehlt.

      - ServerSideApply=true
      # Nutzt kubectl apply --server-side statt client-side.
      # Besser für große Objekte (CRDs etc.) — vermeidet Annotation-Overflow.
      # Standardmäßig nicht nötig, aber für cert-manager empfohlen.

      - RespectIgnoreDifferences=true
      # Kombiniert mit ignoreDifferences unten — ignoriert bestimmte Felder
      # beim Vergleich von Git- und Cluster-Zustand.

  # ────────────────────────────────────────────────────────────────
  # IGNORE DIFFERENCES — Was soll ArgoCD ignorieren?
  # ────────────────────────────────────────────────────────────────
  ignoreDifferences:
  # Manchmal gibt es Felder die Kubernetes automatisch ändert und die
  # ArgoCD nicht als "OutOfSync" markieren soll.
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jqPathExpressions:
      - .spec.conversion
      - .spec.preserveUnknownFields
  # Für cert-manager CRDs: Kubernetes fügt automatisch Felder hinzu die
  # in Git nicht stehen. Ohne ignoreDifferences → immer OutOfSync.
  # Mit ignoreDifferences → diese Felder werden beim Vergleich ignoriert.
```

---

## 10. Das App-of-Apps Pattern in diesem Setup

### Die Idee hinter App-of-Apps

Das eleganteste Pattern in ArgoCD. Die Idee: eine ArgoCD Application
(die "Root Application") überwacht einen Ordner in Git der selbst wieder
ArgoCD Application-Definitionen enthält.

```
Root Application
  → überwacht apps/ Ordner
    → entdeckt apps/traefik.yaml     → erstellt Application "traefik"
    → entdeckt apps/cert-manager.yaml → erstellt Application "cert-manager"
    → entdeckt apps/manifests.yaml   → erstellt Application "manifests"
```

### Die Root Application

```yaml
# bootstrap/root-app.yaml — einmalig manuell anwenden!
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
    path: apps              # ← Überwacht den apps/ Ordner
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd       # ← Applications landen im argocd-Namespace
  syncPolicy:
    automated:
      prune: true           # Gelöschte Apps in apps/ werden auch gelöscht
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Die Ordnerstruktur des gesamten Setups

```
K8s-Helm/
│
├── bootstrap/
│   └── root-app.yaml          ← EINMALIG: kubectl apply -f bootstrap/root-app.yaml
│
├── apps/                      ← ArgoCD Application-Definitionen
│   ├── traefik.yaml           ← "Installiere Traefik"
│   ├── cert-manager.yaml      ← "Installiere cert-manager"
│   └── manifests.yaml         ← "Deploye alle YAMLs aus manifests/"
│
├── values/                    ← Helm-Konfiguration
│   ├── traefik/
│   │   └── values.yaml        ← "Traefik so konfigurieren"
│   └── cert-manager/
│       └── values.yaml        ← "cert-manager so konfigurieren"
│
└── manifests/                 ← Rohes Kubernetes-YAML
    └── argocd/
        └── ingress.yaml       ← ArgoCD per HTTPS erreichbar machen
```

### Wie eine neue App hinzukommt

Du willst eine neue App hinzufügen? Das reicht:

```
1. Neue Datei apps/meine-app.yaml erstellen
2. Falls Helm: values/meine-app/values.yaml erstellen
3. Falls rohes YAML: manifests/meine-app/*.yaml erstellen
4. git add + git commit + git push

ArgoCD entdeckt automatisch:
 → Root Application sieht neue Datei apps/meine-app.yaml
 → Erstellt neue Application "meine-app"
 → Application deployt ihre Inhalte
 → Fertig ✓
```

**Du musst nie wieder direkt kubectl apply ausführen.**
Du musst nie ArgoCD manuell konfigurieren.
Du musst dir nie merken was wo deployed ist — Git ist das Gedächtnis.

---

## 11. Schritt für Schritt: Neue App anlegen

Wir gehen eine komplette neue App durch. Als Beispiel nehmen wir
**Uptime Kuma** — ein Monitoring-Dashboard das prüft ob deine Dienste
erreichbar sind.

---

### Schritt 0: Entscheiden — Helm oder rohes YAML?

**Helm nutzen wenn:**
- Ein offizieller oder gut gewarteter Chart existiert
- Du viele Parameter konfigurieren willst ohne YAML zu schreiben
- Du Updates einfach über Chart-Versionen machen willst

**Rohes YAML nutzen wenn:**
- Kein Chart existiert
- Der Chart zu komplex ist für was du brauchst
- Du maximale Kontrolle über jeden Aspekt willst

Für Uptime Kuma: Es gibt einen Community-Chart, aber er ist nicht sehr gut
gepflegt. Rohes YAML ist hier einfacher und transparenter.

---

### Schritt 1: ArgoCD Application erstellen

Datei: `apps/uptime-kuma.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: uptime-kuma                  # Name in ArgoCD UI
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:Jakobulus123/K8s-Helm.git
    targetRevision: HEAD
    path: manifests/uptime-kuma      # Zeigt auf Schritt 2
    directory:
      recurse: true                  # Unterordner auch durchsuchen
  destination:
    server: https://kubernetes.default.svc
    namespace: uptime-kuma           # Eigener Namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true         # Namespace automatisch erstellen
```

---

### Schritt 2: Kubernetes-Manifeste erstellen

Ordner erstellen: `manifests/uptime-kuma/`

**Datei: `manifests/uptime-kuma/deployment.yaml`**

```yaml
---
# Kein separates Namespace-Manifest nötig — CreateNamespace=true macht das
# ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
  labels:
    app: uptime-kuma
spec:
  replicas: 1                # Uptime Kuma braucht nur 1 Instanz
                             # (hat interne SQLite-DB — nicht horizontal skalierbar)
  selector:
    matchLabels:
      app: uptime-kuma
  strategy:
    type: Recreate           # SQLite-DB verträgt keine zwei gleichzeitigen Schreiber
                             # → Erst alten Pod stoppen, dann neuen starten
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
      - name: uptime-kuma
        image: louislam/uptime-kuma:1    # Offizielles Docker-Image
        ports:
        - containerPort: 3001
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "300m"
        volumeMounts:
        - name: data
          mountPath: /app/data           # Uptime Kuma speichert Daten hier
        livenessProbe:
          httpGet:
            path: /
            port: 3001
          initialDelaySeconds: 30        # Uptime Kuma braucht ~20s zum Starten
          periodSeconds: 30
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 3001
          initialDelaySeconds: 15
          periodSeconds: 10
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: uptime-kuma-data    # Referenz auf PVC unten
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uptime-kuma-data
  namespace: uptime-kuma
spec:
  accessModes:
    - ReadWriteOnce     # Nur ein Pod schreibt gleichzeitig (passend zu replicas: 1)
  resources:
    requests:
      storage: 1Gi      # 1GB für Monitoring-Daten — großzügig bemessen
---
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  selector:
    app: uptime-kuma    # Traffic geht an Pods mit diesem Label
  ports:
  - name: http
    port: 3001          # Service lauscht auf 3001
    targetPort: 3001    # Container auch auf 3001 — hier gleich
```

---

### Schritt 3: HTTPS einrichten

**Datei: `manifests/uptime-kuma/ingress.yaml`**

```yaml
---
# SCHRITT 3a: TLS-Zertifikat bei Let's Encrypt beantragen
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: uptime-kuma-tls
  namespace: uptime-kuma
spec:
  secretName: uptime-kuma-tls
  # cert-manager erstellt ein Secret mit diesem Namen.
  # Das Secret enthält tls.crt und tls.key.
  # Traefik liest es für den TLS Handshake.

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    # Nutzt den bestehenden ClusterIssuer der schon für ArgoCD funktioniert.

  dnsNames:
    - status.deine-domain.de
    # HIER DEINE ECHTE DOMAIN EINTRAGEN.
    # Diese Domain muss per DNS auf die LoadBalancer-IP zeigen!
    # Ohne korrekten DNS-Eintrag schlägt die Let's Encrypt Challenge fehl.
---
# SCHRITT 3b: HTTPS-Route — Traefik leitet Traffic zur App weiter
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: uptime-kuma
  namespace: uptime-kuma
spec:
  entryPoints:
    - websecure              # Port 443 (HTTPS EntryPoint in Traefik)
  routes:
    - match: Host(`status.deine-domain.de`)
      # Traefik prüft den Host-Header des Requests.
      # Passt er? → Weiterleiten zu uptime-kuma Service.
      # Passt nicht? → Nächste Route prüfen.
      kind: Rule
      services:
        - name: uptime-kuma  # Name des K8s-Services (oben definiert)
          port: 3001         # Port des K8s-Services
  tls:
    secretName: uptime-kuma-tls   # Das Secret vom Certificate oben
---
# SCHRITT 3c: HTTP → HTTPS Redirect
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: uptime-kuma-http
  namespace: uptime-kuma
spec:
  entryPoints:
    - web                    # Port 80 (HTTP)
  routes:
    - match: Host(`status.deine-domain.de`)
      kind: Rule
      middlewares:
        - name: redirect-https
          namespace: argocd  # Diese Middleware ist global im argocd-Namespace
                             # und kann von überall referenziert werden
      services:
        - name: uptime-kuma
          port: 3001
```

---

### Schritt 4: DNS-Eintrag setzen

Bevor du pushst: Geh zu deinem Domain-Anbieter und erstelle einen A-Record:

```
Name:  status            (oder was auch immer vor deiner Domain)
Typ:   A
Wert:  89.167.116.105    (die LoadBalancer-IP von Traefik)
TTL:   300 (5 Minuten)
```

Prüfen ob der Eintrag schon aktiv ist:
```bash
nslookup status.deine-domain.de
dig status.deine-domain.de
```

**Warum zuerst DNS?** Let's Encrypt verifiziert die Domain via HTTP-Challenge.
Das funktioniert nur wenn DNS schon korrekt eingetragen ist. Falsche Reihenfolge
→ Zertifikat-Ausstellung schlägt fehl.

---

### Schritt 5: Git Push

```bash
# Alle neuen Dateien zum Staging hinzufügen
git add apps/uptime-kuma.yaml
git add manifests/uptime-kuma/

# Commit mit aussagekräftiger Nachricht
git commit -m "Add uptime-kuma monitoring dashboard"

# Push zu GitHub
git push origin main
```

---

### Schritt 6: In ArgoCD und Kubernetes verfolgen

```bash
# Warte ca. 1-3 Minuten, dann prüfen:

# ArgoCD Application Status
kubectl get applications -n argocd uptime-kuma

# Pods prüfen
kubectl get pods -n uptime-kuma

# Wenn Pod crasht — Logs anschauen:
kubectl logs -n uptime-kuma deployment/uptime-kuma

# Certificate-Status prüfen (braucht 1-3 Minuten)
kubectl get certificate -n uptime-kuma
kubectl describe certificate uptime-kuma-tls -n uptime-kuma
```

---

### Was dann automatisch passiert

```
t=0   Du pushst nach GitHub
t=3m  ArgoCD erkennt neue Datei apps/uptime-kuma.yaml
      → Erstellt Application "uptime-kuma" in ArgoCD
t=3m  Application synct: ArgoCD wendet manifests/uptime-kuma/*.yaml an
      → Namespace "uptime-kuma" wird erstellt
      → PVC "uptime-kuma-data" wird erstellt
      → Deployment "uptime-kuma" wird erstellt
      → Service "uptime-kuma" wird erstellt
      → Certificate "uptime-kuma-tls" wird erstellt
      → IngressRoutes werden erstellt
t=4m  kube-scheduler weist Pod einer Node zu
      kubelet startet Container
      Pod Status: Pending → Running
t=4m  cert-manager sieht neues Certificate-Objekt
      → Stellt Anfrage bei Let's Encrypt
      → Erstellt HTTP-Challenge
      → Let's Encrypt ruft http://status.deine-domain.de/.well-known/acme-challenge/...
      → Wenn DNS korrekt: Challenge erfolgreich ✓
      → TLS-Zertifikat wird ausgestellt
      → Landet in Secret "uptime-kuma-tls"
t=6m  Traefik liest Secret → TLS aktiv
      App per HTTPS erreichbar: https://status.deine-domain.de ✓
```

---

## 12. Echte Beispiele aus diesem Repo — vollständig erklärt

### Die traefik.yaml in apps/ — Zeile für Zeile

```yaml
# apps/traefik.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  # Erscheint in ArgoCD UI als "traefik"
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
    # Wenn ich "traefik" aus ArgoCD lösche → auch der Traefik-Namespace
    # und alle K8s-Objekte darin werden gelöscht. Sauber.
spec:
  project: default
  sources:
    # QUELLE 1: Der eigentliche Helm Chart von Traefik's offiziellem Repo
    - repoURL: https://helm.traefik.io/traefik
      chart: traefik               # Chart-Name
      targetRevision: "33.x"       # Neueste 33er Version
      helm:
        releaseName: traefik       # Helm-Release-Name (erscheint in helm list)
        valueFiles:
          - $values/values/traefik/values.yaml
          # $values → referenziert die zweite Quelle unten
          # /values/traefik/values.yaml → Pfad in DEINEM Repo

    # QUELLE 2: Dein GitHub-Repo — nur als Quelle für die values.yaml
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
      # ref: values → macht diesen Repo-Klon zur "$values" Referenz
      # ArgoCD cloned BEIDE Repos und verknüpft sie

  destination:
    server: https://kubernetes.default.svc
    namespace: traefik             # Traefik bekommt eigenen Namespace
  syncPolicy:
    automated:
      prune: true                  # Gelöschte Traefik-Objekte werden aufgeräumt
      selfHeal: true               # Manuelle kubectl-Änderungen werden rückgängig
    syncOptions:
      - CreateNamespace=true       # Namespace "traefik" auto-erstellen
```

### Das ArgoCD Ingress Manifest — warum jedes Objekt existiert

```yaml
# manifests/argocd/ingress.yaml — 4 Objekte in einer Datei

---
# OBJEKT 1: TLS-Zertifikat für ArgoCD's Domain
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd
  # Im argocd-Namespace, weil das TLS-Secret auch dort sein muss
  # (Traefik und ArgoCD Server müssen auf dasselbe Secret zugreifen)
spec:
  secretName: argocd-tls
  # cert-manager erstellt dieses Secret.
  # Traefik liest es (im IngressRoute unten referenziert).
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    # ClusterIssuer (nicht Issuer!) → funktioniert für alle Namespaces
    # Ein normaler Issuer wäre nur für einen Namespace.
  dnsNames:
    - jakob-argocd.goava.ai
    # Die Domain für die ArgoCD erreichbar sein soll.
    # Let's Encrypt stellt nur dann ein Zertifikat aus wenn:
    # 1. Diese Domain per DNS auf die richtige IP zeigt
    # 2. Der HTTP-01 Challenge-Request erfolgreich ist

---
# OBJEKT 2: HTTPS-Route für ArgoCD
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd
  namespace: argocd
spec:
  entryPoints:
    - websecure              # Nur HTTPS (Port 443)
  routes:
    - match: Host(`jakob-argocd.goava.ai`)
      # Backticks sind Traefik-Syntax — nicht verwechseln mit YAML-Strings!
      # Traefik wertet diesen Ausdruck aus: "Wenn Host-Header = jakob-argocd.goava.ai"
      kind: Rule
      services:
        - name: argocd-server
          # Das ist der K8s-Service den ArgoCD selbst erstellt hat.
          # kubectl get svc -n argocd → du siehst "argocd-server"
          port: 80           # ArgoCD-Server lauscht intern auf Port 80
      middlewares:
        - name: argocd-headers   # Siehe Objekt 4 unten
  tls:
    secretName: argocd-tls       # Das Secret von Objekt 1 oben

---
# OBJEKT 3: HTTP → HTTPS Redirect für ArgoCD
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-http
  namespace: argocd
spec:
  entryPoints:
    - web                    # HTTP (Port 80)
  routes:
    - match: Host(`jakob-argocd.goava.ai`)
      kind: Rule
      middlewares:
        - name: redirect-https   # Leitet auf HTTPS um
          # Diese Middleware ist direkt darunter (Objekt 4) definiert
      services:
        - name: argocd-server
          port: 80
# Warum zwei IngressRoutes statt einer?
# Eine IngressRoute kann nur auf EINEM entryPoint lauschen.
# HTTP und HTTPS sind zwei verschiedene EntryPoints → zwei IngressRoutes nötig.

---
# OBJEKT 4: Middleware — HTTP auf HTTPS umleiten
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true    # 308 Permanent Redirect (Browser speichert das)
                       # permanent: false → 307 Temporary Redirect

---
# OBJEKT 5: Middleware — ArgoCD Header setzen
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-headers
  namespace: argocd
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: https
# WARUM dieser Header?
# Situation: Browser → HTTPS → Traefik → HTTP → ArgoCD
# Das Problem: ArgoCD sieht nur HTTP (den internen Teil).
# ArgoCD denkt: "Ich bin HTTP, ich muss auf HTTPS redirecten."
# → Endlosschleife!
#
# Lösung: Der Header sagt ArgoCD: "Der Proxy vor dir macht TLS."
# "X-Forwarded-Proto: https" → ArgoCD weiß: Ich bin hinter einem HTTPS-Proxy.
# → Kein Self-Redirect. ArgoCD antwortet normal. ✓
#
# Dieses Problem nennt man "Reverse Proxy Header" — fast jede App
# die hinter einem Proxy läuft braucht so eine Konfiguration.
```

---

## 13. Häufige Fehler, ihre Ursachen und Lösungen

### YAML-Syntaxfehler

**Fehler 1: Tab statt Leerzeichen**
```yaml
# FALSCH — Tab vor "replicas"
spec:
	replicas: 3    # Dieser Tab verursacht: yaml: line 3: found character '\t'

# RICHTIG — 2 Leerzeichen
spec:
  replicas: 3
```
**Diagnose:** `kubectl apply -f datei.yaml` gibt sofort einen Parse-Fehler.
**Lösung:** Editor konfigurieren damit Tab-Taste Leerzeichen einfügt.
In VS Code: `"editor.insertSpaces": true, "editor.tabSize": 2`

---

**Fehler 2: Falsche Einrückungstiefe**
```yaml
# FALSCH — containers auf falscher Ebene
spec:
  template:
    spec:
      containers:
      - name: app
          image: nginx    # ← 10 Leerzeichen statt 8 — kaputt!

# RICHTIG
spec:
  template:
    spec:
      containers:
      - name: app
        image: nginx      # ← 8 Leerzeichen (4 für Liste + 4 für Inhalt)
```

---

**Fehler 3: Wert wird als falschem Typ interpretiert**
```yaml
# FALSCH — YAML liest "33.x" als Zahl und schlägt fehl
targetRevision: 33.x

# RICHTIG — String erzwingen
targetRevision: "33.x"

# Weitere Fallen:
port: "8080"     # Manchmal muss es eine Zahl sein, manchmal ein String
                 # → Schau in der Dokumentation nach
enabled: "true"  # Als String statt Boolean — kann Probleme machen
enabled: true    # RICHTIG: Boolean
```

---

### Kubernetes-Fehler

**Fehler 4: Labels stimmen nicht überein (häufigster Anfänger-Fehler)**
```yaml
# FALSCH — Tippfehler im Label
spec:
  selector:
    matchLabels:
      app: meine-app     # ← "meine-app"

  template:
    metadata:
      labels:
        app: meineApp    # ← "meineApp" (anderer Name!) → Deployment findet Pods nicht!
```

**Symptom:** `kubectl get deployment meine-app` zeigt `0/2 READY` obwohl Pods laufen.

**Diagnose:**
```bash
kubectl get pods -n <namespace> --show-labels
# Zeigt welche Labels die Pods haben

kubectl describe deployment meine-app -n <namespace>
# Zeigt den Selector des Deployments
```

---

**Fehler 5: Service-Port und Container-Port passen nicht**
```yaml
# Die App im Container lauscht auf 8080
containers:
- containerPort: 8080

# Aber der Service schickt Traffic an Port 80
service:
  ports:
  - port: 80
    targetPort: 80   # ← 80 statt 8080!
```

**Symptom:** Requests an den Service geben `Connection refused` zurück.

**Diagnose:**
```bash
kubectl get service meine-app -n <namespace> -o yaml | grep -A5 ports
# Zeigt targetPort

kubectl exec -it <pod-name> -n <namespace> -- netstat -tlnp
# Zeigt auf welchem Port die App wirklich lauscht
```

---

**Fehler 6: Falscher Namespace**
```yaml
# Service ist in Namespace "app"
metadata:
  namespace: app

# IngressRoute versucht auf Service in Namespace "default" zuzugreifen
services:
- name: meine-app
  port: 80
  # namespace fehlt! → Traefik sucht in seinem eigenen Namespace
```

**Lösung:** Namespace explizit angeben:
```yaml
services:
- name: meine-app
  namespace: app    # ← Explizit angeben
  port: 80
```

---

**Fehler 7: Image kann nicht gepullt werden**
```
Symptom: Pod Status = "ImagePullBackOff" oder "ErrImagePull"
```

Ursachen:
```bash
# Ursache 1: Image-Name falsch getippt
image: ngnix:1.25    # "ngnix" statt "nginx"

# Ursache 2: Tag existiert nicht
image: nginx:1.99    # Version 1.99 gibt es nicht

# Ursache 3: Privates Registry ohne Credentials
image: meineprivaterepo.io/meine-app:latest
# → Secret mit Registry-Credentials fehlt

# Diagnose:
kubectl describe pod <pod-name> -n <namespace>
# Schau in "Events:" — dort steht der genaue Fehler
```

---

### ArgoCD-Fehler

**Fehler 8: Application bleibt "OutOfSync"**

Mögliche Ursachen:
```bash
# In ArgoCD UI: Details → "Show Diff" anklicken
# Oder per CLI:
kubectl describe application meine-app -n argocd
```

Häufige Ursachen:
- Kubernetes fügt automatisch Felder hinzu die nicht in Git stehen
  → Lösung: `ignoreDifferences` konfigurieren
- `prune: false` und Objekte wurden manuell gelöscht
- Git und Cluster haben wirklich unterschiedliche Werte

---

**Fehler 9: Sync schlägt fehl**
```bash
# In ArgoCD UI: Application → Sync → "Show Details"
# Oder:
kubectl get events -n <ziel-namespace> --sort-by='.lastTimestamp'
# Zeigt Kubernetes-Events — oft erklärt der Fehler hier
```

---

**Fehler 10: Certificate bleibt "False" / Pending**

```bash
kubectl describe certificate meine-app-tls -n <namespace>
# Schau in Events — was schlägt fehl?

kubectl get challenges -A
# Aktive ACME Challenges anzeigen

kubectl describe challenge -n <namespace> <challenge-name>
# Details zur Challenge — oft: DNS-Problem oder Firewall blockiert Port 80
```

Häufige Ursachen:
- DNS-Eintrag fehlt oder zeigt auf falsche IP
- Port 80 ist nicht erreichbar (Firewall!)
- Let's Encrypt Rate Limit erreicht (max 5 fehlgeschlagene Versuche pro Stunde)
- ClusterIssuer Name stimmt nicht überein

---

## 14. Wie du unbekannte Charts liest und verstehst

Wenn du einen neuen Chart nutzen willst, geh so vor:

### Schritt 1: Chart auf Artifact Hub finden

Gehe auf `artifacthub.io` und suche nach der App.
Wichtige Infos auf der Chart-Seite:
- Chart-Repository URL (für `repoURL`)
- Aktuellste Version (für `targetRevision`)
- Link zur values.yaml oder Dokumentation

### Schritt 2: Default values.yaml anschauen

```bash
# Repository temporär hinzufügen (nur zum Nachschauen)
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

# Values anzeigen
helm show values traefik/traefik | less
```

Schau dir folgende Teile besonders an:
- Was ist der Standard-Service-Typ? (ClusterIP/LoadBalancer)
- Gibt es ein `enabled: false` für Features die du brauchst?
- Wie heißen die Felder für Ressourcen, Image, Replicas?
- Gibt es Persistence-Optionen?

### Schritt 3: Chart-Readme lesen

```bash
helm show readme traefik/traefik
```

Oder auf GitHub — die meisten Charts haben eine README.md.

### Schritt 4: Minimal-values.yaml schreiben

Starte mit wenig — ändere nur was du wirklich anders haben willst:

```yaml
# Minimal — nur das Nötigste
service:
  type: ClusterIP    # Kein LoadBalancer nötig wenn Traefik davor ist
replicas: 1
```

Teste ob es so funktioniert. Dann füge nach und nach weitere Anpassungen hinzu.

### Schritt 5: Rendered YAML prüfen (optional aber sehr lehrreich)

```bash
# Zeigt was Helm aus Chart + deinen Values generieren würde
# OHNE es tatsächlich anzuwenden
helm template meine-app traefik/traefik -f values/traefik/values.yaml

# Das kannst du auch für ArgoCD-Apps simulieren:
helm template meine-app traefik/traefik \
  -f values/traefik/values.yaml \
  --namespace traefik > /tmp/rendered.yaml
```

Das zeigt dir exakt welche Kubernetes-Objekte erstellt werden.
Sehr nützlich um zu verstehen was der Chart tut.

---

## 15. Spickzettel — Das Wichtigste auf einen Blick

### Checkliste: Neue App hinzufügen

```
VORBEREITUNG:
□ Helm Chart oder rohes YAML entschieden?
□ Domain beim DNS-Anbieter eingetragen (zeigt auf LoadBalancer-IP)?

DATEIEN ERSTELLEN:
□ apps/<app-name>.yaml — ArgoCD Application
  □ name: <app-name>
  □ namespace: argocd
  □ finalizers gesetzt
  □ source oder sources korrekt
  □ destination.namespace: <app-name>
  □ CreateNamespace=true
  □ automated.prune: true
  □ automated.selfHeal: true

Falls Helm:
□ values/<app-name>/values.yaml — deine Konfiguration
  □ Nur Werte die vom Default abweichen

Falls rohes YAML:
□ manifests/<app-name>/deployment.yaml
  □ selector.matchLabels == template.metadata.labels
  □ Ressourcen (requests + limits) gesetzt
  □ livenessProbe + readinessProbe wenn möglich
□ manifests/<app-name>/service.yaml
  □ selector passt zu Deployment-Labels
  □ port == targetPort == containerPort (oder bewusst unterschiedlich)

Falls HTTPS:
□ manifests/<app-name>/ingress.yaml
  □ Certificate mit richtiger Domain und secretName
  □ IngressRoute websecure mit Host-Regel
  □ IngressRoute web → redirect-https Middleware
  □ DNS-Eintrag gesetzt BEVOR gepusht wird

GIT:
□ git add <alle neuen Dateien>
□ git commit -m "Add <app-name> application"
□ git push

VERIFIZIEREN (nach 2-5 Minuten):
□ kubectl get applications -n argocd → Synced + Healthy?
□ kubectl get pods -n <app-name> → Running?
□ kubectl get certificate -n <app-name> → Ready?
□ https://<deine-domain> → erreichbar?
```

---

### YAML Grundregeln

```
✓ Immer 2 Leerzeichen einrücken — NIEMALS Tabs
✓ Leerzeichen nach Doppelpunkt: key: value
✓ Strings mit Sonderzeichen in Anführungszeichen: "33.x", "true", "1.0.0"
✓ Listen mit Bindestrich + Leerzeichen: - item
✓ Mehrere Objekte trennen mit: ---
✓ Kommentare mit: # Kommentar
✓ Boolean immer klein: true / false (nicht True, Yes, yes)
```

---

### Häufige kubectl-Befehle

```bash
# STATUS PRÜFEN
kubectl get applications -n argocd          # ArgoCD Apps
kubectl get pods -n <namespace>             # Pods
kubectl get pods -n <namespace> -o wide     # Mit Node und IP
kubectl get services -n <namespace>         # Services
kubectl get endpoints -n <namespace>        # Pod-IPs hinter Services
kubectl get certificate -n <namespace>      # TLS-Certs
kubectl get pvc -n <namespace>              # PersistentVolumeClaims

# DETAILS UND FEHLERSUCHE
kubectl describe pod <name> -n <namespace>          # Events + Details
kubectl describe deployment <name> -n <namespace>   # Deployment Status
kubectl describe certificate <name> -n <namespace>  # Cert-Status

# LOGS LESEN
kubectl logs <pod-name> -n <namespace>              # Aktuelle Logs
kubectl logs <pod-name> -n <namespace> -f           # Live-Logs (follow)
kubectl logs <pod-name> -n <namespace> --previous   # Logs vor letztem Crash

# IN POD EINLOGGEN
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# YAML AUSGABE
kubectl get deployment <name> -n <namespace> -o yaml    # Komplett als YAML
kubectl get secret <name> -n <namespace> -o yaml        # Secret als YAML

# ANWENDEN UND TESTEN
kubectl apply -f datei.yaml                 # Anwenden
kubectl apply -f datei.yaml --dry-run=client  # Nur prüfen, nicht anwenden
kubectl delete -f datei.yaml               # Löschen

# ARGOCD-SPEZIFISCH
argocd app list                            # Alle Apps
argocd app get <name>                      # Details
argocd app sync <name>                     # Manuell synchronisieren
argocd app logs <name>                     # App-Logs
```

---

### ArgoCD Application — Minimales Template

```yaml
# Für rohes YAML aus Git-Ordner:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: DEIN-APP-NAME
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
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

```yaml
# Für Helm Chart mit eigenen Values:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: DEIN-APP-NAME
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: CHART-REPO-URL
      chart: CHART-NAME
      targetRevision: "CHART-VERSION"
      helm:
        releaseName: DEIN-APP-NAME
        valueFiles:
          - $values/values/DEIN-APP-NAME/values.yaml
    - repoURL: git@github.com:Jakobulus123/K8s-Helm.git
      targetRevision: HEAD
      ref: values
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

---

### Certificate + IngressRoute — Minimales Template

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
          port: DEIN-SERVICE-PORT
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
          port: DEIN-SERVICE-PORT
```

---

### Deployment — Minimales Template

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEIN-APP-NAME
  namespace: DEIN-APP-NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: DEIN-APP-NAME
  template:
    metadata:
      labels:
        app: DEIN-APP-NAME
    spec:
      containers:
      - name: DEIN-APP-NAME
        image: DEIN-IMAGE:TAG
        ports:
        - containerPort: DEIN-PORT
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
  name: DEIN-APP-NAME
  namespace: DEIN-APP-NAME
spec:
  selector:
    app: DEIN-APP-NAME
  ports:
  - port: DEIN-PORT
    targetPort: DEIN-PORT
```

---

*Die häufigste Frage von Anfängern: "Woher weiß ich ob ich alles richtig gemacht habe?"*
*Antwort: `kubectl describe` auf das Objekt das Probleme macht. Die Events am Ende der
Ausgabe erklären fast immer was fehlt oder falsch ist. Lies sie genau.*
