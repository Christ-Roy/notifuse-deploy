# Notifuse Veridian — Release Procedure (prod)

> Releases prod versionnees `saas-vX.Y.Z` (SemVer Veridian, independant
> de la version upstream Notifuse).

## Convention de tag

| Tag                         | Quand                              | Image GHCR pushee                               |
|-----------------------------|------------------------------------|-------------------------------------------------|
| `saas-vX.Y.Z`               | Release prod (manuel)              | `:saas-vX.Y.Z` (immuable, n'ecrase PAS `:latest`)|
| (push branche `veridian`)   | Auto-build pour staging            | `:vUPSTREAM-veridian.<sha8>` + `:latest`         |
| `:rollback`                 | Auto, image precedente du staging  | (mis a jour auto a chaque build veridian)        |

**Pourquoi versionner :** prod doit pointer sur un tag immuable et tracable.
Pas `:latest` (qui change a chaque push veridian = surprise).

**SemVer Veridian, pas upstream :** la version Veridian (`saas-v1.0.1`) suit
les changements cote Veridian (patches HMAC, paywall, magic links). Independant
de la version upstream Notifuse (`v30.1`, `v30.2`, etc.). Quand on rebase sur
une nouvelle version upstream, on bump la version Veridian (souvent minor).

## Procedure release prod

Repo concerne : **`Christ-Roy/notifuse-veridian` branche `veridian`** (le fork,
pas le monorepo).

### 1. Verifier que `veridian` est verte

```bash
gh run list --repo Christ-Roy/notifuse-veridian --branch veridian --limit 3
```

Le commit qu'on va tagger doit avoir passe `test-go` + `deploy-staging` +
`e2e-staging`. Sinon, on ne tag pas.

### 2. Bump le numero de version

Choisir le nouveau numero (SemVer Veridian) :

- **patch** (`v1.0.1`) : bug fix, pas de changement d'API, pas de migration
- **minor** (`v1.1.0`) : nouvelle feature compatible (ex: nouveau endpoint
  Veridian, nouveau champ optionnel sur veridian_plan)
- **major** (`v2.0.0`) : breaking change cote API ou DB (rebase sur upstream
  vMAJOR superieur, ou changement non retro-compatible des endpoints
  `/api/tenants/*`)

### 3. Creer + push le tag

```bash
cd /tmp/notifuse-fork  # ou n'importe quel clone du fork sur veridian
git checkout veridian
git pull
git tag saas-v1.0.1 -m "Release v1.0.1 — <description courte>"
git push origin saas-v1.0.1
```

Le workflow `Notifuse Veridian CI/CD` se declenche, build l'image, push
`ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.1`. **Ne touche pas
`:latest` ni `:rollback`** (sinon on casserait staging).

### 4. Suivre la CI

```bash
gh run watch --repo Christ-Roy/notifuse-veridian
```

Attendre que `build` soit verte. `deploy-staging` ne tourne pas pour les
tags `saas-v*` (volontaire — pas de re-deploy staging sur tag prod).

### 5. Update du compose Dokploy prod

Le compose Dokploy prod (`compose-transmit-open-source-microchip-k9lvap`)
doit pointer sur le nouveau tag. Editer dans Dokploy UI :

```yaml
services:
  notifuse:
    image: ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.1  # nouveau tag
```

Save + Deploy. Dokploy pull la nouvelle image et restart.

### 6. Verifier prod healthy

```bash
curl -sf https://notifuse.app.veridian.site/api/setup.status && echo OK
ssh prod-pub "docker logs --tail 50 compose-transmit-open-source-microchip-k9lvap-notifuse-1"
```

### 7. Mettre a jour `compose.snippet.yml`

Mettre a jour le snippet de doc pour refleter la version courante :

```bash
cd ~/Bureau/notifuse-deploy/notifuse
sed -i 's|notifuse-veridian:saas-v[0-9.]*|notifuse-veridian:saas-v1.0.1|' compose.snippet.yml
git add compose.snippet.yml
git commit -m "docs: bump compose snippet to saas-v1.0.1"
```

## Rollback

### Rollback rapide (image precedente)

Editer le compose Dokploy prod, remettre le tag precedent :

```yaml
image: ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.0  # version precedente
```

Save + Deploy. Pas besoin de rebuild — l'image immuable est toujours sur GHCR.

### Lister les versions disponibles

```bash
gh api "user/packages/container/notifuse-veridian/versions" \
  --jq '.[] | select(.metadata.container.tags | any(startswith("saas-v"))) | .metadata.container.tags'
```

## Premiere release (baseline)

L'image prod actuelle (digest `sha256:804ea7...`, build 2026-05-08) tourne sans
tag versionne. Pour aligner, on cree retroactivement `saas-v1.0.0` :

```bash
cd /tmp/notifuse-fork
git checkout veridian
git pull
# HEAD veridian = commit 06d8ac75 = ce qui est en prod
git tag saas-v1.0.0 06d8ac755add8ba0f40362d1746d68640840fd05 \
  -m "Release v1.0.0 — baseline (image deja en prod, tag retroactif)"
git push origin saas-v1.0.0
```

Le workflow rebuild l'image et la push en `:saas-v1.0.0`. Apres :
- Update compose Dokploy prod : `image: ...:saas-v1.0.0`
- Save + Deploy (pull la nouvelle image taggee, mais meme contenu)

## FAQ

**Pourquoi pas semantic-release auto ?** Cadence trop faible (releases
manuelles, ~1/semaine), dependance en plus, magie inutile. Un `git tag` clair
suffit.

**Pourquoi pas `:latest` aussi sur les saas-v* ?** Parce que `:latest`
est utilise par le staging auto-deploy. Si on tag un commit ancien
`saas-v1.0.5` (pour rollback ou hotfix), on ne veut pas que staging
soit retrograde silencieusement.

**Et le tag `vUPSTREAM-veridian.<sha>` historique ?** Conserve, pas touche.
Toujours produit a chaque push `veridian`. Sert au staging.
