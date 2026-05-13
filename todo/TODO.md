# Notifuse — TODO detaille

> Source de verite strategique : repo standalone `Christ-Roy/notifuse-deploy`
> (extrait du monorepo `Christ-Roy/veridian-platform` le 2026-05-13).
> UI polish solo : [`UI-REVIEW.md`](./UI-REVIEW.md)
>
> Fork leger de Notifuse OSS pour le rendre SaaS-ready. Boite noire API-only,
> pilotee par le Hub via HMAC. Stack : Go (upstream + patches Veridian).

## Etat actuel (2026-05-13)

- **Fork (code applicatif)** : `Christ-Roy/notifuse-veridian` — branche `veridian` depuis `v30.1`
- **Repo deploy (GitOps)** : `Christ-Roy/notifuse-deploy` — compose + runbook + CI security cron
- **Worktree local** : `~/Bureau/notifuse-deploy/` (le worktree monorepo `veridian-platform-notifuse` a ete supprime 2026-05-13)
- **URL prod** : https://notifuse.app.veridian.site → **`ghcr.io/christ-roy/notifuse-veridian:saas-v1.0.3`** (fork avec patches actifs depuis 2026-05-10, paywall + HMAC actifs)
- **Container prod** : `compose-transmit-open-source-microchip-k9lvap-notifuse-prod-1`
- **Compose ID Dokploy** : `WN0jglLj5bDIrXUFZHNmw` (sourceType: git, customGitUrl: notifuse-deploy.git, branch: main)
- **URL staging** : https://saas-notifuse.staging.veridian.site → `ghcr.io/christ-roy/notifuse-veridian:latest`
- **Sante** : 🟢 prod (saas-v1.0.3 actif, scheduler ~4-5 tasks/sec, 61 workspaces, 0 erreur log) / 🟢 staging
- **Webhook GitHub auto-deploy** : id 622844391 sur `notifuse-deploy` (delete sur `veridian-platform` 2026-05-13 14:30)
- **Fallback monorepo** : `veridian-platform/infra/services/notifuse/`, `runbooks/services/notifuse/`, `todo/apps/notifuse/`, `notifuse/`, workflows — conserves jusqu'a validation 7+ jours (suppression vise vers 2026-05-20)

## Sprint TERMINE — P1.3 Notifuse fork boite noire (2026-05-08)

Sprint complet livre, e2e CI green, pipeline staging validee. Voir
`session_2026-05-08_notifuse_saasification.md` dans memory.

### ✅ Fait

**Setup fork**
- [x] Fork `Christ-Roy/notifuse-veridian` branche `veridian` depuis tag upstream `v30.1`
- [x] Dossier `notifuse/` monorepo : README, MERGING-UPSTREAM.md, .upstream-version, env.example, compose.snippet.yml, DEPLOY-STAGING.md
- [x] Image custom : `ghcr.io/christ-roy/notifuse-veridian:latest` build self-hosted runner

**6 endpoints provisioning HMAC + 2 endpoints additionnels**
- [x] `POST /api/tenants/provision` — workspace + owner reel (role=owner, pas invite) + API key + auto_login_url
- [x] `POST /api/tenants/update-plan` — applique plan + recalcule quota
- [x] `POST /api/tenants/suspend` — flag suspended (middleware paywall bloque envois)
- [x] `POST /api/tenants/resume` — reactive
- [x] `DELETE /api/tenants/:id` — soft delete, refus re-provision 30j (ErrTenantSoftDeleted → 409)
- [x] `GET /api/tenants/:id/status` — plan + quota mois + status temps reel
- [x] `POST /api/workspaces.generateMagicLink` — auth via API key tenant (pas HMAC), magic_link + auto_login_url
- [x] `GET /veridian/auto-login?token=<HMAC>` — page HTML inline localStorage.setItem + redirect /console

**Middleware paywall**
- [x] `internal/http/middleware/veridian_paywall.go` : sync.Map cache TTL 60s, fail-open sur erreur DB
- [x] `VeridianPaywallPathFilter` : applique uniquement sur /api/transactional.send + /api/broadcasts.{create,schedule,sendToIndividual}
- [x] 402 Payment Required si suspended/deleted/quota depasse, fail-open si workspace pas dans veridian_plan (mode self-hosted)

