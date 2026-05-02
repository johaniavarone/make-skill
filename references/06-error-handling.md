# Error Handling — Référence complète

L'error handling est ce qui distingue un scénario prototype d'un scénario production. Make propose 5 directives + une queue d'incomplete executions + des paramètres de scenario qui forment ensemble un système robuste.

---

## 1. Les 5 directives

Chaque module peut recevoir un error handler (clic droit → Add error handler dans l'UI, ou via `onerror` array dans le blueprint). La directive du handler détermine ce qui se passe quand le module échoue.

### Break

**Module handler** : `builtin:Break` v1

**Comportement** : stoppe l'exécution, stocke le bundle dans **Incomplete Executions**, peut être retry automatiquement.

**Quand l'utiliser** : appels API susceptibles d'erreurs transitoires (429, 502, 503, timeouts). C'est la directive la plus importante en production.

```json
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
```

`retryInterval` en minutes. Make utilise un **intervalle fixe**, pas d'exponential backoff natif (workaround section 6).

**Prérequis** : activer **Allow storing of incomplete executions** dans les settings du scénario, sinon Break = échec définitif.

### Resume

**Module handler** : `builtin:Resume` v1

**Comportement** : ignore l'erreur, continue avec un bundle "fallback" prédéfini. Le scénario ne s'arrête pas.

```json
{
  "module": "builtin:Resume",
  "version": 1,
  "mapper": {
    "message": {
      "id": null,
      "status": "fallback"
    }
  }
}
```

**Quand l'utiliser** : enrichissements optionnels (LinkedIn lookup, IP geocoding) où une donnée manquante est tolérable.

### Ignore

**Module handler** : `builtin:Ignore` v1

**Comportement** : skip le bundle courant et continue avec le suivant. Pas de fallback, pas d'erreur loggée comme failure.

```json
{"module": "builtin:Ignore", "version": 1}
```

**Quand l'utiliser** : opérations best-effort sans impact downstream (notification de log non-critique, tag d'audit qui peut manquer).

### Rollback

**Module handler** : `builtin:Rollback` v1

**Comportement** : tente de **revert** les opérations transactionnelles déjà effectuées dans le scénario, puis stoppe avec status `error`.

```json
{
  "module": "builtin:Rollback",
  "version": 1,
  "mapper": {
    "message": "Rollback: order creation failed"
  }
}
```

**Quand l'utiliser** : workflows multi-étapes avec contraintes transactionnelles (créer commande Stripe + créer ligne Airtable + créer Doc → si Doc fail, on veut annuler).

**Limite cruciale** : Make ne peut rollback que les modules qui supportent explicitement la transaction (data stores, certaines DBs). Pour les API calls HTTP arbitraires (Stripe, Slack...), Rollback ne peut pas re-call l'endpoint inverse — il marque juste l'execution comme failed. Pour vraies compensations, il faut écrire les modules de compensation manuellement et router vers eux.

### Commit

**Module handler** : `builtin:Commit` v1

**Comportement** : commit toutes les transactions en cours et stoppe le scénario en `success`. Pas marqué comme erreur.

```json
{
  "module": "builtin:Commit",
  "version": 1,
  "mapper": {
    "message": "Lead not qualified — stopping cleanly"
  }
}
```

**Quand l'utiliser** : arrêts business-logique (lead non qualifié, condition métier qui ne nécessite pas de retry).

---

## 2. Settings de scénario liés à l'error handling

Dans **Scenario settings → Advanced settings** :

### Allow storing of incomplete executions

`metadata.scenario.dlq` dans le blueprint.

**À activer** dès qu'on utilise Break. Sans ça, Break = perte du bundle.

### Number of consecutive errors

`metadata.scenario.maxErrors`. Défaut 3.

Si le scénario échoue N fois consécutives, il est **désactivé automatiquement**. Notification email envoyée.

Ajuster selon volume :
- Scénario à 1 trigger/jour : `maxErrors=1` est OK.
- Scénario haute fréquence : `maxErrors=10+` pour ne pas désactiver pour quelques transients.

### Sequential processing

`metadata.scenario.sequential`. Défaut `false`.

Si `true`, les exécutions du scénario ne se parallélisent pas. Make queue les triggers et les traite un par un. Critique quand l'ordre matters (write into shared resource sans locks).

