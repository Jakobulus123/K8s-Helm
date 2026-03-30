# 6-Wochen Lernplan — Kubernetes, Helm & ArgoCD Internals

> Dieser Plan baut logisch aufeinander auf — jede Woche setzt das Wissen der vorherigen voraus.
> Vorwissen: Cluster manuell aufgebaut, ArgoCD + GitHub-Workflow bekannt, kubectl grundlegend.
> Fokus: Technische Internals — nicht "wie benutze ich es", sondern "warum funktioniert es so".

---

## Übersicht

| Woche | Thema | Warum diese Reihenfolge |
|---|---|---|
| 1 | Kubernetes Internals | Fundament — alles andere baut darauf auf |
| 2 | Netzwerk tief | Braucht K8s-Wissen aus Woche 1 |
| 3 | Helm — Verstehen & Struktur | Braucht K8s-Objekte & YAML aus Woche 1–2 |
| 4 | Helm — Tief & Sicher anwenden | Baut direkt auf Woche 3 auf |
| 5 | ArgoCD Internals | Braucht K8s + Helm + Netzwerk aus Woche 1–4 |
| 6 | Alles zusammen | Synthese — die komplette Prozesskette |

---

## Woche 1 — Kubernetes von innen

**Ziel:** Verstehen was passiert wenn du `kubectl apply` drückst

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | API-Server, etcd, Controller-Manager — was macht welche Komponente | `K8s-lernen-verstehen-verbindungen.md` Kap. 2–3 |
| Di | Reconciliation Loop: wie K8s Desired State vs. Actual State überwacht | `K8s-lernen-verstehen-verbindungen.md` Kap. 4 |
| Mi | Was passiert intern bei `kubectl apply`: API-Server → etcd → Scheduler → kubelet | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 4 |
| Do | Workload-Objekte tief: ReplicaSet, Deployment, wie Rolling Update intern funktioniert | `K8s-lernen-verstehen-verbindungen.md` Kap. 4 |
| Fr | Praxis: `kubectl get events -w` beim Deployment beobachten — jeden Schritt live sehen | — |

---

## Woche 2 — Netzwerk tief

**Ziel:** Jeden Hop eines Requests von Browser bis Pod kennen

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | OSI-Modell: Schichten 1–4 verstehen, wo K8s auf welcher Schicht eingreift | `OSI-Modell.md` Kap. 1–4 |
| Di | kube-proxy, iptables, wie ClusterIP intern funktioniert | `K8s-lernen-verstehen-verbindungen.md` Kap. 5–6 |
| Mi | Traefik intern: wie er IngressRoutes beobachtet, Reverse-Proxy-Mechanik | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 6 |
| Do | cert-manager intern: ACME-Challenge, wie Let's Encrypt TLS ausstellt | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 7 |
| Fr | Praxis: `kubectl exec` in Pod, mit `curl` und `nslookup` Netzwerk selbst erkunden | — |

---

## Woche 3 — Helm verstehen (Theorie & Struktur)

**Ziel:** Helm ist keine Blackbox mehr — verstehen was intern passiert

> Warum erst jetzt: Helm rendert Kubernetes-YAML. Ohne Verständnis von K8s-Objekten (Woche 1)
> und Netzwerk (Woche 2) macht Helm-Output keinen Sinn.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Was Helm ist und warum es existiert: Chart vs. Release vs. Revision | `helm-yaml-lernen.md` Kap. 5 |
| Di | Chart-Struktur: `Chart.yaml`, `values.yaml`, `templates/` — was liegt wo und warum | `helm-yaml-lernen.md` Kap. 6 |
| Mi | `values.yaml` tief: Overrides, Hierarchie, `--set` vs. `-f values.yaml` | `helm-yaml-lernen.md` Kap. 8 |
| Do | Go-Templates: `{{ .Values.x }}`, `{{ .Release.Name }}`, `{{ if }}`, `{{ range }}` | `helm-yaml-lernen.md` Kap. 7 |
| Fr | Praxis: `helm template <chart> -f values.yaml` — Output Zeile für Zeile lesen | — |

