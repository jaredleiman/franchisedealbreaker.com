# FranchiseDealBreaker — Zapier Automation Blueprint
# Two Zaps. One per product tier. Follow this exactly.

---

## ZAP 1: FULL RED FLAG REPORT ($497)

### Overview
Stripe payment confirmed → Tally form submitted → Claude analyzes FDD
→ PDF generated → Email delivered to buyer → Notion queue updated

---

### TRIGGER
App: Tally
Event: New Submission
Form: "FDD Submission — Full Red Flag Report"

Test: Submit a test entry with a sample FDD PDF to confirm trigger fires correctly.

---

### STEP 1 — Create Notion Record (Order Received)
App: Notion
Action: Create Database Item
Database: "FDD Analysis Queue"

Field Mappings:
- Franchise Name → {{franchise_name}} from Tally
- Buyer Name     → {{buyer_name}} from Tally
- Buyer Email    → {{buyer_email}} from Tally
- Report Type    → "Full Report" (hardcoded)
- Status         → "Received" (hardcoded)
- FDD File URL   → {{fdd_file_url}} from Tally
- Submitted At   → {{submission_time}} from Tally
- Notes          → {{buyer_notes}} from Tally

Why first: You want every order logged immediately, even if a later step fails.

---

### STEP 2 — Download FDD PDF
App: Webhooks by Zapier
Action: GET request
URL: {{fdd_file_url}} (the Tally file upload URL)

This downloads the PDF so you can pass it to Claude.
Output: Raw PDF file content (binary)

NOTE: If Tally's file URL is directly accessible, you can skip this step
and pass the URL directly to your Cloudflare Worker in Step 3.

---

### STEP 3 — Send to Claude Analysis (via Cloudflare Worker)
App: Webhooks by Zapier
Action: POST request
URL: https://fdb-analysis.YOUR_SUBDOMAIN.workers.dev

This hits your Cloudflare Worker (see WORKER_SETUP.md).
The Worker accepts the PDF URL + franchise name, calls the Claude API,
and returns the structured JSON analysis.

Request Body (JSON):
{
  "franchise_name": "{{franchise_name}}",
  "buyer_name": "{{buyer_name}}",
  "buyer_email": "{{buyer_email}}",
  "fdd_url": "{{fdd_file_url}}",
  "report_type": "full",
  "buyer_notes": "{{buyer_notes}}"
}

Output: JSON analysis object (see PROMPT_PIPELINE.md for schema)

Expected response time: 60–180 seconds for a 400-page FDD.
Set Zapier webhook timeout to 300 seconds.

---

### STEP 4 — Format PDF Report
App: Webhooks by Zapier OR DocuPilot / PDFMonkey (recommended)
Action: Generate PDF

Recommended tool: PDFMonkey (pdfmonkey.io)
- Create a PDF template in PDFMonkey matching the report design
- Pass the JSON analysis from Step 3 as template variables
- PDFMonkey returns a PDF download URL

Alternative: Use HTML-to-PDF via Cloudflare Worker
(generate HTML from the JSON, convert to PDF with puppeteer)

PDFMonkey Request Body:
{
  "document": {
    "document_template_id": "YOUR_PDFMONKEY_TEMPLATE_ID",
    "payload": {{step3_output_json}},
    "status": "pending"
  }
}

Output: PDF download URL

---

### STEP 5 — Send Report Email
App: Gmail (or SendGrid for better deliverability)
Action: Send Email

To:      {{buyer_email}}
From:    reports@franchisedealbreaker.com
Subject: Your FranchiseDealBreaker Red Flag Report — {{franchise_name}}

Body (plain text fallback):
```
Hi {{buyer_name}},

Your Full Red Flag Report for {{franchise_name}} is attached.

Overall Risk Score: {{step3_overall_score}}/10
Risk Tier: {{step3_risk_tier}}
Critical Flags: {{step3_critical_flags_count}}

Review the full report for item-by-item findings and your 10 questions to ask before signing.

—
FranchiseDealBreaker
franchisedealbreaker.com
hello@franchisedealbreaker.com

This report is for informational purposes only and does not constitute legal advice.
Always consult a licensed franchise attorney before signing any agreement.
```

Attachment: {{step4_pdf_url}} (the PDFMonkey output URL)

---

### STEP 6 — Update Notion Record (Delivered)
App: Notion
Action: Update Database Item

Find record by: Buyer Email + Franchise Name (from Step 1)
Update:
- Status → "Delivered"
- Report Sent → checked

---

### ERROR HANDLING