**Avantage** : évite les race conditions sur les data stores, les Sheets append, les writes Airtable concurrents.
**Coût** : latence si volume élevé.

### Auto commit

`metadata.scenario.autoCommit`. Défaut `true`.

Si `true`, à la fin d'un cycle réussi, Make commit toutes les transactions ouvertes. Désactiver seulement pour des cas avancés.

### Data loss

`metadata.scenario.dataloss`. Défaut `false`.

Si `true`, Make permet de perdre des bundles non traités lors d'un stop forcé ou d'une erreur. À éviter en production.

### Confidential

`metadata.scenario.confidential`. Défaut `false`.

Si `true`, masque les inputs/outputs dans les logs d'exécution. Critique pour RGPD, données médicales, paiements.

---

## 3. Incomplete Executions Queue (DLQ-like)

Quand Break déclenche, le bundle est stocké dans **Incomplete Executions** avec :
- Le payload exact à l'instant de l'erreur
- L'ID du module qui a échoué
- Le message d'erreur
- Le nombre de retries restants

### Accès UI

Scenario → Incomplete executions tab → liste paginée. Pour chaque entrée :
- **Resolve** : marquer comme traité manuellement (purge sans retry).
- **Retry** : relance l'execution depuis le module qui a échoué.

### Accès API

```bash
# Lister
curl "$BASE/scenarios/$ID/incomplete-executions" \
  -H "Authorization: Token $TOKEN"

# Retry
curl -X POST "$BASE/scenarios/$ID/incomplete-executions/$IE_ID/retry" \
  -H "Authorization: Token $TOKEN"
```

Permet d'automatiser le drainage périodique de la queue après un incident.

### Auto-retry par Make

Si la directive Break a `retry: true`, Make tente N fois avec l'intervalle configuré. Si tous les retries échouent, l'entrée reste dans la queue, en attente d'action humaine ou API.

---

## 4. Stratégies par type d'erreur

| Type d'erreur | Directive recommandée | Pourquoi |
|---|---|---|
| HTTP 401/403 (auth) | Break sans retry | Re-auth manuelle nécessaire |
| HTTP 404 (not found) | Resume avec fallback ou Ignore | Probablement pas transient |
| HTTP 429 (rate limit) | Break + retry 5 min, 5 attempts | Reprend quand limite reset |
| HTTP 500/502/503 (transient) | Break + retry 2 min, 3 attempts | API recover usually |
| HTTP timeout | Break + retry 5 min, 3 attempts | Transient |
| Validation error 400 | Break sans retry | Bundle malformé, pas de retry possible |
| Connection error (network) | Break + retry 1 min, 3 attempts | Network blip |
| Quota exhausted (Stripe, OpenAI) | Resume avec log | Continue le scénario, alerter manuellement |
| Module returned no data | Resume avec fallback ou Ignore | Pas vraiment une erreur |

---

## 5. Pattern : exponential backoff manuel

Make ne supporte pas exponential backoff natif. Workaround :

**Approche 1 : escaliers de Break handlers**

Mettre 3 modules HTTP successifs après l'original. Le 1er fait l'appel, son onerror Resume vers le suivant qui fait le même appel après un Sleep de 30s, son onerror vers le 3e après Sleep de 2min, etc.

```
HTTP (try 1) ─ onerror: Resume ─→ Sleep 30s ─→ HTTP (try 2) ─ onerror: Resume ─→ Sleep 2min ─→ HTTP (try 3) ─ onerror: Break
```

Chaque retry écrit dans un Set Variable un compteur, et le prochain Sleep utilise `2^attempt * 30s`.

**Approche 2 : Data Store + sub-scénario récursif**

Sub-scénario "retry-with-backoff" appelé via webhook. Lit le compteur d'attempts dans un data store, calcule le sleep, fait l'appel, et incrémente. Le scénario parent appelle ce sub-scénario via HTTP.

**Approche 3 : intervalle adapté au pattern de l'API**

Pour Stripe (rate limit reset 1s), un Break à 1min suffit. Pour OpenAI tier 1 (3 req/min), Break à 20s. Configurer empiriquement.

---

## 6. Pattern : Circuit Breaker

Quand un service externe est down, ne pas continuer à le hammer. Stocker un flag dans un Data Store.

**Schéma** :

