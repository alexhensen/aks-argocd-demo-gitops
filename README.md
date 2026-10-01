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

Let's Encrypt-validatie van een gloednieuw publiek IP kan de eerste tijd
falen met `Timeout during connect` terwijl het IP wel degelijk bereikbaar is
(HTTP-01 valideert vanaf meerdere netwerklocaties; een nieuw toegewezen
cloud-IP is niet overal meteen even goed routeerbaar). Forceer niet
herhaaldelijk een nieuwe poging: Let's Encrypt hanteert een limiet van vijf
mislukte validaties per uur per hostnaam. Verwijder de `Certificate`
(`kubectl delete certificate -n <ns> <naam>`) pas opnieuw na voldoende
wachttijd, of laat cert-manager het vanzelf opnieuw proberen.
