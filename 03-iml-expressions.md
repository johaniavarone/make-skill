# IML — Integromat Markup Language

IML est le langage d'expression de Make. Syntaxe mustache-like (`{{...}}`) avec évaluation JavaScript-like. Présent partout où on peut mapper une valeur dans un module.

## 1. Syntaxe de base

```
{{expression}}
```

Tout ce qui est entre `{{` et `}}` est évalué. Hors de ces délimiteurs, c'est du texte brut.

```
Bonjour {{1.firstName}}, ton solde est de {{2.balance}}€.
```

Génère par exemple : `Bonjour Marc, ton solde est de 142.50€.`

---

## 2. Référence aux modules précédents

Les modules ont un ID numérique (visible dans le blueprint, pas dans l'UI directement). Pour accéder à leur output :

```
{{1.body}}              ← output du module ID 1, champ "body"
{{2.data.name}}         ← accès profond, dot notation
{{3.items[1].email}}    ← premier élément d'un array (1-based !)
{{3.items[-1].email}}   ← dernier élément
```

**Critique** : Make utilise une indexation **1-based** pour les arrays, contrairement à JavaScript (0-based). `{{1.items[1]}}` = premier item.

`-1` accède au dernier, `-2` à l'avant-dernier, etc.

### Accès dynamique avec `get()`

Quand le nom du champ est dans une variable :

```
{{get(1.body; "user_name")}}
{{get(1.body; parameters.fieldName)}}
{{get(1.body; if(1.lang == "fr"; "name_fr"; "name_en"))}}
```

Indispensable quand le nom est calculé. `.notation` ne marche que pour les noms statiques.

### Bracket notation

```
{{1.body["complex name with spaces"]}}
{{1.body[parameters.dynamicKey]}}
```

---

## 3. Opérateurs

### Algébriques

```
{{1.a + 1.b}}
{{1.a - 1.b}}
{{1.a * 1.b}}
{{1.a / 1.b}}
{{1.a % 1.b}}     ← modulo
```

### Égalité (toujours strictes)

```
{{1.a == 1.b}}    ← équivalent à === en JS
{{1.a != 1.b}}    ← équivalent à !==
```

`=` (un seul signe) est aussi accepté comme synonyme de `==`. Pas de loose equality (`==` JS).

### Relationnels

```
{{1.a < 1.b}}
{{1.a <= 1.b}}
{{1.a > 1.b}}
{{1.a >= 1.b}}
```

### Logiques

```
{{1.a && 1.b}}
{{1.a || 1.b}}
{{!1.a}}
```

Court-circuit comme en JS : `false && X` ne calcule pas X.

### String concatenation

L'opérateur `+` concatène si l'un des deux opérandes est string :

```
{{"Bonjour " + 1.firstName}}
{{1.firstName + " " + 1.lastName}}
```

Mais préférer la concaténation par juxtaposition dans les templates :

```
Bonjour {{1.firstName}} {{1.lastName}}
```

---

## 4. Conditionnelles

### `if(condition; valueIfTrue; valueIfFalse)`

```
{{if(1.amount > 100; "VIP"; "Standard")}}
{{if(contains(1.email; "@"); 1.email; "invalid")}}
```

### `ifempty(value; fallback)`

Retourne `value` si elle n'est ni `null`, ni `undefined`, ni `""`. Sinon retourne `fallback`.

```
{{ifempty(1.optional_phone; "N/A")}}
{{ifempty(get(1.body; "city"); "Unknown")}}
```

**Différence avec `||` JS** : `ifempty` traite `""` (string vide) comme empty, pas `||` qui traite seulement `null`/`undefined`/`false`/`0`/`""`/`NaN`. Pour Make, `ifempty` est plus prévisible que `||`.

### `switch(expression; v1; r1; v2; r2; ...; [else])`

```
{{switch(1.status; "paid"; "✅"; "pending"; "⏳"; "failed"; "❌"; "?")}}
```

Évalue `expression`, retourne le premier `r` dont le `v` correspond. Le dernier argument (sans paire) est l'else optionnel.

### Ternaire avec opérateur (alternatif)

Make accepte aussi la syntaxe ternaire JS dans certaines versions :

```
{{1.amount > 100 ? "VIP" : "Standard"}}
```

Mais `if(...)` est canonique et marche partout.

---

## 5. Strings dans les expressions

### Délimiter avec `"` ou `'`

```
{{"hello world"}}
{{'hello world'}}
```

### Échappement

```
{{"He said \"hi\""}}
{{'It\'s OK'}}
```

### Concaténation

```
{{"Total: " + 1.amount + "€"}}
```

### Templates multi-ligne

Pas de support natif des template strings (backticks). Pour multi-ligne, utiliser `\n` :

```
{{"Line 1\nLine 2\nLine 3"}}
```

Ou la fonction `join()` sur un array :

```
{{join(1.lines; "\n")}}
```

---

## 6. Empty string constant

```
{{emptystring}}
```

Renvoie `""`. Utile pour forcer un champ à vide explicitement (différent de "ne pas envoyer").

---

## 7. Variables système

### `now`

```
{{now}}              ← timestamp courant (Date type)
{{formatDate(now; "YYYY-MM-DD")}}
```

### `timestamp`

```
{{timestamp}}        ← Unix timestamp en secondes
```

### Random

```
{{random}}           ← float random entre 0 et 1
{{floor(random * 100)}}  ← int random 0-99
```

### Pi, E

```
{{pi}}               ← 3.14159...
{{e}}                ← 2.71828...
```

---

## 8. Accès collections vs arrays

### Collection (object)

```json
{ "name": "Alice", "age": 30 }
```

```
{{1.user.name}}      ← "Alice"
{{1.user["age"]}}    ← 30
```

### Array

```json
[ {"name": "Alice"}, {"name": "Bob"} ]
```

```
{{1.users[1].name}}  ← "Alice" (1-based !)
{{1.users[2].name}}  ← "Bob"
{{1.users[-1].name}} ← "Bob" (dernier)
{{length(1.users)}}  ← 2
```

### Array de primitives

```json
["a", "b", "c"]
```

```
{{1.tags[1]}}        ← "a"
{{join(1.tags; ", ")}} ← "a, b, c"
```

### Quand utiliser `.` vs `[]`

- `.` pour les noms statiques connus à l'écriture : `1.user.email`
- `[]` pour les noms dynamiques ou avec caractères spéciaux : `1.body["weird key"]`, `1.users[1+1]`
- `get()` pour les noms qui viennent d'une variable : `get(1.body; parameters.field)`

---

## 9. Fonctions IML built-in

Cinq catégories. Référence détaillée dans `04-functions-reference.md`.

- **General** : `if`, `ifempty`, `switch`, `get`, `omit`, `pick`, `length`, `keys`, `values`, `merge`
- **Math** : `abs`, `ceil`, `floor`, `round`, `sum`, `average`, `min`, `max`, `formatNumber`, `parseNumber`, `random`
- **Text/Binary** : `length`, `upper`, `lower`, `capitalize`, `startCase`, `trim`, `contains`, `startsWith`, `endsWith`, `replace`, `substring`, `split`, `replaceAll`, `ascii`, `base64`, `decodeURL`, `encodeURL`, `escapeHTML`, `escapeMarkdown`, `md5`, `sha1`, `sha256`, `sha512`, `toBinary`, `toString`, `stripHTML`
- **Date/Time** : `now` (variable), `timestamp` (variable), `formatDate`, `parseDate`, `addDays`, `addHours`, `addMinutes`, `addMonths`, `addSeconds`, `addYears`, `setDay`, `setDate`, `setMonth`, `setYear`, `setHour`, `setMinute`, `setSecond`, `dateDifference`
- **Array** : `add`, `contains`, `deduplicate`, `distinct`, `first`, `flatten`, `join`, `keys`, `last`, `length`, `map`, `merge`, `remove`, `reverse`, `shuffle`, `slice`, `sort`, `toArray`, `toCollection`

---

## 10. Pièges courants IML

### Type coercion silencieuse

```
{{1.amount + 1}}     ← si 1.amount est string "10", résultat = "101" (concat) pas 11
```

Toujours `parseNumber` quand on attend un nombre depuis une string :

```
{{parseNumber(1.amount; ".") + 1}}
```

### `null` vs `""` vs `undefined`

Les trois ne sont pas équivalents. `ifempty` les traite tous comme empty, mais `==` non.

```
{{1.value == ""}}     ← true seulement si exactement string vide
{{ifempty(1.value; "default")}}  ← couvre les 3 cas
```

### Espaces dans les expressions

Les espaces entre `{{` et l'expression sont autorisés mais peuvent être normalisés. Préférer compact : `{{1.field}}` plutôt que `{{ 1.field }}`.

### Fonctions case-insensitive (parfois)

`upper`, `Upper`, `UPPER` sont équivalents. Mais par convention, écrire en camelCase : `formatDate`, `parseNumber`.

### Référencer un module qui n'a pas encore tourné

Si module 5 référence `{{3.value}}` et que 3 n'a produit aucun bundle (filter exclu, erreur ignorée), 5 reçoit `undefined` pour ce champ. `ifempty` est ton ami.

### Quotes dans les arguments de fonction

Le séparateur d'argument est `;` (semicolon), pas `,`. Comma fonctionne dans certaines fonctions mais semicolon est le standard.

```
{{ifempty(1.value; "default")}}    ← canonique
{{ifempty(1.value, "default")}}    ← peut casser dans certaines apps
```

### Quoter les strings dans les expressions

```
{{contains(1.email; "@")}}           ← OK
{{contains(1.email; @)}}             ← cassé : @ pas une variable
{{contains(1.tag; 1.expectedTag)}}   ← OK : compare deux variables
```

---

## 11. Debugging IML

### Approche 1 : Set Variable de diagnostic

Insérer un module **Tools → Set Multiple Variables** entre deux modules pour stocker l'évaluation intermédiaire et l'inspecter dans les logs d'exécution.

```
debug_email = {{lower(trim(1.email))}}
debug_age   = {{parseNumber(1.age; ".")}}
debug_isVIP = {{1.amount > 100}}
```

Puis lancer l'execution, ouvrir le bundle de Set Variables, lire les valeurs résolues.

### Approche 2 : Module HTTP avec body de debug

```
URL: https://webhook.site/...
Body: {"raw": "{{1.value}}", "after_trim": "{{trim(1.value)}}", "len": "{{length(1.value)}}"}
```

Webhook.site (gratuit) reçoit et affiche.

### Approche 3 : Note dans le designer

Make permet d'ajouter une note (clic droit → Add note). Y inscrire l'expression attendue pour comparer avec ce qui sort.

---

## 12. IML dans les Custom Apps

Dans le contexte des Custom Apps (Apps Editor), IML s'utilise dans :
- `base.json` : `directives` runtime
- Module `communication` : `url`, `headers`, `qs`, `body`, `response`
- Module `interface` : output schema (mais avec accès limité, pas runtime)

Référence : `references/11-custom-apps.md`.

**Différence cruciale** : dans les Custom Apps IML, on a accès à `parameters.<name>`, `body`, `headers`, `statusCode`, `connection`, `temp`, `data` — variables système qui n'existent pas dans le contexte scenario classique.

---

## 13. Custom IML functions

Plan Enterprise uniquement. Permet d'écrire ses propres fonctions JavaScript réutilisables :

```javascript
function arrayToObject(arr) {
  return arr.reduce((acc, item) => {
    acc[item.key] = item.value;
    return acc;
  }, {});
}
```

Disponibles ensuite dans toutes les expressions IML du team :

```
{{arrayToObject(1.labels)}}
```

Limites : timeout par appel ~3s, total <30s par bundle, pas d'I/O réseau, seulement JavaScript ES6 + `Buffer`.

Voir `references/11-custom-apps.md` pour la création.
