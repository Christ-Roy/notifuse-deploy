# Notifuse — runbook deploy (post-GitOps)

> Créé pendant le sprint SPRINT-GITOPS-VERIDIAN (2026-05-13).
> Décrit comment redéployer Notifuse via le repo `Christ-Roy/notifuse-deploy`.

## Architecture deploy

```
Push docker-compose.yml sur main (repo notifuse-deploy)
   ↓
Webhook GitHub → Dokploy compose-transmit-open-source-microchip-k9lvap
   ↓
docker compose up zero-downtime (healthcheck /healthz gate)
```

Notifuse n'a pas de CI build d'image dans ce repo (l'image est un fork
upstream maintenu sur `Christ-Roy/notifuse-veridian`, branche `veridian`,
publiée sur GHCR avec convention `saas-vX.Y.Z`). Ce repo concerne uniquement
le **compose** de la stack.

## Stack Dokploy notifuse-prod

| Champ | Valeur |
|---|---|
| Compose ID | `compose-transmit-open-source-microchip-k9lvap` (composeId `WN0jglLj5bDIrXUFZHNmw`) |
| Provider | Git |
| Repository | `https://github.com/Christ-Roy/notifuse-deploy.git` |
| Branche | `main` |
| Path | `./docker-compose.yml` |
| Watch paths | `docker-compose.yml` |
| Auto Deploy | webhook GitHub |
| Containers | `compose-transmit-open-source-microchip-k9lvap-notifuse-prod-1` (app) + `-notifuse-prod-db-1` (DB) |
| Image app | `ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.3@sha256:<digest>` |
| Image DB | `postgres:17-alpine@sha256:<digest>` |
| Domaine | `notifuse.app.veridian.site` (Traefik letsencrypt) |
| DB | `notifuse-prod-db:5432/notifuse_system` (Postgres dédié interne au compose) |
| Healthcheck | `wget http://127.0.0.1:8081/healthz` (interval 10s, timeout 5s, 3 retries, start 10s) |
| Healthcheck externe | `GET https://notifuse.app.veridian.site/healthz` → 200 |

## Cas standard — modifier le compose

```bash
cd ~/Bureau/notifuse-deploy
git checkout -b chore/<sujet> origin/main
# ... modif docker-compose.yml ...
git push -u origin chore/<sujet>
gh pr create --fill
gh pr merge --squash
# Le webhook GitHub déclenche Dokploy → docker compose up zero-downtime
```

## Cas — bump de l'image Notifuse (release fork upstream)

Quand le fork `christ-roy/notifuse-veridian` publie une nouvelle release
`saas-vX.Y.Z` sur GHCR, deux options pour la mettre en prod :

### Option 1 (rapide, sans PR) — Dokploy ENV

```
Dokploy UI → Stack notifuse-prod → Environment
  NOTIFUSE_IMAGE_TAG=saas-v1.0.4
  NOTIFUSE_IMAGE_DIGEST=<nouveau digest sha256>
  → Redeploy
```

Récupérer le digest :
```bash
# Après que le tag soit publié sur GHCR
docker manifest inspect ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.4 \
  | jq -r '.config.digest' | sed 's|sha256:||'
```

### Option 2 (audit Git, recommandé pour releases) — PR sur le compose

```bash
cd ~/Bureau/notifuse-deploy
git checkout -b chore/bump-saas-v1.0.4 origin/main
# Edit docker-compose.yml lignes :
#   image: ghcr.io/christ-roy/notifuse-veridian:${NOTIFUSE_IMAGE_TAG:-saas-v1.0.4}@sha256:${NOTIFUSE_IMAGE_DIGEST:-<nouveau digest>}
# Edit .env.example pour aligner les fallbacks
git commit -am "chore: bump image to saas-v1.0.4"
git push -u origin chore/bump-saas-v1.0.4
gh pr create --fill
gh pr merge --squash
```

Note : on **n'est pas obligé** de bumper le digest à chaque release — c'est une défense
en profondeur. Le tag `saas-vX.Y.Z` reste valable car le compose utilise
`${NOTIFUSE_IMAGE_TAG:-saas-v1.0.3}`.

## Rollback prod

### Rollback rapide compose (< 5 min)

```bash
cd ~/Bureau/notifuse-deploy
git checkout main && git pull origin main
git revert -m 1 <merge-commit-sha>
git push origin main
```

Le webhook GitHub redéploie l'état précédent en zero-downtime.

### Rollback d'urgence (image cassée mais compose OK)

Override le tag image via Dokploy ENV :

```
Dokploy UI → Stack notifuse-prod → Environment
  NOTIFUSE_IMAGE_TAG=saas-v1.0.2  (version précédente)
  NOTIFUSE_IMAGE_DIGEST=<digest correspondant>
  → Redeploy
```

### Rollback critique (DB perdue)

