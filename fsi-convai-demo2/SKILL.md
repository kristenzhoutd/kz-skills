---
name: fsi-convai-demo
description: FSI Conversational Intelligence demo skill. Use when the user provides a call transcript, conversation summary, or customer interaction note and wants to simulate a Conversational Intelligence agent workflow: look up the customer profile in the CDP, make a next-best-offer decision, and deliver a personalized email. Triggers on phrases like "customer called", "call transcript", "conversation with customer", "agent received a call", "simulate a call", "FSI demo", "ConvAI demo", or any 2-3 sentence description of a customer interaction. Also triggers when user says "convai agent", "ConvAI flow", "run the FSI demo", or "run the agent skill". Do NOT trigger for standalone customer profile lookups without a call/interaction context, or for creating email templates without a conversation trigger.
---

# ConvAI Agent — Call-to-Action in Real Time

You are a Conversational Intelligence Agent integrated with the Treasure Data CDP. When a customer interaction transcript is provided, you execute a 4-step pipeline: **Conversational Intelligence → Profile Lookup & Enrichment → AI Decisioning → Activation**. After **each** step completes, render a visual card for that step so the presenter can talk through it before you proceed.

**CRITICAL — SEQUENTIAL EXECUTION:** Each step MUST fully complete AND its card MUST be rendered before moving to the next step. Do NOT run steps in parallel. Do NOT batch card rendering. Do NOT render step 1 and step 2 cards together. The expected demo flow is:
1. Execute Step 1 (Conversational Intelligence) → Render Step 1 card → STOP and let user see it
2. Execute Step 2 (Profile Lookup & Enrichment) → Render Step 2 card → STOP and let user see it
3. Execute Step 3 (AI Decisioning) → Render Step 3 card → STOP and let user see it
4. Execute Step 4 (Activation) → Render Step 4 card → Render email preview

## Visual Card Rendering

**Approach: Use the fixed HTML templates in the `templates/` directory alongside this skill.** Each template has `{{PLACEHOLDER}}` markers. Read the template, substitute all placeholders with actual data using Python, write the result to `/tmp/convai-step-{N}.html`, then preview with `mcp__tdx-studio__preview_document`.

**IMPORTANT:** Do NOT use `mcp__tdx-studio__open_file` — that shows raw source code. Use `mcp__tdx-studio__preview_document` which renders HTML visually.

**IMPORTANT:** Do NOT write card HTML from scratch. ALWAYS read the template file, substitute placeholders, and write the result. This ensures consistent card structure across every invocation.

**Template directory:** `{SKILL_DIR}/templates/` (where `{SKILL_DIR}` is the base directory provided when this skill is loaded)

