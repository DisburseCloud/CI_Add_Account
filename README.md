# CI_Add_Account

Zoho CRM integration that automatically creates bank accounts in [CheckIssuing](https://web.checkissuing.com) when Opportunities close.

## What It Does

When an Opportunity reaches "Closed Won" or "Payment Setup" (or when a user clicks the manual button), the Deluge script:

1. Reads deal-specific fields (name, bank number, parent account address)
2. Reads shared defaults from Zoho Org Variables
3. Authenticates with CheckIssuing via OAuth2
4. Creates a bank account via the Add Bank Account API
5. Retrieves the funding source ID and logo ID via List Bank Accounts
6. Stores the IDs back on the Opportunity record

## Setup

### 1. CheckIssuing API Credentials

1. Log into your CheckIssuing client panel
2. Click "API Access" in the user menu (top right)
3. Create new credentials (note the API ID and API Secret)
4. Create separate credentials for sandbox and production

### 2. Create CRM Custom Fields

In Zoho CRM, go to **Settings > Customization > Modules > Deals > Fields** and create:

| Field Name | Type | Section |
|---|---|---|
| `CI_Funding_Source_ID` | Single Line | CheckIssuing Integration |
| `CI_Logo_ID` | Single Line | CheckIssuing Integration |
| `Bank_Number` | Single Line | CheckIssuing Integration |
| `CI_Status` | Single Line | CheckIssuing Integration |

After creating, note the exact API field names from **Settings > Developer Space > APIs > API Names**.

### 3. Create Organization Variables

In **Settings > Developer Space > Organization Variables**, create:

| Variable | Value |
|---|---|
| `CI_API_ID` | Your CheckIssuing API ID |
| `CI_API_Secret` | Your CheckIssuing API Secret |
| `CI_Base_URL` | `https://websb.checkissuing.com` (sandbox) or `https://web.checkissuing.com` (production) |
| `CI_Bank_Name` | Florida Capital Bank |
| `CI_Bank_Address` | 10151 Deerwood Park Blvd, Jacksonville, FL 32256 |
| `CI_Fractional_Routing` | 063112142 |
| `CI_Return_MailTo` | DisburseCloud |
| `CI_Return_Address` | 30212 Tomas ste 355, Rancho Santa Margarita, CA 92688, USA |
| `CI_Void_Text` | VOID 180 DAYS AFTER ISSUE DATE |
| `CI_Default_Signature` | Base64-encoded signature image |
| `CI_Last_Check_Number` | 1000000 (starting value) |

### 4. Create the Deluge Function

1. Go to **Settings > Developer Space > Functions**
2. Create a new function named `ci_add_account`
3. Add parameter: `dealId` (String)
4. Paste the contents of `src/ci_add_account.dg`
5. Save and test

### 5. Create the Custom Button

1. Go to **Settings > Customization > Modules > Deals > Links and Buttons**
2. Create button: "Create CI Account"
3. Placement: View page (details page)
4. Action: Function > `ci_add_account`
5. Pass `dealId` = `{!Deals.Deal_Id}`

### 6. Create the Workflow Rule

1. Go to **Settings > Automation > Workflow Rules**
2. New Rule for module: Deals
3. Trigger: Field update > Stage
4. Condition: Stage = "Closed Won" OR Stage = "Payment Setup"
5. Instant action: Function > `ci_add_account` with dealId as the Deal ID

## Field Mapping

### Deal-Specific (from CRM)

| CheckIssuing Field | CRM Source |
|---|---|
| `name` | Deal Name |
| `name_on_checks` | Deal Name |
| `address_on_checks` | Parent Account billing address |
| `acct_num` | Bank_Number field |

### Shared Defaults (Org Variables)

| CheckIssuing Field | Value |
|---|---|
| `bank_name` | Florida Capital Bank |
| `acct_type` | CHECKING |
| `bank_addr` | 10151 Deerwood Park Blvd, Jacksonville, FL 32256 |
| `fractional_routing` | 063112142 |
| `return_mailto` | DisburseCloud |
| `return_address` | 30212 Tomas ste 355, Rancho Santa Margarita, CA 92688, USA |
| `is_business` | true |
| `void_text` | VOID 180 DAYS AFTER ISSUE DATE |
| `sig` | Default signature (base64) |
| `starting_check_num` | Auto-incremented by 1,000,000 per account |

## Troubleshooting

Check the **CI_Status** field on the Opportunity for error messages:

| Status | Meaning |
|---|---|
| `Success - Account created` | Everything worked |
| `Missing Deal_Name` | Opportunity has no name |
| `Missing Bank_Number` | Bank_Number field is empty |
| `Deal has no parent Account` | No Account linked to the Opportunity |
| `Auth failed: ...` | CheckIssuing API credentials are wrong |
| `Add Account failed: ...` | CheckIssuing rejected the request (check error details) |
| `List Accounts failed: ...` | Could not retrieve account list |
| `Account created but not found in list` | Account was created but name matching failed |

You can also check function execution logs in **Settings > Developer Space > Functions > ci_add_account > Logs**.

## Switching to Production

1. Update `CI_Base_URL` to `https://web.checkissuing.com`
2. Update `CI_API_ID` and `CI_API_Secret` with production credentials
3. Verify with a test Opportunity before going live

## Notes

- The script prevents duplicate submissions by checking if `CI_Funding_Source_ID` is already populated
- Check numbers auto-increment by 1,000,000 per account (tracked in `CI_Last_Check_Number`)
- API endpoint paths (`/api/v1/...`) are assumed from docs and may need adjustment during sandbox testing
- The `acct_num` parameter may not be explicitly documented but is needed since Plaid verification is skipped
