# Blueprint JSON — Structure complète

Le blueprint est le JSON qui définit le graphe de modules d'un scénario. Comprendre sa structure permet de l'éditer programmatiquement, de migrer entre comptes, ou de générer des scénarios depuis du code.

## 1. Squelette racine

```json
{
  "name": "Scenario Name",
  "flow": [ /* array de modules */ ],
  "metadata": {
    "version": 1,
    "scenario": {
      "roundtrips": 1,
      "maxErrors": 3,
      "autoCommit": true,
      "autoCommitTriggerLast": true,
      "sequential": false,
      "confidential": false,
      "dataloss": false,
      "dlq": false,
      "freshVariables": false
    },
    "designer": {
      "orphans": []
    },
    "zone": "eu1.make.com"
  }
}
```

### Champs racine

- **`name`** : nom affiché dans l'UI.
- **`flow`** : array ordonné de modules. L'ordre est le flux d'exécution.
- **`metadata`** : configuration runtime + UI hints.

### `metadata.scenario` (settings runtime)

| Champ | Type | Description |
|-------|------|-------------|
| `roundtrips` | int | Nombre max de cycles complets par exécution. Défaut 1. |
| `maxErrors` | int | Erreurs consécutives avant désactivation auto. Défaut 3. |
| `autoCommit` | bool | Commit automatique en fin d'exécution réussie. |
| `autoCommitTriggerLast` | bool | Commit après le dernier bundle même si erreur. |
| `sequential` | bool | Si `true`, les exécutions ne se parallélisent pas (queue). Critique pour idempotence. |
| `confidential` | bool | Masque les inputs/outputs dans les logs (RGPD). |
| `dataloss` | bool | Si `true`, autorise la perte de bundles non traités au stop forcé. |
| `dlq` | bool | Active la queue d'incomplete executions (DLQ-like). |
| `freshVariables` | bool | Réinitialise les variables à chaque exécution. |

### `metadata.zone`

La zone de l'instance (ex. `eu1.make.com`). Critique pour les imports cross-zone.

---

## 2. Structure d'un module

```json
{
  "id": 1,
  "module": "gateway:CustomWebHook",
  "version": 1,
  "parameters": {
    "hook": 12345,
    "maxResults": 1
  },
  "mapper": {},
  "metadata": {
    "designer": {
      "x": 0,
      "y": 0,
      "name": "Custom webhook"
    },
    "restore": { /* sample data pour le designer */ },
    "expect": [ /* schema des params attendus */ ],
    "interface": [ /* output schema */ ]
  },
  "filter": null,
  "onerror": [ /* error handlers */ ]
}
```

### Champs clés

- **`id`** : entier local au blueprint, croissant. Utilisé dans les références (`{{1.body}}` = output du module ID 1).
- **`module`** : identifiant `app:Action` (ex. `gateway:CustomWebHook`, `airtable2:ActionCreateRecord`). Voir `05-builtin-modules.md`.
- **`version`** : version du module (1, 2, 3...). Si on push avec une mauvaise version → erreur 500.
- **`parameters`** : config "static" du module (connection ID, hook ID, mode). Pas mappable dans l'UI (sauf advanced).
- **`mapper`** : config "dynamic" — c'est ici que vivent les expressions IML (`{{1.body.email}}`). Mappable via le panneau de droite.
- **`metadata.designer`** : position visuelle (x, y), nom personnalisé. Coordonnées sont en pixels logiques.
- **`metadata.expect`** : schema d'inputs (utilisé par le designer pour proposer les champs).
- **`metadata.interface`** : schema d'outputs (utilisé par les modules suivants pour proposer les variables).
- **`filter`** : filter conditionnel **avant** ce module. Si false → bundle skip ce module.
- **`onerror`** : array de modules error handlers (Break, Resume, etc.).

### Différence `parameters` vs `mapper`

C'est le piège n°1 quand on génère des blueprints.