La DB `notifuse-prod-db` utilise un volume Docker nommé externe `infra_notifuse-db-data`.
Le compose ne le supprime jamais. Si la DB est corrompue :

```bash
# Restore depuis le backup Dokploy (cron à valider — P0.1 todo/infra)
# Ou snapshot manuel pre-incident :
ssh prod-pub 'sudo docker exec compose-transmit-open-source-microchip-k9lvap-notifuse-prod-db-1 \
  pg_restore -U postgres -d notifuse_system --clean < /path/to/dump.dump'
```

## Smoke post-deploy

```bash
# Health
curl -sf https://notifuse.app.veridian.site/healthz
# attendu : 200 + body simple

# Console UI
for i in 1 2 3 4 5; do
  curl -sI -o /dev/null -w "%{http_code} " https://notifuse.app.veridian.site/
done; echo
# attendu : 307 ou 200

# Logs app (recherche d'erreurs récentes)
ssh prod-pub 'docker logs --since 60s compose-transmit-open-source-microchip-k9lvap-notifuse-prod-1 2>&1 | grep -iE "error|fatal|panic|5[0-9]{2}"'

# Logs DB
ssh prod-pub 'docker logs --since 60s compose-transmit-open-source-microchip-k9lvap-notifuse-prod-db-1 2>&1 | tail -20'

# Vérifier que le scheduler interne ne fait plus de round-trip Cloudflare
# (cf patch INTERNAL_API_ENDPOINT, dette-technique/002)
ssh prod-pub 'docker logs --since 5m compose-transmit-open-source-microchip-k9lvap-notifuse-prod-1 2>&1 | grep -c "tasks.execute"'
# attendu : volume normal de tâches scheduler (pas 480k/jour)
```

## Validation Service Worker cache (piège connu)

⚠️ Notifuse Console UI utilise un Service Worker qui cache agressivement les
assets Lingui (cf mémoire `project_notifuse_console_sw_cache.md`). Après un deploy :

1. Premier load post-deploy montre parfois l'ancienne version
2. Utiliser `?cachebust=$(date +%s)` pour forcer le bypass SW
3. Ctrl+Shift+R ne suffit pas toujours — `chrome://serviceworker-internals` + Unregister

Pour valider qu'un déploiement a bien pris :
```bash
# Vérifier le SHA de l'image qui tourne réellement
ssh prod-pub 'docker inspect compose-transmit-open-source-microchip-k9lvap-notifuse-prod-1 --format "{{.Image}}"'
# Doit matcher le NOTIFUSE_IMAGE_DIGEST attendu
```

## Secrets — où ils sont

**Dokploy UI → Stack notifuse-prod → Environment.** Liste des noms attendus :
`.env.example` à la racine de ce repo.

⚠️ L'env Dokploy de cette stack est l'env **partagé** `KmNwdMqLi9ye4xZ57WsnC`
(SaaS Veridian / production) — il contient les ENV de toutes les autres apps aussi
(Stripe, Supabase, Twenty…). Faire attention en modifiant : ne pas écraser des
vars utilisées par d'autres stacks.

Aucun secret dans le repo Git. Aucun secret dans le `.env.example` (juste les noms).

Pour les ajouter/modifier : Dokploy UI uniquement. Après chaque modif d'ENV →
Redeploy manuel ou push no-op pour déclencher le webhook.

## Monitoring CVE Notifuse

L'image Notifuse n'a pas de CI build dans ce repo (image upstream produite par
le fork `Christ-Roy/notifuse-veridian`). Pour détecter les CVE :

- **Workflow cron actif** : `.github/workflows/security-cron.yml` — Trivy sur
  l'image deployed quotidiennement à 3h17 UTC, fail si CRIT/HIGH apparait.
- **Scan ponctuel** : `ssh prod-pub 'cd /home/ubuntu/veridian/grafana/trivy && \
  sudo docker compose run --rm trivy image ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.3'`

Si CVE détectée sur l'image upstream → ouvrir une issue sur le fork
`Christ-Roy/notifuse-veridian` pour bumper la base image, attendre la nouvelle
release `saas-vX.Y.Z`, puis bump via la procédure de bump ci-dessus.

Si CVE détectée sur l'image upstream → ouvrir une issue sur le fork
`christ-roy/notifuse-veridian` pour bumper la base image, attendre la nouvelle
release `saas-vX.Y.Z`, puis bump via la procédure ci-dessus.

## Forensique pré-migration (snapshot 2026-05-13)

Conservé dans `/tmp/notifuse-pre-gitops-20260513-1327/` (à garder 7 jours min) :
- `docker-compose.live.yml` — compose Raw original
- `containers-inspect.json` — état complet des 2 containers
- `volumes.txt` — confirme `infra_notifuse-data` et `infra_notifuse-db-data`
- `.env.live` — chmod 600, contient tous les secrets prod (NE JAMAIS commit)
