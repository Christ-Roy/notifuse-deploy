# Merging upstream Notifuse dans notre fork

Procedure pour rebase la branche `veridian` sur un nouveau tag stable
`Notifuse/notifuse vX.Y`.

**Principe** : on suit uniquement les tags stables `vX.Y` (jamais les commits
intermediaires). On rebase notre branche `veridian` par-dessus (pas de merge
commit) pour garder un diff Veridian propre et lisible.

## Pre-requis (one-shot)

```bash
git clone https://github.com/Christ-Roy/notifuse-veridian.git
cd notifuse-veridian
git remote add upstream https://github.com/Notifuse/notifuse.git
git fetch upstream --tags
```

## Procedure de rebase sur un nouveau tag (ex: v30.1 → v31.0)

### 1. Fetch les tags upstream

```bash
git fetch upstream --force --tags
```

Le `--force` est necessaire si upstream a deja retag (rare, mais possible).

### 2. Lire le changelog

```bash
git log upstream/v30.1..upstream/v31.0 -- CHANGELOG.md
```

Identifier les **breaking changes** (renames, suppressions d'endpoints,
changements de signature). Si un breaking change touche nos patches Veridian,
preparer mentalement les conflits attendus avant de rebase.

### 3. Tester l'image upstream brute en staging

Avant de rebase, deployer `notifuse/notifuse:v31.0` brut en staging pour
verifier qu'on part d'une base saine. Si ca casse, ce n'est pas notre fork
qui est en cause — on attend un patch upstream.

```bash
# Update infra/docker-compose.staging.yml
image: notifuse/notifuse:v31.0
# Reporter dans Dokploy + deploy
```

Smoke test : `curl -sS https://saas-notifuse.staging.veridian.site/api/setup.status`

### 4. Rebase la branche `veridian` sur le nouveau tag

```bash
git checkout veridian
git rebase v31.0
```

Conflits attendus (zones touchees par nos patches) :

- `internal/app/app.go` : registration de nos handlers/middleware
- `internal/http/router.go` : route registration (si upstream a refactor)
- `internal/database/migrations.go` : si upstream a ajoute une migration

Pour chaque conflit :

1. Lire le diff upstream pour comprendre l'intention
2. Reapliquer notre patch en respectant la nouvelle structure upstream
3. Si la signature d'une fonction qu'on appelle a change : adapter notre code
4. **NE JAMAIS** abandonner un patch Veridian sous pretexte que le rebase est
   penible — c'est exactement pour ces cas qu'on a une branche dediee

```bash
git add <fichier_resolu>
git rebase --continue
```

### 5. Faire tourner les tests upstream

```bash
make test
```

Si un test natif Notifuse echoue **a cause de notre patch**, c'est qu'on a
casse le contrat upstream. A corriger avant de pousser.

### 6. Faire tourner nos tests Veridian

```bash
go test ./internal/http/... -run "Veridian" -v
go test ./internal/service/... -run "Veridian" -v
go test ./internal/repository/... -run "Veridian" -v
```

### 7. Update `.upstream-version` dans notifuse-deploy

```bash
cd ~/Bureau/notifuse-deploy/notifuse
echo "v31.0" > .upstream-version
git add .upstream-version
git commit -m "chore: bump upstream to v31.0"
```

### 8. Push la branche `veridian`

```bash
cd /home/brunon5/Bureau/notifuse-veridian
git push --force-with-lease origin veridian
```

`--force-with-lease` est OBLIGATOIRE apres rebase. JAMAIS `--force` brut.

### 9. Build image + deploy staging

La CI `.github/workflows/notifuse-ci.yml` du fork builde automatiquement
`ghcr.io/christ-roy/notifuse-veridian:v31.0-veridian.1` apres push.

Update `infra/docker-compose.staging.yml` dans le monorepo, deploy via Dokploy,
smoke test, valide, puis prod.

## Conflits courants (a documenter au fil de l'eau)

### Conflit `internal/app/app.go`

- **Cause** : upstream ajoute/retire des handlers, on doit re-injecter nos
  handlers Veridian dans la chaine.
- **Resolution type** : garder les nouveaux handlers upstream + reajouter
  `RegisterVeridianRoutes(mux, ...)` apres la derniere registration upstream.

### Conflit `internal/database/migrations.go`

- **Cause** : upstream ajoute une migration `vNN`, notre migration
  `veridian_plan` doit s'inserer avant ou apres selon les dependances.
- **Resolution type** : si pas de dependance, mettre nos migrations apres
  toutes celles upstream pour minimiser les conflits futurs.

## Notes

- Upstream est tres actif (v27 → v30.1 en 3 mois). Suivre le rythme.
- Si un rebase devient ingerable (>20 conflits non triviaux), envisager
  `git merge upstream/v31.0` en commit unique plutot que rebase.
- Garder cette doc a jour : chaque conflit non trivial rencontre = paragraphe
  ajoute dans "Conflits courants".
