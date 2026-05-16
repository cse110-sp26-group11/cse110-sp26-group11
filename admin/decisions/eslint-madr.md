---
parent: Architectural Decisions
nav_order: 9
---
# Use ESLint with JSDoc enforcement for JavaScript linting in CI pipeline

## Context and Problem Statement

Our project is built with plain HTML, CSS, and JavaScript — no framework or build step. We need to enforce consistent code style across all contributors and ensure that functions are annotated with JSDoc type hints (e.g. `@param`, `@returns`, `@type`), so that the codebase is readable and type intent is documented. These checks should run automatically on pull requests and block merges if they fail.

The main questions to answer:
- Which linting tool should enforce JSDoc annotations on all JavaScript functions?
- How do we enforce consistent code style without leaving configuration decisions up to individual developers?
- How do we wire this into GitHub Actions as a required status check?

## Considered Options

* ESLint + eslint-config-standard + eslint-plugin-jsdoc
* StandardJS (standalone, entails node/npm)
* PrettierJS
* TypeScript (strong commitment, entails node/npm)

## Decision Outcome

Chosen option: **ESLint + eslint-config-standard + eslint-plugin-jsdoc**, because it enforces JSDoc annotations through the linter, uses Standard's opinionated style rules to eliminate code style debates, and runs cleanly in GitHub Actions on every pull request without introducing a build step or requiring node just yet.

### Consequences

* Good, because JSDoc annotations (`@param`, `@returns`, `@type`) are enforced as linting errors — PRs cannot merge without them.
* Good, because eslint-config-standard settles all style debates (no semicolons, 2-space indent, single quotes) from day one with no per-rule configuration needed.
* Good, because the setup is a single CI step (`npx eslint .`) with no runtime dependencies — the project remains plain HTML/CSS/JS.
* Good, because VS Code's ESLint extension surfaces errors inline for developers without needing to push first.
* Bad, because all function declarations require a full JSDoc block — small callbacks or trivial helpers still need documentation, which adds noise if rules are not tuned


### Confirmation

* Verify that `eslint.config.mjs` is present in the repo root and references both `eslint-config-standard` and `eslint-plugin-jsdoc`.
* Confirm the GitHub Actions workflow runs `npx eslint .` and is set as a required status check on `main` and `dev` branches.
* Spot-check merged PRs: every JavaScript function should have a JSDoc block with typed `@param` and `@returns` tags.

## Pros and Cons of the Options

### ESLint + eslint-config-standard + eslint-plugin-jsdoc

Run: `npm install eslint eslint-config-standard eslint-plugin-jsdoc` **in CI**.

Example enforced function style:

```js
/**
 * Calculates the total price with tax.
 * @param {number} price - The base price
 * @param {number} taxRate - Tax rate as a decimal
 * @returns {number} The total price including tax
 */
function calculateTotal (price, taxRate) {
  return price * (1 + taxRate)
}
```

* Good, because `eslint-plugin-jsdoc` enforces `@param`, `@returns`, and `@type` tags — missing or malformed annotations are linting errors.
* Good, because `eslint-config-standard` acts as a full style ruleset, removing the need to configure individual formatting rules or use Prettier for JS.
* Good, because ESLint's `--fix` flag auto-corrects style issues (spacing, quotes, etc.) while JSDoc issues require intentional human fixes.
* Good, because the plugin supports flat config and works with ESLint 9's `eslint.config.mjs` format.
* Bad, because requires a non-trivial config file (`eslint.config.mjs`) compared to zero-config alternatives.
* Bad, because strict JSDoc rules on every function can frustrate developers writing simpler helper functions.

### StandardJS (standalone)

Homepage: <https://standardjs.com/>

* Good, because truly zero configuration — install and run, no config files needed.
* Good, because eliminates all style debates with a single opinionated ruleset.
* Bad, because **does not support plugin extensions** in the traditional sense — `eslint-plugin-jsdoc` cannot be added, so JSDoc annotation enforcement is not possible. This is a hard blocker for our requirements.
* Bad, because the lack of configurability means we cannot tune rules to fit team preferences over time.

### TypeScript (migrate codebase)

* Good, because TypeScript provides static type-checking stronger than JSDoc alone — it catches errors at the call site, not just in annotations.
* Good, because IDE integration (autocomplete, refactoring) is significantly richer with TypeScript.
* Bad, because migrating a plain JS project to TypeScript mid-sprint introduces a build step, changes the project's tech stack, and creates a large diff with no feature value.
* Bad, because TypeScript requires all team members to learn TypeScript syntax rather than leveraging existing JavaScript knowledge.
* Bad, because it conflicts with the project constraint of using plain HTML/CSS/JS with no framework or build tooling.
* Bad, nodejs is an additional complication.

## More Information

* ESLint flat config migration guide: <https://eslint.org/docs/latest/use/configure/migration-guide>
* eslint-plugin-jsdoc documentation: <https://github.com/gajus/eslint-plugin-jsdoc>
* eslint-config-standard: <https://github.com/standard/eslint-config-standard>
* Alternative: pairing ESLint with `tsc --noEmit --allowJs --checkJs` validates the *correctness* of JSDoc types (not just their presence) and can be added as a second CI step if stricter type checking is needed in future.
