# Build Tooling: npm, Webpack, Babel, Vite

> Package management, module systems, bundlers, and transpilers — how modern JS gets from source to shippable bundles.

## npm / Package Management

- **`package.json`** — manifest: dependencies, `devDependencies`, `peerDependencies`, `optionalDependencies`, scripts, engines.
- **Semantic Versioning (SemVer): `MAJOR.MINOR.PATCH`**
  - `^1.2.3` → `>=1.2.3 <2.0.0` (allows minor + patch)
  - `~1.2.3` → `>=1.2.3 <1.3.0` (allows patch only)
  - exact `1.2.3` → pinned
- **Lockfile** (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`) — pins the *entire resolved dependency graph* for reproducible installs. Always commit it. `npm ci` installs strictly from the lockfile (used in CI).
- **`node_modules` resolution** — historically nested; npm/yarn **hoist** shared deps to the top (flat). Risk: "phantom dependencies" — code imports a package it never declared but works because it was hoisted.
- **pnpm** — uses a global content-addressable store + symlinks, giving a strict non-flat `node_modules`. Faster, disk-efficient, and prevents phantom deps.
- **`peerDependencies`** — "I need this, but the host app provides it" (e.g., a component library declares React as a peer to avoid bundling two Reacts).
- **Supply-chain security** — `npm audit`, lockfile integrity hashes, `npm ci`, pinning, provenance, Dependabot/Renovate.

## Module Systems
- **CommonJS (CJS)** — `require` / `module.exports`, synchronous, Node's legacy default. Not statically analyzable → poor tree-shaking.
- **ES Modules (ESM)** — `import` / `export`, static structure → **enables tree-shaking**, supports async loading in browsers. Modern default.
- Interop issues (default vs named exports, `__esModule` interop) are a common source of build errors.

## Webpack

A **module bundler**: builds a dependency graph from entry points and emits bundles.

### Core concepts
- **Entry** — starting module(s).
- **Output** — where/how bundles are written (`[contenthash]` for cache busting).
- **Loaders** — transform non-JS files into modules (`babel-loader` for JSX/TS, `css-loader`, `style-loader`, `asset/resource` for images). Loaders run **right-to-left / bottom-to-top**.
- **Plugins** — hook into the whole build lifecycle for broader tasks (`HtmlWebpackPlugin`, `MiniCssExtractPlugin`, `DefinePlugin`, `TerserPlugin`).
- **Mode** — `development` (fast, readable) vs `production` (minified, optimized, tree-shaken).

### Optimization features
- **Code splitting** — `SplitChunksPlugin` extracts shared/vendor chunks; **dynamic `import()`** creates lazy-loaded chunks (pairs with `React.lazy`).
- **Tree shaking** — dead-code elimination for ESM. Requires ESM, `"sideEffects": false` in `package.json` (or a precise list), and production mode.
- **Caching** — `[contenthash]` in filenames + a stable module/chunk id strategy so unchanged chunks keep their hash → long-term browser caching.
- **HMR (Hot Module Replacement)** — swap modules at runtime in dev without a full reload.
- **Module Federation** — share code/deps at runtime across independently deployed apps (micro-frontends).

### Build performance
- `cache: { type: 'filesystem' }`, `thread-loader`, narrowing `include`/`exclude`, `resolve` tuning, and using esbuild/swc as the transpiler.

## Babel / SWC / esbuild
- **Babel** — JS/TS/JSX transpiler. Pipeline: parse → AST → transform (via presets/plugins) → generate. `@babel/preset-env` targets browsers via `browserslist`; `@babel/preset-react`, `@babel/preset-typescript`.
- **Polyfills** — Babel transforms *syntax*; runtime features (Promise, fetch) need polyfills (core-js) injected per `browserslist`.
- **SWC (Rust)** & **esbuild (Go)** — much faster transpilers/minifiers, now default in many toolchains (Next.js uses SWC; Vite uses esbuild for dev).

## Vite (modern default)
- **Dev:** serves source over **native ESM** — no bundling in dev, so cold start is near-instant; transforms modules on demand. Uses esbuild for dependency pre-bundling.
- **Prod:** bundles with **Rollup** for optimized output.
- Contrast with Webpack: Webpack bundles the whole app for the dev server (slower cold start on large apps) but is extremely mature/configurable.

### Cheat sheet
| Concept | Key point |
|---|---|
| Loader | Per-file transform (TS→JS, inline CSS) |
| Plugin | Whole-lifecycle task (emit HTML, extract CSS, minify) |
| Tree-shaking | Needs ESM + sideEffects + prod mode |
| Code splitting | Route/component `import()` → lazy chunks |
| `[contenthash]` | Cache-busting + long-term caching |
| `npm ci` | Strict, reproducible install from lockfile (CI) |
| Vite dev | Native ESM, no bundling → fast cold start |

---

### Interview Questions — Build Tooling

**Q1. Loader vs plugin in Webpack?**
> Loaders transform individual files/modules (per-file, e.g., transpile TS, inline CSS). Plugins tap into the entire compilation lifecycle to do broader work (emit HTML, extract CSS, define globals, minify).

**Q2. How does tree-shaking work and what breaks it?**
> Static analysis of ESM `import`/`export` removes unused exports in production. It breaks with CommonJS, dynamic/`require`-style imports, modules with unlisted side effects (fix via `"sideEffects"` field), and re-export barrels that pull in everything.

**Q3. Why commit the lockfile, and `npm install` vs `npm ci`?**
> The lockfile pins the exact resolved graph for reproducible, secure installs. `npm install` can update the lockfile and resolve new versions within ranges; `npm ci` installs strictly and fails if `package.json` and the lockfile disagree — deterministic, ideal for CI.

**Q4. `^` vs `~` in SemVer?**
> `^1.2.3` allows minor + patch updates (`<2.0.0`); `~1.2.3` allows patch only (`<1.3.0`).

**Q5. Why is Vite's dev server faster than Webpack's?**
> Vite serves unbundled native ESM in dev and transforms modules on demand, so startup doesn't scale with app size. Webpack bundles up front. In prod both produce optimized bundles (Vite via Rollup).

**Q6. What are phantom dependencies and how does pnpm prevent them?**
> Phantom deps are packages you import without declaring, working only because npm/yarn hoisted them flat. pnpm uses symlinks + a strict, non-flat `node_modules`, so only declared deps are importable.

**Q7. How do you achieve cache-busting and long-term caching together?**
> Use `[contenthash]` filenames so a file's URL changes only when its content changes, split stable vendor chunks separately, and serve with `Cache-Control: max-age=31536000, immutable`. Unchanged chunks stay cached across deploys.