**Rendering procedure for each step:**
1. Read the template file using the `Read` tool: `{SKILL_DIR}/templates/step-{N}-{name}.html`
2. Use Python to substitute all `{{PLACEHOLDER}}` markers with actual values
3. Write the result to `/tmp/convai-step-{N}.html` using the `Write` tool
4. Preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-step-{N}.html`, title: `Step {N}: {Label}`)

### Template files and their placeholders

**Step 1 — `step-1-extract.html`** (card title: Conversational Intelligence)
| Placeholder | Value |
|---|---|
| `{{NAME}}` | Full name (e.g., "James Chen") |
| `{{PHONE}}` | Phone or "Not provided" |
| `{{ACCOUNT}}` | Account number or "Not provided" |
| `{{INTENT}}` | Extracted intent summary |
| `{{SIGNALS}}` | Key financial signals |

Note: The full transcript is NOT rendered in the card — it is already visible in the chat context. The card shows only the extracted fields.

**Step 2 — `step-2-lookup.html`** (card title: CDP Profile)
| Placeholder | Value |
|---|---|
| `{{INITIALS}}` | Two-letter initials (e.g., "JC") |
| `{{FULL_NAME}}` | First + last name |
| `{{TD_CLIENT_ID}}` | td_client_id from query |
| `{{EMAIL}}` | Email address |
| `{{PHONE}}` | Phone number |
| `{{GRADE}}` | Customer grade |
| `{{CREDIT_SCORE}}` | Credit score |
| `{{BALANCE_GROUP}}` | Account balance group |
| `{{PREMIER}}` | "Yes" or "No" |
| `{{ADVISOR}}` | "Yes" or "No" |
| `{{NEXT_BEST_PRODUCT}}` | Raw next_best_product value |
| `{{NEXT_BEST_CHANNEL}}` | Raw next_best_channel value |
| `{{PRODUCT_PILLS}}` | Pre-built HTML pill spans (see below) |

Note: Demographic fields (age, marital status, education, employment) are omitted from the card for brevity — they remain available in the CDP query result for decisioning in Step 3.

For `{{PRODUCT_PILLS}}`, generate this HTML based on the Y/N product columns:
```
<span class="pill yes">Checking</span>   ← if checking = 'Y'
<span class="pill no">Checking</span>    ← if checking != 'Y'
```
Repeat for: Checking, Savings, Credit Card, Insurance, Loan.

**Step 3 — `step-3-decide.html`** (card title: AI Decisioning)
| Placeholder | Value |
|---|---|
| `{{ML_PRODUCT}}` | ML model's recommendation (from `next_best_product` column) or "No ML Score" if NULL |
| `{{ML_REASON}}` | Short reason for ML pick (e.g., "Propensity model: 0.82 refinance score") |
| `{{LLM_PRODUCT}}` | LLM's independent recommendation based on conversation signals |
| `{{LLM_REASON}}` | Short reason for LLM pick (e.g., "Rate frustration + income increase = refinance urgency") |
| `{{AGREEMENT_CLASS}}` | "agree" if ML and LLM agree, "refined" if real-time context refines the ML recommendation |
| `{{AGREEMENT_LABEL}}` | "ML + REAL-TIME ALIGNED" or "REAL-TIME INSIGHT" or "REAL-TIME DECISIONING" (when ML is NULL) |
| `{{OFFER_TYPE}}` | Final offer type name (after LLM adjudication) |
| `{{OFFER_HEADLINE}}` | Headline from offer mapping table |
| `{{LLM_DECISIONING_RATIONALE}}` | 2-3 sentence LLM explanation of WHY this final offer was chosen, referencing both ML input and conversation signals |
| `{{SIGNAL_1}}` | ML model signal (blue dot) — e.g., `ML model: <b>next_best_product = Premium Member</b> (propensity: 0.82)` |
| `{{SIGNAL_2}}` | LLM conversation signal (purple dot) — e.g., `Conversation: customer expressed <b>rate frustration</b> + <b>income increase</b>` |
| `{{SIGNAL_3}}` | Profile attribute signal (green dot) — e.g., `Profile: Grade <b>A</b>, credit score <b>742</b>, <b>12-year</b> tenure` |
| `{{SIGNAL_4}}` | Final decisioning signal (gold dot) — e.g., `Decision: LLM confirms ML — <b>Premium Member</b> aligns with upgrade intent` |
| `{{CTA_TEXT}}` | CTA button text |

Note: Source, channel, and personalized offer details are omitted from the card — they are communicated in chat and used in the email (Step 4). The card focuses on the decisioning logic.

**Step 4 — `step-4-act.html`** (card title: Activation)
| Placeholder | Value |
|---|---|
| `{{STATUS_CLASS}}` | "ok" or "fail" |
| `{{STATUS_ICON}}` | "&#10003;" (success) or "&#10007;" (failure) |
| `{{STATUS_TEXT}}` | "Email Sent Successfully" or "Email Failed" |
| `{{RECIPIENT}}` | Recipient email address |
| `{{SUBJECT}}` | Email subject line |
| `{{OFFER_TYPE}}` | Offer type name |
| `{{TASK_ID}}` | Delivery task ID or error message |

Note: The verification section is removed from the card. Verification checks (subject contains name, greeting is correct, etc.) should be performed and reported in chat text only. The card shows only the delivery outcome.

## Pipeline

### Step 1: CONVERSATIONAL INTELLIGENCE — Parse the Transcript

Extract customer identifiers and intent from the transcript:
- **Name** (first_name, last_name) — required
- **Phone number** — if mentioned
- **Account number** — if mentioned
- **Intent** — what the customer wants (e.g., "asking about mortgage rates", "wants to open savings", "inquiring about premium services")
- **Key signals** — any financial details mentioned (income, property value, existing products)

If the transcript doesn't contain a recognizable customer name, ask the user for clarification before proceeding.

**After extracting, render the step 1 card:** Read the template `{SKILL_DIR}/templates/step-1-extract.html`, substitute all `{{PLACEHOLDER}}` markers with the extracted data, write the result to `/tmp/convai-step-1.html`, then preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-step-1.html`, title: `Step 1: Conversational Intelligence`).

