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
| Mo | API-Server: was er ist, warum alles über ihn läuft, wie er Anfragen validiert | [KUBERNETES-KONZEPTE.md — Kap. 2](02-kubernetes/KUBERNETES-KONZEPTE.md#2-die-control-plane--das-gehirn) |
| Di | etcd: wo K8s seinen State speichert — warum etcd = Gehirn des Clusters | [KUBERNETES-KONZEPTE.md — Kap. 2](02-kubernetes/KUBERNETES-KONZEPTE.md#2-die-control-plane--das-gehirn) |
| Mi | Scheduler + Controller-Manager: wer entscheidet wo Pods landen und wer sie überwacht | [KUBERNETES-KONZEPTE.md — Kap. 2–3](02-kubernetes/KUBERNETES-KONZEPTE.md#2-die-control-plane--das-gehirn) |
| Do | Reconciliation Loop: Desired State vs. Actual State — das Kernprinzip von K8s | [KUBERNETES-KONZEPTE.md — Kap. 4](02-kubernetes/KUBERNETES-KONZEPTE.md#4-workload-objekte--was-läuft-im-cluster) |
| Fr | Praxis: `kubectl get events -w` beim Deployment beobachten — Control Plane live sehen | [SETUP-DOKU.md — Kap. 2](05-setup/SETUP-DOKU.md#2-kubernetes-architektur-im-detail) |

---

## Woche 2 — K8s Workloads, Config, Storage & RBAC

**Ziel:** Alle K8s-Objekte kennen die im Cluster tatsächlich vorkommen

> Warum eigene Woche: Diese Objekte (Secrets, ConfigMaps, PVCs, RBAC) fehlen in
> den meisten Kurzanleitungen — aber genau hier entstehen die meisten Fehler in der Praxis.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Deployment, ReplicaSet, Pod — wie Rolling Update intern abläuft, was passiert bei Absturz | [KUBERNETES-KONZEPTE.md — Kap. 4](02-kubernetes/KUBERNETES-KONZEPTE.md#4-workload-objekte--was-läuft-im-cluster) |
| Di | ConfigMaps & Secrets: wie Konfiguration in Pods landet, Unterschied env vs. Volume | [KUBERNETES-KONZEPTE.md — Kap. 8](02-kubernetes/KUBERNETES-KONZEPTE.md#8-konfiguration--secrets) |
| Mi | Storage: PersistentVolume, PVC, StorageClass — wie Daten den Pod-Neustart überleben | [KUBERNETES-KONZEPTE.md — Kap. 9](02-kubernetes/KUBERNETES-KONZEPTE.md#9-storage--persistenter-speicher) |
| Do | RBAC: ServiceAccounts, Roles, ClusterRoles — wer darf was im Cluster | [KUBERNETES-KONZEPTE.md — Kap. 10](02-kubernetes/KUBERNETES-KONZEPTE.md#10-rbac--wer-darf-was) |
| Fr | Praxis: `kubectl describe` auf Deployment, ConfigMap, PVC — alles was du gelesen hast live ansehen | [SETUP-DOKU.md — Kap. 3](05-setup/SETUP-DOKU.md#3-wichtige-kubernetes-konzepte-und-ressourcen) + [Kap. 12](05-setup/SETUP-DOKU.md#12-wichtige-kubectl-befehle-referenz) |

---

## Woche 3 — Netzwerk tief

**Ziel:** Jeden Hop eines Requests von Browser bis Pod lückenlos kennen

> Warum eigene Woche: OSI-Modell allein hat 905 Zeilen. kube-proxy + iptables sind
> komplex. Traefik + cert-manager brauchen beide eigenen Fokus.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | OSI-Modell komplett: alle 7 Schichten, wo K8s auf welcher Schicht eingreift | [OSI-NETZWERK.md — komplett](01-grundlagen/OSI-NETZWERK.md) |
| Di | K8s-Netzwerk intern: kube-proxy, iptables-Regeln, wie ClusterIP und DNS funktionieren | [KUBERNETES-KONZEPTE.md — Kap. 5–6](02-kubernetes/KUBERNETES-KONZEPTE.md#5-netzwerk-in-kubernetes--vollständig-erklärt) |
| Mi | Services tief: ClusterIP vs. NodePort vs. LoadBalancer — wann welcher und warum | [KUBERNETES-KONZEPTE.md — Kap. 6](02-kubernetes/KUBERNETES-KONZEPTE.md#6-services--wie-traffic-zu-pods-kommt) |
| Do | Traefik intern: wie er IngressRoutes beobachtet, Reverse-Proxy-Mechanik, Middlewares | [ARCHITEKTUR.md — Kap. 6](02-kubernetes/ARCHITEKTUR.md#6-wie-traefik-traffic-verteilt) |
| Fr | cert-manager intern: ACME-Challenge-Flow, wie Let's Encrypt TLS ausstellt und erneuert | [ARCHITEKTUR.md — Kap. 7](02-kubernetes/ARCHITEKTUR.md#7-wie-cert-manager-tls-zertifikate-holt) |

**Praxis am Wochenende (optional):** `kubectl exec` in Pod, mit `curl` und `nslookup` den Netzwerkweg selbst nachverfolgen

---

## Woche 4 — Helm Basics & Struktur

**Ziel:** Helm ist keine Blackbox mehr — verstehen was intern passiert

> Warum erst jetzt: Helm rendert K8s-YAML. Ohne tiefes K8s-Wissen (Woche 1–2)
> und Netzwerk (Woche 3) versteht man den Output nicht.

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | YAML tief: Syntax, Typen, Einrückung, Fallstricke — die Sprache hinter allem | [YAML-UND-HELM.md — Kap. 1–2](03-konfiguration/YAML-UND-HELM.md#1-yaml--die-sprache-von-kubernetes) |
| Di | Was Helm ist: Chart vs. Release vs. Revision, warum es existiert | [YAML-UND-HELM.md — Kap. 5](03-konfiguration/YAML-UND-HELM.md#5-was-ist-helm-und-warum-braucht-man-es) |
| Mi | Chart-Struktur: `Chart.yaml`, `values.yaml`, `templates/` — was liegt wo und warum | [YAML-UND-HELM.md — Kap. 6](03-konfiguration/YAML-UND-HELM.md#6-helm-chart-struktur--was-ist-wo-und-warum) |
| Do | `values.yaml` tief: Overrides, Hierarchie, `--set` vs. `-f values.yaml` | [YAML-UND-HELM.md — Kap. 8](03-konfiguration/YAML-UND-HELM.md#8-valuesyaml--das-herzstück-der-konfiguration) |
| Fr | Praxis: `helm template <chart> -f values.yaml` — Output Zeile für Zeile lesen und mit K8s-Wissen zuordnen | — |

---

## Woche 5 — Helm Advanced & sicher anwenden

**Ziel:** Helm-Änderungen ohne Angst, Fehler selbst debuggen, fremde Charts lesen

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Go-Templates tief: `{{ .Values.x }}`, `{{ .Release.Name }}`, `{{ if }}`, `{{ range }}`, `{{ include }}` | [YAML-UND-HELM.md — Kap. 7](03-konfiguration/YAML-UND-HELM.md#7-helm-templates--wie-variablen-funktionieren) |
| Di | Helm Hooks: `pre-install`, `post-upgrade`, `pre-delete` — Reihenfolge und was schiefgehen kann | [YAML-UND-HELM.md — Kap. 7](03-konfiguration/YAML-UND-HELM.md#7-helm-templates--wie-variablen-funktionieren) |
| Mi | Häufige Helm-Fehler: warum Cluster kaputtgehen, wie man es vorher erkennt | [YAML-UND-HELM.md — Kap. 13](03-konfiguration/YAML-UND-HELM.md#13-häufige-fehler-ihre-ursachen-und-lösungen) |
| Do | Unbekannte Charts lesen: wie du einen fremden Chart ohne Doku verstehst | [YAML-UND-HELM.md — Kap. 14](03-konfiguration/YAML-UND-HELM.md#14-wie-du-unbekannte-charts-liest-und-verstehst) |
| Fr | Helm + ArgoCD: wie ArgoCD intern `helm template` aufruft — dann eigenen Mini-Chart schreiben und testen | [YAML-UND-HELM.md — Kap. 9–10](03-konfiguration/YAML-UND-HELM.md#9-argocd-application--der-kleber-zwischen-git-und-cluster) |

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
| Mo | ArgoCD-Komponenten: Application Controller, Repo Server, API Server, Dex — wer macht was intern | [ARCHITEKTUR.md — Kap. 5](02-kubernetes/ARCHITEKTUR.md#5-wie-argocd-intern-funktioniert) |
| Di | ArgoCD Reconciliation Loop: wie er Git-Zustand mit Cluster-Zustand vergleicht — Schritt für Schritt | [KUBERNETES-KONZEPTE.md — Kap. 11](02-kubernetes/KUBERNETES-KONZEPTE.md#11-argocd--gitops-erklärt) |
| Mi | CRDs: was eine `Application`-Ressource intern ist, wie K8s sie verarbeitet | [YAML-UND-HELM.md — Kap. 9](03-konfiguration/YAML-UND-HELM.md#9-argocd-application--der-kleber-zwischen-git-und-cluster) |
| Do | OutOfSync, Degraded, Health-Checks: warum sie entstehen, wie ArgoCD sie berechnet | [FEHLER-DOKU.md — alle Fehler](../troubleshooting/FEHLER-DOKU.md) |
| Fr | Praxis: `kubectl get application -n argocd -o yaml` + Controller-Logs lesen — rohen ArgoCD-State verstehen | — |

---

## Woche 7 — Synthese: Alles zusammen

**Ziel:** Die komplette Prozesskette von Git Push bis laufendem Pod aus dem Kopf erklären können

| Tag | Aufgabe | Quelle |
|---|---|---|
| Mo | Komplette Prozesskette: Git Push → ArgoCD → Helm render → kubectl apply → Pod läuft | [ARCHITEKTUR.md — Kap. 3](02-kubernetes/ARCHITEKTUR.md#3-die-reise-eines-git-commits) + [Kap. 10](02-kubernetes/ARCHITEKTUR.md#10-das-vollständige-zusammenspiel) |
| Di | Ausfallszenarien: was passiert wenn ArgoCD, Traefik oder cert-manager ausfällt | [ARCHITEKTUR.md — Kap. 8–9](02-kubernetes/ARCHITEKTUR.md#8-warum-jede-komponente-die-andere-braucht) |
| Mi | HOW-TO-CREATE komplett lesen: Setup von Null — mit jetzt tiefem Wissen alles nachvollziehen | [CLUSTER-AUFBAUEN.md — komplett](../setup/CLUSTER-AUFBAUEN.md) |
| Do | SETUP-DOKU Kap. 9–12: Setup-Schritte + Debugging-Referenz + kubectl-Befehle | [SETUP-DOKU.md — Kap. 9](05-setup/SETUP-DOKU.md#9-das-setup-schritt-für-schritt) + [Kap. 12](05-setup/SETUP-DOKU.md#12-wichtige-kubectl-befehle-referenz) |
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
| `lernen/SETUP-DOKU.md` | 1.570 | Woche 1, 2, 7 |
| `setup/SEALED-SECRETS.md` | 359 | Zusatz: Secrets sicher in Git verwalten |

---

## Add-on: Tägliche Wiederholung (1 Stunde zuhause)

> Der 7-Wochen-Plan bleibt unverändert — dieser Block ist ein paralleles System
> das abends zuhause läuft. Ziel: was tagsüber neu gelernt wurde, abends festigen.
> Kein neuer Stoff. Nur Wiederholung und Verständnis vertiefen.

### Das Prinzip

Spaced Repetition: neues Wissen vergisst man schnell wenn man es nicht wiederholt.
Die erste Wiederholung am selben Abend ist die wichtigste — danach sinkt die Kurve langsamer.

```
Tag 0 (Arbeit):    Neues Thema lernen
Tag 0 (Abend):     Wiederholung des heutigen Themas       ← wichtigste Wiederholung
Tag 1 (Abend):     Wiederholung von gestern + vorgestern
Tag 3 (Abend):     Wochenthemen nochmal komplett
Tag 7 (Abend):     Woche komplett zusammenfassen
```

### Die 1 Stunde aufteilen

| Zeit | Aktivität |
|------|-----------|
| 0–15 min | Heutiges Thema nochmal lesen (nur Überschriften + Code-Blöcke) |
| 15–35 min | Aus dem Kopf erklären — was war das Thema? Wie funktioniert es? |
| 35–50 min | Am Cluster nachvollziehen (kubectl, ArgoCD UI, Logs lesen) |
| 50–60 min | Eine offene Frage aufschreiben die noch unklar ist |

### Was "aus dem Kopf erklären" bedeutet

Nicht nochmal lesen. Stattdessen:
- Dokument schließen
- In eigenen Worten aufschreiben oder laut erklären was du heute gelernt hast
- Dann Dokument öffnen und prüfen was du vergessen hast

Was du vergessen hast → das ist genau das was du morgen nochmal liest.

### Wiederholungsplan pro Woche

**Woche 1 — Control Plane**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | API-Server + etcd — wie kommunizieren sie? |
| Di-Abend | etcd + Scheduler — was speichert etcd, wie wählt der Scheduler? |
| Mi-Abend | Mo + Di zusammen — Reconciliation Loop von Anfang bis Ende erklären |
| Do-Abend | Reconciliation Loop + was passiert bei `kubectl apply`? |
| Fr-Abend | Ganze Woche: Control Plane komplett aus dem Kopf skizzieren |

**Woche 2 — Workloads**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | Deployment → ReplicaSet → Pod — Hierarchie erklären |
| Di-Abend | ConfigMap vs. Secret — Unterschied, wie sie in Pods landen |
| Mi-Abend | PVC/PV — was passiert wenn ein Pod neu startet? |
| Do-Abend | RBAC — wer darf was, ServiceAccount erklären |
| Fr-Abend | Woche 1 + 2 zusammen — alle Objekte und ihre Beziehungen |

**Woche 3 — Netzwerk**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | OSI-Schichten 1–4 aus dem Kopf — was passiert auf welcher Schicht? |
| Di-Abend | kube-proxy: wie kommt ein Request von ClusterIP zum Pod? |
| Mi-Abend | ClusterIP vs. NodePort vs. LoadBalancer — wann welcher? |
| Do-Abend | Traefik: Request kommt rein — was passiert Schritt für Schritt? |
| Fr-Abend | Kompletter Netzwerkweg: Browser → DNS → Traefik → Service → Pod |

**Woche 4 — Helm Basics**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | YAML: 3 Fallstricke aufschreiben die du heute gelernt hast |
| Di-Abend | Was ist ein Helm Chart? Struktur aus dem Kopf zeichnen |
| Mi-Abend | Chart.yaml + values.yaml + templates/ — was liegt wo und warum? |
| Do-Abend | `helm template` ausführen und Output einem K8s-Objekt zuordnen |
| Fr-Abend | Woche 4 gesamt: was macht Helm intern wenn du `helm install` rufst? |

**Woche 5 — Helm Advanced**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | Go-Template: ein einfaches `{{ if }}` und `{{ range }}` selbst schreiben |
| Di-Abend | Helm Hooks: Reihenfolge aufschreiben (pre-install → install → post-install) |
| Mi-Abend | Einen Helm-Fehler aus `FEHLER-DOKU.md` nochmal nachvollziehen |
| Do-Abend | Einen fremden Chart öffnen und 3 Dinge erklären was er macht |
| Fr-Abend | Woche 4 + 5: `helm template` → Output → was applyt ArgoCD davon? |

**Woche 6 — ArgoCD**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | ArgoCD-Komponenten: wer macht was — ohne Nachschauen aufschreiben |
| Di-Abend | Reconciliation Loop: Git Push → Pod läuft — alle Schritte |
| Mi-Abend | Application-CRD: was bedeuten source, destination, syncPolicy? |
| Do-Abend | ArgoCD UI: OutOfSync-App anschauen und Diff verstehen |
| Fr-Abend | Woche 1–6: Komplette Prozesskette von `git push` bis `Pod Running` |

**Woche 7 — Synthese**

| Abend | Wiederhole |
|-------|-----------|
| Mo-Abend | Prozesskette nochmal — diesmal mit Zeitangaben (wie lange dauert was?) |
| Di-Abend | Ausfallszenario: "Traefik ist weg" — was passiert? Was nicht? |
| Mi-Abend | CLUSTER-AUFBAUEN.md: jeden Schritt lesen und erklären warum er nötig ist |
| Do-Abend | Offene Fragen der letzten 7 Wochen nochmal nachschlagen |
| Fr-Abend | Abschluss: die komplette Architektur aus dem Kopf skizzieren |

### Cluster-Befehle für die Wiederholung

Diese Befehle helfen dir dabei Theorie am echten Cluster zu verankern:

```bash
# Was läuft gerade — Überblick verschaffen
kubectl get all -A

# Reconciliation beobachten — ArgoCD live
kubectl get application -n argocd -w

# Netzwerkweg nachvollziehen
kubectl get endpoints -A
kubectl get svc -A

# Helm was hat ArgoCD gerendert?
kubectl get application traefik -n argocd -o jsonpath='{.status.history[0].source}'

# Events der letzten Stunde
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Logs eines Controllers lesen
kubectl logs -n argocd deployment/argocd-application-controller --tail=50
```

### Wenn du 1 Stunde nicht schaffst

10 Minuten reichen für das Minimum:
1. Heutiges Thema — eine Überschrift lesen
2. Die Frage beantworten: "Was ist das Kernprinzip davon?"
3. Einen kubectl-Befehl ausführen der damit zu tun hat

Besser 10 Minuten täglich als einmal pro Woche 2 Stunden.
