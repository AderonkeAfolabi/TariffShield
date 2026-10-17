# eslint-config-tariffshield

Shared ESLint configuration for TariffShield workspaces. `apps/api`, `apps/web`, and
`packages/sdk` all extend this config so TypeScript lint rules stay consistent across
the monorepo. New workspace packages should do the same.

## Usage

This is an [ESLint shareable config](https://eslint.org/docs/latest/extend/shareable-configs)
in the legacy `.eslintrc` format (not flat config). Add it to your workspace's
`.eslintrc.json`:

```json
{
  "extends": ["tariffshield"],
  "ignorePatterns": ["dist/**", "node_modules/**"]
}
```

`"tariffshield"` resolves to this package because its name is `eslint-config-tariffshield` —
ESLint's `extends` shorthand drops the `eslint-config-` prefix. The `extends` array is
applied in order, so frameworks can be layered on top and override these rules:

```json
{
  "extends": ["tariffshield", "next/core-web-vitals"]
}
```

Then add a `lint` script to the workspace's `package.json`, matching the existing apps:

```json
{
  "scripts": {
    "lint": "eslint --max-warnings 0 src"
  }
}
```

and run it with `npm run lint --workspace=<your-workspace>`.

## Peer dependencies

The config relies on these peers, pinned once in the root `package.json` devDependencies —
workspaces extending it do not need to declare them:

| Package                            | Required version |
| ---------------------------------- | ---------------- |
| `eslint`                           | `>= 8`           |
| `@typescript-eslint/parser`        | `>= 7`           |
| `@typescript-eslint/eslint-plugin` | `>= 7`           |

## What's included

- Parser `@typescript-eslint/parser` with the `@typescript-eslint` plugin.
- Extends `eslint:recommended` and `plugin:@typescript-eslint/recommended`.
- `node` + `es2022` environments.
- Project rule tweaks: `@typescript-eslint/no-explicit-any` and
  `@typescript-eslint/no-non-null-assertion` are off, and `@typescript-eslint/no-unused-vars`
  allows `_`-prefixed arguments/variables.

## Adding a new workspace

1. Add an `.eslintrc.json` as shown above.
2. If the workspace is TypeScript, make sure its `tsconfig.json` is listed in the root
   `.eslintrc.js` `parserOptions.project` so type-aware rules can parse it.
3. Add a `lint` script (see above) and, if you want it enforced in CI, add the workspace
   to the matrix in `.github/workflows/lint.yml`.