**Webhook emitter sortant vers Hub**
- [x] `internal/service/veridian_webhook_emitter.go` : async (goroutine), HMAC-SHA256 X-Veridian-Notifuse-Signature
- [x] Events : tenant.provisioned, tenant.suspended, tenant.resumed, tenant.deleted, tenant.plan_changed, email.sent, email.bounced, email.complaint, tenant.quota_exceeded
- [x] Retry exponential backoff 3x sur 5xx/réseau, abandon sur 4xx
- [x] Noop si HUB_WEBHOOK_URL ou HUB_WEBHOOK_SECRET vides

**Endpoint admin wipe-test-tenants** (pour CI / admin platform)
- [x] `POST /api/veridian/admin/wipe-test-tenants` HMAC, body {prefix} ou {tenant_ids[]}
- [x] Refuse wildcards SQL et prefix < 3 chars
- [x] Skip safety_client_prefixes (apicalinfo, robinix, lyon, loyer, veridiansite)
- [x] Pour chaque match : WorkspaceService.DeleteWorkspace upstream (DROP DATABASE) + planRepo.HardDelete

**Patches upstream non-invasifs**
- [x] `UserService.GenerateMagicCodeForVeridian` : code en clair retourne (privileged), pas d'envoi email
- [x] `UserService.CreateAutoLoginSession` : session JWT directe pour user existant
- [x] Table `veridian_plan` (system DB) : workspace_id, plan, status, quota, suspended_at, deleted_at, emails_sent_this_month, last_reset_at
- [x] Repository `veridian_plan_postgres.go` : Get, Upsert, UpdatePlan, Suspend, Resume, SoftDelete, IncrementEmailsSent, ResetMonthlyCounters, HardDelete, ListByPrefix

**Tests Go**
- [x] 49 tests Veridian (sqlmock + httptest + gomock) sur repo + middleware HMAC + middleware paywall + service + webhook emitter
- [x] Tests upstream Notifuse passent toujours (0 regression sur ./internal/... ./pkg/...)

**E2E Playwright CI complet**
- [x] saasification.spec.ts : 12 steps (provision + idempotent + auto-login headful + magic link API key + status + suspend cache TTL 65s + 402 + resume + delete soft)
- [x] chaos-provisioning.spec.ts : 5 concurrent same tenant, 5 concurrent distincts, replay HMAC, tampering, delete→re-provision 409
- [x] chaos-paywall.spec.ts : suspend/resume cache 60s, delete = 402, path filter precision
- [x] chaos-magic-link.spec.ts : flow nominal headful + tampering URL token
- [x] chaos-status-and-plan.spec.ts : status apres provision/suspend/delete, upgrade/downgrade, suspend deja suspended idempotent
- [x] CI run all green : Go tests → Build GHCR → Deploy staging Dokploy + force pull + cleanup wipe → e2e staging (BLOQUANT) → deploy prod (manual)

**CI/CD pipeline complet**
- [x] `.github/workflows/veridian-ci.yml` dans le fork : test-go, build self-hosted, deploy staging, e2e staging, deploy prod manual only
- [x] Runner self-hosted enregistre sur dev-server-1 (`actions-runner-notifuse-veridian` service systemd)
- [x] Cleanup CI via API HMAC `/api/veridian/admin/wipe-test-tenants` (pas de SQL direct)
- [x] Deploy prod = workflow_dispatch manual seulement (pas auto-push, regle absolue Veridian)

## ⚠️ Reste a faire (par ordre de priorite)

### Bump prod : passer de v27 upstream → fork veridian:latest

> Bloquant pour activer le pilotage Hub en prod. Aujourd'hui prod tourne v27
> brute, sans HMAC, sans paywall, sans auto-login. Le fork est valide en
> staging mais aucun client prod ne beneficie encore des features Veridian.

- [ ] **Backup DB Notifuse prod** (pg_dump avant tout bump — clients reels :
  ismailelmouaddab, guilhemjacquet1, truy, etc.)
