# Project Milestones & Engineering Guide: `members.return215.xyz`

This document tracks your progress across the 6 milestones.
All data object names and type references link directly to their formal definitions in **[`docs/DESIGN.md`](./DESIGN.md)**.

---

## Milestone 1: Project Initialization & Sample Content

### Conceptual Deep Dive
1. **Native ES Modules (`"type": "module"`)**:
   - In Node ESM, relative imports **must** include the full file extension (`import { validateMember } from './schema.js'`).
   - `__dirname` is not defined globals in ESM; instead, use `import.meta.dirname` (standardized in Node 21.2+) or `new URL('.', import.meta.url)`.
2. **YAML Parsing Pitfalls**:
   - YAML treats unquoted `#` preceded by whitespace as a comment. A hex color like `accent: #e63946` without quotes becomes `accent: null`! Always quote hex strings (`"#e63946"`).
   - YAML sexagesimal/octal quirks: Strings like `"08-19"` or version numbers should be quoted so YAML doesn't coerce them into unexpected numbers or dates.
3. **Co-located Content Bundles**:
   - Storing `index.md`, `avatar.png`, and `sheet.pdf` together inside `content/members/<slug>/` makes each member self-contained. Moving, archiving, or adding a character is a single folder operation rather than hunting across separate data and image trees.

### Task Checklist
- [ ] **1.1**: Create `.gitignore` ignoring `node_modules/`, `dist/`, `.DS_Store`, and `.env*`.
- [ ] **1.2**: Create `package.json` with:
  - `"type": "module"`
  - `"engines": { "node": ">=22.0.0" }`
  - `"scripts"`: `"build": "node src/build.js"`, `"dev": "node src/server.js"`, `"test": "node --test"`
