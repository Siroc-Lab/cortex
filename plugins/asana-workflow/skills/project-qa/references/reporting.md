# Reporting

## Confidence Levels

Use exactly one of these for each finding:

| Level | Definition | Required Evidence |
|---|---|---|
| **Confirmed** | Reproduced with evidence | Screenshot, screencast, console error, or network trace showing the issue |
| **Likely** | Strong reasoning, not fully reproduced | Partial evidence + explanation of what couldn't be verified and why |
| **Suspicion** | Noticed during investigation, unverified | Description of what was observed + what would need to happen to verify |

## Report Structure

Every report follows this structure:

### 1. Answer
Direct answer to the operator's question. Lead with this — no preamble.

### 2. Root Cause (if applicable)
What's causing the behavior. Include confidence level. If source code is available, include file:line references.

### 3. Reproduction Steps
Numbered steps anyone can follow:

```
1. Navigate to [URL]
2. Click [element]
3. Enter [value] in [field]
4. Click [button]
5. **Expected:** [what should happen]
6. **Actual:** [what happens instead]
```

Be specific: exact URLs, exact element descriptions, exact inputs. Someone with no context should be able to follow these steps.

### 4. Evidence
Include all captured evidence:

- **Screenshots** — at key moments (before, during, after the issue)
- **Screen recordings** — for transitions, animations, multi-step flows (via `experimentalScreencast`)
- **Console errors** — exact error messages with stack traces
- **Network traces** — failed requests, unexpected responses, timing issues

### 5. Source Context (if source code available)
File and line references explaining the code path that produces the behavior:

```
The disabled state comes from `src/components/CheckoutButton.tsx:34` where
`isProcessing` is set to true on click but never reset when the payment
API returns a 422 error (line 51 catches the error but doesn't call
`setIsProcessing(false)`).
```

### 6. Recommendation
Suggested fix or next steps. Be specific — name the file, the function, the change.

## Output

The report IS the artifact. The skill takes a question as input and produces the report as output. No side effects — no memory persistence, no ticket creation, no file writes. The operator decides what to do with the report.
