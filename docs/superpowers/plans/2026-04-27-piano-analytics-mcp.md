# Piano Analytics MCP — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Créer un MCP TypeScript dans `/Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp/` qui expose 5 outils permettant à Claude d'interroger Piano Analytics en langage naturel.

**Architecture:** MCP "2 phases" — discovery (list_sites, list_properties, list_metrics via catalogue statique) puis query (query_data, query_total via l'API Piano Analytics v3). Transport stdio. Authentification par API key via header `x-api-key`.

**Tech Stack:** Node.js 25, TypeScript 5, `@modelcontextprotocol/sdk`, `zod`, `vitest`

**Spec de référence:** `docs/superpowers/specs/2026-04-27-piano-analytics-mcp-design.md`

---

## Carte des fichiers

| Fichier | Responsabilité |
|---|---|
| `src/index.ts` | Point d'entrée : instancie McpServer, enregistre les 5 outils, connecte stdio |
| `src/piano/types.ts` | Interfaces TypeScript pour l'API Piano (params, réponses, erreurs) |
| `src/piano/period.ts` | Conversion `period_preset` / dates ISO → objet période Piano v3 |
| `src/piano/client.ts` | Wrapper HTTP : `POST api.atinternet.io/v3/data/*` avec auth |
| `src/catalog/properties.ts` | Catalogue statique ~50 propriétés Piano avec clé, description, exemple |
| `src/catalog/metrics.ts` | Catalogue statique ~40 métriques Piano avec clé, description, unité |
| `src/tools/list-sites.ts` | Lit `PIANO_SITE_IDS` / `PIANO_DEFAULT_SITE_ID` et retourne la liste |
| `src/tools/list-properties.ts` | Filtre le catalogue propriétés sur `search` optionnel |
| `src/tools/list-metrics.ts` | Filtre le catalogue métriques sur `search` optionnel |
| `src/tools/query-data.ts` | Construit et exécute `getData`, gère la pagination |
| `src/tools/query-total.ts` | Construit et exécute `getTotal` |
| `src/piano/period.test.ts` | Tests unitaires du mapping de période |
| `src/catalog/properties.test.ts` | Tests intégrité catalogue propriétés + filtre |
| `src/catalog/metrics.test.ts` | Tests intégrité catalogue métriques + filtre |
| `src/piano/client.test.ts` | Tests client HTTP avec fetch mocké |
| `.claude-plugin/plugin.json` | Métadonnées plugin pour le marketplace |
| `package.json` | Dépendances, scripts build/test/dev |
| `tsconfig.json` | Config TypeScript (ESNext, NodeNext modules) |
| `README.md` | Instructions d'installation et config Claude |

---

## Task 1 : Scaffold du projet

**Files:**
- Create: `piano-analytics-mcp/package.json`
- Create: `piano-analytics-mcp/tsconfig.json`
- Create: `piano-analytics-mcp/vitest.config.ts`
- Create: `piano-analytics-mcp/.claude-plugin/plugin.json`

- [ ] **Step 1 : Créer la structure de dossiers**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins
mkdir -p piano-analytics-mcp/src/tools
mkdir -p piano-analytics-mcp/src/piano
mkdir -p piano-analytics-mcp/src/catalog
mkdir -p piano-analytics-mcp/.claude-plugin
mkdir -p piano-analytics-mcp/dist
```

- [ ] **Step 2 : Créer `package.json`**

```json
{
  "name": "piano-analytics-mcp",
  "version": "1.0.0",
  "description": "MCP server for querying Piano Analytics data in natural language",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.10.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 3 : Créer `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

- [ ] **Step 4 : Créer `vitest.config.ts`**

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
  },
});
```

- [ ] **Step 5 : Créer `.claude-plugin/plugin.json`**

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

- [ ] **Step 6 : Installer les dépendances**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp
npm install
```

Expected : `node_modules/` créé, pas d'erreur.

- [ ] **Step 7 : Commit**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins
git add piano-analytics-mcp/package.json piano-analytics-mcp/tsconfig.json piano-analytics-mcp/vitest.config.ts piano-analytics-mcp/.claude-plugin/plugin.json
git commit -m "feat(piano-analytics-mcp): scaffold project"
```

---

## Task 2 : Types TypeScript

**Files:**
- Create: `piano-analytics-mcp/src/piano/types.ts`

- [ ] **Step 1 : Créer `src/piano/types.ts`**

```typescript
export type PeriodPreset =
  | "today"
  | "yesterday"
  | "last_7_days"
  | "last_30_days"
  | "current_week"
  | "last_week"
  | "current_month"
  | "last_month";

export type PianoGranularity = "D" | "W" | "M" | "Y";

export type PianoSingleRelativePeriod = {
  type: "R";
  granularity: PianoGranularity;
  offset: number;
};

export type PianoRangeRelativePeriod = {
  type: "R";
  granularity: PianoGranularity;
  startOffset: number;
  endOffset: number;
};

export type PianoAbsolutePeriod = {
  type: "D";
  start: string;
  end: string;
};

export type PianoPeriod =
  | PianoSingleRelativePeriod
  | PianoRangeRelativePeriod
  | PianoAbsolutePeriod;

export interface PianoFilter {
  metric?: Record<string, unknown>;
  property?: Record<string, unknown>;
}

export interface PianoQueryParams {
  columns: string[];
  space: { s: number[] };
  period: { p1: [PianoPeriod] };
  sort?: string[];
  filter?: PianoFilter;
  "max-results"?: number;
  "page-num"?: number;
}

export interface PianoRow {
  [key: string]: string | number | boolean | null;
}

export interface PianoGetDataResponse {
  DataFeed: {
    Rows: PianoRow[];
    RowCounts: {
      Total: number;
      CurPage: number;
      MaxPerPage: number;
    };
  };
}

export interface PianoGetTotalResponse {
  DataFeed: {
    Total: PianoRow;
  };
}

export interface PianoErrorResponse {
  Message: string;
  ErrorCode?: string;
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/piano/types.ts
git commit -m "feat(piano-analytics-mcp): add Piano API TypeScript types"
```

---

## Task 3 : Utilitaire de mapping de période

**Files:**
- Create: `piano-analytics-mcp/src/piano/period.ts`
- Create: `piano-analytics-mcp/src/piano/period.test.ts`

- [ ] **Step 1 : Écrire les tests en premier**

```typescript
// src/piano/period.test.ts
import { describe, it, expect } from "vitest";
import { buildPeriod } from "./period.js";

describe("buildPeriod", () => {
  it("maps 'today' to single relative period offset 0", () => {
    const result = buildPeriod({ period_preset: "today" });
    expect(result).toEqual({ type: "R", granularity: "D", offset: 0 });
  });

  it("maps 'yesterday' to single relative period offset -1", () => {
    const result = buildPeriod({ period_preset: "yesterday" });
    expect(result).toEqual({ type: "R", granularity: "D", offset: -1 });
  });

  it("maps 'last_7_days' to range relative period", () => {
    const result = buildPeriod({ period_preset: "last_7_days" });
    expect(result).toEqual({
      type: "R",
      granularity: "D",
      startOffset: -7,
      endOffset: -1,
    });
  });

  it("maps 'last_30_days' to range relative period", () => {
    const result = buildPeriod({ period_preset: "last_30_days" });
    expect(result).toEqual({
      type: "R",
      granularity: "D",
      startOffset: -30,
      endOffset: -1,
    });
  });

  it("maps 'current_week' to weekly period offset 0", () => {
    const result = buildPeriod({ period_preset: "current_week" });
    expect(result).toEqual({ type: "R", granularity: "W", offset: 0 });
  });

  it("maps 'last_week' to weekly period offset -1", () => {
    const result = buildPeriod({ period_preset: "last_week" });
    expect(result).toEqual({ type: "R", granularity: "W", offset: -1 });
  });

  it("maps 'current_month' to monthly period offset 0", () => {
    const result = buildPeriod({ period_preset: "current_month" });
    expect(result).toEqual({ type: "R", granularity: "M", offset: 0 });
  });

  it("maps 'last_month' to monthly period offset -1", () => {
    const result = buildPeriod({ period_preset: "last_month" });
    expect(result).toEqual({ type: "R", granularity: "M", offset: -1 });
  });

  it("maps absolute dates to D-type period", () => {
    const result = buildPeriod({
      period_start: "2026-04-01",
      period_end: "2026-04-27",
    });
    expect(result).toEqual({
      type: "D",
      start: "2026-04-01",
      end: "2026-04-27",
    });
  });

  it("throws when no period information is provided", () => {
    expect(() => buildPeriod({})).toThrow(
      "Vous devez fournir period_preset ou period_start + period_end"
    );
  });

  it("throws when only period_start is provided without period_end", () => {
    expect(() => buildPeriod({ period_start: "2026-04-01" })).toThrow(
      "Vous devez fournir period_preset ou period_start + period_end"
    );
  });
});
```

- [ ] **Step 2 : Lancer les tests pour vérifier qu'ils échouent**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp
npm test
```

Expected : FAIL — `Cannot find module './period.js'`

- [ ] **Step 3 : Implémenter `src/piano/period.ts`**

```typescript
import type { PeriodPreset, PianoPeriod } from "./types.js";

interface PeriodInput {
  period_preset?: PeriodPreset;
  period_start?: string;
  period_end?: string;
}

export function buildPeriod(input: PeriodInput): PianoPeriod {
  if (input.period_preset) {
    return PRESET_MAP[input.period_preset];
  }

  if (input.period_start && input.period_end) {
    return { type: "D", start: input.period_start, end: input.period_end };
  }

  throw new Error(
    "Vous devez fournir period_preset ou period_start + period_end"
  );
}

const PRESET_MAP: Record<PeriodPreset, PianoPeriod> = {
  today: { type: "R", granularity: "D", offset: 0 },
  yesterday: { type: "R", granularity: "D", offset: -1 },
  last_7_days: { type: "R", granularity: "D", startOffset: -7, endOffset: -1 },
  last_30_days: {
    type: "R",
    granularity: "D",
    startOffset: -30,
    endOffset: -1,
  },
  current_week: { type: "R", granularity: "W", offset: 0 },
  last_week: { type: "R", granularity: "W", offset: -1 },
  current_month: { type: "R", granularity: "M", offset: 0 },
  last_month: { type: "R", granularity: "M", offset: -1 },
};
```

- [ ] **Step 4 : Lancer les tests**

```bash
npm test
```

Expected : 11 tests PASS.

- [ ] **Step 5 : Commit**

```bash
git add piano-analytics-mcp/src/piano/period.ts piano-analytics-mcp/src/piano/period.test.ts
git commit -m "feat(piano-analytics-mcp): add period mapping utility with tests"
```

---

## Task 4 : Catalogue des propriétés Piano

**Files:**
- Create: `piano-analytics-mcp/src/catalog/properties.ts`
- Create: `piano-analytics-mcp/src/catalog/properties.test.ts`

- [ ] **Step 1 : Écrire les tests en premier**

```typescript
// src/catalog/properties.test.ts
import { describe, it, expect } from "vitest";
import { PROPERTIES, filterProperties } from "./properties.js";

describe("PROPERTIES catalog", () => {
  it("has at least 20 entries", () => {
    expect(PROPERTIES.length).toBeGreaterThanOrEqual(20);
  });

  it("every entry has key, description, and example", () => {
    for (const prop of PROPERTIES) {
      expect(prop.key, `${prop.key} missing key`).toBeTruthy();
      expect(prop.description, `${prop.key} missing description`).toBeTruthy();
      expect(prop.example, `${prop.key} missing example`).toBeTruthy();
    }
  });

  it("all keys are unique", () => {
    const keys = PROPERTIES.map((p) => p.key);
    expect(new Set(keys).size).toBe(keys.length);
  });
});

describe("filterProperties", () => {
  it("returns all properties when no search term", () => {
    expect(filterProperties(undefined)).toEqual(PROPERTIES);
  });

  it("filters by key substring (case-insensitive)", () => {
    const result = filterProperties("device");
    expect(result.length).toBeGreaterThan(0);
    expect(result.every((p) => p.key.includes("device") || p.description.toLowerCase().includes("device"))).toBe(true);
  });

  it("filters by description substring (case-insensitive)", () => {
    const result = filterProperties("appareil");
    expect(result.length).toBeGreaterThan(0);
  });

  it("returns empty array when no match", () => {
    expect(filterProperties("xyznonexistent")).toEqual([]);
  });
});
```

- [ ] **Step 2 : Lancer les tests pour vérifier qu'ils échouent**

```bash
npm test
```

Expected : FAIL — `Cannot find module './properties.js'`

- [ ] **Step 3 : Implémenter `src/catalog/properties.ts`**

```typescript
export interface PropertyEntry {
  key: string;
  description: string;
  example: string;
}

export const PROPERTIES: PropertyEntry[] = [
  { key: "device_type", description: "Type d'appareil", example: "Desktop, Mobile, Tablet, TV, Bot" },
  { key: "browser", description: "Navigateur web", example: "Chrome, Firefox, Safari" },
  { key: "os", description: "Système d'exploitation", example: "Windows, macOS, Android, iOS" },
  { key: "country", description: "Pays de l'utilisateur", example: "France, United States, Germany" },
  { key: "language", description: "Langue du navigateur", example: "fr, en, de" },
  { key: "connection_type", description: "Type de connexion réseau", example: "broadband, mobile, wifi" },
  { key: "page", description: "URL ou chemin de la page visitée", example: "/accueil, /offres-emploi" },
  { key: "page_chapter1", description: "Chapitre 1 de la page (arborescence)", example: "emploi" },
  { key: "page_chapter2", description: "Chapitre 2 de la page (arborescence)", example: "offres" },
  { key: "page_chapter3", description: "Chapitre 3 de la page (arborescence)", example: "ingenieur" },
  { key: "previous_page", description: "Page précédente visitée", example: "/accueil" },
  { key: "src", description: "Source de visite (canal)", example: "Direct, Organic search, Referrer" },
  { key: "src_detail", description: "Détail de la source de visite", example: "google, bing, newsletter" },
  { key: "src_type", description: "Type de source marketing", example: "Search, Social, Email" },
  { key: "referrer", description: "URL de la page référente", example: "https://google.fr" },
  { key: "search_keyword", description: "Mot-clé de recherche utilisé", example: "emploi informatique paris" },
  { key: "campaign", description: "Nom de la campagne marketing", example: "summer-2026" },
  { key: "campaign_type", description: "Type de campagne marketing", example: "display, email, cpc" },
  { key: "medium", description: "Medium de la campagne marketing", example: "cpc, organic, referral" },
  { key: "visit_bounce", description: "Indicateur de rebond (visite d'une seule page)", example: "true, false" },
  { key: "user_id", description: "Identifiant utilisateur authentifié", example: "user_12345" },
  { key: "user_category", description: "Catégorie de l'utilisateur", example: "premium, freemium, guest" },
  { key: "site_id", description: "Identifiant du site Piano Analytics", example: "123456" },
  { key: "event_name", description: "Nom de l'événement déclenché", example: "click.cta, form.submit" },
  { key: "event_collection", description: "Collection d'événements", example: "page.display, click" },
];

export function filterProperties(search: string | undefined): PropertyEntry[] {
  if (!search) return PROPERTIES;
  const term = search.toLowerCase();
  return PROPERTIES.filter(
    (p) =>
      p.key.toLowerCase().includes(term) ||
      p.description.toLowerCase().includes(term)
  );
}
```

- [ ] **Step 4 : Lancer les tests**

```bash
npm test
```

Expected : tous les tests de `properties.test.ts` PASS.

- [ ] **Step 5 : Commit**

```bash
git add piano-analytics-mcp/src/catalog/properties.ts piano-analytics-mcp/src/catalog/properties.test.ts
git commit -m "feat(piano-analytics-mcp): add properties catalog with tests"
```

---

## Task 5 : Catalogue des métriques Piano

**Files:**
- Create: `piano-analytics-mcp/src/catalog/metrics.ts`
- Create: `piano-analytics-mcp/src/catalog/metrics.test.ts`

- [ ] **Step 1 : Écrire les tests en premier**

```typescript
// src/catalog/metrics.test.ts
import { describe, it, expect } from "vitest";
import { METRICS, filterMetrics } from "./metrics.js";

describe("METRICS catalog", () => {
  it("has at least 15 entries", () => {
    expect(METRICS.length).toBeGreaterThanOrEqual(15);
  });

  it("every entry has key, description, and unit", () => {
    for (const metric of METRICS) {
      expect(metric.key, `${metric.key} missing key`).toBeTruthy();
      expect(metric.description, `${metric.key} missing description`).toBeTruthy();
      expect(metric.unit, `${metric.key} missing unit`).toBeTruthy();
    }
  });

  it("all keys start with 'm_'", () => {
    expect(METRICS.every((m) => m.key.startsWith("m_"))).toBe(true);
  });

  it("all keys are unique", () => {
    const keys = METRICS.map((m) => m.key);
    expect(new Set(keys).size).toBe(keys.length);
  });
});

describe("filterMetrics", () => {
  it("returns all metrics when no search term", () => {
    expect(filterMetrics(undefined)).toEqual(METRICS);
  });

  it("finds m_visits when searching 'visit'", () => {
    const result = filterMetrics("visit");
    expect(result.some((m) => m.key === "m_visits")).toBe(true);
  });

  it("finds m_bounce_rate when searching 'rebond'", () => {
    const result = filterMetrics("rebond");
    expect(result.some((m) => m.key === "m_bounce_rate" || m.key === "m_bounces")).toBe(true);
  });

  it("returns empty array when no match", () => {
    expect(filterMetrics("xyznonexistent")).toEqual([]);
  });
});
```

- [ ] **Step 2 : Lancer les tests pour vérifier qu'ils échouent**

```bash
npm test
```

Expected : FAIL — `Cannot find module './metrics.js'`

- [ ] **Step 3 : Implémenter `src/catalog/metrics.ts`**

```typescript
export interface MetricEntry {
  key: string;
  description: string;
  unit: string;
}

export const METRICS: MetricEntry[] = [
  { key: "m_visits", description: "Nombre de visites", unit: "entier" },
  { key: "m_page_views", description: "Pages vues", unit: "entier" },
  { key: "m_users", description: "Utilisateurs uniques", unit: "entier" },
  { key: "m_bounces", description: "Visites avec rebond (une seule page)", unit: "entier" },
  { key: "m_bounce_rate", description: "Taux de rebond", unit: "pourcentage" },
  { key: "m_avg_duration", description: "Durée moyenne de visite", unit: "secondes" },
  { key: "m_page_views_per_visit", description: "Pages vues par visite", unit: "décimal" },
  { key: "m_new_visitors", description: "Nouveaux visiteurs", unit: "entier" },
  { key: "m_returning_visitors", description: "Visiteurs récurrents", unit: "entier" },
  { key: "m_events", description: "Nombre total d'événements", unit: "entier" },
  { key: "m_conversions", description: "Nombre de conversions", unit: "entier" },
  { key: "m_conversion_rate", description: "Taux de conversion", unit: "pourcentage" },
  { key: "m_time_spent", description: "Temps total passé sur le site", unit: "secondes" },
  { key: "m_avg_page_duration", description: "Durée moyenne sur une page", unit: "secondes" },
  { key: "m_clicks", description: "Nombre de clics", unit: "entier" },
  { key: "m_impressions", description: "Nombre d'impressions", unit: "entier" },
  { key: "m_orders", description: "Nombre de commandes", unit: "entier" },
  { key: "m_revenue", description: "Revenus générés", unit: "devise" },
  { key: "m_exit_rate", description: "Taux de sortie", unit: "pourcentage" },
  { key: "m_entry_pages", description: "Pages d'entrée uniques", unit: "entier" },
];

export function filterMetrics(search: string | undefined): MetricEntry[] {
  if (!search) return METRICS;
  const term = search.toLowerCase();
  return METRICS.filter(
    (m) =>
      m.key.toLowerCase().includes(term) ||
      m.description.toLowerCase().includes(term)
  );
}
```

- [ ] **Step 4 : Lancer les tests**

```bash
npm test
```

Expected : tous les tests de `metrics.test.ts` PASS.

- [ ] **Step 5 : Commit**

```bash
git add piano-analytics-mcp/src/catalog/metrics.ts piano-analytics-mcp/src/catalog/metrics.test.ts
git commit -m "feat(piano-analytics-mcp): add metrics catalog with tests"
```

---

## Task 6 : Client HTTP Piano Analytics

**Files:**
- Create: `piano-analytics-mcp/src/piano/client.ts`
- Create: `piano-analytics-mcp/src/piano/client.test.ts`

- [ ] **Step 1 : Écrire les tests en premier**

```typescript
// src/piano/client.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { PianoClient } from "./client.js";
import type { PianoQueryParams } from "./types.js";

const MOCK_API_KEY = "test-api-key-123";

const BASE_PARAMS: PianoQueryParams = {
  columns: ["device_type", "m_visits"],
  space: { s: [123456] },
  period: { p1: [{ type: "R", granularity: "D", offset: -1 }] },
};

describe("PianoClient.getData", () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  it("calls getData endpoint with correct headers", async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({
        DataFeed: {
          Rows: [{ device_type: "Desktop", m_visits: 1000 }],
          RowCounts: { Total: 1, CurPage: 1, MaxPerPage: 50 },
        },
      }),
    });
    vi.stubGlobal("fetch", mockFetch);

    const client = new PianoClient(MOCK_API_KEY);
    await client.getData(BASE_PARAMS);

    expect(mockFetch).toHaveBeenCalledWith(
      "https://api.atinternet.io/v3/data/getData",
      expect.objectContaining({
        method: "POST",
        headers: expect.objectContaining({
          "x-api-key": MOCK_API_KEY,
          "Content-Type": "application/json",
        }),
      })
    );
  });

  it("throws PianoApiError on non-ok response", async () => {
    vi.stubGlobal("fetch", vi.fn().mockResolvedValue({
      ok: false,
      status: 401,
      json: async () => ({ Message: "Unauthorized", ErrorCode: "AUTH_ERROR" }),
    }));

    const client = new PianoClient(MOCK_API_KEY);
    await expect(client.getData(BASE_PARAMS)).rejects.toThrow("Unauthorized");
  });

  it("returns parsed DataFeed rows", async () => {
    vi.stubGlobal("fetch", vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({
        DataFeed: {
          Rows: [
            { device_type: "Desktop", m_visits: 1000 },
            { device_type: "Mobile", m_visits: 500 },
          ],
          RowCounts: { Total: 2, CurPage: 1, MaxPerPage: 50 },
        },
      }),
    }));

    const client = new PianoClient(MOCK_API_KEY);
    const result = await client.getData(BASE_PARAMS);

    expect(result.DataFeed.Rows).toHaveLength(2);
    expect(result.DataFeed.RowCounts.Total).toBe(2);
  });
});

