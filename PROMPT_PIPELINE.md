# FranchiseDealBreaker — Claude Prompt Pipeline
# The FDD Analysis Engine

This document contains the complete prompt structure for the FDD analysis engine.
Use these prompts verbatim in your Zapier Claude action or direct API calls.

---

## HOW THE PIPELINE WORKS

1. Buyer submits FDD via Tally form (PDF upload + metadata)
2. Zapier fires on form submission
3. PDF text is extracted (see extraction note below)
4. System prompt + extracted text is sent to Claude API
5. Claude returns structured JSON analysis
6. JSON is formatted into PDF report via formatting step
7. PDF is emailed to buyer

---

## PDF TEXT EXTRACTION NOTE

Claude cannot directly receive PDF file uploads via Zapier.
You have two options:

**Option A (Recommended — simplest):**
Use the Claude API directly with document support.
Send the PDF as a base64-encoded document in the API call.
The Claude API natively handles PDFs up to ~100MB.
See: https://docs.anthropic.com/en/docs/build-with-claude/pdf-support

**Option B (Zapier-native):**
Add a Zapier step before the Claude action:
- Use "PDF.co" or "Adobe PDF Extract" Zapier integration
- Extract text from the uploaded PDF
- Pass the extracted text string to Claude

Option A is cleaner and more accurate. Use it if you're comfortable
with a lightweight Cloudflare Worker (same pattern as ContractorBidPro).

---

## SYSTEM PROMPT
## (Paste this into the "System" field of your Claude API call or Zapier Claude action)

```
You are an expert franchise due diligence analyst with deep knowledge of FTC Franchise Rule requirements, all 23 mandatory FDD Items, and historical patterns of franchisee failure across franchise systems.

Your job is to analyze Franchise Disclosure Documents (FDDs) and produce structured, plain-English risk assessments for prospective franchise buyers who are not lawyers.

You are NOT providing legal advice. You are providing information synthesis and analysis — the same function as a financial data provider summarizing a 10-K filing. Always make this clear.

YOUR ANALYSIS PRINCIPLES:
- Be direct and honest. If something is a red flag, say so clearly.
- Write for a smart non-lawyer. No legal jargon without plain-English explanation.
- Focus on the items that historically correlate most with franchisee failure: Items 3, 12, 17, 19, 20, 21.
- Flag omissions as prominently as problematic disclosures. What is NOT disclosed is often as important as what is.
- Assign risk scores based on the weighted framework provided.
- Never fabricate data. If something is not disclosed or not present in the document, say so explicitly.
- Your output must be valid JSON matching the schema provided exactly.
```

---

## USER PROMPT (FULL RED FLAG REPORT — $497 TIER)
## Replace [EXTRACTED_FDD_TEXT] with the actual FDD content.
## Replace [FRANCHISE_NAME] and [BUYER_EMAIL] with Tally form values.

