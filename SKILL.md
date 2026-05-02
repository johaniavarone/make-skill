---
name: make-expert
description: Expert Make.com (Integromat) de niveau senior — référence complète pour concevoir, construire, auditer et debugger des scénarios Make. Couvre l'API Make v2, la structure JSON des blueprints, le langage IML (expressions, fonctions built-in, fonctions IML custom), les modules built-in (Router, Iterator, Aggregator, Tools, HTTP, Webhook), l'error handling avancé (Break/Resume/Rollback/Commit/Ignore + exponential backoff), les data stores, les webhooks, le serveur MCP Make, l'optimisation des opérations, et les patterns de migration Make→n8n. Utiliser cette compétence dès que l'utilisateur mentionne Make, Integromat, un scénario, un blueprint, un module Make, l'API Make, le MCP Make, IML, ou décrit une tâche d'automatisation qu'on construirait avec des modules connectés (déclencheur → action → action). Déclencher aussi sur les questions de debug ("opérations qui explosent", "iterator qui rend des bundles vides", "scénario qui boucle", "webhook qui ne déclenche pas"), les questions de design ("comment faire idempotent", "comment retry intelligemment", "comment limiter les ops"), et les demandes de refactoring de scénarios existants. À utiliser AVANT de répondre depuis la mémoire seule.
---

# Make.com Expert Skill

Référence complète Make.com (anciennement Integromat) pour concevoir, construire, auditer et debugger des scénarios production-ready. Distillée depuis la documentation officielle (developers.make.com, help.make.com, apps.make.com), l'API v2, et les patterns de production réels.

**À jour : Mai 2026.** API v2 (current). Plateforme Make Cloud + White Label.

---

## Comment utiliser cette skill

Le savoir détaillé est dans `references/`. Chaque fichier est self-contained et couvre un domaine. Ouvrir uniquement les fichiers pertinents pour la question — pas besoin de tout charger à chaque fois.

| Fichier | Quand le lire |
|---------|---------------|
| `references/01-api-make.md` | Tout appel API Make : auth, pagination, scenarios CRUD, blueprints push/pull, hooks, connections, data stores, executions, folders, organisations/teams, custom variables |
| `references/02-blueprint-json.md` | Structure JSON d'un blueprint, IDs de modules, mappers, metadata, encodage du blueprint comme string, références entre modules |
| `references/03-iml-expressions.md` | Langage IML, syntaxe `{{...}}`, opérateurs, conditionnelles `if`/`ifempty`/`switch`, accès aux collections/arrays, indexation 1-based, `get()` pour clés dynamiques |
| `references/04-functions-reference.md` | Référence complète des fonctions built-in : General (`if`, `switch`, `get`, `ifempty`), Math, Text/Binary, Date/Time, Array. Avec tokens de parsing/formatting |
| `references/05-builtin-modules.md` | Modules Tools, Flow Control (Router, Iterator, Array Aggregator, Text/Numeric Aggregators), HTTP, JSON, Webhooks, Sleep, Set/Get Variable, Compose String, Repeater, Converger |
| `references/06-error-handling.md` | Les 5 directives (Break, Resume, Rollback, Commit, Ignore), exponential backoff, Incomplete Executions, sequential processing, Throw module, retry strategies |
| `references/07-webhooks-datastores.md` | Webhooks (création, queue, IP allowlist, payload validation), Data Stores (création, indexes, TTL, patterns dedup/cache/circuit breaker) |
| `references/08-mcp-server.md` | Make MCP Server : OAuth vs MCP token, transports (Streamable HTTP, SSE, stateless), scenarios as tools, scopes, timeouts, retrieval async post-timeout |
| `references/09-best-practices.md` | Idempotence, optimisation opérations, naming conventions, design patterns, sécurité (secret-check sur webhooks publics), monitoring, environnements (dev/staging/prod) |
| `references/10-migration-n8n.md` | Mapping modules Make → n8n, traduction des expressions, gotchas de migration, ce qui n'a pas d'équivalent direct |
| `references/11-custom-apps.md` | Custom Apps (Apps Editor) : Base, Connections, Modules, RPCs, IML functions custom, mappable parameters dynamiques, App Review |

Pour les questions multi-domaines (ex. "scénario qui pull un Airtable, route selon un statut, génère un Doc, et notifie Slack avec retry intelligent"), lire plusieurs fichiers : `02-blueprint-json.md` + `05-builtin-modules.md` + `06-error-handling.md`.

---

## Principes opérationnels (cross-cutting)

Ces règles s'appliquent partout et doivent guider chaque recommandation.

### 1. Webhook trigger > polling chaque fois que possible

Les triggers polling (Watch Records, Watch Files, Watch Rows planifiés) consomment des opérations à chaque cycle même si rien n'a changé. Quand l'app source supporte les webhooks (Softr "Send to Make", Airtable Automations → webhook, Stripe events, etc.), recommander la variante webhook. C'est moins cher, plus rapide, et évite les confusions de backfill.

