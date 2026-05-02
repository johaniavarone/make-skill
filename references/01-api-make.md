# API Make v2 — Référence complète

L'API Make v2 expose la totalité des actions disponibles dans l'UI : scenarios, blueprints, hooks, connections, data stores, executions, organisations, teams, users, custom apps, custom variables, IML functions, RPCs.

## 1. Bases

### Hostname zoné (obligatoire)

```
BASE="https://eu1.make.com/api/v2"
```

Zones possibles : `eu1`, `eu2`, `us1`, `us2`. Sans zone, retour 404. Vérifier la zone dans **Profile → Account**.

### Authentification

Header obligatoire :

```
Authorization: Token YOUR_API_KEY
```

Récupération du token : **Profile → API/MCP → API tokens → Add token**. Choisir les scopes minimaux nécessaires (lecture seule pour audit, lecture+écriture pour push de blueprint, etc.).

### Pagination

Tous les endpoints `list` supportent :

```
?pg[limit]=100&pg[offset]=0&pg[sortBy]=name&pg[sortDir]=asc
```

`pg[limit]` max 1000 selon endpoint. Toujours paginer pour les comptes >100 entités.

### TeamId (souvent obligatoire)

La majorité des endpoints `list` exigent `?teamId=YOUR_TEAM_ID`. Récupérer via `GET /teams` ou dans l'URL de l'UI (`make.com/team/{id}/...`).

### Headers utiles

```
Content-Type: application/json
Accept: application/json
imt-admin: 1     # nécessaire pour certains endpoints admin (rare)
```

### Rate limits

Limites par team, fenêtre glissante. En cas de 429, lire le header `Retry-After`. Bulk operations : insérer un Sleep entre les calls (200-500ms en prod).

---

## 2. Scenarios

### List scenarios

```bash
curl "$BASE/scenarios?teamId=$TEAM_ID&pg[limit]=100" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Champs retournés : `id`, `name`, `isEnabled`, `scheduling` (`{type, interval}`), `folderId`, `lastEdit`, `usedPackages`, `nextExec`, `description`.

### Get scenario details

```bash
curl "$BASE/scenarios/$SCENARIO_ID" \
  -H "Authorization: Token $MAKE_API_KEY"
```

### Create scenario

```bash
curl -X POST "$BASE/scenarios" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "teamId": 12345,
    "name": "My Scenario",
    "blueprint": "{\"name\":\"My Scenario\",\"flow\":[...],\"metadata\":{...}}",
    "scheduling": {"type": "indefinitely", "interval": 900}
  }'
```

`scheduling.type` : `indefinitely` (toutes les `interval` secondes), `once` (une seule fois), `on-demand` (manuel uniquement). Webhook-triggered : peut être `on-demand`, l'execution part du payload entrant.

### Update scenario (name, scheduling, folderId)

```bash
curl -X PATCH "$BASE/scenarios/$SCENARIO_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"Renamed","scheduling":{"type":"indefinitely","interval":1800}}'
```

### Activate / Deactivate

```bash
# Activate
curl -X POST "$BASE/scenarios/$SCENARIO_ID/start" \
  -H "Authorization: Token $MAKE_API_KEY"

# Deactivate
curl -X POST "$BASE/scenarios/$SCENARIO_ID/stop" \
  -H "Authorization: Token $MAKE_API_KEY"
```

### Run scenario manually (on-demand)

```bash
curl -X POST "$BASE/scenarios/$SCENARIO_ID/run" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"data":{"key":"value"},"responsive":true}'
```

`responsive:true` attend la fin et renvoie le résultat (timeout limité). `false` lance et retourne immédiatement.

### Delete scenario

```bash
curl -X DELETE "$BASE/scenarios/$SCENARIO_ID?confirmed=true" \
  -H "Authorization: Token $MAKE_API_KEY"
```

`?confirmed=true` est obligatoire (garde-fou anti-suppression accidentelle).

### Clone scenario

```bash
curl -X POST "$BASE/scenarios/$SCENARIO_ID/clone" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"name":"Clone of My Scenario","teamId":12345}'
```

---

## 3. Blueprints

### Get blueprint (le JSON qui définit le graph de modules)

```bash
curl "$BASE/scenarios/$SCENARIO_ID/blueprint" \
  -H "Authorization: Token $MAKE_API_KEY" \
  | jq '.response.blueprint'
