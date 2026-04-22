# Debugging: Warum die UI nicht kam (`:latest` in GitOps)

## Was war das Symptom?

Der Browser zeigte `Cannot GET /` (Express-404) für die miniapp — also keine UI,
obwohl im Repo längst eine Version mit HTML-Seite existierte.

## Was war tatsächlich falsch?

Im Cluster lief ein **älteres** Image, als in der Registry lag:

| Wer | Image-Digest |
|---|---|
| Pods im Cluster | `sha256:f4540d…` (alt, kein `/`-Handler) |
| `:latest` in GHCR | `sha256:2f2bbff4…` (neu, mit UI) |

Die Pods waren 43h alt und hatten nie ein neues Image gezogen.

## Image-Tags — was sie eigentlich sind

Ein Image besteht aus zwei Arten von Adressen:

```
ghcr.io/jakobulus123/miniapp:latest
                             ^^^^^^
                             Tag  =  menschenlesbares Label, mutable

ghcr.io/jakobulus123/miniapp@sha256:2f2bbff4d5ed2218eadaa91b...
                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                             Digest =  SHA-Hash des Image-Inhalts, immutable
```

**Der Digest ist die Wahrheit.** Er ist der SHA256 über den tatsächlichen
Image-Content (Layer, Manifest, Config). Bekommst du zweimal denselben Digest,
hast du byte-identischen Inhalt.

**Der Tag ist nur ein Zeiger** auf einen Digest. In der Registry liegt im
Grunde eine Tabelle:

| Tag | zeigt auf Digest |
|---|---|
| `latest` | `sha256:2f2bbff4…` |
| `v1.0.0` | `sha256:2f2bbff4…` |
| `6445eabc…` | `sha256:2f2bbff4…` |
| `c1e93196…` | `sha256:f4540d65…` |

Ein `docker push :latest` überschreibt einfach den Eintrag in der Tabelle —
der alte Digest bleibt abrufbar, aber das Label `latest` zeigt jetzt woanders
hin. *Das* ist was Tags „mutable" macht.

### Welche Tag-Arten gibt es und wann helfen sie?

| Tag-Typ | Beispiel | Mutable? | Gut für … | Problem |
|---|---|---|---|---|
| `:latest` | `:latest` | ja | lokale Experimente | zeigt *irgendwohin*, Prod-Gift |
| Umgebung | `:stable`, `:prod` | ja | manuelle Promotion | kein Audit, Rollback-Hölle |
| Branch | `:main`, `:feature-x` | ja | CI-Preview-Deploys | jeder Push verschiebt's |
| Datum | `:2026-04-22` | ja¹ | Nightly-Builds | Kollisionen bei 2 Builds/Tag |
| Semver | `:v1.2.3` | konventionell nein | Releases, Humans | Konvention, keine Garantie |
| Commit SHA | `:6445eabc…` | de-facto nein | CI, GitOps | lang, nicht hübsch |
| Digest | `@sha256:2f2bbff4…` | garantiert nein | kritische Prod | unleserlich, kein Rollback-Pfad |

¹ *Technisch überschreibbar wie jedes Tag — nur Konvention hält einen davon ab.*

### Warum nicht einfach alles mit `:latest`?

Das Tag selbst ist nicht kaputt — es macht genau das, was draufsteht: „zeig auf
den neuesten Push". Problematisch wird es, sobald **Entscheidungen davon
abhängen**, die Reproduzierbarkeit brauchen:

- **Cache-Verhalten**: Der Node zieht ein Image mit Tag `:latest` einmal und
  cacht den Digest. Selbst `imagePullPolicy: Always` hilft nur *beim
  Pod-Start* — ein Pod, der schon läuft, kümmert sich nicht um neue Pushes.
  → Zwei Pods mit gleichem Tag können unterschiedliche Digests fahren.
- **Rollback undefiniert**: „Version von gestern" — welcher Digest war das?
  Ohne Aufzeichnung weg.
- **Drift zwischen Environments**: Staging hat `:latest` von Dienstag,
  Prod von Montag. Keiner merkt's, bis was knallt.
