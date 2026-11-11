# Renovate

Nightly Dependency-Bot fuer meine Projekt-Repos. Selbst gehostet, OSS-Image
(AGPL-3.0), kein Mend-Account.

Deckt alle Oekosysteme der Projekte in einem Lauf ab: npm, NuGet (.NET 10),
Maven (Quarkus), Docker und pip.

| Repo | Org | Stack |
|---|---|---|
| RodeoTCG | eigen | npm (Shopify Hydrogen) |
| card-sniper | eigen | pip, Docker |
| cardomat | Raffaelio | npm (Shopify Hydrogen) |
| SW_Install | Dinamo77 | NuGet, npm, Docker |
| tarot-beratung | LisasSeelenpost | Maven, npm, Docker |
| vido | Michael-VIDO | Maven, npm, Docker |

Dieses Repo (`cluster`) steht absichtlich **nicht** in der Liste -- die
Helm-Chart-Versionen hier macht schon der woechentliche `auto-update`-Job.

## Inbetriebnahme

Der CronJob ist mit `suspend: true` ausgeliefert und laeuft nicht, bis die
folgenden drei Schritte durch sind.

### 1. GitHub-Token anlegen

Ein **fine-grained PAT** auf <https://github.com/settings/personal-access-tokens>.
Kein GitHub App Token: ein App-Token ist auf *eine* Org gescoped, die
Kundenrepos liegen aber in vier verschiedenen Orgs. Ein PAT, mit dem ich
ueberall Collaborator bin, erledigt alle sechs Repos in einem Lauf.

Zugriff auf die sechs Repos aus der Tabelle, mit:

| Permission | Level | Wofuer |
|---|---|---|
| Contents | Read and write | Branches fuer die PRs |
| Pull requests | Read and write | PRs aufmachen |
| Issues | Read and write | Dependency Dashboard |
| Metadata | Read-only | wird automatisch verlangt |

Dazu auf Org-Ebene **Members: Read-only**, sonst kann Renovate die
Kundenrepos nicht aufloesen.

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

Das ist Absicht: vier der sechs Repos gehoeren Kunden. In cardomat sind
~27 Dependencies veraltet -- ohne diese Bremse stehen da ueber Nacht
entsprechend viele PRs im Repo eines Kunden.

Ausnahme: fuer die eigenen Repos (RodeoTCG, card-sniper) laufen Minor- und
Patch-Updates ohne Nachfrage durch.

Soll ein Kundenrepo auch ohne Nachfrage PRs bekommen, das Repo in
`configmap.yaml` in die `matchRepositories`-Liste der zweiten `packageRule`
aufnehmen.

## Gotify

Absichtlich nicht eingebaut. Dafuer muesste das Container-Command um einen
Shell-Wrapper erweitert werden, und ein Fehler darin laesst den ganzen Lauf
scheitern -- fuer einen Job, der sowieso ueber GitHub-Issues benachrichtigt,
ein schlechter Tausch. Fehlgeschlagene Jobs sieht man ueber die bestehende
Monitoring-/alertmanager-gotify-Kette.

## Updates des Bots selbst

Der Image-Tag ist gepinnt. `latest` waere hier besonders unklug: ein
Dependency-Bot, der sich selbst unkontrolliert aktualisiert, ist genau das
Problem, das er loesen soll. Aktuelle Tags:
<https://hub.docker.com/r/renovate/renovate/tags>
