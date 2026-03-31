# 7-Wochen Lernplan — Kubernetes, Helm & ArgoCD Internals

> Dieser Plan baut logisch aufeinander auf — jede Woche setzt das Wissen der vorherigen voraus.
> Vorwissen: Cluster manuell aufgebaut, ArgoCD + GitHub-Workflow bekannt, kubectl grundlegend.
> Fokus: Technische Internals — nicht "wie benutze ich es", sondern "warum funktioniert es so".

---

## Übersicht

| Woche | Thema | Tage | Warum diese Zeit |
|---|---|---|---|
| 1 | K8s Control Plane & Reconciliation | 5 | Fundament für alles — zu schnell = alles später unklar |
| 2 | K8s Workloads, Config, Storage, RBAC | 5 | Vervollständigt K8s-Wissen — fehlt im Cluster sonst als Lücke |
| 3 | Netzwerk tief | 5 | OSI allein hat 905 Zeilen — braucht eigene Woche |
| 4 | Helm — Basics & Struktur | 5 | Helm rendert K8s-YAML — erst nach Woche 1+2 sinnvoll |
| 5 | Helm — Advanced & sicher anwenden | 5 | Baut direkt auf Woche 4 auf, größte Datei im Repo |
| 6 | ArgoCD Internals | 5 | Braucht K8s + Helm + Netzwerk als Vorwissen |
| 7 | Synthese — alles zusammen | 5 | Prozesskette komplett, HOW-TO-CREATE, Lücken schließen |

**Gesamt: 7 Wochen — alle Dateien vollständig abgedeckt**

---

## Woche 1 — K8s Control Plane & Reconciliation

**Ziel:** Verstehen was im Hintergrund passiert wenn du `kubectl apply` drückst

> Warum eine ganze Woche: Das ist das Fundament für Helm, ArgoCD und Netzwerk.
> Wer hier zu schnell drüber liest, versteht später nicht warum Dinge kaputt gehen.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | API-Server: was er ist, warum alles über ihn läuft, wie er Anfragen validiert | `lernen/KUBERNETES-KONZEPTE.md` Kap. 2 |
| Di | etcd: wo K8s seinen State speichert — warum etcd = Gehirn des Clusters | `lernen/KUBERNETES-KONZEPTE.md` Kap. 2 |
| Mi | Scheduler + Controller-Manager: wer entscheidet wo Pods landen und wer sie überwacht | `lernen/KUBERNETES-KONZEPTE.md` Kap. 2–3 |
| Do | Reconciliation Loop: Desired State vs. Actual State — das Kernprinzip von K8s | `lernen/KUBERNETES-KONZEPTE.md` Kap. 4 |
| Fr | Praxis: `kubectl get events -w` beim Deployment beobachten — Control Plane live sehen | `lernen/SETUP-DOKU.md` Kap. 2 |

---

## Woche 2 — K8s Workloads, Config, Storage & RBAC

**Ziel:** Alle K8s-Objekte kennen die im Cluster tatsächlich vorkommen

> Warum eigene Woche: Diese Objekte (Secrets, ConfigMaps, PVCs, RBAC) fehlen in
> den meisten Kurzanleitungen — aber genau hier entstehen die meisten Fehler in der Praxis.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Deployment, ReplicaSet, Pod — wie Rolling Update intern abläuft, was passiert bei Absturz | `lernen/KUBERNETES-KONZEPTE.md` Kap. 4 |
| Di | ConfigMaps & Secrets: wie Konfiguration in Pods landet, Unterschied env vs. Volume | `lernen/KUBERNETES-KONZEPTE.md` Kap. 8 |
| Mi | Storage: PersistentVolume, PVC, StorageClass — wie Daten den Pod-Neustart überleben | `lernen/KUBERNETES-KONZEPTE.md` Kap. 9 |
| Do | RBAC: ServiceAccounts, Roles, ClusterRoles — wer darf was im Cluster | `lernen/KUBERNETES-KONZEPTE.md` Kap. 10 |
| Fr | Praxis: `kubectl describe` auf Deployment, ConfigMap, PVC — alles was du gelesen hast live ansehen | `lernen/SETUP-DOKU.md` Kap. 3 + 12 |

