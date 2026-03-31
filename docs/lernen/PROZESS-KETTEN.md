# Prozess-Ketten — Wie alles zusammenhängt

> Jede Kette gehört zu einer Woche im Lernplan.
> Ziel: nicht auswendig lernen — sondern die Kette verstehen und erklären können.
> Wenn du eine Kette aus dem Kopf erklären kannst, hast du die Woche verstanden.

---

## Woche 1 — Control Plane & Reconciliation

### Kette 1: Was passiert wenn du `kubectl apply` drückst

```
Du: kubectl apply -f deployment.yaml
  → API-Server empfängt Anfrage
  → validiert das YAML (Schema, Rechte)
  → schreibt Objekt in etcd
  → etcd bestätigt Schreiben
  → API-Server sendet Watch-Event an alle Controller
  → Deployment Controller sieht neues Deployment
  → erstellt ReplicaSet
  → ReplicaSet Controller sieht neues ReplicaSet
  → erstellt Pod-Objekte (noch kein Node)
  → Scheduler sieht ungebundene Pods
  → wählt passenden Node (Ressourcen, Taints, Affinity)
  → schreibt Node-Zuweisung in etcd
  → kubelet auf dem Node sieht seinen Pod
  → zieht Container-Image
  → startet Container
  → meldet Status zurück an API-Server
  → Pod Status: Running
```

### Kette 2: Reconciliation Loop — das Kernprinzip

```
Controller liest Desired State aus etcd
  → vergleicht mit Actual State (was läuft wirklich)
  → Differenz gefunden?
      → Ja: Aktion ausführen (Pod erstellen / löschen / updaten)
      → Nein: nichts tun
  → schläft kurz
  → wiederholt ewig
```

Beispiel: Du löschst einen Pod manuell
```
Pod gelöscht
  → ReplicaSet Controller bemerkt: 2 Pods erwartet, 1 vorhanden
  → erstellt neuen Pod
  → Desired State wieder erreicht
```

### Kette 3: etcd — warum es das Gehirn ist

```
Jede Änderung im Cluster:
  → geht durch API-Server
  → landet in etcd
  → etcd ist die einzige Source of Truth

etcd fällt aus:
  → API-Server kann nichts lesen/schreiben
  → Cluster läuft noch (laufende Pods bleiben)
  → aber keine neuen Deployments, keine Änderungen möglich
```

---

## Woche 2 — Workloads, Config, Storage & RBAC

### Kette 4: Rolling Update — wie ein Deployment sicher updatet

```
Du änderst das Image in values.yaml → ArgoCD sync
  → Deployment Controller sieht neue Pod-Template
  → erstellt neues ReplicaSet (neue Version)
  → startet 1 neuen Pod (neue Version)
  → wartet bis Pod Ready (readinessProbe bestanden)
  → skaliert altes ReplicaSet um 1 runter
  → wiederholt bis alle Pods neu sind
  → altes ReplicaSet bleibt (auf 0) → für Rollback
```

### Kette 5: Service → Label-Selector → Endpoint → Traffic

```
Du erstellst einen Service mit selector: app=my-app
  → Endpoints Controller (im kube-controller-manager) beobachtet API-Server
  → sucht Pods im gleichen Namespace mit Label app=my-app
  → prüft: sind diese Pods Ready? (readinessProbe)
  → trägt Pod-IPs in Endpoints-Objekt ein

Endpoints-Objekt existiert:
  → kube-proxy sieht neue Endpoints
  → schreibt iptables-Regeln auf jedem Node
  → Traffic an ClusterIP wird zu Pod-IP umgeleitet

Pod stirbt:
  → Endpoints Controller entfernt IP aus Endpoints
  → kube-proxy aktualisiert iptables
  → kein Traffic mehr an tote IP
```

**Wo es schiefgeht:**
```
Service selector: app=my-app
Pod Label:        app=myapp    ← ein Zeichen falsch
  → Endpoints Controller findet keine Pods
  → Endpoints bleibt leer
  → kube-proxy schreibt keine Regel
  → Traffic schlägt fehl — ohne Fehlermeldung
```

### Kette 6: ConfigMap/Secret → Pod

```
ConfigMap wird erstellt
  → in etcd gespeichert

Pod referenziert ConfigMap:
  Option A: als Umgebungsvariable
    → kubelet liest ConfigMap beim Pod-Start
    → setzt env-Variable im Container
    → Änderung an ConfigMap → Pod muss neu starten

  Option B: als Volume
    → kubelet mounted ConfigMap als Dateien
    → Änderungen werden automatisch aktualisiert (nach ~1 min)
    → kein Neustart nötig
```

### Kette 7: PVC → Daten überleben Pod-Neustart

```
Du erstellst einen PVC (PersistentVolumeClaim)
  → StorageClass wird aufgerufen
  → StorageClass erstellt PersistentVolume (echten Speicher)
  → PVC wird an PV gebunden

Pod läuft mit PVC:
  → kubelet mounted PV in den Container
  → Daten werden auf PV geschrieben

Pod stirbt / wird neu gestartet:
  → neuer Pod startet auf gleichem oder anderem Node
  → kubelet mounted denselben PV wieder ein
  → Daten sind noch da
```

