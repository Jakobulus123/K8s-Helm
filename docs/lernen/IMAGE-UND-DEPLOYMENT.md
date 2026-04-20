# Image & Deployment verstehen

## Was ist ein Image?

Ein Image ist ein eingefrorener Snapshot deiner App — Code, Dependencies, Konfiguration, alles drin.
Aus einem Image entstehen zur Laufzeit Pods (Container).

```
Image    = Bauplan (unveränderlich)
Container = das was tatsächlich läuft (aus dem Bauplan erstellt)
```

---

## Woher kommt das Image?

GitHub Actions baut automatisch ein Image wenn du Code pushst:

```
1. Du pushst Code → GitHub Repo
2. GitHub Actions führt docker build aus
3. Image wird nach GHCR gepusht

Ergebnis: ghcr.io/jakobulus123/miniapp:latest
```

GHCR (GitHub Container Registry) ist der Ort wo dein Image gespeichert liegt — vergleichbar mit einem App Store für Container Images.

---

## Woher nimmt die values.yaml ihre Daten?

Sie nimmt nichts — **sie ist selbst die Datenquelle.**

Du schreibst in `charts/miniapp/values.yaml`:
```yaml
image:
  repository: ghcr.io/jakobulus123/miniapp
  tag: "latest"
```

Das Helm Chart Template (`charts/miniapp/templates/deployment.yaml`) liest das:
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
# wird zu: ghcr.io/jakobulus123/miniapp:latest
```

---

## Der komplette Deployment-Flow

```
Du pushst Code
      │
      ▼
GitHub Actions baut Image + pusht nach GHCR
      │
      ▼
ghcr.io/jakobulus123/miniapp:v2   ← liegt hier
      │
Du änderst in values.yaml:        tag: "v2"
      │
      ▼
Du pushst values.yaml nach GitHub
      │
      ▼
ArgoCD bemerkt neuen Commit (alle ~3 Min)
      │
      ▼
ArgoCD liest apps/miniapp.yaml
  → schau in charts/miniapp/
  → lese values.yaml
  → hol Image :v2
      │
      ▼
Kubernetes stoppt alte Pods, startet neue mit :v2
      │
      ▼
🚀 Neue Version läuft
```

---

## Neue Version deployen — Schritt für Schritt

**Schritt 1 — Code pushen**
```bash
git push  # in deinem App-Repo
```
GitHub Actions baut automatisch ein neues Image und pusht es nach GHCR.

**Schritt 2 — Tag in values.yaml anpassen**

In `charts/miniapp/values.yaml`:
```yaml
image:
  tag: "v2"   # vorher: "latest" oder "v1"
```

**Schritt 3 — Ins K8s-Helm Repo pushen**
```bash
cd ~/K8s
git add charts/miniapp/values.yaml
git commit -m "bump miniapp to v2"
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519" git push
```

**Schritt 4 — ArgoCD beobachten**

ArgoCD erkennt den neuen Commit innerhalb von ~3 Minuten und deployed automatisch.
Status prüfen:
```bash
kubectl get pods -n miniapp
kubectl get application miniapp -n argocd
```

---

## Das Problem mit `latest`

`latest` ist ein bewegliches Ziel — heute ist `latest` = v1, morgen = v2.

**Problem:** ArgoCD sieht keine Änderung in der values.yaml wenn sich nur das Image ändert aber der Tag gleich bleibt. Kein neuer Commit → kein neues Deployment.

**Lösung:** Konkrete Tags nutzen:
```yaml
tag: "v2"          # Versionsnummer
tag: "abc1234"     # Git Commit Hash (eindeutig)
```

So ist immer klar welche Version läuft und ein Rollback ist einfach:
```yaml
tag: "v1"   # einfach zurückändern und pushen
```
