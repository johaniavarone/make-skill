# Webhooks & Data Stores

Les deux primitives "infra" de Make. Les webhooks reçoivent les triggers des systèmes externes ; les data stores fournissent un stockage persistant clé-valeur natif.

---

## 1. Webhooks

### Création UI

Dans le scénario : ajouter le module **Webhooks → Custom webhook** → cliquer **Add** → nommer le hook → Make génère une URL unique.

### Création API

```bash
curl -X POST "$BASE/hooks" \
  -H "Authorization: Token $MAKE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Stripe events",
    "teamId": 12345,
    "typeName": "gateway-webhook"
  }'
```

`typeName` : `gateway-webhook` (HTTP) ou `gateway-mailhook` (email).

### URL produite

Format : `https://hook.<zone>.make.com/<HOOK_TOKEN>`. `HOOK_TOKEN` est secret — le révéler équivaut à donner accès au trigger. Si compromis, **delete le hook** (l'URL devient invalide) et créer un nouveau.

### Méthodes HTTP supportées

POST, GET, PUT, DELETE, PATCH. Make accepte tout. Les paramètres :
- **Query string** : disponibles en `{{hook.qs.X}}` ou directement en propriétés racine selon learn.
- **Body JSON** : parsé automatiquement, accessible via `{{hook.X}}`.
- **Body form-urlencoded** : parsé en object.
- **Multipart** : fichiers accessibles comme attachments.
- **Headers** : `{{hook.headers.X}}`.

### Learn data structure

Quand on crée un webhook, sa data structure est inconnue. Pour activer l'autocomplete des champs, **lancer le mode "Determine data structure"** dans l'UI puis envoyer un payload de test. Make analyse et fige le schema.

Si le payload de prod a plus de champs que celui de learn, ils restent accessibles via `{{hook.X}}` mais pas dans l'UI dropdown.

Pour relearn : Webhook UI → Redetermine. Important après changement majeur du système source.

### Synchronous response

Activer **Get response** dans les settings du hook → ajouter un module **Webhook Response** au bout du scénario. Permet de renvoyer une réponse au caller (200 + body custom).

Cas d'usage : webhook intelligent qui valide un payload et renvoie OK/KO synchroniquement (Stripe attend un 200, sinon retry).

**Limite** : le scénario doit terminer en <40s pour un timeout par défaut. Les opérations longues doivent retourner 202 Accepted immédiatement et continuer en async.

### Validation de signature HMAC (Stripe, GitHub)

Les webhooks Stripe et GitHub envoient une signature dans un header pour authentifier l'origine.

**Stripe** (`Stripe-Signature` header) :

```
1. Récupérer le secret du webhook endpoint dans Stripe dashboard
2. Stocker dans Make → Custom Variable: STRIPE_WEBHOOK_SECRET
3. Module 1: Webhook custom (raw body activé!)
4. Module 2: Set Variable
   - timestamp = first item of split(headers.stripe-signature; ",")
   - signature = second item
5. Module 3: Compose String pour signed_payload
   - {{timestamp}}.{{rawBody}}
6. Module 4: Compute HMAC SHA256 with secret
   (HTTP module vers un service de hash, ou fonction custom IML si Enterprise)
7. Module 5: Filter : computed_hmac == signature
8. Suite du scénario uniquement si filter passe
```

**Important** : par défaut Make parse le body. Pour vérifier la signature, on a besoin du body **brut** non-modifié. Activer **Raw body** dans le webhook settings → accessible via `{{hook.rawBody}}`.

### Webhooks queue & rate

Quand le scénario est inactif ou saturé, Make queue les payloads (jusqu'à un certain volume défini par le plan). Si la queue déborde → 503 au caller.

**Pour vider une queue** :
```bash
curl -X POST "$BASE/hooks/$HOOK_ID/ping" \
  -H "Authorization: Token $TOKEN"
```

Effet : drop tous les payloads en attente.

**Pour augmenter le throughput** :
- Activer **Process all data** dans le scénario settings (consume plusieurs payloads par execution si dispo).
- Augmenter `maxResults` du module Webhook (1 → 100 par exemple).
- Mettre le scénario en mode `sequential: false` si l'ordre n'importe pas.

### IP allowlist

Make publie ses ranges IP de sortie pour les calls HTTP outbound. Pour les webhooks **inbound**, n'importe quel client peut hit l'URL si tokenné. Pour restreindre :

- **Plan Enterprise / White Label** : option d'IP allowlist dans le scénario.
- **Sinon** : ajouter en module 1 un Filter qui valide `X-Forwarded-For` ou `CF-Connecting-IP` contre une liste autorisée.

### Mailhook

```bash
# Créer
curl -X POST "$BASE/hooks" \
  -H "Authorization: Token $TOKEN" \
  -d '{"name":"Support emails","teamId":12345,"typeName":"gateway-mailhook"}'
```

URL générée : `<random>@hook.<zone>.make.com`. Forwarder les emails reçus vers cette adresse depuis un alias domaine (Gmail, M365, etc.).

Output : `from`, `to`, `cc`, `subject`, `text`, `html`, `attachments[]`, `headers`.

---

## 2. Data Stores

### Concept

Storage clé-valeur natif Make, persistant entre exécutions. Quotas selon plan : 10 MB sur free, plusieurs GB en paid.

### Cas d'usage

- **Dedup ledger** : key = source ID, value = synced_at. Avant d'agir, vérifier que la key n'existe pas (ou date est ancienne).
- **Cache** : key = paramètre, value = résultat d'une API coûteuse. TTL via le champ expiresAt.
- **State machine** : key = entity ID, value = état courant (jsonifié).
- **Rate limit tracking** : key = service_name, value = count + reset_at. Incrémenter avant chaque call, reset périodiquement.
- **Circuit breaker flag** : key = service_x_down, value = true, expiresAt = now+10min.

### Création UI

**Tools → Data stores → Add data store** → nom + data structure (schéma JSON optionnel) + size (MB).

### Data structure

Définit le schema des values. Exemple :

```json
{
  "name": "Customer state",
  "spec": [
    {"name": "customer_id", "type": "text", "required": true},
    {"name": "synced_at", "type": "date"},
    {"name": "status", "type": "text"},
    {"name": "metadata", "type": "any"}
  ]
}
```

Sans data structure : value est libre (any JSON), mais pas indexée.

### Modules dans un scénario

| Module | ID interne | Action |
|--------|-----------|--------|
| Add a record | `datastore:Add` | Insert (failed si key existe déjà) |
| Update a record | `datastore:Update` | Update existing (failed si pas trouvé) |
| Add or update | `datastore:UpdateOrAdd` | Upsert |
| Get a record | `datastore:Get` | Read by key |
| Search records | `datastore:SearchRecords` | Filter par expression |
| Delete a record | `datastore:Delete` | Delete by key |
| Delete all records | `datastore:DeleteAll` | Truncate |
| Count records | `datastore:CountRecords` | Compteur global |
| Iterate over records | `datastore:GetRecords` | Loop sur tous |

### Pattern : dedup ledger

```
Trigger (Webhook ou Watch records)
  ↓
DataStore Get (key = {{1.source_id}})
  ↓
Filter: 1.exists = false  (record n'existe pas → on traite)
  ↓
[modules de traitement]
  ↓
DataStore Add (key = {{1.source_id}}, data = {synced_at: now})
```

Si scénario rejoue le même payload (replay history, retry) : le 2e passage trouve la key, filter bloque, pas de doublon créé.

### Pattern : cache 1h

```
Trigger
  ↓
DataStore Get (key = "expensive_query_result")
  ↓
Router :
  Route 1 (cache hit ET pas expiré) :
    Filter: 1.exists AND parseDate(1.data.cached_at) > addHours(now; -1)
    → utiliser 1.data.result
  Route 2 (miss ou expiré) :
    Filter: !1.exists OR parseDate(1.data.cached_at) <= addHours(now; -1)
    → API call
    → DataStore Add or Update key=cached_at=now, result=...
```

### Pattern : rate limit tracking

Pour un service avec quota 100 calls/heure :

```
Avant chaque call :
  DataStore Get (key = "openai_calls")
  Filter: ifempty(1.data.count; 0) < 100
  HTTP call to OpenAI
  DataStore Update (key = "openai_calls", count++ )

Reset périodique :
  Scénario séparé planifié toutes les heures :
  DataStore Update (key = "openai_calls", count=0, reset_at=now)
```

### Limites / gotchas

- **Concurrency** : pas de transactions natives. Si deux exécutions parallèles updatent la même key, last-write-wins. Pour évier les races, mettre le scénario en `sequential: true`.
- **Pas de TTL natif** : le champ `expiresAt` n'est pas honoré par Make automatiquement. Il faut un scénario nettoyeur planifié qui scan et delete les records expirés.
- **Quota** : 10 MB sur free. Surveiller via API `GET /data-stores/{id}/data?pg[limit]=1` et inspecter `meta.total`.
- **Performance** : Get par clé est O(1), mais Search records sans index est O(N) sur tout le store.
- **Pas de query langagière** : pas de "WHERE field > X". Filter dans Make sur le bundle output.

---

## 3. Custom Variables vs Data Stores

| Aspect | Custom Variable | Data Store |
|--------|----------------|------------|
| Scope | Team | Team |
| Mutabilité | Update via UI/API | CRUD via modules ou API |
| Cas d'usage | Config statique (URLs, secrets, IDs) | State runtime, dedup, cache |
| Volume | 1 valeur par variable | Plusieurs records par store |
| Lecture | `{{customVariable.X}}` | Via module `datastore:Get` |
| Coût en ops | 0 (intégré) | 1 op par accès |

**Règle** : valeur change rarement et utilisée partout → Custom Variable. Donnée changeante par scénario/par item → Data Store.

---

## 4. Patterns d'architecture combinant les deux

### Multi-environnement (dev/staging/prod)

```
Custom Variables :
  ENV = "prod"
  AIRTABLE_BASE_DEV = "appXXX"
  AIRTABLE_BASE_PROD = "appYYY"

Module Compose String :
  base_id = {{if(customVariable.ENV == "prod"; customVariable.AIRTABLE_BASE_PROD; customVariable.AIRTABLE_BASE_DEV)}}

Modules Airtable utilisent {{X.base_id}}
```

Switcher d'env : changer une seule variable.

### Feature flags

```
Custom Variable :
  FEATURE_NEW_PRICING = "true"

Filter dans le scénario :
  customVariable.FEATURE_NEW_PRICING == "true"
```

### Distributed lock simple

```
Avant section critique :
  DataStore Add (key = "lock_X", value = {acquired_at: now})
  ← échoue si déjà existant (Add est exclusif)

Si Add échoue → autre execution a le lock → Sleep + retry, ou skip

À la fin de la section :
  DataStore Delete (key = "lock_X")
```

Limite : pas de TTL → si le scénario crash en plein lock, la clé reste. Mitigation : scénario watchdog qui delete les locks > N minutes.
