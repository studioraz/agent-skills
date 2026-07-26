# Skill: Create Renovate configuration for a Magento repository

You create a project-level `renovate.json` for Magento repositories.

## Goal

Generate a working, minimal, project-level Renovate config for a Magento Composer project by analyzing:

- `composer.json`
- `auth.json` if present
- an existing working Renovate pattern from similar Magento projects

The generated file must follow the Studio Raz Magento Renovate conventions below.

---

## Required behavior

### 1) Base structure

Generate a `renovate.json` with this base structure:

- `$schema` set to Renovate schema
- `extends` containing:
  - `config:base`
  - `:timezone(Asia/Jerusalem)`
  - `:prHourlyLimitNone`
  - `:prConcurrentLimitNone`
- `"dependencyDashboard": true`
- `"schedule": ["before 3am"]`
- `"includePaths": ["composer.json"]`
- `"composer": { "enabled": true }`

Use Renovate config keys exactly as documented. `hostRules` is the correct place for credentials, `packageRules` is the correct place for package-level behavior, and `groupName` groups matched updates[...]

---

## 2) Inputs to inspect

### composer.json
Read and analyze:

- `require`
- `require-dev`
- `repositories`
- `config.platform.php`

Use this to determine:

- which private Composer registries are project-specific
- which vendor packages exist
- which Hyvä compat modules exist and which base modules they correspond to
- whether Amasty packages exist
- whether Magento or Studio Raz package sources are referenced
- whether `config.platform.php` exists skip it

### composer.json PHP platform normalization (required)
Always ensure `composer.json` contains `config.platform.php` with value `8.3.0`.

- If `config.platform.php` is missing, add it while preserving the rest of the existing `composer.json` structure:

  ```json
  "config": {
    "platform": {
      "php": "8.3.0"
    }
  }
  ```

- If `config.platform.php` exists with a different value, skip it.
- This normalization is required whenever generating or updating the Magento Renovate-related project setup.

### auth.json
If `auth.json` exists, inspect the `http-basic` entries.

Use it to derive Renovate `hostRules` for the registries that should be configured at project level.

Do not transform or normalize credentials values. Copy them exactly.

---

## 3) Organization-level exceptions

Do **not** add project-level `hostRules` for these, because they are configured at Renovate organization level:

- Magento credentials / `repo.magento.com`
- Studio Raz Packagist credentials / `repo.packagist.com` / Studio Raz repository credentials

If these are present in `auth.json` or `composer.json`, recognize them but omit them from the generated project config.

---

## 4) Project-level hostRules

Add `hostRules` for registries that are project-specific and required for dependency lookup.

Typical examples may include:

- Hyvä private Packagist
- vendor-specific private Composer repositories not managed at org level
- Amasty private Composer repository credentials when present in `auth.json`

When generating `hostRules`:

- use `matchHost`
- use `hostType: "packagist"` for Composer/Packagist-style hosts
- copy credentials from `auth.json` as-is
- prefer the most specific host or URL needed for matching
- if a matching repository URL exists in `composer.json`, prefer using the repository URL as `matchHost`
- otherwise use the host from `auth.json`

Renovate supports credentials through `hostRules`, and `matchHost` can be either a hostname or a full base URL. `hostRules` is specifically intended for authenticated hosts.

### Credential copy rule
If a host should be included in project-level `hostRules`, copy the `username` and `password` from `auth.json` exactly as they appear.

Do not replace them with placeholders.  
Do not redact them.  
Do not rename them.  
Do not modify their casing or formatting.

### Mirasvit exception
If a Mirasvit private repository URL already contains embedded credentials directly in the repository URL, do **not** add a `hostRules` entry for Mirasvit.

Treat embedded credentials in the repository URL as sufficient for Renovate authentication in this case.

### Amasty credentials exception
Even though all `amasty/*` package updates must be disabled, if Amasty private repository credentials exist in `auth.json`, copy them into project-level `hostRules` exactly as-is.

Disabling Amasty updates does **not** change the requirement to include Amasty private credentials from `auth.json` when present.

---

## 5) Always skip these dependency categories

Add `packageRules` to disable:

### Magento platform/core upgrades
Disable updates for Magento platform/core packages that should not be upgraded automatically in project PRs, including when present:

- `magento/product-community-edition`
- `magento/magento2-base`
- `magento/framework`

Purpose: prevent platform upgrade PRs.

### require-dev
Disable all `require-dev` dependencies using `matchDepTypes`.

### Amasty
Disable all `amasty/*` packages.

Reason: Renovate cannot access the Amasty repository for this workflow, so those packages should be skipped.

Use `enabled: false` inside matching `packageRules`, which Renovate supports for disabling matched dependencies.

