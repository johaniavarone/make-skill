# Modules built-in Make — Référence

Modules disponibles sans connection externe : Flow control, Tools, HTTP, JSON, Text parser, Webhooks, Sleep. Ce sont les briques de base de tout scénario.

---

## 1. Webhooks

### Custom webhook (trigger)

**Module** : `gateway:CustomWebHook` v1

Reçoit des payloads HTTP (POST/GET) depuis un système externe. Génère une URL unique.

**Paramètres clés** :
- `hook` : ID du hook (référence à un hook créé via API ou auto-généré).
- `maxResults` : combien de bundles traiter par exécution (défaut 1, peut monter à plusieurs si la queue est pleine).

**Comportement** :
- Si scénario actif et URL receive un payload → exécution lancée immédiatement.
- Si scénario inactif → payloads stockés dans la queue (sous une limite).
- Méthode HTTP supportée : POST, GET, PUT, DELETE, PATCH.
- Body JSON, form-encoded, ou multipart.

**Production URL** :
```
https://hook.eu1.make.com/{HOOK_TOKEN}
```

Récupérée via API `GET /hooks/{id}` dans `url` ou copiée depuis l'UI.

**Sécurité** : pas d'auth par défaut. Pour les webhooks publics, ajouter immédiatement après le trigger un module **Filter** qui vérifie un secret partagé :

```
Filter condition: 1.secret = customVariable.WEBHOOK_SECRET
```

Mieux : valider une signature HMAC (Stripe, GitHub) en module suivant.

### Custom webhook response

**Module** : `gateway:WebhookRespond` v1

Renvoie une réponse synchrone au système qui a appelé le webhook. Utile pour les webhooks "intelligents" qui doivent renvoyer 200 OK ou un payload calculé.

```json
{
  "id": 5,
  "module": "gateway:WebhookRespond",
  "version": 1,
  "mapper": {
    "status": 200,
    "body": "{\"ok\":true,\"id\":\"{{4.id}}\"}",
    "headers": [
      {"name": "Content-Type", "value": "application/json"}
    ]
  }
}
```

**Important** : cohabite avec le mode "synchronous webhook" sur le hook. À activer dans l'UI du hook (Settings → Get response).

### Mailhook

**Module** : `mail:CustomEmailTrigger` v1

Reçoit un email à une adresse `*@hook.eu1.make.com` générée par le hook. Parsing automatique : `from`, `subject`, `text`, `html`, `attachments[]`.

---

## 2. Flow control

### Basic Router

**Module** : `builtin:BasicRouter` v1

Éclate l'exécution en plusieurs branches (routes). Chaque route a ses propres modules.

```json
{
  "id": 2,
  "module": "builtin:BasicRouter",
  "version": 1,
  "mapper": null,
  "routes": [
    { "flow": [/* route 1 */] },
    { "flow": [/* route 2 */] }
  ]
}
```

Le filtre sur le **premier module de chaque route** détermine la condition d'entrée. Sans filter, tous les bundles passent dans toutes les routes.

**`:else:` route** (fallback) : bouton dans l'UI pour activer une route catch-all qui s'exécute si aucune autre n'a matché. **Critique** : si `:else:` est OFF et toutes les conditions filent à false → bundles perdus silencieusement.

### Iterator

**Module** : `builtin:BasicFeeder` v1

Éclate un array en bundles individuels. Les modules suivants reçoivent un bundle par item.

```json
{
  "id": 3,
  "module": "builtin:BasicFeeder",
  "version": 1,
  "mapper": {
    "array": "{{2.results}}"
  }
}
```

Output : pour chaque item, un bundle accessible via `{{3.value}}` (primitives) ou `{{3.field}}` (objects).

**Coût** : 1 op pour l'iterator + 1 op pour chaque module downstream par item. 100 items × 3 modules = 1 + 300 = **301 ops**.

### Array Aggregator

**Module** : `builtin:BasicAggregator` v1

Regroupe les bundles d'un Iterator en un seul bundle avec un array. Toujours appairer avec un Iterator (`parameters.feeder = ID_iterator`).

