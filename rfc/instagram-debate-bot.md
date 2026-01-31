# RFC: Instagram Debate-Bot

**Status:** Draft  
**Author:** fparticle team  
**Created:** 2026-01-31  
**Last Updated:** 2026-01-31

---

## 1. Introduction

### 1.1 What is the Instagram Debate-Bot?

The Instagram Debate-Bot is a lightweight, stateless automation tool that engages with Instagram post commenters by presenting counter-arguments drawn exclusively from a single, locally-stored numbered Markdown article. The bot monitors comments on designated Instagram posts, identifies claims or statements that can be debated, and responds with relevant citations and arguments from the article.

### 1.2 Purpose

This tool is designed to:
- **Educate:** Share evidence-based arguments from curated articles with Instagram audiences
- **Engage:** Foster meaningful debate and discussion in comment sections
- **Scale:** Automate the process of responding to common misconceptions or opposing viewpoints
- **Maintain Quality:** Ensure all responses are grounded in a single, vetted knowledge source

The bot is intended for accounts that share educational or advocacy content and want to systematically engage with commenters using well-researched arguments.

---

## 2. Goals

### 2.1 Primary Goals

1. **Accurate Citation:** All bot responses must cite specific sections from the source article (e.g., "§1.1.1", "§2.3") and include the exact argument or evidence
2. **Relevance Filtering:** Only respond to comments that present debatable claims related to the article's subject matter
3. **Human-Like Tone:** Generate responses that are conversational, respectful, and non-repetitive
4. **Full Transparency:** Clearly identify the bot as automated and provide links to the full article
5. **Zero Hallucination:** Never invent facts, citations, or arguments not present in the source article

### 2.2 Secondary Goals

1. **Minimal Infrastructure:** Run without persistent databases or vector stores
2. **Audit Trail:** Maintain human-readable logs of all interactions for review and improvement
3. **Human Oversight:** Enable manual review and approval of responses before posting (optional mode)
4. **Graceful Degradation:** Handle API rate limits, token limits, and edge cases without crashing

---

## 3. Hard Constraints

These constraints are **non-negotiable** and define the architecture:

1. **NO DATABASE:** The system must not use any persistent database (SQL, NoSQL, or otherwise)
2. **NO VECTOR STORE:** No embeddings, no semantic search, no vector databases (Pinecone, Chroma, FAISS, etc.)
3. **FULL ARTICLE FEED:** The entire source article must be fed to the LLM on every run, along with all relevant comment context
4. **SINGLE SOURCE:** Only one numbered Markdown article serves as the knowledge base per deployment
5. **STATELESS OPERATION:** Each run of the bot must be independent and self-contained (except for audit logs)
6. **FILE-BASED STATE:** Use simple JSON files on disk for tracking comments, pending responses, and audit logs

---

## 4. Design Principles

### 4.1 KISS (Keep It Simple, Stupid)

- Use the simplest possible solution for each component
- Prefer plain text files and JSON over complex data structures
- Avoid premature optimization
- Minimize dependencies

### 4.2 DRY (Don't Repeat Yourself)

- Template all prompt generation
- Reuse comment processing logic
- Centralize configuration
- Extract common validation rules

### 4.3 Explicit Over Implicit

- All behavior must be configurable and documented
- No "magic" defaults or hidden logic
- Clear error messages with actionable suggestions

---

## 5. Architecture Overview

