# Hotel flow — worked examples and the failure modes

Read [hotels.md](hotels.md) first. This file holds the parts only some runs need: worked conversations, what
differs from hotel to hotel, and the failures that look like something else.

The examples are taken from test responses (an operator's test account, shopped through the hotel
shopping service), shortened, with rate keys left out.
Prices and hotels will differ on your run; the shapes and the sentences will not.

**If the answer came back early** (`earlyReturn.trigger: count`, `pending` lists three hotels), the
same opening ends differently:

> "... I've priced 9 of the 12 hotels near the field so far; three are still pricing, so I'll check
> them before you choose."

then `shop_hotels { continuationToken }` once, and present the whole list again. If the summary
says *"The first 12 hotels left only 1 with rooms, so the next 12 were priced"*, say that too -
it is why some hotels are further out.

**If the answer carries `moreHotels`** (the first 12 are done and at least 3 have rooms; an FBO at MDT shopping for
N1234, Jan 10-12, had the Sheraton, the Residence Inn and the Homewood), present those three at once and offer
the rest - no continuation:

> "Three hotels have rooms for Jan 10-12: the Residence Inn at $275.28 a room all in (free to cancel until Jan 9), the
> Homewood Suites at $337.71 and the Sheraton at $486.18. I can check 12 more if you want - about 12 seconds. Which one, and
> which card - the Amex ending 2004 or the Visa ending 1111?"

"Book them both at the Residence Inn, Amex" books the Residence Inn straight away (`confirmedTotal: 275.28`, the per-room
figure). "Anything cheaper?" or "none of those" is the cue for `shop_hotels { continuationToken: moreHotels.token }`.

## Worked example 2 — "something they can cancel until 6"

> "Hotel at Teterboro on the 13th. Plans might change — they need to be able to cancel up to
> 6 PM that day."

```
shop_hotels { airportCode: "KTEB", checkIn: "2026-10-13", nights: 1,
              guests: [{travelerId}], freeCancelUntil: "18:00" }
```

The `summary`: *"... Top choice is Travelodge South Hackensack, 1.465 miles out, 111.16 USD all
in. Free to cancel until 6:00 PM EDT on Oct 13. 7 of the hotels shown cannot be cancelled free
until 6:00 PM on Oct 13 and are listed after the ones that can, each on its most flexible rate."*
Each of those seven carries `FLEXIBILITY_NOT_AVAILABLE`, for example *"No rate at Hilton
Hasbrouck Heights can be cancelled free until 6:00 PM on Oct 13. The most flexible there is free
cancellation until 5:59 PM EDT on Oct 13, on the FLTBR rate, which is the one shown."*

> "Two hotels let them cancel free until 6 PM on the 13th: the Travelodge South Hackensack at
> 111.16 USD all in, and the Knights Inn at 144.43. The other seven can't go that late — the
> Hilton's FlightBridge rate is 163.91 but free cancellation ends at 5:59 PM, a minute short.
> Shall I book the Travelodge?"

Note what the mode changed: without `freeCancelUntil` the Travelodge's top rate was a cheaper one
whose terms did not parse; asking for a deadline picked the rate that provably meets it.

## Worked example 3 — the crew is already there

> "They're already in town. Cheapest room tonight, they won't cancel."

```
shop_hotels { airportCode: "KTEB", checkIn: "<tonight>", nights: 1, guests: [{travelerId}],
              ratePreference: "lowest" }
```

`lowest` picks the cheapest rate even when it is non-refundable. Read `reversal` on what you
quote: here the top row happened to be refundable (Knights Inn, 144.43 USD, free until 6:00 PM
EDT), while the Courtyard's selected rate was `isNonRefundableRoom: true`. If `isNonRefundableRoom` is true or the row carries `CANCEL_TERMS_CONFLICT`, treat the rate as non-refundable whatever `cancelFree` or a deadline says, and tell the user the hotel's terms disagree.
(The Courtyard's "Member Prepay Non-Refundable" plan also sent a free-cancel deadline; it is
non-refundable, and its row says `CANCEL_TERMS_CONFLICT`.) The recommendation said
`INSIDE_FREE_CANCEL_WINDOW` — tonight's deadline is hours away — so it is a commitment:

