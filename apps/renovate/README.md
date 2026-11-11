# Renovate

Nightly Dependency-Bot fuer meine Projekt-Repos. Selbst gehostet, OSS-Image
(AGPL-3.0), kein Mend-Account.

Deckt alle Oekosysteme der Projekte in einem Lauf ab: npm, Maven (Quarkus),
Docker und pip.

| Repo | Org | Stack |
|---|---|---|
| RodeoTCG | eigen | npm (Shopify Hydrogen) |
| card-sniper | eigen | pip, Docker |
| cardomat | Raffaelio | npm (Shopify Hydrogen) |
| tarot-beratung | LisasSeelenpost | Maven, npm, Docker |
| vido | Michael-VIDO | Maven, npm, Docker |

Dieses Repo (`cluster`) steht absichtlich **nicht** in der Liste -- die
Helm-Chart-Versionen hier macht schon der woechentliche `auto-update`-Job.

## Inbetriebnahme

Der CronJob ist mit `suspend: true` ausgeliefert und laeuft nicht, bis die
folgenden drei Schritte durch sind.

### 1. GitHub-Token anlegen

Ein **Classic PAT** auf <https://github.com/settings/tokens>, Scope `repo`.

Warum classic und nicht fine-grained: ein fine-grained Token ist laut
GitHub-Doku auf die Repos *eines* Owners begrenzt ("Each token is limited to
access resources owned by a single user or organization"). Die Kundenrepos
liegen in drei verschiedenen Orgs, dazu kommt der eigene Account -- das
waeren vier Tokens, und Renovate nimmt pro Lauf nur eines. Ein Classic PAT
umspannt alle Orgs und den eigenen Account.

Ein GitHub App Token hat dasselbe Problem: auch das ist auf eine Org
gescoped.

Der Scope `repo` deckt alles ab, was Renovate braucht -- Branches, PRs und
Issues (fuer das Dependency Dashboard). `workflow` ist nur noetig, wenn
Renovate auch Dateien unter `.github/workflows/` anfassen soll; derzeit hat
keines der fuenf Repos Workflows.

Zwei Haken:

- Der Scope ist breit. `repo` gilt fuer *alle* Repos, auf die ich Zugriff
  habe, nicht nur die fuenf in der Liste. Die Einschraenkung auf die fuenf
  macht allein Renovates `repositories`-Liste.
- Orgs koennen Classic PATs blockieren. Falls eine Kunden-Org das tut,
  erscheint sie nicht bzw. der Zugriff schlaegt fehl -- dann muss ein Admin
  dort den Zugriff erlauben, oder das Repo fliegt aus der Liste.

Laufzeit begrenzen (90 Tage) und im Kalender vermerken -- laeuft das Token
ab, scheitert der Job still.

### 2. Token versiegeln

Der sealed-secrets-Controller ist nicht ueber dieses Repo deployt, also
zuerst nachsehen, wo er laeuft:

```sh
kubectl get deploy -A | grep sealed-secrets
```

Dann (`<ns>` durch das Ergebnis ersetzen):

```sh
kubectl create secret generic renovate-github-token \
  --namespace renovate \
  --from-literal=RENOVATE_TOKEN='github_pat_...' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml --controller-namespace <ns> \
  > apps/renovate/github-token-sealedsecret.yaml
```

Dann in `kustomization.yaml` die Zeile `- github-token-sealedsecret.yaml`
einkommentieren.

### 3. Scharf schalten

In `cronjob.yaml` `suspend: true` auf `false` setzen, committen, ArgoCD
synct.

Ersten Lauf nicht abwarten, sondern antriggern:

```sh
kubectl create job -n renovate --from=cronjob/renovate renovate-manual
kubectl logs -n renovate -f job/renovate-manual
```

## Wie es sich verhaelt

Standardmaessig entsteht **kein PR von allein**. Renovate legt pro Repo ein
Issue "Dependency Dashboard" an und baut einen PR erst, wenn dort eine
Checkbox angehakt wird (`dependencyDashboardApproval: true`).

Das ist Absicht: drei der fuenf Repos gehoeren Kunden. In cardomat sind
~27 Dependencies veraltet -- ohne diese Bremse stehen da ueber Nacht
entsprechend viele PRs im Repo eines Kunden.

Ausnahme: fuer die eigenen Repos (RodeoTCG, card-sniper) laufen Minor- und
Patch-Updates ohne Nachfrage durch.

Soll ein Kundenrepo auch ohne Nachfrage PRs bekommen, das Repo in
`configmap.yaml` in die `matchRepositories`-Liste der zweiten `packageRule`
aufnehmen.

## Gotify (notify.yaml)

Ein **eigener** CronJob um 03:30, nicht als Wrapper um das Renovate-Command:
ein Fehler im Benachrichtigungs-Skript soll den Renovate-Lauf nicht
mitreissen.

Meldet drei Dinge und schweigt sonst:

| Ereignis | Prioritaet |
|---|---|
| neue Renovate-PRs in cardomat oder vido | 7 |
| neue Renovate-PRs nur in den uebrigen Repos | 4 |
| Renovate-Lauf fehlgeschlagen | 8 |
| kein Renovate-Lauf im Zeitfenster | 8 |

cardomat und vido sind hoeher priorisiert, weil die beiden Shops live und in
taeglicher Benutzung sind.

Der letzte Punkt ist der wichtigste: ein stiller Ausfall -- typischerweise ein
abgelaufenes GitHub-Token -- faellt sonst wochenlang nicht auf. Genau das
Problem, das dieses Setup loesen soll.

Renovate-PRs werden am Branch-Prefix `renovate/` erkannt, nicht am Author:
mit einem PAT ist der Author der Kontoinhaber, nicht ein Bot. Dadurch werden
Dependabot-PRs (`dependabot/...`) korrekt ignoriert.

### Inbetriebnahme

Der CronJob ist mit `suspend: true` ausgeliefert. Es braucht einen
Gotify-App-Token:

```sh
printf 'Gotify-Token: '; read -rs GT; echo
kubectl create secret generic renovate-gotify \
  --namespace renovate \
  --from-literal=GOTIFY_TOKEN="$GT" \
  --dry-run=client -o yaml \
  | kubeseal --format yaml --controller-namespace kube-system \
  > apps/renovate/gotify-sealedsecret.yaml
unset GT
```

SealedSecrets sind auf Name **und** Namespace versiegelt -- der Token aus
`cluster-cleanup-gotify` laesst sich nicht wiederverwenden, selbst wenn es
derselbe Gotify-App-Token waere.

Dann in `kustomization.yaml` die Zeile `- gotify-sealedsecret.yaml`
einkommentieren und in `notify.yaml` `suspend` auf `false` setzen.

Testlauf, ohne bis 03:30 zu warten:

```sh
kubectl create job -n renovate --from=cronjob/renovate-notify notify-test
kubectl logs -n renovate -f job/notify-test
```

Beim Testlauf ausserhalb des 90-Minuten-Fensters nach einem Renovate-Lauf
kommt eine "Renovate ausgefallen"-Meldung. Das ist korrekt und zeigt, dass
die Ausfallerkennung funktioniert.

## Updates des Bots selbst

Der Image-Tag ist gepinnt. `latest` waere hier besonders unklug: ein
Dependency-Bot, der sich selbst unkontrolliert aktualisiert, ist genau das
Problem, das er loesen soll. Aktuelle Tags:
<https://hub.docker.com/r/renovate/renovate/tags>
