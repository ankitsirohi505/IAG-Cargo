# Aerolink Cargo — Credit Application (GitHub Pages)

A static 2-step Web-to-Lead form. Posts directly to Salesforce → a `Lead` record is created in the `accounthierachydemo` org (`00Dbm00000yzWEvEAM`). Step 2 loads trade-account offerings live from the org via a public Apex REST endpoint.

Nothing here needs a build step. The whole thing is one `index.html` file.

---

## 1 · Host on GitHub Pages

1. Create a GitHub repo (public, e.g. `aerolink-credit-app`).
2. Copy `docs/index.html` (this folder) to the repo root, or put the folder as-is under `/docs`.
3. In the repo → Settings → Pages:
   - **Source**: Deploy from a branch
   - **Branch**: `main` · `/docs` (or `main` · `/` if you put `index.html` at the root)
4. Save. GitHub gives you a URL like `https://<your-username>.github.io/aerolink-credit-app/`. That's the customer-facing form.

## 2 · Salesforce fields already deployed

These live in the `force-app/main/default/objects/Lead/fields/` folder and have already been deployed to the org:

| API name | Label | Type |
|---|---|---|
| `Trading_Name__c` | Trading Name | Text(255) |
| `VAT_Number__c` | VAT Number | Text(40) |
| `Registered_Company_Number__c` | Registered Company Number | Text(40) |
| `Generic_Company_Email__c` | Generic Company Email | Email |
| `Estimated_Monthly_Currency__c` | Estimated Monthly Currency | Picklist (EUR/GBP/USD/INR/AED) |
| `Estimated_Monthly_Amount__c` | Estimated Monthly Amount | Currency(18,2) |
| `BA_Contact_Name__c` | Account Manager Name | Text(120) |
| `BA_Contact_Email__c` | Account Manager Email | Email |
| `Address_Line_2__c` | Address Line 2 | Text(255) |
| `Selected_Account_Ids__c` | Selected Account IDs | Long Text |
| `Selected_Account_Names__c` | Selected Account Names | Long Text |

Plus these on Account (for the Step 2 offerings):

| API name | Purpose |
|---|---|
| `Trade_Account_Type__c` | Picklist of the 10 account types (Export CASS, Cargo GHA …) |
| `Product_Code__c` | Z103, Z102, Z103-0107 … |
| `Is_Public_Offering__c` | Show this Account on the public form |
| `Offering_Tagline__c` | One-line description shown under the account name |

The **Web-to-Lead field IDs** are already wired into `index.html` under `CONFIG.FIELD_IDS`.

## 3 · Seeded Accounts

Eleven `Account` records were inserted, one per trade account type:

- Export CASS – Europe Hub · Export CASS – Americas · Export non-CASS – Global
- Courier CASS – Global · Courier non-CASS – Global
- UK Import CASS – Heathrow · UK Import non-CASS – Regional
- Import non-CASS – EU
- Cargo GHA – Menzies UK · Cargo GSA – Air Logistics Group
- Mail non-IATA – Global Postal

All are flagged `Is_Public_Offering__c = true`. Add or edit any Account with that flag on to change what shows up in Step 2 — no code change needed.

---

## 4 · Turn on the live Account feed (one-off Salesforce Setup)

Until you do this, Step 2 shows a hard-coded copy of the seeded accounts (safe fallback). To go live:

### 4.1 Create a Salesforce Site
- Setup → **User Interface → Sites and Domains → Sites** → **New**
- Label: `Aerolink Trade Credit`
- Site URL: `trade-credit`
- Active: ✓
- Default Web Address: the auto-generated `.my.site.com` URL is fine
- Save

### 4.2 Grant Guest User access
Click the site → **Public Access Settings**:
- **Enabled Apex Class Access** → add **AccountLookupService**
- **Object Settings → Account** → Read on the object + Read on these fields: `Name`, `Trade_Account_Type__c`, `Product_Code__c`, `Region__c`, `Offering_Tagline__c`, `Is_Public_Offering__c`, `BillingCountry`
- Save

### 4.3 CORS whitelist for GitHub Pages
- Setup → **Security → CORS** → **New**
- Origin URL: `https://<your-username>.github.io` (no path, no trailing slash)
- Save

### 4.4 Wire the URL into `index.html`
Get the Site's public REST URL (Setup → Sites → your site → "Custom URL" column). Put this in `CONFIG.ACCOUNTS_API` inside `index.html`:

```js
ACCOUNTS_API: "https://<yoursite>-dev-ed.develop.my.site.com/services/apexrest/accounts"
```

Push the change. Step 2 will now fetch offerings live from your org.

---

## 5 · Auto-response email (optional but nice)

Setup → **Feature Settings → Marketing → Leads → Auto-Response Rules** → New rule, criteria `Lead Source EQUALS Web`, then attach a Lightning Email Template ("Thank you for your credit application, reference: {!Lead.Name}"). Fires the moment the Lead lands.

## 6 · Assignment (route to a queue)

Setup → **Leads → Lead Assignment Rules** → New rule → default criteria `Lead Source EQUALS Web` → owner = a Queue you create called `Customer_Data_Stewardship`. The stewardship team picks up applications from that queue.

---

## 7 · Safety switches

Inside `index.html`, top of `<script>`:

```js
const CONFIG = {
  ORG_ID:       "00Dbm00000yzWEvEAM",  // your org
  ACCOUNTS_API: "",                    // empty = use bundled demo offerings
  LIVE_SUBMIT:  true,                  // false = show receipt without POSTing
  ...
};
```

- `LIVE_SUBMIT: false` → click through both steps, see the receipt, **no** Lead created. Perfect for demos.
- `LIVE_SUBMIT: true` → real POST to Salesforce.

## 8 · Test the live post from your machine

Uncomment these debug lines inside the form (already commented in the file):

```html
<input type="hidden" name="debug"      value="1">
<input type="hidden" name="debugEmail" value="you@example.com">
```

Salesforce sends you an email breaking down every field that was accepted or dropped. Remove after testing.