describe("PianoClient.getTotal", () => {
  it("calls getTotal endpoint", async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({
        DataFeed: { Total: { m_visits: 5000 } },
      }),
    });
    vi.stubGlobal("fetch", mockFetch);

    const client = new PianoClient(MOCK_API_KEY);
    await client.getTotal(BASE_PARAMS);

    expect(mockFetch).toHaveBeenCalledWith(
      "https://api.atinternet.io/v3/data/getTotal",
      expect.anything()
    );
  });
});
```

- [ ] **Step 2 : Lancer les tests pour vérifier qu'ils échouent**

```bash
npm test
```

Expected : FAIL — `Cannot find module './client.js'`

- [ ] **Step 3 : Implémenter `src/piano/client.ts`**

```typescript
import type {
  PianoGetDataResponse,
  PianoGetTotalResponse,
  PianoQueryParams,
} from "./types.js";

const BASE_URL = "https://api.atinternet.io/v3/data";

export class PianoApiError extends Error {
  constructor(
    message: string,
    public readonly errorCode: string | undefined,
    public readonly status: number
  ) {
    super(message);
    this.name = "PianoApiError";
  }
}

export class PianoClient {
  constructor(private readonly apiKey: string) {}

  async getData(params: PianoQueryParams): Promise<PianoGetDataResponse> {
    return this.post<PianoGetDataResponse>(`${BASE_URL}/getData`, params);
  }

