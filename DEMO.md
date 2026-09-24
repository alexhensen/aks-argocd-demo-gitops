# Demo draaiboek

Een doorloop van ongeveer twaalf minuten. Zet vooraf twee vensters klaar:
de Argo CD UI en een browser met de `main` omgeving.

| Rol | URL |
|---|---|
| Argo CD | https://argocd.demo.alexhensen.com (basic auth: `demo`, dan admin-login) |
| main | https://app.demo.alexhensen.com |
| preview | `https://pr-<nummer>.demo.alexhensen.com` |

Het basic-auth-wachtwoord staat in het sessiebestand
`files/argocd-basic-auth-password.txt`; dit is een extra laag vóór de
Argo CD-inlogpagina zelf.

Wachtwoord ophalen:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

## 1. Het uitgangspunt

Laat de `main` omgeving zien: blauw, drie landen. Open in Argo CD de
applicatie `demo-app-main` en wijs op de image tag. Die is een commit, geen
`latest`. Daardoor is altijd terug te zien wélke code draait.

## 2. Niemand deployt naar het cluster

Open `bootstrap/application-main.yaml` in de GitOps repository en laat de
commit-historie zien: elke regel "Promoveer … naar main" is door CI gezet.
CI heeft geen kubeconfig en geen cluster-credentials; de enige schrijfactie
is een commit. Het cluster trekt zelf.

## 3. Een pull request krijgt een eigen wereld

Open een pull request met een zichtbare wijziging, bijvoorbeeld een extra land
in de seed data van `src/Program.cs`.

Terwijl de build loopt: laat zien dat er nog géén omgeving is. Dat is bewust.
Pas als de image in GHCR staat, zet de workflow het label `preview`. De
`pullRequest` generator van Argo CD filtert op dat label.

Ongeveer een halve minuut later verschijnt `preview-pr-<n>` in Argo CD. De
URL staat als comment onder de pull request.

## 4. Isolatie tonen

Zet de preview en `main` naast elkaar. Andere kleur, ander pull request
nummer, andere commit, andere landenlijst.

Voeg in de preview een land toe. Ververs `main`: daar verandert niets. De
staat hoort bij de omgeving, niet bij de applicatie.

## 5. Drift verdwijnt vanzelf

```bash
kubectl delete deployment demo-app -n preview-pr-<n>
```

Ververs de pagina: kort stuk. Binnen een minuut staat hij er weer. Argo CD
vergelijkt continu met Git en herstelt wat afwijkt. Dat is het verschil
tussen "we hebben ooit gedeployed" en "dit is wat er hoort te draaien".

## 6. Opruimen is niet optioneel

Sluit de pull request.

```bash
kubectl get ns | grep preview
```

De namespace is weg. Geen opruimticket, geen vergeten omgeving, geen kosten
die blijven doorlopen. De levensduur van de omgeving is gelijk aan de
levensduur van de pull request.

## Vragen die altijd komen

**Wat als de build faalt?** Dan komt het label er niet en ontstaat er geen
omgeving. Een kapotte commit kan geen preview opleveren.

**En data?** Deze demo houdt alles in het geheugen. In de praktijk is dat de
moeilijkste vraag: een eigen database per preview, of een gedeelde database
met een schema per omgeving. Dat is een ontwerpkeuze, geen Argo CD instelling.

**En identiteiten en secrets?** Zie hieronder; dit is meestal de echte rem.

## Wat deze demo bewust níét laat zien

Eerlijk blijven over de afstand tot productie maakt het verhaal sterker.

- Het cluster heeft een publiek endpoint. Sinds de koppeling met
  `demo.alexhensen.com` draait alles wel over HTTPS met een geldig
  Let's Encrypt-certificaat.
- Er is geen authenticatie voor de previews; iedereen met de URL komt erbij.
  De preview-ingress zet wel `noindex` zodat ze niet in zoekmachines komen.
- Er zit geen database, geen migratie en geen realistische testdata in.
- Er is geen automatische vervaltijd; een pull request die maanden openstaat
  houdt zijn omgeving. In productie hoort daar een TTL op.
- Workload identity ontbreekt. Bij een federated credential die vastzit aan
  `system:serviceaccount:<namespace>:<naam>` werkt een gekopieerd
  serviceaccount in een nieuwe namespace niet zomaar; per preview een
  federated credential aanmaken en opruimen is extra werk.
- Het GitHub token staat als los secret in de `argocd` namespace. Voor iets
  wat langer meegaat hoort daar een GitHub App achter.

## Kosten

Het cluster kost ongeveer honderdvijftig euro per maand als het blijft
draaien. Tussen presentaties door:

```bash
az aks stop  --resource-group rg-argocd-demo --name aks-argocd-demo
az aks start --resource-group rg-argocd-demo --name aks-argocd-demo
```

Na een `start` houdt het cluster hetzelfde ingress IP zolang de
LoadBalancer service blijft bestaan. Verandert het IP toch, werk dan de
hostnamen bij zoals beschreven in de README.
