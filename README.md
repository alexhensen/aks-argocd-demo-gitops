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

## URL's

| Omgeving | URL |
|---|---|
| Argo CD | http://argocd.20.103.113.146.nip.io |
| main | http://app.20.103.113.146.nip.io |
| preview | `http://pr-<nummer>.20.103.113.146.nip.io` |

`nip.io` vertaalt een IP in de hostnaam naar datzelfde IP, dus er is geen
DNS-zone nodig.

## Opnieuw opbouwen na een nieuw cluster

Het IP van de ingress controller zit in deze bestanden verwerkt. Bij een nieuw
cluster verandert dat IP en moet het op drie plekken worden bijgewerkt:
`argocd-values.yaml`, `application-main.yaml` en `applicationset-previews.yaml`.