  async getTotal(params: PianoQueryParams): Promise<PianoGetTotalResponse> {
    return this.post<PianoGetTotalResponse>(`${BASE_URL}/getTotal`, params);
  }

  private async post<T>(url: string, params: PianoQueryParams): Promise<T> {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "x-api-key": this.apiKey,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(params),
    });

    const data = await response.json();

    if (!response.ok) {
      const errorData = data as { Message?: string; ErrorCode?: string };
      throw new PianoApiError(
        errorData.Message ?? `HTTP ${response.status}`,
        errorData.ErrorCode,
        response.status
      );
    }

    return data as T;
  }
}
```

- [ ] **Step 4 : Lancer les tests**

```bash
npm test
```

Expected : 5 tests PASS.

- [ ] **Step 5 : Commit**

```bash
git add piano-analytics-mcp/src/piano/client.ts piano-analytics-mcp/src/piano/client.test.ts
git commit -m "feat(piano-analytics-mcp): add Piano HTTP client with tests"
```

---

## Task 7 : Outil `list_sites`

**Files:**
- Create: `piano-analytics-mcp/src/tools/list-sites.ts`

- [ ] **Step 1 : Créer `src/tools/list-sites.ts`**

```typescript
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export interface SiteEntry {
  id: number;
  name: string;
  isDefault: boolean;
}

