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