```json
{
  "id": 7,
  "module": "builtin:BasicAggregator",
  "version": 1,
  "parameters": {
    "feeder": 3,
    "groupBy": "",
    "stopProcessingOnEmpty": false
  },
  "mapper": {
    "fields": ["email", "amount"]
  }
}
```

**`groupBy`** : expression IML qui crée un sous-array par valeur unique. Vide = un seul array global.

```json
"groupBy": "{{4.country}}"
```

→ produit autant de bundles que de pays distincts, chacun contenant l'array des items de ce pays.

**`fields`** : limite quels champs sont propagés. Performance + clarté.

### Text Aggregator

**Module** : `builtin:TextAggregator` v1

Concatène les bundles en une string unique.

```json
{
  "parameters": {"feeder": 3, "rowSeparator": "\n"},
  "mapper": {"value": "{{4.email}}: {{4.status}}"}
}
```

Sortie : une string `email1: status1\nemail2: status2\n...`. Idéal pour générer des rapports texte ou des CSV simples.

### Numeric Aggregator

**Module** : `builtin:NumericAggregator` v1

```json
{
  "parameters": {"feeder": 3, "fn": "sum"},
  "mapper": {"value": "{{4.amount}}"}
}
```

`fn` : `sum`, `avg`, `min`, `max`, `count`.

### Repeater

**Module** : `builtin:Repeater` v1

Génère N bundles séquentiels. Utile pour boucles connues d'avance.

```json
{
  "module": "builtin:Repeater",
  "version": 1,
  "mapper": {
    "repeats": 5,
    "step": 1,
    "start": 1
  }
}
```

Sortie : 5 bundles avec `{{X.i}}` valant 1, 2, 3, 4, 5.

### Converger

**Module** : `builtin:Converger` v1

Synchronise plusieurs branches d'un Router en un point unique. Tous les bundles de toutes les routes amont sont collectés avant que le suivant tourne. Rare, surtout utile pour des "wait-all" multi-route.

---

## 3. Tools (variables, breakpoints, basic trigger)

### Set Variable

**Module** : `util:SetVariable` v1

Stocke une valeur sous un nom, accessible plus tard via Get Variable.

```json
{
  "module": "util:SetVariable",
  "version": 1,
  "mapper": {
    "name": "user_id",
    "scope": "roundtrip",   
    "value": "{{2.id}}"
  }
}
```

**Scopes** :
- `roundtrip` : disponible jusqu'à la fin du roundtrip courant.
- `execution` : disponible pour toute l'exécution (multi-roundtrip).

### Set Multiple Variables

**Module** : `util:SetVariables` v1

Plus efficace que plusieurs Set Variable consécutifs (1 op au lieu de N).

```json
{
  "module": "util:SetVariables",
  "version": 1,
  "mapper": {
    "scope": "roundtrip",
    "variables": [
      {"name": "user_id", "value": "{{2.id}}"},
      {"name": "user_email", "value": "{{2.email}}"}
    ]
  }
}
```

### Get Variable / Get Multiple Variables

```json
{"module": "util:GetVariable", "mapper": {"name": "user_id"}}
{"module": "util:GetVariables", "mapper": {"variables": ["user_id", "user_email"]}}
```

### Compose String

**Module** : `util:ComposeString` v1

Construit une string via template multi-ligne. Plus lisible que de gros blocs IML inline.

```json
{
  "module": "util:ComposeString",
  "version": 1,
  "mapper": {
    "text": "Bonjour {{1.firstName}},\n\nVotre commande {{2.order_id}} est confirmée.\nMontant : {{formatNumber(2.amount; 2)}}€"
  }
}
```

### Increment Function

**Module** : `util:FunctionIncrementor` v1

Compteur global (par scénario, persistant). Utilisé pour générer des numéros de séquence.

```json
{
  "mapper": {"start": 1, "step": 1, "reset": "never"}
}
```

`reset` : `never`, `daily`, `monthly`, `weekly`.

### Sleep

**Module** : `util:FunctionSleep` v1