export function loadSitesFromEnv(): SiteEntry[] {
  const siteIdsRaw = process.env.PIANO_SITE_IDS;
  const defaultSiteId = process.env.PIANO_DEFAULT_SITE_ID
    ? parseInt(process.env.PIANO_DEFAULT_SITE_ID, 10)
    : undefined;

  if (!siteIdsRaw) {
    throw new Error(
      "Variable d'environnement PIANO_SITE_IDS manquante (ex: 123456,789012)"
    );
  }

  return siteIdsRaw.split(",").map((rawId) => {
    const id = parseInt(rawId.trim(), 10);
    if (isNaN(id)) {
      throw new Error(
        `PIANO_SITE_IDS contient une valeur invalide: "${rawId.trim()}"`
      );
    }
    return { id, name: `Site ${id}`, isDefault: id === defaultSiteId };
  });
}

export function registerListSitesTool(server: McpServer): void {
  server.tool(
    "list_sites",
    "Liste les sites Piano Analytics configurés. Appeler en premier pour connaître les site_id disponibles.",
    {},
    async () => {
      try {
        const sites = loadSitesFromEnv();
        return {
          content: [
            {
              type: "text" as const,
              text: JSON.stringify(sites, null, 2),
            },
          ],
        };
      } catch (error) {
        return {
          content: [
            {
              type: "text" as const,
              text: `Erreur de configuration: ${error instanceof Error ? error.message : String(error)}`,
            },
          ],
          isError: true,
        };
      }
    }
  );
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/tools/list-sites.ts
git commit -m "feat(piano-analytics-mcp): add list_sites tool"
```

---

## Task 8 : Outil `list_properties`

**Files:**
- Create: `piano-analytics-mcp/src/tools/list-properties.ts`

- [ ] **Step 1 : Créer `src/tools/list-properties.ts`**

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { filterProperties } from "../catalog/properties.js";

export function registerListPropertiesTool(server: McpServer): void {
  server.tool(
    "list_properties",
    "Liste les propriétés (dimensions) disponibles dans Piano Analytics. Utiliser pour trouver les clés exactes avant de construire une requête. Exemple: 'device' pour trouver device_type.",
    { search: z.string().optional().describe("Filtre par nom ou description, insensible à la casse. Ex: 'device', 'appareil', 'page'") },
    async ({ search }) => {
      const properties = filterProperties(search);
      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify(properties, null, 2),
          },
        ],
      };
    }
  );
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/tools/list-properties.ts
git commit -m "feat(piano-analytics-mcp): add list_properties tool"
```

---

## Task 9 : Outil `list_metrics`

**Files:**
- Create: `piano-analytics-mcp/src/tools/list-metrics.ts`

- [ ] **Step 1 : Créer `src/tools/list-metrics.ts`**

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { filterMetrics } from "../catalog/metrics.js";

export function registerListMetricsTool(server: McpServer): void {
  server.tool(
    "list_metrics",
    "Liste les métriques disponibles dans Piano Analytics. Utiliser pour trouver les clés exactes avant de construire une requête. Exemple: 'visit' pour trouver m_visits.",
    { search: z.string().optional().describe("Filtre par nom ou description, insensible à la casse. Ex: 'visit', 'bounce', 'conversion'") },
    async ({ search }) => {
      const metrics = filterMetrics(search);
      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify(metrics, null, 2),
          },
        ],
      };
    }
  );
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/tools/list-metrics.ts
git commit -m "feat(piano-analytics-mcp): add list_metrics tool"
```

---

## Task 10 : Outil `query_data`

**Files:**
- Create: `piano-analytics-mcp/src/tools/query-data.ts`

- [ ] **Step 1 : Créer `src/tools/query-data.ts`**

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { PianoClient, PianoApiError } from "../piano/client.js";
import { buildPeriod } from "../piano/period.js";
import type { PianoQueryParams } from "../piano/types.js";

const PERIOD_PRESET_VALUES = [
  "today",
  "yesterday",
  "last_7_days",
  "last_30_days",
  "current_week",
  "last_week",
  "current_month",
  "last_month",
] as const;

export function registerQueryDataTool(server: McpServer, client: PianoClient): void {
  server.tool(
    "query_data",
    `Interroge Piano Analytics et retourne des données tabulaires. Utiliser list_properties et list_metrics d'abord pour trouver les bonnes clés de colonnes.
    