> "Cheapest is the Knights Inn South Hackensack, 0.9 miles from the field, 144.43 USD all in. It's
> free to cancel until 6:00 PM today, so booking it is effectively a commitment. Book it for John?"

## Worked example 4 — a restricted rate, without and then with eligibility

> "Is there a government rate near Teterboro on the 13th?"

The default shop carries `RESTRICTED_RATES_EXCLUDED`, `excluded: ["senior", "aaa", "military"]`.
Government is not in the list — on this account the government-style rate shows as
military — so say what exists and ask:

> "I didn't find a government rate for that night. Some hotels do have military, AAA and senior
> rates; they're left out unless the traveller qualifies. Is John eligible for any of those?"

> "He's active-duty military."

```
shop_hotels { ..., eligibleFor: ["military"] }
```

Now the Travelodge's `selected` rate is `MIL`, `restricted: "military"`, 104.98 USD all in, free
until 4:00 PM EDT on Oct 13:

> "With the military rate, the Travelodge South Hackensack is 104.98 USD all in, free to cancel
> until 4:00 PM EDT on Oct 13. The hotel can ask for military ID at check-in."

Booking that rate without `eligibleFor` is refused before anything reaches the hotel:
`needs: [{ field: "eligibleFor", value: "military", question: "This is a rate for military
personnel. Does the guest qualify? The hotel can ask for proof at check-in." }]`. Ask that
question; book with `eligibleFor: ["military"]` only on a yes.

## Worked example 5 — an FBO rep, off a reservation

> FBO rep at Grand Rapids: "Find the crew of Wednesday's arrival a hotel."

```
fbo_search_reservations { ... }       → reservationId
shop_hotels { reservationId }
```

`stay`: KGRR, Oct 14 to Oct 16 (arrival to departure), rooms 1, `guests: []`, `guestSource: "no
named crew on reservation 97222"`. The FBO's own location was sent, so FBO-negotiated rates would
show as `fbo`; none priced, so every rate is `other` (on an older client `OTHER_TIER_ONLY` fires,
and it only says no negotiated rate was returned). The recommendation carries
`GUESTS_UNKNOWN` and `POLICY_UNKNOWN`.

> "For the Oct 14–16 visit, closest value is the Red Roof Inn Grand Rapids Airport, 2.4 miles,
> 119 USD all in for the two nights, though its cancellation terms aren't confirmed. The Econo
> Lodge is 146.63, free to cancel until 4:00 PM EDT on Oct 13. Both are public rates. The
> reservation lists the crew only as a count — who's staying, and is it one room or two?"

## Worked example 6 — the shopping service is down

```
shop_hotels { airportCode: "KTEB", checkIn: "2026-10-13", nights: 1 }
→ HTTP 503, Retry-After: 5
  { "code": "SHOP_CONTEXT_UNAVAILABLE", "retryable": true, "audience": "user",
    "detail": "The hotel shopping service didn't answer; try again in a moment. Nothing was booked." }
```

Wait five seconds, retry once. If it fails again:

> "The hotel shopping service isn't answering right now, so I can't see rooms or prices for
> Teterboro on the 13th. Nothing was booked. Want me to try again in a minute?"

No hotel names, no "there are no hotels", no prices from an earlier search.

## Worked example 7 — booking, the preview, and a price that moved

> "Book the Renaissance for John."

```
book_hotel { hotelId, bookingRateKey, checkIn: "2026-10-13", nights: 1,
             guests: [{travelerId}], confirmedTotal: 443.45, clientRequestId }
```

The rate is above the company's $200 nightly warning level, so the server books nothing and answers
`CONFIRMATION_REQUIRED`, `reasons: ["OVER_RATE_THRESHOLD"]`, with one question:

> "Book Renaissance Meadowlands Hotel for John Smith, 2026-10-13 to 2026-10-14, 443.45 USD all in?
> It is free to cancel until Oct 11 (the hotel's terms give the day, not a time; FlightBridge
> assumes 6:00 PM EDT). After that the hotel charges 443.45 USD."

Say why first — above the company's nightly limit — then ask that
question as written. Note the deadline: the shop had shown this rate free until 12:00 AM on Oct 11;
the booking's own check read the terms as a day only. The booking's words are the ones to use.
On a yes:

