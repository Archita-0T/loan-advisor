# LoanAdvisor AI — READMEe

## Architecture

A single `index.html` file containing all HTML, CSS, and JavaScript inline. No build step, no external framework, no npm dependencies.

### Component Overview

```
index.html
├── <style>        — All CSS (variables, layout, chat bubbles, tables, voice/lang UI, responsive)
├── <body>         — App shell (header + language selector, disclaimer, message list, voice status bar, input bar + mic button)
└── <script>
    ├── PRODUCTS[]              Hardcoded loan catalog (6 products incl. Top-up Loan)
    ├── LANG_NAMES / LANG_BCP47 Language display names and BCP-47 locale codes (EN/HI/TA/MR)
    ├── calcEMI()               Reducing-balance EMI formula
    ├── getEligibleProducts()   FOIR + employment + amount + requiresExistingLoan filter
    ├── sortByRisk()            Sorts eligible products by user risk appetite
    ├── tenureOptions()         Returns 3 spread tenure points per product
    ├── extractAmount()         Parses ₹ amounts from natural text (lakh/crore/k/plain)
    ├── extractEmploymentType() Regex classifier → salaried / self-employed / business
    ├── extractRiskProfile()    Regex classifier → conservative / moderate / aggressive
    ├── extractTenure()         Parses months/years from text
    ├── updateProfileFromText() Orchestrates all field extraction from each user message
    ├── buildSystemContext()    Assembles system prompt + language instruction + risk guidance + profile + EMI tables
    ├── callLLM()               Fetch wrapper for the LLM API (multi-shape response parsing, one retry on empty)
    ├── speakText()             Voice output via SpeechSynthesis API
    ├── initRecognition()       Initialises SpeechRecognition with current language locale
    ├── startListening()        Activates mic; transcribes speech → sendMessage()
    ├── stopListening()         Stops recognition; resets mic button state
    ├── chipsForContext()       Returns context-aware quick-reply chips per conversation stage
    ├── buildTenureTable()      Renders HTML table comparing 3 tenure options for a product
    ├── generateSummaryHTML()   Builds standalone downloadable loan summary HTML
    ├── downloadSummary()       Triggers browser download of the summary
    ├── sendMessage()           Orchestrates user input → profile update → LLM call → render
    └── DOM helpers             addMessage() (with speak button), addTypingIndicator(), markdownToHtml(), escapeHtml()
```

### Data Flow

1. User types or speaks a message → `sendMessage()` is called (voice input transcribes via SpeechRecognition first).
2. `updateProfileFromText()` scans the message with regex patterns and populates `userProfile` fields: `loanAmount`, `income`, `existingEMI`, `employmentType`, `purpose`, `preferredTenure`, and `riskProfile`.
3. `getEligibleProducts()` filters the catalog against the current profile using income, FOIR, employment type, loan amount range, and the `requiresExistingLoan` flag (for Top-up Loan).
4. `sortByRisk()` reorders the eligible products based on the user's risk appetite (conservative → lowest-rate first; aggressive → growth products first).
5. `buildSystemContext()` assembles the full system prompt including: language instruction (if non-English selected), risk guidance, current `userProfile` JSON, and pre-calculated EMI tables for 3 tenure options per eligible product.
6. `callLLM()` sends `[system context + full chat history]` as a single `prompt` string to the LLM wrapper.
7. The reply is rendered as a chat bubble with a 🔊 Read aloud button, context-aware quick-reply chips, and a Download Summary banner once a recommendation has been made.

---

## Prompt Strategy

### System Prompt Design

The system prompt is prepended on **every** LLM call so the model always has:
- Role definition ("responsible AI Loan Advisor")
- Hard rules (only catalog products, always show EMI, never guarantee approval)
- **Language instruction** — if a non-English language is selected, the prompt includes `"Respond entirely in [Language]"` so the LLM replies in Hindi, Tamil, or Marathi
- **Risk guidance** — if `riskProfile` is set, the prompt includes a note to prioritise products matching the user's risk appetite
- Current `userProfile` JSON (all 7 fields including `riskProfile`)
- Pre-calculated EMI tables for 3 tenure options per eligible product, already sorted by risk preference

This "context injection" approach means the LLM never needs to calculate EMIs itself — it reads pre-computed numbers and presents them naturally, eliminating arithmetic hallucination.

### Conversation Strategy

The system prompt instructs the model to ask for one missing field at a time. The fields are collected in order: loan amount → purpose → income → existing EMI → employment type → risk profile → preferred tenure. Combined with client-side `updateProfileFromText()`, this creates a hybrid approach:
- **Fast extraction:** common patterns (₹5 lakh, salaried, no EMI, conservative) are parsed client-side instantly via regex.
- **Graceful fallback:** if extraction fails, the LLM asks the right follow-up question conversationally.
- **Quick-reply chips** adapt to each stage of the conversation, offering the most likely next inputs as one-tap options.