---

## Woche 3 — Netzwerk tief

**Ziel:** Jeden Hop eines Requests von Browser bis Pod lückenlos kennen

> Warum eigene Woche: OSI-Modell allein hat 905 Zeilen. kube-proxy + iptables sind
> komplex. Traefik + cert-manager brauchen beide eigenen Fokus.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | OSI-Modell komplett: alle 7 Schichten, wo K8s auf welcher Schicht eingreift | `lernen/OSI-NETZWERK.md` komplett |
| Di | K8s-Netzwerk intern: kube-proxy, iptables-Regeln, wie ClusterIP und DNS funktionieren | `lernen/KUBERNETES-KONZEPTE.md` Kap. 5–6 |
| Mi | Services tief: ClusterIP vs. NodePort vs. LoadBalancer — wann welcher und warum | `lernen/KUBERNETES-KONZEPTE.md` Kap. 6 |
| Do | Traefik intern: wie er IngressRoutes beobachtet, Reverse-Proxy-Mechanik, Middlewares | `lernen/ARCHITEKTUR.md` Kap. 6 |
| Fr | cert-manager intern: ACME-Challenge-Flow, wie Let's Encrypt TLS ausstellt und erneuert | `lernen/ARCHITEKTUR.md` Kap. 7 |

**Praxis am Wochenende (optional):** `kubectl exec` in Pod, mit `curl` und `nslookup` den Netzwerkweg selbst nachverfolgen

---

## Woche 4 — Helm Basics & Struktur

**Ziel:** Helm ist keine Blackbox mehr — verstehen was intern passiert

> Warum erst jetzt: Helm rendert K8s-YAML. Ohne tiefes K8s-Wissen (Woche 1–2)
> und Netzwerk (Woche 3) versteht man den Output nicht.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | YAML tief: Syntax, Typen, Einrückung, Fallstricke — die Sprache hinter allem | `lernen/YAML-UND-HELM.md` Kap. 1–2 |
| Di | Was Helm ist: Chart vs. Release vs. Revision, warum es existiert | `lernen/YAML-UND-HELM.md` Kap. 5 |
| Mi | Chart-Struktur: `Chart.yaml`, `values.yaml`, `templates/` — was liegt wo und warum | `lernen/YAML-UND-HELM.md` Kap. 6 |
| Do | `values.yaml` tief: Overrides, Hierarchie, `--set` vs. `-f values.yaml` | `lernen/YAML-UND-HELM.md` Kap. 8 |
| Fr | Praxis: `helm template <chart> -f values.yaml` — Output Zeile für Zeile lesen und mit K8s-Wissen zuordnen | — |

---

## Woche 5 — Helm Advanced & sicher anwenden

**Ziel:** Helm-Änderungen ohne Angst, Fehler selbst debuggen, fremde Charts lesen

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Go-Templates tief: `{{ .Values.x }}`, `{{ .Release.Name }}`, `{{ if }}`, `{{ range }}`, `{{ include }}` | `lernen/YAML-UND-HELM.md` Kap. 7 |
| Di | Helm Hooks: `pre-install`, `post-upgrade`, `pre-delete` — Reihenfolge und was schiefgehen kann | `lernen/YAML-UND-HELM.md` Kap. 7 |
| Mi | Häufige Helm-Fehler: warum Cluster kaputtgehen, wie man es vorher erkennt | `lernen/YAML-UND-HELM.md` Kap. 13 |
| Do | Unbekannte Charts lesen: wie du einen fremden Chart ohne Doku verstehst | `lernen/YAML-UND-HELM.md` Kap. 14 |
| Fr | Helm + ArgoCD: wie ArgoCD intern `helm template` aufruft — dann eigenen Mini-Chart schreiben und testen | `lernen/YAML-UND-HELM.md` Kap. 9–10 |

> **Die wichtigste Helm-Regel:**
> ```
> helm template → Output prüfen → dann erst helm upgrade
> ```
> Nie direkt upgraden ohne vorher den gerenderten Output gelesen zu haben.

---

## Woche 6 — ArgoCD Internals