```
Analyze the following Franchise Disclosure Document for [FRANCHISE_NAME] and produce a complete Red Flag Report.

BUYER CONTEXT:
- Franchise being evaluated: [FRANCHISE_NAME]
- Report type: Full Red Flag Report (all 23 Items)

FDD CONTENT:
[EXTRACTED_FDD_TEXT]

---

SCORING FRAMEWORK:
Score each item 1–10 (1 = very clean, 10 = severe red flag).
Weight the overall score as follows:
- Item 3 (Litigation): 20%
- Item 19 (Earnings Claims): 20%
- Item 12 (Territory): 15%
- Item 20 (Turnover): 15%
- Item 21 (Financials): 15%
- Item 17 (Renewal/Termination): 10%
- All other items combined: 5%

RISK TIER THRESHOLDS:
- 1.0–3.9: Low Risk
- 4.0–6.4: Medium Risk  
- 6.5–10.0: High Risk

---

RETURN YOUR ANALYSIS AS VALID JSON ONLY. No preamble, no explanation outside the JSON structure. Match this schema exactly:

{
  "franchise_name": "string",
  "report_date": "YYYY-MM-DD",
  "overall_score": 0.0,
  "risk_tier": "Low Risk | Medium Risk | High Risk",
  "critical_flags_count": 0,
  "warning_flags_count": 0,
  "clear_items_count": 0,
  "executive_summary": "string — 3–4 sentences. Plain English. Direct verdict on whether this is a concerning system and why.",
  "items": [
    {
      "item_number": 3,
      "item_title": "Litigation History",
      "score": 0.0,
      "severity": "Critical | High Risk | Watch | Clear",
      "headline": "string — one punchy sentence summarizing the finding",
      "detail": "string — 3–5 sentences of plain-English analysis. What was found, what it means, why it matters.",
      "flag": true
    }
    // ... repeat for all 23 items
  ],
  "top_red_flags": [
    {
      "item_number": 0,
      "title": "string",
      "one_liner": "string — one sentence plain-English summary of the red flag"
    }
    // include only items scored 6.5+, max 5
  ],
  "questions_for_franchisor": [
    "string — specific, direct question the buyer should ask before signing"
    // exactly 10 questions, ordered by importance
  ],
  "not_legal_advice_disclaimer": "This report is AI-powered information synthesis for educational purposes only and does not constitute legal advice. FranchiseDealBreaker is not a law firm. Always consult a licensed franchise attorney before signing any franchise agreement or making any investment decision."
}
```

Analyze every one of the 23 required FDD Items. If an item is not present or not disclosed in the document, note this explicitly in the detail field — omissions are themselves a finding.
```

---

## USER PROMPT (QUICK SCAN — $97 TIER)
## Shorter analysis, 5 items only.

```
Analyze the following Franchise Disclosure Document for [FRANCHISE_NAME] and produce a Quick Scan focusing on the 5 most critical items.

BUYER CONTEXT:
- Franchise being evaluated: [FRANCHISE_NAME]
- Report type: Quick Scan (Items 3, 12, 19, 20, 21 only)

FDD CONTENT:
[EXTRACTED_FDD_TEXT]

---

SCORING FRAMEWORK:
Score each item 1–10 (1 = very clean, 10 = severe red flag).
Compute overall score as equal-weighted average of the 5 items.

RISK TIER THRESHOLDS:
- 1.0–3.9: Low Risk
- 4.0–6.4: Medium Risk  
- 6.5–10.0: High Risk

---

RETURN VALID JSON ONLY. No preamble. Match this schema exactly:

{
  "franchise_name": "string",
  "report_date": "YYYY-MM-DD",
  "report_type": "Quick Scan",
  "overall_score": 0.0,
  "risk_tier": "Low Risk | Medium Risk | High Risk",
  "executive_summary": "string — 2–3 sentences. Plain English. Direct gut-check verdict.",
  "items": [
    {
      "item_number": 3,
      "item_title": "Litigation History",
      "score": 0.0,
      "severity": "Critical | High Risk | Watch | Clear",
      "headline": "string",
      "detail": "string — 2–3 sentences plain English"
    }
    // repeat for Items 12, 19, 20, 21 only
  ],
  "top_concern": "string — single most important thing this buyer should investigate further",
  "upgrade_prompt": "For a complete 23-Item analysis including territory rights, renewal terms, financial health, and 10 questions to ask your franchisor, upgrade to the Full Red Flag Report at franchisedealbreaker.com",
  "not_legal_advice_disclaimer": "This report is AI-powered information synthesis for educational purposes only and does not constitute legal advice. Always consult a licensed franchise attorney before signing."
}
```
```

---

## DATABASE PAGE GENERATION PROMPT
## Use this to generate free content for the 500+ franchise profile pages.
## Feed it a franchise name — Claude generates SEO-ready page content from its training data + public FDD knowledge.