- [ ] **Pattern green/blue** (cf `project_blue_green_pattern.md` memory) :
  - [ ] Ajouter service `notifuse-green` au compose prod (image fork) → sous-domaine `notifuse-green.app.veridian.site`
  - [ ] Partage la meme `notifuse-postgres` que blue (les migrations Veridian sont additives : table `veridian_plan` + 2 methodes UserService, pas de breaking schema)
  - [ ] Robert teste sur green (provision tenant test, auto-login, envoi mail, suspend, etc.)
  - [ ] Si OK : switch labels Traefik blue ↔ green (5s downtime)
  - [ ] Garder blue 24-48h en standby pour rollback rapide
  - [ ] Apres validation prolongee : drop blue + retag green = `notifuse`
- [ ] Configurer secrets Dokploy prod compose (NOTIFUSE_HUB_API_SECRET / WEBHOOK_SECRET / WEBHOOK_URL)
- [ ] Script blue/green pret dans `/tmp/setup_blue_green_prod.py` (a finaliser)

### Endpoint additionnels cote fork (demandes par Hub UI P1.6)

- [ ] `POST /api/veridian/admin/rotate-api-key` — regenere l'API key tenant et invalide l'ancienne (utilise par bouton Hub "Regenerer API key" dans page integrations Notifuse)
- [ ] `GET /api/veridian/admin/list-tenants` — liste tous les tenants Veridian-managed avec status (paginated, pour admin Hub unifie)
- [ ] `POST /api/veridian/admin/inject-template` — push un template MJML Veridian default dans un workspace specifique (pour onboarding nouveaux tenants)

### CVE / dette technique upstream

- [ ] **Bumper Go 1.25.4 → 1.25.5+ ou 1.26** dans Dockerfile upstream :
  19 vulns Go stdlib detectees par govulncheck (crypto/x509, net, html/template,
  net/http, etc.) — a inclure au prochain rebase tag upstream
- [ ] **Console frontend npm audit fix** : 1 HIGH `liquidjs` DoS, 1 moderate `postcss` XSS — fix dispo via `npm audit fix` cote upstream
- [ ] **Notification center** : 1 moderate, non bloquant
- [ ] **Pas de version `beta` Notifuse upstream en prod** — actuellement on suit les tags stables, OK

### Robustesse e2e CI (improvements continus)

- [ ] **Reduce e2e cleanup spam** : actuellement le workflow boucle sur ~25 prefixes connus. Mieux : exposer `GET /api/veridian/admin/list-tenants?prefix_filter=test` puis wipe en 1 seul appel
- [ ] **Setup wizard Notifuse auto-skip** au premier boot : actuellement la 1ere fois qu'on deploie un container fresh il faut passer le wizard manuellement (ROOT_EMAIL, SMTP). Patch upstream pour skip si toutes les env vars sont set
- [ ] **DB connection pool monitoring** : log la metric `connection_count` dans logs structures pour pouvoir alerter avant saturation
- [ ] **Cleanup hook Playwright** dans `globalTeardown` : appelle wipe-test-tenants avec le prefix du run (`__test_${RUN_ID}_*`) au lieu de cumuler entre runs

## ⚠️ Freemium effectivement INFINI actuellement (pas un bug — decision Robert)

> Le middleware paywall verifie `emails_sent_this_month >= monthly_email_quota`,
> mais le compteur n'est JAMAIS incremente (le hook `IncrementEmailsSent` n'est
> pas branche dans le hot path d'envoi). En pratique aujourd'hui : un tenant
> free peut envoyer 1 million d'emails/mois sans etre bloque.
>
> **Decision Robert (2026-05-08)** : ne PAS fix ca isolement. On l'integre dans
> le chantier complet P1.7 SaaS Lifecycle (cf section ci-dessous) pour livrer
> une experience coherente :
> - Compteur branche
> - Reset mensuel paresseux
> - UI tenant pour voir son quota / upgrade
> - Stripe connecte pour autoriser l'upgrade
> - Soft-delete UI lisible
> - Lifetime override Hub-driven
>
> Avant ca : tous les tenants free profitent d'un quota illimite de fait.
> Ce n'est pas une regression : aujourd'hui en prod (image upstream v27)
> il n'y a meme pas le concept de plan/quota cote Notifuse, donc personne
> n'est bloque non plus.

### Backlog Notifuse-specific (long terme)

#### 🎯 Decision produit : pas de cron crade, gestion d'etat tenant lisible

> Robert ne veut PAS d'un cron silencieux qui purge les soft-deleted. A la place :
> le tenant voit dans sa console **pourquoi** il est bloque + **comment** se
> debloquer. Plus professionnel, transparent, oblige le user a agir.