Pause l'exécution. **Coûte 1 op même pendant le sleep.**

```json
{"mapper": {"duration": 5}}
```

`duration` en secondes, max 300 (5 min).

Utilisé pour : éviter rate limits en bulk, attendre la dispo d'une ressource async.

### Basic Trigger

**Module** : `util:BasicTrigger` v1

Trigger custom qui prend une définition manuelle d'array d'items. Utile pour tests, scénarios planifiés sur dataset fixe.

### Basic Aggregator de bundles arbitraires

**Module** : `util:Aggregator` v1

Différent de `builtin:BasicAggregator` (qui est appairé à un Iterator). Celui-ci agrège les bundles arrivant sur ce module quelle que soit leur source.

---

## 4. HTTP

### Make a Request

**Module** : `http:ActionSendData` v3

Le couteau suisse. À utiliser quand :
- L'app source/cible n'a pas de connecteur Make natif.
- Le connecteur natif manque une action.
- Performance : un appel HTTP direct est souvent moins cher en ops qu'un module connecteur multi-call.

```json
{
  "id": 5,
  "module": "http:ActionSendData",
  "version": 3,
  "parameters": {
    "handleErrors": false,
    "useNewZeroDateMode": true
  },
  "mapper": {
    "url": "https://api.example.com/v2/users",
    "method": "POST",
    "headers": [
      {"name": "Authorization", "value": "Bearer {{customVariable.API_TOKEN}}"},
      {"name": "Content-Type", "value": "application/json"}
    ],
    "qs": [
      {"name": "page", "value": "1"}
    ],
    "bodyType": "raw",
    "contentType": "application/json",
    "data": "{\"email\":\"{{1.email}}\",\"name\":\"{{1.name}}\"}",
    "parseResponse": true,
    "timeout": 40,
    "followRedirect": true,
    "rejectUnauthorized": true,
    "gzip": true,
    "useTls": false,
    "ca": ""
  }
}
```

**Champs critiques** :
- `bodyType` : `raw`, `x_www_form_urlencoded`, `multipart_form_data`, `binary`. Choisir selon l'API cible.
- `parseResponse` : true → Make parse JSON/XML automatiquement → `{{5.body.field}}` accessible. False → `{{5.body}}` est string brute.
- `timeout` : en secondes, max 300.
- `handleErrors` (parameters) : false = laisser les 4xx/5xx déclencher l'error path. true = capter en `body` et `statusCode` dans le bundle output.

### Variants

- `http:ActionSendDataBasicAuth` : avec basic auth header automatique.
- `http:ActionGetFile` : download un fichier (binaire).
- `http:ActionMakeOAuth2Request` : signe avec une connection OAuth2 enregistrée dans Make (sans gérer le refresh manuellement).

---

## 5. JSON

### Parse JSON

**Module** : `json:ParseJSON` v1

Parse une string JSON en bundle structuré. Nécessaire si la source vient d'un webhook brut, d'un Text parser, etc.

```json
{
  "module": "json:ParseJSON",
  "version": 1,
  "parameters": {
    "type": ""    
  },
  "mapper": {
    "json": "{{1.payload}}"
  }
}
```

**`parameters.type`** : ID d'un Data Structure prédéfini (créé dans Tools → Data Structures). Permet de typer la sortie pour que les modules suivants voient les champs. Sans data structure, output reste générique (untyped).

### Transform to JSON

**Module** : `json:TransformToJSON` v1

Stringify un object en JSON.

```json
{
  "mapper": {
    "object": "{{1.userObject}}"
  }
}
```

### Aggregate to JSON

**Module** : `json:AggregateToJSON` v1

Pendant Iterator → cet aggregator → bundle agrégé en string JSON unique.

---

## 6. Text Parser

### Match Pattern (regex)

**Module** : `regexp:Match` v1

```json
{
  "mapper": {
    "pattern": "(\\d{3})-(\\d{4})",
    "global": true,
    "ignoreCase": false,
    "multiline": false,
    "text": "{{1.body}}"
  }
}
```

