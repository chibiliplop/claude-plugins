# Piano Analytics MCP — Design Spec

**Date:** 2026-04-27  
**Auteur:** Thomas NABET  
**Statut:** Approuvé

---

## Contexte

Piano Analytics (ex AT Internet) est la solution d'analytics utilisée chez HelloWork. Ce MCP permet d'interroger les données Piano Analytics en langage naturel depuis Claude, sans connaître les noms techniques des métriques ou propriétés.

---

## Objectif

Créer un MCP TypeScript exposant des outils de discovery et de requête pour l'API Piano Analytics v3, afin de permettre à Claude de répondre à des questions analytiques en langage naturel (reporting quotidien, investigation ad hoc, suivi de KPIs).

---

## Emplacement

```
/Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp/
```

Le MCP vit dans le repo `claude-plugins` en tant que package TypeScript autonome, référencé dans le marketplace.

**`plugin.json` :**
```json
{
  "name": "piano-analytics-mcp",
  "version": "1.0.0",
  "description": "MCP server for querying Piano Analytics data in natural language",
  "type": "mcp-server",
  "entrypoint": "dist/index.js",
  "skills": [],
  "agents": []
}
```

---

## Architecture

### Approche retenue : MCP "2 phases" avec discovery

**Phase 1 — Discovery** : Claude explore les métadonnées disponibles avant de construire une requête.  
**Phase 2 — Query** : Claude exécute la requête avec les bons paramètres.

Puisque Piano Analytics n'expose pas d'endpoint API pour lister les métriques/propriétés, la discovery s'appuie sur un catalogue statique intégré (~50 propriétés, ~40 métriques). Les sites sont configurés via variables d'environnement.

### Transport

`stdio` — processus local déclaré dans la config Claude Desktop / Claude Code.

---

## Structure du projet

```
piano-analytics-mcp/
├── .claude-plugin/
│   └── plugin.json
├── src/
│   ├── index.ts                  # point d'entrée MCP (stdio server)
│   ├── tools/
│   │   ├── list-sites.ts
│   │   ├── list-properties.ts
│   │   ├── list-metrics.ts
│   │   ├── query-data.ts
│   │   └── query-total.ts
│   ├── piano/
│   │   ├── client.ts             # wrapper HTTP → api.atinternet.io
│   │   └── types.ts              # types TypeScript (QueryParams, PianoResponse, etc.)
│   └── catalog/
│       ├── properties.ts         # catalogue statique des propriétés Piano
│       └── metrics.ts            # catalogue statique des métriques Piano
├── package.json
├── tsconfig.json
└── README.md
```

---

## Stack technique

| Dépendance | Usage |
|---|---|
| `@modelcontextprotocol/sdk` | SDK MCP officiel (stdio server) |
| `zod` | Validation des paramètres d'outils |
| `node-fetch` / `fetch` natif Node 18+ | Appels HTTP vers Piano Analytics |
| `tsx` | Exécution TypeScript directe (dev) |
| `typescript` | Compilation pour distribution |

---

## Configuration

Variables d'environnement requises :

```env
PIANO_API_KEY=<clé générée sur my.piano.io/profile/#/apikeys>
PIANO_SITE_IDS=123456,789012        # liste séparée par virgules
PIANO_DEFAULT_SITE_ID=123456        # site utilisé si non précisé dans la requête
```

**Prérequis Piano :** L'API key doit appartenir à un utilisateur avec le rôle Administrateur, Délégué, Analyste Avancé, Analyste, ou un rôle personnalisé avec l'outil "Gérer les données".

---

## Outils MCP exposés

### `list_sites`

Retourne les sites configurés.

```
Entrée  : aucune
Sortie  : [{ id: number, name: string, isDefault: boolean }]
Source  : variables d'environnement PIANO_SITE_IDS / PIANO_DEFAULT_SITE_ID
```

### `list_properties`

Retourne le catalogue des propriétés Piano disponibles.

```
Entrée  : { search?: string }      // filtre optionnel, ex: "device"
Sortie  : [{ key: string, description: string, example: string }]
Source  : catalogue statique (src/catalog/properties.ts)
```

Exemples de propriétés incluses : `device_type`, `visit_src`, `page`, `browser`, `os`, `country`, `src_detail`, `visit_bounce`, `user_id`.

### `list_metrics`

Retourne le catalogue des métriques Piano disponibles.

```
Entrée  : { search?: string }
Sortie  : [{ key: string, description: string, unit: string }]
Source  : catalogue statique (src/catalog/metrics.ts)
```

Exemples de métriques incluses : `m_visits`, `m_page_views`, `m_users`, `m_bounces`, `m_bounce_rate`, `m_avg_duration`, `m_events`, `m_conversions`.

### `query_data`

Exécute un appel `getData` vers `https://api.atinternet.io/v3/data/getData`.

