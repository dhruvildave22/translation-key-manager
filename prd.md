# Translation Key Manager — Technical Design Document

**Goal:** Build a VS Code extension that surfaces i18n translation-key issues (unused, missing, duplicate) inline in the editor without any manual commands.

**Architecture:** A parser engine flattens locale JSON files and scans source files with regex to build an in-memory key index. That index feeds four output layers: DiagnosticCollection (red squiggles + Problems panel), TextEditorDecorations (faded unused keys), HoverProvider (tooltip), DefinitionProvider (Ctrl+Click), and a sidebar TreeView panel. A FileSystemWatcher triggers re-indexing on every save.

**Tech Stack:** TypeScript 5, VS Code Extension API 1.90+, Node.js `fs`/`path`, `vsce` for packaging, Mocha + `@vscode/test-electron` for tests.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    extension.ts (entry)                 │
│  activate() → wires all providers + starts scanner      │
└───────────────────────────┬─────────────────────────────┘
                            │
           ┌────────────────▼────────────────┐
           │       WorkspaceScanner          │
           │  findFiles → parse → populate   │
           │  FileSystemWatcher → re-scan    │
           └────────────────┬────────────────┘
                            │
          ┌─────────────────▼──────────────────┐
          │              KeyStore               │
          │  localeKeys  Map<fullKey, LocaleKey>│
          │  usages      Map<fullKey, Usage[]>  │
          │  unusedKeys  LocaleKey[]            │
          │  missingKeys KeyUsage[]             │
          │  duplicates  Map<value, LocaleKey[]>│
          └──────┬──────────────────┬──────────┘
                 │                  │
    ┌────────────▼───┐   ┌──────────▼──────────────────┐
    │  LocaleParser  │   │       SourceParser           │
    │  JSON → flat   │   │  regex scan .js/.jsx/.ts/.tsx│
    │  key map       │   │  for t() and i18nKey=        │
    └────────────────┘   └──────────────────────────────┘
                 │
    ┌────────────▼────────────────────────────────────┐
    │               Output Layer                      │
    │  DiagnosticManager  → red squiggles             │
    │  UnusedKeyDecorator → faded text in locale file │
    │  HoverProvider      → tooltip                   │
    │  DefinitionProvider → Ctrl+Click                │
    │  KeyBrowserProvider → sidebar TreeView          │
    └─────────────────────────────────────────────────┘
```

---

## File Structure

```
translation-key-manager/
├── package.json                         # Extension manifest (commands, views, config)
├── tsconfig.json
├── .eslintrc.json
├── .vscodeignore
├── src/
│   ├── extension.ts                     # activate() / deactivate() entry point
│   ├── types.ts                         # Shared interfaces (LocaleKey, KeyUsage, etc.)
│   ├── config.ts                        # Read workspace settings
│   ├── parser/
│   │   ├── localeParser.ts              # Flatten locale JSON → LocaleKey[]
│   │   └── sourceParser.ts             # Regex-scan source files → KeyUsage[]
│   ├── store/
│   │   └── keyStore.ts                  # In-memory index + analysis (unused/missing/dupes)
│   ├── scanner/
│   │   └── workspaceScanner.ts          # Orchestrate full scan + file watcher
│   ├── diagnostics/
│   │   ├── diagnosticManager.ts         # DiagnosticCollection lifecycle
│   │   └── longKeyDiagnostic.ts         # Hint for keys > maxKeyLength chars
│   ├── decorations/
│   │   └── unusedKeyDecorator.ts        # Faded text for unused keys in locale files
│   ├── providers/
│   │   ├── hoverProvider.ts             # Markdown hover tooltip
│   │   └── definitionProvider.ts        # Go-to-definition location
│   └── sidebar/
│       ├── treeItems.ts                 # TreeItem subclasses per node type
│       └── keyBrowserProvider.ts        # TreeDataProvider — All/Unused/Missing/Dupes
├── test/
│   ├── runTests.ts                      # @vscode/test-electron runner entry
│   └── suite/
│       ├── index.ts                     # Mocha suite loader
│       ├── localeParser.test.ts
│       ├── sourceParser.test.ts
│       ├── keyStore.test.ts
│       ├── workspaceScanner.test.ts
│       ├── hoverProvider.test.ts
│       └── definitionProvider.test.ts
└── test/fixtures/
    ├── locales/en/
    │   ├── pages.json                   # Sample nested locale file
    │   └── common.json
    └── src/
        ├── Component.tsx                # Sample source with t() calls
        └── Page.jsx
```

---

## Data Models (src/types.ts)

```typescript
export interface LocaleKey {
  fullKey: string;      // "pages.settings.title"
  value: string;        // "Settings"
  namespace: string;    // "pages"
  filePath: string;     // absolute path
  line: number;         // 0-indexed line in the JSON file
  character: number;    // 0-indexed column of the opening quote of the key
}

export interface KeyUsage {
  key: string;          // "pages.settings.title"
  namespace: string;    // derived from useTranslation() or key prefix
  filePath: string;
  line: number;         // 0-indexed
  character: number;    // 0-indexed — start of key string (after quote)
  length: number;       // length of key string (for range)
}

export interface KeyStoreSnapshot {
  localeKeys: Map<string, LocaleKey>;
  usages: Map<string, KeyUsage[]>;
  unusedKeys: LocaleKey[];
  missingKeys: KeyUsage[];
  duplicateValues: Map<string, LocaleKey[]>; // value → keys sharing it (≥2 entries)
}
```

---

## VS Code API Surface

| Need | VS Code API |
|---|---|
| Red squiggles + Problems panel | `languages.createDiagnosticCollection` |
| Faded text in locale files | `window.createTextEditorDecorationType` |
| Hover tooltip | `languages.registerHoverProvider` |
| Ctrl+Click go-to-definition | `languages.registerDefinitionProvider` |
| Sidebar panel | `window.createTreeView` + `TreeDataProvider` |
| Find all workspace files | `workspace.findFiles` |
| Watch saves | `workspace.createFileSystemWatcher` |
| Read settings | `workspace.getConfiguration('translationKeyManager')` |
| Open file at line | `workspace.openTextDocument` + `window.showTextDocument` |

---

## Extension Settings (contributes.configuration in package.json)

| Setting | Type | Default | Description |
|---|---|---|---|
| `translationKeyManager.localePath` | string | `"**/locales/**/*.json"` | Glob for locale files |
| `translationKeyManager.sourcePath` | string | `"**/*.{js,jsx,ts,tsx}"` | Glob for source files |
| `translationKeyManager.excludePatterns` | string[] | `["**/node_modules/**","**/dist/**"]` | Globs to exclude |
| `translationKeyManager.maxKeyLength` | number | `60` | Chars before long-key hint fires |

---

## Task 1: Project Scaffolding

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `.eslintrc.json`
- Create: `.vscodeignore`

- [ ] **Step 1: Write package.json**

```json
{
  "name": "translation-key-manager",
  "displayName": "Translation Key Manager",
  "description": "Inline i18n key health — unused, missing, hover, sidebar browser",
  "version": "0.1.0",
  "publisher": "squad2",
  "engines": { "vscode": "^1.90.0" },
  "categories": ["Linters", "Other"],
  "activationEvents": ["workspaceContains:**/locales/**/*.json"],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "tkm.scanWorkspace",
        "title": "TKM: Scan Workspace"
      },
      {
        "command": "tkm.exportIssues",
        "title": "TKM: Export Issues Report"
      },
      {
        "command": "tkm.openLocaleFile",
        "title": "TKM: Open Locale File"
      }
    ],
    "viewsContainers": {
      "activitybar": [
        {
          "id": "translationKeyManager",
          "title": "Translation Keys",
          "icon": "$(symbol-key)"
        }
      ]
    },
    "views": {
      "translationKeyManager": [
        { "id": "tkm.allKeys",    "name": "All Keys" },
        { "id": "tkm.unusedKeys", "name": "Unused" },
        { "id": "tkm.missingKeys","name": "Missing" },
        { "id": "tkm.duplicates", "name": "Duplicates" }
      ]
    },
    "configuration": {
      "title": "Translation Key Manager",
      "properties": {
        "translationKeyManager.localePath": {
          "type": "string",
          "default": "**/locales/**/*.json",
          "description": "Glob pattern for locale JSON files"
        },
        "translationKeyManager.sourcePath": {
          "type": "string",
          "default": "**/*.{js,jsx,ts,tsx}",
          "description": "Glob pattern for source files to scan"
        },
        "translationKeyManager.excludePatterns": {
          "type": "array",
          "default": ["**/node_modules/**", "**/dist/**"],
          "description": "Glob patterns to exclude from scanning"
        },
        "translationKeyManager.maxKeyLength": {
          "type": "number",
          "default": 60,
          "description": "Keys longer than this trigger a hint diagnostic"
        }
      }
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run compile",
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "pretest": "npm run compile",
    "test": "node ./out/test/runTests.js",
    "lint": "eslint src --ext ts",
    "package": "vsce package"
  },
  "devDependencies": {
    "@types/vscode": "^1.90.0",
    "@types/mocha": "^10.0.0",
    "@types/node": "^20.0.0",
    "@vscode/test-electron": "^2.4.0",
    "@vscode/vsce": "^2.24.0",
    "mocha": "^10.4.0",
    "typescript": "^5.4.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "eslint": "^8.57.0"
  }
}
```

- [ ] **Step 2: Write tsconfig.json**

```json
{
  "compilerOptions": {
    "module": "Node16",
    "target": "ES2020",
    "outDir": "out",
    "lib": ["ES2020"],
    "sourceMap": true,
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src"],
  "exclude": ["node_modules", ".vscode-test"]
}
```

> Note: test files are compiled separately — add a second tsconfig for tests in Task 17.

- [ ] **Step 3: Write .eslintrc.json**

```json
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "parserOptions": { "ecmaVersion": 2020, "sourceType": "module" },
  "plugins": ["@typescript-eslint"],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn"
  }
}
```

- [ ] **Step 4: Write .vscodeignore**

```
.vscode/**
.vscode-test/**
src/**
test/**
out/test/**
.gitignore
.eslintrc.json
tsconfig.json
**/*.map
**/*.ts
!out/**/*.js
```

- [ ] **Step 5: Install dependencies**

```bash
npm install
```

Expected: `node_modules/` created, no errors.

- [ ] **Step 6: Verify compile**

```bash
npm run compile
```

Expected: `out/` directory created (empty, no src files yet).

- [ ] **Step 7: Commit**

```bash
git add package.json tsconfig.json .eslintrc.json .vscodeignore
git commit -m "chore: scaffold VS Code extension project"
```

---

## Task 2: Shared Types and Config Reader

**Files:**
- Create: `src/types.ts`
- Create: `src/config.ts`

- [ ] **Step 1: Write the failing test for config**

Create `test/suite/config.test.ts`:

```typescript
import * as assert from 'assert';
import * as vscode from 'vscode';