### Hallucination Safeguards

- EMI, total interest, and total repayment are calculated deterministically in JavaScript — the LLM only reads and narrates these values, never computes them.
- The system prompt strictly limits the LLM to the 6-product catalog. It cannot invent new products or rates.
- The disclaimer "Final approval is subject to underwriting and document verification" is mandated in every response by the system prompt.

### History Management

The full `chatHistory` array is serialised as `User: …\nAssistant: …` lines and appended after the system prompt. This gives the model complete conversational context without needing a server-side session.

---

## Mock Data Assumptions

| Product | Key Assumption |
|---|---|
| Personal Loan | 10.5% flat annual rate; reducing balance; salaried or self-employed |
| Salary Advance | Short-term bridge; salaried only; capped at ₹1 lakh |
| BNPL | 0% interest used for EMI calculation (interest-free period); 24% kicks in after 3 months but this is noted in UI, not re-calculated per month |
| SME Business Loan | Requires self-employed or business owner; higher minimum income (₹50k) |
| Secured Loan | Lowest rate (8%) due to collateral; highest max tenure (120 months) |
| Top-up Loan | 11% p.a.; available only to borrowers with an existing loan (existingEMI > 0); salaried or self-employed; ₹50k–₹15L |

FOIR cap is 50% — standard RBI guideline for retail lending. The check uses the mid-tenure EMI of each product to decide eligibility; final FOIR may vary by chosen tenure.

---

## Test Cases

### Test 1 — Salaried, medium income, personal loan
- **Profile:** Loan: ₹3L | Income: ₹40,000 | Existing EMI: ₹0 | Employment: Salaried
- **Expected:** Personal Loan and Secured Loan eligible; Salary Advance ineligible (amount too high); SME ineligible (employment type)

### Test 2 — Business owner, high loan amount
- **Profile:** Loan: ₹10L | Income: ₹80,000 | Existing EMI: ₹5,000 | Employment: Business
- **Expected:** SME Business Loan and Secured Loan eligible; Personal Loan ineligible (employment); Salary Advance ineligible

### Test 3 — Low income, small BNPL purchase
- **Profile:** Loan: ₹20,000 | Income: ₹12,000 | Existing EMI: ₹0 | Employment: Salaried
- **Expected:** BNPL eligible only (min income ₹10k met); Personal Loan ineligible (min income ₹25k); Salary Advance eligible if FOIR passes

### Test 4 — High existing EMI causes FOIR failure
- **Profile:** Loan: ₹5L | Income: ₹35,000 | Existing EMI: ₹15,000 | Employment: Salaried
- **Expected:** Most products fail FOIR check (existing EMI already ~43% of income); only Secured Loan with long tenure might pass

### Test 5 — Self-employed, gold loan
- **Profile:** Loan: ₹8L | Income: ₹60,000 | Existing EMI: ₹8,000 | Employment: Self-employed
- **Expected:** Personal Loan, SME Business Loan, and Secured Loan all eligible; BNPL eligible but likely under-recommended for this amount; Top-up Loan eligible (existing EMI > 0)

### Test 6 — Risk profile influences recommendation order
- **Profile:** Loan: ₹3L | Income: ₹50,000 | Existing EMI: ₹0 | Employment: Salaried | Risk: Conservative
- **Expected:** Secured Loan ranked first (lowest rate, most stable), Personal Loan second; aggressive products deprioritised

### Test 7 — Top-up loan with no existing loan
- **Profile:** Loan: ₹2L | Income: ₹40,000 | Existing EMI: ₹0 | Employment: Salaried
- **Expected:** Top-up Loan excluded (requiresExistingLoan = true, existingEMI = 0); Personal Loan and Secured Loan eligible

---

## Security & Privacy

### Prototype vs. Production

This prototype runs entirely client-side with no backend. Below is an explicit description of how security and privacy would be handled in a real production implementation.

### User-Level Access Control & Data Isolation

In production, each session would be tied to an authenticated user identity:

- **Authentication:** Users log in via OAuth 2.0 / SSO. A short-lived JWT is issued on successful login and sent as a Bearer token on every API request.
- **Session scoping:** The chat history and `userProfile` object are stored server-side, keyed by `userId`. No user can access another user's profile or conversation — the backend enforces this by validating the JWT on every request before returning or persisting data.
- **Simulated equivalent in this prototype:** `userProfile` and `chatHistory` live in JavaScript memory, scoped to the browser tab. Closing the tab destroys all data — no cross-user leakage is possible in this single-user prototype.