```

### Push blueprint (PATCH du scenario)

**Critique** : la valeur `blueprint` doit être une **string JSON-encodée**, pas un objet JSON brut. Erreur n°1 sur l'API.

```bash
blueprint=$(cat blueprint.json)
curl -X PATCH "$BASE/scenarios/$SCENARIO_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  --data "$(jq -n --arg bp "$blueprint" '{blueprint: $bp}')"
```

Push ne déclenche pas l'activation. Activer ensuite avec `/start`.

### Toujours pull-before-edit

Le scénario peut avoir été modifié dans l'UI. Pull → modifier localement → push.

```bash
# 1. Pull
curl "$BASE/scenarios/$SCENARIO_ID/blueprint" \
  -H "Authorization: Token $MAKE_API_KEY" \
  | jq '.response.blueprint' > current.json

# 2. Edit localement (jq, sed, script Python)

# 3. Push
blueprint=$(cat current.json)
curl -X PATCH "$BASE/scenarios/$SCENARIO_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  --data "$(jq -n --arg bp "$blueprint" '{blueprint: $bp}')"
```

---

## 4. Connections

```bash
# List
curl "$BASE/connections?teamId=$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get specific
curl "$BASE/connections/$CONNECTION_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Verify (test si OAuth toujours valide / API key encore bonne)
curl -X POST "$BASE/connections/$CONNECTION_ID/verify" \
  -H "Authorization: Token $MAKE_API_KEY"

# Test (équivalent du bouton "Test" dans l'UI)
curl -X POST "$BASE/connections/$CONNECTION_ID/test" \
  -H "Authorization: Token $MAKE_API_KEY"

# Update label / metadata
curl -X PATCH "$BASE/connections/$CONNECTION_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"name":"Production Stripe"}'

# Delete
curl -X DELETE "$BASE/connections/$CONNECTION_ID?confirmed=true" \
  -H "Authorization: Token $MAKE_API_KEY"
```

**Gotcha** : les connection IDs sont locaux à un compte/team. Importer un blueprint d'un compte vers un autre exige de re-mapper toutes les connection IDs dans le blueprint avant le push.

---

## 5. Hooks (Webhooks)

```bash
# List
curl "$BASE/hooks?teamId=$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get
curl "$BASE/hooks/$HOOK_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Create
curl -X POST "$BASE/hooks" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Stripe webhook",
    "teamId": 12345,
    "typeName": "gateway-webhook"
  }'

# Delete
curl -X DELETE "$BASE/hooks/$HOOK_ID?confirmed=true" \
  -H "Authorization: Token $MAKE_API_KEY"

# Enable / Disable
curl -X POST "$BASE/hooks/$HOOK_ID/enable" \
  -H "Authorization: Token $MAKE_API_KEY"
curl -X POST "$BASE/hooks/$HOOK_ID/disable" \
  -H "Authorization: Token $MAKE_API_KEY"

# Learn (rebuild data structure from incoming sample)
curl -X POST "$BASE/hooks/$HOOK_ID/learn-start" \
  -H "Authorization: Token $MAKE_API_KEY"
curl -X POST "$BASE/hooks/$HOOK_ID/learn-stop" \
  -H "Authorization: Token $MAKE_API_KEY"

# Ping (vider la queue de payloads non traités)
curl -X POST "$BASE/hooks/$HOOK_ID/ping" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Réponse de `Get` inclut `url` : c'est l'URL production à coller dans le système externe.

---

## 6. Data Stores

```bash
# List
curl "$BASE/data-stores?teamId=$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Create
curl -X POST "$BASE/data-stores" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{
    "name": "Dedup ledger",
    "teamId": 12345,
    "datastructureId": 67890,
    "maxSizeMB": 10
  }'

# List records (data)
curl "$BASE/data-stores/$DS_ID/data?pg[limit]=100" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get specific record
curl "$BASE/data-stores/$DS_ID/data/$RECORD_KEY" \
  -H "Authorization: Token $MAKE_API_KEY"

# Add record
curl -X POST "$BASE/data-stores/$DS_ID/data" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"key":"customer_123","data":{"synced_at":"2026-05-01"}}'

# Update record
curl -X PATCH "$BASE/data-stores/$DS_ID/data/$RECORD_KEY" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"data":{"synced_at":"2026-05-02"}}'

# Delete record
curl -X DELETE "$BASE/data-stores/$DS_ID/data/$RECORD_KEY" \
  -H "Authorization: Token $MAKE_API_KEY"

# Delete all records
curl -X POST "$BASE/data-stores/$DS_ID/data/delete-all" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Quotas : 10 MB par data store sur free, plus généreux sur paid plans.

---

## 7. Executions (history)

```bash
# List executions for a scenario
curl "$BASE/scenarios/$SCENARIO_ID/logs?pg[limit]=10" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get single execution
curl "$BASE/scenarios/$SCENARIO_ID/logs/$EXECUTION_ID" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Status values : `success`, `warning`, `error`, `incomplete`. `warning` = au moins un module a généré un warning mais le scénario s'est terminé. `incomplete` = stocké pour retry.

### Incomplete executions (DLQ)