Add a Zapier "Error Handler" path for Step 3 (Claude call):
- If Step 3 fails or times out:
  - Send email to hello@franchisedealbreaker.com with subject "FAILED ANALYSIS — {{franchise_name}} / {{buyer_email}}"
  - Update Notion record Status to "Error"
  - Send buyer a delay notification email:
    "Hi {{buyer_name}}, we're experiencing a brief delay processing your {{franchise_name}} report. You'll receive it within 4 hours. We apologize for the inconvenience."

---

## ZAP 2: QUICK SCAN ($97)

### Overview
Identical structure to Zap 1 with these differences:
- Trigger form: "FDD Submission — Quick Scan"
- Step 1 Notion: Report Type = "Quick Scan"
- Step 3 Worker: report_type = "quick" (triggers shorter prompt)
- Step 5 Email Subject: "Your FranchiseDealBreaker Quick Scan — {{franchise_name}}"
- Delivery window in email: "12 hours" instead of "24 hours"

---

## CLOUDFLARE WORKER SETUP
## File: fdb-analysis-worker.js
## Deploy to: fdb-analysis.YOUR_SUBDOMAIN.workers.dev

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    const body = await request.json();
    const { franchise_name, buyer_email, fdd_url, report_type, buyer_notes } = body;

    // Fetch the PDF from Tally's file URL
    const pdfResponse = await fetch(fdd_url);
    const pdfBuffer = await pdfResponse.arrayBuffer();
    const pdfBase64 = btoa(String.fromCharCode(...new Uint8Array(pdfBuffer)));

    // Select prompt based on report type
    const systemPrompt = env.SYSTEM_PROMPT; // Store in Worker env variables
    const userPrompt = report_type === 'full'
      ? env.USER_PROMPT_FULL.replace('[FRANCHISE_NAME]', franchise_name)
      : env.USER_PROMPT_QUICK.replace('[FRANCHISE_NAME]', franchise_name);

    // Call Claude API
    const claudeResponse = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': env.ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01'
      },
      body: JSON.stringify({
        model: 'claude-opus-4-5',
        max_tokens: 4096,
        system: systemPrompt,
        messages: [{
          role: 'user',
          content: [
            {
              type: 'document',
              source: {
                type: 'base64',
                media_type: 'application/pdf',
                data: pdfBase64
              }
            },
            {
              type: 'text',
              text: userPrompt
            }
          ]
        }]
      })
    });

    const claudeData = await claudeResponse.json();

    // Parse JSON from Claude response
    let analysis;
    try {
      analysis = JSON.parse(claudeData.content[0].text);
    } catch (e) {
      return new Response(JSON.stringify({ error: 'Parse failed', raw: claudeData.content[0].text }), {
        status: 500,
        headers: { 'Content-Type': 'application/json' }
      });
    }

    return new Response(JSON.stringify(analysis), {
      headers: { 'Content-Type': 'application/json' }
    });
  }
};
```

### Worker Environment Variables to Set:
- ANTHROPIC_API_KEY — your Claude API key
- SYSTEM_PROMPT — full system prompt from PROMPT_PIPELINE.md
- USER_PROMPT_FULL — full report user prompt from PROMPT_PIPELINE.md
- USER_PROMPT_QUICK — quick scan user prompt from PROMPT_PIPELINE.md

---

## PDFMONKEY TEMPLATE SETUP

1. Sign up at pdfmonkey.io (free tier: 100 PDFs/month — enough to start)
2. Create a new template called "FDB Full Red Flag Report"
3. Design the template using their HTML editor
4. Use these variable placeholders in the template:
   - {{franchise_name}}, {{report_date}}, {{overall_score}}, {{risk_tier}}
   - {{executive_summary}}
   - Loop through {{items}} array for item-by-item findings
   - Loop through {{top_red_flags}} for the flagged items section
   - Loop through {{questions_for_franchisor}} for the 10 questions
5. Note your template ID — paste into Zapier Step 4

The template should match the newspaper aesthetic of the landing page.
Black/cream/red. Playfair Display headings. Courier for labels.

---

## TOTAL MONTHLY TOOL COSTS AT SCALE

| Tool            | Free Tier       | Paid If Needed        |
|-----------------|-----------------|----------------------|
| Zapier          | 100 tasks/mo    | $20/mo (Starter)     |
| PDFMonkey       | 100 PDFs/mo     | $29/mo (250 PDFs)    |
| Cloudflare Workers | 100K req/day | Free for this volume |
| Notion          | Free            | Free                 |
| Gmail/SendGrid  | Free to start   | $20/mo at volume     |
| Claude API      | Pay per use     | ~$40/mo at 50 reports|

Total tool overhead at 50 reports/month: ~$110/month
Revenue at 50 reports/month: ~$24,850
Net margin: ~99.6%