- [ ] **Page `/console/billing` enrichie cote frontend Notifuse** (patch Veridian
  React, dans `console/src/`) :
  - [ ] Banner persistant en haut de toutes les pages si `status != active` :
    - `suspended` → "⚠️ Compte suspendu — paiement en retard. [Regulariser]"
    - `deleted` → "🗑️ Workspace en cours de suppression. Reactiver avant <date>. [Reactiver]"
    - `quota_exceeded` → "📧 Quota mensuel atteint (10000/10000). [Upgrade]"
  - [ ] Page `/console/billing` complete : plan actuel, quota mois en cours
    avec barre de progression, historique consommation (graph), bouton "Changer de plan"
  - [ ] Bouton "Reactiver maintenant" (si suspended) → POST API qui notifie
    le Hub via webhook → Hub gere le retour Stripe (ou override gratuit pour client site vitrine)
  - [ ] Bouton "Annuler la suppression" (si deleted_at != null) → POST API
    qui clear `deleted_at` (uniquement valable dans la fenetre 30j)
  - [ ] Footer permanent : status + quota + lien doc/support

- [ ] **Hard delete = decision explicite, pas un cron** :
  - [ ] Le user click "Supprimer definitivement maintenant" dans `/console/billing/danger-zone`
    → modal confirmation → `DELETE /api/tenants/:id?hard=true` → wipe immediat
  - [ ] Si le user fait rien apres soft-delete : le tenant **reste** en
    `status=deleted` indefiniment, banner en haut + emails desactives. Pas
    de purge silencieuse derriere son dos. Robert peut wipe manuellement
    via skill admin si besoin de recuperer la place.
  - [ ] **Reactivation possible a tout moment** : tant que les donnees existent,
    `POST /api/tenants/resume` (depuis Hub) ou bouton frontend → clear `deleted_at`

- [ ] **Reset compteurs mensuels** : pas un cron Dokploy mais un check
  paresseux au moment du `IncrementEmailsSent` :
  - Si `last_reset_at` est dans un mois precedent → ResetMonthlyCounters AVANT increment
  - Avantage : pas de cron a maintenir, garanti exact (jamais de fenetre ou compteur pas reset)
  - Code dans `internal/service/veridian_service.go` ou middleware paywall

#### 🎯 Decision produit : plans pilotes par le Hub, PAS par Stripe direct

> Robert vend des sites vitrines + offre Notifuse a vie aux acheteurs.
> Le plan Notifuse ne peut PAS etre derive automatiquement de la subscription
> Stripe d'un user — il y a des cas business (cadeau commercial, partenariat,
> tenant interne Robert) ou le plan est decide par le Hub.

**Architecture cible** :
- Stripe = source de verite des paiements **users** (subscription standard)
- Hub = source de verite des **plans tenants** (decision business — peut override Stripe)
- Notifuse = boite noire qui applique ce que le Hub dit (`updatePlan`)

**Plans Notifuse** (a etendre dans `internal/domain/veridian.go` `PlanQuotas`) :
- `free` : 500 emails/mois (signup standard)
- `pro` : 10 000 emails/mois (subscription Stripe €29)
- `business` : 50 000 emails/mois (subscription Stripe €49)
- `enterprise` : -1 unlimited (custom deals)
- **`lifetime_site_vitrine` : 5 000 emails/mois** ← nouveau plan, offert a vie
  aux clients ayant achete un site vitrine chez Veridian
- **`lifetime_partner` : 10 000 emails/mois** ← partenaires/influenceurs
- **`internal` : unlimited** ← tenants internes Robert (test, demo, support)

- [ ] Etendre `PlanQuotas` map avec ces nouveaux plans
- [ ] Etendre `WipeTestTenantsInput.SafetyClientPrefixes` defaut pour inclure
  les tenants `lifetime_*` et `internal_*` (eviter wipe accidentel)
- [ ] Cote Hub : table `tenants` doit avoir une colonne `plan_source` (`stripe` /
  `manual` / `lifetime_site_vitrine` / `lifetime_partner` / `internal`)
- [ ] Cote Hub : interface admin `/admin/tenants/[id]/billing` permet a Robert
  de changer le plan d'un tenant en 1 click + raison (`gift_site_vitrine`,
  `partner_program`, etc.)