### Amasty grouping rule behavior
Even though all `amasty/*` package updates are disabled, still detect Amasty base + Hyvä compatibility package pairs and generate grouping rules for confirmed pairs using the same base/Hyvä matching logic.

Do not skip the grouping step just because the matched packages belong to `amasty/*`.

---

## 6) Tightly-coupled Mirasvit package groups

Some Mirasvit packages have hard Composer version constraints on each other and **must be updated in the same PR**. Renovate's partial updates break Composer's lock-file resolution for these packages.

### Known tightly-coupled group

The following packages form one tightly-coupled set and must always be grouped together when any of them are present in `composer.json`:

- `mirasvit/module-tm`
- `mirasvit/module-fb-marketing`
- `mirasvit/module-fb-marketing-hyva`
- `mirasvit/module-gtm`
- `mirasvit/module-gtm-hyva`

### Rule

When **two or more** packages from this set exist in `composer.json`:

1. Collect all present members of the set.
2. Generate a single `packageRules` entry with:
   - `matchPackageNames`: list of all present members
   - `groupName`: `"Mirasvit TM / FB Marketing / GTM"`
   - `description`: `"Group interdependent Mirasvit TM/FB Marketing/GTM packages together"`
3. **Do not** generate separate, narrower Hyvä compat grouping rules (see section 6b) for any package that is already covered by this broader group. This avoids duplicate or conflicting `groupName` values for the same packages.

If only one member of the set is present, no group rule is needed (a single-package group has no effect).

---

## 6b) Hyvä compat grouping rules

Detect all Hyvä compatibility modules and group each compat module with its corresponding base module.

### Grouping rule
A Hyvä compatibility package may match its base package using either of these patterns:

#### Pattern A: suffix-based match
If a package ends with `-hyva`, find its matching base package by removing the `-hyva` suffix.

Examples:

- `mirasvit/module-seo-hyva` → base `mirasvit/module-seo`
- `mirasvit/module-blog-mx-hyva` → base `mirasvit/module-blog-mx`
- `vendor/package-hyva` → base `vendor/package`

#### Pattern B: explicit vendor mapping for Hyvä packages
Some Hyvä compat packages use a different package naming convention and must be matched by known mapping rules.

Support explicit mappings such as:

- `hyva-themes/magento2-smile-elasticsuite` → base `smile/elasticsuite`
- `hyva-themes/magento2-mageworx-giftcards` → base `mageworx/module-giftcard`

Use these explicit mappings when suffix-based matching does not apply.

### Matching algorithm
Determine base/compat pairs in this order:

1. Find packages ending with `-hyva` and map them by removing the `-hyva` suffix.
2. Apply known explicit Hyvä-to-base mappings for packages that do not follow the `-hyva` suffix convention.
3. Only create a grouping rule if **both** the compat package and the base package exist in `composer.json`.

This grouping step applies even when the base package belongs to `amasty/*` and Amasty updates are disabled.

### Generated package rule
If **both** base and Hyvä compat module exist in `composer.json`, generate a `packageRules` entry with:

- `matchPackageNames`: `[base, compat]`
- `groupName`: a human-friendly shared group title
- optional `description`: explain that base and Hyvä compat are grouped together

### Group name rules
Prefer readable names. Examples:

- `Mirasvit SEO Modules`
- `Mirasvit Blog MX Modules`
- `Smile ElasticSuite Modules`
- `MageWorx Gift Card Modules`

If no nice product label can be inferred, use a safe fallback like:

- `<base package name> + Hyva`

Grouping related packages via `groupName` is a supported Renovate pattern.

### Examples

- `mirasvit/module-seo` + `mirasvit/module-seo-hyva`
- `smile/elasticsuite` + `hyva-themes/magento2-smile-elasticsuite`
- `mageworx/module-giftcard` + `hyva-themes/magento2-mageworx-giftcards`

### Do not create a grouping rule when:
- the Hyvä package exists without its base package
- the base package exists without a compat package
- the pairing is ambiguous and no explicit mapping rule exists
- both packages in the pair are already covered by the tightly-coupled Mirasvit group (section 6)

---

## 7) Pattern extraction guidance

Use the existing working Magento Renovate file as a pattern reference, but improve it where needed:

- keep the config minimal
- avoid unnecessary hostRules
- do not include org-level credentials at project level
- preserve the convention of using `packageRules` for skips and grouping
- follow consistent descriptions
- ensure generated JSON is valid and cleanly formatted

---

## 8) Output requirements

Return only the final `renovate.json` content unless the user asks for explanation.

The output must be:

- valid JSON
- formatted with 2-space indentation
- project-ready
- minimal, but complete enough to work

If credentials are available in `auth.json` and the host should be included, copy them into `renovate.json` exactly as-is.

---

## 9) Decision rules