Quand le polling est inévitable, étirer l'intervalle au maximum tolérable et utiliser une `Limit` serrée pour éviter le replay d'historique au premier run.

### 2. Concevoir pour l'idempotence

Les scénarios Make rejouent sur retries et replays d'historique. Toute action side-effect (Create, Send, Add) doit pouvoir être exécutée deux fois sans casse. Patterns standards :

- **Search-then-branch** : search la destination par clé unique → Router → une branche update, l'autre create.
- **Data Store comme ledger de dedup** : key = source_id, TTL = window de replay accepté. Avant l'action, lire le store ; après l'action, écrire.
- **Stamp synced-at** : marquer la source avec un timestamp de sync pour que le poll suivant skip.
- **Idempotency key dans l'API cible** : Stripe, GitHub, etc. acceptent un header `Idempotency-Key`. L'utiliser systématiquement.

### 3. Iterator + Aggregator vont par paire

Pour opérer sur chaque item d'un array : `source-array → Iterator → modules per-item → Array Aggregator (Source Module = l'Iterator)`. Sauter l'aggregator → bug "seul le dernier bundle est passé". L'aggregator regroupe les bundles que l'iterator a éclatés pour que la suite reçoive un seul bundle agrégé.

### 4. Filtrer AVANT l'iterator, jamais après

Une `Limit` ou un Filter sur le module source coûte ~0 ops. Le même filter après l'Iterator coûte 1 op par item filtré. Sur 1000 items dont 950 seront rejetés, c'est 950 ops gaspillées.

### 5. Error handlers ne sont pas optionnels en prod

Tout module qui appelle une API externe doit avoir un error handler avec une directive explicite :
- **Break** + retry pour les erreurs transitoires (429, 502, 503, timeouts)
- **Resume** avec bundle de fallback pour les enrichissements optionnels
- **Ignore** pour les calls best-effort sans impact downstream
- **Rollback** pour les opérations transactionnelles multi-étapes
- **Commit** pour les arrêts business-logic (ex. "lead pas qualifié")

Le comportement par défaut (faire crasher le scénario entier) est rarement ce qu'on veut.

### 6. Surveillance du budget d'opérations

Chaque appel de module = 1 op. L'Iterator coûte 1 op + 1 op par item downstream. Une feuille de 100 lignes → Iterator → Update = 1 + 100 = **101 ops**, pas 2. Toujours estimer le coût au moment du design, pas après le premier dépassement de quota.

### 7. Pull-before-edit pour les blueprints via API

Avant de modifier un blueprint via l'API, **toujours** fetch la version courante. Le scénario peut avoir été édité dans l'UI entre-temps. Push aveugle = perte d'édits.

### 8. Le blueprint doit être une string JSON-encodée pour le push API

La gotcha API la plus courante : envoyer le blueprint comme objet JSON brut au lieu d'une string échappée. Toujours utiliser le pattern `jq -n --arg bp "$blueprint" '{"blueprint": $bp}'`.

### 9. Activation séparée du push

Pousser un blueprint ne l'active pas. Après push → `POST /scenarios/{id}/start` pour activer.

### 10. Zone prefix obligatoire

`api.make.com` sans zone (`eu1`, `eu2`, `us1`, `us2`) renvoie 404. Toujours utiliser le hostname zoné complet, vérifier la zone du compte dans Profile → Account.

---

## Output style pour les specs de scénario

Quand l'utilisateur demande de **concevoir** un scénario (pas juste répondre à une question), structurer la réponse pour qu'elle soit implémentable directement :

1. **Trigger** — module + connection + paramètres clés (ex. "Webhook Custom, secret check via Filter")
2. **Modules en ordre** — numérotés ; pour chacun : nom + mappings clés + filter éventuel qui suit
3. **Error handling** — quels modules ont besoin d'un handler et quelle directive
4. **Scheduling / activation** — intervalle ou URL webhook
5. **Estimation opérations** — ops par exécution (cas nominal et cas dégradé)
6. **Edge cases à tester** — au moins 2-3 (vide, dupliqué, payload mal formé, rate limit, etc.)
7. **Blueprint JSON** (si demandé explicitement) — module-by-module avec IDs

Garder ce format sauf demande explicite de prose.

---

## Style d'écriture (préférences user)

- **Pas de bullet lists dans les articles ou contenus longs** : paragraphes développés avec exemples concrets et sources fiables.
- **Pour les specs techniques (ce SKILL et docs internes)** : structure claire avec titres, mais pas de listes de surface ; chaque point développé.
- **Vocabulaire** : marketing et produit, pas méthodologie agile. Faits, direct, orienté résultat.
- **Pas de ton conversationnel ou émotionnel** : factuel, posé, expert.
