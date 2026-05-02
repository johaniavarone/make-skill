# Best Practices Make.com

Conventions et patterns testés en production. Ne sont pas des règles absolues, mais des règles par défaut sensées dans 90% des cas.

---

## 1. Naming conventions

### Scenarios

Format recommandé : `[Domaine] Action source → cible`.

```
[CRM] Sync new Airtable contact → HubSpot
[Billing] Create Stripe customer on Softr signup
[Internal] Daily digest of failed scenarios → Slack
[Webhook] Stripe events router
```

L'avantage : tri alphabétique groupe les scénarios par domaine. Cherchable dans la liste qui devient longue rapidement.

### Folders

Organiser par client ou par domaine fonctionnel. Éviter les folders profonds (>2 niveaux) : Make UI les supporte mais devient lourd.

```
Clients/
  Star Academy/
  Pierre Suppa/
  Nöje/
Internal/
  Monitoring/
  Onboarding/
Templates/
```

### Modules

Renommer **systématiquement** chaque module dans le designer (clic droit → Rename). Le nom par défaut "Airtable - Search records" devient illisible à 15 modules. Préférer :

```
"Find existing customer by email"
"Create invoice line"
"Notify ops on failure"
```

### Connections

Format : `<Service> - <Compte> - <Env>`.

```
Stripe - Acme Production
Stripe - Acme Test
Airtable - Personal
Gmail - johan@jipsconseil.com
```

### Hooks (webhooks)

Format : `<Source system> - <Event type>`.

```
Stripe - Customer events
Softr - Form: Contact us
GitHub - PR events
```

### Custom variables

`UPPER_SNAKE_CASE` pour les constantes globales :

```
ENV
AIRTABLE_BASE_PROD
WEBHOOK_SECRET_STRIPE
SLACK_CHANNEL_ALERTS
```

### Data stores

Format : `<purpose>-<scope>`.

```
dedup-stripe-events
cache-openai-translations
state-customer-onboarding
```

---

## 2. Idempotence (règle d'or)

Tout scénario qui rejoue (retry, replay) doit produire le même état final qu'au premier passage.

### Patterns

**Search-then-create** : avant de créer une ressource, search par clé unique. Si trouvé, update ; sinon, create.

```
Module: Airtable Search Records (filter: email = {{1.email}})
   ↓
Router :
  Route 1 (1.results.length > 0) :
    Filter: length(2.results) > 0
    → Airtable Update Record (record_id = 2.results[1].id)
  Route 2 (none) :
    Filter: length(2.results) == 0
    → Airtable Create Record
```

**Dedup ledger** : voir `07-webhooks-datastores.md`. Data store comme registre des IDs déjà traités.

**Idempotency-Key header** : sur Stripe, GitHub, Square, et beaucoup d'APIs modernes :

```json
"headers": [
  {"name": "Idempotency-Key", "value": "{{1.event_id}}"}
]
```

L'API rejette le 2e call avec la même clé (ou retourne le résultat du premier).

**Stamp synced-at** : marquer la source après traitement. Le poll suivant (`Last edited > last_synced`) skip naturellement.

---

## 3. Optimisation des opérations

### Compter avant de construire

Au design, estimer les ops par exécution :

```
Trigger (Watch records) : 1 op
Filter avant Iterator : 0 op (filter sur module précédent gratuit)
Iterator sur 50 items : 1 op
Per item × 3 modules : 3 × 50 = 150 ops
Aggregator : 1 op
Send notification : 1 op

Total : 154 ops par exécution
× 10 exécutions/jour = 1540 ops/jour = 46 200 ops/mois
```

Si le plan permet 100k ops/mois, on a de la marge. Si 10k, à reconsidérer.

### Optimisations classiques

**Filter avant Iterator** : pousser le filter en amont. 100 items × Iterator vs 100 items filtered → 5 items × Iterator = 95 ops économisées.

**Set Multiple Variables au lieu de N Set Variable** : 1 op vs N ops.

**Parse JSON dans le bon module** : si l'output d'un module HTTP est `parseResponse: true`, pas besoin d'un Parse JSON après. Économise 1 op par bundle.

**Bundle response shape** : éviter de faire 5 calls API quand 1 call avec un meilleur endpoint suffit. Privilégier les endpoints `/batch` ou `/list` quand dispo.

**Webhook trigger > polling** : voir principe global. Polling coûte des ops à vide.

**Limit sur les triggers polling** : sans limit, le premier run peut sucer 1000 items d'un coup. Toujours `limit: 5` ou `10` pour le warm-up.