### Include a hostRule when:
- the repository is private
- it is project-specific
- Renovate needs credentials to resolve packages
- it is not already managed at organization level
- it is an Amasty private registry with credentials present in `auth.json`, even though `amasty/*` updates are disabled

### Do not include a hostRule when:
- it is Magento org-level config
- it is Studio Raz org-level config
- the registry is unused by dependencies in `composer.json`
- it is a Mirasvit private repository whose repository URL already embeds the credentials

### Create a grouping rule when:
- a Hyvä compat package exists
- the corresponding base package also exists
- the pairing is confirmed either by suffix rule or explicit mapping

### Do not create a grouping rule when:
- the Hyvä package exists without its base package
- the base package exists without a compat package
- the pairing is ambiguous

---

## 10) Recommended packageRules template

Use patterns like:

- disable Magento/core platform packages
- disable `require-dev`
- disable `amasty/*`
- add one grouping rule per discovered base/Hyvä pair

---

## 11) Example generation workflow

1. Read `composer.json`
2. Extract all packages from `require`
3. **Check for tightly-coupled Mirasvit packages** (section 6): if two or more of `mirasvit/module-tm`, `mirasvit/module-fb-marketing`, `mirasvit/module-fb-marketing-hyva`, `mirasvit/module-gtm`, `mirasvit/module-gtm-hyva` are present, generate the single broader group rule and note which packages are now covered
4. Find all package names ending in `-hyva`
5. For each `-hyva` package, derive the base package by removing `-hyva`
6. Find explicit Hyvä mapping matches for packages like `hyva-themes/magento2-smile-elasticsuite` and `hyva-themes/magento2-mageworx-giftcards`
7. If the mapped base package exists, generate a grouping rule — **skip if both packages are already in the tightly-coupled Mirasvit group**
8. Inspect `repositories` in `composer.json`
9. Inspect `http-basic` in `auth.json`
10. Add only project-specific `hostRules`, including Amasty credentials when present in `auth.json`
11. Omit Magento and Studio Raz org-level credentials
12. Omit Mirasvit `hostRules` when credentials are already embedded directly in the repository URL
13. Normalize `composer.json` `config.platform.php` to `8.3.0` (add it if missing, update it if different, preserve existing `composer.json` structure)
14. Add ignore rules for:
   - Magento/core packages
   - `require-dev`
   - `amasty/*`
15. Still generate confirmed base + Hyvä grouping rules even for Amasty package pairs
16. Output final valid `renovate.json`

---

## 12) Default output template

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    ":timezone(Asia/Jerusalem)",
    ":prHourlyLimitNone",
    ":prConcurrentLimitNone"
  ],
  "dependencyDashboard": true,
  "schedule": [
    "before 3am"
  ],
  "includePaths": [
    "composer.json"
  ],
  "composer": {
    "enabled": true
  },
  "hostRules": [],
  "packageRules": []
}
```

Populate `hostRules` and `packageRules` based on the rules above.

---

## 13) Notes for Magento projects

- Magento Composer repos often require auth, but org-level Renovate config may already cover common hosts.
- Private vendor repos should only be included when needed.
- Composer repositories in `composer.json` do not automatically mean Renovate needs project-level credentials; only include those that are both private and project-specific.
- For Mirasvit private repositories, if credentials are already embedded directly in the repository URL, do not add a project-level `hostRules` entry.
- For Amasty private repositories, if credentials are present in `auth.json`, copy them into project-level `hostRules` even though `amasty/*` package updates are disabled.
- Amasty base + Hyvä compat module grouping should still be generated when both packages are present.
- For Magento repos with many vendor modules, grouping base + Hyvä compat packages reduces noisy PRs.

---

## 14) Validation checklist

Before returning the file, verify:

- JSON is valid
- no duplicated package rules
- Magento org-level hostRule is omitted
- Studio Raz org-level hostRule is omitted
- included project-level credentials were copied exactly as-is from `auth.json`
- Mirasvit `hostRules` are omitted when credentials are embedded directly in repository URLs
- Amasty credentials from `auth.json` are included in project-level `hostRules` when present
- Amasty is disabled
- `require-dev` is disabled
- `composer.json` has `config.platform.php` set to `8.3.0`
- existing `composer.json` structure is preserved when normalizing `config.platform.php`
- all discovered base/Hyvä pairs are grouped
- explicit mappings for known Hyvä packages were applied
- Amasty base + Hyvä pairs are still grouped when both packages exist
- tightly-coupled Mirasvit packages (TM / FB Marketing / GTM) are grouped together when two or more are present
- no separate narrower Hyvä compat rules exist for packages already covered by the broader Mirasvit TM/FB Marketing/GTM group
- config is limited to `composer.json`
- Composer manager is enabled

```