Exemples de columns: ["device_type", "m_visits"], ["page", "m_page_views", "m_bounce_rate"]
Exemples de sort: ["-m_visits"] (décroissant), ["page"] (croissant)`,
    {
      columns: z
        .array(z.string())
        .min(1)
        .describe('Colonnes à retourner. Mélange de propriétés et métriques. Ex: ["device_type", "m_visits"]'),
      site_id: z
        .number()
        .int()
        .optional()
        .describe("ID du site Piano Analytics. Si omis, utilise PIANO_DEFAULT_SITE_ID"),
      period_preset: z
        .enum(PERIOD_PRESET_VALUES)
        .optional()
        .describe("Période prédéfinie. Prioritaire sur period_start/period_end"),
      period_start: z
        .string()
        .optional()
        .describe("Date de début ISO 8601 (YYYY-MM-DD). Requis si pas de period_preset"),
      period_end: z
        .string()
        .optional()
        .describe("Date de fin ISO 8601 (YYYY-MM-DD). Requis si pas de period_preset"),
      sort: z
        .array(z.string())
        .optional()
        .describe('Tri des résultats. Préfixer par - pour décroissant. Ex: ["-m_visits"]'),
      filter: z
        .object({
          metric: z.record(z.unknown()).optional(),
          property: z.record(z.unknown()).optional(),
        })
        .optional()
        .describe('Filtres Piano. Ex: {"property": {"device_type": {"$eq": "Mobile"}}}'),
      max_results: z
        .number()
        .int()
        .min(1)
        .max(10000)
        .optional()
        .default(100)
        .describe("Nombre max de lignes retournées (défaut: 100, max: 10000)"),
      page: z
        .number()
        .int()
        .min(1)
        .optional()
        .default(1)
        .describe("Numéro de page pour la pagination (défaut: 1)"),
    },
    async ({ columns, site_id, period_preset, period_start, period_end, sort, filter, max_results, page }) => {
      try {
        const resolvedSiteId = resolveSiteId(site_id);
        const period = buildPeriod({ period_preset, period_start, period_end });

        const params: PianoQueryParams = {
          columns,
          space: { s: [resolvedSiteId] },
          period: { p1: [period] },
        };

        if (sort) params.sort = sort;
        if (filter) params.filter = filter;
        if (max_results) params["max-results"] = max_results;
        if (page) params["page-num"] = page;

        const response = await client.getData(params);

        const result = {
          rows: response.DataFeed.Rows,
          total_rows: response.DataFeed.RowCounts.Total,
          page: response.DataFeed.RowCounts.CurPage,
        };

        return {
          content: [{ type: "text" as const, text: JSON.stringify(result, null, 2) }],
        };
      } catch (error) {
        const message =
          error instanceof PianoApiError
            ? `Erreur Piano API (${error.status}): ${error.message}`
            : error instanceof Error
            ? error.message
            : String(error);

        return {
          content: [{ type: "text" as const, text: message }],
          isError: true,
        };
      }
    }
  );
}

function resolveSiteId(siteId: number | undefined): number {
  if (siteId !== undefined) return siteId;

  const defaultRaw = process.env.PIANO_DEFAULT_SITE_ID;
  if (!defaultRaw) {
    throw new Error(
      "site_id non fourni et PIANO_DEFAULT_SITE_ID non configuré"
    );
  }

  const parsed = parseInt(defaultRaw, 10);
  if (isNaN(parsed)) {
    throw new Error(
      `PIANO_DEFAULT_SITE_ID invalide: "${defaultRaw}" (doit être un entier)`
    );
  }
  return parsed;
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/tools/query-data.ts
git commit -m "feat(piano-analytics-mcp): add query_data tool"
```