- `parameters` = config statique d'instance (ex. `connection: 1234`, `hook: 5678`, `useNewZeroDateMode: true`). Pas évalué runtime.
- `mapper` = valeurs runtime, supportent IML (`{{1.body}}`). Évalué à chaque exécution.

Exemple module HTTP :

```json
{
  "id": 3,
  "module": "http:ActionSendData",
  "version": 3,
  "parameters": {
    "handleErrors": false,
    "useNewZeroDateMode": true
  },
  "mapper": {
    "url": "{{2.api_url}}/users",
    "method": "POST",
    "headers": [
      {"name": "Authorization", "value": "Bearer {{1.token}}"}
    ],
    "qs": [],
    "bodyType": "raw",
    "contentType": "application/json",
    "data": "{\"email\":\"{{1.email}}\"}",
    "useMtls": false,
    "shareCookies": false,
    "ca": "",
    "rejectUnauthorized": true,
    "followRedirect": true,
    "useQuerystring": false,
    "gzip": true,
    "useTls": false,
    "parseResponse": true,
    "timeout": 40
  }
}
```

---

## 3. Modules de flow control (Router, Iterator, Aggregator)

### Router

```json
{
  "id": 5,
  "module": "builtin:BasicRouter",
  "version": 1,
  "mapper": null,
  "metadata": { "designer": { "x": 300, "y": 0 } },
  "routes": [
    {
      "flow": [
        {
          "id": 6,
          "module": "http:ActionSendData",
          "version": 3,
          "mapper": { /* ... */ },
          "filter": {
            "name": "Type = invoice",
            "conditions": [[
              {"a": "{{1.type}}", "b": "invoice", "o": "text:equal"}
            ]]
          }
        }
      ]
    },
    {
      "flow": [ /* autre route */ ]
    }
  ]
}
```

Chaque route a son propre `flow` (array de modules). Le filter sur le **premier module de la route** définit la condition d'entrée.

### Iterator

```json
{
  "id": 7,
  "module": "builtin:BasicFeeder",
  "version": 1,
  "mapper": {
    "array": "{{6.results}}"
  }
}
```