- [ ] **1.3**: Install `gray-matter` and `marked` (`npm install gray-matter marked`).
- [ ] **1.4**: Create sample Member 1 (`content/members/<slug-1>/index.md`) exercising **full** [`MemberFrontmatter`](./DESIGN.md#41-memberfrontmatter-root-input-object) including [`DndBlockInput`](./DESIGN.md#48-dndblockinput) (with both string and [`ReskinObject`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) fields), [`EquipmentBlock`](./DESIGN.md#413-equipmentblock) (`loadouts` and `vehicles`), and placeholder asset files (`avatar.png`, `sheet.pdf`, etc.).
- [ ] **1.5**: Create sample Member 2 (`content/members/<slug-2>/index.md`) **without** `dnd` or `equipment` blocks, plus its placeholder `avatar.png`, to verify optional branch handling.

### Official Documentation Links
- [Node.js ESM (`import.meta.dirname`)](https://nodejs.org/api/esm.html#importmetadirname)
- [Node.js `package.json` `"type"` field](https://nodejs.org/api/packages.html#type)

### Self-Check Verification
```bash
node -e "import('gray-matter').then(() => import('marked')).then(() => console.log('Dependencies OK'))"
ls -la content/members/*
```

---

## Milestone 2: Strict Schema & Asset Validator (`src/schema.js` & `test/schema.test.js`)

### Conceptual Deep Dive
1. **Error Accumulation vs. Fail-on-First-Throw**:
   - If a validator throws an `Error` on the first invalid field, an author fixing a character sheet has to run `npm run build` 5 times to discover 5 typos.
   - Instead, pass an `errors = []` accumulator array through your validation helpers and return a [`ValidationResult`](./DESIGN.md#61-validationresult) (`{ valid: errors.length === 0, errors }`).
2. **Path Traversal Containment (`RelativeAssetPath`)**:
   - Never trust a relative path like `avatar: "../../secret.txt"` by merely concatenating strings.
   - Resolve the path with `const resolved = path.resolve(memberDir, relPath)` and check `const rel = path.relative(memberDir, resolved)`. If `rel.startsWith('..') || path.isAbsolute(rel)`, the path escapes the member's directory and must be rejected before touching the filesystem.
3. **Polymorphic Type Narrowing (`ReskinnableFieldInput`)**:
   - Fields like `dnd.species` and items in `dnd.features` accept either a non-empty `string` or a [`ReskinObject`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) (`{ canon, sheet, note? }`). Writing a dedicated helper `validateReskinnableField(value, fieldName, errors)` keeps your `dnd` validator clean and DRY.

### Module Contract (`src/schema.js`)
Export the following functions:
- `validateSlug(slug)`
  - Validates [`Slug`](./DESIGN.md#31-slug) format (`/^[a-z0-9]+(?:-[a-z0-9]+)*$/`). Returns `boolean`.
- `resolveAndVerifyAsset(memberDir, relPath)` (async)
  - Validates [`RelativeAssetPath`](./DESIGN.md#34-relativeassetpath): checks path confinement inside `memberDir` and verifies the target exists and is a regular file (`(await fs.stat(resolved)).isFile()`).
- `validateMember(slug, rawData, memberDir)` (async)
  - Validates `rawData` against [`MemberFrontmatter`](./DESIGN.md#41-memberfrontmatter-root-input-object) and all nested object definitions ([`ThemeConfig`](./DESIGN.md#43-themeconfig), [`GalleryItemInput`](./DESIGN.md#44-galleryiteminput), [`ExternalLink`](./DESIGN.md#45-externallink), [`LoreBlock`](./DESIGN.md#46-loreblock), [`DndBlockInput`](./DESIGN.md#48-dndblockinput), [`EquipmentBlock`](./DESIGN.md#413-equipmentblock)).
  - Returns `Promise<ValidationResult>` ([`ValidationResult`](./DESIGN.md#61-validationresult)).

### Edge Cases to Handle
- `rawData` is `null`, an array, or not a plain object (`typeof val === 'object' && val !== null && !Array.isArray(val)`).
- `dnd.abilities` is provided, but is missing one of the 6 ability scores (`str`, `dex`, `con`, `int`, `wis`, `cha`) or has a float/out-of-range number (`1..30`).
- `avatar` points to a directory instead of a file, or attempts `../` traversal.
- `links[].url` or `dnd.sheets.ddb` uses `javascript:alert(1)` instead of `http:` / `https:`.

### Task Checklist
- [ ] **2.1**: Implement scalar & helper validators in `src/schema.js` (`Slug`, `MemberStatus`, `HexColor`, `HttpUrl`, `RelativeAssetPath`, `ReskinnableFieldInput`).
- [ ] **2.2**: Implement full `validateMember(slug, rawData, memberDir)` covering core fields, `theme`, `gallery`, `lore`, `dnd`, `equipment`, and `links`.
- [ ] **2.3**: Write unit tests in `test/schema.test.js` using `node:test` (`describe`, `it`) and `node:assert/strict` covering:
  - Valid full member (with D&D and Equipment) passes.
  - Valid minimal member (no D&D, no Equipment) passes.
  - Missing required fields accumulate multiple errors.
  - Invalid `ReskinObject` (missing `canon` or `sheet`) fails.
  - Path traversal (`../`) and missing asset files fail.
  - Invalid URL protocols (`javascript:`) and invalid hex colors fail.

### Official Documentation Links
- [Node.js Test Runner (`node:test`)](https://nodejs.org/api/test.html)
- [Node.js Strict Assertions (`node:assert/strict`)](https://nodejs.org/api/assert.html#strict-assertion-mode)
- [Node.js `fsPromises.stat()`](https://nodejs.org/api/fs.html#fspromisesstatpath-options)
- [Node.js `path.resolve()` & `path.relative()`](https://nodejs.org/api/path.html#pathrelativefrom-to)
- [WHATWG `URL` Constructor](https://nodejs.org/api/url.html#new-urlinput-base)

### Self-Check Verification
```bash
npm test
```

---

## Milestone 3: Content Loader, Normalizer, & Markdown Parser (`src/parser.js`)

### Conceptual Deep Dive
1. **Data Normalization Boundaries**:
   - Why normalize in `src/parser.js`? Because authoring ergonomics (`MemberFrontmatter`) and consumption ergonomics (`MemberDetail`) have opposite goals. Authors want concise YAML with optional fields and shorthand strings; templates and API clients want predictable shapes with zero `undefined` checks.
   - Converting every [`ReskinnableFieldInput`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) into a [`ReskinObject`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) and every [`RelativeAssetPath`](./DESIGN.md#34-relativeassetpath) into a [`PublicAssetUrl`](./DESIGN.md#35-publicasseturl) in one place means `renderer.js` and `build.js` never have to guess types.
2. **Concurrent File I/O (`Promise.all`)**:
   - Reading 20 member folders sequentially with `for (const dir of dirs) { await load(dir); }` serializes disk I/O latency. Mapping folders to promises and awaiting `Promise.all(...)` executes file reads and stat checks concurrently.
3. **Customizing `marked` for Co-located Assets**:
   - If a member's Markdown body includes `![Chassis Diagram](./diagram.png)`, browser requests on `/members/<slug>/` or API consumers reading `bodyHtml` need that `src` rewritten to `/members/<slug>/diagram.png` (while leaving external `https://...` images untouched).

### Module Contract (`src/parser.js`)
Export the following functions:
- `normalizeReskin(input)`
  - Converts [`ReskinnableFieldInput`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) (`string | ReskinObject`) into a strict [`ReskinObject`](./DESIGN.md#42-reskinnablefieldinput--reskinobject) (`{ canon: string, sheet: string | null, note: string | null }`).
- `toPublicAssetUrl(slug, relPath)`
  - Converts a [`RelativeAssetPath`](./DESIGN.md#34-relativeassetpath) (e.g., `"./avatar.png"`) into a [`PublicAssetUrl`](./DESIGN.md#35-publicasseturl) (`"/members/<slug>/avatar.png"`).
- `toMemberSummary(memberDetail)`
  - Projects a full [`MemberDetail`](./DESIGN.md#56-memberdetail) into a lightweight [`MemberSummary`](./DESIGN.md#55-membersummary).
- `loadAllMembers(contentRootDir)` (async)
  - Scans `contentRootDir` (`content/members`), validates each directory with `validateMember`, normalizes all fields into [`MemberDetail`](./DESIGN.md#56-memberdetail), compiles Markdown via `marked`, and returns `Promise<MemberDetail[]>` sorted by `order` ascending, then `name.localeCompare()`.
  - Throws an aggregated `Error` if any member fails validation.

### Task Checklist
- [ ] **3.1**: Implement `normalizeReskin(input)` and `toPublicAssetUrl(slug, relPath)`.
- [ ] **3.2**: Implement Markdown compilation with relative image URL rewriting for `/members/<slug>/...`.
- [ ] **3.3**: Implement `parseMemberDirectory(slug, memberDir)` returning a normalized [`MemberDetail`](./DESIGN.md#56-memberdetail) (including computed `hasDnd` and [`DndSummary`](./DESIGN.md#54-dndsummary)).
- [ ] **3.4**: Implement `loadAllMembers(contentRootDir)` with concurrent loading (`Promise.all`) and deterministic sorting.
- [ ] **3.5**: Add unit tests in `test/parser.test.js` verifying normalization of `ReskinObject`, `DndSummary` formatting, asset URL resolution, and sorting order.

### Official Documentation Links
- [Node.js `fsPromises.readdir()` with `{ withFileTypes: true }`](https://nodejs.org/api/fs.html#fspromisesreaddirpath-options)
- [Node.js `fsPromises.readFile()`](https://nodejs.org/api/fs.html#fspromisesreadfilepath-options)
- [Node.js `path.posix.normalize()`](https://nodejs.org/api/path.html#pathposix)

### Self-Check Verification
```bash
npm test
node -e "import('./src/parser.js').then(m => m.loadAllMembers('./content/members')).then(res => console.log('Loaded members:', res.map(x => x.slug)))"
```

---

## Milestone 4: Template Engine & HTML Security (`src/renderer.js` & `src/templates/`)

### Conceptual Deep Dive
1. **Context-Aware XSS Escaping (`escapeHtml`)**:
   - Even in a static site generator where you author the Markdown files, unescaped frontmatter interpolation causes broken HTML layout (e.g., a vehicle name containing `"Kestrel" <Mk.II>` or an `&` in a URL/summary) and potential XSS if external data is ever imported.
   - Always escape `&`, `<`, `>`, `"`, and `'` on all data strings before injecting them into HTML text or attribute contexts.
2. **Per-Character Theme Scoping with CSS Custom Properties**:
   - By injecting validated [`ThemeConfig`](./DESIGN.md#43-themeconfig) values as inline custom properties on the `<body>` or `<main>` element (`style="--member-accent: #e63946; --member-bg: #121316;"`), your `public/css/style.css` can define fallback defaults (`--member-accent: #6366f1`) and style borders, badges, and stat bars dynamically without generating separate CSS files per member.
3. **Progressive Enhancement**:
   - Pre-render all member cards inside `dist/index.html` at build time with `data-name`, `data-role`, `data-group`, `data-tags`, `data-order`, and `data-has-dnd` attributes. The directory works immediately before JS loads (and for search engines/curl), and `public/js/directory.js` can filter/sort the DOM or hydrate from `/api/members.json`.

### Module Contract (`src/renderer.js`)
Export the following functions:
- `escapeHtml(str)`
  - Converts `&`, `<`, `>`, `"`, `'` to their HTML entity equivalents.
- `formatAbilityModifier(score)`
  - Computes D&D ability modifier `Math.floor((score - 10) / 2)` and formats with sign (`"+4"`, `"+0"`, `"-1"`).
- `renderIndexPage(members)` (async)
  - Accepts [`MemberSummary[]`](./DESIGN.md#55-membersummary) (or [`MemberDetail[]`](./DESIGN.md#56-memberdetail)), injects rendered directory cards and unique tag filter buttons into `src/templates/index.html` and wraps with `src/templates/layout.html`. Returns `Promise<string>`.
- `renderMemberPage(member)` (async)
  - Accepts a [`MemberDetail`](./DESIGN.md#56-memberdetail), renders the profile infobox (`LoreBlock`), conditional D&D tactical card (`DndBlockOutput` with Canon-vs-Sheet reskin badges and PDF/DDB/Dicecloud sheet buttons), conditional Equipment cards (`LoadoutEntry[]` and `VehicleEntry[]`), gallery, and `bodyHtml` into `src/templates/member.html` wrapped with `src/templates/layout.html`. Returns `Promise<string>`.

### Task Checklist
- [ ] **4.1**: Implement `escapeHtml(str)` and `formatAbilityModifier(score)` in `src/renderer.js`.
- [ ] **4.2**: Create HTML templates in `src/templates/` (`layout.html`, `index.html`, `member.html`) and global styles in `public/css/style.css`.
- [ ] **4.3**: Implement `renderIndexPage(members)` and `renderMemberPage(member)` in `src/renderer.js`.
- [ ] **4.4**: Add unit tests in `test/renderer.test.js` testing `escapeHtml`, `formatAbilityModifier`, Canon-vs-Sheet rendering, and verifying that optional blocks (`dnd: null`, `equipment: null`) omit empty sections cleanly.

### Self-Check Verification
```bash
npm test
```

---

## Milestone 5: The Build Orchestrator & Static JSON API (`src/build.js`)

### Conceptual Deep Dive
1. **Clean, Idempotent Builds**:
   - If you rename or delete a member in `content/members/`, stale files will linger in `dist/` unless `dist/` is cleaned first. Use `await fs.rm(distDir, { recursive: true, force: true })` after validation succeeds (or clean and recreate `dist/` cleanly).
2. **Selective Co-located Asset Copying**:
   - Each `content/members/<slug>/` folder contains `index.md` alongside binary assets (`avatar.png`, `sheet.pdf`, etc.). When copying member folders to `dist/members/<slug>/`, filter out `index.md` so raw source Markdown isn't unnecessarily duplicated over the generated `index.html`.
3. **API Payload Projection**:
   - Write `dist/api/members.json` using `members.map(toMemberSummary)` ([`MemberSummary[]`](./DESIGN.md#55-membersummary)) and `dist/api/members/<slug>.json` using the full [`MemberDetail`](./DESIGN.md#56-memberdetail).

### Module Contract (`src/build.js`)
- Export `buildSite(options)` (async) so both the CLI (`npm run build`) and the dev server (`src/server.js`) can invoke it programmatically.
- When `src/build.js` is executed directly (`import.meta.url === ...` or `process.argv[1]`), run `buildSite()`, log build duration using `performance.now()`, and set `process.exitCode = 1` on error.

### Task Checklist
- [ ] **5.1**: Implement `buildSite()` in `src/build.js` to:
  1. Load & validate all members via `loadAllMembers()`.
  2. Clean and recreate `dist/`, `dist/api/members/`, and `dist/members/`.
  3. Write `dist/api/members.json` and `dist/api/members/<slug>.json`.
  4. Render and write `dist/index.html` and `dist/members/<slug>/index.html`.
  5. Copy `public/` to `dist/` and copy co-located member assets (excluding `index.md`) to `dist/members/<slug>/`.
- [ ] **5.2**: Run `npm run build` and inspect the generated `dist/` tree and JSON files.

### Official Documentation Links
- [Node.js `fsPromises.rm()`](https://nodejs.org/api/fs.html#fspromisesrmpath-options)
- [Node.js `fsPromises.mkdir()`](https://nodejs.org/api/fs.html#fspromisesmkdirpath-options)
- [Node.js `fsPromises.writeFile()`](https://nodejs.org/api/fs.html#fspromiseswritefilefile-data-options)
- [Node.js `fsPromises.cp()`](https://nodejs.org/api/fs.html#fspromisescpsrc-dest-options)
- [Node.js `performance.now()` (`node:perf_hooks`)](https://nodejs.org/api/perf_hooks.html#performancenow)

### Self-Check Verification
```bash
npm run build
ls -R dist/
cat dist/api/members.json
```

---

## Milestone 6: Local Dev Server & Client-Side Interactivity (`src/server.js` & `public/js/directory.js`)

### Conceptual Deep Dive
1. **Streaming Static Files & Backpressure (`fs.createReadStream`)**:
   - Reading large binary files (like `.pdf` character sheets, audio files, or high-res gallery images) into memory with `await fs.readFile()` buffers the entire file in RAM for every concurrent request.
   - Piping `fs.createReadStream(filePath).pipe(res)` streams chunks directly from the OS file descriptor to the TCP socket, respecting TCP backpressure automatically.
2. **URL Routing, Directory Index Resolution, & Path Confinement**:
   - Strip query strings (`new URL(req.url, 'http://localhost').pathname`), decode URI components safely, resolve against `dist/`, verify the resolved path starts with `dist/`, and automatically map directory requests (like `/members/vaelen-vance/`) to `index.html`.
3. **CORS for External API Consumers**:
   - External browser apps fetching `/api/members.json` will be blocked by the browser's Same-Origin Policy unless responses under `/api/` include `Access-Control-Allow-Origin: *`.
4. **Debouncing `fs.watch`**:
   - Saving a file in most editors triggers 2–4 rapid OS filesystem events (write temp file, rename, chmod). Wrap the rebuild callback in a ~150ms `setTimeout` debounce and a `building` lock flag so builds never overlap.

### Task Checklist
- [ ] **6.1**: Implement `src/server.js`:
  - Run an initial `await buildSite()` on startup.
  - Create an HTTP server (`http.createServer`) serving `dist/` with a complete MIME type map (`.html`, `.css`, `.js`, `.json`, `.png`, `.jpg`, `.webp`, `.svg`, `.ogg`, `.mp3`, `.pdf`).
  - Add `Access-Control-Allow-Origin: *` for `/api/*` routes.
  - Watch `content/`, `src/`, and `public/` using `fs.watch(..., { recursive: true })` with a debounce timer to rebuild automatically on file changes.
- [ ] **6.2**: Implement `public/js/directory.js`:
  - Wire up live text search (matching `name`, `role`, `summary`, `group`, `tags`), tag pill filtering, a "D&D Characters Only" filter toggle, and sorting (`order` vs. alphabetical `name`).
  - Sync active filters with URL query parameters (`window.history.replaceState`) so filtered views are shareable.

### Official Documentation Links
- [Node.js `http.createServer()`](https://nodejs.org/api/http.html#httpcreateserveroptions-requestlistener)
- [Node.js `fs.createReadStream()`](https://nodejs.org/api/fs.html#fscreatereadstreampath-options)
- [Node.js `fs.watch()`](https://nodejs.org/api/fs.html#fswatchfilename-options-listener)

### Self-Check Verification
```bash
npm run dev
# In another terminal:
curl -I http://localhost:3000/api/members.json
```