```
book_hotel { ...the same arguments..., confirm: true, confirmedTotal: <preview.total> }
```

When `reasons` also holds **`PRICE_CHANGED`**, the user agreed to a lower total than the vendor
now asks (in this run the agreed figure was 221.73 and the fresh price 443.45). `confirm` does not
carry across it:

> "The price has gone up since you saw it: the Renaissance is now 443.45 USD all in, not 221.73.
> Do you still want it?"

When it books, read `summary` back as it stands. Its shape:

> "Booked Hilton Hasbrouck Heights for John Smith, 2026-10-06 to 2026-10-07 (1 night), 163.91 USD
> all in. Confirmation [the number returned]. Guaranteed to John Smith's stored card on file. Not
> what you wanted? It's free to cancel until 6:00 PM EDT on Oct 6. Just say so and I'll cancel it."

The undo sentence is last on purpose: it is the line the user acts on. It is worded from the
rate's own terms, so when the terms did not parse it says that instead.

## Cancelling

> "Actually, cancel the Hilton."

```
get_existing_orders { orderType: "hotel" }       → orderId
cancel_order { orderId }
```

Recap the booking and get a yes first. When the free-cancellation deadline has passed, or the
terms never parsed, the server answers `CANCEL_MAY_INCUR_PENALTY` with the terms: read them to the
user and call again with `confirm: true` only if they still want it. On the deadline's own day,
when the hotel gave the day but not a time, it asks too (`reasons: [CANCEL_TIME_UNCERTAIN]`):

> "Today is the last day to cancel the Fairfield free, but the hotel didn't say by what time -
> FlightBridge assumes 6:00 PM, and they may charge 249 USD already. Cancel anyway?"

A cancellation number comes back on success; it can differ from the confirmation number, so quote
the one that was returned.

## Changing a booking — worked conversations

A guest change:

> **User:** Change Bob's room at the Hilton to Jane Smith.
> **You:** That makes a new reservation in Jane's name with its own confirmation number, and
> Bob's reservation, confirmation H2345678, is cancelled once Jane's is booked. Booking it now.
> *(modify_hotel)* Booked Hilton Hasbrouck Heights for Jane Smith, Oct 6 to 7, 163.91 USD all in,
> new confirmation H3456789. Bob's booking, confirmation H2345678, is cancelled
> (cancellation number H2345678). Free to cancel until 6:00 PM EDT on Oct 6; just say so.

A date change that costs more previews first:

> **User:** Move the crew's hotel to the 7th.
> **You:** *(modify_hotel answers CONFIRMATION_REQUIRED)* Book the Hilton Hasbrouck Heights for
> Jane Smith, Oct 7 to 8, 176.31 USD all in? This creates a NEW reservation with a new
> confirmation number; Jane Smith's confirmation H3456789 will be cancelled. The new rate is
> 12.40 USD higher than Jane Smith's.
> **User:** Yes.
> **You:** *(modify_hotel with confirm: true)* ...reads `summary`: the new confirmation, then the
> old one cancelled.

When the old room will not cancel (`rebook: partial`), the summary says: *"...NEW confirmation
H3456789. Bob Smith's booking, confirmation H2345678, could NOT be cancelled and is still
held, so both rooms are currently booked. I'll retry the cancel, or you can cancel it with
cancel_order."*

## Hotels differ in what their systems return, and you will notice

Which system a hotel is reached through is FlightBridge's configuration, never something to
mention. What you will see:

- prices arrive all in with a cancellation deadline on most rates; a day-only deadline is flagged
  `cancelTimeUncertain` (placed at 6 PM);
- for some hotels the booking's own price check **cannot read the terms**, so the preview carries
  `POLICY_UNKNOWN`;
- test inventory is thin and moves between calls (see [hotels.md](hotels.md)).

Two consequences for how you talk:

- **Expect some bookings to ask first** even when the shop showed a clean deadline. Ask
  its question; it is not a sign that something went wrong.
- **A rate with no rate code is still a real, bookable rate.** Describe it by room and price
  rather than inventing a code for it.

## Failures that look like something else

**A property with no rooms is not a property that does not exist.** `bookable: false` rows are
filtered out by default. If the user asks about a hotel by name and it is not in the list, shop
again with `includeUnbookable: true` before telling them it is not near that airport.