### Kette 8: RBAC — wer darf was

```
Pod will API-Server ansprechen (z.B. ArgoCD liest Deployments)
  → Pod hat einen ServiceAccount (default wenn nicht angegeben)
  → ServiceAccount hat Token (automatisch gemounted)
  → API-Server prüft: welche Role ist an diesen ServiceAccount gebunden?
  → RoleBinding verbindet ServiceAccount → Role
  → Role definiert: welche Verben (get, list, watch) auf welche Ressourcen
  → erlaubt oder verweigert
```

---

## Woche 3 — Netzwerk

### Kette 9: kube-proxy — nur ein Regelschreiber

```
kube-proxy läuft auf jedem Node als DaemonSet
  → beobachtet API-Server: neue Endpoints?
  → Ja: schreibt iptables-Regel auf diesem Node
  → Nein: nichts tun

Traffic fließt:
  Pod A → ClusterIP:80
    → iptables fängt Paket ab (im Kernel)
    → DNAT: ClusterIP → Pod-IP (z.B. 10.244.1.5:8080)
    → Paket geht direkt zu Pod B
    → kube-proxy sieht den Traffic nie
```

### Kette 10: Kompletter Netzwerkweg — Browser bis Pod

```
Browser: https://meine-app.example.com
  → DNS-Auflösung → öffentliche IP des Clusters
  → TCP-Verbindung auf Port 443
  → Traefik empfängt Request (als LoadBalancer/NodePort)
  → TLS-Terminierung (cert-manager hat Zertifikat besorgt)
  → Traefik liest IngressRoute
  → matched Host-Header
  → leitet weiter an Service: my-app-service:80
  → Service ClusterIP
  → iptables (kube-proxy hat Regel geschrieben)
  → DNAT zu Pod-IP
  → Pod antwortet
  → zurück durch Traefik
  → Browser zeigt Seite
```

### Kette 11: cert-manager — wie TLS-Zertifikate entstehen

```
Du erstellst ein Certificate-Objekt
  → cert-manager sieht es
  → erstellt ACME-Challenge bei Let's Encrypt
  → Let's Encrypt gibt Challenge-Token zurück
  → cert-manager erstellt temporären Pod/Ingress der Token ausliefert
  → Let's Encrypt ruft http://deine-domain/.well-known/acme-challenge/<token> ab
  → Validierung erfolgreich
  → Let's Encrypt stellt Zertifikat aus
  → cert-manager speichert Zertifikat als Secret
  → Traefik liest Secret → TLS aktiv
```

---

## Woche 4 — Helm Basics

### Kette 12: Was Helm intern macht bei `helm install`

```
helm install my-release ./my-chart -f values.yaml
  → Helm liest Chart.yaml (Metadaten)
  → liest values.yaml (deine Overrides)
  → merged mit Default-values.yaml aus Chart
  → rendert alle templates/ mit Go-Template-Engine
  → {{ .Values.x }} wird durch echte Werte ersetzt
  → Ergebnis: fertiges K8s-YAML
  → helm schickt YAML an API-Server (wie kubectl apply)
  → speichert Release-Info als Secret in K8s (für Rollback)
```

### Kette 13: values.yaml Override-Hierarchie

```
Werte werden in dieser Reihenfolge überschrieben (letzter gewinnt):

1. Chart default values.yaml
2. -f my-values.yaml (deine Datei)
3. --set key=value (direkt in CLI)

Beispiel:
  Chart default:    replicaCount: 1
  dein values.yaml: replicaCount: 3
  --set:            replicaCount: 5
  → Ergebnis: 5
```

---

## Woche 5 — Helm Advanced

### Kette 14: Sicherer Helm-Upgrade Ablauf

```
NIEMALS direkt:
  helm upgrade my-release ./chart -f values.yaml  ← gefährlich

IMMER erst:
  1. helm template my-release ./chart -f values.yaml
     → rendered YAML anschauen — was wird wirklich deployed?

  2. helm diff upgrade my-release ./chart -f values.yaml
     → nur die Änderungen sehen

  3. Erst wenn Output verstanden:
     helm upgrade my-release ./chart -f values.yaml
```

### Kette 15: Helm Hook Reihenfolge

```
helm install:
  pre-install Hook läuft (z.B. Datenbank-Migration)
    → wartet bis Hook-Pod erfolgreich beendet
  → dann: normale Templates werden applied
  → Pod startet
  post-install Hook läuft (z.B. Smoke-Test)

helm upgrade:
  pre-upgrade Hook → Änderungen applied → post-upgrade Hook

Fehler im pre-install Hook:
  → Installation stoppt
  → keine K8s-Objekte werden erstellt
```

### Kette 16: ArgoCD ruft Helm intern auf

