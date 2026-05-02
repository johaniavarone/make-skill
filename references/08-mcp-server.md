# Make MCP Server

Le serveur MCP (Model Context Protocol) officiel Make permet à un système IA (Claude, ChatGPT, etc.) d'exécuter des scénarios Make et de gérer le compte Make depuis l'IA.

---

## 1. Concept

MCP est un protocole standard de communication entre un IA et un système externe. Le serveur MCP Make :

- **Expose les scénarios actifs et on-demand comme des "tools" callables** par l'IA.
- **Donne accès aux endpoints de management** (lister/modifier scénarios, connections, hooks, data stores, teams, organisations).

L'IA agit comme **MCP client**, le serveur Make comme **MCP server**. La communication suit un standard ouvert.

### Bénéfices

- **Tools dynamiques** : tout scénario Make activé devient un outil utilisable par l'IA.
- **Bidirectionnel** : l'IA lit l'état du compte et écrit dedans.
- **Auth déléguée** : OAuth ou MCP token, pas de clés API à hardcoder dans le client.

---

## 2. Auth & connexion

Deux modes d'authentification.

### Mode 1 : OAuth (recommandé pour utilisateurs finaux)

L'IA redirige l'utilisateur vers Make pour login + autoriser les scopes. Token géré automatiquement.

**URL de connexion** :
```
https://mcp.make.com
```

(serveur centralisé). Pour SSE :
```
https://mcp.make.com/sse
```

### Mode 2 : MCP token (recommandé pour scripts/agents)

L'utilisateur génère un token statique dans **Profile → API/MCP → MCP tokens** avec scopes choisis.

**URLs** :

| Transport | URL |
|-----------|-----|
| Streamable HTTP (default) | `https://<MAKE_ZONE>/mcp/u/<MCP_TOKEN>/stateless` |
| Streamable HTTP via /stream | `https://<MAKE_ZONE>/mcp/u/<MCP_TOKEN>/stream` |
| SSE | `https://<MAKE_ZONE>/mcp/u/<MCP_TOKEN>/sse` |
| Avec auth header | `https://<MAKE_ZONE>/mcp/stateless` + `Authorization: Bearer <MCP_TOKEN>` |

`<MAKE_ZONE>` = ton zone hostname, ex. `eu1.make.com`, `eu2.make.com`, `us1.make.com`.

### Scopes à choisir

- **Run your scenarios** : exposer les scénarios comme tools callables. Disponible sur tous les plans.
- **View and modify scenarios** : management. Plans paid.
- **View and modify connections / hooks / data stores** : management. Plans paid.
- **View and modify teams / orgs** : management. Plans paid.
- **scenarios:read** : nécessaire pour récupérer les outputs après timeout.

Choisir le minimum nécessaire.

---

## 3. Configuration côté client (Claude Desktop, ChatGPT, etc.)

### Claude Desktop (config JSON)

```json
{
  "mcpServers": {
    "make": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://eu1.make.com/mcp/u/MY_TOKEN/sse"
      ]
    }
  }
}
```

Pour OAuth :

```json
{
  "mcpServers": {
    "make": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mcp.make.com/sse"
      ]
    }
  }
}
```

### Claude Web / mobile

Aller dans **Settings → Connectors → Add MCP server** et coller l'URL. OAuth s'enclenche automatiquement.

### Custom client

Tout client compatible MCP fonctionne. Streamable HTTP est le transport préféré (plus reliable que SSE en pratique).

---

## 4. Scénarios as tools

Quand l'IA se connecte, elle voit la liste des scénarios **actifs** (et **on-demand**) du compte comme des tools.

### Définition d'un tool propre

Pour qu'un scénario soit un bon tool, il faut :

1. **Description claire** dans la fiche du scénario (UI → Description). L'IA lit ça pour décider quand l'invoquer.
2. **Inputs et outputs explicites** : configurer les **Scenario inputs** et **Scenario outputs** dans l'UI.

#### Scenario inputs

Dans le designer Make, ouvrir le panneau **Scenario inputs** → définir les paramètres que l'appelant fournit.

```json
{
  "inputs": [
    {"name": "customer_email", "type": "text", "required": true},
    {"name": "amount", "type": "number"},
    {"name": "metadata", "type": "any"}
  ]
}
```

Ces inputs sont accessibles dans le scénario via `{{scenario.inputs.customer_email}}`.

#### Scenario outputs

Module **Output** à la fin du scénario. Le bundle qu'il reçoit est renvoyé à l'IA.

```json
{
  "outputs": {
    "success": true,
    "invoice_id": "INV-12345",
    "url": "https://..."
  }
}
```

### Naming des tools (côté MCP)

Le tool s'appelle `make_scenario_<scenario_id>_<scenario_name>`. Maximum 56 caractères par défaut. Override via query param :

```
?maxToolNameLength=120
```

Range autorisée : 32-160 chars.

---

## 5. Timeouts

### Scenario run tools

| Endpoint | Timeout |
|----------|---------|
| `https://mcp.make.com` (OAuth) | 25s |
| `https://<ZONE>/mcp/...` (token) | 40s |

Si le scénario tourne au-delà → timeout avec une réponse :

