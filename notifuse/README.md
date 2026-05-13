# Notifuse — Fork Veridian (saasifie)

Fork de [Notifuse/notifuse](https://github.com/Notifuse/notifuse) avec patches
Veridian pour le rendre **vraiment SaaS-ready** : pilotable HMAC depuis le Hub,
paywall Go natif, magic link cross-app, webhooks sortants vers le Hub.

## Repos

- **Upstream** : https://github.com/Notifuse/notifuse (branche `main`, tags `vX.Y`)
- **Fork Veridian** : https://github.com/Christ-Roy/notifuse-veridian
- **Branche Veridian** : `veridian` (tous nos patches vivent ici, **jamais** sur `main`)
- **Image custom** : `ghcr.io/christ-roy/notifuse-veridian:saas-vX.Y.Z`
  pour la prod (SemVer Veridian, tag manuel — voir `RELEASE.md`).
  Auto-build `:vUPSTREAM-veridian.<sha8>` + `:latest` a chaque push
  `veridian` pour le staging. Build par self-hosted runner sur le dev
  server (workflow `.github/workflows/veridian-ci.yml` cote fork).

## Version upstream actuelle

Voir `.upstream-version` (actuellement `v30.1`, sortie 2026-04-27).

## Ce dossier (`notifuse/` dans le monorepo)

Ce dossier **ne contient pas le code Notifuse** — il sert de point de
synchronisation entre le monorepo Veridian et le fork :

- `README.md` — ce fichier
- `RELEASE.md` — procedure de release prod versionnee (`saas-vX.Y.Z`)
- `DEPLOY-STAGING.md` — procedure first-time setup staging
- `MERGING-UPSTREAM.md` — procedure rebase de la branche `veridian` sur les
  nouveaux tags `vX.Y` upstream
- `.upstream-version` — version upstream actuellement utilisee
- `compose.snippet.yml` — entree docker-compose Veridian (reportee dans
  `infra/docker-compose.{prod,staging}.yml`)
- `env.example` — env vars Veridian-specific (HUB_API_SECRET, HUB_WEBHOOK_URL,
  MAGIC_LINK_SECRET, etc.) en plus de celles upstream

Le code Notifuse vit dans le repo `Christ-Roy/notifuse-veridian` sur la branche
`veridian`. Le monorepo en consomme l'image GHCR.

## Patches Veridian (sur branche `veridian` du fork)

| Fichier ajoute                                          | Role                                              |
|---------------------------------------------------------|---------------------------------------------------|
| `internal/http/middleware/veridian_hmac.go`             | Verifie X-Veridian-Hub-Signature sur /api/tenants/* |
| `internal/http/middleware/veridian_paywall.go`          | Bloque envois si suspended ou quota depasse       |
| `internal/http/veridian_handler.go`                     | 6 endpoints standard saas (provision, suspend, ...) |
| `internal/http/veridian_magic_handler.go`               | POST /api/workspaces.generateMagicLink (auth API key tenant) |
| `internal/service/veridian_service.go`                  | Logique provision : workspace + owner reel + API key |
| `internal/service/veridian_webhook_emitter.go`          | Push events vers Hub (tenant.*, email.*)          |
| `internal/repository/veridian_plan_repository.go`       | CRUD table veridian_plan (status, quota, suspended) |
| `internal/database/schema/veridian_plan.go`             | Migration table veridian_plan                     |
| `internal/domain/veridian.go`                           | Types : VeridianPlan, ProvisionInput, etc.        |

## Endpoints Veridian (en plus des endpoints upstream)

Tous proteges par middleware HMAC `X-Veridian-Hub-Signature` (voir
`docs/saas-standards.md` §6.1 dans le monorepo).

| Methode + Path                         | Body                                              | Reponse                                            |
|----------------------------------------|---------------------------------------------------|----------------------------------------------------|
| `POST /api/tenants/provision`          | `{ tenant_id, owner_email, plan }`                | `{ workspace_id, owner_user_id, api_key, magic_link }` |
| `POST /api/tenants/update-plan`        | `{ tenant_id, plan }`                             | `{ tenant_id, plan, applied_at }`                  |
| `POST /api/tenants/suspend`            | `{ tenant_id, reason }`                           | `{ tenant_id, suspended_at }`                      |
| `POST /api/tenants/resume`             | `{ tenant_id }`                                   | `{ tenant_id, resumed_at }`                        |
| `DELETE /api/tenants/:id`              | —                                                 | `{ tenant_id, deleted_at }`                        |
| `GET /api/tenants/:id/status`          | —                                                 | `{ status, plan, usage, limits }`                  |
| `POST /api/workspaces.generateMagicLink` | `{ user_email }` (auth: API key tenant Notifuse) | `{ magic_link, expires_at }`                       |

## Env vars Veridian (a configurer en plus d'upstream)

```
HUB_API_SECRET=                # secret partage avec le Hub pour HMAC verify
HUB_WEBHOOK_URL=               # ex: https://app.veridian.site/api/webhooks/notifuse
HUB_WEBHOOK_SECRET=            # secret pour signer les events sortants
VERIDIAN_DEFAULT_PLAN=free     # plan par defaut sur provision si non precise
```

## Workflow update upstream

1. Lire le changelog upstream (`https://github.com/Notifuse/notifuse/releases`)
2. Tester le tag cible en staging (image upstream brute)
3. Suivre `MERGING-UPSTREAM.md` pour rebase la branche `veridian` sur le tag
4. Build image GHCR via CI
5. Deploy staging Dokploy, smoke test, deploy prod

## Statut

- **Sprint en cours** : P1.3 (voir `../todo/apps/notifuse/TODO.md`)
- **Fork** : OK (Christ-Roy/notifuse-veridian)
- **Branche veridian** : OK (depuis v30.1)
- **Patches Go** : en cours
- **Image custom** : en cours
- **CI** : `.github/workflows/notifuse-ci.yml` (a compléter en P1.3)