```
You are writing free educational content for a franchise research database page about [FRANCHISE_NAME].

Write the following content sections as valid JSON. Base your analysis on publicly known information about this franchise system, including publicly available FDD data, news coverage, and franchisee community discussions. Do not fabricate specific numbers — if you are uncertain of a specific figure, describe it qualitatively or note it as approximate.

Return valid JSON only. No preamble. Schema:

{
  "franchise_name": "string",
  "slug": "string — URL-safe lowercase hyphenated",
  "category": "string — e.g. Fitness, Food & Beverage, Personal Services",
  "investment_range": "string — e.g. $150K–$450K",
  "franchise_fee": "string — e.g. $35,000",
  "royalty_rate": "string — e.g. 6% of gross sales",
  "ad_fund": "string — e.g. 2% of gross sales",
  "unit_count": "string — approximate total units",
  "founded_year": "string",
  "term_length": "string — e.g. 10 years",
  "free_risk_score": 0.0,
  "risk_tier": "Low Risk | Medium Risk | High Risk",
  "risk_score_class": "score-low | score-med | score-high",
  "pill_class": "pill-watch | pill-high | pill-critical",
  "risk_label": "Low | Medium | High",
  "item3_summary": "string — 2–3 sentences on publicly known litigation history or lack thereof",
  "item3_class": "clear | warning | flagged",
  "item3_severity": "Clear | Watch | Critical",
  "item3_pill": "pill-watch | pill-high | pill-critical",
  "item19_summary": "string — 2–3 sentences on whether Item 19 disclosures exist and what is publicly known",
  "item19_class": "clear | warning | flagged",
  "item19_severity": "Clear | Watch | Critical",
  "item19_pill": "pill-watch | pill-high | pill-critical",
  "item20_summary": "string — 2–3 sentences on franchisee turnover based on publicly known data",
  "item20_class": "clear | warning | flagged",
  "item20_severity": "Clear | Watch | Critical",
  "item20_pill": "pill-watch | pill-high | pill-critical",
  "seo_paragraph_1": "string — 2–3 natural sentences introducing the franchise for SEO. Mention full franchise name naturally.",
  "seo_paragraph_2": "string — 2–3 sentences on investment, buyer profile, operations.",
  "seo_paragraph_3": "string — 2–3 sentences on what prospective buyers commonly ask about this franchise.",
  "cost_faq_answer": "string — plain-English answer to 'how much does this franchise cost'",
  "litigation_faq_answer": "string — plain-English answer to litigation history question",
  "earnings_faq_answer": "string — plain-English answer to Item 19 earnings question",
  "similar_franchises": [
    {"name": "string", "slug": "string", "risk_label": "Low | Medium | High", "pill_class": "pill-watch | pill-high | pill-critical"}
  ]
}
```
```

---

## API CALL STRUCTURE (for Cloudflare Worker or direct integration)

```javascript
const response = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-api-key": YOUR_API_KEY,
    "anthropic-version": "2023-06-01"
  },
  body: JSON.stringify({
    model: "claude-opus-4-5",          // Use Opus for full reports (best analysis quality)
    // model: "claude-haiku-4-5-20251001", // Use Haiku for database page generation (cheaper)
    max_tokens: 4096,
    system: SYSTEM_PROMPT,             // From above
    messages: [
      {
        role: "user",
        content: [
          {
            type: "document",
            source: {
              type: "base64",
              media_type: "application/pdf",
              data: BASE64_ENCODED_PDF  // From Tally file upload
            }
          },
          {
            type: "text",
            text: USER_PROMPT           // From above, with [FRANCHISE_NAME] replaced
          }
        ]
      }
    ]
  })
});

const data = await response.json();
const analysisJSON = JSON.parse(data.content[0].text);
```

---

## COST PER REPORT ESTIMATE

| Report Type       | Model Used    | Est. Input Tokens | Est. Output Tokens | Est. Cost  |
|-------------------|---------------|-------------------|--------------------|------------|
| Full Report ($497)| claude-opus-4-5| ~50,000 (400pg PDF)| ~2,000            | ~$0.80     |
| Quick Scan ($97)  | claude-opus-4-5| ~50,000            | ~800              | ~$0.55     |
| DB Page Gen       | claude-haiku  | ~500               | ~800              | ~$0.003    |

Margin at scale is effectively 100% minus API costs.
At 50 full reports/month: ~$40 in API costs against $24,850 revenue.
