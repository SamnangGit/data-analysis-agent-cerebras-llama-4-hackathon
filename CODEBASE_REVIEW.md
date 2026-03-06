# Codebase Review

## Scope
- Reviewed server (`server/app/**`) and web client (`client/**`) code for architecture, security, reliability, and maintainability.
- Performed a syntax sanity check with `python -m compileall server/app client`.

## Executive Summary
The project has a clear structure (models/controllers/routers/agents) and compiles successfully, but there are several high-priority risks around security, runtime resilience, and database write consistency.

## Priority Findings

### 1) Critical: HTML injection/XSS risk in client chat rendering
**Where**
- `client/app.js`

**What**
- Chat messages are rendered via `innerHTML` for both markdown and non-markdown paths.
- `content` may include server/model-provided text (`data.explanation`) and therefore untrusted strings.

**Why it matters**
- Untrusted content rendered with `innerHTML` can execute scripts in the browser context.

**Recommendation**
- Sanitize content before rendering (e.g., DOMPurify), or render with `textContent` and a safe markdown renderer.
- Avoid direct `innerHTML` for arbitrary strings.

---

### 2) Critical: LLM-generated SQL execution lacks strict safety gate
**Where**
- `server/app/utils/db_ops.py`
- `server/app/agents/sql_agent.py`

**What**
- SQL from the model is executed directly with `connection.execute(text(sql_query))`.
- Validation only checks that first significant line starts with `SELECT`; this is not a robust SQL safety policy.

**Why it matters**
- Prompt-injection or model mistakes can generate harmful/expensive statements.
- A "starts with SELECT" check can be bypassed or still allow dangerous operations depending on dialect and server settings.

**Recommendation**
- Enforce a strict parser-based whitelist (single statement, SELECT-only AST).
- Reject multi-statements and disallow mutation keywords/functions at parser level.
- Execute via read-only DB role and query timeout.

---

### 3) High: Manual ID generation creates race conditions
**Where**
- `server/app/utils/db_ops.py`

**What**
- New IDs are assigned using `get_last_record_ids()` + 1 before insert.

**Why it matters**
- Concurrent writes can generate duplicate IDs and transaction failures.

**Recommendation**
- Use database-native autoincrement/identity/sequence columns.
- Remove manual `last_id + 1` assignment logic.

---

### 4) High: API logging middleware may leak sensitive data and can break on non-standard responses
**Where**
- `server/app/middlewares/api_logger.py`

**What**
- Logs full request/response body to `logs/api.log`.
- Assumes `res_body[0]` exists and decodes it directly.

**Why it matters**
- Sensitive content (tokens, personal data, SQL output) may be persisted in plaintext logs.
- Certain response types (empty/streaming/chunked) can fail with current assumptions.

**Recommendation**
- Redact sensitive fields and truncate payload sizes.
- Handle empty/multi-chunk responses safely.
- Prefer structured logging with severity and correlation IDs.

---

### 5) High: Startup tightly couples API to Telegram bot initialization
**Where**
- `server/app/application_manager.py`
- `server/app/utils/telegram_bot.py`

**What**
- App lifespan always initializes and starts polling Telegram bot.
- Chat IDs are parsed with `int(os.getenv(...))` without fallback validation.

**Why it matters**
- Missing/malformed env vars can crash startup even for API-only use.
- Operationally brittle across environments.

**Recommendation**
- Make Telegram startup optional behind feature flags.
- Validate environment variables and fail gracefully with clear errors.

---

### 6) Medium: Hard-coded frontend API endpoint
**Where**
- `client/app.js`

**What**
- Fetch target is fixed to `http://127.0.0.1:8000/...`.

**Why it matters**
- Breaks deployment portability (staging/prod, reverse proxies, container networks).

**Recommendation**
- Use environment-configurable base URL or relative path.

---

### 7) Medium: Request validation in router is manual and inconsistent
**Where**
- `server/app/routers/api.py`

**What**
- Uses `await request.json()` and ad-hoc checks instead of typed request models.
- Error handling is partly commented out in `/analyse` endpoint.

**Why it matters**
- Inconsistent validation behavior and less maintainable API contracts.

**Recommendation**
- Introduce Pydantic request schemas for `/extract` and `/analyse`.
- Restore/standardize exception handling and response model.

---

## Positive Notes
- Project organization is clear and approachable.
- SQL generation includes a restricted marker and initial output cleaning.
- Compile-time syntax check passes for server modules.

## Suggested Implementation Order
1. Fix XSS rendering path in `client/app.js`.
2. Add strict SQL safety controls + read-only DB role.
3. Replace manual ID generation with DB-managed keys.
4. Harden and redact middleware logging.
5. Decouple Telegram startup from API lifecycle.
6. Move client API base URL to config.
7. Add Pydantic request models and clean router handlers.