Output : array de matches, chaque match exposant `match`, `groups[1]`, `groups[2]`...

### Replace

**Module** : `regexp:Replace` v1

```json
{
  "mapper": {
    "pattern": "\\b\\d{4}\\s\\d{4}\\s\\d{4}\\s\\d{4}\\b",
    "replacement": "[REDACTED]",
    "global": true,
    "text": "{{1.body}}"
  }
}
```

### Get URLs / Emails / Phone numbers

Modules dédiés pour extraire des entités spécifiques d'un texte. Plus simple qu'écrire la regex.

### HTML to Text

**Module** : `regexp:HTMLtoText` v1

Strip HTML et garde le texte. Idéal pour parser des emails HTML.

---

## 7. Tools utilitaires extra

### Switch (Tools)

**Module** : `util:Switcher` v1

Module Switch (différent de la fonction `switch()`). Routing par valeur exacte sans avoir besoin d'un Router complet.

```json
{
  "mapper": {
    "switchInput": "{{1.status}}",
    "cases": [
      {"in": "active", "out": "{{customVariable.ACTIVE_LABEL}}"},
      {"in": "pending", "out": "{{customVariable.PENDING_LABEL}}"}
    ],
    "default": "Unknown"
  }
}
```

### Throw

**Module** : `builtin:Throw` v1

Génère volontairement une erreur. Utilisé pour stopper un chemin et l'envoyer dans Incomplete Executions ou pour tester l'error handling.

```json
{
  "mapper": {
    "message": "Validation failed: missing email"
  }
}
```

---

## 8. Tableau de référence rapide

| Module | ID | Usage typique |
|--------|----|----|
| Webhook custom | `gateway:CustomWebHook` | Trigger HTTP entrant |
| Webhook respond | `gateway:WebhookRespond` | Renvoyer réponse synchrone |
| Mailhook | `mail:CustomEmailTrigger` | Trigger sur email reçu |
| Router | `builtin:BasicRouter` | Branchement multi-route |
| Iterator | `builtin:BasicFeeder` | Éclater array → bundles |
| Array aggregator | `builtin:BasicAggregator` | Regrouper en array |
| Text aggregator | `builtin:TextAggregator` | Concat strings |
| Numeric aggregator | `builtin:NumericAggregator` | Sum/avg/min/max |
| Repeater | `builtin:Repeater` | Boucler N fois |
| Converger | `builtin:Converger` | Sync multi-routes |
| Set variable | `util:SetVariable` | Stocker une valeur |
| Set multiple variables | `util:SetVariables` | Stocker plusieurs (1 op) |
| Get variable | `util:GetVariable` | Lire une valeur |
| Get multiple variables | `util:GetVariables` | Lire plusieurs (1 op) |
| Compose string | `util:ComposeString` | Template multi-ligne |
| Increment | `util:FunctionIncrementor` | Compteur séquence |
| Sleep | `util:FunctionSleep` | Pause |
| Basic trigger | `util:BasicTrigger` | Trigger manuel/test |
| Switch | `util:Switcher` | Mapping valeur → valeur |
| HTTP request | `http:ActionSendData` | Appel API générique |
| HTTP get file | `http:ActionGetFile` | Download fichier |
| Parse JSON | `json:ParseJSON` | String → object |
| Transform to JSON | `json:TransformToJSON` | Object → string |
| Aggregate to JSON | `json:AggregateToJSON` | Iterator → JSON array |
| Match pattern | `regexp:Match` | Regex extraction |
| Replace | `regexp:Replace` | Regex substitution |
| HTML to text | `regexp:HTMLtoText` | Strip HTML |
| Throw | `builtin:Throw` | Erreur volontaire |
| Break (handler) | `builtin:Break` | Retry + DLQ |
| Resume (handler) | `builtin:Resume` | Continuer avec fallback |
| Rollback (handler) | `builtin:Rollback` | Revert + stop |
| Commit (handler) | `builtin:Commit` | Stop comme succès |
| Ignore (handler) | `builtin:Ignore` | Skip ce bundle |