```
Entrée : {
  columns       : string[]              // métriques + propriétés
  site_id?      : number                // défaut : PIANO_DEFAULT_SITE_ID
  period_start? : string                // "2026-04-01" (ISO 8601)
  period_end?   : string                // "2026-04-27" (ISO 8601)
  period_preset?: "today"
                | "yesterday"
                | "last_7_days"
                | "last_30_days"
                | "current_week"
                | "last_week"
                | "current_month"
                | "last_month"
  sort?         : string[]              // ex: ["-m_visits"] (- = desc)
  filter?       : {
    metric?   : object                  // ex: { "m_visits": { "$gt": 100 } }
    property? : object                  // ex: { "device_type": { "$eq": "Mobile" } }
  }
  max_results?  : number                // défaut 100, max 10000
  page?         : number                // défaut 1
}

Sortie : {
  rows        : object[]
  total_rows  : number
  page        : number
}
```

**Mapping période interne :**

| `period_preset` | Conversion Piano |
|---|---|
| `today` | `{"type": "R", "granularity": "D", "offset": 0}` |
| `yesterday` | `{"type": "R", "granularity": "D", "offset": -1}` |
| `last_7_days` | `{"type": "R", "granularity": "D", "startOffset": -7, "endOffset": -1}` |
| `last_30_days` | `{"type": "R", "granularity": "D", "startOffset": -30, "endOffset": -1}` |
| `current_week` | `{"type": "R", "granularity": "W", "offset": 0}` |
| `last_week` | `{"type": "R", "granularity": "W", "offset": -1}` |
| `current_month` | `{"type": "R", "granularity": "M", "offset": 0}` |
| `last_month` | `{"type": "R", "granularity": "M", "offset": -1}` |

- `period_start` + `period_end` sont convertis en période absolue `{"type": "D", "start": ..., "end": ...}`
- Si ni preset ni dates fournies : erreur explicite retournée à Claude

### `query_total`

Exécute un appel `getTotal` vers `https://api.atinternet.io/v3/data/getTotal`.

```
Entrée : {
  columns       : string[]
  site_id?      : number
  period_start? : string
  period_end?   : string
  period_preset?: (même liste que query_data)
  filter?       : { metric?: object, property?: object }
}

Sortie : { totals: object }
```

---

## Authentification Piano Analytics

Header HTTP : `x-api-key: <PIANO_API_KEY>`

Endpoint base : `https://api.atinternet.io/v3/data/`

---

## Opérateurs de filtre disponibles (référence)

| Type | Opérateurs |
|---|---|
| Nombre | `$eq`, `$neq`, `$in`, `$gt`, `$gte`, `$lt`, `$lte`, `$na`, `$undefined` |
| Chaîne | `$eq`, `$neq`, `$in`, `$lk`, `$nlk`, `$start`, `$nstart`, `$end`, `$nend` |
| Date | `$eq`, `$gt`, `$gte`, `$lt`, `$lte` |
| Booléen | `$eq`, `$neq` |
| Logique | `$AND`, `$OR` (sur les propriétés) |

---

## Flux type (langage naturel → API)

> **Question :** "Combien de visites mobiles la semaine dernière ?"

1. Claude appelle `list_metrics` avec `search: "visites"` → trouve `m_visits`
2. Claude appelle `list_properties` avec `search: "device"` → trouve `device_type`
3. Claude appelle `query_data` avec :
   ```json
   {
     "columns": ["device_type", "m_visits"],
     "period_preset": "last_week",
     "filter": { "property": { "device_type": { "$eq": "Mobile" } } },
     "sort": ["-m_visits"]
   }
   ```
4. Claude présente le résultat en langage naturel

---

## Distribution

### Local (usage personnel)
Déclaré dans `~/.claude/claude_desktop_config.json` :
```json
{
  "mcpServers": {
    "piano-analytics": {
      "command": "node",
      "args": ["/Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp/dist/index.js"],
      "env": {
        "PIANO_API_KEY": "...",
        "PIANO_SITE_IDS": "...",
        "PIANO_DEFAULT_SITE_ID": "..."
      }
    }
  }
}
```

### Partageable (équipe HelloWork)
- Clone du repo `claude-plugins`
- `cd piano-analytics-mcp && npm install && npm run build`
- Copier la config MCP ci-dessus avec ses propres credentials

---

## Limites API Piano Analytics

| Contrainte | Valeur |
|---|---|
| Colonnes max par requête | 50 |
| Lignes max par appel | 10 000 |
| Lignes max total (pagination) | 200 000 |
| Segments max par requête | 6 |
| Appels concurrents par utilisateur | 5 |
| Appels concurrents par organisation | 20 |

---

## Hors scope (v1)

- Cache des métadonnées (Option C — peut être ajouté en v2)
- Skills Claude Code associés
- Support des segments Piano
- Évolutions temporelles (`evo` parameter)
- Endpoint `getRowCount`
