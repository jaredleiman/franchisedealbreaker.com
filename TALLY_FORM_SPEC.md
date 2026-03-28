# FranchiseDealBreaker — Tally Form Specifications

Build two separate Tally forms — one per product tier.
Link each directly from its corresponding Stripe payment confirmation page
(set the Stripe "confirmation page" to the appropriate Tally form URL).

---

## FORM 1: FULL RED FLAG REPORT ($497)
## Tally form slug suggestion: franchisedealbreaker-full-report

### Form Settings
- Title: "FDD Submission — Full Red Flag Report"
- Button label: "Submit My FDD"
- Redirect after submit: Custom thank-you page (see thank-you page spec below)
- Notifications: Email notification to hello@franchisedealbreaker.com on every submission
- Show progress bar: Yes

### Fields (in order)

**Page 1 — Confirmation**

1. STATEMENT BLOCK (not a field — display text)
   Text: "Thank you for your purchase. Please complete this form to submit your FDD for analysis.
   Your Full Red Flag Report will be delivered to your email within 24 hours."

2. FIELD: Buyer First Name
   Type: Short text
   Label: "Your first name"
   Required: Yes

3. FIELD: Buyer Email
   Type: Email
   Label: "Email address to receive your report"
   Required: Yes
   Note: This is where the PDF will be sent. Double-check it.

**Page 2 — Franchise Details**

4. FIELD: Franchise Name
   Type: Short text
   Label: "Name of the franchise you are evaluating"
   Placeholder: "e.g. Anytime Fitness, Subway, Orangetheory"
   Required: Yes

5. FIELD: Are you evaluating a new franchise or a resale?
   Type: Multiple choice (single select)
   Options:
     - New franchise (buying directly from franchisor)
     - Resale (buying from an existing franchisee)
     - Renewal (renewing an existing agreement)
   Required: Yes

6. FIELD: Total investment range you are considering
   Type: Multiple choice (single select)
   Options:
     - Under $100,000
     - $100,000 – $250,000
     - $250,000 – $500,000
     - $500,000 – $1,000,000
     - Over $1,000,000
   Required: Yes

7. FIELD: How far along are you in the process?
   Type: Multiple choice (single select)
   Options:
     - Just started researching
     - Received FDD, reviewing options
     - In active discussions with franchisor
     - Ready to sign soon
   Required: Yes

**Page 3 — Document Upload**

8. FIELD: FDD Upload
   Type: File upload
   Label: "Upload your Franchise Disclosure Document (FDD)"
   Accepted formats: PDF
   Max file size: 50MB
   Required: Yes
   Helper text: "Upload the complete FDD as provided by the franchisor.
   If you received it digitally, attach the PDF. If you have a scanned copy, that works too."

9. FIELD: Anything specific you want us to focus on?
   Type: Long text
   Label: "Specific concerns or questions (optional)"
   Placeholder: "e.g. I'm worried about the territory clause. The broker mentioned the Item 19 numbers look good but I want verification."
   Required: No
   Max characters: 500

**Page 4 — Confirmation**

10. STATEMENT BLOCK
    Text: "You're all set. We'll have your Full Red Flag Report in your inbox within 24 hours.
    Check your spam folder if you don't see it. Questions? Email hello@franchisedealbreaker.com"

---

## FORM 2: QUICK SCAN ($97)
## Tally form slug suggestion: franchisedealbreaker-quick-scan

### Form Settings
- Title: "FDD Submission — Quick Scan"
- Button label: "Submit My FDD"
- Same redirect/notification settings as above

### Fields (simplified — fewer pages)

1. STATEMENT BLOCK: "Thank you for your Quick Scan purchase. Submit your FDD and we'll analyze the 5 critical items within 12 hours."

2. Buyer First Name — Short text, required

3. Buyer Email — Email, required

4. Franchise Name — Short text, required

5. FDD Upload — File upload, PDF, 50MB max, required

6. Specific concerns (optional) — Long text, 300 char max

7. STATEMENT BLOCK: "Submitted. Your Quick Scan report will arrive within 12 hours."

---

## ZAPIER TRIGGER SETUP

In Zapier, use "Tally — New Submission" as your trigger for both forms.
Set up two separate Zaps — one per form/tier.

The key field mappings you'll use in subsequent Zapier steps:

| Tally Field         | Zapier Variable Name (use these downstream) |
|---------------------|---------------------------------------------|
| Buyer First Name    | {{buyer_name}}                              |
| Buyer Email         | {{buyer_email}}                             |
| Franchise Name      | {{franchise_name}}                          |
| FDD Upload URL      | {{fdd_file_url}}                            |
| Specific Concerns   | {{buyer_notes}}                             |
| Investment Range    | {{investment_range}}                        |
| Process Stage       | {{process_stage}}                           |

---

## THANK-YOU PAGE

After Tally form submission, redirect to a static thank-you page hosted on GitHub Pages.
Create `thankyou.html` in your repo root.

Content:
- Headline: "Your FDD has been received."
- Subhead: "Full Red Flag Report — within 24 hours. Quick Scan — within 12 hours."
- Body: "We'll send your report to [email]. Check your spam folder if you don't see it within the window.
  Have a question? Email hello@franchisedealbreaker.com"
- CTA: "← Return to FranchiseDealBreaker"

---

## NOTION QUEUE SETUP

Create a Notion database called "FDD Analysis Queue" with these properties:

| Property          | Type         | Notes                              |
|-------------------|--------------|------------------------------------|
| Franchise Name    | Title        | Auto-filled from Tally             |
| Buyer Name        | Text         |                                    |
| Buyer Email       | Email        |                                    |
| Report Type       | Select       | Full Report / Quick Scan           |
| Status            | Select       | Received / In Analysis / Delivered / Error |
| FDD File URL      | URL          | Tally file URL                     |
| Submitted At      | Date         |                                    |
| Deadline          | Date         | Auto-calculated: +24h or +12h      |
| Report Sent       | Checkbox     |                                    |
| Notes             | Text         | Buyer's optional notes             |

Add a Zapier step after Tally submission to create a new row in this database.
This is your QA dashboard — you can see every order, its status, and its deadline at a glance.
