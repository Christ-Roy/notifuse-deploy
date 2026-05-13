# Notifuse — UI Review queue (solo polish)

> Notifuse a son propre UI upstream (Go + templates). Les reviews concernent nos
> customisations Veridian (dashboard tenant, templates mails par defaut, paywall UI).
>
> **Workflow** :
> 1. Cloner le fork `Christ-Roy/notifuse-veridian`
> 2. `make dev` dans le dossier notifuse
> 3. Polish les ecrans Veridian-specific
> 4. Commit sur la branche `veridian`

---

## A reviewer

_(vide — fork pas encore fait)_

**Format entree** :

```markdown
### [YYYY-MM-DD] Ecran/template
- **Contexte** : sprint P1.3, livre par teammate <nom>
- **URL dev** : http://localhost:8080/path
- **Fichiers** : `notifuse/internal/ui/...`, `notifuse/templates/...`
- **A polish** :
  - [ ] Branding Veridian (logo, couleurs)
  - [ ] Responsive
- **Notes agent** : ...
```

---

## Reviewed (archive)

_(rien encore reviewe)_