- [ ] **Webhook Stripe NE doit PAS override** un tenant `plan_source != stripe` :
  si `customer.subscription.deleted` arrive et le tenant a `plan_source=lifetime_*`,
  on log mais on NE downgrade pas
- [ ] Audit log Hub : tracer chaque changement de plan avec qui (user / Robert / Stripe webhook) + raison

**Cas d'usage concret** :
1. Client X achete site vitrine + 1 an de support chez Robert
2. Robert provisione site web (skill `/create-site`) + tenant Notifuse via Hub
3. Hub set `plan_source=lifetime_site_vitrine` + `plan=lifetime_site_vitrine`
4. NotifuseClient.updatePlan("lifetime_site_vitrine") → 5000 emails/mois actifs
5. Plus tard : si Stripe webhook payment_failed arrive (Stripe lie a un autre service de Robert), le Hub voit `plan_source != stripe` et **n'override pas** le plan Notifuse

#### Chantier groupe P1.7 — SaaS Lifecycle Complet (Stripe + paywall + lifetime + soft-delete UI)

> A traiter en UN SEUL sprint pour livrer une experience coherente. Tout
> est inutile sans le reste : compteur quota sans UI quota = user bloque
> sans comprendre, Stripe webhook sans plan_source = override accidents,
> soft-delete sans UI = purge silencieuse pas pro.
>
> **Pre-requis bloquants** (a faire avant) :
> - Bump prod fork Notifuse (sinon on patche dans le vide)
> - Migration table `tenants` Hub avec `notifuse_plan_source` (cf TODO Hub P1.6)

##### Cote fork Notifuse (Go)

- [ ] **Brancher hook `IncrementEmailsSent`** : dans `internal/service/email_service.go`,
  apres un envoi reussi (status=sent ou delivered), call async
  `planRepo.IncrementEmailsSent(workspace_id, 1)`. Pas de blocking : si DB
  down, log warn, on n'echoue pas l'envoi.
- [ ] **Reset paresseux mensuel** : dans le middleware paywall, AVANT
  `IsBlocked()`, check si `last_reset_at < date_trunc('month', now())`.
  Si oui → `planRepo.ResetMonthlyCounters(workspaceID)` puis re-fetch.
  Pas de cron Dokploy.
- [ ] **Frontend Veridian patch** : page `/console/billing` enrichie (cf
  decision produit "pas de cron crade" plus haut)
- [ ] **Endpoint admin `rotate-api-key`** (demande Hub UI cf P1.6)

##### Cote Hub (TypeScript)

- [ ] Migration table `tenants` : `notifuse_plan_source`,
  `notifuse_plan_set_by_user_id`, `notifuse_plan_set_reason`,
  `notifuse_plan_set_at` (cf TODO Hub P1.6)
- [ ] Webhooks Stripe brances : `customer.subscription.updated` →
  `NotifuseClient.updatePlan` SI `plan_source=stripe` uniquement
- [ ] Webhook Stripe `invoice.payment_failed` → `suspendWorkspace`
- [ ] Webhook Stripe `invoice.payment_succeeded` apres suspend → `resumeWorkspace`
- [ ] Webhook Stripe `customer.subscription.deleted` → `softDeleteWorkspace`
- [ ] Page `/admin/tenants/[id]/billing` admin pour override plan manuel
- [ ] Skill `/notifuse-grant-lifetime` pour automatiser apres `/create-site`

##### Plans a creer

- [ ] Etendre `PlanQuotas` Go avec `lifetime_site_vitrine` (5000),
  `lifetime_partner` (10000), `internal` (-1)
- [ ] Aligne avec products Stripe (lookup_keys correspondants si Stripe-paid,
  ignore si lifetime/internal)

##### Validation finale

- [ ] E2E Playwright : signup user → freemium 500 → 500 envois → 501e = 402 →
  upgrade Stripe pro → quota 10000 → 1000 envois OK → admin override lifetime →
  quota 5000 nouveau plan → soft-delete via cancel Stripe → banner visible →
  reactivation possible
- [ ] Tester avec **vrai client lifetime** : Robert finit un site vitrine,
  /create-site provisione + skill /notifuse-grant-lifetime → client recoit
  email avec auto_login_url → arrive console → voit "Plan : Site Vitrine
  Lifetime — 5000 emails/mois — Offert par Veridian"

