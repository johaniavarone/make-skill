# Custom Apps Make.com

Les Custom Apps permettent de créer ses propres connecteurs Make en empaquetant des appels d'API sous forme de modules réutilisables. Une fois publiés (privé ou public), ils s'utilisent comme n'importe quel autre app du catalogue. Ce fichier couvre l'architecture des Custom Apps, le langage IML (Integromat Markup Language) qui les anime, le workflow de développement local, et le processus de review pour publication publique.

## Quand créer une Custom App

Une Custom App se justifie quand on a besoin de réutiliser le même service tiers dans plusieurs scénarios et qu'il n'existe pas de connecteur officiel, ou que le connecteur officiel ne couvre pas les endpoints nécessaires. Pour un usage ponctuel dans un seul scénario, le module HTTP générique suffit. La rentabilité d'une Custom App apparaît à partir de 3-5 scénarios qui consomment la même API, ou dès qu'on veut distribuer le connecteur à une équipe ou à des clients.

Les cas typiques sont les API internes d'entreprise (CRM maison, ERP propriétaire, microservices métier), les API tierces récentes pas encore couvertes par Make, ou les fournisseurs de niche (logiciels métier verticaux). Pour Johan, une Custom App pertinente serait par exemple un connecteur Pennylane si l'usage devient régulier, ou un connecteur YouSign avec les endpoints custom utilisés dans le CRM Agapè.

## Architecture d'une Custom App

Une Custom App se compose de cinq sections principales qu'on configure dans l'Apps Editor (UI Make) ou en local via VS Code :

**Base.** Configuration commune à tous les modules : URL de base de l'API, headers par défaut, gestion d'erreurs globale, logging. Tout ce qui se répète dans chaque module se factorise ici. Exemple typique : `baseUrl: "https://api.example.com/v1"`, headers `Authorization: Bearer {{connection.token}}`, `Content-Type: application/json`. La base est exprimée en IML et héritée par tous les modules.

**Connections.** Définit comment l'utilisateur s'authentifie auprès de l'API. Trois types principaux : OAuth 2.0 (le plus courant pour les APIs grand public), API key (champ secret stocké), et JWT (avec génération signée au moment de l'appel). Une Custom App peut déclarer plusieurs connections (par exemple OAuth pour les utilisateurs et API key pour les admins). La connection définit aussi l'éventuel test endpoint pour valider que les credentials marchent au moment de la création.