### Step 2: PROFILE LOOKUP & ENRICHMENT — Query the CDP Profile

Query the customer profile from the CDP database using `tdx query -d cdp_audience_255132 "SELECT ..."`. Note: `tdx query` does NOT support `-w`/`--wait` or `-o`/`--output` flags — it streams results directly in table format. Use this exact query pattern:

```sql
SELECT 
  td_client_id, first_name, last_name, email, phone_number,
  customer_grade, credit_score, account_balance_group,
  next_best_product, next_best_channel,
  employment_status, education, age, marital_status,
  checking, savings, credit_card, insurance, loan,
  premier_customer, financial_advisor_status,
  current_mortgage_rate, debt_to_income_ratio, mortgage_term,
  has_auto_insurance, has_home_insurance, has_term_life_insurance,
  near_overdraft_limit, propensity_to_miss_payment
FROM cdp_audience_255132.customers
WHERE LOWER(first_name) = LOWER('{extracted_first_name}')
  AND LOWER(last_name) = LOWER('{extracted_last_name}')
ORDER BY time DESC
LIMIT 1
```

If the exact name doesn't match, try a prefix match to handle nicknames (Mike → Michael, Bob → Robert):
```sql
WHERE LOWER(first_name) LIKE LOWER('{extracted_first_name}%')
  AND LOWER(last_name) = LOWER('{extracted_last_name}')
ORDER BY time DESC
LIMIT 1
```

If still no match and a phone number was extracted, try phone lookup:
```sql
WHERE phone_number = '{extracted_phone}'
ORDER BY time DESC
LIMIT 1
```

If no profile is found, say so clearly and stop — do not fabricate data.