#### Reste du backlog (independant de P1.7)

- [ ] **Metrics d'envoi par tenant exposees au Hub** : bounce rate, open rate, click rate via API HMAC (lecture des tables existantes Notifuse upstream)
- [ ] **Templates emails Veridian par defaut** : logo Veridian, couleurs charte, footer legal RGPD — injectes au provision via `inject-template` endpoint
- [ ] **Magic link cross-app** : un magic link Hub-side qui logge en meme temps Hub + Notifuse + Twenty + Prospection (vrai SSO Veridian, chantier douloureux dans TODO-LIVE)
- [ ] **Audit log Notifuse** : event log persistent table pour suspend/resume/delete/plan_changed (defense en profondeur si Hub webhook fail)

## Bugs connus

### Resolus pendant le sprint
- ~~`workspace already exists` sur 2eme provision~~ → fix : ctx root pour lookup idempotence (commit `fix(veridian): idempotence Provision utilise ctx root`)
- ~~`create api key: this user already exists`~~ → fix : prefix unique `veridian-api-<tenant_id>`
- ~~Magic link prod sans code~~ → fix : `GenerateMagicCodeForVeridian` retourne le code en clair (privileged path)
- ~~Frontend Notifuse ne consume pas `?code=` URL~~ → solution : auto_login_url HMAC avec page HTML qui localStorage.setItem(jwt) + redirect

### Connus residuels (non bloquants)
- **Race CreateDatabase** : provisions concurrents (>5 simultanees) peuvent avoir des 5xx transitoires. Le NotifuseClient TS Hub retry 2x sur 5xx. Tolerance e2e : 80% successes au moins. Workaround long terme : mutex Go cote service Veridian sur Provision (1 seul a la fois par tenant_id).
- **Cache paywall TTL 60s** : window de 60s apres suspend pendant lequel des emails peuvent encore partir. Acceptable pour MVP. Long terme : push invalidation cache (channel Go) quand suspend appele.
- **CVE Go stdlib v1.25.4** (cf section CVE)

## Decisions techniques

### Architecture du fork
- **Branche dediee `veridian`** : minimise le diff avec upstream pour faciliter les merges. Tous nos changements dans cette branche, jamais sur `main`.
- **Garder la CI upstream + ajouter** : leurs tests valident que nos modifs ne cassent pas le coeur.
- **Boite noire API-only** : aucun appel direct DB Notifuse depuis l'exterieur. Hub parle HTTP, point. Cleanup CI passe par `/api/veridian/admin/wipe-test-tenants` (pas de SQL).

### Securite
- **HMAC-SHA256** pour pilotage Hub → Notifuse (`X-Veridian-Hub-Signature`)
- **HMAC-SHA256** pour events Notifuse → Hub (`X-Veridian-Notifuse-Signature`)
- **Token auto_login self-contained** : `base64(payload).hex(hmac)` avec TTL 60s
- **Drift max 5 min** sur les requetes HMAC pour anti-replay
- **Safety client prefixes** : refus de wipe si tenant_id matche un prefix client reel meme avec HMAC valide

