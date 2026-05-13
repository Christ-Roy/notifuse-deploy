# Deploy Notifuse Veridian — Staging Procedure

> Documentation deploy staging apres push fork. A executer une seule fois,
> ensuite la CI gere les bumps automatiques.

## Etat actuel (2026-05-07)

- Fork `Christ-Roy/notifuse-veridian` branche `veridian` PUSHED ✅
- Monorepo branche `main` PUSHED ✅
- CI fork : `Notifuse Veridian CI/CD` declenche au push, build image GHCR
- CI monorepo : Hub CI/CD + Notifuse scaffold check declenches

## Pre-requis avant deploy staging

### 1. Generer les secrets HMAC

```bash
HUB_API_SECRET=$(openssl rand -hex 32)
HUB_WEBHOOK_SECRET=$(openssl rand -hex 32)
echo "HUB_API_SECRET=${HUB_API_SECRET}"
echo "HUB_WEBHOOK_SECRET=${HUB_WEBHOOK_SECRET}"
```

Garder ces valeurs : elles doivent matcher entre Hub et Notifuse.

### 2. Configurer Dokploy stack staging Notifuse

Ouvrir Dokploy UI → stack `staging-notifuse` → onglet `Environment` :

```
NOTIFUSE_HUB_API_SECRET=<HUB_API_SECRET genere ci-dessus>
NOTIFUSE_HUB_WEBHOOK_URL=https://saas-hub.staging.veridian.site/api/webhooks/notifuse
NOTIFUSE_HUB_WEBHOOK_SECRET=<HUB_WEBHOOK_SECRET genere ci-dessus>
VERIDIAN_DEFAULT_PLAN=free
```

Le compose `infra/docker-compose.staging.yml` du monorepo lit ces vars
via interpolation `${...}`.

### 3. Configurer Dokploy stack staging Hub

Memes secrets (cote Hub) :

```
NOTIFUSE_HUB_API_SECRET=<MEME valeur que ci-dessus>
NOTIFUSE_HUB_WEBHOOK_SECRET=<MEME valeur que ci-dessus>
```

### 4. Reporter les composes dans Dokploy

Coller `infra/docker-compose.staging.yml` (mis a jour avec image
`ghcr.io/christ-roy/notifuse-veridian:latest`) dans Dokploy stack staging.

## Ordre de deploy

### Etape 1 : Attendre CI fork verte

```bash
gh run list --repo Christ-Roy/notifuse-veridian --limit 3
gh run watch <run_id> --repo Christ-Roy/notifuse-veridian
```

La CI fork inclut `deploy-staging` automatique. Si Dokploy API + token
sont configures dans les secrets GitHub :

- DOKPLOY_API_TOKEN
- DOKPLOY_URL
- DOKPLOY_NOTIFUSE_STAGING_COMPOSE_ID
- DOKPLOY_NOTIFUSE_PROD_COMPOSE_ID

Sinon : deploy manuel via Dokploy UI quand l'image est sur GHCR.

### Etape 2 : Verifier image GHCR

```bash
docker pull ghcr.io/christ-roy/notifuse-veridian:latest
docker inspect ghcr.io/christ-roy/notifuse-veridian:latest | grep "Created"
```

### Etape 3 : Deploy staging

Soit auto via CI (si secrets Dokploy configures), soit manuel via Dokploy UI :
- Dokploy → stack `staging-notifuse` → bouton "Redeploy"
- Verifier les logs : `Server starting on 0.0.0.0:8081`
- Healthcheck : `curl https://saas-notifuse.staging.veridian.site/api/setup.status`

### Etape 4 : Setup wizard premiere fois

Premier boot : `is_installed=false`. Deux options :
- Manuel : ouvrir https://saas-notifuse.staging.veridian.site/console/setup
  et passer le wizard (ROOT_EMAIL, SMTP)
- Skip wizard via env vars upstream Notifuse (a investiguer si feasible
  dans un patch futur)

### Etape 5 : Smoke test endpoints Veridian

```bash
# Test rapide HMAC depuis terminal (curl + openssl)
HUB_API_SECRET=<la valeur>
TS=$(date +%s%3N)
BODY='{"tenant_id":"smoke1","owner_email":"smoke@test.veridian.site","plan":"free"}'
SIG=$(echo -n "${TS}.${BODY}" | openssl dgst -sha256 -hmac "${HUB_API_SECRET}" -hex | awk '{print $2}')

curl -X POST https://saas-notifuse.staging.veridian.site/api/tenants/provision \
  -H "Content-Type: application/json" \
  -H "X-Veridian-Hub-Signature: ${SIG}" \
  -H "X-Veridian-Timestamp: ${TS}" \
  -d "${BODY}"
```

Reponse attendue : `200 OK { workspace_id, owner_user_id, api_key, magic_link, ... }`

### Etape 6 : E2E Playwright headful

```bash
cd ~/Bureau/notifuse-veridian/tests/e2e-veridian
npm install
npx playwright install chromium
NOTIFUSE_URL=https://saas-notifuse.staging.veridian.site \
HUB_API_SECRET=<la valeur> \
npx playwright test --headed
```

Doit faire passer les 5 specs (saasification + 4 chaos).

### Etape 7 : Bump prod (apres staging vert)

Auto via CI workflow : si e2e-staging vert → deploy-prod auto.
Sinon manuel via Dokploy UI stack `prod-notifuse`.

**Note importante** : la prod actuelle tourne `notifuse/notifuse:v27.0`
upstream. Le bump v27 → v30.1-veridian.1 inclut migration auto v28+v29+v30
+ veridian_plan. **Backup DB notifuse-postgres prod avant bump**.

## Rollback en cas de probleme

CI inclut rollback auto sur fail e2e-prod. Sinon manuellement :

```bash
# Dokploy UI : changer image vers ghcr.io/christ-roy/notifuse-veridian:rollback
# (image precedente automatiquement retaguee par la CI)
```

Ou retour a l'image upstream pure :

```bash
# Dokploy UI : image = notifuse/notifuse:v30.1
# (perd les patches Veridian mais redonne un Notifuse fonctionnel)
```

## Variables d'env recap

| Variable                      | Cote        | Description                                   |
|-------------------------------|-------------|-----------------------------------------------|
| `HUB_API_SECRET`              | Notifuse    | Secret HMAC verify Hub→Notifuse              |
| `HUB_WEBHOOK_URL`             | Notifuse    | URL Hub recv events                           |
| `HUB_WEBHOOK_SECRET`          | Notifuse    | Secret signe events sortants                  |
| `VERIDIAN_DEFAULT_PLAN`       | Notifuse    | Plan par defaut sur provision (`free`)        |
| `NOTIFUSE_HUB_API_SECRET`     | Hub         | Memes valeur que `HUB_API_SECRET`             |
| `NOTIFUSE_HUB_WEBHOOK_SECRET` | Hub         | Memes valeur que `HUB_WEBHOOK_SECRET`         |
| `NOTIFUSE_API_URL`            | Hub         | URL Notifuse interne ou publique              |
