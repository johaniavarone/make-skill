# Migration Make ↔ n8n

Ce fichier sert de référence rapide quand un scénario Make doit être porté vers n8n (ou inversement). Make et n8n partagent la même philosophie de workflow visuel, mais leurs primitives, leur syntaxe d'expression et leur modèle d'itération diffèrent suffisamment pour qu'une migration littérale produise des bugs subtils. Cette section couvre les correspondances de modules, la traduction des expressions IML vers les expressions JavaScript de n8n, et les pièges classiques.

## Quand migrer

La migration Make → n8n se justifie principalement pour trois raisons : besoin de self-hosting (n8n est open source, Make ne l'est pas), volume d'opérations qui rend Make économiquement non viable (n8n facture à l'exécution de workflow, pas au module), ou besoin d'écrire du code arbitraire dans le workflow (le node Code de n8n est plus puissant que les Make Functions). À l'inverse, n8n → Make se justifie si on a besoin d'une UI plus accessible aux non-développeurs, d'un catalogue d'apps natives plus large, ou d'une plateforme managée sans ops.

## Mapping des modules

Le tableau ci-dessous donne les correspondances les plus fréquentes. Tous les modules ne se traduisent pas un-pour-un : certaines primitives Make (notamment Iterator et Aggregator) demandent une réécriture conceptuelle.

| Make | n8n | Notes |
|------|-----|-------|
| `gateway:CustomWebHook` | Webhook | n8n supporte aussi les webhooks de test vs production via deux URLs distinctes |
| `gateway:WebhookRespond` | Respond to Webhook | Doit être branché à un Webhook node configuré en mode `responseMode: responseNode` |
| `mailhook` | Email Trigger (IMAP) | Pas de mailhook adresse @hook.eu1.make.com en n8n, on utilise IMAP ou un service tiers |
| `builtin:BasicRouter` | Switch ou If | Switch pour multi-branches, If pour binaire ; pas d'équivalent direct du `:else:` de Make |
| `builtin:BasicFeeder` (Iterator) | Split In Batches ou exécution naturelle | n8n itère naturellement sur un array d'items en sortie d'un node, l'Iterator explicite est rarement nécessaire |
| `builtin:BasicAggregator` (Array Aggregator) | Aggregate ou Merge | Aggregate avec mode `aggregateAllItemData` reproduit le comportement Array Aggregator |
| `builtin:TextAggregator` | Aggregate (mode `concatenate`) ou Code node | Pour des séparateurs et conditions complexes, le Code node est plus simple |
| `http:ActionSendData` | HTTP Request | n8n a une UI plus riche pour les auth (predefined credentials types), même paramètres de base |
| `json:ParseJSON` | Pas nécessaire | n8n parse automatiquement le JSON dans les réponses HTTP ; pour parser une string JSON explicite, utiliser une Code node ou une Set expression `{{ JSON.parse($json.body) }}` |
| `json:CreateJSON` | Set node | Composer un objet en mappant les champs |
| `airtable2:*` | Airtable | Modules natifs équivalents avec credentials Airtable PAT |
| `openai-gpt-3:CreateCompletion` ou `openai-gpt-3:CreateChat` | OpenAI | Le node OpenAI de n8n couvre Chat, Completion, Embeddings, Image |
| `google:ActionSendEmail` | Gmail | Credentials OAuth2 Google |
| `google-sheets:*` | Google Sheets | Modules équivalents, opérations légèrement réorganisées |
| `tools:SetVariable` | Set | Le node Set en n8n écrit les variables sur le bundle courant |
| `tools:GetVariable` | Référence directe `$('Node Name').item.json.varName` | Pas besoin de module Get explicite |
| `util:FunctionSleep` | Wait | Wait node avec mode `waitTill: After Time Interval` |
| `util:FunctionThrow` | Stop and Error | Le node Stop and Error force un arrêt avec message |
| `text-parser:ParseRegExp` | Code node ou expression | Pas de node Text Parser dédié, utiliser regex en JS |
| Custom Webhooks Make | Workflow Trigger via webhook | Idem Make, mais URL de structure différente |

## Traduction des expressions IML → expressions n8n

Les expressions n8n utilisent JavaScript (mode `=` pour expressions, ou `{{ ... }}` dans les champs). Les références entre nodes sont nominales (par nom de node) et non par index numérique comme en Make.

| Make IML | n8n expression | Commentaire |
|----------|----------------|-------------|
| `{{1.body}}` | `{{ $('Webhook').item.json.body }}` | n8n utilise le nom du node ; pour le node précédent immédiat, `$json.body` |
| `{{2.id}}` | `{{ $('HTTP Request').item.json.id }}` | Toujours référencer par nom |
| `{{1.array[1]}}` | `{{ $json.array[0] }}` | n8n est 0-based comme JS, Make est 1-based |
| `{{1.array.length}}` | `{{ $json.array.length }}` | Identique |
| `{{lower(1.email)}}` | `{{ $json.email.toLowerCase() }}` | Méthode JS native |
| `{{upper(1.name)}}` | `{{ $json.name.toUpperCase() }}` | |
| `{{capitalize(1.name)}}` | `{{ $json.name.charAt(0).toUpperCase() + $json.name.slice(1).toLowerCase() }}` | Pas de capitalize built-in en JS |
| `{{trim(1.text)}}` | `{{ $json.text.trim() }}` | |
| `{{contains(1.text; "foo")}}` | `{{ $json.text.includes('foo') }}` | |
| `{{startsWith(1.text; "foo")}}` | `{{ $json.text.startsWith('foo') }}` | |
| `{{replace(1.text; "a"; "b")}}` | `{{ $json.text.replace('a', 'b') }}` | Premier match seulement, comme Make |
| `{{replaceAll(1.text; "a"; "b")}}` | `{{ $json.text.replaceAll('a', 'b') }}` ou `replace(/a/g, 'b')` | |
| `{{substring(1.text; 0; 5)}}` | `{{ $json.text.substring(0, 5) }}` | |
| `{{split(1.text; ",")}}` | `{{ $json.text.split(',') }}` | Retourne un array |
| `{{join(1.array; ", ")}}` | `{{ $json.array.join(', ') }}` | |
| `{{length(1.text)}}` | `{{ $json.text.length }}` | Pour string ou array |
| `{{ifempty(1.x; "default")}}` | `{{ $json.x \|\| "default" }}` | Opérateur OR fait le job pour null/undefined/'' |
| `{{if(1.x = "yes"; "OK"; "KO")}}` | `{{ $json.x === "yes" ? "OK" : "KO" }}` | Ternaire JS |
| `{{switch(1.s; "a"; 1; "b"; 2; 0)}}` | Pas direct, utiliser if-else chain ou un objet de mapping `{{ ({a:1,b:2})[$json.s] \|\| 0 }}` | |
| `{{now}}` | `{{ $now }}` | Variable globale n8n (Luxon DateTime) |
| `{{timestamp}}` | `{{ $now.toMillis() }}` | |
| `{{formatDate(now; "YYYY-MM-DD")}}` | `{{ $now.toFormat('yyyy-MM-dd') }}` | n8n utilise Luxon, tokens différents (`yyyy` minuscule pour année 4 chiffres) |
| `{{parseDate(1.date; "YYYY-MM-DD")}}` | `{{ DateTime.fromFormat($json.date, 'yyyy-MM-dd') }}` | Il faut importer DateTime en haut de l'expression |
| `{{addDays(now; 7)}}` | `{{ $now.plus({days: 7}) }}` | API Luxon |
| `{{sum(1.array)}}` | `{{ $json.array.reduce((a,b) => a+b, 0) }}` | Pas de sum built-in en JS |
| `{{max(1.array)}}` | `{{ Math.max(...$json.array) }}` | |
| `{{round(1.x; 2)}}` | `{{ Math.round($json.x * 100) / 100 }}` | |
| `{{md5(1.text)}}` | Demande un Code node avec `crypto.createHash('md5').update(text).digest('hex')` | Pas built-in dans les expressions |
| `{{base64(1.text)}}` | `{{ Buffer.from($json.text).toString('base64') }}` | |
| `{{encodeURL(1.url)}}` | `{{ encodeURIComponent($json.url) }}` | |

## Différences conceptuelles importantes

**Itération naturelle vs explicite.** En Make, chaque module traite un bundle à la fois et un Iterator est nécessaire pour exploser un array en bundles individuels. En n8n, dès qu'un node retourne plusieurs items, le node suivant s'exécute automatiquement une fois par item. Cela rend l'Iterator explicite rarement nécessaire en n8n, et l'Aggregator est remplacé par un Aggregate node ou simplement par le retour du résultat sans aggregation (chaque item poursuit son chemin).

**Routes vs branches.** Le Router de Make distribue chaque bundle à toutes les routes dont le filtre passe (ou à `:else:` si aucun ne passe). Le Switch de n8n ne distribue qu'à une seule branche par item (la première qui match). Pour reproduire le comportement Make où plusieurs branches peuvent traiter le même bundle, il faut dupliquer le node ou utiliser If en parallèle.

**Credentials.** Make stocke les connexions au niveau du compte ou de l'équipe, n8n les stocke comme `credentials` au niveau de l'instance. Lors de la migration, toutes les connexions doivent être recréées : aucun export ne contient les secrets. Documenter à l'avance la liste des credentials à provisionner économise du temps.

**Error handling.** Make a 5 directives explicites (Break/Resume/Rollback/Commit/Ignore) attachables à chaque module. n8n a un modèle plus simple : `continueOnFail: true` au niveau du node fait passer l'erreur dans la sortie du node (qu'on peut traiter avec un If node sur `$json.error`), et un `Error Trigger` workflow capture les erreurs des autres workflows. Le concept d'Incomplete Executions Queue n'a pas d'équivalent direct.

**JSON parsing.** En Make, parser une string JSON nécessite un module `json:ParseJSON` explicite. En n8n, les réponses HTTP sont déjà parsées en objet JS par défaut, et pour parser une string, une expression `{{ JSON.parse($json.body) }}` suffit dans un Set node.

**Webhook URL et test mode.** Make webhooks ont une URL stable du jour de leur création. n8n distingue webhook de test (URL différente, écoute uniquement quand le workflow est ouvert dans l'éditeur en mode test) et webhook de production (active uniquement si le workflow est activé). Il faut donc activer le workflow n8n explicitement pour que la production URL réponde.

## Gotchas migration

**Index 1-based vs 0-based.** Toute expression qui accède à un élément d'array par index doit être convertie : `array[1]` en Make est le premier élément, `array[0]` en n8n est le premier. Une migration littérale produit du off-by-one silencieux.

**Connections re-link obligatoire.** L'export blueprint Make contient un `__IMTCONN__: 1234567` qui référence une connexion inexistante en n8n. Toutes les connexions doivent être recréées et re-mappées. Idem dans l'autre sens.

**Pas d'équivalent à `:else:` Router.** Si le scénario Make repose sur la logique `:else:` du Router pour traiter les bundles qui ne matchent aucune route, il faut créer une branche explicite avec une condition négative en n8n (Switch en mode `Fallback Output: Extra` ou un If avec la négation des conditions précédentes).

**Aggregators avec sources multiples.** Si un Aggregator Make recevait des bundles de plusieurs Iterators imbriqués, la migration en n8n demande un Merge ou Aggregate avec une logique explicite, parfois plusieurs nodes en cascade.

**Confidential et data loss.** Make a des flags `confidential` (logs masqués) et `dataloss` (continue malgré erreurs critiques). n8n n'a pas d'équivalent direct ; il faut implémenter manuellement la non-journalisation (en évitant les logs verbeux côté node) et la tolérance aux erreurs (`continueOnFail`).

**Custom IML functions vs Code node.** Les custom IML functions Make (Enterprise) doivent être réécrites en Code node JavaScript en n8n. Le Code node est plus puissant (accès à des libs npm via `npm install` sur l'instance n8n), donc cette migration libère plutôt qu'elle ne contraint.

**Sub-scenarios vs sub-workflows.** Le pattern Make « scénario A appelle scénario B via webhook » se traduit naturellement en n8n par un node `Execute Workflow` qui appelle un autre workflow par ID. C'est plus propre car typé et synchrone par défaut.

## Outil et workflow de migration

Il n'existe pas d'outil officiel de migration automatique Make → n8n. La méthode la plus efficace consiste à :

1. Exporter le blueprint Make en JSON via `GET /scenarios/{id}/blueprint`.
2. Lister tous les modules utilisés et leurs paramètres clés.
3. Créer le workflow n8n vide, ajouter chaque node un par un en remplissant les paramètres équivalents.
4. Recréer toutes les connexions/credentials à la main.
5. Tester avec un payload identique côté Make et n8n, comparer les outputs bundle par bundle.
6. Pour les expressions complexes, traduire en JavaScript en s'aidant du tableau ci-dessus.
7. Activer le workflow n8n et désactiver le scénario Make en parallèle pendant 24-48h pour valider.

Pour les migrations à grande échelle (50+ scénarios), un script Node qui lit les blueprints Make et génère des squelettes de workflows n8n peut accélérer le travail répétitif, mais la validation manuelle reste indispensable.
