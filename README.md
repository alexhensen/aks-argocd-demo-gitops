# aks-argocd-demo-gitops

De gewenste toestand van het cluster. Argo CD leest deze repository; er wordt
nooit vanaf een laptop of pipeline naar het cluster gedeployed.

## Structuur

| Pad | Inhoud |
|---|---|
| `charts/demo-app` | Eén Helm chart die zowel de permanente als de tijdelijke omgevingen beschrijft |
| `bootstrap/root.yaml` | App-of-apps: beheert alles in `bootstrap/`, inclusief zichzelf |
| `bootstrap/application-main.yaml` | De permanente `main` omgeving |
| `bootstrap/applicationset-previews.yaml` | Genereert één omgeving per pull request |
| `bootstrap/argocd-values.yaml` | Helm waarden waarmee Argo CD zelf is geïnstalleerd |
| `DEMO.md` | Draaiboek voor de presentatie |

Dezelfde chart voor preview en main is bewust: wat je in een pull request test
is exact het manifest dat straks naar main gaat.

## Hoe een preview ontstaat

1. Een pull request in `aks-argocd-demo-app` bouwt een image met tag `sha-<commit>`.
2. Pas ná een geslaagde push zet CI het label `preview` op de pull request.
3. De `pullRequest` generator ziet dat label en genereert een `Application`.
4. Argo CD maakt namespace `preview-pr-<n>` en rolt precies die commit uit.
5. Sluit de pull request, dan verdwijnt het label uit beeld, vervalt de
   `Application` en ruimt Argo CD de namespace op.

Het label is de poort: zonder geslaagde build geen omgeving.

## Hoe main wordt bijgewerkt

CI heeft geen inloggegevens voor het cluster. Na een merge naar main werkt de
workflow alleen de image tag in `bootstrap/application-main.yaml` bij. Argo CD
ziet die commit en rolt uit. Daardoor is de Git-historie van deze repository
tegelijk het deployment-logboek.

## URL's

| Omgeving | URL |
|---|---|
| Argo CD | https://argocd.demo.alexhensen.com (basic auth + admin login) |
| main | https://app.demo.alexhensen.com |
| preview | `https://pr-<nummer>.demo.alexhensen.com` |

Het domein `alexhensen.com` heeft al een wildcard-record dat naar de
bestaande productiesite wijst. Om daar niet mee te botsen staat er precies
één extra laag onder: `demo.alexhensen.com` en `*.demo.alexhensen.com` zijn
expliciete A-records naar het ingress IP, bij TransIP toegevoegd (niet in
Azure DNS — de DNS-zone van dit domein zit bij de registrar). TLS-certificaten
komen automatisch van Let's Encrypt via cert-manager; zie
`bootstrap/cluster-issuers.yaml`.

## Azure-omgeving

Het cluster `aks-argocd-demo` (resourcegroep `rg-argocd-demo`, regio
`westeurope`) draait in een losse, lege Azure-subscription die uitsluitend
voor deze demo dient. Gebruik bij elk `az`-commando altijd `--subscription`
expliciet: de resourcegroepnaam is niet uniek over subscriptions heen, en de
default subscription van de CLI-sessie kan per gebruiker/moment verschillen.

## Opnieuw opbouwen na een nieuw cluster of migratie naar een andere subscription

Het IP van de ingress controller zit in deze bestanden verwerkt. Bij een nieuw
cluster (ook bij verhuizing naar een andere Azure-subscription) verandert dat
IP en moet het bijgewerkt worden: het TransIP A-record voor `demo` en
`*.demo`, plus (als er tijdelijk met nip.io wordt gewerkt voordat DNS is
aangepast) de hostnamen in `argocd-values.yaml`, `application-main.yaml` en
`applicationset-previews.yaml`.

Stappen voor een migratie naar een andere subscription (reproduceerbaar,
zonder data om te migreren):

1. Resourcegroep en AKS-cluster aanmaken in de doel-subscription met dezelfde
   specificaties (1 node, `Standard_D2as_v5`, `--load-balancer-sku standard`).
2. `ingress-nginx`, `cert-manager` en Argo CD installeren met de Helm-waarden
   uit dit repository (`bootstrap/argocd-values.yaml`).
3. De secrets `argocd-basic-auth` en `github-token` overzetten (bevatten
   gevoelige waarden, staan bewust niet in Git).
4. `bootstrap/cluster-issuers.yaml` en `bootstrap/root.yaml` toepassen; Argo
   CD synchroniseert vanaf dat moment zelfstandig alles uit deze repository,
   inclusief eventuele openstaande preview-omgevingen.
5. Het nieuwe ingress-IP ophalen en de TransIP A-records (`demo` en `*.demo`)
   bijwerken.
6. Pas na een geslaagde TLS-uitgifte (`kubectl get certificate -A`) het oude
   cluster verwijderen — niet eerder, anders is er geen werkende fallback.

### Bekend probleem: gloednieuwe subscription, inconsistente externe bereikbaarheid

Bij de migratie naar een subscription die voor het eerst publieke resources
kreeg, bleek het ingress-IP van buitenaf inconsistent bereikbaar: prima
vanaf sommige netwerken, maar `Timeout during connect` vanaf andere —
onder meer vanaf een GitHub Actions-runner (Azure `centralus`) en vanaf zowel
Let's Encrypt als ZeroSSL (twee onafhankelijke CA's, dus geen CA-specifiek
probleem). Uitgesloten als oorzaak, met bewijs:

- NSG, load balancer-regels en -probes: identiek aan de werkende configuratie
  en met `az network nsg rule show` bevestigd dat poort 80 én 443 open staan
  voor `Internet`.
- nginx/cert-manager zelf: een `kubectl run curltest ... curl http://<ip>/`
  **vanuit een pod in hetzelfde cluster** kreeg gewoon de verwachte 308-
  redirect; de ingress werkt dus correct.
- Het specifieke IP: een volledig nieuw toegewezen publiek IP (andere
  Azure-range) vertoonde exact hetzelfde patroon, dus het zat niet aan één
  ongelukkig IP-adres vast.

Conclusie: dit is een Azure-platformkarakteristiek van een *net voor het
eerst publieke resources uitrollende* subscription, geen fout in deze
repository of in het cluster. Twee opties:

1. **Wachten** (kan langer duren dan de gebruikelijke BGP-propagatie van
   enkele minuten — in de praktijk eerder uren). Forceer geen herhaalde
   `Certificate`-verwijdering: Let's Encrypt en ZeroSSL hanteren allebei een
   limiet van ongeveer vijf mislukte validaties per uur per hostnaam
   (per CA/account apart geteld).
2. **Een Azure-supportticket openen** onder vermelding van deze bevindingen
   als het na enkele uren nog steeds niet oplost; dit is geen probleem dat
   via Kubernetes- of NSG-configuratie op te lossen is.

Zodra het probleem is verdwenen, pakt cert-manager automatisch de
certificaatuitgifte weer op zonder verdere actie.