---

## Task 11 : Outil `query_total`

**Files:**
- Create: `piano-analytics-mcp/src/tools/query-total.ts`

- [ ] **Step 1 : Créer `src/tools/query-total.ts`**

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { PianoClient, PianoApiError } from "../piano/client.js";
import { buildPeriod } from "../piano/period.js";
import type { PianoQueryParams } from "../piano/types.js";

const PERIOD_PRESET_VALUES = [
  "today",
  "yesterday",
  "last_7_days",
  "last_30_days",
  "current_week",
  "last_week",
  "current_month",
  "last_month",
] as const;

export function registerQueryTotalTool(server: McpServer, client: PianoClient): void {
  server.tool(
    "query_total",
    "Retourne les totaux agrégés des métriques Piano Analytics pour une période donnée. Idéal pour les KPIs globaux sans détail par dimension.",
    {
      columns: z
        .array(z.string())
        .min(1)
        .describe('Colonnes à agréger. Ex: ["m_visits", "m_page_views", "m_users"]'),
      site_id: z
        .number()
        .int()
        .optional()
        .describe("ID du site Piano Analytics. Si omis, utilise PIANO_DEFAULT_SITE_ID"),
      period_preset: z
        .enum(PERIOD_PRESET_VALUES)
        .optional()
        .describe("Période prédéfinie"),
      period_start: z
        .string()
        .optional()
        .describe("Date de début ISO 8601 (YYYY-MM-DD)"),
      period_end: z
        .string()
        .optional()
        .describe("Date de fin ISO 8601 (YYYY-MM-DD)"),
      filter: z
        .object({
          metric: z.record(z.unknown()).optional(),
          property: z.record(z.unknown()).optional(),
        })
        .optional()
        .describe("Filtres Piano à appliquer avant l'agrégation"),
    },
    async ({ columns, site_id, period_preset, period_start, period_end, filter }) => {
      try {
        const resolvedSiteId = resolveSiteId(site_id);
        const period = buildPeriod({ period_preset, period_start, period_end });

        const params: PianoQueryParams = {
          columns,
          space: { s: [resolvedSiteId] },
          period: { p1: [period] },
        };

        if (filter) params.filter = filter;

        const response = await client.getTotal(params);

        return {
          content: [
            {
              type: "text" as const,
              text: JSON.stringify({ totals: response.DataFeed.Total }, null, 2),
            },
          ],
        };
      } catch (error) {
        const message =
          error instanceof PianoApiError
            ? `Erreur Piano API (${error.status}): ${error.message}`
            : error instanceof Error
            ? error.message
            : String(error);

        return {
          content: [{ type: "text" as const, text: message }],
          isError: true,
        };
      }
    }
  );
}