**Ziel:** Verstehen was ArgoCD unter der Haube macht — nicht wie man es benutzt

> Warum erst jetzt: ArgoCD nutzt intern Helm (Woche 4–5), kommuniziert über
> K8s-APIs (Woche 1) und Netzwerk (Woche 3). Jetzt macht alles Sinn.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | ArgoCD-Komponenten: Application Controller, Repo Server, API Server, Dex — wer macht was intern | `lernen/ARCHITEKTUR.md` Kap. 5 |
| Di | ArgoCD Reconciliation Loop: wie er Git-Zustand mit Cluster-Zustand vergleicht — Schritt für Schritt | `lernen/KUBERNETES-KONZEPTE.md` Kap. 11 |
| Mi | CRDs: was eine `Application`-Ressource intern ist, wie K8s sie verarbeitet | `lernen/YAML-UND-HELM.md` Kap. 9 |
| Do | OutOfSync, Degraded, Health-Checks: warum sie entstehen, wie ArgoCD sie berechnet | `troubleshooting/FEHLER-DOKU.md` alle Fehler |
| Fr | Praxis: `kubectl get application -n argocd -o yaml` + Controller-Logs lesen — rohen ArgoCD-State verstehen | — |

---

## Woche 7 — Synthese: Alles zusammen

**Ziel:** Die komplette Prozesskette von Git Push bis laufendem Pod aus dem Kopf erklären können

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Komplette Prozesskette: Git Push → ArgoCD → Helm render → kubectl apply → Pod läuft | `lernen/ARCHITEKTUR.md` Kap. 3 + 10 |
| Di | Ausfallszenarien: was passiert wenn ArgoCD, Traefik oder cert-manager ausfällt | `lernen/ARCHITEKTUR.md` Kap. 8–9 |
| Mi | HOW-TO-CREATE komplett lesen: Setup von Null — mit jetzt tiefem Wissen alles nachvollziehen | `setup/CLUSTER-AUFBAUEN.md` komplett |
| Do | SETUP-DOKU Kap. 9–12: Setup-Schritte + Debugging-Referenz + kubectl-Befehle | `lernen/SETUP-DOKU.md` Kap. 9–12 |
| Fr | Praxis: Neue App von Null — YAML schreiben, Chart verstehen, ArgoCD-Sync beobachten, Fehler selbst lösen | — |

---

## Spickzettel — Wichtige Befehle pro Woche

```bash
# Woche 1+2 — K8s Internals beobachten
kubectl get events -w
kubectl describe pod <name>
kubectl get all -A
kubectl get cm,secret,pvc -n <namespace>

# Woche 3 — Netzwerk erkunden
kubectl exec -it <pod> -- /bin/sh
curl http://<service>.<namespace>.svc.cluster.local
nslookup <service>.<namespace>.svc.cluster.local

# Woche 4+5 — Helm sicher nutzen
helm template <release> <chart> -f values.yaml
helm diff upgrade <release> <chart> -f values.yaml
helm history <release>
helm get manifest <release>

# Woche 6 — ArgoCD Internals
kubectl get application -n argocd -o yaml
kubectl logs -n argocd deployment/argocd-application-controller
kubectl logs -n argocd deployment/argocd-repo-server

# Woche 7 — Alles zusammen
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
```

---

## Was in welcher Datei steckt

| Datei | Zeilen | Hauptsächlich genutzt in |
|---|---|---|
| `lernen/KUBERNETES-KONZEPTE.md` | 1.457 | Woche 1, 2, 3, 6 |
| `lernen/ARCHITEKTUR.md` | 1.126 | Woche 1, 3, 6, 7 |
| `lernen/OSI-NETZWERK.md` | 905 | Woche 3 |
| `lernen/YAML-UND-HELM.md` | 2.898 | Woche 4, 5, 6 |
| `troubleshooting/FEHLER-DOKU.md` | 455 | Woche 6 |
| `setup/CLUSTER-AUFBAUEN.md` | 1.318 | Woche 7 |
| `lernen/SETUP-DOKU.md` | 1.570 | Woche 1, 2, 7 |\n| `setup/SEALED-SECRETS.md` | 359 | Zusatz: Secrets sicher in Git verwalten |