**Cache via Data Store** : si une API retourne souvent la même chose, cacher. Lecture data store = 1 op vs API call = 1 op + latence.

---

## 4. Sécurité

### Secrets

**Custom Variables** pour les secrets non-sensibles (URLs, IDs publics).

**Connections** pour les API keys, OAuth, tokens. Make les chiffre. Ne jamais hardcoder un secret dans un mapper IML.

**Webhook secrets** : mettre dans Custom Variable, valider en module 2 du scénario.

### Validation des inputs

Tout webhook public doit valider :
1. **Existence** des champs critiques (`{{1.email}}` non null).
2. **Format** (email, URL, nombre).
3. **Authenticity** (signature HMAC, IP allowlist, secret partagé).

### Logs et confidentialité

`metadata.scenario.confidential: true` masque inputs/outputs dans les logs. À activer pour :
- Données médicales
- Données paiement (cartes, IBAN)
- PII étendu (passport, NIN)
- Logs RH

Sinon, accepter que l'opérateur Make peut voir les bundles dans les logs (ce qui est normal pour debug).

### Connections partagées

Un team peut avoir plusieurs membres avec accès aux mêmes connections. Ne pas mettre de connection "personal Gmail" sur un team partagé : tout membre peut s'en servir. Préférer des comptes "service" dédiés (`make@client.com`).

---

## 5. Architecture multi-environnement

### Setup minimal : 2 teams

- Team **Production** : scénarios live.
- Team **Staging** : copies pour tester.

Connections séparées entre les deux teams (Stripe Test vs Stripe Live, Airtable bases dev vs prod).

### Setup avancé : feature branches

Pour développement en cours :
- Cloner le scénario en `[WIP] <name>` dans le même team
- Travailler dessus tant qu'inactif
- Une fois testé : copier le blueprint, l'appliquer sur le scénario prod, activer

Workflow :
```bash
# 1. Pull WIP
curl ".../scenarios/$WIP_ID/blueprint" > wip.json

# 2. Pull prod
curl ".../scenarios/$PROD_ID/blueprint" > prod.json

# 3. Diff (visualiser changements)
diff <(jq -S . wip.json) <(jq -S . prod.json)

# 4. Push WIP comme prod (avec back-up de prod)
cp prod.json prod.backup.$(date +%s).json
curl -X PATCH ".../scenarios/$PROD_ID" \
  --data "$(jq -n --arg bp "$(cat wip.json)" '{blueprint:$bp}')"
```

### Configuration via Custom Variables

```
ENV = "prod"   (ou "staging", "dev")

AIRTABLE_BASE_PROD = "appXXX"
AIRTABLE_BASE_STAGING = "appYYY"
```

Et dans le scénario :

```
{{if(customVariable.ENV == "prod"; customVariable.AIRTABLE_BASE_PROD; customVariable.AIRTABLE_BASE_STAGING)}}
```

Switcher d'env = changer une seule variable.

---

## 6. Monitoring

### Dashboard d'erreurs

Scénario planifié quotidien :
1. Pour chaque scénario critique (liste hardcodée ou fetched via API) :
2. Lister les executions sur les dernières 24h avec status `error`.
3. Agréger en text aggregator.
4. Envoyer dans Slack/Email.

### Alertes en temps réel

Pour les erreurs critiques (paiement, sécurité) : ne pas attendre le digest quotidien. Notifier dans Slack au moment de l'erreur via un module dans la route Resume du handler.

### Métriques côté infra

API `GET /organizations/{id}/usage` donne les ops consommés. Plot dans un dashboard Notion/Webflow pour suivre la consommation par scénario.

---

## 7. Refactoring d'un scénario "monstre"

Quand un scénario dépasse 30+ modules avec routes profondes :

### Symptômes