```
[trigger] → [DataStore Get key=service_x_down]
          ↓
        Filter: 1.value != "true"  (skip si down)
          ↓
        [HTTP call to service X]
          ↓ onerror Break
        [DataStore Set key=service_x_down value=true expiresAt=now+10min]
```

Au prochain trigger, le filter exclut les bundles → service down skipped pendant 10 min. À expiration, on retente.

---

## 7. Pattern : DLQ logger

Capturer toutes les erreurs dans une table dédiée pour audit a posteriori.

```
[module qui peut fail]
  ↓ onerror
[Break]
  ↓ (avant Break en fait : route via Resume puis Airtable Create Record)

OU :

[module qui peut fail]
  ↓ onerror Resume avec bundle de fallback {error: "..."}
  ↓
Filter: error != null
  ↓
Airtable Create Record (table = "Errors log")
  ↓
Slack message #ops-alerts
```

Avantage : visibilité complète des erreurs sans interrompre le flow.

---

## 8. Throw (erreur volontaire)

Module : `builtin:Throw`. Génère une erreur custom. Utile pour :

**Validation explicite** :

```
Webhook → Filter (1.email == "") → Throw (message: "Missing email")
```

Si filter passe (email vide), Throw déclenche → onerror du Throw → log + alerte.

**Tester l'error handling** : Throw pour simuler un échec et vérifier que les handlers et la DLQ fonctionnent en pre-prod.

---

## 9. Erreurs courantes et fixes

### "Module returned no data" (warning)

Pas une erreur, mais le module n'a renvoyé aucun bundle (ex. search sans résultat). Les modules suivants ne tournent pas. Pour forcer un fallback : Resume avec un bundle de defaults.

### "RuntimeError: ReferenceError: X is not defined"

Expression IML qui référence un module ID inexistant ou un champ jamais peuplé. Toujours `ifempty` pour défaut.

### "BundleValidationError"

Bundle ne correspond pas au schema attendu (ex. champ obligatoire absent). Vérifier l'output du module précédent dans les logs.

### "ConnectionError"

Connection (OAuth/API key) invalide ou expirée. Reconnecter dans **Connections**.

### "DataError"

Données mal typées ou incomplètes envoyées à l'API cible. Inspecter le bundle input du module qui a fail dans l'execution log.

### Scénario désactivé automatiquement

Les `maxErrors` ont été dépassés. Email envoyé. Vérifier la queue d'Incomplete Executions, fixer la cause racine, retry, réactiver.

---

## 10. Monitoring & alerting

### Email automatique

Make envoie un email à l'owner du scénario quand :
- Le scénario est désactivé automatiquement
- Un bundle est stocké en Incomplete Executions (option : dans Settings)

### Slack/Discord notifs (manuel)

Ajouter une route dédiée dans chaque scénario critique :

```
[error handler Break]
  ↓ avant Break, faire un Resume vers :
[Slack Create Message channel=#ops-alerts text="Scenario X failed: {{error.message}}"]
  ↓ puis Break
```

### Dashboard custom

Récupérer périodiquement via API les `executions` avec status `error` et les pousser dans un dashboard externe (Grafana, custom Webflow page, Notion).

```bash
# Cron toutes les heures
curl "$BASE/scenarios/$ID/logs?status=error&pg[limit]=100" \
  -H "Authorization: Token $TOKEN" \
  | jq '.response.scenarioLogs[] | {id, started, error}' \
  > errors_$(date +%Y%m%d_%H).json
```

---

## 11. Checklist error handling production

Avant d'activer un scénario en prod, vérifier :

- Tout module qui appelle une API externe a un error handler.
- Les Break handlers ont des paramètres de retry adaptés au type d'erreur attendu.
- `Allow storing of incomplete executions` est activé.
- `maxErrors` est ajusté (typique : 5-10 pour HF, 1-3 pour LF).
- `sequential` est activé si l'ordre matters.
- Les enrichissements optionnels utilisent Resume avec fallback explicite.
- Les opérations transactionnelles ont un Rollback ou des modules de compensation.
- Une route ou un module log les erreurs dans une destination consultable (Sheet, Slack, Airtable).
- Test de simulation d'erreur fait via Throw, vérifié que tout fonctionne.
- L'idempotence du scénario est validée (lancer 2x avec le même payload ne crée pas de doublon).