### Magic link self-contained (pas single-use)
- TTL 60s = "vie courte mais reutilisable dans la fenetre"
- Pas de tracking usage : evite race conditions entre Hub Generation et user click
- Securite = TTL court (perd l'URL = invalide en 60s)
- Frontend Notifuse upstream stocke l'auth dans **localStorage** (pas cookie), donc l'auto-login passe par une page HTML inline qui set localStorage puis redirect

### Plans et quotas
Hardcoded dans `internal/domain/veridian.go` `PlanQuotas` :
- free : 500 emails/mois
- pro : 10 000
- business : 50 000
- enterprise : -1 (unlimited)

Override possible via env var `VERIDIAN_QUOTA_OVERRIDE` (a coder si besoin).

## Workflow rebase upstream

Voir `notifuse/MERGING-UPSTREAM.md` du monorepo. Procedure documentee :
1. `git fetch upstream --tags`
2. Lire CHANGELOG entre tag actuel et nouveau tag
3. Tester l'image upstream brute en staging (avant rebase)
4. `git checkout veridian && git rebase v<NEW>`
5. Resoudre conflits (zones touchees : `internal/app/app.go`, schema, migrations)
6. `make test` upstream + nos tests Veridian
7. Update `.upstream-version` dans monorepo
8. `git push --force-with-lease origin veridian`
9. CI rebuild image GHCR + deploy staging

Conflits courants documentes au fil de l'eau dans MERGING-UPSTREAM.md.

## Notes agents (chantiers en cours)

_(sprint P1.3 termine — pas de chantier actif cote fork applicatif)_

### Follow-ups identifies 2026-05-13 (extraction GitOps)

- [ ] **Headers securite** : `obs check security` rapporte 6 headers manquants
  sur `notifuse.app.veridian.site` (HSTS, x-frame-options, x-content-type-options,
  referrer-policy, +2). Config Notifuse upstream, **pas un probleme de la
  migration GitOps**. A patcher cote fork `christ-roy/notifuse-veridian` (PR sur
  branche `veridian` qui ajoute middleware headers ou Traefik labels override).
- [ ] **CORS `*` + Credentials: true** : aussi `obs check security` CRIT, meme
  source (Notifuse upstream). Memes notes.
- [ ] **Cleanup monorepo** : a partir du 2026-05-20 (7 jours apres extraction
  validee 2026-05-13), supprimer du monorepo `veridian-platform` :
  - `infra/services/notifuse/`
  - `runbooks/services/notifuse/`
  - `todo/apps/notifuse/`
  - `notifuse/` (scaffold)
  - `.github/workflows/notifuse-ci.yml`
  - `.github/workflows/notifuse-security-cron.yml`
  - Ouvrir PR `chore/monorepo-remove-notifuse` sur `veridian-platform` similaire
    a `chore/monorepo-remove-prospection` deja en cours
- [ ] **Bug `_trivy-image.yml`** (cote monorepo `veridian-platform`) : tag
  `aquasecurity/trivy-action@0.36.0` invalide (le bon = `v0.36.0` avec prefixe).
  Casse `hub-security-cron`, `prospection-security-cve`, et l'ancien
  `notifuse-security-cron` du monorepo. Notre nouveau cron sur `notifuse-deploy`
  utilise directement `@v0.36.0` (correct) donc on est OK ici. Cross-app issue,
  hors scope deploy notifuse.
- [ ] **Tag `:latest` upstream** : Dokploy ENV pourrait avoir
  `NOTIFUSE_IMAGE_TAG=saas-v1.0.3` (releases SemVer Veridian). Confirmer
  qu'aucune drift par rapport au compose Git (envoyer un push qui modifie
  juste un commentaire → si Dokploy redeploy avec un tag different, c'est
  qu'un override ENV est en place).

## Recently shipped

- **2026-05-13** : **Extraction Notifuse hors monorepo** vers repo standalone
  `Christ-Roy/notifuse-deploy`. Stack Dokploy bascule de `sourceType: raw`
  (compose colle dans l'UI) vers `sourceType: git` pointant sur le nouveau repo.
  Webhook GitHub migre, fallback monorepo conserve 7 jours. Validation prod
  fonctionnelle complete (scheduler 4-5 tasks/sec, 61 workspaces, 0 erreur).
  Decouvertes documentees : (1) manifest digest registry vs `.Image` local,
  (2) piege Dokploy Domains (injection labels Traefik en parallele = dual-router,
  fix via `domain.delete` API), (3) tag `aquasecurity/trivy-action@v0.36.0`
  necessite le prefixe `v`. Sequence PR monorepo : #89 #91 #95 #96 #97 #98
  #100 ; PR notifuse-deploy : #1.
- **2026-05-08** : Sprint P1.3 Notifuse-saasification complet. Fork v30.1, 7 endpoints HMAC + auto-login + admin wipe, paywall middleware, webhook emitter, 49 tests Go, e2e CI green, pipeline staging validee. Voir `session_2026-05-08_notifuse_saasification.md` memory.
- **2026-05-08** : Endpoint admin `/api/veridian/admin/wipe-test-tenants` (HMAC) pour cleanup CI propre — remplace les SQL directs DROP DATABASE par appels API + safety_client_prefixes.
- **2026-05-08** : Auto-login URL self-contained (`/veridian/auto-login?token=...`) → click bouton Hub = user logge direct dans console Notifuse comme owner, sans saisie code email. Page HTML inline localStorage + redirect SPA.
