# Fonctions built-in Make — Référence complète

Cinq catégories, chaque fonction documentée avec signature et exemple.

---

## 1. General functions

### `if(expression; valueIfTrue; valueIfFalse)`

```
{{if(1.amount > 100; "VIP"; "Standard")}}
```

### `ifempty(value; fallback)`

Retourne `value` si elle est non-empty (≠ null, undefined, ""). Sinon `fallback`.

```
{{ifempty(1.phone; "N/A")}}
```

### `switch(expression; v1; r1; v2; r2; ...; [else])`

```
{{switch(1.status; "paid"; "✅"; "pending"; "⏳"; "?")}}
```

### `get(object; path)`

Accès dynamique. `path` peut être string, expression, ou nombre.

```
{{get(1.body; "user.email")}}
{{get(1.body; parameters.fieldName)}}
{{get(1.array; 1+1)}}
```

### `omit(object; key1; key2; ...)`

Retourne l'object sans les clés listées.

```
{{omit(1.user; "password"; "secret")}}
```

### `pick(object; key1; key2; ...)`

Retourne l'object avec uniquement les clés listées.

```
{{pick(1.user; "id"; "email"; "name")}}
```

### `length(value)`

Nombre de caractères (string) ou d'items (array).

```
{{length(1.email)}}      ← 17
{{length(1.users)}}      ← 5
```

### `keys(object)`

Array des clés.

```
{{keys(1.user)}}         ← ["id", "email", "name"]
```

### `values(object)`

Array des valeurs.

```
{{values(1.user)}}       ← [42, "a@b.c", "Alice"]
```

### `merge(obj1; obj2; ...)`

Fusion d'objets. Les valeurs de droite écrasent celles de gauche.

```
{{merge(1.defaults; 1.overrides)}}
```

---

## 2. Math functions

### `abs(number)`

```
{{abs(-5)}}              ← 5
{{abs(3.7)}}             ← 3.7
```

### `ceil(number)`

Arrondi supérieur.

```
{{ceil(1.2)}}            ← 2
{{ceil(-1.2)}}           ← -1
```

### `floor(number)`

Arrondi inférieur.

```
{{floor(1.9)}}           ← 1
{{floor(-1.2)}}          ← -2
```

### `round(number; [decimals])`

Arrondi standard (nearest).

```
{{round(1.5)}}           ← 2
{{round(1.456; 2)}}      ← 1.46
```

### `sum(array)` ou `sum(v1; v2; ...)`

```
{{sum(1.amounts)}}
{{sum(10; 20; 30)}}      ← 60
```

### `average(array)` ou `average(v1; v2; ...)`

```
{{average(1.scores)}}
{{average(80; 90; 100)}} ← 90
```

### `min` / `max`

```
{{min(1.values)}}
{{max(10; 20; 5)}}       ← 20
```

### `formatNumber(number; decimals; [decSep]; [thousandsSep])`

```
{{formatNumber(1234567.89; 2)}}              ← "1,234,567.89"
{{formatNumber(1234567.89; 2; ","; " ")}}    ← "1 234 567,89"
{{formatNumber(0.5; 0)}}                     ← "1" (round half up)
```

### `parseNumber(text; decimalSeparator)`

```
{{parseNumber("1,234.56"; ".")}}             ← 1234.56
{{parseNumber("1.234,56"; ",")}}             ← 1234.56
{{parseNumber("$42.99"; ".")}}               ← 42.99 (strip non-numeric)
```

### `random`

Variable, pas fonction.

```
{{random}}               ← 0.7283... (float entre 0 et 1)
{{floor(random * 100)}}  ← int 0-99
```

### `pi`, `e`

Constantes.

---

## 3. Text/Binary functions

### Casing

```
{{upper("hello")}}                ← "HELLO"
{{lower("HELLO")}}                ← "hello"
{{capitalize("hello world")}}     ← "Hello world"
{{startCase("hello world")}}      ← "Hello World"
```

`startCase` = title case (chaque mot capitalisé). `capitalize` = première lettre seulement.

### Whitespace

```
{{trim("  hello  ")}}             ← "hello"
```

### Recherche