- **GitOps-Sync-Lüge**: ArgoCD sagt „Synced", weil im Git `:latest` steht und
  im Cluster `:latest` läuft. Dass *hinter* `:latest` zwei verschiedene
  Digests stehen, interessiert den Differ nicht.
- **Keine Audit-Spur**: Du kannst nicht aus `git log` rekonstruieren, welcher
  Build wann lief.

### Warum nicht einfach immer Digests (`@sha256:…`)?

Digests sind maximal korrekt, aber unpraktisch:

- 64-Hex-Zeichen — niemand tippt das freiwillig.
- Man sieht nicht, *welche* Version gemeint ist (ist das v1.2.3 oder v1.2.4?).
- Rollback heißt „alten Digest raussuchen" — kein semantischer Pfad zurück.

**In der Praxis nimmt man Tags als menschenlesbare Zeiger und verlässt sich
auf Konvention für Immutability.** Semver-Tags (`:v1.2.3`) oder Commit-SHAs
werden nach dem ersten Push nicht überschrieben, selbst wenn die Registry das
technisch erlauben würde. Für besonders kritische Deploys pinnt man
zusätzlich den Digest (`image:tag@sha256:…` — Tag für Lesbarkeit, Digest für
Garantie).

### Faustregeln

- **Lokal / Dev-Maschine**: `:latest` ist ok — du willst eh immer das Neueste.
- **CI-Builds / PR-Previews**: Commit-SHA oder Branch-Name — eindeutig pro Build.
- **Staging / Prod via GitOps**: Commit-SHA oder Semver, nie `:latest`. Jeder
  Deploy ist ein Git-Commit, der den Tag bumpt.
- **Sicherheitskritisch**: Digest-Pinning dazu.

Der Rest dieser Doku geht davon aus, dass „Tag benutzen" = Commit-SHA-Tag
heißt. Daher hat der Fix weiter unten `tag: "6445eabc…"` statt `tag: "v1.0.0"` —
mit einer GitHub Action, die pro Commit baut, fällt einem der SHA-Tag
praktisch in den Schoß.

## Warum zieht ArgoCD nicht automatisch das neue `:latest`?

Das ist der zentrale Denkfehler. **ArgoCD synct Manifest-Stand, nicht Image-Stand.**

1. ArgoCD liest den Helm-Chart aus Git.
2. Der Chart sagt: „Deployment mit Image `miniapp:latest`".
3. ArgoCD checkt: Läuft ein Deployment mit Image `miniapp:latest`? → Ja → **Synced**.
4. Dass sich der *Digest* hinter `:latest` geändert hat, interessiert ArgoCD nicht —
   weil im Git-Manifest nichts anders geworden ist.

Ergebnis: Solange man `:latest` benutzt, kann jede Registry-Version im Cluster
landen oder auch nicht — ArgoCD trifft keine Entscheidung darüber.

## Warum hilft `imagePullPolicy: Always` nicht?

`Always` heißt: *beim Pod-Start* frisches Image ziehen.
Ein laufender Pod wird davon nicht neugestartet. Die 43h alten Pods behielten
ihren gecachten Digest — bis jemand einen Rollout triggert.

## Warum war `kubectl rollout restart` keine gute Lösung?

Mit `syncPolicy.automated.selfHeal: true` erkennt ArgoCD jede Abweichung vom
Git-Stand und macht sie rückgängig. Ein manueller `kubectl rollout restart`
setzt eine Annotation auf das Deployment; selfHeal kann das als Drift werten
und den Restart-Marker rauswerfen. Außerdem bleibt der Fix nirgends
dokumentiert — nach dem nächsten Sync ist alles beim Alten.

**Regel:** In GitOps-Clustern gehören Änderungen in den Git-Repo, nicht in
`kubectl edit`.

## Der Fix

In `charts/miniapp/values.yaml`:

```diff
 image:
   repository: ghcr.io/jakobulus123/miniapp
-  pullPolicy: Always
-  tag: "latest"
+  pullPolicy: IfNotPresent
+  tag: "6445eabc66585128a1d8ee187bbc01a80b52bd6f"
```

