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

## Opnieuw opbouwen na een nieuw cluster

Het IP van de ingress controller zit in deze bestanden verwerkt. Bij een nieuw
cluster verandert dat IP en moet het bijgewerkt worden: het TransIP A-record
voor `demo` en `*.demo`, plus (als er tijdelijk met nip.io wordt gewerkt
voordat DNS is aangepast) de hostnamen in `argocd-values.yaml`,
`application-main.yaml` en `applicationset-previews.yaml`.
