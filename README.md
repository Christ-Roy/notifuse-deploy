# notifuse-deploy

> Repo de déploiement GitOps pour la stack Notifuse Veridian.
> Extrait du monorepo `Christ-Roy/veridian-platform` le 2026-05-13.

## Ce que contient ce repo

Ce repo **ne contient PAS de code applicatif**. Il contient uniquement l'infrastructure de déploiement de Notifuse en production + le tracking du fork upstream :

| Fichier | Rôle |
|---|---|
| `docker-compose.yml` | Compose source de vérité pour Dokploy (GitOps). Pin SHA-digest, DEPLOY_ENV-aware pour blue-green. |
| `.env.example` | Liste des variables d'env attendues par la stack (sans valeurs — secrets dans Dokploy UI). |
| `runbooks/deploy.md` | Runbook complet : redeploy, bump image, rollback, smoke, secrets. |
| `.github/workflows/security-cron.yml` | Cron Trivy quotidien sur l'image deployed (détecte CVE upstream). |
| `notifuse/` | Scaffold fork upstream : `README.md` (overview saasification), `RELEASE.md` (convention `saas-vX.Y.Z`), `MERGING-UPSTREAM.md` (procédure rebase), `.upstream-version` (version Notifuse trackée), `DEPLOY-STAGING.md`, `env.example`, `compose.snippet.yml`. |
| `todo/TODO.md` | Chantiers Notifuse, follow-ups identifiés, recently shipped. |
| `todo/UI-REVIEW.md` | File d'attente UI polish solo. |

## Source du code applicatif

L'image Docker `ghcr.io/christ-roy/notifuse-veridian:saas-vX.Y.Z` est buildée par le **fork** :
- Repo : <https://github.com/Christ-Roy/notifuse-veridian>
- Branche : `veridian` (tous les patches Veridian — pilotage HMAC depuis Hub, paywall Go natif, magic link cross-app, webhooks sortants)
- Workflow build : `.github/workflows/veridian-ci.yml` sur le runner self-hosted du dev server

## Auto-deploy Dokploy

Dokploy (`compose-transmit-open-source-microchip-k9lvap`) pointe sur ce repo en mode Git provider :

| Champ | Valeur |
|---|---|
| Provider | Git |
| Repository | `https://github.com/Christ-Roy/notifuse-deploy.git` |
| Branch | `main` |
| Compose path | `./docker-compose.yml` |
| Auto Deploy | ✅ webhook GitHub |

Chaque push sur `main` qui touche `docker-compose.yml` → webhook → redeploy zero-downtime.

## Comment ça s'utilise

### Cas standard — modifier le compose

```bash
git checkout -b chore/<sujet>
# ... edit docker-compose.yml ...
git push -u origin chore/<sujet>
gh pr create --fill
gh pr merge --squash
# Le webhook GitHub déclenche Dokploy → docker compose up zero-downtime
```

### Cas — bump image Notifuse (release fork upstream)

Deux options :

**Rapide (sans PR)** : Dokploy UI → Stack notifuse-prod → Environment → modifier `NOTIFUSE_IMAGE_TAG` + `NOTIFUSE_IMAGE_DIGEST` → Redeploy.

**Auditable (PR)** : modifier les valeurs par défaut dans `docker-compose.yml` + `.env.example`, ouvrir une PR.

### Rollback

```bash
git revert -m 1 <merge-commit-sha>
git push origin main
# Webhook GitHub → Dokploy redéploie l'état précédent
```

## Historique

- **2026-05-13** : Migration GitOps initiale réalisée dans le monorepo `Christ-Roy/veridian-platform` (PR #89, #95, #96, #97, #98, #100), puis **extrait dans ce repo standalone** suite à la décision de séparer chaque app en son propre repo deploy. Migration scaffold fork + TODO ajoutée le même jour (commit `6e9e993`). Worktree monorepo `~/Bureau/veridian-platform-notifuse/` supprimé, nouveau worktree de travail : `~/Bureau/notifuse-deploy/`.

  Pendant la transition, le compose, runbook, scaffold et TODO restent également dans le monorepo (`infra/services/notifuse/`, `runbooks/services/notifuse/`, `notifuse/`, `todo/apps/notifuse/`) comme **fallback** — ils seront supprimés du monorepo à partir du 2026-05-20 (7+ jours de prod stable sur ce repo).

  Découvertes documentées pendant la migration (cf `todo/TODO.md` follow-ups) :
  1. Manifest digest registry vs `.Image` local (Docker veut le digest publié sur le registry, pas l'image ID local renvoyé par `docker inspect <container>`)
  2. Piège Dokploy Domains : tant qu'un Domain est configuré dans l'UI, Dokploy injecte ses propres labels Traefik en parallèle de ceux du compose Git → dual-router. Fix : `POST /api/domain.delete`
  3. Tag `aquasecurity/trivy-action@v0.36.0` nécessite le préfixe `v` (le tag `0.36.0` sans `v` n'existe pas → fail "action not found")

## Sécurité

- Aucun secret en clair dans ce repo (uniquement des `${VAR}` interpolés depuis le `.env` Dokploy)
- L'env Dokploy de cette stack est l'env partagé `KmNwdMqLi9ye4xZ57WsnC` (SaaS Veridian / production) — il contient les ENV de toutes les autres apps (Stripe, Supabase, Twenty…)
- Trivy cron quotidien 3h17 UTC scanne l'image deployed
- Cf `runbooks/deploy.md` pour les détails sécurité et la procédure de rotation des secrets
