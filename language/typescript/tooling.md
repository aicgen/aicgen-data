# TypeScript Tooling And Project Setup

## Runtime And Module System

- Target a current Node.js **LTS** (declare it in `package.json` `engines`) and write **ES modules**: `"type": "module"`.
- Use `NodeNext` module resolution for Node apps and libraries; use `Bundler` only when a bundler resolves imports.
- Use explicit file extensions in relative imports under `NodeNext` (`import { x } from './x.js'`).
- Let the compiler check types and a bundler/runtime emit code; keep `tsc --noEmit` as the type-check gate.

```json
{
  "name": "my-service",
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --check .",
    "test": "vitest run",
    "build": "tsc -p tsconfig.build.json"
  }
}
```

## tsconfig: Strict By Default

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true
  },
  "include": ["src", "test"]
}
```

- `noUncheckedIndexedAccess` makes `array[i]` and index signatures `T | undefined`, which catches real bugs.
- `verbatimModuleSyntax` forces `import type` for type-only imports, keeping emitted imports honest.
- Do not weaken `strict` per file with `// @ts-nocheck`. Use `// @ts-expect-error` with a reason for a single known case; it fails when the error disappears.

## Types That Hold Up

```typescript
// satisfies: check against a type without widening the literal types
const routes = {
  home: '/',
  user: '/users/:id',
} satisfies Record<string, `/${string}`>;

// as const + union for closed sets
const ROLES = ['admin', 'member', 'guest'] as const;
type Role = (typeof ROLES)[number];

// unknown at boundaries, narrow with a validator
function parseRole(value: unknown): Role {
  if (typeof value === 'string' && (ROLES as readonly string[]).includes(value)) {
    return value as Role;
  }
  throw new Error(`invalid role: ${String(value)}`);
}

// Discriminated unions model states; switch is exhaustive
type Result<T> = { ok: true; value: T } | { ok: false; error: Error };
function unwrap<T>(result: Result<T>): T {
  if (!result.ok) throw result.error;
  return result.value;
}

// Explicit resource management (TS 5.2+): cleanup runs on every path
async function readAll(path: string): Promise<string> {
  await using file = await openFile(path); // file[Symbol.asyncDispose]()
  return file.readText();
}
```

- Validate external data (HTTP bodies, env vars, JSON files) at the boundary with a schema library (zod, valibot, arktype) and derive the static type from the schema. Types alone do not validate runtime data.
- Prefer `unknown` over `any`; prefer `readonly` arrays and properties for data that must not change.
- Avoid enums for new code unless you need reverse mapping; use `as const` objects or string unions.

## Lint And Format

- **ESLint** with the flat config (`eslint.config.js`) and `typescript-eslint` type-aware rules for correctness (`no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`).
- **Prettier** (or Biome) owns formatting; do not argue style in review. Run the check in CI.
- Keep config in the repository and run the same commands locally and in CI.

```javascript
// eslint.config.js
import tseslint from 'typescript-eslint';

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  { languageOptions: { parserOptions: { projectService: true } } },
  { rules: { '@typescript-eslint/no-floating-promises': 'error' } },
);
```

## Testing

- **Vitest** (fast, ESM-native) or Jest; co-locate `*.test.ts` or use `test/`.
- Test behaviour through public APIs; fake time with the runner's fake timers; avoid network and real clocks.
- Run `tsc --noEmit` over tests too so test code is type-checked.

## Dependencies

- Commit the lockfile; install with `npm ci` (or the frozen-lockfile equivalent) in CI.
- Keep `dependencies` to what ships; build and test tools go in `devDependencies`.
- Audit regularly and see the supply-chain security guideline for cooldowns and CI hardening.

## Quick Checklist

- [ ] `"type": "module"`, `NodeNext`, `engines` set to a supported LTS
- [ ] `strict` plus `noUncheckedIndexedAccess` and `verbatimModuleSyntax`
- [ ] External input validated with a schema; no `any` at boundaries
- [ ] `typecheck`, `lint`, `format --check` and `test` run in CI
