# CI_Add_Account Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Automate CheckIssuing bank account creation from Zoho CRM Opportunities, storing the returned funding source ID and logo ID back on the Opportunity.

**Architecture:** A single Deluge function (`ci_add_account`) called by both a workflow rule and a custom button. The function authenticates via OAuth2, calls Add Bank Account, then List Bank Accounts to retrieve IDs, and updates the CRM record. All shared config lives in Zoho Org Variables.

**Tech Stack:** Zoho CRM Deluge scripting, CheckIssuing REST API, OAuth2 client_credentials

---

### Task 1: Create CRM Custom Fields

These fields must be manually created in Zoho CRM before the script can reference them.

**Step 1: Create fields on the Deals (Opportunities) module**

In Zoho CRM → Settings → Modules → Deals → Fields:

- `CI_Funding_Source_ID` — Single Line, section: "CheckIssuing Integration"
- `CI_Logo_ID` — Single Line, section: "CheckIssuing Integration"
- `Bank_Number` — Single Line, section: "CheckIssuing Integration"
- `CI_Status` — Single Line, section: "CheckIssuing Integration" (for logging success/error messages)

**Step 2: Note the API names**

After creating the fields, go to Settings → Developer Space → APIs → API Names and note the exact API field names for each (e.g., `CI_Funding_Source_ID` may become `CI_Funding_Source_ID1` if there's a conflict). Record these for use in the script.

**Step 3: Commit a reference doc**

Save the field API names to the repo so we have a record.

---

### Task 2: Set Up Zoho Org Variables

These store shared defaults and API credentials.

**Step 1: Create Organization Variables**

In Zoho CRM → Settings → Developer Space → Organization Variables, create:

| Variable Name | Value |
|---|---|
| `CI_API_ID` | (your sandbox API ID) |
| `CI_API_Secret` | (your sandbox API Secret) |
| `CI_Bank_Name` | Florida Capital Bank |
| `CI_Bank_Address` | 10151 Deerwood Park Blvd, Jacksonville, FL 32256 |
| `CI_Fractional_Routing` | 063112142 |
| `CI_Return_MailTo` | DisburseCloud |
| `CI_Return_Address` | 30212 Tomas ste 355, Rancho Santa Margarita, CA 92688, USA |
| `CI_Void_Text` | VOID 180 DAYS AFTER ISSUE DATE |
| `CI_Default_Signature` | (base64 encoded signature image) |
| `CI_Last_Check_Number` | 1000000 |
| `CI_Base_URL` | https://websb.checkissuing.com |

**Step 2: Commit**

Note the variable names in the repo README for reference.

---

### Task 3: Write the Deluge Function — OAuth2 Authentication

**Files:**
- Create: `src/ci_add_account.dg` (Deluge source stored in repo for version control)

**Step 1: Write the OAuth2 helper section**

This gets a Bearer token from CheckIssuing. In Deluge, we use `invokeurl` with Basic Auth.

```java
// ===== AUTHENTICATE WITH CHECKISSUING =====
apiId = zoho.adminapi.getVariable("CI_API_ID");
apiSecret = zoho.adminapi.getVariable("CI_API_Secret");
baseUrl = zoho.adminapi.getVariable("CI_Base_URL");

authUrl = baseUrl + "/api/v1/oauth/token";
authHeader = "Basic " + zoho.encryption.base64Encode(apiId + ":" + apiSecret);

authResponse = invokeurl
[
    url: authUrl
    type: POST
    parameters: "grant_type=client_credentials"
    headers: {"Authorization": authHeader, "Content-Type": "application/x-www-form-urlencoded"}
];

if (authResponse.get("access_token") == null)
    // Update opportunity with error and exit
    zoho.crm.updateRecord("Deals", dealId, {"CI_Status": "Auth failed: " + authResponse.toString()});
    return;
end if;

accessToken = authResponse.get("access_token");
info "CI Auth successful, token obtained";
```

**Step 2: Test in Zoho sandbox**

Create a temporary Deluge function in Zoho CRM (Settings → Developer Space → Functions), paste just the auth section, hardcode a test `dealId`, and run it. Verify you get a token back.

**Step 3: Commit**

```bash
git add src/ci_add_account.dg
git commit -m "feat: add OAuth2 authentication for CheckIssuing API"
```

---

### Task 4: Write the Deluge Function — Read CRM Data & Build Payload

**Files:**
- Modify: `src/ci_add_account.dg`

**Step 1: Add the function header and CRM data reading**

This goes at the top of the function, before the auth block from Task 3.

```java
// ===== CI_ADD_ACCOUNT FUNCTION =====
// Input: dealId (String) - the Zoho CRM Deal/Opportunity record ID
//
// Called by: Workflow Rule (stage change) and Custom Button

// Guard: prevent duplicate submissions
deal = zoho.crm.getRecordById("Deals", dealId);
existingFundingId = ifnull(deal.get("CI_Funding_Source_ID"), "");
if (existingFundingId != "")
    info "CI account already exists for this deal. Funding Source ID: " + existingFundingId;
    return;
end if;

// ===== READ DEAL-SPECIFIC FIELDS =====
dealName = deal.get("Deal_Name");
bankNumber = deal.get("Bank_Number");

// Get parent account address
accountId = deal.get("Account_Name").get("id");
account = zoho.crm.getRecordById("Accounts", accountId);
// Build address from Account fields
street = ifnull(account.get("Billing_Street"), "");
city = ifnull(account.get("Billing_City"), "");
state = ifnull(account.get("Billing_State"), "");
zip = ifnull(account.get("Billing_Code"), "");
country = ifnull(account.get("Billing_Country"), "");
addressOnChecks = street + ", " + city + ", " + state + " " + zip;
if (country != "")
    addressOnChecks = addressOnChecks + ", " + country;
end if;

// ===== READ SHARED DEFAULTS FROM ORG VARIABLES =====
bankName = zoho.adminapi.getVariable("CI_Bank_Name");
bankAddr = zoho.adminapi.getVariable("CI_Bank_Address");
fractionalRouting = zoho.adminapi.getVariable("CI_Fractional_Routing");
returnMailTo = zoho.adminapi.getVariable("CI_Return_MailTo");
returnAddress = zoho.adminapi.getVariable("CI_Return_Address");
voidText = zoho.adminapi.getVariable("CI_Void_Text");
defaultSig = zoho.adminapi.getVariable("CI_Default_Signature");
lastCheckNum = zoho.adminapi.getVariable("CI_Last_Check_Number").toLong();
startingCheckNum = lastCheckNum + 1000000;
```

**Step 2: Commit**

```bash
git add src/ci_add_account.dg
git commit -m "feat: add CRM data reading and payload building"
```

---

### Task 5: Write the Deluge Function — Add Bank Account API Call

**Files:**
- Modify: `src/ci_add_account.dg`

**Step 1: Add the API call after the auth block**

```java
// ===== CALL ADD BANK ACCOUNT =====
addAccountUrl = baseUrl + "/api/v1/bank-account/add";

addParams = Map();
addParams.put("name", dealName);
addParams.put("bank_name", bankName);
addParams.put("name_on_checks", dealName);
addParams.put("address_on_checks", addressOnChecks);
addParams.put("acct_num", bankNumber);
addParams.put("sig", defaultSig);
addParams.put("acct_type", "CHECKING");
addParams.put("bank_addr", bankAddr);
addParams.put("starting_check_num", startingCheckNum.toString());
addParams.put("fractional_routing", fractionalRouting);
addParams.put("return_mailto", returnMailTo);
addParams.put("return_address", returnAddress);
addParams.put("is_business", "true");
addParams.put("void_text", voidText);

addResponse = invokeurl
[
    url: addAccountUrl
    type: POST
    parameters: addParams
    headers: {"Authorization": "Bearer " + accessToken}
];

info "Add Account Response: " + addResponse.toString();

if (addResponse.get("status") != 1)
    errors = ifnull(addResponse.get("errors"), List());
    errorMsg = "Add Account failed: " + errors.toString();
    zoho.crm.updateRecord("Deals", dealId, {"CI_Status": errorMsg});
    return;
end if;

info "CI Account created successfully";
```

**Step 2: Test in Zoho sandbox**

Run the function with a test deal that has `Deal_Name`, `Bank_Number`, and a parent Account with a billing address. Check the CheckIssuing sandbox dashboard to confirm the account was created.

**Step 3: Commit**

```bash
git add src/ci_add_account.dg
git commit -m "feat: add CheckIssuing Add Bank Account API call"
```

---

### Task 6: Write the Deluge Function — List Bank Accounts & Store IDs

**Files:**
- Modify: `src/ci_add_account.dg`

**Step 1: Add List Bank Accounts call and ID extraction**

```java
// ===== FETCH ACCOUNT IDS =====
listUrl = baseUrl + "/api/v1/bank-account/list";

listResponse = invokeurl
[
    url: listUrl
    type: POST
    headers: {"Authorization": "Bearer " + accessToken}
];

info "List Accounts Response: " + listResponse.toString();

if (listResponse.get("status") != 1)
    zoho.crm.updateRecord("Deals", dealId, {"CI_Status": "List accounts failed: " + listResponse.toString()});
    return;
end if;

// Find the account matching our deal name
accounts = listResponse.get("accounts");
fundingSourceId = "";
logoId = "";
foundAccount = false;

for each accountKey in accounts.keys()
    acct = accounts.get(accountKey);
    if (acct.get("name") == dealName)
        fundingSourceId = acct.get("id");
        logoId = ifnull(acct.get("logo"), "");
        foundAccount = true;
        break;
    end if;
end for;

if (!foundAccount)
    zoho.crm.updateRecord("Deals", dealId, {"CI_Status": "Account created but not found in list. Name searched: " + dealName});
    return;
end if;

// ===== UPDATE CRM RECORD =====
updateMap = Map();
updateMap.put("CI_Funding_Source_ID", fundingSourceId);
updateMap.put("CI_Logo_ID", logoId);
updateMap.put("CI_Status", "Success - Account created");
zoho.crm.updateRecord("Deals", dealId, updateMap);

// ===== UPDATE LAST CHECK NUMBER =====
zoho.adminapi.updateVariable("CI_Last_Check_Number", startingCheckNum.toString());

info "CI Integration complete. Funding Source ID: " + fundingSourceId + ", Logo: " + logoId;
```

**Step 2: Test end-to-end in Zoho sandbox**

Run the complete function. Verify:
- CheckIssuing sandbox shows the new account
- Opportunity record has `CI_Funding_Source_ID` and `CI_Logo_ID` populated
- `CI_Last_Check_Number` org variable incremented by 1,000,000
- `CI_Status` shows "Success - Account created"

**Step 3: Commit**

```bash
git add src/ci_add_account.dg
git commit -m "feat: add List Bank Accounts call and CRM update with IDs"
```

---

### Task 7: Create the Custom Button

**Step 1: Create the button in Zoho CRM**

Settings → Customization → Modules → Deals → Links and Buttons → New Button:

- Button name: `Create CI Account`
- Placement: View page (details page)
- Action: Deluge function → select `ci_add_account`
- Pass `dealId` = `{!Deals.Deal_Id}`

**Step 2: Test**

Open a test Opportunity → click "Create CI Account" → verify the full flow works.

---

### Task 8: Create the Workflow Rule

**Step 1: Create the workflow in Zoho CRM**

Settings → Automation → Workflow Rules → New Rule:

- Module: Deals
- Rule trigger: Field update → Stage
- Condition: Stage equals "Closed Won" OR Stage equals "Payment Setup"
- Instant action: Function → select `ci_add_account`
- Pass `dealId` as the Deal ID

**Step 2: Test**

Change a test Opportunity's stage to "Closed Won" → verify the function fires and the account is created.

---

### Task 9: Write README and Final Commit

**Files:**
- Create: `README.md`

**Step 1: Write the README**

Document:
- Project purpose
- Prerequisites (CheckIssuing sandbox account, Zoho CRM admin access)
- Setup steps (org variables, custom fields, function, button, workflow rule)
- Field mapping reference
- Troubleshooting (check `CI_Status` field for errors)

**Step 2: Final commit and push**

```bash
git add README.md
git commit -m "docs: add setup guide and field mapping reference"
git push origin main
```

---

## Verification Checklist

1. Sandbox credentials configured in Zoho Org Variables
2. Custom fields visible on Opportunity layout
3. Custom button click → account created in CI sandbox → IDs stored on Opportunity
4. Workflow rule fires on stage change → same result
5. Duplicate guard works (running twice doesn't create a second account)
6. Error states produce clear messages in `CI_Status`
7. `CI_Last_Check_Number` increments correctly across multiple runs
8. All code committed and pushed to `DisburseCloud/CI_Add_Account`

## Notes

- **API endpoint paths** (`/api/v1/oauth/token`, `/api/v1/bank-account/add`, `/api/v1/bank-account/list`) are assumed based on the docs. Verify exact paths against the CheckIssuing sandbox — the docs don't explicitly show full URL paths.
- **`acct_num` parameter** — the Add Bank Account docs didn't explicitly list this field, but since we're skipping Plaid verification, it likely needs to be passed. Verify during sandbox testing.
- **Zoho field API names** — Zoho sometimes appends numbers to custom field API names (e.g., `CI_Funding_Source_ID1`). Check actual API names after creating the fields.
- **Switch to production** — when ready, update `CI_Base_URL` org variable to `https://web.checkissuing.com` and update `CI_API_ID`/`CI_API_Secret` to production credentials.