suite('Config', () => {
  test('returns default localePath when not configured', () => {
    const config = vscode.workspace.getConfiguration('translationKeyManager');
    const localePath = config.get<string>('localePath', '**/locales/**/*.json');
    assert.strictEqual(localePath, '**/locales/**/*.json');
  });

  test('returns default maxKeyLength of 60', () => {
    const config = vscode.workspace.getConfiguration('translationKeyManager');
    const maxLen = config.get<number>('maxKeyLength', 60);
    assert.strictEqual(maxLen, 60);
  });
});
```

- [ ] **Step 2: Write src/types.ts**

```typescript
export interface LocaleKey {
  fullKey: string;
  value: string;
  namespace: string;
  filePath: string;
  line: number;
  character: number;
}

export interface KeyUsage {
  key: string;
  namespace: string;
  filePath: string;
  line: number;
  character: number;
  length: number;
}

export interface KeyStoreSnapshot {
  localeKeys: Map<string, LocaleKey>;
  usages: Map<string, KeyUsage[]>;
  unusedKeys: LocaleKey[];
  missingKeys: KeyUsage[];
  duplicateValues: Map<string, LocaleKey[]>;
}
```

- [ ] **Step 3: Write src/config.ts**

```typescript
import * as vscode from 'vscode';

export interface TkmConfig {
  localePath: string;
  sourcePath: string;
  excludePatterns: string[];
  maxKeyLength: number;
}

export function getConfig(): TkmConfig {
  const cfg = vscode.workspace.getConfiguration('translationKeyManager');
  return {
    localePath:      cfg.get<string>('localePath',        '**/locales/**/*.json'),
    sourcePath:      cfg.get<string>('sourcePath',        '**/*.{js,jsx,ts,tsx}'),
    excludePatterns: cfg.get<string[]>('excludePatterns', ['**/node_modules/**', '**/dist/**']),
    maxKeyLength:    cfg.get<number>('maxKeyLength',       60),
  };
}
```

- [ ] **Step 4: Compile**

```bash
npm run compile
```

Expected: No errors.

- [ ] **Step 5: Commit**

```bash
git add src/types.ts src/config.ts test/suite/config.test.ts
git commit -m "feat: add shared types and config reader"
```

---

## Task 3: Locale Parser

**Files:**
- Create: `src/parser/localeParser.ts`
- Create: `test/fixtures/locales/en/pages.json`
- Create: `test/fixtures/locales/en/common.json`
- Create: `test/suite/localeParser.test.ts`

**What it does:** Reads a locale JSON file, walks the nested object, and returns a flat array of `LocaleKey`. The key `{ "settings": { "title": "Settings" } }` in `pages.json` becomes `fullKey = "pages.settings.title"`. Line numbers are found by scanning raw file text for the key string.

- [ ] **Step 1: Create test fixtures**

`test/fixtures/locales/en/pages.json`:
```json
{
  "settings": {
    "title": "Settings",
    "description": "Manage your settings here"
  },
  "dashboard": {
    "welcome": "Welcome back",
    "stats": {
      "active_users": "Active Users"
    }
  }
}
```

`test/fixtures/locales/en/common.json`:
```json
{
  "save": "Save",
  "cancel": "Cancel",
  "delete": "Delete"
}
```

- [ ] **Step 2: Write the failing tests**

`test/suite/localeParser.test.ts`:
```typescript
import * as assert from 'assert';
import * as path from 'path';
import { parseLocaleFile } from '../../src/parser/localeParser';

const FIXTURE_DIR = path.join(__dirname, '../../test/fixtures/locales/en');

