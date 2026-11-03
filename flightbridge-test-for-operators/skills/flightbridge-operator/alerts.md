# Alerts

An **alert** is FlightBridge telling you that something about an order no longer fits its trip, or
that a vendor answered. A leg moved, was cancelled, or changed its airport, FBO or tail. A traveler
left the leg. A vendor declined. FlightBridge never fixes an order by itself: it raises the alert,
and a person decides. These tools let you see the alerts, clear them, and fix them through the same
booking tools you already use.

## Say it once

Every reply carries `alertSummary` with the company's counts: `open`, `actionable`, `urgent`
(service within 24 hours). When `open` is above zero, say it **once** in the conversation, in one
sentence, and offer to look: "3 alerts need attention, 2 of them today. Want me to go through
them?" Say it again only if the count grows. Never repeat it on every reply.

## Read them: `alerts`

`alerts {}` lists the company's open alerts, as the web's Requests > Alerts page does: everyone in
the company sees every alert. `mine: true` keeps only the trips you may see.

Each alert has:

- `type` in words and the web's `reason` ("Aircraft tail changed from N4455 to N54321");
- `order`: the vendor, confirmation, dates and status;
- `leg` as it is **now**: times, airports, FBO, tail, crew;
- `mismatch`: what no longer fits. For example, "the room is for Nov 10, but the leg now arrives
  Nov 11". Read this back in plain words;
- `urgency`: `now` (under 6 hours), `today`, `soon` (under 3 days) or `later`;
- `resolution`: the options, the **recommended** one first.

`alerts { since }` (a UTC time) also returns `news`: orders confirmed, cancellations confirmed,
cars in position and orders viewed since then. News is good news; it needs no action.

## Fix one: `manage_alert { action: "resolve", alertId }`

Offer the recommended option and the others in a sentence each. Then call `resolve` with the
`option` the user chose (or none, for the recommended one).

- The **first call previews**: it returns the underlying tool's preview, such as the new room
  and its total, or a question in words. Nothing changes. Ask that question exactly.
- Call again with `confirm: true` after the user says yes. Pass `confirmedTotal` when the preview
  had a total.
- On success the order is changed and the alert is cleared. The reply says both.
- An outcome that is not certain, or a refusal, leaves the alert open and says why.

What the options do:

| option | what it does |
|---|---|
| `rebook` | books the same hotel for the leg's new nights, then cancels the old booking |
| `update` | re-sends an FBO notice with the new times or tail, or moves a catering delivery |
| `cancel` | cancels the order; the cancellation terms are read out first |
| `reshop` | shops for alternatives (a new airport, or a vendor that declined); the user chooses, then you book with the usual tool |
| `keep` | leaves the order as it is and clears the alert |
| `tell` | nothing to run here: say what to do in FlightBridge, with the link |

A cancel that may carry a penalty stops and returns the terms. Read them to the user, and pass
`confirmTerms: true` only after they accept them.

`resolve_all` fixes every alert whose recommended option needs no choice (a re-send, a cancel at no
cost, a same-hotel rebook) as **one** plan: preview it, then `confirm: true`. It stops at the
first outcome that is not certain.

## Clear without fixing: `manage_alert { action: "dismiss", alertIds }`

This is the web's Dismiss. When an alert could be fixed, the tool asks once whether to clear it
anyway. `permanent: true` is the web's "Don't show again": FlightBridge then never raises that
kind of alert on that order again. Say that before you use it. Some companies do not allow it.

## Standing rules: `set_rule`

A user can ask for a rule, such as "Always re-send the FBO notice when the tail changes" or
"Always rebook my hotel when a leg moves, as long as it is not more expensive and still free to
cancel". Turn the request into `set_rule` with `resolution`, `alertTypes`, `orderTypes`,
`maxCostIncreasePct` (default 0, never more expensive), `freeCancelOnly` (default true) and
`minHoursToService` (default 6). The first call reads the rule back in words. Save it with
`confirm: true` only after the user agrees.

- A rule runs the **next time alerts are read**, as the user who is reading them. When it did
  something, the reply starts with `autoResolved`. Tell the user what was done before anything
  else.
- A rule never picks a new vendor. It never cancels when the penalty is not known. When a guard
  stops it (the price moved, the terms are unknown), the alert stays open and `proposed` says why.
- `scope: "company"` makes a rule for everyone in the company and needs the Company Admin role.
- `alerts { includeRules: true }` lists the rules; `clear_rule { ruleId }` removes one.