**Modules.** Les actions exposées dans l'éditeur de scénario. Quatre types : `action` (un appel CRUD ou autre), `search` (retourne plusieurs items, supporte le mode `Get all` ou `Limit n`), `trigger` (instant via webhook ou polling avec stockage de l'epoch), et `feeder` (alimente des arrays pour itération). Chaque module définit ses paramètres d'entrée (mappable depuis d'autres modules), son corps de requête, le mapping de la réponse vers les outputs exposés en aval, et son interface paramétrique (champs visibles dans l'éditeur).

**RPCs (Remote Procedure Calls).** Endpoints internes appelés au moment où l'utilisateur configure un module dans l'éditeur, typiquement pour peupler des dropdowns dynamiques. Exemple : un module « Send to Channel » a un dropdown `channelId` ; quand l'utilisateur ouvre le dropdown, un RPC appelle `GET /channels` et retourne la liste pour permettre la sélection. Les RPCs ne consomment pas d'opérations à l'exécution (ils tournent uniquement en mode édition).

**IML Functions.** Fonctions custom écrites en JavaScript pour les transformations spécifiques à l'app. Réservées au plan Enterprise (réactivation progressive après une vulnérabilité). Cas d'usage typiques : génération de signature HMAC pour API maison, parsing d'un format propriétaire, dérivation de tokens.

## IML dans les Custom Apps

L'IML utilisé dans les Custom Apps va plus loin que celui utilisé dans les scénarios : on peut composer des objets de configuration entiers (URL, qs, headers, body, response mapping) à partir de l'input utilisateur via des templates IML.

Exemple de communication module Action « Create Contact » côté communication :

```json
{
  "url": "/contacts",
  "method": "POST",
  "body": {
    "email": "{{parameters.email}}",
    "firstName": "{{parameters.firstName}}",
    "lastName": "{{parameters.lastName}}",
    "tags": "{{parameters.tags}}"
  },
  "response": {
    "output": {
      "id": "{{body.id}}",
      "email": "{{body.email}}",
      "createdAt": "{{body.created_at}}"
    }
  }
}
```

L'objet `parameters` contient les champs configurés par l'utilisateur dans l'éditeur, mappés depuis les bundles précédents. L'objet `body` dans la section response correspond au corps de la réponse HTTP retournée par l'API. Le mapping `output` traduit les champs de l'API en champs exposés en aval (renommage, transformation, normalisation).

L'interface paramétrique se définit dans la section `parameters` du module :

```json
[
  {
    "name": "email",
    "type": "email",
    "label": "Email",
    "required": true
  },
  {
    "name": "firstName",
    "type": "text",
    "label": "First name"
  },
  {
    "name": "lastName",
    "type": "text",
    "label": "Last name"
  },
  {
    "name": "tags",
    "type": "array",
    "label": "Tags",
    "spec": { "type": "text" }
  }
]
```

Les types disponibles couvrent les primitives (`text`, `number`, `boolean`, `date`, `email`, `url`), les structures (`array`, `collection`), les sélecteurs (`select` statique, `select` avec RPC), les credentials (`password`, masqué dans les logs), et le `buffer` pour les fichiers binaires.

## Custom IML Functions (Enterprise)

Quand les fonctions built-in IML ne suffisent pas, on peut écrire des fonctions JavaScript custom. La fonction est définie comme une simple `function` qui reçoit des paramètres et retourne une valeur, exécutée dans un sandbox isolé sans accès au filesystem ni au réseau.

```javascript
function generateHMACSignature(secret, payload) {
    const crypto = require('crypto');
    return crypto
        .createHmac('sha256', secret)
        .update(JSON.stringify(payload))
        .digest('hex');
}
```

Une fois définie, on l'utilise dans les modules :

```json
{
  "headers": {
    "X-Signature": "{{generateHMACSignature(connection.secret, parameters)}}"
  }
}
```

Limitations actuelles : disponible uniquement pour les comptes Enterprise (réactivation progressive après une vulnérabilité), pas d'accès aux modules npm arbitraires (seules quelques libs whitelistées comme `crypto`, `lodash`, `moment`), timeout de quelques secondes par exécution. Pour des transformations complexes, mieux vaut basculer la logique côté API ou utiliser un module HTTP avec un proxy maison.

## Workflow de développement

Deux approches possibles : Apps Editor en ligne ou développement local via VS Code.

**Apps Editor (UI).** Tout se fait dans le navigateur sur `apps.make.com/make`. Avantages : simple à démarrer, pas de setup, on voit immédiatement le rendu de l'interface utilisateur. Inconvénients : pas de versioning Git, pas d'auto-complétion riche, copier-coller de gros blocs JSON pénible. Recommandé pour l'exploration et les apps simples.

**VS Code local.** Make fournit une extension VS Code « Make Apps Editor » qui synchronise les fichiers de l'app avec le serveur Make. On édite chaque section dans des fichiers JSON séparés (`base.iml.json`, `connections/oauth2.iml.json`, `modules/createContact/communication.iml.json`, etc.), on commit dans Git, et on push vers Make. Avantages : versioning, diff, code review, collaboration. Recommandé pour toute app sérieuse qu'on prévoit de maintenir dans le temps.

Le workflow VS Code typique : `Sign in` à Make, créer ou linker une app, l'extension télécharge tous les fichiers dans un dossier local, on édite, on `Push` pour synchroniser vers le serveur Make, on teste dans un scénario, on commit dans Git. Chaque fichier IML est validé syntaxiquement à l'enregistrement.

## RPCs et UX dynamique

Un RPC se déclare comme un module light (pas exposé dans l'éditeur de scénario) qui retourne une liste pour peupler un select dynamique. Exemple :

```json
{
  "url": "/channels",
  "method": "GET",
  "response": {
    "iterate": "{{body.channels}}",
    "output": {
      "label": "{{item.name}}",
      "value": "{{item.id}}"
    }
  }
}
```

Puis dans le module qui consomme ce RPC :

```json
{
  "name": "channelId",
  "type": "select",
  "label": "Channel",
  "options": {
    "store": "rpc://listChannels"
  }
}
```

L'utilisateur voit un dropdown qui s'autopeuple à partir de l'API au moment où il ouvre le champ. Les RPCs supportent aussi la pagination (`response.pagination`) et le filtrage côté serveur via des paramètres passés depuis le module appelant. C'est l'un des plus gros différenciateurs UX entre une Custom App propre et un usage du module HTTP générique.

## App Review pour publication publique

Une Custom App peut être conservée privée (utilisable uniquement par le compte ou l'équipe qui l'a créée), partagée avec une organisation, ou soumise à publication publique sur le catalogue Make. La publication publique passe par un processus de review (App Review) géré par l'équipe Make.

Critères de review couvrants : qualité du code IML (pas de hardcoding de credentials, gestion d'erreurs propre, naming cohérent), couverture fonctionnelle (au minimum les opérations CRUD principales du service), qualité de l'UX (labels clairs, descriptions, dropdowns dynamiques quand pertinent), documentation (README, screenshots, lien vers la doc API), et accord avec les guidelines de design Make (icône à la bonne taille, palette compatible).

Le timing typique est de 2 à 6 semaines selon la file. Une app rejetée reçoit un feedback détaillé et peut être resoumise. Pour une app interne ou client, garder en privé est généralement plus simple.

## Patterns d'usage avancés

**Multi-region API.** Si l'API a plusieurs régions (eu/us/ap), exposer la région comme paramètre de connection et utiliser un IML pour construire l'URL : `baseUrl: "https://{{connection.region}}.api.example.com/v1"`. Cela évite de créer une app par région.

**Pagination automatique.** Pour les modules de type `search`, gérer la pagination côté Custom App via la section `pagination` qui définit comment extraire le cursor et l'envoyer au prochain appel. Make appellera l'endpoint en boucle jusqu'à épuisement, en consommant une opération par page.

**Webhooks dédiés.** Pour les triggers instant, déclarer une section `webhook` dans le module qui définit comment Make doit s'enregistrer auprès de l'API tierce (POST sur `/webhooks` avec l'URL Make en payload), et comment se désinscrire. Cela évite à l'utilisateur final de configurer manuellement le webhook côté provider.

**Refresh token OAuth.** Pour les connections OAuth, déclarer la section `refresh` qui définit comment échanger un refresh token contre un nouvel access token. Make appelle automatiquement ce flow quand un access token expire, transparent pour l'utilisateur.

**Rate limiting natif.** Si l'API a un rate limit connu, le déclarer dans `base.communication.options.throttling` permet à Make de pacer automatiquement les appels et d'éviter les 429. Plus propre que de gérer le throttling dans chaque scénario consommateur.

## Limitations à connaître

Pas d'accès au filesystem ni au réseau arbitraire dans les IML functions. Pas d'exécution de code Python ou autre langage. Pas de support natif des grands fichiers (le buffer est limité en taille, pour les uploads volumineux mieux vaut utiliser le module HTTP générique avec streaming). Pas de cache built-in (les RPCs sont rappelés à chaque ouverture de dropdown, sauf si on implémente du throttling côté API).

Le développement et la maintenance d'une Custom App sont un investissement non négligeable : la doc Make sur les Custom Apps est moins riche que la doc utilisateur, et les retours de l'App Review peuvent prendre du temps. Pour une utilisation occasionnelle d'une API, le module HTTP reste plus rapide à mettre en œuvre.
