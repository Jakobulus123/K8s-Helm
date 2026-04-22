# Lernen — Kubernetes, GitOps, Deployment

Dokumentationssammlung für den Weg vom Software-Entwickler hin zum Verständnis
der Plattform, auf der der Code läuft. Strukturiert nach pädagogischer
Progression.

## Einstieg

1. **[LERNPLAN.md](LERNPLAN.md)** — 7-Wochen-Roadmap. Fängst du hier an, führt
   dich der Plan durch alle Themen unten in sinnvoller Reihenfolge.

## Ordner nach Thema

### [`01-grundlagen/`](01-grundlagen/) — Netzwerk & Theorie

Die Basis, auf der Kubernetes aufsetzt. Ohne diese Grundlagen sind viele
K8s-Details magisch.

- [`OSI-NETZWERK.md`](01-grundlagen/OSI-NETZWERK.md) — OSI-7-Schichten,
  wo Kubernetes und Traefik sich einklinken.

### [`02-kubernetes/`](02-kubernetes/) — Konzepte & Architektur

Was Kubernetes ist und wie die Objekte zusammenspielen.

- [`KUBERNETES-KONZEPTE.md`](02-kubernetes/KUBERNETES-KONZEPTE.md) —
  Objekt-Referenz (Pod, Deployment, Service, Ingress, …).
- [`ARCHITEKTUR.md`](02-kubernetes/ARCHITEKTUR.md) — Gesamtbild: Control
  Plane, ArgoCD, Traefik, cert-manager und deren Zusammenspiel.

### [`03-konfiguration/`](03-konfiguration/) — YAML & Helm

Das Handwerkszeug, mit dem du Manifeste und Charts schreibst.

- [`YAML-UND-HELM.md`](03-konfiguration/YAML-UND-HELM.md) — YAML-Grundlagen,
  Helm-Charts, Templating, Values, Hooks, typische Fehler.

### [`04-deployment/`](04-deployment/) — Vom Code zur laufenden App

Der komplette Flow: Commit → Image → Registry → Chart → ArgoCD → Pod.

- [`IMAGE-UND-DEPLOYMENT.md`](04-deployment/IMAGE-UND-DEPLOYMENT.md) —
  kompakter Überblick des Deployment-Flows.
- [`LEITFADEN-IMAGE-TAG-DEPLOYMENT.md`](04-deployment/LEITFADEN-IMAGE-TAG-DEPLOYMENT.md)
  — pädagogischer Deep-Dive: Tag vs. Digest, GitOps-Konsequenzen,
  Zwei-Repo-Realität, Developer → DevOps Brücke, Debug-Checkliste.
- [`PROZESS-KETTEN.md`](04-deployment/PROZESS-KETTEN.md) — 20 End-to-End
  Abläufe im Cluster, synchron zum Lernplan.

### [`05-setup/`](05-setup/) — Konkrete Installationsschritte

Wie dieser Cluster gebaut ist. Referenz, nicht Lernstoff.

- [`SETUP-DOKU.md`](05-setup/SETUP-DOKU.md) — Kubernetes + ArgoCD + Traefik
  + cert-manager von Null an installieren.

### [`06-debug/`](06-debug/) — Debug-Fälle aus der Praxis

Konkrete Bugs, warum sie passiert sind, wie sie gefixt wurden.

- [`DEBUG-IMAGE-TAG-PROBLEM.md`](06-debug/DEBUG-IMAGE-TAG-PROBLEM.md) —
  Pods laufen auf altem Image, obwohl `:latest` in der Registry neuer ist.
  Warum GitOps dabei nichts merkt und was der richtige Fix ist.

---

## Vorschlag für die Lese-Reihenfolge

Wenn du ganz neu anfängst und diese Doku strukturiert durchgehen willst:

1. **LERNPLAN.md** überfliegen — gibt dir das Big Picture.
2. **01-grundlagen/** — OSI einmal durcharbeiten (Grundlage für alles Netz).
3. **02-kubernetes/KUBERNETES-KONZEPTE.md** — Objekte lernen.
4. **02-kubernetes/ARCHITEKTUR.md** — Zusammenspiel verstehen.
5. **03-konfiguration/YAML-UND-HELM.md** — damit du Manifeste lesen kannst.
6. **04-deployment/LEITFADEN-IMAGE-TAG-DEPLOYMENT.md** — der Gesamtflow als
   Narrative.
7. Bei konkreten Problemen: **06-debug/**.
8. Wenn du den Cluster selbst nachbauen willst: **05-setup/SETUP-DOKU.md**.

## Bei Änderungen

Neue Debug-Fälle gehören in `06-debug/`. Neue Konzept-Deep-Dives in das
passende thematische Verzeichnis. Dieses README sollte mitgepflegt werden,
damit es als Index funktioniert.