### Sensitive Data Handling

Financial data (income, EMI, loan amount) is inherently sensitive:

| Data | Prototype Handling | Production Handling |
|---|---|---|
| Chat history | In-memory JS array, lost on refresh | Encrypted at rest in DB; retained per retention policy |
| User profile (income, EMI) | In-memory only | Stored in user-scoped encrypted record; never logged in plaintext |
| LLM prompt (contains profile) | Sent to LLM API directly | Sent via backend proxy; PII masked or tokenised before leaving org boundary |
| API token | Embedded in client JS (visible in DevTools) | Stored as server-side environment variable; never exposed to client |

### API Token & Backend Proxy

The LLM API token is currently hardcoded in client-side JavaScript — this is acceptable for a prototype but a critical issue in production:

- In production, the frontend sends requests to an **internal backend proxy** (e.g. `/api/llm/query`).
- The proxy validates the user's JWT, then forwards the request to the LLM API using a server-side secret token.
- The LLM API token is never sent to the browser.

### Data Minimisation

- Core eligibility fields collected: `loanAmount`, `income`, `existingEMI`, `employmentType`.
- Advisory fields collected: `riskProfile` (influences product ordering), `purpose` and `preferredTenure` (improve recommendation quality).
- No PII (name, PAN, Aadhaar, phone) is collected or sent to the LLM in this prototype.

### Audit Trail

In production, every LLM call (prompt + response) would be logged to an append-only audit store with `userId`, `timestamp`, and a hash of the prompt — enabling post-hoc review of AI recommendations for regulatory compliance.

### Responsible AI Safeguards

- The system prompt explicitly prohibits the LLM from guaranteeing loan approval.
- Every recommendation response includes the disclaimer: *"Final approval is subject to underwriting and document verification."*
- The LLM is grounded to the product catalog — it cannot recommend products that don't exist or fabricate interest rates.
- EMI calculations are done deterministically in JavaScript, not by the LLM, eliminating arithmetic hallucination risk.

---

## Bonus Features Implemented

### Risk Profile
Users are asked whether they prefer a conservative, moderate, or aggressive approach after employment type is collected. This field is:
- Extracted from text via regex (`conservative`, `safe`, `low risk`, `aggressive`, `growth`, etc.)
- Injected into the system prompt so the LLM factors it into its recommendation narrative
- Used client-side to sort eligible products before injecting them into the prompt (conservative → secured/low-rate products first; aggressive → higher-growth products first)

### Top-up Loan (6th Product)
Added as a distinct product at 11% p.a. with a `requiresExistingLoan: true` flag. The eligibility filter excludes it when `existingEMI === 0`, accurately simulating the real-world requirement that top-up loans are only offered to existing borrowers.

### Multilingual Support Simulation
A language dropdown in the header allows switching between English, Hindi, Tamil, and Marathi. On switch, the system prompt instructs the LLM to respond entirely in the selected language. The Web Speech API locale is also updated accordingly for accurate voice recognition and synthesis.

### Voice Input (SpeechRecognition API)
A microphone button in the input bar activates the browser's built-in Speech Recognition (Chrome/Edge). Spoken input is transcribed and sent as a message. The recognition locale matches the selected language. Falls back gracefully (button disabled) in unsupported browsers.

### Voice Output (SpeechSynthesis API)
Every bot message has a "🔊 Read aloud" button that uses the browser's Speech Synthesis API to read the response in the selected language. An animated indicator shows while speaking; clicking again cancels playback.

---

## Known Limitations

1. **Profile extraction is regex-based**, not AI-powered. Complex phrasing ("my take-home after tax is around fifty thousand") may not parse correctly. The LLM will catch this conversationally, but `userProfile` won't update until a recognisable pattern appears.

2. **BNPL rate simplification.** EMI is calculated at 0% for the full tenure. The 24% post-3-month rate is mentioned in context but not factored into the comparison table — a real implementation would split the schedule.

3. **No authentication or rate limiting.** The API token is embedded in client-side JS and visible in browser DevTools. For production, proxy through a backend.

4. **No persistent session.** Refreshing the page resets `userProfile` and `chatHistory`. A production build would store state in a backend session or localStorage.

5. **Multilingual support is LLM-dependent.** The language selector instructs the LLM to respond in Hindi, Tamil, or Marathi, and the voice recognition/synthesis locale is updated accordingly. However, the static UI elements (header, disclaimer, chip labels) remain in English. Full localisation of UI text would require a translation layer for each supported language.

6. **Tenure comparison is capped at 3 options** per product (min/mid/max). Finer-grained comparisons would need a separate UI control.

7. **No document checklist or application handoff.** The chatbot advises but cannot initiate an application or verify documents — this would require integration with a loan origination system.