Dann: `git commit && git push` → ArgoCD erkennt die Manifest-Änderung →
Deployment bekommt ein neues `image:` Feld → alte Pods werden durch
neue ersetzt → UI erscheint.

## Warum SHA-Tag statt `:latest`?

- **Immutable**: `ghcr.io/…/miniapp:6445eabc…` zeigt für immer auf denselben
  Build. Kein „mystery digest".
- **Reproduzierbar**: `git checkout <commit>` → gleicher Code, gleiches Image.
- **ArgoCD-tauglich**: Jeder neue Build braucht einen Git-Commit, der den Tag
  bumpt. Deployment-Entscheidung lebt komplett in Git.
- **Rollback trivial**: `git revert` → alter Tag → alter Build.

`IfNotPresent` reicht, weil SHA-Tags eh nie neu belegt werden — der Node
cached einmal und gut.

## Was bedeutet das für den Workflow?

Bei `:latest`:
```
Code push → GH Actions baut :latest → Registry hat neues Image → NIEMAND deployt es
```

Mit SHA-Tag:
```
Code push → GH Actions baut :<sha> und pusht → Chart braucht einen Tag-Bump in Git
```

Drei Wege, den Tag-Bump zu machen:

1. **Manuell**: Nach jedem Build den Tag in `values.yaml` updaten und committen.
2. **ArgoCD Image Updater**: Zusatz-Controller, der die Registry pollt und den
   Tag-Bump automatisch commitet. Setup: <https://argocd-image-updater.readthedocs.io/>
3. **CI-Commit-Step**: Die GitHub Action, die das Image baut, commitet im
   Anschluss den neuen Tag in den Helm-Repo.

## Weitere Fallstricke aus der Session

### Es gab (scheinbar) keinen Ingress

`kubectl get ingress -n miniapp` zeigte nichts — weil Traefik eigene CRDs
benutzt (`IngressRoute`, nicht Standard-`Ingress`).

Richtiger Check:
```
kubectl get ingressroutes.traefik.io -n miniapp
```

Die IngressRoutes kommen aus `manifests/miniapp/ingress.yaml` und werden von
der separaten ArgoCD-App `manifests` (nicht `miniapp`) deployt.

### DNS war ein Red Herring beim Image-Problem

Der 42h hängende Let's-Encrypt-Challenge (`jakob-miniapp.goava.ai` zeigt per
OVH-DNS auf `80.158.91.8` statt Cluster-IP `89.167.116.105`) ist ein
**separates, echtes Problem** — aber es war nicht die Ursache für die fehlende
UI. Die UI war in lokalem `kubectl port-forward` trotzdem nicht da, weil das
gecachte Image keinen `/`-Handler hatte. DNS-Fix und Image-Fix sind
unabhängig.

### ArgoCD sieht neue Git-Commits nicht sofort

Default-Refresh-Interval ist ~3 Minuten. Zum sofortigen Erzwingen:

```
kubectl annotate app -n argocd miniapp argocd.argoproj.io/refresh=hard --overwrite
```

## Zum lokalen Testen der App

Weil DNS noch nicht zeigt und du sowieso nur localhost willst:

```
# auf dem Cluster-Host
kubectl port-forward -n miniapp svc/miniapp 9090:80

# auf dem eigenen PC (SSH-Tunnel)
ssh -L 9090:localhost:9090 <user>@<cluster-host>

# Browser
http://localhost:9090
```

## Kurz-Checkliste bei „Pods laufen aber App ist alt"

1. Welches Image läuft tatsächlich? → `kubectl get pods -n <ns> -o jsonpath='{.items[*].status.containerStatuses[*].imageID}'`
2. Wird `:latest` oder ein fester Tag benutzt? (`:latest` → immer suspekt)
3. Wie alt sind die Pods? → `kubectl get pods -n <ns>` — ältere Pods cachen altes Image.
4. Wer managed das Deployment? ArgoCD, Flux, `kubectl`? → `kubectl get app -n argocd`
5. Ist Git-Revision in ArgoCD = aktueller HEAD? → `.status.sync.revisions`
6. Falls GitOps + `:latest`: Tag in Git auf SHA bumpen und committen.