```
ArgoCD sync wird getriggert
  → Repo Server clont Git-Repo
  → erkennt: das ist ein Helm-Chart (Chart.yaml vorhanden)
  → ruft intern auf: helm template <release> <chart> -f values.yaml
  → bekommt fertiges K8s-YAML
  → vergleicht mit aktuellem Cluster-State
  → Diff? → apply
  → kein Diff? → Already Synced
```

---

## Woche 6 — ArgoCD Internals

### Kette 17: Git Push → Pod läuft (komplette ArgoCD-Kette)

```
git push → GitHub
  → ArgoCD Repo Server pollt Git alle 3 min (oder Webhook)
  → erkennt neuen Commit
  → Application Controller vergleicht:
      Git-State (was im Repo steht)
      vs. Cluster-State (was wirklich läuft)
  → Unterschied → Status: OutOfSync
  → bei syncPolicy: automated → sync startet automatisch
  → Repo Server rendert Helm-Templates
  → Application Controller schickt YAML an API-Server
  → API-Server → etcd → Controller → Scheduler → kubelet → Pod
  → Status: Synced + Healthy
```

### Kette 18: Wie ArgoCD OutOfSync berechnet

```
ArgoCD holt Live-State:
  kubectl get <alle Ressourcen der App> -o yaml

ArgoCD holt Desired State:
  helm template ... (aus Git)

Vergleich:
  → Felder die ArgoCD managed werden verglichen
  → Unterschied gefunden → OutOfSync
  → Kein Unterschied → Synced

Typische OutOfSync-Fallen:
  → jemand hat manuell kubectl apply gemacht
  → Annotation/Label wurde von K8s automatisch hinzugefügt
  → ignoredDifferences in Application-CRD kann das unterdrücken
```

---

## Woche 7 — Synthese

### Kette 19: Alles zusammen — von git push bis Browser-Response

```
Du: git push

GitHub:
  → Webhook an ArgoCD (oder Polling alle 3 min)

ArgoCD Repo Server:
  → clont Repo
  → helm template → fertiges YAML

ArgoCD Application Controller:
  → vergleicht Git vs. Cluster
  → OutOfSync → sync
  → YAML an API-Server

API-Server:
  → validiert
  → schreibt in etcd

Deployment Controller:
  → sieht Änderung
  → Rolling Update: neues ReplicaSet, neue Pods

Scheduler:
  → weist Pods einem Node zu

kubelet:
  → zieht Image, startet Container

Endpoints Controller:
  → Pod wird Ready
  → trägt IP in Endpoints ein

kube-proxy:
  → schreibt iptables-Regel

Traefik:
  → IngressRoute zeigt auf Service
  → Service → iptables → Pod

Browser:
  → Request → Traefik → Pod → Response
```

### Kette 20: Ausfallszenarien — was passiert wenn X wegfällt

```
ArgoCD fällt aus:
  → laufende Pods bleiben → Nutzer merken nichts
  → kein neues Deployment möglich
  → kein Sync bei Git-Änderungen

Traefik fällt aus:
  → alle externen Requests schlagen fehl
  → interne Pod-zu-Pod Kommunikation läuft weiter
  → cluster-intern alles normal

cert-manager fällt aus:
  → laufende Zertifikate bleiben gültig
  → Erneuerung schlägt fehl wenn Zertifikat abläuft
  → nach 90 Tagen: TLS-Fehler

etcd fällt aus:
  → Cluster läuft noch (laufende Pods bleiben)
  → API-Server kann nichts lesen/schreiben
  → keine neuen Deployments, keine kubectl-Befehle
  → nach Restart: State wird aus etcd wiederhergestellt

kube-proxy fällt aus (ein Node):
  → neue Endpoints werden nicht in iptables eingetragen
  → bestehende Regeln bleiben → bestehender Traffic läuft
  → neue Pods auf diesem Node bekommen keinen Traffic
```

---

## Schnell-Referenz: Welche Kette in welcher Woche

| Kette | Thema | Woche |
|---|---|---|
| 1 | kubectl apply → Pod läuft | 1 |
| 2 | Reconciliation Loop | 1 |
| 3 | etcd als Single Source of Truth | 1 |
| 4 | Rolling Update | 2 |
| 5 | Service → Endpoints → Traffic | 2 |
| 6 | ConfigMap/Secret → Pod | 2 |
| 7 | PVC → Daten überleben | 2 |
| 8 | RBAC → wer darf was | 2 |
| 9 | kube-proxy als Regelschreiber | 3 |
| 10 | Browser → Pod kompletter Weg | 3 |
| 11 | cert-manager TLS-Zertifikat | 3 |
| 12 | helm install intern | 4 |
| 13 | values.yaml Hierarchie | 4 |
| 14 | Sicherer Helm-Upgrade | 5 |
| 15 | Helm Hook Reihenfolge | 5 |
| 16 | ArgoCD ruft Helm auf | 5 |
| 17 | Git Push → Pod (ArgoCD komplett) | 6 |
| 18 | OutOfSync Berechnung | 6 |
| 19 | Alles zusammen | 7 |
| 20 | Ausfallszenarien | 7 |