### 5.1 High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Instagram Webhook                        │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────────┐
│                   Webhook Receiver (Flask/FastAPI)            │
│  - Verify webhook signature                                   │
│  - Extract comment data                                       │
│  - Write to pending_comments.json                             │
└───────────────┬───────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────────┐
│                  Main Processing Script                        │
│  1. Load article from articles/ directory                     │
│  2. Load pending_comments.json                                │
│  3. For each comment:                                         │
│     - Load full discussion context                            │
│     - Build LLM prompt with article + context                 │
│     - Request response from LLM                               │
│     - Validate response (citations, tone, relevance)          │
│     - Save to audit_log.json or no_match_log.json            │
│  4. Post approved responses to Instagram                      │
└───────────────┬───────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────────┐
│                    File-Based State                            │
│  - pending_comments.json (comments awaiting processing)       │
│  - audit_log.json (all generated responses + metadata)        │
│  - no_match_log.json (comments that didn't warrant response)  │
│  - posted_ids.txt (simple list of already-responded IDs)      │
└───────────────────────────────────────────────────────────────┘
```

### 5.2 Directory Structure

```
/
├── articles/
│   └── arguments-against-the-big-three.md  # Source article
├── state/
│   ├── pending_comments.json
│   ├── audit_log.json
│   ├── no_match_log.json
│   └── posted_ids.txt
├── src/
│   ├── webhook_receiver.py      # Webhook endpoint
│   ├── processor.py              # Main processing loop
│   ├── instagram_api.py          # Instagram Graph API wrapper
│   ├── llm_client.py             # LLM API wrapper
│   ├── validator.py              # Response validation
│   └── config.py                 # Configuration
├── templates/
│   ├── debate_prompt.txt         # LLM prompt template
│   └── match_check_prompt.txt    # Relevance check template
├── tests/
│   └── ...
├── .env.example
├── requirements.txt
└── README.md
```

---

## 6. Step-by-Step Processing Pipeline

### 6.1 Webhook Reception (Real-Time)

**Trigger:** Instagram sends a POST request when a new comment is created

1. **Verify Signature:**
   - Extract `X-Hub-Signature-256` header
   - Compute HMAC-SHA256 of request body using `APP_SECRET`
   - Compare computed signature with header value
   - Reject if mismatch

2. **Parse Payload:**
   - Extract: `comment_id`, `post_id`, `comment_text`, `user_id`, `username`, `timestamp`
   - Validate required fields are present

3. **Check Duplicates:**
   - Read `posted_ids.txt`
   - If `comment_id` exists, ignore (already processed)

4. **Save to Pending:**
   - Append comment data to `pending_comments.json`
   - Format: `{"comment_id": "...", "post_id": "...", "text": "...", "user": "...", "timestamp": "..."}`

5. **Respond 200 OK:**
   - Immediately return success to Instagram (required within 5 seconds)

### 6.2 Main Processing Loop (Batch/Scheduled)

**Trigger:** Cron job every 5 minutes, or manual invocation

1. **Load Article:**
   - Read the Markdown article from `articles/` directory
   - Parse to extract numbered sections (§1.1, §1.1.1, etc.)
   - Store full text in memory for this run

2. **Load Pending Comments:**
   - Read `pending_comments.json`
   - If empty, exit gracefully
   - Otherwise, process each comment sequentially

3. **For Each Comment:**

   **3.1 Fetch Thread Context:**
   - Use Instagram Graph API to fetch:
     - Original post caption/text
     - Parent comment (if this is a reply)
     - Previous bot responses in the same thread (if any)
   - Build conversation context string

   **3.2 Relevance Check (Optional LLM Call):**
   - Prompt LLM: "Does this comment present a claim or question related to [article topic]?"
   - If NO → Log to `no_match_log.json`, continue to next comment
   - If YES → Proceed to response generation

   **3.3 Generate Response:**
   - Build prompt using template (see §7)
   - Include: Full article, comment text, thread context
   - Call LLM API (e.g., OpenAI GPT-4, Anthropic Claude)
   - Parse response

   **3.4 Validate Response:**
   - Check all citations exist in article (e.g., "§1.1.1" is a valid section)
   - Verify no hallucinated facts
   - Check character length (Instagram limit: 2,200 chars)
   - Ensure respectful tone (no profanity, insults)
   - If validation fails → Log error, mark comment as failed

   **3.5 Store Result:**
   - Append to `audit_log.json` with full metadata:
     ```json
     {
       "comment_id": "...",
       "comment_text": "...",
       "generated_response": "...",
       "citations_used": ["§1.1.1", "§2.3"],
       "token_count": 450,
       "timestamp": "...",
       "status": "approved" | "rejected" | "pending_review"
     }
     ```

4. **Post Responses:**
   - If auto-post enabled:
     - For each approved response, call Instagram API to post comment
     - Add `comment_id` to `posted_ids.txt`
   - If manual review mode:
     - Wait for human approval (external process/dashboard)

5. **Cleanup:**
   - Remove processed comments from `pending_comments.json`
   - Rotate logs if they exceed size limit

### 6.3 Human Review Workflow (Optional)

1. Dashboard displays audit_log entries with `status: "pending_review"`
2. Human reviews generated response
3. Human approves or rejects
4. If approved → Post to Instagram, update status
5. If rejected → Log reason, mark as rejected

---

## 7. Webhook Verification & Subscription

### 7.1 Initial Setup (One-Time)

Instagram requires a verification step before sending webhooks:

1. **Create Webhook Endpoint:**
   - URL: `https://yourdomain.com/webhook/instagram`
   - Must be publicly accessible and use HTTPS

2. **Implement Verification Handler:**
   ```python
   @app.get("/webhook/instagram")
   def verify_webhook(request):
       mode = request.args.get('hub.mode')
       token = request.args.get('hub.verify_token')
       challenge = request.args.get('hub.challenge')
       
       if mode == 'subscribe' and token == VERIFY_TOKEN:
           return challenge, 200
       else:
           return "Forbidden", 403
   ```

3. **Configure in Meta Developer Console:**
   - Go to App Dashboard → Products → Webhooks
   - Subscribe to `comments` field for your Instagram account
   - Provide callback URL and verify token
   - Instagram will send GET request to verify

### 7.2 Ongoing Webhook Reception

After verification, Instagram sends POST requests for new comments:

```python
@app.post("/webhook/instagram")
def receive_webhook(request):
    # 1. Verify signature (see §6.1)
    signature = request.headers.get('X-Hub-Signature-256')
    if not verify_signature(request.body, signature, APP_SECRET):
        return "Invalid signature", 403
    
    # 2. Parse payload
    data = request.json()
    for entry in data.get('entry', []):
        for change in entry.get('changes', []):
            if change['field'] == 'comments':
                process_comment(change['value'])
    
    return "OK", 200
```

### 7.3 Subscription Management

- Use Instagram Graph API to manage subscriptions programmatically:
  ```
  POST /{app-id}/subscriptions
  ?object=instagram
  &callback_url={url}
  &fields=comments
  &verify_token={token}
  &access_token={app-token}
  ```

---

## 8. Prompt Templates

### 8.1 Main Debate Response Prompt

**File:** `templates/debate_prompt.txt`

```
You are a debate assistant bot for an Instagram account focused on {{TOPIC}}.

Your task is to generate a polite, evidence-based response to the following Instagram comment.

RULES:
1. ONLY use arguments and evidence from the provided article below
2. Cite specific sections using the §X.Y.Z format (e.g., "§1.1.1")
3. Be conversational and respectful, not academic or robotic
4. If the comment agrees with the article, acknowledge it briefly
5. If the comment disagrees, present counter-evidence from the article
6. Keep response under 2,000 characters
7. Do NOT invent facts or citations not in the article
8. End with a question to encourage further discussion

ARTICLE:
{{FULL_ARTICLE_TEXT}}

---

INSTAGRAM POST CAPTION:
{{POST_CAPTION}}

COMMENT TO RESPOND TO:
User: {{USERNAME}}
Text: {{COMMENT_TEXT}}

{{#if THREAD_CONTEXT}}
PREVIOUS DISCUSSION IN THIS THREAD:
{{THREAD_CONTEXT}}
{{/if}}

---

Generate your response below. Start with a brief acknowledgment of the user's point, then present counter-arguments with citations. Be friendly and curious, not combative.

RESPONSE:
```

### 8.2 Relevance Check Prompt

**File:** `templates/match_check_prompt.txt`

```
You are a filter bot determining if an Instagram comment is relevant to a specific article.

ARTICLE TOPIC:
{{ARTICLE_TITLE}}

ARTICLE SUMMARY:
{{ARTICLE_FIRST_PARAGRAPH}}

COMMENT:
"{{COMMENT_TEXT}}"

QUESTION:
Does this comment present a claim, question, or viewpoint that can be meaningfully addressed using arguments from this article?

Answer ONLY with "YES" or "NO", followed by a one-sentence explanation.

ANSWER:
```

### 8.3 Response Format

All LLM responses should follow this structure:

```
@{{USERNAME}} [Acknowledgment or greeting]

[Main argument with citation, e.g.:]
Research actually shows that {{CLAIM}}. According to §1.1.1, {{EVIDENCE}}.

[Additional point if needed:]
There's also the issue of {{SECOND_CLAIM}}. The data in §2.3 indicates {{EVIDENCE}}.

[Closing question:]
What's your take on {{QUESTION}}?

---
🤖 This is an automated response. Full article: {{ARTICLE_LINK}}
```

---

## 9. JSON Schemas

### 9.1 `pending_comments.json`

Stores comments awaiting processing.

```json
{
  "version": "1.0",
  "comments": [
    {
      "comment_id": "18123456789",
      "post_id": "17987654321",
      "username": "fitness_enthusiast",
      "user_id": "12345",
      "text": "But everyone says squats are the king of exercises!",
      "timestamp": "2026-01-31T04:15:00Z",
      "received_at": "2026-01-31T04:15:05Z"
    }
  ]
}
```

**Schema:**
- `version` (string): Schema version for future compatibility
- `comments` (array): List of comment objects
  - `comment_id` (string, required): Instagram comment ID
  - `post_id` (string, required): Instagram post ID
  - `username` (string, required): Commenter's username
  - `user_id` (string, required): Commenter's user ID
  - `text` (string, required): Comment text content
  - `timestamp` (string, required): When comment was created (ISO 8601)
  - `received_at` (string, required): When webhook was received (ISO 8601)

### 9.2 `audit_log.json`

Stores all generated responses with metadata.

```json
{
  "version": "1.0",
  "entries": [
    {
      "id": "log_001",
      "comment_id": "18123456789",
      "post_id": "17987654321",
      "username": "fitness_enthusiast",
      "comment_text": "But everyone says squats are the king of exercises!",
      "generated_response": "@fitness_enthusiast I hear you – squats are often...",
      "citations_used": ["§1.1.1", "§1.2.1", "§1.3.1"],
      "token_count_in": 3500,
      "token_count_out": 450,
      "model": "gpt-4-turbo",
      "status": "approved",
      "posted": true,
      "posted_at": "2026-01-31T04:20:00Z",
      "reviewed_by": "human_operator_1",
      "timestamp": "2026-01-31T04:18:00Z",
      "validation_passed": true,
      "validation_errors": []
    }
  ]
}
```

**Schema:**
- `version` (string): Schema version
- `entries` (array): List of log entries
  - `id` (string, required): Unique log entry ID
  - `comment_id` (string, required): Instagram comment ID
  - `post_id` (string, required): Instagram post ID
  - `username` (string, required): Commenter username
  - `comment_text` (string, required): Original comment
  - `generated_response` (string, required): Bot's generated response
  - `citations_used` (array of strings): Section references used (e.g., ["§1.1.1"])
  - `token_count_in` (integer): Input tokens to LLM
  - `token_count_out` (integer): Output tokens from LLM
  - `model` (string): LLM model identifier
  - `status` (enum): `"approved"`, `"rejected"`, `"pending_review"`
  - `posted` (boolean): Whether response was posted to Instagram
  - `posted_at` (string, nullable): When posted (ISO 8601)
  - `reviewed_by` (string, nullable): Human reviewer ID (if manual review)
  - `timestamp` (string, required): When log entry was created (ISO 8601)
  - `validation_passed` (boolean): Whether validation checks passed
  - `validation_errors` (array of strings): List of validation error messages

### 9.3 `no_match_log.json`

Stores comments deemed irrelevant or unmatchable.

```json
{
  "version": "1.0",
  "entries": [
    {
      "id": "nomatch_001",
      "comment_id": "18987654321",
      "post_id": "17987654321",
      "username": "random_user",
      "comment_text": "Nice photo!",
      "reason": "Generic praise, no debatable claim",
      "relevance_score": 0.1,
      "timestamp": "2026-01-31T04:16:00Z"
    }
  ]
}
```

**Schema:**
- `version` (string): Schema version
- `entries` (array): List of no-match entries
  - `id` (string, required): Unique entry ID
  - `comment_id` (string, required): Instagram comment ID
  - `post_id` (string, required): Instagram post ID
  - `username` (string, required): Commenter username
  - `comment_text` (string, required): Original comment
  - `reason` (string, required): Human-readable reason for not responding
  - `relevance_score` (float, optional): Numeric relevance score (0.0-1.0)
  - `timestamp` (string, required): When assessed (ISO 8601)

---

## 10. Token Limit & Truncation Policy

### 10.1 Input Token Budget

**Target:** Keep total input under 100,000 tokens per request (for GPT-4 Turbo)

**Allocation:**
- **Article:** ~30,000 tokens (priority: keep full article)
- **System Prompt + Instructions:** ~2,000 tokens
- **Comment + Thread Context:** ~5,000 tokens
- **Buffer:** ~63,000 tokens remaining

### 10.2 Truncation Strategy

If input exceeds token limit (rare, but possible with long articles):

1. **Preserve Article Structure:**
   - Keep article header/introduction (first 1,000 tokens)
   - Keep all section headers and numbers (§X.Y.Z)
   - Truncate body text of least-relevant sections first

2. **Thread Context Truncation:**
   - If thread is long, keep only:
     - Most recent 3 exchanges
     - Original post caption
   - Summarize older context with: "[Previous discussion covered: X, Y, Z]"

3. **Never Truncate:**
   - The comment being responded to
   - Critical citations referenced in recent context

### 10.3 Output Token Limit

- **Instagram Limit:** 2,200 characters
- **Target Length:** 1,200-1,800 characters (comfortable margin)
- **Minimum Length:** 200 characters (ensure substance)

If LLM generates response exceeding 2,000 characters:
1. Log warning in audit log
2. Truncate at last complete sentence before 2,000 char limit
3. Add "[continued in next comment]" if multi-part response is needed

### 10.4 Cost Management

- Track cumulative token usage per day
- Set daily budget limit (e.g., $10/day = ~200,000 tokens for GPT-4)
- If budget exceeded, pause processing and alert operator

---

## 11. Post-Generation Validation Rules

All generated responses must pass these checks before being saved or posted:

### 11.1 Citation Validation

- **Rule:** Every citation (e.g., "§1.1.1") must exist in the source article
- **Check:** Regex match all `§\d+(\.\d+)*` patterns, verify against article structure
- **Failure:** If invalid citation found, reject response and log error

### 11.2 Hallucination Detection

- **Rule:** No facts or studies can be mentioned unless explicitly in article
- **Check:** 
  - Extract all claims from response
  - Verify each claim appears in article text
  - Use fuzzy matching for paraphrased content
- **Failure:** If novel claim detected, reject and flag for review

### 11.3 Length Validation

- **Rule:** Response must be 200-2,000 characters
- **Check:** `len(response) >= 200 and len(response) <= 2000`
- **Failure:** If too short, regenerate with "expand" instruction. If too long, truncate.

### 11.4 Tone Validation

- **Rule:** No profanity, personal attacks, or aggressive language
- **Check:** 
  - Use profanity filter (e.g., `better-profanity` library)
  - Check for negative sentiment indicators
- **Failure:** If detected, reject and log

### 11.5 Format Validation

- **Rule:** Must include:
  - Username mention (@username)
  - At least one citation
  - Bot disclosure footer
- **Check:** Regex match for `@\w+`, `§\d+`, and footer text
- **Failure:** If missing, append missing elements or reject

### 11.6 Duplicate Detection

- **Rule:** Don't post identical responses to different comments
- **Check:** Compare generated response to last 10 responses in audit log
- **Failure:** If >90% similar (Levenshtein distance), regenerate or skip

---

## 12. Implementation Tasks (Prioritized for LLM Code Generation)

### Phase 1: Foundation (Days 1-2)

**Task 1.1: Project Setup**
- **Acceptance Criteria:**
  - [ ] Directory structure created as per §5.2
  - [ ] `requirements.txt` with: Flask/FastAPI, requests, python-dotenv, openai/anthropic
  - [ ] `.env.example` with all config variables documented
  - [ ] README with setup instructions

**Task 1.2: Article Parser**
- **Acceptance Criteria:**
  - [ ] Function: `load_article(filepath) -> dict`
  - [ ] Parses Markdown to extract numbered sections
  - [ ] Returns structure: `{"§1.1": "text...", "§1.1.1": "text...", ...}`
  - [ ] Handles nested numbering up to 4 levels (§X.Y.Z.W)
  - [ ] Unit tests for sample article

**Task 1.3: Config Management**
- **Acceptance Criteria:**
  - [ ] `config.py` loads from environment variables
  - [ ] Validates required vars: `INSTAGRAM_ACCESS_TOKEN`, `LLM_API_KEY`, `APP_SECRET`
  - [ ] Provides sensible defaults for optional vars
  - [ ] Raises clear error if required var missing

### Phase 2: Webhook & State (Days 3-4)

**Task 2.1: Webhook Receiver**
- **Acceptance Criteria:**
  - [ ] Flask/FastAPI endpoint at `/webhook/instagram`
  - [ ] GET handler for verification (returns challenge)
  - [ ] POST handler for comment events
  - [ ] Signature verification using HMAC-SHA256
  - [ ] Writes to `pending_comments.json` (atomic file write)
  - [ ] Returns 200 OK within 5 seconds

**Task 2.2: File-Based State Manager**
- **Acceptance Criteria:**
  - [ ] Functions: `load_pending()`, `save_pending()`, `load_audit_log()`, `append_audit()`, etc.
  - [ ] Uses file locks to prevent race conditions
  - [ ] Creates files with proper schema if they don't exist
  - [ ] Validates JSON schema on read

**Task 2.3: Duplicate Tracking**
- **Acceptance Criteria:**
  - [ ] Maintains `posted_ids.txt` as newline-delimited list
  - [ ] Function: `is_duplicate(comment_id) -> bool`
  - [ ] Prevents double-processing of same comment

### Phase 3: LLM Integration (Days 5-6)

**Task 3.1: LLM Client**
- **Acceptance Criteria:**
  - [ ] Wrapper for OpenAI/Anthropic API
  - [ ] Function: `generate_response(prompt, max_tokens) -> (response_text, token_counts)`
  - [ ] Handles rate limiting with exponential backoff
  - [ ] Logs all API calls with token usage
  - [ ] Configurable model selection

**Task 3.2: Prompt Builder**
- **Acceptance Criteria:**
  - [ ] Loads templates from `templates/` directory
  - [ ] Function: `build_debate_prompt(article, comment, context) -> str`
  - [ ] Performs variable substitution ({{VARIABLE}})
  - [ ] Truncates input if exceeds token budget (per §10.2)
  - [ ] Unit tests with mock data

**Task 3.3: Relevance Filter**
- **Acceptance Criteria:**
  - [ ] Function: `is_relevant(comment_text, article_summary) -> (bool, reason)`
  - [ ] Uses lightweight LLM call (e.g., GPT-3.5-turbo)
  - [ ] Returns boolean + explanation string
  - [ ] Logs all irrelevant comments to `no_match_log.json`

### Phase 4: Validation & Instagram API (Days 7-8)

**Task 4.1: Response Validator**
- **Acceptance Criteria:**
  - [ ] Implements all rules from §11
  - [ ] Function: `validate_response(response, article) -> (bool, errors)`
  - [ ] Returns list of validation errors if any
  - [ ] Unit tests for each validation rule

**Task 4.2: Instagram Graph API Client**
- **Acceptance Criteria:**
  - [ ] Function: `get_post_caption(post_id) -> str`
  - [ ] Function: `get_thread_context(comment_id, depth=3) -> list[dict]`
  - [ ] Function: `post_comment(post_id, text) -> comment_id`
  - [ ] Handles API errors gracefully (rate limits, expired tokens)
  - [ ] Logs all API calls

### Phase 5: Main Processing Loop (Days 9-10)

**Task 5.1: Comment Processor**
- **Acceptance Criteria:**
  - [ ] Main function: `process_pending_comments()`
  - [ ] Implements pipeline from §6.2
  - [ ] Processes comments sequentially
  - [ ] Saves to audit log after each comment
  - [ ] Handles exceptions without crashing

**Task 5.2: Auto-Post Logic**
- **Acceptance Criteria:**
  - [ ] Configurable auto-post vs. manual review mode
  - [ ] If auto-post: Posts immediately after validation
  - [ ] If manual: Sets status to "pending_review"
  - [ ] Updates `posted_ids.txt` after posting

**Task 5.3: Logging & Monitoring**
- **Acceptance Criteria:**
  - [ ] Python logging configured (INFO level to file, ERROR to stderr)
  - [ ] Logs all key events: comment received, response generated, validation passed/failed, posted
  - [ ] Daily log rotation

### Phase 6: Testing & Refinement (Days 11-12)

**Task 6.1: Unit Tests**
- **Acceptance Criteria:**
  - [ ] 80%+ code coverage
  - [ ] Tests for: article parser, prompt builder, validator, state manager
  - [ ] Mock external APIs (LLM, Instagram)

**Task 6.2: Integration Tests**
- **Acceptance Criteria:**
  - [ ] End-to-end test: Webhook → Processing → Validation
  - [ ] Uses test data from `tests/fixtures/`
  - [ ] Verifies audit log entries

**Task 6.3: Deployment Documentation**
- **Acceptance Criteria:**
  - [ ] README with deployment instructions (local, Docker, cloud)
  - [ ] Instructions for Instagram app setup
  - [ ] Environment variable documentation
  - [ ] Troubleshooting guide

---

## 13. Testing Checklist & Acceptance Thresholds

### 13.1 Unit Testing

**Minimum Thresholds:**
- [ ] Code coverage: ≥80%
- [ ] All validation rules tested individually
- [ ] Article parser handles malformed Markdown gracefully
- [ ] State manager handles concurrent access (file locking)

**Test Cases:**
- Article with 100+ numbered sections
- Comment with special characters, emojis, mentions
- Response exceeding character limit
- Invalid citations in generated response
- Duplicate comment IDs

### 13.2 Integration Testing

**Scenarios:**
- [ ] Webhook receives comment → saved to pending → processed → posted
- [ ] Irrelevant comment filtered out and logged
- [ ] Response fails validation → logged as rejected, not posted
- [ ] Manual review mode: response marked as pending_review
- [ ] Duplicate comment ignored

### 13.3 End-to-End Testing

**Smoke Test (Use Test Instagram Account):**
1. Deploy bot to staging environment
2. Post test comment on test Instagram post
3. Verify webhook received within 10 seconds
4. Manually trigger processor
5. Verify response generated and validated
6. Check Instagram for posted reply
7. Verify audit log entry created

**Acceptance Criteria:**
- [ ] Full cycle completes in <60 seconds
- [ ] Response contains valid citations
- [ ] No errors in logs

### 13.4 Performance Testing

**Benchmarks:**
- [ ] Webhook response time: <2 seconds (target: <1 second)
- [ ] Process 10 comments: <5 minutes (with LLM latency)
- [ ] Token usage per comment: <10,000 tokens average
- [ ] File operations: <100ms per read/write

### 13.5 Security Testing

**Checklist:**
- [ ] Webhook signature verification prevents spoofing
- [ ] Environment variables not hardcoded or logged
- [ ] No PII stored in logs (mask usernames in debug logs)
- [ ] Rate limiting on webhook endpoint (e.g., 100 req/min)

---

## 14. Dashboard & Human Workflows

### 14.1 Review Dashboard (Optional, Phase 2)

**Purpose:** Allow humans to review, approve, or reject generated responses.

**Features:**
- Display pending responses (status: "pending_review")
- Show:
  - Original comment
  - Generated response
  - Citations used
  - Validation status
- Actions:
  - Approve (posts to Instagram)
  - Reject (logs reason, doesn't post)
  - Edit (modify response, then approve)

**Tech Stack:** Simple web UI (Flask + Bootstrap, or React)

### 14.2 Manual Workflows

**Daily Review (Recommended):**
1. Check `audit_log.json` for rejected responses
2. Review `no_match_log.json` to adjust relevance thresholds
3. Monitor token usage and costs
4. Read sample of posted responses for quality

**Weekly Review:**
1. Analyze response patterns (which citations used most)
2. Update article if common counter-arguments arise
3. Refine prompt templates based on tone/style feedback

**Monthly Review:**
1. Evaluate bot effectiveness (engagement metrics on Instagram)
2. Consider expanding to multiple articles
3. Update validation rules based on edge cases

---

## 15. Security & Operational Notes

### 15.1 Security Best Practices

1. **Secrets Management:**
   - Store `APP_SECRET`, `ACCESS_TOKEN`, `LLM_API_KEY` in environment variables
   - Never commit secrets to version control
   - Use secret management service in production (e.g., AWS Secrets Manager)

2. **Webhook Security:**
   - Always verify HMAC signature
   - Use HTTPS only (Instagram requires it)
   - Rate limit webhook endpoint to prevent abuse

3. **API Token Security:**
   - Use short-lived Instagram access tokens (refresh every 60 days)
   - Rotate tokens regularly
   - Monitor for unusual API usage (could indicate token compromise)

4. **Data Privacy:**
   - Don't store unnecessary user data (only comment text, no user profiles)
   - Mask or anonymize usernames in debug logs
   - Comply with Instagram's Platform Terms and Data Policy

### 15.2 Operational Considerations

**Rate Limits:**
- Instagram Graph API: 200 calls per hour per user
- LLM APIs: Varies by provider (track usage to avoid overages)
- Implement exponential backoff for all API calls

**Error Handling:**
- Webhook failures: Instagram retries for 24 hours
- LLM API failures: Retry up to 3 times, then log as failed
- Instagram posting failures: Log and alert, but don't retry (avoid spam)

**Monitoring:**
- Set up alerts for:
  - Webhook endpoint downtime
  - API errors (>5% error rate)
  - Token usage exceeding budget
  - Processing queue growing (>100 pending comments)

**Backup & Recovery:**
- Regularly backup `state/` directory
- Keep audit logs for at least 90 days
- Document recovery procedures for data loss

### 15.3 Compliance & Ethics

**Instagram Policies:**
- Clearly label bot as automated (in bio and responses)
- Don't spam or harass users
- Respect rate limits and API guidelines
- Don't scrape or store user data beyond what's necessary

**Ethical Guidelines:**
- Ensure article is factually accurate and well-sourced
- Don't use bot for misinformation or manipulation
- Provide opt-out mechanism (users can ask bot to stop replying)
- Be transparent about the bot's purpose and limitations

---

## 16. Deliverables for Code Generation

When handing this RFC to an LLM or engineer for implementation, provide:

### 16.1 Required Inputs

1. **This RFC Document:** Complete specification
2. **Sample Article:** The actual Markdown article to be used (e.g., `arguments-against-the-big-three.md`)
3. **Instagram Credentials:**
   - App ID & App Secret
   - Access Token (with `instagram_basic`, `instagram_manage_comments` permissions)
   - Verify Token (for webhook setup)
4. **LLM API Credentials:**
   - Provider (OpenAI, Anthropic, etc.)
   - API key
   - Model name (e.g., `gpt-4-turbo`)

### 16.2 Expected Outputs

1. **Codebase:**
   - Functional Python application matching architecture in §5
   - All files organized per directory structure in §5.2
   - Well-commented code with docstrings

2. **Configuration:**
   - `.env.example` with all required variables
   - `requirements.txt` with pinned versions
   - `README.md` with setup and usage instructions

3. **Tests:**
   - Unit tests for core functions
   - Integration test suite
   - Test fixtures for sample data

4. **Documentation:**
   - API documentation (if dashboard included)
   - Deployment guide (local, Docker, cloud)
   - Troubleshooting guide

5. **Deployment Artifacts:**
   - Dockerfile (optional)
   - `docker-compose.yml` (optional)
   - Cloud deployment config (e.g., `app.yaml` for Google App Engine)

### 16.3 Implementation Instructions for LLM

**Prompt Example:**

```
You are a senior Python engineer. Implement the Instagram Debate-Bot according to the RFC provided.

REQUIREMENTS:
1. Follow the architecture in §5 exactly
2. Implement all tasks from §12 in order
3. Ensure all validation rules from §11 are enforced
4. Write clean, well-documented code with type hints
5. Include error handling for all external API calls
6. Create comprehensive tests achieving 80%+ coverage

CONSTRAINTS:
- Python 3.10+
- Use FastAPI for webhook server
- Use OpenAI's Python library for LLM calls
- No external databases or vector stores
- All state in JSON files

DELIVERABLES:
- Complete codebase
- requirements.txt
- README.md with setup instructions
- .env.example
- Unit tests

Start with Task 1.1 (Project Setup) and proceed sequentially through Phase 6.
After completing each task, verify the acceptance criteria are met before moving on.
```

### 16.4 Validation Checklist for Code Review

Before deploying the generated code, verify:

- [ ] All files from §5.2 directory structure are present
- [ ] Webhook verification handler implemented correctly (§7.1)
- [ ] Webhook signature validation uses HMAC-SHA256 (§7.2)
- [ ] Article parser extracts numbered sections (§12, Task 1.2)
- [ ] All prompt templates present and used (§8)
- [ ] JSON schemas match spec exactly (§9)
- [ ] Token truncation logic implemented (§10)
- [ ] All 6 validation rules enforced (§11)
- [ ] Duplicate detection prevents re-processing (§6.1, step 3)
- [ ] Audit log captures all required metadata (§9.2)
- [ ] Instagram API client handles rate limits (§12, Task 4.2)
- [ ] Unit tests pass with ≥80% coverage (§13.1)
- [ ] End-to-end smoke test passes (§13.3)
- [ ] No secrets hardcoded (§15.1)
- [ ] README includes complete setup instructions

---

## 17. Future Enhancements (Out of Scope for v1)

These features are explicitly **not** included in the initial implementation but may be added later:

1. **Multi-Article Support:** Support multiple source articles, with automatic selection based on post context
2. **Conversation Memory:** Track multi-turn discussions to avoid repetition
3. **Advanced Matching:** Use embeddings for semantic relevance filtering (violates "no vector store" constraint, so requires architecture change)
4. **Automatic Learning:** Analyze which responses get positive engagement and refine prompts
5. **Multi-Language Support:** Detect comment language and respond accordingly
6. **Image Analysis:** Use GPT-4 Vision to analyze images in posts/comments
7. **Batch Posting:** Combine multiple responses into a single comment to reduce API calls
8. **Real-Time Streaming:** Use Server-Sent Events to update dashboard in real-time
9. **A/B Testing:** Test different prompt variations and measure effectiveness
10. **Integration with Other Platforms:** Extend to Facebook, Reddit, Twitter/X

---

## 18. Glossary

- **LLM:** Large Language Model (e.g., GPT-4, Claude)
- **Webhook:** HTTP callback triggered by Instagram when events occur
- **HMAC:** Hash-based Message Authentication Code (for signature verification)
- **Instagram Graph API:** Facebook/Meta's API for Instagram integrations
- **Token:** Unit of text processed by LLMs (roughly 0.75 words)
- **Citation:** Reference to a specific section in the article (e.g., "§1.1.1")
- **Audit Log:** Comprehensive record of all bot actions for review and debugging
- **Relevance Score:** Numeric measure (0.0-1.0) of how related a comment is to the article
- **Truncation:** Shortening of text to fit within token or character limits
- **Validation:** Process of checking generated responses against quality rules before posting

---

## 19. References & Resources

**Instagram Platform Documentation:**
- Webhooks: https://developers.facebook.com/docs/graph-api/webhooks/
- Comments API: https://developers.facebook.com/docs/instagram-api/guides/comments/
- Getting Started: https://developers.facebook.com/docs/instagram-api/getting-started/

**LLM APIs:**
- OpenAI: https://platform.openai.com/docs/
- Anthropic Claude: https://docs.anthropic.com/

**Python Libraries:**
- FastAPI: https://fastapi.tiangolo.com/
- Requests: https://docs.python-requests.org/
- Python-dotenv: https://pypi.org/project/python-dotenv/

**Best Practices:**
- Instagram Platform Terms: https://developers.facebook.com/terms/
- GDPR Compliance: https://gdpr.eu/
- Bot Disclosure Guidelines: https://www.ftc.gov/business-guidance/resources/disclosures-101-social-media-influencers

---

## 20. Revision History

| Version | Date       | Author        | Changes                          |
|---------|------------|---------------|----------------------------------|
| 1.0     | 2026-01-31 | fparticle team| Initial RFC creation             |

---

## 21. Approval & Sign-Off

This RFC is ready for implementation once:

- [ ] Technical review completed by senior engineer
- [ ] Security review completed
- [ ] Instagram app setup and credentials obtained
- [ ] LLM API access and budget approved
- [ ] Sample article finalized and reviewed
- [ ] Legal/compliance review completed (if required)

**Approvers:**
- Tech Lead: _______________________
- Security: _______________________
- Product Owner: _______________________

---

**END OF RFC**

This document provides a complete, implementable specification for the Instagram Debate-Bot. All design decisions are justified, all edge cases are addressed, and all implementation tasks are prioritized with clear acceptance criteria. This RFC is suitable for handoff to a code-generating LLM or a development team for implementation.