```json
{
  "instruction": "The Tool execution has started, but did not complete yet. Use a corresponding Tool for getting Execution Results to check for the result asynchronously.",
  "executionId": "4eb841b3fe3d48a5b2b59e2eed3f55be",
  "scenarioId": 1
}
```

L'IA peut alors appeler le tool **Get Execution Result** avec `executionId` pour récupérer la sortie quand le scénario finit. Make garde le résultat 40 minutes.

**Prérequis** : scope `scenarios:read` activé.

### Management tools

| Endpoint | Timeout |
|----------|---------|
| `https://mcp.make.com` (OAuth) | 30s |
| `<ZONE>/mcp/stateless` | 60s |
| `<ZONE>/mcp/sse` ou `/stream` | 5min 20s |

---

## 6. Patterns d'usage IA + Make

### Pattern 1 : agent IA qui lit/écrit dans un Airtable client

L'utilisateur pose une question dans Claude. Claude reconnaît qu'elle nécessite des données du CRM Airtable. Via MCP Make :

1. Claude appelle `scenarios:list` pour découvrir un scénario `lookup_customer_by_email`.
2. Claude appelle ce scénario avec `email = "..."`.
3. Le scénario tourne dans Make : Airtable Search Records → return.
4. Claude reçoit le résultat et répond à l'utilisateur.

Pas besoin d'écrire un connecteur Airtable côté Claude — Make joue ce rôle.

### Pattern 2 : auto-création de scénarios par IA

Avec scope `scenarios:create`, l'IA peut créer un scénario à partir d'un blueprint qu'elle génère. Workflow :

1. User : "fait-moi un scénario qui sync mon Stripe vers Notion".
2. Claude génère le blueprint JSON.
3. Claude appelle l'API via MCP : `POST /scenarios` avec le blueprint.
4. Claude renvoie l'URL du scénario à activer manuellement.

### Pattern 3 : agent de monitoring

L'IA, sur déclenchement programmé (ou question utilisateur), :

1. Lit les `executions` des scénarios critiques sur 24h.
2. Identifie les échecs.
3. Récupère les payloads d'incomplete executions.
4. Propose un fix ou retry directement.

---

## 7. Sécurité

### MCP token = pleine puissance

Un MCP token avec scope `View and modify` peut tout faire dans le compte (créer/supprimer scénarios, exfiltrer connections incl. credentials chiffrés...). Traiter comme un secret de niveau API key root.

**Bonnes pratiques** :
- Un token par client MCP (Claude desktop laptop, Claude desktop home, agent serveur dédié...) pour pouvoir révoquer individuellement.
- Scopes minimaux. Si l'usage est lecture seule, choisir uniquement les `:read`.
- Rotation périodique (semestrielle).
- Storage : mettre dans un keychain OS, pas en clair dans config.

### OAuth refresh tokens

Si le client MCP utilise OAuth, le refresh token vit dans le client. Même précautions de stockage.

### Scenarios as tools : access control

Tous les scénarios actifs sont exposés au MCP token. Pour restreindre :
- **Désactiver** les scénarios qu'on ne veut pas exposer.
- **Mode on-demand** : pas exposé sur tous les plans.
- **White Label** : accès plus granulaire.

Voir doc officielle : `developers.make.com/mcp-server/connect-using-mcp-token/scenarios-as-tools-access-control`.

### Logs

Les invocations MCP apparaissent dans le scenario log Make standard. Auditer périodiquement pour détecter usages anormaux.

---

## 8. Limitations actuelles

- **Pas de streaming des outputs partiels** : un scénario MCP appelé renvoie son output une fois terminé, pas par chunk.
- **Pas d'accès aux comptes White Label cross-instance** : un MCP token est lié à un seul compte.
- **Tool naming truncation** : noms longs tronqués, peut créer des collisions si scénarios trop similaires en début de nom.
- **Quota d'opérations** : appels MCP consomment des opérations Make comme un trigger normal.

---

## 9. MCP server **non-officiel** : D1DX/make-mcp

Alternative à l'MCP officiel : [D1DX/make-mcp](https://github.com/D1DX/make-mcp).

**Différences** :
- Auth via API key Make standard (pas MCP token spécifique).
- Expose 200+ modules avec auto-healing des blueprints.
- Pensé pour le workflow "pull-edit-push" depuis un éditeur (Claude Code, Cursor).
- Open source, déployable en self-hosted.

**Cas d'usage** : développement et maintenance massive de scénarios via IA. L'MCP officiel est plutôt orienté runtime (l'IA invoque des scénarios), tandis que D1DX est orienté authoring (l'IA écrit des scénarios).

**Setup** :
```json
{
  "mcpServers": {
    "make": {
      "command": "npx",
      "args": ["-y", "@d1dx/make-mcp"],
      "env": {
        "MAKE_API_KEY": "...",
        "MAKE_ZONE": "eu1.make.com",
        "MAKE_TEAM_ID": "12345"
      }
    }
  }
}
```

---

## 10. Documentation officielle

- Hub : https://developers.make.com/mcp-server
- OAuth : https://developers.make.com/mcp-server/connect-using-oauth
- MCP token : https://developers.make.com/mcp-server/connect-using-mcp-token
- Scenarios as tools access control : https://developers.make.com/mcp-server/connect-using-mcp-token/scenarios-as-tools-access-control