```
{{contains("hello world"; "world")}}    ← true
{{startsWith("hello"; "he")}}           ← true
{{endsWith("hello"; "lo")}}             ← true
{{indexOf("hello"; "ll")}}              ← 3 (1-based)
```

### Modification

```
{{replace("hello world"; "world"; "Make")}}   ← "hello Make"
{{replaceAll("a-b-c"; "-"; "_")}}             ← "a_b_c"
{{substring("hello"; 1; 3)}}                  ← "hel" (start; end exclusif, 1-based)
{{substring("hello"; 2)}}                     ← "ello" (jusqu'à la fin)
```

### Split / Join

```
{{split("a,b,c"; ",")}}           ← ["a", "b", "c"]
{{join(1.array; ", ")}}           ← "a, b, c"
```

### Length

```
{{length("hello")}}               ← 5
```

### Encoding

```
{{base64("Make")}}                       ← "TWFrZQ=="
{{toString(toBinary("TWFrZQ=="; "base64"))}}  ← "Make" (decode)
{{encodeURL("hello world")}}             ← "hello%20world"
{{decodeURL("hello%20world")}}           ← "hello world"
{{escapeHTML("<b>x</b>")}}               ← "&lt;b&gt;x&lt;/b&gt;"
{{escapeMarkdown("**bold**")}}           ← "\\*\\*bold\\*\\*"
```

### Hashing

```
{{md5("hello")}}                  ← "5d41402abc4b2a76b9719d911017c592"
{{sha1("hello")}}                 ← "aaf4c61d..."
{{sha256("hello")}}
{{sha512("hello")}}
```

### Diacritiques / ASCII

```
{{ascii("café résumé")}}                  ← "café résumé" (strip non-ASCII chars)
{{ascii("café résumé"; true)}}            ← "cafe resume" (remove diacritics)
```

### HTML stripping

```
{{stripHTML("<p>Hello <b>world</b></p>")}}   ← "Hello world"
```

### Conversions binaires

```
{{toBinary("hello"; "utf8")}}        ← Buffer
{{toString(buffer; "base64")}}       ← string
```

Encodings supportés : `utf8`, `ascii`, `base64`, `hex`, `latin1`, `ucs2`, `binary`.

---

## 4. Date/Time functions

### Variables date

```
{{now}}              ← Date type, timestamp courant en timezone scenario
{{timestamp}}        ← Unix timestamp en secondes (number)
```

### `formatDate(date; format; [timezone])`

Convertit une Date en string formatée.

```
{{formatDate(now; "YYYY-MM-DD")}}                         ← "2026-05-01"
{{formatDate(now; "DD/MM/YYYY HH:mm:ss")}}                ← "01/05/2026 14:30:00"
{{formatDate(1.created_at; "MMMM DD, YYYY"; "America/New_York")}}
```

### `parseDate(text; format; [timezone])`

Convertit une string en Date.

```
{{parseDate("2026-05-01"; "YYYY-MM-DD")}}
{{parseDate("01/05/2026"; "DD/MM/YYYY")}}
{{parseDate("1714572000"; "X")}}                          ← Unix timestamp en secondes
{{parseDate("1714572000123"; "x")}}                       ← Unix timestamp en ms
{{parseDate("2026-05-01 10:00"; "YYYY-MM-DD HH:mm"; "Europe/Paris")}}
```

### Add (immutable, retourne nouvelle date)

```
{{addYears(now; 1)}}
{{addMonths(now; 3)}}
{{addDays(now; 7)}}
{{addHours(now; -2)}}     ← soustraction
{{addMinutes(now; 30)}}
{{addSeconds(now; 90)}}
```

### Set (immutable, change un composant)

```
{{setYear(now; 2030)}}
{{setMonth(now; "January")}}    ← nom anglais accepté
{{setMonth(now; 1)}}            ← number 1-12
{{setDate(now; 1)}}             ← jour du mois
{{setDay(now; "Monday")}}       ← jour de la semaine (peut être nom ou number)
{{setHour(now; 9)}}
{{setMinute(now; 0)}}
{{setSecond(now; 0)}}
```

### Différence entre dates

Pas de fonction native `dateDifference` officielle stable, mais pattern :

```
{{round((1.endDate - 1.startDate) / 1000 / 60 / 60 / 24)}}
```

= différence en jours. Soustraire deux Date donne des millisecondes. Diviser pour obtenir l'unité voulue.

