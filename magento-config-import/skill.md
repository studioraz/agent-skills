---
name: Generate Magento Configuration Values
description: Generate import-ready Magento 2 YAML or JSON configuration values for the semaio/Magento2-ConfigImportExport module, including scopes, environment layering, placeholders, and safe override directives.
---

# Generate Magento Configuration Values

## Purpose

Use this skill to create valid Magento 2 configuration-as-code files for the `semaio/Magento2-ConfigImportExport` module. It generates import-ready YAML or JSON configuration definitions that can be used with `php bin/magento config:data:import`.

This skill is intended for Magento teams that manage configuration outside the admin panel and want repeatable, version-controlled, environment-aware workflows.

## When to use

Use this skill when you need to:

- Generate Magento config values for import
- Create YAML or JSON config files for `core_config_data`
- Prepare `base/` and environment-specific config files
- Model config for `default`, `websites`, or `stores` scopes
- Use placeholders for secrets, theme references, or encrypted values
- Preserve or delete values safely during import
- Organize config into recursive vendor/module/feature directories

## What this skill does

This skill can:

- Generate valid YAML or JSON for Magento config import
- Output flat or hierarchical config structures
- Create values for:
  - `default`
  - `websites`
  - `stores`
- Support numeric scope IDs or scope codes where appropriate
- Recommend whether config belongs in:
  - `base/`
  - `dev/`
  - `staging/`
  - `production/`
  - nested environment paths
- Use supported placeholders:
  - `%env(VARIABLE)%`
  - `%theme(frontend/Vendor/theme)%`
  - `%encrypt(value)%`
- Use control directives:
  - `!!KEEP`
  - `!!DELETE`
- Suggest folder/file naming conventions compatible with the module
- Produce output that is ready to import with Magento CLI

## Input expectations

Provide as much of the following as possible:

- Magento config path, for example:
  - `web/secure/base_url`
  - `general/store_information/name`
  - `design/theme/theme_id`
- Desired value
- Target scope:
  - `default`
  - `websites`
  - `stores`
- Scope ID or code
- Target environment:
  - `base`
  - `dev`
  - `staging`
  - `production`
  - nested environment path such as `dev/team-a`
- Output format:
  - `yaml`
  - `json`
- Whether the value should be:
  - literal
  - environment-driven
  - theme-resolved
  - encrypted
  - preserved
  - deleted

## Output requirements

When generating results, always:

1. Produce valid import-ready YAML or JSON.
2. Use the correct scope nesting format.
3. Keep examples directly usable.
4. Prefer safe secrets handling.
5. Make the folder placement clear when relevant.
6. Match the conventions expected by `semaio/Magento2-ConfigImportExport`.

## Supported value patterns

Use these patterns when appropriate:

- `%env(MY_ENV_VAR)%`
- `%theme(frontend/Vendor/theme)%`
- `%encrypt(mySecretValue)%`
- `!!KEEP`
- `!!DELETE`

## Guidance

### Scope rules

Use these scope keys exactly:

- `default`
- `websites`
- `stores`

Examples:

```yaml
general/store_information/name:
  default:
    0: 'Example Store'
```

```yaml
general/store_information/name:
  websites:
    1: 'Example Website'
```

```yaml
general/store_information/name:
  stores:
    my_store_code: 'Example Store View'
```

### Folder placement

Use:

- `base/` for shared values that apply across environments
- environment folders like `dev/`, `staging/`, `production/` for overrides
- nested environment folders like `dev/team-a/` for layered overrides
- recursive subdirectories when organizing by vendor, feature, or package

Example paths:

- `config/store/base/general.yaml`
- `config/store/production/web.yaml`
- `config/store/dev/vendor/package.yaml`

### Secrets and sensitive values

Prefer:

- `%env(VARIABLE)%` for secrets supplied at runtime or in CI/CD

Avoid hard-coding secrets unless the user explicitly asks for it.

Example:

```yaml
payment/provider/api_key:
  default:
    0: '%env(PAYMENT_PROVIDER_API_KEY)%'
```

### Theme references

Use theme placeholders instead of hard-coded IDs when portability matters.

Example:

```yaml
design/theme/theme_id:
  default:
    0: '%theme(frontend/Vendor/theme)%'
```

### Encrypted values

Use `%encrypt(value)%` when the user explicitly wants Magento to import an encrypted value.

Example:

```yaml
payment/provider/secret_key:
  default:
    0: '%encrypt(mySecretKey)%'
```

### Keep and delete directives

Use `!!KEEP` to preserve an existing value:

```yaml
vendorx/general/api_key:
  default:
    0: '!!KEEP'
```

Use `!!DELETE` to remove a value:

```yaml
vendorx/general/api_key:
  default:
    0: '!!DELETE'
```

## Behavioral rules

- Do not invent Magento config paths for real settings if the correct path is unknown.
- Do not hard-code secrets unless explicitly requested.
- Prefer placeholders for sensitive values.
- Preserve exact Magento scope semantics.
- Prefer `base/` for shared config and environment folders for overrides.
- Keep generated output concise and import-ready.
- If the user asks for an example only, give the smallest correct example.
- If the user asks for a file, provide a complete file content example.

## Example tasks

- Generate a YAML config value for `web/secure/base_url` in production.
- Create a JSON config file for a website-scoped store name.
- Put a payment API key into config using `%env(...)%`.
- Generate config to keep a production-only secret unchanged.
- Generate config to delete a store-scoped setting.
- Create a base config plus a staging override.
- Create recursive config examples grouped by vendor.

## Example outputs

```yaml name=config/store/production/web.yaml
web/secure/base_url:
  default:
    0: 'https://example.com/'
```

```yaml name=config/store/base/payment.yaml
payment/provider/api_key:
  default:
    0: '%env(PAYMENT_PROVIDER_API_KEY)%'
```

```yaml name=config/store/staging/payment.yaml
payment/provider/secret_key:
  default:
    0: '!!KEEP'
```

```json name=config/store/dev/general.json
{
  "general/store_information/name": {
    "stores": {
      "my_store_code": "Example Store View"
    }
  }
}
```

## Recommended response style

- Start with the finished config snippet or file
- Briefly explain scope and placement only if helpful
- Prefer YAML unless the user asks for JSON
- Prefer secure patterns over literal secrets
- Make the result easy to paste into a Magento repository