function resolveSiteId(siteId: number | undefined): number {
  if (siteId !== undefined) return siteId;

  const defaultRaw = process.env.PIANO_DEFAULT_SITE_ID;
  if (!defaultRaw) {
    throw new Error(
      "site_id non fourni et PIANO_DEFAULT_SITE_ID non configuré"
    );
  }

  const parsed = parseInt(defaultRaw, 10);
  if (isNaN(parsed)) {
    throw new Error(
      `PIANO_DEFAULT_SITE_ID invalide: "${defaultRaw}" (doit être un entier)`
    );
  }
  return parsed;
}
```

- [ ] **Step 2 : Vérifier la compilation**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Commit**

```bash
git add piano-analytics-mcp/src/tools/query-total.ts
git commit -m "feat(piano-analytics-mcp): add query_total tool"
```

---

## Task 12 : Point d'entrée MCP

**Files:**
- Create: `piano-analytics-mcp/src/index.ts`

- [ ] **Step 1 : Créer `src/index.ts`**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { PianoClient } from "./piano/client.js";
import { registerListSitesTool } from "./tools/list-sites.js";
import { registerListPropertiesTool } from "./tools/list-properties.js";
import { registerListMetricsTool } from "./tools/list-metrics.js";
import { registerQueryDataTool } from "./tools/query-data.js";
import { registerQueryTotalTool } from "./tools/query-total.js";

function createServer(): McpServer {
  const apiKey = process.env.PIANO_API_KEY;
  if (!apiKey) {
    throw new Error(
      "Variable d'environnement PIANO_API_KEY manquante. Générer une clé sur my.piano.io/profile/#/apikeys"
    );
  }

  const client = new PianoClient(apiKey);
  const server = new McpServer({
    name: "piano-analytics",
    version: "1.0.0",
  });

  registerListSitesTool(server);
  registerListPropertiesTool(server);
  registerListMetricsTool(server);
  registerQueryDataTool(server, client);
  registerQueryTotalTool(server, client);

  return server;
}

async function main(): Promise<void> {
  const server = createServer();
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch((error) => {
  console.error("Erreur fatale:", error);
  process.exit(1);
});
```