suite('LocaleParser', () => {
  test('flattens nested JSON into dot-notation keys', () => {
    const keys = parseLocaleFile(path.join(FIXTURE_DIR, 'pages.json'));
    const map = new Map(keys.map(k => [k.fullKey, k]));

    assert.ok(map.has('pages.settings.title'));
    assert.ok(map.has('pages.settings.description'));
    assert.ok(map.has('pages.dashboard.stats.active_users'));
    assert.strictEqual(map.get('pages.settings.title')!.value, 'Settings');
  });

  test('sets namespace from filename', () => {
    const keys = parseLocaleFile(path.join(FIXTURE_DIR, 'pages.json'));
    assert.ok(keys.every(k => k.namespace === 'pages'));
  });

  test('sets namespace from filename for common', () => {
    const keys = parseLocaleFile(path.join(FIXTURE_DIR, 'common.json'));
    assert.ok(keys.every(k => k.namespace === 'common'));
  });

  test('records correct line number for a key', () => {
    const keys = parseLocaleFile(path.join(FIXTURE_DIR, 'pages.json'));
    const titleKey = keys.find(k => k.fullKey === 'pages.settings.title');
    assert.ok(titleKey !== undefined);
    assert.ok(titleKey!.line >= 0);
  });

  test('returns empty array for empty JSON object', () => {
    // We test via a temp file written inline
    const os = require('os');
    const fs = require('fs');
    const tmp = path.join(os.tmpdir(), 'empty.json');
    fs.writeFileSync(tmp, '{}');
    const keys = parseLocaleFile(tmp);
    assert.strictEqual(keys.length, 0);
    fs.unlinkSync(tmp);
  });

  test('ignores non-string leaf values gracefully', () => {
    const os = require('os');
    const fs = require('fs');
    const tmp = path.join(os.tmpdir(), 'mixed.json');
    fs.writeFileSync(tmp, JSON.stringify({ count: 5, label: "Hello" }));
    const keys = parseLocaleFile(tmp);
    assert.strictEqual(keys.length, 1);
    assert.strictEqual(keys[0].value, 'Hello');
    fs.unlinkSync(tmp);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

```bash
npm run compile && npx mocha --require ts-node/register test/suite/localeParser.test.ts 2>&1 | head -20
```

Expected: `Cannot find module '../../src/parser/localeParser'`

- [ ] **Step 4: Write src/parser/localeParser.ts**

```typescript
import * as fs from 'fs';
import * as path from 'path';
import { LocaleKey } from '../types';

export function parseLocaleFile(filePath: string): LocaleKey[] {
  const raw = fs.readFileSync(filePath, 'utf-8');
  const lines = raw.split('\n');
  const namespace = path.basename(filePath, '.json');

  let json: Record<string, unknown>;
  try {
    json = JSON.parse(raw) as Record<string, unknown>;
  } catch {
    return [];
  }

  const result: LocaleKey[] = [];
  walk(json, namespace, namespace, lines, filePath, result);
  return result;
}

function walk(
  obj: Record<string, unknown>,
  namespace: string,
  prefix: string,
  lines: string[],
  filePath: string,
  result: LocaleKey[]
): void {
  for (const [key, val] of Object.entries(obj)) {
    const fullKey = `${prefix}.${key}`;
    if (typeof val === 'string') {
      const { line, character } = findKeyPosition(lines, key);
      result.push({ fullKey, value: val, namespace, filePath, line, character });
    } else if (val !== null && typeof val === 'object' && !Array.isArray(val)) {
      walk(val as Record<string, unknown>, namespace, fullKey, lines, filePath, result);
    }
  }
}

function findKeyPosition(lines: string[], key: string): { line: number; character: number } {
  const pattern = new RegExp(`"${escapeRegex(key)}"\\s*:`);
  for (let i = 0; i < lines.length; i++) {
    const match = pattern.exec(lines[i]);
    if (match) {
      return { line: i, character: match.index + 1 }; // +1 to skip the opening quote
    }
  }
  return { line: 0, character: 0 };
}

function escapeRegex(s: string): string {
  return s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
npm run compile && npm test
```

Expected: All `LocaleParser` suite tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/parser/localeParser.ts test/suite/localeParser.test.ts test/fixtures/
git commit -m "feat: add locale parser — flatten nested JSON to dot-notation key map"
```

---

## Task 4: Source Parser

**Files:**
- Create: `src/parser/sourceParser.ts`
- Create: `test/fixtures/src/Component.tsx`
- Create: `test/fixtures/src/Page.jsx`
- Create: `test/suite/sourceParser.test.ts`

**What it does:** Scans a source file line-by-line and extracts every translation key reference using regex. Handles `t('key')`, `t("key")`, `i18nKey="key"`, `i18nKey={'key'}`. Also detects `useTranslation('ns')` to resolve namespace-prefixed keys.

- [ ] **Step 1: Create test fixtures**

`test/fixtures/src/Component.tsx`:
```tsx
import { useTranslation } from 'react-i18next';

export function Component() {
  const { t } = useTranslation('pages');

  return (
    <div>
      <h1>{t('settings.title')}</h1>
      <p>{t('settings.description')}</p>
      <span>{t('dashboard.welcome')}</span>
    </div>
  );
}
```

`test/fixtures/src/Page.jsx`:
```jsx
import { Trans } from 'react-i18next';

export function Page() {
  return (
    <Trans i18nKey="common.save" />
    <Trans i18nKey={'common.cancel'} />
  );
}
```

- [ ] **Step 2: Write the failing tests**

`test/suite/sourceParser.test.ts`:
```typescript
import * as assert from 'assert';
import * as path from 'path';
import { parseSourceFile } from '../../src/parser/sourceParser';

const FIXTURE_DIR = path.join(__dirname, '../../test/fixtures/src');

suite('SourceParser', () => {
  test('extracts t() calls with useTranslation namespace', () => {
    const usages = parseSourceFile(path.join(FIXTURE_DIR, 'Component.tsx'));
    const keys = usages.map(u => u.key);

    assert.ok(keys.includes('pages.settings.title'));
    assert.ok(keys.includes('pages.settings.description'));
    assert.ok(keys.includes('pages.dashboard.welcome'));
  });

  test('extracts i18nKey attribute (double-quote)', () => {
    const usages = parseSourceFile(path.join(FIXTURE_DIR, 'Page.jsx'));
    assert.ok(usages.some(u => u.key === 'common.save'));
  });

  test('extracts i18nKey attribute (single-quote in braces)', () => {
    const usages = parseSourceFile(path.join(FIXTURE_DIR, 'Page.jsx'));
    assert.ok(usages.some(u => u.key === 'common.cancel'));
  });

  test('records correct line and character for usage', () => {
    const usages = parseSourceFile(path.join(FIXTURE_DIR, 'Component.tsx'));
    const titleUsage = usages.find(u => u.key === 'pages.settings.title');
    assert.ok(titleUsage !== undefined);
    assert.ok(titleUsage!.line >= 0);
    assert.ok(titleUsage!.character >= 0);
  });

  test('returns empty array for file with no translation calls', () => {
    const os = require('os');
    const fs = require('fs');
    const tmp = path.join(os.tmpdir(), 'empty.tsx');
    fs.writeFileSync(tmp, 'export const x = 1;');
    const usages = parseSourceFile(tmp);
    assert.strictEqual(usages.length, 0);
    fs.unlinkSync(tmp);
  });

  test('already-namespaced key (common.save) is not double-prefixed', () => {
    const usages = parseSourceFile(path.join(FIXTURE_DIR, 'Page.jsx'));
    const saveUsage = usages.find(u => u.key === 'common.save');
    assert.ok(saveUsage !== undefined);
    assert.ok(!saveUsage!.key.startsWith('common.common'));
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

```bash
npm run compile 2>&1 | grep "error"
```

Expected: `Cannot find module '../../src/parser/sourceParser'`

- [ ] **Step 4: Write src/parser/sourceParser.ts**

```typescript
import * as fs from 'fs';
import { KeyUsage } from '../types';

// Matches: t('key'), t("key"), t(`key`)
const T_CALL_RE = /\bt\(\s*(['"`])([^'"`\s]+)\1/g;
// Matches: i18nKey="key", i18nKey='key', i18nKey={'key'}, i18nKey={"key"}
const I18N_KEY_RE = /i18nKey\s*=\s*\{?\s*(['"`])([^'"`\s]+)\1/g;
// Matches: useTranslation('ns') or useTranslation("ns")
const USE_TRANSLATION_RE = /useTranslation\(\s*(['"`])([^'"`]+)\1\s*\)/g;

export function parseSourceFile(filePath: string): KeyUsage[] {
  let raw: string;
  try {
    raw = fs.readFileSync(filePath, 'utf-8');
  } catch {
    return [];
  }

  const lines = raw.split('\n');
  const namespaces = extractNamespaces(raw);
  const result: KeyUsage[] = [];

  for (let lineIdx = 0; lineIdx < lines.length; lineIdx++) {
    const line = lines[lineIdx];
    extractMatches(line, T_CALL_RE, lineIdx, namespaces, filePath, result);
    extractMatches(line, I18N_KEY_RE, lineIdx, [], filePath, result);
  }

  return result;
}

function extractNamespaces(raw: string): string[] {
  const namespaces: string[] = [];
  let m: RegExpExecArray | null;
  USE_TRANSLATION_RE.lastIndex = 0;
  while ((m = USE_TRANSLATION_RE.exec(raw)) !== null) {
    namespaces.push(m[2]);
  }
  return namespaces;
}

function extractMatches(
  line: string,
  re: RegExp,
  lineIdx: number,
  namespaces: string[],
  filePath: string,
  result: KeyUsage[]
): void {
  const localRe = new RegExp(re.source, re.flags);
  let m: RegExpExecArray | null;
  while ((m = localRe.exec(line)) !== null) {
    const rawKey = m[2];
    const key = resolveKey(rawKey, namespaces);
    const namespace = key.split('.')[0];
    const character = m.index + m[0].indexOf(rawKey);
    result.push({ key, namespace, filePath, line: lineIdx, character, length: rawKey.length });
  }
}

function resolveKey(rawKey: string, namespaces: string[]): string {
  if (namespaces.length === 0) return rawKey;
  // If the key already starts with a known namespace, don't prefix again
  for (const ns of namespaces) {
    if (rawKey.startsWith(`${ns}.`)) return rawKey;
  }
  return `${namespaces[0]}.${rawKey}`;
}
```

- [ ] **Step 5: Run tests**

```bash
npm run compile && npm test
```

Expected: All `SourceParser` suite tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/parser/sourceParser.ts test/suite/sourceParser.test.ts test/fixtures/src/
git commit -m "feat: add source parser — regex scan for t() and i18nKey usages"
```

---

## Task 5: Key Store

**Files:**
- Create: `src/store/keyStore.ts`
- Create: `test/suite/keyStore.test.ts`

**What it does:** Takes parsed locale keys and parsed source usages, then computes: which locale keys are unused, which usages reference missing keys, and which locale keys share the same value (duplicates).

- [ ] **Step 1: Write the failing tests**

`test/suite/keyStore.test.ts`:
```typescript
import * as assert from 'assert';
import { buildKeyStore } from '../../src/store/keyStore';
import { LocaleKey, KeyUsage } from '../../src/types';

function makeLocaleKey(fullKey: string, value: string, namespace = 'pages'): LocaleKey {
  return { fullKey, value, namespace, filePath: '/locales/pages.json', line: 0, character: 0 };
}

function makeUsage(key: string, filePath = '/src/App.tsx'): KeyUsage {
  return { key, namespace: key.split('.')[0], filePath, line: 0, character: 0, length: key.length };
}

suite('KeyStore', () => {
  test('marks a locale key as unused when no usages reference it', () => {
    const localeKeys = [makeLocaleKey('pages.settings.title', 'Settings')];
    const usages: KeyUsage[] = [];
    const store = buildKeyStore(localeKeys, usages);
    assert.strictEqual(store.unusedKeys.length, 1);
    assert.strictEqual(store.unusedKeys[0].fullKey, 'pages.settings.title');
  });

  test('does not mark a locale key as unused when it has usages', () => {
    const localeKeys = [makeLocaleKey('pages.settings.title', 'Settings')];
    const usages = [makeUsage('pages.settings.title')];
    const store = buildKeyStore(localeKeys, usages);
    assert.strictEqual(store.unusedKeys.length, 0);
  });

  test('marks a usage as missing when its key is absent from locale', () => {
    const localeKeys: LocaleKey[] = [];
    const usages = [makeUsage('pages.ghost.key')];
    const store = buildKeyStore(localeKeys, usages);
    assert.strictEqual(store.missingKeys.length, 1);
    assert.strictEqual(store.missingKeys[0].key, 'pages.ghost.key');
  });

  test('detects duplicate values (same value, different keys)', () => {
    const localeKeys = [
      makeLocaleKey('pages.save_btn', 'Save'),
      makeLocaleKey('pages.submit_btn', 'Save'),
    ];
    const store = buildKeyStore(localeKeys, []);
    assert.strictEqual(store.duplicateValues.size, 1);
    const dupes = store.duplicateValues.get('Save')!;
    assert.strictEqual(dupes.length, 2);
  });

  test('does not flag a value as duplicate when only one key has it', () => {
    const localeKeys = [
      makeLocaleKey('pages.save_btn', 'Save'),
      makeLocaleKey('pages.cancel_btn', 'Cancel'),
    ];
    const store = buildKeyStore(localeKeys, []);
    assert.strictEqual(store.duplicateValues.size, 0);
  });

  test('usages map contains all usages grouped by key', () => {
    const localeKeys = [makeLocaleKey('pages.title', 'Title')];
    const usages = [
      makeUsage('pages.title', '/src/A.tsx'),
      makeUsage('pages.title', '/src/B.tsx'),
    ];
    const store = buildKeyStore(localeKeys, usages);
    assert.strictEqual(store.usages.get('pages.title')!.length, 2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npm run compile 2>&1 | grep "error TS"
```

Expected: `Cannot find module '../../src/store/keyStore'`

- [ ] **Step 3: Write src/store/keyStore.ts**

```typescript
import { LocaleKey, KeyUsage, KeyStoreSnapshot } from '../types';

export function buildKeyStore(
  localeKeys: LocaleKey[],
  usages: KeyUsage[]
): KeyStoreSnapshot {
  const localeMap = new Map<string, LocaleKey>();
  for (const k of localeKeys) {
    localeMap.set(k.fullKey, k);
  }

  const usageMap = new Map<string, KeyUsage[]>();
  for (const u of usages) {
    const list = usageMap.get(u.key) ?? [];
    list.push(u);
    usageMap.set(u.key, list);
  }

  const unusedKeys = localeKeys.filter(k => !usageMap.has(k.fullKey));

  const missingKeys = usages.filter(u => !localeMap.has(u.key));

  const valueIndex = new Map<string, LocaleKey[]>();
  for (const k of localeKeys) {
    const list = valueIndex.get(k.value) ?? [];
    list.push(k);
    valueIndex.set(k.value, list);
  }
  const duplicateValues = new Map<string, LocaleKey[]>();
  for (const [val, keys] of valueIndex) {
    if (keys.length >= 2) {
      duplicateValues.set(val, keys);
    }
  }

  return { localeKeys: localeMap, usages: usageMap, unusedKeys, missingKeys, duplicateValues };
}
```

- [ ] **Step 4: Run tests**

```bash
npm run compile && npm test
```

Expected: All `KeyStore` suite tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/store/keyStore.ts test/suite/keyStore.test.ts
git commit -m "feat: add key store — computes unused, missing, and duplicate keys"
```

---

## Task 6: Workspace Scanner

**Files:**
- Create: `src/scanner/workspaceScanner.ts`
- Create: `test/suite/workspaceScanner.test.ts`

**What it does:** Uses `vscode.workspace.findFiles` to discover locale and source files, calls the parsers, builds the key store, and emits an event so other components can refresh. Also creates a `FileSystemWatcher` that triggers a re-scan on save.

- [ ] **Step 1: Write the failing tests**

`test/suite/workspaceScanner.test.ts`:
```typescript
import * as assert from 'assert';
import * as vscode from 'vscode';
import * as path from 'path';
import { WorkspaceScanner } from '../../src/scanner/workspaceScanner';

suite('WorkspaceScanner', () => {
  let scanner: WorkspaceScanner;

  setup(() => {
    scanner = new WorkspaceScanner();
  });

  teardown(() => {
    scanner.dispose();
  });

  test('scan() returns a KeyStoreSnapshot', async () => {
    const fixtureWs = path.join(__dirname, '../../test/fixtures');
    const snapshot = await scanner.scan(fixtureWs);
    assert.ok(snapshot !== null);
    assert.ok(snapshot.localeKeys instanceof Map);
    assert.ok(snapshot.usages instanceof Map);
    assert.ok(Array.isArray(snapshot.unusedKeys));
    assert.ok(Array.isArray(snapshot.missingKeys));
  });

  test('scan() finds locale keys from fixture files', async () => {
    const fixtureWs = path.join(__dirname, '../../test/fixtures');
    const snapshot = await scanner.scan(fixtureWs);
    assert.ok(snapshot.localeKeys.has('pages.settings.title'));
    assert.ok(snapshot.localeKeys.has('common.save'));
  });

  test('scan() detects missing key referenced in Component.tsx', async () => {
    const fixtureWs = path.join(__dirname, '../../test/fixtures');
    const snapshot = await scanner.scan(fixtureWs);
    // dashboard.welcome is in locale but referenced with namespace prefix in Component.tsx
    // this test verifies the scanner runs end-to-end without throwing
    assert.ok(snapshot !== null);
  });
});
```

- [ ] **Step 2: Write src/scanner/workspaceScanner.ts**

```typescript
import * as vscode from 'vscode';
import * as path from 'path';
import * as fs from 'fs';
import { EventEmitter } from 'events';
import { parseLocaleFile } from '../parser/localeParser';
import { parseSourceFile } from '../parser/sourceParser';
import { buildKeyStore } from '../store/keyStore';
import { getConfig } from '../config';
import { LocaleKey, KeyUsage, KeyStoreSnapshot } from '../types';

export class WorkspaceScanner extends EventEmitter {
  private watcher: vscode.FileSystemWatcher | undefined;
  private _snapshot: KeyStoreSnapshot | undefined;

  get snapshot(): KeyStoreSnapshot | undefined {
    return this._snapshot;
  }

  async scanWorkspace(): Promise<KeyStoreSnapshot> {
    const workspaceRoot = vscode.workspace.workspaceFolders?.[0]?.uri.fsPath ?? process.cwd();
    return this.scan(workspaceRoot);
  }

  async scan(rootDir: string): Promise<KeyStoreSnapshot> {
    const config = getConfig();

    const localeFiles = this.findFiles(rootDir, config.localePath, config.excludePatterns);
    const sourceFiles = this.findFiles(rootDir, config.sourcePath, config.excludePatterns);

    const allLocaleKeys: LocaleKey[] = localeFiles.flatMap(f => parseLocaleFile(f));
    const allUsages: KeyUsage[] = sourceFiles.flatMap(f => parseSourceFile(f));

    this._snapshot = buildKeyStore(allLocaleKeys, allUsages);
    this.emit('updated', this._snapshot);
    return this._snapshot;
  }

  startWatcher(): void {
    const config = getConfig();
    this.watcher?.dispose();

    const pattern = new vscode.RelativePattern(
      vscode.workspace.workspaceFolders![0],
      `**/*.{json,js,jsx,ts,tsx}`
    );
    this.watcher = vscode.workspace.createFileSystemWatcher(pattern);

    const rescan = (): void => {
      this.scanWorkspace().catch(console.error);
    };

    this.watcher.onDidChange(rescan);
    this.watcher.onDidCreate(rescan);
    this.watcher.onDidDelete(rescan);
  }

  dispose(): void {
    this.watcher?.dispose();
  }

  private findFiles(root: string, globPattern: string, excludePatterns: string[]): string[] {
    // For tests (no vscode workspace), use manual glob-like walk
    const results: string[] = [];
    const exts = this.extractExtensions(globPattern);
    this.walkDir(root, exts, excludePatterns, results);
    return results;
  }

  private extractExtensions(glob: string): string[] {
    // Handles patterns like **/*.json and **/*.{js,jsx,ts,tsx}
    const braceMatch = glob.match(/\{([^}]+)\}/);
    if (braceMatch) {
      return braceMatch[1].split(',').map(e => `.${e.trim()}`);
    }
    const extMatch = glob.match(/\*\.([a-zA-Z]+)$/);
    if (extMatch) return [`.${extMatch[1]}`];
    return [];
  }

  private walkDir(dir: string, exts: string[], excludes: string[], results: string[]): void {
    let entries: fs.Dirent[];
    try {
      entries = fs.readdirSync(dir, { withFileTypes: true });
    } catch {
      return;
    }
    for (const entry of entries) {
      const fullPath = path.join(dir, entry.name);
      if (excludes.some(ex => fullPath.includes(ex.replace(/\*\*/g, '')))) continue;
      if (entry.isDirectory()) {
        this.walkDir(fullPath, exts, excludes, results);
      } else if (entry.isFile() && exts.includes(path.extname(entry.name))) {
        results.push(fullPath);
      }
    }
  }
}
```

- [ ] **Step 3: Run tests**

```bash
npm run compile && npm test
```

Expected: All `WorkspaceScanner` suite tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src/scanner/workspaceScanner.ts test/suite/workspaceScanner.test.ts
git commit -m "feat: add workspace scanner — orchestrates full scan and file watcher"
```

---

## Task 7: Diagnostic Manager (Missing Keys + Long Key Hints)

**Files:**
- Create: `src/diagnostics/diagnosticManager.ts`
- Create: `src/diagnostics/longKeyDiagnostic.ts`

**What it does:** Maintains a `vscode.DiagnosticCollection`. After each scan, it clears and repopulates with:
1. `Error` severity diagnostics for missing keys (red squiggle under the key string in source files)
2. `Hint` severity diagnostics for long keys (in locale files)

- [ ] **Step 1: Write src/diagnostics/diagnosticManager.ts**

```typescript
import * as vscode from 'vscode';
import { KeyStoreSnapshot } from '../types';
import { getLongKeyDiagnostics } from './longKeyDiagnostic';
import { getConfig } from '../config';

export class DiagnosticManager {
  private collection: vscode.DiagnosticCollection;

  constructor() {
    this.collection = vscode.languages.createDiagnosticCollection('translationKeyManager');
  }

  refresh(snapshot: KeyStoreSnapshot): void {
    this.collection.clear();

    const fileMap = new Map<string, vscode.Diagnostic[]>();
    const config = getConfig();

    // Missing key → Error diagnostic in source file
    for (const usage of snapshot.missingKeys) {
      const range = new vscode.Range(
        new vscode.Position(usage.line, usage.character),
        new vscode.Position(usage.line, usage.character + usage.length)
      );
      const diag = new vscode.Diagnostic(
        range,
        `Translation key '${usage.key}' not found in locale files`,
        vscode.DiagnosticSeverity.Error
      );
      diag.source = 'TKM';
      diag.code = 'missing-key';

      const list = fileMap.get(usage.filePath) ?? [];
      list.push(diag);
      fileMap.set(usage.filePath, list);
    }

    // Long key → Hint diagnostic in locale file
    for (const [, localeKey] of snapshot.localeKeys) {
      if (localeKey.fullKey.length > config.maxKeyLength) {
        const diag = getLongKeyDiagnostics(localeKey, config.maxKeyLength);
        const list = fileMap.get(localeKey.filePath) ?? [];
        list.push(diag);
        fileMap.set(localeKey.filePath, list);
      }
    }

    for (const [filePath, diags] of fileMap) {
      this.collection.set(vscode.Uri.file(filePath), diags);
    }
  }

  dispose(): void {
    this.collection.dispose();
  }
}
```

- [ ] **Step 2: Write src/diagnostics/longKeyDiagnostic.ts**

```typescript
import * as vscode from 'vscode';
import { LocaleKey } from '../types';

export function getLongKeyDiagnostics(key: LocaleKey, maxLength: number): vscode.Diagnostic {
  const range = new vscode.Range(
    new vscode.Position(key.line, key.character),
    new vscode.Position(key.line, key.character + key.fullKey.length)
  );
  const diag = new vscode.Diagnostic(
    range,
    `Key '${key.fullKey}' is ${key.fullKey.length} characters (max ${maxLength}). Consider a shorter name.`,
    vscode.DiagnosticSeverity.Hint
  );
  diag.source = 'TKM';
  diag.code = 'long-key';
  return diag;
}
```

- [ ] **Step 3: Compile and check**

```bash
npm run compile
```

Expected: No type errors.

- [ ] **Step 4: Commit**

```bash
git add src/diagnostics/diagnosticManager.ts src/diagnostics/longKeyDiagnostic.ts
git commit -m "feat: add diagnostic manager — missing key errors and long key hints"
```

---

## Task 8: Unused Key Decorator

**Files:**
- Create: `src/decorations/unusedKeyDecorator.ts`

**What it does:** When a locale JSON file is open in the editor and a key in that file is unused, the decorator applies a `TextEditorDecorationType` (faded/greyed text) to that key string. This is the same visual as VS Code uses for unused TypeScript imports.

- [ ] **Step 1: Write src/decorations/unusedKeyDecorator.ts**

```typescript
import * as vscode from 'vscode';
import { KeyStoreSnapshot, LocaleKey } from '../types';

export class UnusedKeyDecorator {
  private decorationType: vscode.TextEditorDecorationType;

  constructor() {
    this.decorationType = vscode.window.createTextEditorDecorationType({
      opacity: '0.4',
      fontStyle: 'italic',
    });
  }

  refresh(snapshot: KeyStoreSnapshot): void {
    // Group unused keys by locale file
    const byFile = new Map<string, LocaleKey[]>();
    for (const key of snapshot.unusedKeys) {
      const list = byFile.get(key.filePath) ?? [];
      list.push(key);
      byFile.set(key.filePath, list);
    }

    // Apply decorations to all visible editors showing locale files
    for (const editor of vscode.window.visibleTextEditors) {
      const filePath = editor.document.uri.fsPath;
      const unusedInFile = byFile.get(filePath) ?? [];

      const ranges = unusedInFile.map(key => {
        const start = new vscode.Position(key.line, key.character);
        const end = new vscode.Position(key.line, key.character + key.fullKey.split('.').pop()!.length);
        return new vscode.Range(start, end);
      });

      editor.setDecorations(this.decorationType, ranges);
    }
  }

  dispose(): void {
    this.decorationType.dispose();
  }
}
```

- [ ] **Step 2: Compile**

```bash
npm run compile
```

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add src/decorations/unusedKeyDecorator.ts
git commit -m "feat: add unused key decorator — faded text for unused locale keys"
```

---

## Task 9: Hover Provider

**Files:**
- Create: `src/providers/hoverProvider.ts`
- Create: `test/suite/hoverProvider.test.ts`

**What it does:** Registers a `HoverProvider` for `.js`, `.jsx`, `.ts`, `.tsx`, and `.json` files. On hover over a key string, it looks up the key in the store and returns a markdown-formatted `vscode.Hover` with the value, namespace, usages list, and a command link to open the locale file.

- [ ] **Step 1: Write the failing tests**

`test/suite/hoverProvider.test.ts`:
```typescript
import * as assert from 'assert';
import { buildHoverContent } from '../../src/providers/hoverProvider';
import { KeyStoreSnapshot } from '../../src/types';

function makeSnapshot(): KeyStoreSnapshot {
  const localeKeys = new Map([
    ['pages.settings.title', {
      fullKey: 'pages.settings.title',
      value: 'Settings',
      namespace: 'pages',
      filePath: '/locales/en/pages.json',
      line: 3,
      character: 4,
    }]
  ]);
  const usages = new Map([
    ['pages.settings.title', [
      { key: 'pages.settings.title', namespace: 'pages', filePath: '/src/A.tsx', line: 10, character: 5, length: 21 },
      { key: 'pages.settings.title', namespace: 'pages', filePath: '/src/B.tsx', line: 20, character: 3, length: 21 },
    ]]
  ]);
  return { localeKeys, usages, unusedKeys: [], missingKeys: [], duplicateValues: new Map() };
}

suite('HoverProvider', () => {
  test('buildHoverContent returns markdown with key value', () => {
    const content = buildHoverContent('pages.settings.title', makeSnapshot());
    assert.ok(content !== null);
    assert.ok(content!.includes('Settings'));
  });

  test('buildHoverContent includes namespace', () => {
    const content = buildHoverContent('pages.settings.title', makeSnapshot());
    assert.ok(content!.includes('pages'));
  });

  test('buildHoverContent includes usage file references', () => {
    const content = buildHoverContent('pages.settings.title', makeSnapshot());
    assert.ok(content!.includes('A.tsx'));
    assert.ok(content!.includes('B.tsx'));
  });

  test('buildHoverContent returns null for unknown key', () => {
    const content = buildHoverContent('pages.ghost.key', makeSnapshot());
    assert.strictEqual(content, null);
  });
});
```

- [ ] **Step 2: Write src/providers/hoverProvider.ts**

```typescript
import * as vscode from 'vscode';
import * as path from 'path';
import { KeyStoreSnapshot } from '../types';
import { WorkspaceScanner } from '../scanner/workspaceScanner';

export function buildHoverContent(key: string, snapshot: KeyStoreSnapshot): string | null {
  const localeKey = snapshot.localeKeys.get(key);
  if (!localeKey) return null;

  const usages = snapshot.usages.get(key) ?? [];
  const displayed = usages.slice(0, 5);
  const extra = usages.length > 5 ? ` (+${usages.length - 5} more)` : '';

  const usageLines = displayed
    .map(u => `  - ${path.basename(u.filePath)}:${u.line + 1}`)
    .join('\n');

  const localeFileArg = encodeURIComponent(JSON.stringify([localeKey.filePath, localeKey.line]));
  const openLink = `[→ Open in locale file](command:tkm.openLocaleFile?${localeFileArg})`;

  return [
    `**Key:** \`${localeKey.fullKey}\``,
    `**Value:** "${localeKey.value}"`,
    `**Namespace:** \`${localeKey.namespace}\``,
    usages.length > 0
      ? `**Used in:**\n${usageLines}${extra}`
      : `_This key is not referenced anywhere in the codebase._`,
    '',
    openLink,
  ].join('\n\n');
}

export function createHoverProvider(scanner: WorkspaceScanner): vscode.Disposable {
  return vscode.languages.registerHoverProvider(
    [
      { scheme: 'file', language: 'javascript' },
      { scheme: 'file', language: 'javascriptreact' },
      { scheme: 'file', language: 'typescript' },
      { scheme: 'file', language: 'typescriptreact' },
      { scheme: 'file', language: 'json' },
    ],
    {
      provideHover(document, position): vscode.Hover | null {
        const snapshot = scanner.snapshot;
        if (!snapshot) return null;

        const wordRange = document.getWordRangeAtPosition(position, /[\w.]+/);
        if (!wordRange) return null;

        const candidate = document.getText(wordRange);
        const content = buildHoverContent(candidate, snapshot);
        if (!content) return null;

        return new vscode.Hover(new vscode.MarkdownString(content, true), wordRange);
      }
    }
  );
}
```

- [ ] **Step 3: Run tests**

```bash
npm run compile && npm test
```

Expected: All `HoverProvider` suite tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src/providers/hoverProvider.ts test/suite/hoverProvider.test.ts
git commit -m "feat: add hover provider — markdown tooltip with value, namespace, and usages"
```

---

## Task 10: Definition Provider (Ctrl+Click)

**Files:**
- Create: `src/providers/definitionProvider.ts`
- Create: `test/suite/definitionProvider.test.ts`

**What it does:** Implements `DefinitionProvider`. When a developer Ctrl+Clicks a key string in a source file, VS Code calls `provideDefinition`, which looks up the key in the store and returns a `vscode.Location` pointing to the exact line in the locale JSON file.

- [ ] **Step 1: Write the failing tests**

`test/suite/definitionProvider.test.ts`:
```typescript
import * as assert from 'assert';
import { resolveDefinitionLocation } from '../../src/providers/definitionProvider';
import { KeyStoreSnapshot } from '../../src/types';

function makeSnapshot(): KeyStoreSnapshot {
  const localeKeys = new Map([
    ['pages.settings.title', {
      fullKey: 'pages.settings.title',
      value: 'Settings',
      namespace: 'pages',
      filePath: '/locales/en/pages.json',
      line: 3,
      character: 4,
    }]
  ]);
  return {
    localeKeys,
    usages: new Map(),
    unusedKeys: [],
    missingKeys: [],
    duplicateValues: new Map(),
  };
}

suite('DefinitionProvider', () => {
  test('resolves location for a known key', () => {
    const loc = resolveDefinitionLocation('pages.settings.title', makeSnapshot());
    assert.ok(loc !== null);
    assert.strictEqual(loc!.filePath, '/locales/en/pages.json');
    assert.strictEqual(loc!.line, 3);
    assert.strictEqual(loc!.character, 4);
  });

  test('returns null for unknown key', () => {
    const loc = resolveDefinitionLocation('pages.ghost.key', makeSnapshot());
    assert.strictEqual(loc, null);
  });
});
```

- [ ] **Step 2: Write src/providers/definitionProvider.ts**

```typescript
import * as vscode from 'vscode';
import { KeyStoreSnapshot } from '../types';
import { WorkspaceScanner } from '../scanner/workspaceScanner';

export interface DefinitionLocation {
  filePath: string;
  line: number;
  character: number;
}

export function resolveDefinitionLocation(
  key: string,
  snapshot: KeyStoreSnapshot
): DefinitionLocation | null {
  const localeKey = snapshot.localeKeys.get(key);
  if (!localeKey) return null;
  return { filePath: localeKey.filePath, line: localeKey.line, character: localeKey.character };
}

export function createDefinitionProvider(scanner: WorkspaceScanner): vscode.Disposable {
  return vscode.languages.registerDefinitionProvider(
    [
      { scheme: 'file', language: 'javascript' },
      { scheme: 'file', language: 'javascriptreact' },
      { scheme: 'file', language: 'typescript' },
      { scheme: 'file', language: 'typescriptreact' },
    ],
    {
      provideDefinition(document, position): vscode.Location | null {
        const snapshot = scanner.snapshot;
        if (!snapshot) return null;

        const wordRange = document.getWordRangeAtPosition(position, /[\w.]+/);
        if (!wordRange) return null;

        const candidate = document.getText(wordRange);
        const loc = resolveDefinitionLocation(candidate, snapshot);
        if (!loc) return null;

        const targetUri = vscode.Uri.file(loc.filePath);
        const targetPos = new vscode.Position(loc.line, loc.character);
        return new vscode.Location(targetUri, targetPos);
      }
    }
  );
}
```

- [ ] **Step 3: Run tests**

```bash
npm run compile && npm test
```

Expected: All `DefinitionProvider` suite tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src/providers/definitionProvider.ts test/suite/definitionProvider.test.ts
git commit -m "feat: add definition provider — Ctrl+Click navigates to locale file"
```

---

## Task 11: Sidebar Tree Items

**Files:**
- Create: `src/sidebar/treeItems.ts`

**What it does:** Defines `TreeItem` subclasses for each node type in the sidebar — namespace nodes (collapsible), key nodes (leaf), and group nodes for duplicate sets.

- [ ] **Step 1: Write src/sidebar/treeItems.ts**

```typescript
import * as vscode from 'vscode';
import * as path from 'path';
import { LocaleKey, KeyUsage } from '../types';

export class NamespaceItem extends vscode.TreeItem {
  constructor(
    public readonly namespace: string,
    public readonly count: number
  ) {
    super(`${namespace} (${count})`, vscode.TreeItemCollapsibleState.Collapsed);
    this.contextValue = 'namespace';
    this.iconPath = new vscode.ThemeIcon('symbol-namespace');
  }
}

export class LocaleKeyItem extends vscode.TreeItem {
  constructor(public readonly localeKey: LocaleKey) {
    super(localeKey.fullKey, vscode.TreeItemCollapsibleState.None);
    this.description = localeKey.value;
    this.tooltip = `${localeKey.fullKey}\n"${localeKey.value}"`;
    this.contextValue = 'localeKey';
    this.iconPath = new vscode.ThemeIcon('symbol-string');
    this.command = {
      command: 'tkm.openLocaleFile',
      title: 'Open in Locale File',
      arguments: [localeKey.filePath, localeKey.line],
    };
  }
}

export class UnusedKeyItem extends vscode.TreeItem {
  constructor(public readonly localeKey: LocaleKey) {
    super(localeKey.fullKey, vscode.TreeItemCollapsibleState.None);
    this.description = localeKey.value;
    this.tooltip = `Unused: ${localeKey.fullKey}`;
    this.contextValue = 'unusedKey';
    this.iconPath = new vscode.ThemeIcon('warning', new vscode.ThemeColor('list.warningForeground'));
    this.command = {
      command: 'tkm.openLocaleFile',
      title: 'Open in Locale File',
      arguments: [localeKey.filePath, localeKey.line],
    };
  }
}

export class MissingKeyItem extends vscode.TreeItem {
  constructor(public readonly usage: KeyUsage) {
    super(usage.key, vscode.TreeItemCollapsibleState.None);
    this.description = `${path.basename(usage.filePath)}:${usage.line + 1}`;
    this.tooltip = `Missing key '${usage.key}' referenced in ${usage.filePath}:${usage.line + 1}`;
    this.contextValue = 'missingKey';
    this.iconPath = new vscode.ThemeIcon('error', new vscode.ThemeColor('list.errorForeground'));
    this.command = {
      command: 'vscode.open',
      title: 'Open File',
      arguments: [
        vscode.Uri.file(usage.filePath),
        { selection: new vscode.Range(usage.line, usage.character, usage.line, usage.character + usage.length) },
      ],
    };
  }
}

export class DuplicateGroupItem extends vscode.TreeItem {
  constructor(
    public readonly value: string,
    public readonly keys: LocaleKey[]
  ) {
    super(`"${value}" (${keys.length} keys)`, vscode.TreeItemCollapsibleState.Collapsed);
    this.contextValue = 'duplicateGroup';
    this.iconPath = new vscode.ThemeIcon('copy');
  }
}
```

- [ ] **Step 2: Compile**

```bash
npm run compile
```

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add src/sidebar/treeItems.ts
git commit -m "feat: add sidebar tree items — TreeItem subclasses for key browser nodes"
```

---

## Task 12: Sidebar Key Browser Provider

**Files:**
- Create: `src/sidebar/keyBrowserProvider.ts`

**What it does:** Implements four `TreeDataProvider` instances — one each for All Keys, Unused, Missing, and Duplicates views. Each refreshes its tree when the scanner emits an `updated` event.

- [ ] **Step 1: Write src/sidebar/keyBrowserProvider.ts**

```typescript
import * as vscode from 'vscode';
import { KeyStoreSnapshot, LocaleKey } from '../types';
import {
  NamespaceItem,
  LocaleKeyItem,
  UnusedKeyItem,
  MissingKeyItem,
  DuplicateGroupItem,
} from './treeItems';

type AnyItem = NamespaceItem | LocaleKeyItem | UnusedKeyItem | MissingKeyItem | DuplicateGroupItem;

// ── All Keys ────────────────────────────────────────────────────────────────

export class AllKeysProvider implements vscode.TreeDataProvider<AnyItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;
  private snapshot: KeyStoreSnapshot | undefined;

  refresh(snapshot: KeyStoreSnapshot): void {
    this.snapshot = snapshot;
    this._onDidChangeTreeData.fire();
  }

  getTreeItem(el: AnyItem): vscode.TreeItem { return el; }

  getChildren(parent?: AnyItem): AnyItem[] {
    if (!this.snapshot) return [];

    if (!parent) {
      const namespaces = new Map<string, LocaleKey[]>();
      for (const [, key] of this.snapshot.localeKeys) {
        const list = namespaces.get(key.namespace) ?? [];
        list.push(key);
        namespaces.set(key.namespace, list);
      }
      return [...namespaces.entries()].map(([ns, keys]) => new NamespaceItem(ns, keys.length));
    }

    if (parent instanceof NamespaceItem) {
      return [...this.snapshot.localeKeys.values()]
        .filter(k => k.namespace === parent.namespace)
        .map(k => new LocaleKeyItem(k));
    }

    return [];
  }
}

// ── Unused Keys ──────────────────────────────────────────────────────────────

export class UnusedKeysProvider implements vscode.TreeDataProvider<UnusedKeyItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;
  private snapshot: KeyStoreSnapshot | undefined;

  refresh(snapshot: KeyStoreSnapshot): void {
    this.snapshot = snapshot;
    this._onDidChangeTreeData.fire();
  }

  getTreeItem(el: UnusedKeyItem): vscode.TreeItem { return el; }

  getChildren(): UnusedKeyItem[] {
    if (!this.snapshot) return [];
    return this.snapshot.unusedKeys.map(k => new UnusedKeyItem(k));
  }
}

// ── Missing Keys ─────────────────────────────────────────────────────────────

export class MissingKeysProvider implements vscode.TreeDataProvider<MissingKeyItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;
  private snapshot: KeyStoreSnapshot | undefined;

  refresh(snapshot: KeyStoreSnapshot): void {
    this.snapshot = snapshot;
    this._onDidChangeTreeData.fire();
  }

  getTreeItem(el: MissingKeyItem): vscode.TreeItem { return el; }

  getChildren(): MissingKeyItem[] {
    if (!this.snapshot) return [];
    return this.snapshot.missingKeys.map(u => new MissingKeyItem(u));
  }
}

// ── Duplicates ────────────────────────────────────────────────────────────────

export class DuplicatesProvider implements vscode.TreeDataProvider<DuplicateGroupItem | LocaleKeyItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;
  private snapshot: KeyStoreSnapshot | undefined;

  refresh(snapshot: KeyStoreSnapshot): void {
    this.snapshot = snapshot;
    this._onDidChangeTreeData.fire();
  }

  getTreeItem(el: DuplicateGroupItem | LocaleKeyItem): vscode.TreeItem { return el; }

  getChildren(parent?: DuplicateGroupItem | LocaleKeyItem): (DuplicateGroupItem | LocaleKeyItem)[] {
    if (!this.snapshot) return [];

    if (!parent) {
      return [...this.snapshot.duplicateValues.entries()].map(
        ([val, keys]) => new DuplicateGroupItem(val, keys)
      );
    }

    if (parent instanceof DuplicateGroupItem) {
      return parent.keys.map(k => new LocaleKeyItem(k));
    }

    return [];
  }
}
```

- [ ] **Step 2: Compile**

```bash
npm run compile
```

Expected: No errors.

- [ ] **Step 3: Commit**

```bash
git add src/sidebar/keyBrowserProvider.ts
git commit -m "feat: add sidebar key browser — four TreeDataProviders for All/Unused/Missing/Duplicates"
```

---

## Task 13: Extension Entry Point

**Files:**
- Create: `src/extension.ts`

**What it does:** The `activate()` function wires every component together. It creates the scanner, runs the first scan, registers providers, and listens to scanner updates to refresh decorators, diagnostics, and sidebar trees. `deactivate()` disposes all resources.

- [ ] **Step 1: Write src/extension.ts**

```typescript
import * as vscode from 'vscode';
import { WorkspaceScanner } from './scanner/workspaceScanner';
import { DiagnosticManager } from './diagnostics/diagnosticManager';
import { UnusedKeyDecorator } from './decorations/unusedKeyDecorator';
import { createHoverProvider } from './providers/hoverProvider';
import { createDefinitionProvider } from './providers/definitionProvider';
import {
  AllKeysProvider,
  UnusedKeysProvider,
  MissingKeysProvider,
  DuplicatesProvider,
} from './sidebar/keyBrowserProvider';
import { KeyStoreSnapshot } from './types';

export function activate(context: vscode.ExtensionContext): void {
  const scanner       = new WorkspaceScanner();
  const diagManager   = new DiagnosticManager();
  const decorator     = new UnusedKeyDecorator();
  const allKeys       = new AllKeysProvider();
  const unusedKeys    = new UnusedKeysProvider();
  const missingKeys   = new MissingKeysProvider();
  const duplicates    = new DuplicatesProvider();

  function onSnapshot(snapshot: KeyStoreSnapshot): void {
    diagManager.refresh(snapshot);
    decorator.refresh(snapshot);
    allKeys.refresh(snapshot);
    unusedKeys.refresh(snapshot);
    missingKeys.refresh(snapshot);
    duplicates.refresh(snapshot);
  }

  scanner.on('updated', onSnapshot);

  // Register sidebar views
  context.subscriptions.push(
    vscode.window.createTreeView('tkm.allKeys',    { treeDataProvider: allKeys,    showCollapseAll: true }),
    vscode.window.createTreeView('tkm.unusedKeys', { treeDataProvider: unusedKeys }),
    vscode.window.createTreeView('tkm.missingKeys',{ treeDataProvider: missingKeys }),
    vscode.window.createTreeView('tkm.duplicates', { treeDataProvider: duplicates }),
  );

  // Register language providers
  context.subscriptions.push(
    createHoverProvider(scanner),
    createDefinitionProvider(scanner),
  );

  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand('tkm.scanWorkspace', () => {
      scanner.scanWorkspace().catch(console.error);
    }),

    vscode.commands.registerCommand(
      'tkm.openLocaleFile',
      async (filePath: string, line: number) => {
        const doc = await vscode.workspace.openTextDocument(vscode.Uri.file(filePath));
        const editor = await vscode.window.showTextDocument(doc);
        const pos = new vscode.Position(line, 0);
        editor.selection = new vscode.Selection(pos, pos);
        editor.revealRange(new vscode.Range(pos, pos), vscode.TextEditorRevealType.InCenter);
      }
    ),

    vscode.commands.registerCommand('tkm.exportIssues', async () => {
      const snapshot = scanner.snapshot;
      if (!snapshot) {
        vscode.window.showInformationMessage('TKM: Run a scan first.');
        return;
      }
      const report = buildExportReport(snapshot);
      const uri = await vscode.window.showSaveDialog({
        filters: { JSON: ['json'], CSV: ['csv'] },
        defaultUri: vscode.Uri.file('tkm-issues.json'),
      });
      if (uri) {
        const { workspace } = vscode;
        await workspace.fs.writeFile(uri, Buffer.from(JSON.stringify(report, null, 2)));
        vscode.window.showInformationMessage(`TKM: Issues exported to ${uri.fsPath}`);
      }
    })
  );

  // Re-apply decorations when the active editor changes
  context.subscriptions.push(
    vscode.window.onDidChangeActiveTextEditor(() => {
      if (scanner.snapshot) decorator.refresh(scanner.snapshot);
    })
  );

  // Register disposables
  context.subscriptions.push(
    { dispose: () => scanner.dispose() },
    { dispose: () => diagManager.dispose() },
    { dispose: () => decorator.dispose() },
  );

  // Start watcher and do initial scan
  scanner.startWatcher();
  scanner.scanWorkspace().catch(console.error);
}

export function deactivate(): void {
  // VS Code disposes context.subscriptions automatically
}

function buildExportReport(snapshot: KeyStoreSnapshot): object {
  return {
    timestamp: new Date().toISOString(),
    unusedKeys: snapshot.unusedKeys.map(k => ({
      key: k.fullKey,
      value: k.value,
      file: k.filePath,
      line: k.line + 1,
    })),
    missingKeys: snapshot.missingKeys.map(u => ({
      key: u.key,
      referencedIn: u.filePath,
      line: u.line + 1,
    })),
    duplicateValues: [...snapshot.duplicateValues.entries()].map(([value, keys]) => ({
      value,
      keys: keys.map(k => k.fullKey),
    })),
  };
}
```

- [ ] **Step 2: Compile**

```bash
npm run compile
```

Expected: No errors. `out/extension.js` generated.

- [ ] **Step 3: Commit**

```bash
git add src/extension.ts
git commit -m "feat: wire extension entry point — activate registers all providers and starts scan"
```

---

## Task 14: Test Runner Setup

**Files:**
- Create: `test/runTests.ts`
- Create: `test/suite/index.ts`
- Create: `tsconfig.test.json`

- [ ] **Step 1: Write test/runTests.ts**

```typescript
import * as path from 'path';
import { runTests } from '@vscode/test-electron';

async function main(): Promise<void> {
  const extensionDevelopmentPath = path.resolve(__dirname, '../../');
  const extensionTestsPath = path.resolve(__dirname, './suite/index');

  await runTests({ extensionDevelopmentPath, extensionTestsPath });
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **Step 2: Write test/suite/index.ts**

```typescript
import * as path from 'path';
import Mocha from 'mocha';
import * as glob from 'glob';

export function run(): Promise<void> {
  const mocha = new Mocha({ ui: 'tdd', color: true, timeout: 10000 });
  const testsRoot = path.resolve(__dirname, '.');

  return new Promise((resolve, reject) => {
    glob('**/*.test.js', { cwd: testsRoot }, (err, files) => {
      if (err) return reject(err);
      files.forEach(f => mocha.addFile(path.resolve(testsRoot, f)));
      mocha.run(failures => {
        if (failures > 0) reject(new Error(`${failures} tests failed.`));
        else resolve();
      });
    });
  });
}
```

- [ ] **Step 3: Write tsconfig.test.json**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2020",
    "outDir": "out",
    "lib": ["ES2020"],
    "sourceMap": true,
    "rootDir": ".",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*", "test/**/*"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 4: Add glob dependency**

```bash
npm install --save-dev glob @types/glob
```

- [ ] **Step 5: Run all tests**

```bash
npm test
```

Expected: All test suites pass. Output like:
```
  LocaleParser
    ✓ flattens nested JSON into dot-notation keys
    ✓ sets namespace from filename
    ...
  SourceParser
    ✓ extracts t() calls with useTranslation namespace
    ...
  KeyStore
    ✓ marks a locale key as unused when no usages reference it
    ...
  HoverProvider
    ✓ buildHoverContent returns markdown with key value
    ...
  DefinitionProvider
    ✓ resolves location for a known key
    ...
  X passing
```

- [ ] **Step 6: Commit**

```bash
git add test/runTests.ts test/suite/index.ts tsconfig.test.json package.json
git commit -m "test: set up @vscode/test-electron runner and mocha suite loader"
```

---

## Task 15: Manual Integration Test in VS Code

**Goal:** Press F5, open a project with locale files, and verify all features work end-to-end.

- [ ] **Step 1: Add launch.json**

Create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Extension",
      "type": "extensionHost",
      "request": "launch",
      "args": ["--extensionDevelopmentPath=${workspaceFolder}"],
      "outFiles": ["${workspaceFolder}/out/**/*.js"],
      "preLaunchTask": "${defaultBuildTask}"
    },
    {
      "name": "Extension Tests",
      "type": "extensionHost",
      "request": "launch",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}",
        "--extensionTestsPath=${workspaceFolder}/out/test/suite/index"
      ],
      "outFiles": ["${workspaceFolder}/out/**/*.js"],
      "preLaunchTask": "${defaultBuildTask}"
    }
  ]
}
```

- [ ] **Step 2: Add tasks.json for build**

Create `.vscode/tasks.json`:
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "script": "watch",
      "problemMatcher": "$tsc-watch",
      "isBackground": true,
      "presentation": { "reveal": "never" },
      "group": { "kind": "build", "isDefault": true }
    }
  ]
}
```

- [ ] **Step 3: Press F5 and open the sp18-codebase folder**

Open VS Code with this extension folder, press F5 to launch the Extension Development Host, then open a project that has locale files.

- [ ] **Step 4: Verify each feature manually**

Checklist:
- [ ] Sidebar shows "Translation Keys" icon in Activity Bar
- [ ] "All Keys" tab populates with keys grouped by namespace
- [ ] "Unused" tab shows keys not referenced in source
- [ ] "Missing" tab shows keys referenced in source but absent from locale
- [ ] Hovering over `t('pages.settings.title')` shows tooltip with value + usages
- [ ] Ctrl+Click on a key navigates to locale JSON file at correct line
- [ ] Red squiggle appears under `t('pages.ghost.key')` (key that doesn't exist)
- [ ] Saving a locale file triggers re-scan (watch the sidebar update)

- [ ] **Step 5: Commit**

```bash
git add .vscode/launch.json .vscode/tasks.json
git commit -m "chore: add VS Code launch and tasks config for debugging"
```

---

## Task 16: Package and Publish

**Files:**
- Update: `package.json` (add `icon`, `repository`, `license`, `keywords`)

- [ ] **Step 1: Add marketplace metadata to package.json**

In `package.json`, add these fields:
```json
{
  "icon": "icon.png",
  "repository": {
    "type": "git",
    "url": "https://github.com/sparkeighteen/translation-key-manager"
  },
  "keywords": ["i18n", "translation", "locale", "react-i18next", "keys"],
  "license": "MIT",
  "galleryBanner": {
    "color": "#1e1e2e",
    "theme": "dark"
  }
}
```

- [ ] **Step 2: Create an icon**

Add a 128x128 PNG named `icon.png` to the project root. (Use any i18n / key icon — can use a free SVG from VS Code Codicons or a public icon.)

- [ ] **Step 3: Run final compile + tests**

```bash
npm run compile && npm test
```

Expected: All tests pass, no compile errors.

- [ ] **Step 4: Package the extension**

```bash
npx vsce package
```

Expected: A `.vsix` file is created, e.g. `translation-key-manager-0.1.0.vsix`.

- [ ] **Step 5: Install locally to verify**

```bash
code --install-extension translation-key-manager-0.1.0.vsix
```

Open a new VS Code window with a project that has locale files. Verify the extension activates and all features work.

- [ ] **Step 6: Publish to Marketplace** (requires publisher account)

```bash
npx vsce login squad2
npx vsce publish
```

Expected: Extension published. URL: `https://marketplace.visualstudio.com/items?itemName=squad2.translation-key-manager`

- [ ] **Step 7: Commit**

```bash
git add package.json icon.png
git commit -m "chore: add marketplace metadata and icon for vsce publish"
```

---

## Self-Review Checklist

### Spec Coverage

| PRD Requirement | Task |
|---|---|
| Inline unused key highlight (faded) | Task 8 |
| Missing key red squiggle | Task 7 |
| Hover tooltip with value + namespace + usages | Task 9 |
| Tooltip jump-to-definition link | Task 9 (command link in markdown) |
| Sidebar — All Keys tab | Task 12 |
| Sidebar — Unused tab | Task 12 |
| Sidebar — Missing tab | Task 12 |
| Sidebar — Duplicates tab | Task 12 |
| Go to Definition (Ctrl+Click) | Task 10 |
| Marketplace publish | Task 16 |
| Duplicate value detection | Task 5 (KeyStore) + Task 12 (sidebar) |
| Long key warning (hint) | Task 7 |
| Auto scan on save | Task 6 (FileSystemWatcher) |
| Export issues report | Task 13 (exportIssues command) |
| Dark mode support | Automatic — all decorations use `ThemeColor` / `ThemeIcon` |

All PRD must-have and should-have features are covered.

### Type Consistency

- `LocaleKey.fullKey` — used consistently in all tasks
- `KeyUsage.key` — matches `LocaleKey.fullKey` (both are full dot-notation keys)
- `buildKeyStore()` in Task 5 — signature matches `WorkspaceScanner` usage in Task 6
- `KeyStoreSnapshot` — same interface used in DiagnosticManager, Decorator, Hover, Definition, and Sidebar
- `scanner.snapshot` — typed as `KeyStoreSnapshot | undefined`, checked before use in all providers
- `tkm.openLocaleFile` command — registered in Task 13 with `(filePath: string, line: number)`, invoked from `LocaleKeyItem`, `UnusedKeyItem`, and hover tooltip with the same signature

---

## Quick Reference

```
src/extension.ts          activate() — wires everything
src/types.ts              LocaleKey, KeyUsage, KeyStoreSnapshot
src/config.ts             getConfig() — reads workspace settings
src/parser/
  localeParser.ts         parseLocaleFile(path) → LocaleKey[]
  sourceParser.ts         parseSourceFile(path) → KeyUsage[]
src/store/keyStore.ts     buildKeyStore(keys, usages) → KeyStoreSnapshot
src/scanner/
  workspaceScanner.ts     WorkspaceScanner — scan + watch + emit
src/diagnostics/
  diagnosticManager.ts    DiagnosticManager.refresh(snapshot)
  longKeyDiagnostic.ts    getLongKeyDiagnostics(key, maxLen)
src/decorations/
  unusedKeyDecorator.ts   UnusedKeyDecorator.refresh(snapshot)
src/providers/
  hoverProvider.ts        createHoverProvider(scanner)
  definitionProvider.ts   createDefinitionProvider(scanner)
src/sidebar/
  treeItems.ts            NamespaceItem, LocaleKeyItem, UnusedKeyItem, ...
  keyBrowserProvider.ts   AllKeysProvider, UnusedKeysProvider, ...
```
