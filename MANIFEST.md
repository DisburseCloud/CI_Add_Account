# MANIFEST

## Stack
- **Language:** Zoho Deluge (`.dg`)
- **Platform:** Zoho CRM + CheckIssuing REST API (OAuth2 client_credentials)
- **No build system** — scripts are copy-pasted into Zoho's function editor

## Structure

```
CI_Add_Account/
├── README.md                          # Setup guide, field mapping, troubleshooting
├── MANIFEST.md                        # This file
├── src/
│   └── ci_add_account.dg              # Core Deluge script (148 lines); source of truth for both
│                                      #   standalone.ci_add_account (workflow) and
│                                      #   button.ci_add_account_button (manual button)
└── docs/
    └── plans/
        ├── 2026-03-02-ci-add-account-design.md   # Technical design doc
        └── 2026-03-02-ci-add-account-plan.md     # Implementation task plan
```

## Key Relationships
- `src/ci_add_account.dg` is deployed **twice** in Zoho — once as `standalone.ci_add_account`
  (for workflow automation) and once as `button.ci_add_account_button` (for the manual CRM button).
  Zoho does not allow button functions to call standalone functions, so logic is intentionally duplicated.
- The script reads config from Zoho Org Variables (CI_API_ID, CI_API_Secret, CI_Base_URL, etc.)
  rather than hardcoding values — swap CI_Base_URL to switch between sandbox and production.
- `CI_Last_Check_Number` org variable is mutated at runtime (incremented by 1,000,000 per account).