- Designer lent (drag-drop laggy).
- Difficile de comprendre le flow.
- Difficile à debug (où l'erreur a-t-elle eu lieu ?).
- Un seul scénario = un seul point de défaillance.

### Patterns de découpage

**Sub-scénarios via webhook** :

Scénario A `webhook → router → 3 routes`. Chaque route appelle un sous-scénario via HTTP module ciblant un webhook custom de B/C/D.

```
A : Webhook → Router
  Route invoice : HTTP POST → B (webhook URL)
  Route refund : HTTP POST → C (webhook URL)
  Route subscription : HTTP POST → D (webhook URL)

B : Webhook → ... (logique invoice)
C : Webhook → ... (logique refund)
D : Webhook → ... (logique subscription)
```

Avantages : chaque sous-scénario est petit, testable indépendamment, redéployable seul. Désavantage : 1 hop HTTP supplémentaire = +2 ops par appel.

**Make Functions (Enterprise)** : factoriser des bouts de logique en custom functions JavaScript. Utilisable partout, 0 op par appel.

**Custom Apps** : pour la logique vraiment répétée (intégration à une API custom client), créer une Custom App qui expose les modules réutilisables.

---

## 8. Tests

Make n'a pas de framework de tests natif. Workarounds :

### Test via "Run once"

Le bouton "Run once" dans le designer permet de déclencher avec un payload de test (le dernier reçu, ou input manuel via Tools → Basic Trigger).

### Test environments isolés

Pour un scénario qui écrit dans Stripe :
- Scénario WIP avec connection Stripe **Test** (pas Live).
- Une fois validé, dupliquer + remplacer la connection.

### Suite de payloads tests

Stocker dans Tools → Data Structures des payloads exemples typés. Module `util:BasicTrigger` les rejoue à la demande.

### CI/CD léger

Script externe qui :
1. Fetch les blueprints prod via API.
2. Compare au blueprint local (git).
3. Si diff non voulu → alerte.

```bash
# Dans un cron Linux
for SCENARIO_ID in $PROD_IDS; do
  curl ".../scenarios/$SCENARIO_ID/blueprint" \
    | jq -S '.response.blueprint' \
    > current/$SCENARIO_ID.json
done

git diff --exit-code current/ || \
  curl -X POST $SLACK_WEBHOOK -d '{"text":"Scénarios prod ont changé hors git !"}'
```

---

## 9. Patterns de design qui marchent

### Pattern : enrichissement progressif

Trigger → fetch source data → enrichir step 1 → enrichir step 2 → action finale. Chaque enrichissement est optionnel (Resume avec fallback).

### Pattern : queue + worker

Plusieurs sources (webhooks variés) écrivent dans un Data Store ou Airtable "queue table". Un seul scénario worker (sequential, schedule chaque 5 min) drain la queue. Avantage : centraliser la logique, throttling unifié, retry centralisé.

### Pattern : fan-out

Un trigger éclate en N actions parallèles via Router (toutes les routes activées sans filter). Risque : pas de coordination. Utiliser un Converger en aval si on doit attendre toutes les routes.

### Pattern : circuit breaker (voir `07-webhooks-datastores.md`)

Pour gérer un service externe instable.

---

## 10. Anti-patterns à éviter

### Itérer pour tout transformer en string

Mauvais :
```
Iterator → Compose String → Aggregator
```

Bon :
```
Compose String avec map() inline : {{join(map(1.users; "email"); ",")}}
```

Économie : N+2 ops → 1 op.

### HTTP module pour ce qu'un connecteur fait nativement

Si Make a un module Airtable, l'utiliser. Le HTTP module n'apporte un gain que si le connecteur natif est limité.

### Filter à false silencieux dans un Router

Toujours activer la route fallback `:else:` (voir 02 et 05). Sans elle, les bundles non matchés disparaissent.

### Pas d'error handler en prod

Le défaut Make = scénario désactivé après N erreurs consécutives. En prod, c'est une perte de service. Toujours avoir au moins Resume sur les modules tolérants à l'erreur.

### Custom Variables remplies en clair par le pipeline

Une Custom Variable est globale et accessible en lecture par tous les modules. Si elle contient un secret runtime, le secret est readable dans les logs. Pour les secrets dynamiques, préférer les data stores avec confidential mode + connection chiffrée.

### Polling fréquent sans nécessité

Un Watch records toutes les 15 min sur une table qui change 1× / jour = waste. Soit étirer l'intervalle, soit basculer en webhook (Airtable Automations + webhook custom).

### Modifier le blueprint en parallèle UI/API

Si un dev humain édite dans l'UI pendant qu'un script API push, last-write-wins → perte d'édits. Coordonner par convention (lock simple via Custom Variable, ou règles d'équipe).

---

## 11. Output style pour les specs (rappel)

Quand on conçoit un scénario, livrer :

1. **Trigger** : module + connection + paramètres.
2. **Modules en ordre** : numérotés, mappings clés, filters.
3. **Error handling** : qui a un handler, quelle directive.
4. **Scheduling** : intervalle ou URL webhook.
5. **Estimation ops** : nominal et pic.
6. **Edge cases** : 2-3 minimum (vide, dupliqué, payload mal formé).
7. **Blueprint JSON** : si demandé explicitement.