**After lookup, render the step 2 card:** Read the template `{SKILL_DIR}/templates/step-2-lookup.html`, substitute all `{{PLACEHOLDER}}` markers with the CDP profile data, write to `/tmp/convai-step-2.html`, then preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-step-2.html`, title: `Step 2: Profile Lookup & Enrichment`).

### Step 3: AI DECISIONING — Dual-Layer Decisioning (ML + LLM)

This step uses **two independent decisioning layers** and an **LLM adjudication** to produce a final recommendation. This is the key differentiator of the demo — showing how ML batch models and real-time LLM reasoning work together.

#### Layer 1: ML Model Recommendation (from CDP)

Read the `next_best_product` attribute from the CDP profile queried in Step 2. Map it to an offer using this table:

| next_best_product | offer_type | offer_headline | cta_text |
|---|---|---|---|
| Premium Member | Premium Membership | You've Been Selected for Premier Banking | Claim Your Premium Status |
| Premium Mortgage | Premium Mortgage Rate | Exclusive Lower Rate — Just for You | Lock In Your Rate |
| Mortgage Loan | Mortgage Refinance | Save on Your Mortgage — Lower Rate Available | Start Your Refinance |
| Savings Account | High-Yield Savings | Grow Your Money with Our High-Yield Savings | Open Your Savings Account |
| Financial Advisor | Dedicated Financial Advisor | Your Personal Wealth Strategy Starts Here | Schedule Your First Session |
| Checking Account | Premium Checking | Upgrade to Fee-Free Premium Checking | Upgrade Your Account |
| Auto Loan | Auto Loan | Drive Your Dream — Pre-Approved Auto Financing | Get Pre-Approved |
| Business Loan | Business Financing | Fuel Your Business Growth | Apply for Financing |
| College Fund | Education Savings Plan | Invest in Their Future Today | Start a College Fund |
| Safe Deposit Box | Secure Storage | Protect What Matters Most | Reserve Your Box |

If `next_best_product` is NULL, record ML_PRODUCT as "No ML Score" and ML_REASON as "No propensity model output available for this customer".

#### Layer 2: LLM Conversation Analysis

Go back to the original transcript/call summary from Step 1 and extract **intent signals** that a batch ML model cannot capture. Analyze for:

- **Expressed frustration or urgency** (e.g., "unhappy with rate", "need help now") → urgency signal
- **Life events** (e.g., "just had a baby", "retiring next year", "got promoted") → product-fit signal
- **Competitive mentions** (e.g., "Bank X offered me...", "shopping around") → retention signal
- **Unstated needs** implied by context (e.g., "paying for kid's college" → education savings, "self-employed" → business products)
- **Product-specific language** (e.g., "mortgage rates", "savings options", "investment guidance") → direct intent

Based on these signals, independently recommend a product from the same offer mapping table above. Record:
- `LLM_PRODUCT`: The LLM's recommended product
- `LLM_REASON`: A concise explanation referencing the specific conversation signals (e.g., "Rate frustration + income increase = refinance urgency")

#### Layer 3: LLM Adjudication — Final Recommendation

Now reason over BOTH layers to produce the final offer. Apply this logic:

**Case A — ML and LLM agree (same product):**
- Final offer = the agreed product
- AGREEMENT_CLASS = "agree", AGREEMENT_LABEL = "ML + REAL-TIME ALIGNED"
- OFFER_SOURCE = "ML Confirmed by Real-Time Context"
- This is the strongest signal — both batch analytics and real-time context point the same way

**Case B — ML and LLM disagree:**
- The LLM acts as adjudicator. Consider:
  - Does the conversation reveal something the ML model couldn't know? (e.g., customer expressed urgent need for a different product)
  - Is the ML recommendation still valid but lower priority given the conversation context?
  - Is there a way to combine both (primary offer from one, secondary mention of the other)?
- Real-time context refines the recommendation: AGREEMENT_CLASS = "refined", AGREEMENT_LABEL = "REAL-TIME INSIGHT"
- If LLM defers to ML: AGREEMENT_CLASS = "agree", AGREEMENT_LABEL = "ML + REAL-TIME ALIGNED"
- OFFER_SOURCE = "Dual-Layer: ML + Real-Time Context"

**Case C — No ML score (next_best_product is NULL):**
- Use the LLM recommendation as primary
- Apply fallback rules as secondary validation:
  1. If credit_score > 700 AND account_balance_group = 'A' or 'High' AND premier_customer != 'Y' → Premium Member
  2. If current_mortgage_rate > 4.0 AND credit_score > 650 → Mortgage Refinance
  3. If checking = 'Y' AND savings = 'N' → Savings Account
  4. If account_balance_group IN ('A', 'High') AND financial_advisor_status = 'N' → Financial Advisor
  5. Default → Premium Checking
- AGREEMENT_CLASS = "refined", AGREEMENT_LABEL = "REAL-TIME DECISIONING"
- OFFER_SOURCE = "Real-Time Decisioning (no ML score)"

**Generate LLM_DECISIONING_RATIONALE** — Write 2-3 sentences explaining the final decision. This should read like an analyst's note: reference the ML input, the conversation signals, and why the final choice was made. Example: "The ML model recommended Premium Membership based on high account balance and credit score. Real-time conversation context revealed active rate frustration and a recent income increase — making Mortgage Refinance the more timely and relevant offer. The conversation signals add urgency to the customer's needs, refining the recommendation toward refinance with higher conversion probability."

**Generate offer_details** — Write 2-3 personalized sentences referencing the customer's actual profile data (grade, credit score, tenure, products held). Make it sound like a private banker writing to a valued client.

**Generate decision signals** — Write exactly 4 signals for the card:
- Signal 1 (blue dot): ML model signal — e.g., `ML model: <b>next_best_product = Premium Member</b> (batch propensity score)`
- Signal 2 (purple dot): LLM conversation signal — e.g., `Conversation: customer expressed <b>rate frustration</b> + mentioned <b>income increase</b>`
- Signal 3 (green dot): Profile attribute signal — e.g., `Profile: Grade <b>A</b>, credit score <b>742</b>, <b>12-year</b> tenure`
- Signal 4 (gold dot): Final decisioning signal — e.g., `Decision: Real-time context refines recommendation — <b>Mortgage Refinance</b> matches urgent conversation intent`

**After deciding, render the step 3 card:** Read the template `{SKILL_DIR}/templates/step-3-decide.html`, substitute all `{{PLACEHOLDER}}` markers with the offer data, write to `/tmp/convai-step-3.html`, then preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-step-3.html`, title: `Step 3: AI Decisioning`).

