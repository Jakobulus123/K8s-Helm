# Sealed Secrets — Installation und Workflow

> Diese Dokumentation beschreibt wie Sealed Secrets installiert wird,
> warum es notwendig ist und wie der tägliche Workflow aussieht.
>
> Zielgruppe: Jemand der bereits ArgoCD + GitOps im Einsatz hat und
> Secrets sicher in Git verwalten möchte.

---

## Inhaltsverzeichnis

1. [Warum Sealed Secrets?](#1-warum-sealed-secrets)
2. [Wie es funktioniert](#2-wie-es-funktioniert)
3. [Installation — Controller im Cluster](#3-installation--controller-im-cluster)
4. [Installation — kubeseal CLI](#4-installation--kubeseal-cli)
5. [Master Key sichern (kritisch!)](#5-master-key-sichern-kritisch)
6. [Workflow — Secret verschlüsseln und deployen](#6-workflow--secret-verschlüsseln-und-deployen)
7. [Secret aktualisieren](#7-secret-aktualisieren)
8. [Wichtige Konzepte und Fallstricke](#8-wichtige-konzepte-und-fallstricke)
9. [Checkliste](#9-checkliste)

---

## 1. Warum Sealed Secrets?

Kubernetes Secrets sind nur Base64-kodiert — das ist **keine Verschlüsselung**.

```bash
echo "bWVpblBhc3N3b3Jk" | base64 -d
# Ausgabe: meinPasswort
```

Wer Zugriff auf dein Git-Repo hat, kann jeden Secret sofort lesen.
Sealed Secrets löst dieses Problem durch echte asymmetrische Verschlüsselung.

---

## 2. Wie es funktioniert

```
Lokal:    Secret YAML → kubeseal (verschlüsselt mit Public Key) → SealedSecret YAML
Git:      SealedSecret YAML committen (sicher, verschlüsselt)
Cluster:  ArgoCD deployt SealedSecret → Controller entschlüsselt → echtes K8s Secret
```

Es gibt zwei Komponenten:
- **Controller** — läuft im Cluster, kennt den Private Key, entschlüsselt SealedSecrets
- **kubeseal CLI** — läuft lokal, verschlüsselt Secrets mit dem Public Key des Controllers

---

## 3. Installation — Controller im Cluster

Der Controller wird wie alle anderen Apps über ArgoCD deployt.

### 3.1 Aktuelle Chart-Version prüfen

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami/sealed-secrets --versions | head -5
```

### 3.2 ArgoCD Application erstellen

Datei: `apps/sealed-secrets.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - repoURL: https://charts.bitnami.com/bitnami
      chart: sealed-secrets
      targetRevision: "2.5.19"
      helm:
        releaseName: sealed-secrets
        parameters:
          - name: fullnameOverride
            value: sealed-secrets-controller
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> **Warum `fullnameOverride: sealed-secrets-controller`?**
> Die kubeseal CLI sucht standardmäßig nach einem Service namens
> `sealed-secrets-controller`. Ohne diesen Parameter generiert Helm einen
> anderen Namen und kubeseal findet den Controller nicht.

> **Warum `kube-system`?**
> Der Controller braucht Zugriff auf Secrets clusterübergreifend.
> `kube-system` ist der empfohlene Namespace.

### 3.3 Committen und deployen

```bash
git add apps/sealed-secrets.yaml
git commit -m "Add sealed-secrets ArgoCD application"
git push
```

ArgoCD (Root App) erkennt die neue Datei automatisch und deployt den Controller.

### 3.4 Deployment prüfen

```bash
# Pod muss Running sein
kubectl get pods -n kube-system | grep sealed

# Service-Name muss exakt "sealed-secrets-controller" sein
kubectl get svc -n kube-system | grep sealed
```

Erwartete Ausgabe:
```
sealed-secrets-controller-xxxx   1/1   Running   0   1m
sealed-secrets-controller         ClusterIP   10.x.x.x   <none>   8080/TCP
```

---

## 4. Installation — kubeseal CLI

Die CLI läuft lokal auf deiner Maschine — nicht im Cluster.

> **Wichtig:** kubeseal Version muss zur Controller APP VERSION passen.
> Controller APP VERSION prüfen: `helm search repo bitnami/sealed-secrets`

```bash
KUBESEAL_VERSION=0.31.0

wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
rm kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal

kubeseal --version
```

### Verbindung zum Controller testen

```bash
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system
```

Ausgabe: Ein langer PEM-Zertifikatsblock. Wenn dieser erscheint, ist alles korrekt verbunden.

---

## 5. Master Key sichern (kritisch!)

Der Controller generiert beim ersten Start ein RSA-Schlüsselpaar.
Der **Private Key** liegt nur im Cluster. Ohne diesen Key können alle
SealedSecrets nach einem Cluster-Neuaufbau nie mehr entschlüsselt werden.

### Key exportieren

```bash
kubectl get secret \
  -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > ~/backups/sealed-secrets-master-key-BACKUP.yaml

chmod 600 ~/backups/sealed-secrets-master-key-BACKUP.yaml
```

### Regeln für den Backup

- Sicher aufbewahren (Passwort-Manager, verschlüsseltes Medium)
- **NIEMALS in Git committen**
- Nach jeder Key Rotation erneut exportieren (alle 30 Tage automatisch)

### .gitignore absichern

```bash
echo "sealed-secrets-master-key-BACKUP*.yaml" >> .gitignore
```

### Key wiederherstellen (nach Cluster-Neuaufbau)

```bash
kubectl apply -f ~/backups/sealed-secrets-master-key-BACKUP.yaml

# Controller neu starten damit er den Key einliest
kubectl rollout restart deployment sealed-secrets-controller -n kube-system
```

---

## 6. Workflow — Secret verschlüsseln und deployen

Das ist der tägliche Workflow wenn du ein neues Secret brauchst.

### Schritt 1: Temporäres Secret YAML erstellen

```bash
kubectl create secret generic mein-secret \
  --namespace=meine-app \
  --from-literal=DB_PASSWORD=meinPasswort \
  --from-literal=DB_USER=admin \
  --dry-run=client \
  -o yaml > /tmp/mein-secret.yaml
```

> `--dry-run=client -o yaml` erstellt das Secret nur lokal — nichts wird in den Cluster geschrieben.

### Schritt 2: Secret verschlüsseln

```bash
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < /tmp/mein-secret.yaml \
  > manifests/meine-app/mein-secret-sealed.yaml

# Temporäre Datei löschen
rm /tmp/mein-secret.yaml
```

### Schritt 3: SealedSecret committen

```bash
git add manifests/meine-app/mein-secret-sealed.yaml
git commit -m "Add sealed secret for meine-app"
git push
```

ArgoCD deployt das SealedSecret → Controller entschlüsselt → echtes K8s Secret entsteht.

### Schritt 4: Prüfen

```bash
# SealedSecret vorhanden
kubectl get sealedsecret -n meine-app

# Echtes Secret wurde erstellt
kubectl get secret mein-secret -n meine-app

# Inhalt prüfen
kubectl get secret mein-secret -n meine-app \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### Schritt 5: Secret in der App nutzen

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mein-secret      # exakt der Name aus dem SealedSecret
        key: DB_PASSWORD
```

---

## 7. Secret aktualisieren

Wenn sich ein Secret-Wert ändert, läuft der Prozess komplett von vorne:

```bash
kubectl create secret generic mein-secret \
  --namespace=meine-app \
  --from-literal=DB_PASSWORD=neuesPasswort \
  --dry-run=client -o yaml | \
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  > manifests/meine-app/mein-secret-sealed.yaml

git add manifests/meine-app/mein-secret-sealed.yaml
git commit -m "Update sealed secret for meine-app"
git push
```

---

## 8. Wichtige Konzepte und Fallstricke

### Namespace-Bindung

SealedSecrets sind standardmäßig an **Namespace + Name** gebunden.

- Ein SealedSecret für Namespace `production` funktioniert **nicht** in `staging`
- Das verhindert, dass jemand ein Secret in einen anderen Namespace kopiert
- Für mehrere Namespaces muss für jeden separat verschlüsselt werden

### Key Rotation

- Der Controller rotiert den Key automatisch alle **30 Tage**
- Alte Keys bleiben erhalten — alte SealedSecrets funktionieren weiter
- Nach jeder Rotation den Backup neu exportieren

### Cluster-Bindung

- SealedSecrets sind an den Cluster-Key gebunden
- Ein SealedSecret von Cluster A funktioniert nicht auf Cluster B
- Für mehrere Cluster muss für jeden Cluster separat verschlüsselt werden

---

## 9. Checkliste

```
[ ] apps/sealed-secrets.yaml erstellt und committed
[ ] ArgoCD Root App gesynct
[ ] Pod Running:  kubectl get pods -n kube-system | grep sealed
[ ] Service-Name korrekt: sealed-secrets-controller
[ ] kubeseal CLI installiert: kubeseal --version
[ ] Verbindung getestet: kubeseal --fetch-cert ...
[ ] Master Key gesichert unter ~/backups/ (nie in Git!)
[ ] .gitignore Eintrag gesetzt
[ ] Erstes Test-Secret erfolgreich verschlüsselt und deployed
```