---

## Woche 4 — Helm tief & sicher anwenden

**Ziel:** Helm-Änderungen ohne Angst durchführen, Fehler selbst debuggen

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Helm Hooks: `pre-install`, `post-upgrade`, `pre-delete` — Reihenfolge & Auswirkungen | `helm-yaml-lernen.md` Kap. 7 |
| Di | Unbekannte Charts lesen: wie du einen fremden Chart verstehst ohne Doku | `helm-yaml-lernen.md` Kap. 14 |
| Mi | Häufige Helm-Fehler und ihre Ursachen — warum Cluster kaputtgehen können | `helm-yaml-lernen.md` Kap. 13 |
| Do | Helm + ArgoCD: wie ArgoCD intern `helm template` aufruft bevor es applied | `helm-yaml-lernen.md` Kap. 9–10 |
| Fr | Praxis: Eigenen Mini-Chart von Null schreiben, mit `helm template` testen | — |

> **Regel für diese beiden Helm-Wochen:**
> ```
> helm template → Output prüfen → dann erst helm upgrade
> ```
> Nie direkt upgraden ohne vorher den gerenderten Output gelesen zu haben.

---

## Woche 5 — ArgoCD Internals

**Ziel:** Verstehen was ArgoCD unter der Haube macht — nicht wie man es benutzt

> Warum erst jetzt: ArgoCD verwendet intern Helm (Woche 3–4) und kommuniziert über
> K8s-Netzwerk (Woche 1–2). Ohne dieses Wissen bleibt ArgoCD eine Blackbox.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | ArgoCD-Komponenten: Application Controller, Repo Server, API Server, Dex — wer macht was | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 5 |
| Di | Reconciliation Loop in ArgoCD: wie er Git-Zustand mit Cluster-Zustand vergleicht | `K8s-lernen-verstehen-verbindungen.md` Kap. 11 |
| Mi | Custom Resource Definitions (CRDs): was ist eine `Application`-Ressource intern | `helm-yaml-lernen.md` Kap. 9 |
| Do | Warum OutOfSync entsteht, wie Health-Checks funktionieren, was `Degraded` bedeutet | `FEHLER-DOKU.md` alle Fehler |
| Fr | Praxis: `kubectl get application -n argocd -o yaml` — rohen State lesen und verstehen | — |

---

## Woche 6 — Alles zusammen

**Ziel:** Die komplette Prozesskette von Git Push bis laufendem Pod komplett verstehen

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Komplette Prozesskette lesen: Git Push → ArgoCD → Helm render → apply → Pod | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 3 + 10 |
| Di | Was passiert wenn eine Komponente ausfällt: ArgoCD down, Traefik down, cert-manager down | `WIE-ALLES-ZUSAMMENHAENGT.md` Kap. 8–9 |
| Mi | HOW-TO-CREATE von Null lesen: gesamtes Setup nachvollziehen mit jetzt tiefem Verständnis | `HOW-TO-CREATE.md` komplett |
| Do | Offene Fragen klären, Lücken schließen — was ist noch unklar? | alle Dateien |
| Fr | Praxis: Neue App von Null anlegen — YAML schreiben, Helm-Chart verstehen, ArgoCD-Sync beobachten | — |

---

## Spickzettel — Wichtige Befehle pro Woche

```bash
# Woche 1 — K8s Internals beobachten
kubectl get events -w
kubectl describe pod <name>
kubectl get all -A

# Woche 2 — Netzwerk erkunden
kubectl exec -it <pod> -- /bin/sh
curl http://<service>.<namespace>.svc.cluster.local
nslookup <service>

# Woche 3+4 — Helm sicher nutzen
helm template <release> <chart> -f values.yaml
helm diff upgrade <release> <chart> -f values.yaml
helm history <release>

# Woche 5 — ArgoCD Internals
kubectl get application -n argocd -o yaml
kubectl logs -n argocd deployment/argocd-application-controller
kubectl logs -n argocd deployment/argocd-repo-server

# Woche 6 — Alles zusammen
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl rollout status deployment/<name>
```
