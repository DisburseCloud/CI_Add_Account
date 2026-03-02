# CI_Add_Account Integration Design

## Context

DisburseCloud needs to automate bank account provisioning in CheckIssuing when opportunities close in Zoho CRM. Currently this is a manual process. This integration will call CheckIssuing's Add Bank Account API from a Deluge script, then immediately retrieve the funding source ID and logo ID via List Bank Accounts, and store them back on the Opportunity record.

## Architecture

**Single Deluge script** with two triggers:
1. **Workflow rule** — fires when Opportunity stage = "Closed Won" or "Payment Setup"
2. **Custom button** — manual trigger on the Opportunity record for ad-hoc use

Both triggers call the same function.

### Flow

```
Opportunity triggers script
  → Read deal-specific fields (Opportunity Name, Parent Account address)
  → Read org variables (API creds, defaults, last check number)
  → OAuth2 client_credentials → get Bearer token
  → POST Add Bank Account to CheckIssuing
  → POST List Bank Accounts to CheckIssuing
  → Match new account by name, extract id + logo
  → Update Opportunity with CI_Funding_Source_ID + CI_Logo_ID
  → Increment CI_Last_Check_Number org variable by 1,000,000
```

No Plaid verification. No callback/webhook. Everything happens in one synchronous script.

## Field Mapping

### Deal-Specific (from CRM)

| CheckIssuing Field | Source | Value |
|---|---|---|
| `name` | Opportunity.Deal_Name | (varies per deal) |
| `name_on_checks` | Opportunity.Deal_Name | (varies per deal) |
| `address_on_checks` | Opportunity.Account_Name → Account.Address | (varies per deal) |
| `acct_num` | Opportunity.Bank_Number | (varies per deal) |

### Shared Defaults (Zoho Org Variables)

| CheckIssuing Field | Org Variable Name | Default Value |
|---|---|---|
| `bank_name` | `CI_Bank_Name` | Florida Capital Bank |
| `acct_type` | — | CHECKING (hardcoded) |
| `bank_addr` | `CI_Bank_Address` | 10151 Deerwood Park Blvd, Jacksonville, FL 32256 |
| `fractional_routing` | `CI_Fractional_Routing` | 063112142 |
| `return_mailto` | `CI_Return_MailTo` | DisburseCloud |
| `return_address` | `CI_Return_Address` | 30212 Tomas ste 355, Rancho Santa Margarita, CA 92688, USA |
| `is_business` | — | true (hardcoded) |
| `void_text` | `CI_Void_Text` | VOID 180 DAYS AFTER ISSUE DATE |
| `sig` | `CI_Default_Signature` | Base64 encoded signature image |
| `starting_check_num` | `CI_Last_Check_Number` | Incremented by 1,000,000 each call |

### API Credentials (Zoho Org Variables)

| Variable | Purpose |
|---|---|
| `CI_API_ID` | CheckIssuing API ID (Basic Auth username) |
| `CI_API_Secret` | CheckIssuing API Secret (Basic Auth password) |

### CRM Fields to Create on Opportunity

| Field Name | Type | Purpose |
|---|---|---|
| `CI_Funding_Source_ID` | Single Line | Stores the account `id` from CheckIssuing |
| `CI_Logo_ID` | Single Line | Stores the `logo` value from CheckIssuing |
| `Bank_Number` | Single Line | Bank/account number for the funding source |

## Authentication

1. POST to CheckIssuing OAuth endpoint with Basic Auth (API_ID:API_Secret)
2. Body: `grant_type=client_credentials`
3. Response: `{ "token_type": "Bearer", "expires_in": 3600, "access_token": "..." }`
4. Use `Authorization: Bearer <token>` on subsequent requests

**Endpoints:**
- Sandbox: `https://websb.checkissuing.com/`
- Production: `https://web.checkissuing.com/`

## API Calls

### 1. Add Bank Account
- Method: POST
- Content-Type: `application/x-www-form-urlencoded`
- All fields sent as form data
- Response: `{ "status": 1, "errors": [], "onboard_uri": "..." }`

### 2. List Bank Accounts
- Method: POST
- No parameters
- Response: `{ "status": 1, "accounts": { "<id>": { "id", "name", "logo", ... } } }`
- Match by `name` = Opportunity Name to find the newly created account

## Error Handling

- If OAuth fails → log error, update Opportunity status field with error message
- If Add Bank Account fails → log error, parse `errors` array, update Opportunity
- If List Bank Accounts fails or account not found → log error, update Opportunity
- Prevent duplicate submissions — check if `CI_Funding_Source_ID` is already populated before running

## Deliverables

1. **Deluge function:** `ci_add_account` — the main integration script
2. **Zoho Org Variables:** list above (13 variables)
3. **Zoho Workflow Rule:** trigger on Opportunity stage change
4. **Zoho Custom Button:** manual trigger on Opportunity layout
5. **Custom CRM Fields:** `CI_Funding_Source_ID`, `CI_Logo_ID` on Opportunity
6. **Documentation:** setup guide in the GitHub repo README

## Verification

1. Set up sandbox credentials in Zoho Org Variables
2. Create a test Opportunity with all required fields
3. Click the custom button → verify account created in CheckIssuing sandbox
4. Verify `CI_Funding_Source_ID` and `CI_Logo_ID` populated on the Opportunity
5. Verify `CI_Last_Check_Number` incremented by 1,000,000
6. Test the workflow rule by changing an Opportunity to "Closed Won"
7. Test error handling with invalid data