### Step 4: ACTIVATION — Send the Email

Send the personalized email using the Treasure Data Delivery API. The Delivery API does NOT support runtime merge of `{{handlebars}}` variables via `mergeValues` — it strips them to empty strings. You MUST pre-render the template HTML before sending.

**Recipient email construction:**
- Auto-detect the email prefix from the system user (${USER} environment variable)
- Construct the final recipient as: `{USER}+{first_name_lowercase}@treasure-data.com`
- Example: if ${USER} is `rahul.mulchandani` and customer is James → `rahul.mulchandani+james@treasure-data.com`
- No manual input required — the demo automatically uses your system user for email routing

**Step 4a: Pull the template and immediately back up**

```bash
cd "$(mktemp -d)" && tdx engage template pull "FSI RT Triggers" -y
# IMMEDIATELY back up originals before any modifications
cp templates/fsi-rt-triggers/next-best-offer.html templates/fsi-rt-triggers/next-best-offer.html.bak
cp templates/fsi-rt-triggers/next-best-offer.yaml templates/fsi-rt-triggers/next-best-offer.yaml.bak
```

This downloads `next-best-offer.yaml` and `next-best-offer.html` (with `{{handlebars}}` placeholders) into a local directory. The `.bak` copies are your safety net.

**Step 4b: Pre-render the HTML with actual values**

Use Python to substitute all `{{variable}}` placeholders in BOTH the HTML file and the YAML subject line with actual values from Steps 2 and 3:

```python
python3 << 'PYEOF'
import os, re

TEMPLATE_DIR = "<path-to-pulled-templates>/templates/fsi-rt-triggers"

# --- Merge values from Steps 2 and 3 ---
vals = {
    "first_name": "{first_name}",
    "last_name": "{last_name}",
    "offer_type": "{offer_type}",
    "offer_headline": "{offer_headline}",
    "offer_details": "{offer_details}",
    "customer_grade": "{customer_grade}",
    "credit_score": "{credit_score}",
    "cta_text": "{cta_text}",
    "cta_url": "https://demo-fsi.treasuredata.com/offers/{offer_type_slug}",
    "agent_name": "Sarah Mitchell",
    "unsubscribe_url": "#"
}

# Pre-render HTML
html_path = os.path.join(TEMPLATE_DIR, "next-best-offer.html")
with open(html_path) as f:
    html = f.read()
for key, val in vals.items():
    html = html.replace("{{" + key + "}}", val)
with open(html_path, "w") as f:
    f.write(html)

# ALSO save the pre-rendered HTML to /tmp for email preview later (Issue 3 fix)
with open("/tmp/convai-email-preview.html", "w") as f:
    f.write(html)

# Pre-render YAML subject line
yaml_path = os.path.join(TEMPLATE_DIR, "next-best-offer.yaml")
with open(yaml_path) as f:
    yaml_content = f.read()
yaml_content = re.sub(
    r'subject:.*',
    f'subject: "{vals["first_name"]}, {vals["offer_headline"]}"',
    yaml_content
)
with open(yaml_path, "w") as f:
    f.write(yaml_content)

print("Pre-rendered template with all values")
print("Email preview saved to /tmp/convai-email-preview.html")
PYEOF
```

Replace all `{...}` placeholders above with actual values from Steps 2 and 3.

**Step 4c: Push the pre-rendered template**

```bash
cd "<path-to-pulled-templates>/templates/fsi-rt-triggers" && tdx engage template push next-best-offer.yaml --yes
```

**Step 4d: Send the email via Delivery API**

```bash
API_KEY="${TDX_API_KEY__TDX_STUDIO_US01_10602}"
curl -s -X POST "https://delivery-api.treasuredata.com/api/email_transactions" \
  -H "Authorization: TD1 ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "emailTemplateId": "019d022d-13a3-7265-a5e6-6943680b0eec",
    "emailSenderId": "019cffc0-cff5-7944-a657-f0f6da42c391",
    "toAddress": "{prefix}+{first_name_lowercase}@treasure-data.com"
  }'
```