- [ ] **Step 2 : Vérifier la compilation complète**

```bash
npx tsc --noEmit
```

Expected : aucune erreur.

- [ ] **Step 3 : Lancer tous les tests**

```bash
npm test
```

Expected : tous les tests PASS (period × 11, properties × 7, metrics × 7, client × 5).

- [ ] **Step 4 : Compiler en dist/**

```bash
npm run build
```

Expected : dossier `dist/` créé avec `index.js` et les fichiers `.js` correspondants.

- [ ] **Step 5 : Commit**

```bash
git add piano-analytics-mcp/src/index.ts
git commit -m "feat(piano-analytics-mcp): add MCP server entry point"
```

---

## Task 13 : Build final, marketplace et README

**Files:**
- Modify: `/Users/tnabet/Dev/tools/claude-plugins/.claude-plugin/marketplace.json`
- Create: `piano-analytics-mcp/README.md`

- [ ] **Step 1 : Mettre à jour `marketplace.json`**

Ajouter dans le tableau `"plugins"` de `/Users/tnabet/Dev/tools/claude-plugins/.claude-plugin/marketplace.json` :

```json
{
  "name": "piano-analytics-mcp",
  "description": "MCP server for querying Piano Analytics data in natural language. Exposes list_sites, list_properties, list_metrics, query_data, query_total.",
  "version": "1.0.0",
  "source": "./piano-analytics-mcp/",
  "author": {
    "name": "tnabet"
  }
}
```

- [ ] **Step 2 : Créer `piano-analytics-mcp/README.md`**

```markdown
# piano-analytics-mcp

MCP server pour interroger Piano Analytics en langage naturel depuis Claude.

## Prérequis

- Node.js 18+
- Une API key Piano Analytics (générer sur [my.piano.io/profile/#/apikeys](https://my.piano.io/profile/#/apikeys))
  - Rôle requis : Administrateur, Délégué, Analyste Avancé, Analyste, ou rôle personnalisé avec "Gérer les données"

## Installation

```bash
git clone <repo>
cd claude-plugins/piano-analytics-mcp
npm install
npm run build
```

## Configuration Claude Desktop / Claude Code

Ajouter dans `~/.claude/claude_desktop_config.json` (ou `~/.config/claude/claude_desktop_config.json` selon la plateforme) :

```json
{
  "mcpServers": {
    "piano-analytics": {
      "command": "node",
      "args": ["/chemin/absolu/vers/claude-plugins/piano-analytics-mcp/dist/index.js"],
      "env": {
        "PIANO_API_KEY": "votre-cle-api",
        "PIANO_SITE_IDS": "123456,789012",
        "PIANO_DEFAULT_SITE_ID": "123456"
      }
    }
  }
}
```

## Variables d'environnement

| Variable | Requis | Description |
|---|---|---|
| `PIANO_API_KEY` | Oui | Clé API Piano Analytics |
| `PIANO_SITE_IDS` | Oui | IDs de sites séparés par virgule |
| `PIANO_DEFAULT_SITE_ID` | Non | Site utilisé si non précisé dans la requête |

## Outils disponibles

| Outil | Description |
|---|---|
| `list_sites` | Liste les sites configurés |
| `list_properties` | Catalogue des propriétés Piano (filtre par `search`) |
| `list_metrics` | Catalogue des métriques Piano (filtre par `search`) |
| `query_data` | Requête tabulaire getData |
| `query_total` | Agrégats globaux getTotal |

## Exemple d'usage

> "Combien de visites mobiles la semaine dernière ?"

Claude appellera automatiquement :
1. `list_metrics` → trouve `m_visits`
2. `list_properties` → trouve `device_type`
3. `query_data` avec `columns: ["device_type", "m_visits"]`, `period_preset: "last_week"`, filtre Mobile

## Limites API Piano Analytics

- 50 colonnes max par requête
- 10 000 lignes max par appel
- 5 appels concurrents max par utilisateur
```

- [ ] **Step 3 : Lancer les tests une dernière fois**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins/piano-analytics-mcp
npm test
```

Expected : tous les tests PASS.

- [ ] **Step 4 : Build final**

```bash
npm run build
ls dist/
```

Expected : `index.js`, `piano/`, `tools/`, `catalog/` dans `dist/`.

- [ ] **Step 5 : Commit final**

```bash
cd /Users/tnabet/Dev/tools/claude-plugins
git add piano-analytics-mcp/README.md .claude-plugin/marketplace.json
git commit -m "feat(piano-analytics-mcp): add README and register in marketplace"
```