Itère sur `mapper.array`. Les modules suivants reçoivent un bundle par item, accessible via `{{7.value}}` (ou `{{7.field}}` si l'array contient des objets).

### Array Aggregator

```json
{
  "id": 9,
  "module": "builtin:BasicAggregator",
  "version": 1,
  "parameters": {
    "feeder": 7,            
    "groupBy": "",          
    "stopProcessingOnEmpty": false
  },
  "mapper": {
    "fields": ["email", "status"]
  }
}
```

`parameters.feeder` = ID de l'iterator dont on agrège la sortie. **Critique** : sans cet appariement, l'aggregator ne sait pas où regrouper.

`mapper.fields` = quels champs du bundle propager dans l'array agrégé.

`parameters.groupBy` = expression IML qui crée un sous-bundle par valeur unique (mode group-by SQL). Vide = un seul bundle agrégé.

### Text Aggregator

```json
{
  "id": 10,
  "module": "builtin:TextAggregator",
  "version": 1,
  "parameters": {
    "feeder": 7,
    "rowSeparator": "\n"
  },
  "mapper": {
    "value": "{{7.email}}: {{7.status}}"
  }
}
```

Concatène les bundles en une seule string.

### Numeric Aggregator

```json
{
  "id": 11,
  "module": "builtin:NumericAggregator",
  "version": 1,
  "parameters": {
    "feeder": 7,
    "fn": "sum"
  },
  "mapper": {
    "value": "{{7.amount}}"
  }
}
```

`fn` : `sum`, `avg`, `min`, `max`, `count`.

---

## 4. Filters (entre modules)

```json
"filter": {
  "name": "Has email and is active",
  "conditions": [
    [
      {"a": "{{1.email}}", "o": "exists"},
      {"a": "{{1.status}}", "b": "active", "o": "text:equal"}
    ],
    [
      {"a": "{{1.is_admin}}", "b": "true", "o": "text:equal"}
    ]
  ]
}
```

**Logique** : AND dans les arrays internes, OR entre les arrays externes.

Le filtre ci-dessus = `(email_exists AND status=active) OR is_admin=true`.

### Opérateurs disponibles (liste exhaustive)

**Texte** : `text:equal`, `text:notEqual`, `text:contain`, `text:notContain`, `text:startWith`, `text:notStartWith`, `text:endWith`, `text:notEndWith`, `text:pattern` (regex), `text:notPattern`.

**Numérique** : `number:equal`, `number:notEqual`, `number:greater`, `number:greaterOrEqual`, `number:less`, `number:lessOrEqual`.

**Date** : `date:equal`, `date:notEqual`, `date:after`, `date:afterOrEqual`, `date:before`, `date:beforeOrEqual`, `date:within`, `date:notWithin`.

**Existence** : `exists`, `notExist`.

**Boolean** : `boolean:isTrue`, `boolean:isFalse`.

**Array** : `array:contain`, `array:notContain`, `array:length:equal`, `array:length:greater`, `array:length:less`.

### Continue when no module succeeds

Sur les routers, l'option **fallback `:else:`** dans l'UI (toggle) crée une route filter `else: true`. Si toutes les routes filter à false **et que `:else:` est OFF**, les bundles tombent dans un trou silencieux. Toujours activer `:else:` ou avoir une route catch-all.

---

## 5. Error handlers (`onerror`)

Tout module peut avoir un array `onerror` qui contient un (ou plusieurs) module(s) handler.

```json
{
  "id": 5,
  "module": "http:ActionSendData",
  "version": 3,
  "mapper": { /* ... */ },
  "onerror": [
    {
      "id": 6,
      "module": "builtin:Break",
      "version": 1,
      "mapper": {
        "retry": true,
        "retryAttempts": 3,
        "retryInterval": 15
      }
    }
  ]
}
```

Types de handlers (le `module` du handler) :
- `builtin:Break` — store dans Incomplete Executions, retry possible
- `builtin:Resume` — continuer comme si pas d'erreur, fallback bundle dans `mapper.message`
- `builtin:Rollback` — revert et stop scenario
- `builtin:Commit` — commit transactions, stop comme succès
- `builtin:Ignore` — skip ce bundle, continuer le suivant

Voir `06-error-handling.md` pour le détail.

---

## 6. Webhook trigger params

```json
{
  "id": 1,
  "module": "gateway:CustomWebHook",
  "version": 1,
  "parameters": {
    "hook": 67890,        
    "maxResults": 1       
  },
  "mapper": {},
  "metadata": {
    "designer": {"name": "Custom webhook"},
    "expect": [
      {
        "name": "maxResults",
        "type": "number",
        "label": "Maximum number of results"
      }
    ],
    "interface": [
      {"name": "email", "type": "text", "label": "Email"},
      {"name": "amount", "type": "number", "label": "Amount"}
    ]
  }
}
```

`parameters.hook` = ID du webhook (créé via API ou UI). Le payload reçu est mappé selon `metadata.interface`.

---

## 7. Routes complexes : exemple complet

Scénario : webhook entrant → router → 2 routes (invoice / refund) → chacune avec error handling.

```json
{
  "name": "Stripe events router",
  "flow": [
    {
      "id": 1,
      "module": "gateway:CustomWebHook",
      "version": 1,
      "parameters": {"hook": 11111, "maxResults": 1},
      "mapper": {}
    },
    {
      "id": 2,
      "module": "builtin:BasicRouter",
      "version": 1,
      "mapper": null,
      "routes": [
        {
          "flow": [
            {
              "id": 3,
              "module": "airtable2:ActionCreateRecord",
              "version": 3,
              "parameters": {"__IMTCONN__": 22222},
              "mapper": {
                "base": "appXXXX",
                "table": "Invoices",
                "record": {
                  "Customer": "{{1.customer_email}}",
                  "Amount": "{{1.amount_total}}"
                }
              },
              "filter": {
                "name": "Invoice events",
                "conditions": [[
                  {"a": "{{1.type}}", "b": "invoice.paid", "o": "text:equal"}
                ]]
              },
              "onerror": [
                {
                  "id": 4,
                  "module": "builtin:Break",
                  "version": 1,
                  "mapper": {"retry": true, "retryAttempts": 3, "retryInterval": 15}
                }
              ]
            }
          ]
        },
        {
          "flow": [
            {
              "id": 5,
              "module": "slack:CreateMessage",
              "version": 5,
              "parameters": {"__IMTCONN__": 33333},
              "mapper": {
                "channel": "#refunds",
                "text": "Refund: {{1.customer_email}} — {{1.amount_total}}"
              },
              "filter": {
                "name": "Refund events",
                "conditions": [[
                  {"a": "{{1.type}}", "b": "charge.refunded", "o": "text:equal"}
                ]]
              }
            }
          ]
        }
      ]
    }
  ],
  "metadata": {
    "version": 1,
    "scenario": {
      "roundtrips": 1,
      "maxErrors": 3,
      "autoCommit": true,
      "autoCommitTriggerLast": true,
      "sequential": false,
      "confidential": false,
      "dataloss": false,
      "dlq": false,
      "freshVariables": false
    },
    "designer": {"orphans": []},
    "zone": "eu1.make.com"
  }
}
```

---

## 8. Connection IDs : la pire gotcha de l'export/import

Les modules référencent les connections via `parameters.__IMTCONN__` (ou `parameters.connection` selon les apps anciennes).

```json
"parameters": {
  "__IMTCONN__": 67890
}
```

**67890** est l'ID local au compte d'origine. Importer le blueprint dans un autre compte sans renommer cet ID → erreur "Connection not found" ou "Module is missing connection".

**Workflow d'import propre** :
1. Pull le blueprint source
2. Lister toutes les connections référencées : `jq '.. | objects | select(.["__IMTCONN__"]) | .["__IMTCONN__"]' blueprint.json | sort -u`
3. Dans le compte cible, créer les connections équivalentes
4. Mapper old_id → new_id et substituer dans le blueprint
5. Push

Idem pour `hook` (webhooks) et les références à des Data Stores ou Custom Variables.

---

## 9. Tips d'édition programmatique

### Lister tous les modules d'un blueprint avec jq

```bash
jq '[.. | objects | select(.module) | {id, module, version}]' blueprint.json
```

### Renommer un module

```bash
jq '(.flow[] | select(.id == 3) | .metadata.designer.name) = "Renamed module"' \
   blueprint.json > new.json
```

### Changer la valeur d'un mapper (ex. mettre à jour un endpoint URL)

```bash
jq '(.flow[] | select(.module == "http:ActionSendData") | .mapper.url) |= sub("api.old.com"; "api.new.com")' \
   blueprint.json > new.json
```

### Substituer toutes les connections

```bash
jq --argjson map '{"67890":"99999"}' \
   '(.. | objects | select(.["__IMTCONN__"]) | .["__IMTCONN__"]) |= ($map[tostring] // .)' \
   blueprint.json > new.json
```

---

## 10. Limites pratiques

- **Profondeur de routes imbriquées** : pas de limite stricte, mais >3 niveaux devient illisible. Refactor en sous-scénarios appelés via webhook.
- **Nombre de modules par scénario** : pas de limite officielle, mais >50 modules indique souvent un design à éclater.
- **Taille du blueprint** : pas de limite documentée, en pratique <1 MB.
- **IDs de modules** : entiers, croissants. Si on insère manuellement un module, choisir un ID > max existant pour éviter collisions.