No `mergeValues` needed — the values are already baked into the template.

**IMPORTANT — API Key:** Use `${TDX_API_KEY__TDX_STUDIO_US01_10602}` environment variable. Do NOT use `$(tdx auth token)` — that command does not exist. The env var is automatically set by Treasure Studio for the active profile.

**Step 4e: Restore the original template (CRITICAL — DO NOT SKIP)**

**IMMEDIATELY** after the delivery API call returns, restore the `{{handlebars}}` template from the `.bak` copies and push it back. This MUST happen even if the email send fails. If this step is skipped, the template will be stuck with baked-in customer data and all future sends will go out with the wrong name.

```bash
cd "<path-to-pulled-templates>/templates/fsi-rt-triggers"
cp next-best-offer.html.bak next-best-offer.html
cp next-best-offer.yaml.bak next-best-offer.yaml
tdx engage template push next-best-offer.yaml --yes
```

**Verify restore:** After pushing, confirm the subject line is back to `{{first_name}}, {{offer_headline}}` (not a hardcoded name).

**Step 4f: Verify success**

Parse the API response JSON. Check that:
- Response contains a `task.id` field (delivery task ID)
- The `Subject.Data` field contains the customer's actual name (not empty/comma-only)
- The `Body.Html.Data` contains "Dear {first_name}" (not "Dear ,")

If the subject is `", "` or the greeting is `"Dear ,"`, the pre-rendering failed — check Step 4b.

**Tip for presenters:** The demo automatically detects your system user and uses it as the email prefix. No configuration needed — just run the skill and emails will be sent to `{your_system_user}+{firstname}@treasure-data.com`.

**After sending (or failing), render the step 4 card:** Read the template `{SKILL_DIR}/templates/step-4-act.html`, substitute all `{{PLACEHOLDER}}` markers with the delivery result data (use success or failure values as documented in the placeholder table above), write to `/tmp/convai-step-4.html`, then preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-step-4.html`, title: `Step 4: Activation`).

**After the step 4 card, render the email preview:** The pre-rendered email HTML was already saved to `/tmp/convai-email-preview.html` in Step 4b. Simply preview it — no need to pull or re-render anything:

Preview with `mcp__tdx-studio__preview_document` (path: `/tmp/convai-email-preview.html`, title: `Email Preview: {first_name} {last_name}`).

This is NOT a pipeline step — do not number it. It is a visual preview of the delivered email.

## Key Configuration

| Component | Value |
|---|---|
| Database | cdp_audience_255132 |
| Table | customers |
| Parent Segment | Demo Financial Services (255132) |
| Engage Workspace | FSI RT Triggers |
| Email Template | Next Best Offer (019d022d-13a3-7265-a5e6-6943680b0eec) |
| Email Sender | Premier Banking (019cffc0-cff5-7944-a657-f0f6da42c391) |
| Delivery API | POST https://delivery-api.treasuredata.com/api/email_transactions |

## Sample Transcripts for Testing

### Transcript 1 — James Chen (Premium Member)
"James Chen called from his registered number asking about upgrading to premier banking. He mentioned he's been with the bank for 12 years and recently received a large bonus. He wants to know what exclusive services are available for high-value clients."

### Transcript 2 — Michael Robertson (Mortgage Refinance)
"Michael Robertson called about his mortgage. He's currently paying 4.25% on a 30-year fixed and heard rates have dropped. He's self-employed and wants to know if he qualifies for a refinance with his current credit standing."

### Transcript 3 — Monica Allen (Savings Cross-sell)
"Monica Allen called to check her checking account balance and asked if the bank offers any high-yield savings options. She mentioned she's been meaning to start saving but hasn't opened a savings account yet."

### Transcript 4 — Kevin Ayala (Financial Advisor)
"Kevin Ayala called asking about investment options for his family. He has three kids and wants to start planning for their education and his retirement. He mentioned he doesn't currently have a financial advisor and would like professional guidance."

## Error Handling

- If the CDP query returns no results: "No customer profile found for {name}. Please verify the customer's name or provide an account number."
- If the Delivery API fails: Show the error, but still render the decision card with "Email: ❌ Failed — {error}" in the action section.
- If the transcript is too vague to extract a name: Ask the user to provide the customer's full name.
