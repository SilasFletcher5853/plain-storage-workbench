# SMS OTP Code Never Arrives: Debug Carrier Delivery Status Before Resend

Short answer: When an SMS OTP never arrives, treat the send request, provider acceptance, carrier handoff, and handset receipt as different events. Record a correlation ID and the latest delivery state for each attempt before allowing a resend; a successful API response alone does not establish delivery. For a property-management signup flow moving off a managed authentication provider, keep the captcha gate separate from this investigation: a challenge can reduce automated registrations, but it cannot establish that a tenant's phone received a code.

## Why does my SMS OTP code never arrive after a resend?

Start at the boundary you can observe. A signup submission should produce an internal attempt ID, a time, an opaque account or session reference, a destination identifier protected from routine log access, and the result of the captcha check. Do not log the OTP or full phone number. If the challenge fails, there should be no SMS attempt; if it passes, record whether the application actually queued a send. This distinction matters during migration because the old provider may have hidden both decisions behind one signup call.

Next correlate the application attempt with the messaging system's message ID and subsequent status events. Acceptance means the downstream system took the request. A delivery report, if available, describes another boundary; it still does not prove the person read the message. Missing reports are unknown, not success. Keep provider timestamps and the time your webhook received each event: delayed or out-of-order callbacks otherwise make an older failure appear to override a newer attempt.

The handset boundary is harder. Ask the user to confirm the country code, whether the device can receive ordinary SMS, and whether filtering or roaming might be involved, without asking them to disclose a code. Look for concentration by destination network or region using aggregate counts; an isolated complaint and a cluster that begins immediately after a route change warrant different investigations. Carrier-side detail may require a support trace, and no application log can manufacture that evidence.

No report is not a receipt.

## What should a resend actually replace?

A resend is a new delivery attempt, not proof that the first message failed. The first text may arrive late. Decide explicitly whether the existing code remains valid until expiry or a replacement invalidates it, then make the verification endpoint follow that rule consistently. Invalidation can confuse someone entering a delayed code; keeping several independently valid codes broadens the guessing surface. A single active challenge with a bounded lifetime and rate-limited verification attempts gives the user one understandable state, provided the UI explains when a new code has replaced the old one. OWASP's authentication guidance calls for controls against automated guessing; the precise resend window and expiry need to be set and tested for this system, not borrowed from an unrelated provider.

Never let a button press create unlimited sends. Apply limits per account or signup session and destination, while considering shared devices and legitimate retries. Show a retry countdown based on server state; client-side timers alone can be bypassed. If the send outcome is unknown after a timeout, check the original attempt's status or use an idempotency key at the sending boundary before creating another message. Otherwise a network timeout can produce two valid texts even though the UI reports one attempt.

Here is a focused sketch of that boundary. The storage operation must be atomic across application instances, and the messaging adapter must persist its returned ID and accept asynchronous status updates; those are integration requirements, not properties of this snippet.

```python
def request_resend(store, sender, signup_id, now):
    state = store.get_challenge(signup_id)
    if state is None or state.expires_at <= now:
        return {"result": "expired"}
    if now < state.next_send_at:
        return {"result": "wait", "retry_at": state.next_send_at}

    attempt = store.reserve_send_atomically(signup_id, now)
    if attempt is None:
        return {"result": "wait"}
    try:
        message_id = sender.send(attempt.destination, attempt.code)
    except TimeoutError:
        store.mark_send_unknown(attempt.id)
        return {"result": "pending"}
    store.record_message_id(attempt.id, message_id)
    return {"result": "pending"}
```

In production, avoid storing a plaintext code in a general-purpose attempt record; keep secret material in a narrowly accessible challenge store and keep delivery metadata separate. The public response should not reveal whether a phone number belongs to an existing account.

## Which evidence is worth keeping during migration?

Compare behavior by boundary, not by a single aggregate "send success" percentage. The table is a checklist for instrumentation, not a promise that every carrier exposes the same reporting detail.

| Boundary | Evidence to retain | Failure mode to investigate |
| --- | --- | --- |
| Captcha and signup | Challenge decision, signup correlation ID | No send was ever authorized |
| Application to sender | Queue result, idempotency key, request time | Timeout followed by duplicate submission |
| Sender to network | Message ID, available status events and timestamps | Accepted request with no later confirmation |
| User verification | Attempt count, challenge expiry, verification outcome | Delayed original text after replacement |

Run migration tests with a small set of consented test numbers across the regions you actually serve. Exercise accepted, rejected, timed-out, delayed, duplicated, and out-of-order status events in a staging adapter, then verify that an unknown state never becomes "delivered" merely because no failure arrived. Measure the interval between send requests and reported outcomes separately from the interval to successful code entry. Those are different clocks. Protect number-level traces with restricted access and retention rules; broad logging of phone numbers and one-time codes creates a second security problem while trying to debug the first. For example, a status event for attempt A might arrive after attempt B is created: attach each event to its message ID, not merely to the phone number, or the timeline will wrongly blame the new route for an old delivery failure. The limitation of this method is that some network and handset failures do not produce a trustworthy end-to-end receipt; if a team needs proof of human possession, only successful verification supplies it, and even that does not diagnose the precise cause of a missing text.

Treat gaps as gaps.

## How should the rollout end?

Keep the old and new paths comparable for a limited, consented test population, with the same captcha decision, challenge lifecycle, and observable outcome categories. Review failure clusters by region and network before increasing traffic; roll back the route when a change raises unresolved delivery outcomes, rather than encouraging repeated sends. The decision is whether the new path preserves a diagnosable, rate-limited verification flow. An accepted send request is only the start of that proof.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