```bash
# List
curl "$BASE/scenarios/$SCENARIO_ID/incomplete-executions?pg[limit]=50" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get detail
curl "$BASE/scenarios/$SCENARIO_ID/incomplete-executions/$IE_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Retry
curl -X POST "$BASE/scenarios/$SCENARIO_ID/incomplete-executions/$IE_ID/retry" \
  -H "Authorization: Token $MAKE_API_KEY"

# Delete (purge)
curl -X DELETE "$BASE/scenarios/$SCENARIO_ID/incomplete-executions/$IE_ID" \
  -H "Authorization: Token $MAKE_API_KEY"
```

---

## 8. Folders (organisation des scenarios)

```bash
# List folder tree
curl "$BASE/scenarios-foldertree?teamId=$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Create folder
curl -X POST "$BASE/scenarios-folders" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"name":"Client A","teamId":12345}'

# Update / Delete : PATCH / DELETE sur /scenarios-folders/$FOLDER_ID
```

---

## 9. Organizations & Teams

```bash
# Organizations the user belongs to
curl "$BASE/organizations" \
  -H "Authorization: Token $MAKE_API_KEY"

# Teams in an organization
curl "$BASE/teams?organizationId=$ORG_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Create team
curl -X POST "$BASE/teams" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"name":"Production","organizationId":99999}'

# Invite user to organization
curl -X POST "$BASE/users/invite" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{
    "email":"john@example.com",
    "organizationId":99999,
    "usersRoleId":1
  }'

# Update user team role
curl -X POST "$BASE/users/$USER_ID/user-team-roles/$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"usersRoleId":2}'
```

User roles (`usersRoleId`) : 1 = Owner, 2 = Admin, 3 = Member, 4 = Operator, 5 = Monitoring.

---

## 10. Custom variables

Variables d'organisation/team réutilisables dans n'importe quel scénario via le module **Tools → Get Variable**.

```bash
# List
curl "$BASE/custom-variables?teamId=$TEAM_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Create
curl -X POST "$BASE/custom-variables" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"name":"API_BASE_URL","value":"https://api.client.com/v2","teamId":12345}'

# Update value
curl -X PATCH "$BASE/custom-variables/$VAR_ID" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -d '{"value":"https://api.client.com/v3"}'

# Delete
curl -X DELETE "$BASE/custom-variables/$VAR_ID" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Cas d'usage : URLs d'API, identifiants de templates Google Docs, slugs de bases Airtable, secrets HMAC. Modifier la variable propage à tous les scénarios qui la lisent.

---

## 11. Usage data

```bash
# Operations consumed
curl "$BASE/organizations/$ORG_ID/usage" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Retour : ops consommées par scenario, par jour, par team. Utile pour identifier les scénarios "fuyards" (qui consomment dispoportionnellement).

---

## 12. Custom Apps API

```bash
# List custom apps for an organization
curl "$BASE/sdk/apps?organizationId=$ORG_ID" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get app detail
curl "$BASE/sdk/apps/$APP_NAME/$APP_VERSION" \
  -H "Authorization: Token $MAKE_API_KEY"

# List modules of an app
curl "$BASE/sdk/apps/$APP_NAME/$APP_VERSION/modules" \
  -H "Authorization: Token $MAKE_API_KEY"

# Get module
curl "$BASE/sdk/apps/$APP_NAME/$APP_VERSION/modules/$MODULE_NAME" \
  -H "Authorization: Token $MAKE_API_KEY"
```

Voir `references/11-custom-apps.md` pour la création/modification.

---

## 13. Erreurs courantes API et fixes

**404 `Resource not found`** : zone manquante dans BASE, ou ID inexistant. Vérifier `eu1`/`us1` et l'ID via list.

**400 `Invalid blueprint format`** : blueprint envoyé comme objet JSON brut au lieu de string. Utiliser `jq -n --arg bp "$bp" '{blueprint:$bp}'`.

**401 `Unauthorized`** : token expiré ou révoqué. Régénérer dans Profile → API.

**403 `Insufficient scopes`** : token créé sans le scope nécessaire. Régénérer avec scopes étendus (lecture+écriture sur le bon ressource).

**422 `Cannot delete scenario in use`** : scénario actif → POST `/stop` puis DELETE.

**429 `Rate limited`** : insérer Sleep entre calls bulk, lire `Retry-After`.

**500 sur push blueprint** : souvent un module avec `module` qui n'existe pas (typo) ou un `version` invalide. Vérifier que le module est listé dans `references/05-builtin-modules.md` ou existe dans l'app installée.

---

## 14. Documentation officielle

- Hub développeur : https://developers.make.com/api-documentation
- Apps : https://apps.make.com/make
- Help center API : https://help.make.com/api
- Status : https://status.make.com