### Patterns courants

**Premier du mois courant** :
```
{{setDate(now; 1)}}
```

**Dernier du mois courant** :
```
{{addDays(setDate(addMonths(now; 1); 1); -1)}}
```

**Lundi de la semaine en cours** :
```
{{setDay(now; "Monday")}}
```

**Date dans 3 jours ouvrés** : il faut une logique custom (Make n'a pas de business-days natif). Workaround : module Tools → Sleep + boucle, ou IML function custom.

---

## 5. Tokens de date (parsing & formatting)

### Year/Month/Day

| Token | Output | Description |
|-------|--------|-------------|
| `YYYY` | 1970-9999 | 4-digit year |
| `YY` | 70-29 | 2-digit year |
| `Y` | -10000+ | year with sign |
| `Q` | 1-4 | quarter |
| `Qo` | 1st-4th | quarter ordinal |
| `M` | 1-12 | month |
| `Mo` | 1st-12th | month ordinal |
| `MM` | 01-12 | month padded |
| `MMM` | Jan-Dec | month abbrev |
| `MMMM` | January-December | month full |
| `D` | 1-31 | day of month |
| `Do` | 1st-31st | day ordinal |
| `DD` | 01-31 | day padded |
| `DDD` | 1-365 | day of year |
| `DDDD` | 001-365 | day of year padded |

### Week

| Token | Output | Description |
|-------|--------|-------------|
| `ddd` | Mon | weekday short |
| `dddd` | Monday | weekday full |
| `gggg` | 2026 | ISO week year |
| `ww` | 01-53 | ISO week number |
| `e` | 1-7 | ISO day of week |

### Time

| Token | Output | Description |
|-------|--------|-------------|
| `H` | 0-23 | hour (24h) |
| `HH` | 00-23 | hour padded |
| `h` | 1-12 | hour (12h) |
| `hh` | 01-12 | hour padded |
| `mm` | 00-59 | minutes |
| `ss` | 00-59 | seconds |
| `SSS` | 000-999 | ms |
| `A` | AM/PM | meridiem |
| `a` | am/pm | meridiem lower |
| `Z` | +02:00 | timezone offset |
| `ZZ` | +0200 | timezone offset compact |
| `X` | 1714572000 | Unix timestamp (s) |
| `x` | 1714572000123 | Unix timestamp (ms) |

### Format ISO 8601 (standard interne Make)

```
2026-05-01T12:36:35.000Z
```

`T` sépare date et heure. `Z` = UTC. C'est le format que Make utilise pour stocker/passer les dates.

---

## 6. Array functions

### `add(array; v1; v2; ...)`

Ajoute des éléments à un array.

```
{{add(1.tags; "new"; "promo")}}
```

### `contains(array; value)`

```
{{contains(1.roles; "admin")}}
```

### `deduplicate(array)`

Supprime les doublons (sur primitives).

```
{{deduplicate(1.emails)}}
```

### `distinct(array; [key])`

Pour arrays d'objects : dédupe sur la valeur d'une clé.

```
{{distinct(1.contacts; "email")}}
{{distinct(1.users; "address.city")}}    ← dot notation pour nested
```

### `first(array)`, `last(array)`

```
{{first(1.results)}}
{{last(1.history)}}
```

### `flatten(array)`

Aplatit un array imbriqué.

```
{{flatten(1.matrix)}}    ← [[1,2],[3,4]] → [1,2,3,4]
```

### `join(array; separator)`

```
{{join(1.tags; ", ")}}
{{join(map(1.users; "email"); ";")}}    ← liste d'emails séparés par ;
```

### `length(array)`

Voir General.

### `map(array; key; [filterKey]; [filterValues])`

**La fonction array la plus puissante.** Extrait une valeur de chaque item, avec filter optionnel.

```
{{map(1.contacts; "email")}}
{{map(1.contacts; "email"; "type"; "work,personal")}}
```

Le 2e exemple : extrait `email` mais seulement pour les contacts dont `type` vaut `work` OU `personal`.

### `merge(array1; array2; ...)`

Concatène plusieurs arrays.

```
{{merge(1.list1; 1.list2)}}
```

### `remove(array; v1; v2; ...)`

Retire des valeurs (primitives only).

```
{{remove(1.tags; "draft"; "test")}}
```

### `reverse(array)`

```
{{reverse(1.history)}}
```

### `shuffle(array)`

Mélange aléatoire.

```
{{shuffle(1.contestants)}}
```

### `slice(array; start; [end])`

Sous-array. **0-based** ici (exception !).

```
{{slice(1.items; 0; 5)}}    ← 5 premiers
{{slice(1.items; -3)}}      ← 3 derniers
```

### `sort(array; [order]; [key])`

```
{{sort(1.numbers)}}                          ← ascending par défaut
{{sort(1.numbers; "desc")}}
{{sort(1.contacts; "asc"; "name")}}
{{sort(1.contacts; "asc ci"; "name")}}       ← case-insensitive
{{sort(1.contacts; "desc"; "address.city")}} ← nested key
```

Orders : `asc`, `desc`, `asc ci`, `desc ci`.

### `toArray(collection)`

Convertit un object en array de paires `{key, value}`.

```
{{toArray(1.user)}}    ← [{key:"id",value:42}, {key:"email",value:"a@b.c"}]
```

### `toCollection(array; keyField; valueField)`

Inverse de `toArray`.

```
{{toCollection(1.pairs; "key"; "value")}}
```

---

## 7. Patterns d'usage avancés

### Filtrer un array d'objects en une expression

Pas de `filter()` direct, mais `map()` avec filter remplit le rôle :

```
{{map(1.users; "id"; "active"; "true")}}    ← IDs des users actifs
```

### Compter les items qui matchent

```
{{length(map(1.users; "id"; "status"; "active"))}}
```

### Top N par tri + slice

```
{{slice(sort(1.scores; "desc"); 0; 10)}}    ← top 10 scores
```

### Concaténer des emails par catégorie

```
{{join(map(1.contacts; "email"; "consent"; "yes"); ";")}}
```

### Grouper par clé (workaround sans groupBy natif)

Pas natif en IML. Utiliser un Array Aggregator avec `groupBy` à la place du module agrégateur.

### Date du jour en string sans heure

```
{{formatDate(now; "YYYY-MM-DD")}}
```

### Date il y a X jours, formatée

```
{{formatDate(addDays(now; -7); "YYYY-MM-DD")}}
```

### Vérifier qu'une string ressemble à un email (validation simple)

```
{{contains(1.email; "@") && contains(1.email; ".")}}
```

Pour validation rigoureuse, utiliser un module Text Parser → Match Pattern avec regex.

### Conversion string en number en gardant les float

```
{{parseNumber(1.priceString; ".")}}
```

### Vider un tableau si toutes les valeurs sont vides

```
{{if(length(remove(1.tags; emptystring)) == 0; emptystring; 1.tags)}}
```

---

## 8. Fonctions custom (Enterprise)

Plan Enterprise uniquement, dans **Team → Functions** dans l'UI Make.

```javascript
function arrayToObject(arr) {
  return arr.reduce((acc, item) => {
    acc[item.key] = item.value;
    return acc;
  }, {});
}
```

Limites :
- Total timeout par invocation : ~3s
- Pas d'I/O (pas de `fetch`, pas de `require`)
- Seulement JS ES6 + objet `Buffer`
- Visibilité : team (admin pour créer, member pour utiliser)

Bonnes pratiques :
- Toujours `return`. Pas de side-effect possible.
- Tester en isolé avec console.log via le DevTool de Make.
- Documenter les params en commentaires JSDoc.
- Versionner via Make Grid si disponible.

Exemples utiles :

```javascript
// arrayToCSV
function arrayToCSV(arr, sep = ";") {
  if (!arr || arr.length === 0) return "";
  const keys = Object.keys(arr[0]);
  const header = keys.join(sep);
  const rows = arr.map(o => keys.map(k => o[k]).join(sep));
  return [header, ...rows].join("\n");
}

// hash dictionnaire
function indexBy(arr, key) {
  return arr.reduce((acc, x) => { acc[x[key]] = x; return acc; }, {});
}

// formatage français des montants
function formatEUR(amount) {
  return new Intl.NumberFormat('fr-FR', {
    style: 'currency', currency: 'EUR'
  }).format(amount);
}
```
