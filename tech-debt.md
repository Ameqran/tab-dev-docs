# Tech Debt

## 2026-04-03

- **Clean Architecture wiring debt (tabular-review-backend, PR #431):** `SubmissionExtractionEventBridge` in the application layer directly depends on infrastructure concrete classes (`SSEManager`, `SSEEventFactory`), which violates the clean architecture rule that application code should not import infrastructure implementations outside wiring/composition.
- **Proposed fix:** introduce application-level SSE ports (publisher/factory interfaces), implement them as infrastructure adapters, and instantiate/inject those adapters only in application wiring (`application/submissions/index.ts`).