**A hotel that did not answer is not a hotel with no rooms - and a hotel that answered "nothing" is
not a slow one.** `HOTELS_TIMED_OUT` names the hotels left out at the time budget: ask again to include
them. `HOTELS_NO_RATES` names the hotels that answered with nothing for these dates: offer other dates
or a wider radius, not a retry.

**An empty result during a vendor outage looks identical to a quiet night.** The tell is
`availabilityUnconfirmed`, which fires when everything came back empty. Say the search could not
be confirmed and offer to retry, rather than reporting a city with no hotels.

**A rate that showed in the search can refuse to price at booking.** That is ordinary hotel-system
behaviour — on test inventory doubly so — and the property is usually still fine on its other
rates. Offer the next rate, and never explain it in terms of the systems behind FlightBridge.

**A stale key silently prices the wrong thing.** Carry `bookingRateKey`; a
`RATE_NOT_IN_CURRENT_LIST` warning from `get_hotel` means you carried the wrong one.

**A refusal about dates is a 400 with a code, not a failure of the search.** `CHECKIN_IN_PAST`,
`CHECKOUT_NOT_AFTER_CHECKIN`, `STAY_TOO_LONG` (over 30 nights), `BAD_DATE` and
`INVALID_FREE_CANCEL_UNTIL` each name exactly what to fix. Fix the argument and call again rather
than reporting that hotels could not be searched. Dates are judged in the **airport's** day, so a
late-evening request for "tonight" is accepted where a UTC reading would have refused it.

## Rooms, guests and who is actually staying

One room per crew member is the default, and it is a default rather than a rule — crews share
rooms, and someone may be joining. Ask once when the number matters and the leg did not say:
*"Two rooms, one each? Or are they sharing?"*

`guests` names who is staying: it clears `GUESTS_UNKNOWN` from the recommendation and carries into
the booking (it does not change the shop's prices). Pass travellerIds you were
given by a tool, and pass names only when you have no id.

## More worked conversations

**An FBO rep books the top pick for a visiting crew member with no card and no email** (Bob Smith,
ACME's N1234, MDT reservation 99284):

> Rep: "Book the top pick for Bob Smith."
> You: one `shop_hotels { reservationId, live: true, maxWaitSeconds: 25 }` (the user already said book - one
> complete, bookable answer, no continuation), then `book_hotel` on the recommendation.
> Reply: `PAYMENT_METHOD_REQUIRED` with `cardRequest: { offer: true }` (ACME's wallet at your FBO is
> empty), and the message says Bob also has no email.
> You (one call, nothing asked first): `request_card { reservationId: 99284, travelerId }` (an operator:
> `request_card { legId, travelerId }`), then: "Click this link to securely enter the credit card to be used for this booking, or let me know an email address if you want me to email this link to the traveler or other arranger." and the `link`.
> Rep: "Email it to Alex." -> `card_request_status { requestId, sendTo: { contact: { firstName: "Alex" } } }`
> - the same link, emailed; read the `summary` (who got it, masked).
> Rep, later: "Is Bob's card in yet?" -> `card_request_status { reservationId: 99284 }`: say its `statusWords`
> ("opened" / "identity confirmed" / "card saved"); when it is completed, book with no `vaultCardId`.
> Caller starts reading a card number: do not repeat it - "No need to read it to me; enter it on the secure
> page, or I can email the link." When ACME's wallet already holds one card, `book_hotel` uses it by itself;
> with several, the refusal lists them as `walletCards` by label: offer those first.

Never read out more of a card than its label, and never explain a refusal in terms of the systems
behind FlightBridge.

**"Why isn't our FlightBridge rate showing?"** (Embassy Suites Charlotte, `fbRateEvidence:
{ configured: true, requestedThisShop: true, returned: false }`):

> "FlightBridge's rate is set up at this hotel, but the hotel returned no rooms on it for these dates -
> negotiated rates are not guaranteed to be available unless the agreement is last-room-availability."

Said only because the user asked. The rows themselves were presented as what they are: the tier-`other`
rate is one of FlightBridge's negotiated rates, open to every customer, with its own cancellation terms
read out.
