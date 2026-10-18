# Hotels: shopping, booking, changing and cancelling

You are helping an FBO rep book rooms for a visiting crew's night-stop. Three tools do the work:
**`shop_hotels`** (properties near your airport or for a reservation), **`get_hotel`** (one property in
full) and **`book_hotel`** (the booking). Cancelling is **`cancel_order`**, and
**`get_existing_orders`** with `orderType: "hotel"` lists what is booked. The server's own
`instructions` carry the operating rules; this skill adds the judgment. **Read "FBO reps: a room for a
visiting crew" near the end of this file first**: the reservation, the rate card and your FBO's order
change how a shop starts and how it is presented.

`shop_hotels` and `get_hotel` are **looks**. They search and price and sell nothing, so call them
as freely as the question needs. `book_hotel` makes a real reservation.

**The environment.** The connector names its environment at the top of its
instructions and in its results; in a test environment every booking is a test booking. **A test
environment returns fewer hotels and may not confirm cancellation terms, and prices are re-checked
at booking** — so a rate the shop showed can come back at booking as `RATE_NO_LONGER_AVAILABLE`,
`VENDOR_REFUSED` or a higher price: offer the next one. None of that is a defect to apologise for.
Say nothing about speed unless asked.

## Step 0 — what you need before you can shop

A stay needs a **place**, a **window** and a **number of rooms**. You usually have to ask for at
most one of them.

- **A reservation is the best input.** `shop_hotels { reservationId }` takes the field, the nights and the
  arrival crew off the visit ("FBO reps: a room for a visiting crew" below).
- **No leg** → `airportId` (from `search_airports`) or `airportCode`, plus `checkIn`. One night
  is assumed unless you say otherwise. An explicit `rooms` always wins.
- **Guests.** Pass `guests: [{travelerId}]` whenever you know who is staying — from the
  reservation's manifest, `fbo_search_travelers` (inside the customer) or `get_traveler_profile`.
   It names who is staying for
  the recommendation and the booking; without it the recommendation will always refuse to stand on
  its own (`GUESTS_UNKNOWN`). It does not change the shop's prices. A leg shop fills `stay.guests` from the named crew. Never pass the login
  identity as a travelerId; it is a different kind of id.

**Read `stay.defaultsApplied` back to the user.** It names every default taken, in words — *"checkIn
2026-10-19, the night before the leg's 06:15 arrival (before 08:00 after a flight of 5 hours or
more)"*, *"checkOut 2026-10-21 from the next departure at this airport"*, *"rooms 2 from the leg's
named crew"*. A crew discovering at check-in that you booked the wrong night is the failure this
line prevents, and the early-arrival night is the one most likely to surprise them.

**Who you are talking to.** The server instructions name the persona. An FBO rep with one
location gets `shop_hotels` with no airport defaulted to it (and their FBO's own negotiated rates
requested) - one call, read `stay.defaultsApplied` back; several locations answer `needs: location`
once.

## Step 1 — shop, in the mode the user asked for

```
shop_hotels { reservationId }                          // best
shop_hotels { airportId, checkIn, nights }             // ad hoc
shop_hotels { airportCode: "KTEB", checkIn, rooms: 3, freeCancelUntil: "18:00" }
```

The mode decides which rate stands for each hotel and ranks it:

- **`ratePreference: "lowestFlexible"`** (default) — the cheapest rate that can be cancelled free.
- **`ratePreference: "lowest"`** — the cheapest even if non-refundable: the crew is already there,
  or the user says they will not cancel.
- **`freeCancelUntil`** (lowestFlexible only) — the latest time the user may need to cancel, in
  **hotel local time**: `"18:00"` / `"6pm"` is that time on the check-in day, or
  `"yyyy-MM-ddTHH:mm"`. A rate qualifies when it is free to cancel until then or later; a deadline
  the hotel gave as a day without a time qualifies only on a **later** day. An unreadable value,
  or one sent with `lowest`, is refused before anything is searched (`INVALID_FREE_CANCEL_UNTIL`,
  400): fix it and shop again.

Either way, a negotiated rate (`flightBridge`, `fbo`, `operator`, `shared`) **within 5 % of the
cheapest** qualifying rate is chosen instead, best tier first. The chosen rate is
**`bestRates.selected`**; the row's **`selection`** says why (`tieBreak` names the cheaper rate it
beat).

Narrowing, in the order people ask for it: `chain` (prefers it; `chainOnly: true` filters),
`maxRate` (nightly), `radiusMiles`, `maxRows` (default 10, max 25). `includeUnbookable: true`
keeps properties with no rooms — only when the user asks what exists rather than what is free.

The list comes back **already ranked**, and `rankedBy` states the playbook: bookable; with
`freeCancelUntil`, hotels that meet it before the rest; the chain asked for; within the company's
thresholds; the selected rate's price; a negotiated tier; distance. **Present it in that order.**
The ranking is the auditable answer to "why did you pick that one", and yours is not.

### Early return - the first hotels at once, the rest on one continuation

**Do not pass `maxWaitSeconds` or `maxRows` unless the user asks** for the complete list or a
limited one. The defaults ARE the early return: passing either switches it off, and the user waits
for every hotel.

**The one exception: the user has already said to BOOK.** "Book the top pick", "book the cheapest
free-cancel", "put the crew in the Sheraton" - the booking needs the complete list (the top pick can
change while hotels are still pricing), so shop ONCE with **`maxWaitSeconds: 25`** and book from
that answer: one call instead of an early answer plus a continuation. Off a reservation, or at an FBO rep's own
airport, add **`live: true`** (`shop_hotels { reservationId, live: true, maxWaitSeconds: 25 }`): a fresh pre-shop
or the location's rate card answers from the cache with no booking keys, which would cost the extra call this rule saves. The early return is for
browsing and quick answers; when the next step is a booking, it only adds a call.

`shop_hotels` answers as soon as **enough hotels are fully priced** with a bookable rate, with the
first three shortlist hotels among them and a short quiet spell, or when its wait runs out, whichever
comes first (`minHotels`, `maxWaitSeconds`). **The defaults are connector settings, not fixed
numbers; the answer's `earlyReturn.effective` reports the values this shop actually used**, and
quote those if the user asks how long it waits. The wait counts from when pricing starts:
`earlyReturn.waitedMs` is the whole time since your call, and `prePricingMs` the part spent finding
the airport and the hotel list before pricing began. `earlyReturn.trigger` says which:
`count` (enough hotels were in - not a clock running out), `complete` (everything was in; there is
no continuation), or `cap` (the time limit; fewer may be in, which happens under load).

- **Every row shown is final.** A hotel still pricing is never a row; it is named in `pending`.
  Present the rows at once and say how many are still pricing: *"Nine hotels are priced; three are
  still pricing."*
- Then call **`shop_hotels { continuationToken }` once**, or when the user asks about a pending
  hotel - **unless the answer carries `moreHotels`** (below): then present and offer, and wait. It returns the whole shop again, re-ranked; `arrivedSinceLastCall` names what is new. A
  hotel the shop's time ran out on comes back in `unpricedHotels` with `reason: budget` - say it
  did not answer in time, not that it is full.
- **Never say a pending hotel has no rooms.** While `recommendationBasis` is `partial` the
  recommendation carries `HOTELS_STILL_PRICING` and is never `autoBookable`: the list may still
  change, so collect the continuation before booking. A row from either
  answer books the same way.
- The summary carries one sentence about it - *"3 of 9 hotels have rooms so far; 3 more are being
  priced - call shop_hotels with continuationToken for the rest."* (it counts the hotels WITH
  ROOMS, which is what the rows show, of the hotels examined) - read it back as written.

**More hotels when the first ones were thin.** When the first 12 hotels leave fewer than 6 with
rooms, the next 12 on the shortlist are priced in the same shop, behind every other shopper's
searches. Their rows carry `wave: 2` and rank with the rest; the summary says *"The first 12 hotels
left only 1 with rooms, so the next 12 were priced."* Read it back - it explains why some hotels
are further out than usual.

**The second wave comes back in two steps.** When the continuation finds the first wave done and the second still
pricing, it answers AT ONCE with the first wave's hotels and a NEW `continuationToken` - the summary says *"...the next 12 are
being priced"* (or *"Checking 12 more hotels"*). Present what is in straight away (*"Three hotels have rooms so far - the
Residence Inn at $264 is the best value; I'm checking 12 more."*), then call `shop_hotels { continuationToken }` with the NEW
token once for the rest. The old token is spent.

**`moreHotels` - the second wave OFFERED, not waited for.** When the first wave is complete and at least 3 hotels
already have rooms, the answer carries `moreHotels {pending, token, aboutSeconds}` and the summary says *"3 hotels have rooms;
I can check 12 more if you want (about 12 s)."* Then:
- **Present the hotels shown at once, and offer the rest in the summary's words.** Do NOT call the continuation yet.
- Call **`shop_hotels { continuationToken: moreHotels.token }`** only when the user **asks for more**, **picks none** of
  the hotels shown, or none fits what they asked for (price, flexibility, bed). The rest keep pricing in the background, so
  a later call answers from what has finished; they have a budget of their own (`budgetSeconds`), and a hotel past it comes
  back in `unpricedHotels` with `reason: budget` - "did not answer in time", never "full".
- **"Book X" for a hotel already shown books it** - no continuation first. The recommendation carries
  `MORE_HOTELS_OFFERED` (not `HOTELS_STILL_PRICING`) and is never `autoBookable`, so do not book unasked; when the user names
  the hotel, book it as it stands. The held shop lasts 2 minutes after its last answer; past that, `CONTINUATION_EXPIRED`
  means shop again.
- **Stragglers are offered the same way.** When 8 or more hotels already have rooms and only hotels of the first
  list are still pricing, the answer carries `moreHotels {wave: 1}` and the summary says *"10 hotels have rooms; 1 more is still
  pricing - I can wait for it if you want."* When the answer carries `moreHotels`, present what you have and OFFER; call the
  continuation only when the rep wants the rest or none of the shown hotels fits. The continuation's `resolved` says what
  became of each pending hotel (`shown`, `belowCutoff`, `noRooms`, `budget`, `failed`) - read its one clause back.
- Without `moreHotels` (fewer than 3 hotels with rooms) nothing changes: call the continuation once for the rest, as above.

**Rooms follow the guests.** Naming two guests prices two rooms (`stay.defaultsApplied`: "rooms 2 (one per named guest)"),
even before they are on the visit's manifest; totals and `availableQuantity` are judged against that. Pass `rooms` only when
the user asks for a different count.

**A deadline that cannot be one.** When a rate's `reversal` carries `deadlineImplausible` (for example a time
stamped to the second that lies months before arrival, or a deadline after check-in), its free cancellation is UNCONFIRMED:
`cancelFree` is null and the rate is never the flexible pick. Say *"the hotel's cancellation terms could not be confirmed"*
and quote the penalty; never quote the stamped time as a free-cancel deadline.

## Step 2 — present it honestly

**The selected rate is the hotel's price.** Quote `bestRates.selected`, which is written in full.
`bestRates` also holds the cheapest per tier (`flightBridge`, `fbo`, `operator`, `shared`, `other`)
plus `lowest` and `lowestFlexible`, which answer "is there a FlightBridge rate there?". **The
answer is compact (`responseShape: "compact"`):** a tier that is the same rate as another is
written `{ "sameAs": "selected" }` (or the tier it repeats) - read it as that rate; any other tier is a
short alternate (`bookingRateKey`, `rateCode`, `tier`, `bed`, `total`, `cancelFree`,
`freeCancelUntilText`, warning codes) - `get_hotel` has its full terms. A row's warnings speak for its
selected rate; a threshold or no-flexible-rate warning on a row may be the code alone when the
top-level warning of that code already gives the message and each hotel's numbers. Name a tier for what it is —
*"the FlightBridge rate, $163.91 all in"* — and when a negotiated rate costs more than the 5 %
margin above a public one, give both and let the user choose.
**Savings: when a rate carries `comparison.savings` (or `savingsText`), say it in one clause with the public figure and whether it is the same room** - *"the FlightBridge rate is $143 - $31 under the hotel's $174 best available for the same king room"* (`roomMatch` `lowestPublic`: "under its lowest public rate").
With `cheaperPublic`, say the public rate is cheaper and let the user choose (a negotiated rate may still be preferred for its terms or loyalty). Never compute a saving yourself from two rows; use the field.

**The default mode prefers terms that are confirmed.** A rate with confirmed free cancellation
beats one whose terms are not confirmed yet, unless the unconfirmed one is more than 10 % cheaper
(`selection.termsRule` says when a cheaper unconfirmed rate was passed over). When an unconfirmed
rate is still the one shown, the row carries **`TERMS_UNCONFIRMED`** and the summary names it -
*"it is shown because it is 19% cheaper than the cheapest rate with confirmed free cancellation"* -
so say that the terms are not confirmed, name the confirmed alternative, and offer
`freeCancelUntil` if the user needs a free cancel they can rely on.

**A deadline that cannot be met is said, not hidden.** `FLEXIBILITY_NOT_AVAILABLE` names the time
asked for and the most flexible deadline that hotel has, and the hotel is listed after those that
qualify; read it as written. `NO_FLEXIBLE_RATE`: nothing there is refundable.
`FREE_CANCEL_DEADLINE_UNKNOWN`: the deadlines are not known yet. The top-level `warnings` keep one
of each code; every hotel's own are on its row, so read the row before speaking about a hotel.
A row's warnings speak for its **selected** rate: `OVER_RATE_THRESHOLD` on a row means the rate you
are quoting is over the company's level, and the top-level one lists those hotels in `hotels`
(`count`, `name`, `nightly`, `threshold`). An alternate over the level carries the warning on its
own `bestRates` entry only — mention it if the user asks for that rate.

**A king bed is the default room.** Every rate carries `bed` {type: king | queen |
double | twin | sofa | unknown, count}, read from its room description. Each hotel's `bestRates.selected`
is chosen from its **king** rooms first, by the same playbook (mode, negotiated within 5 %, deadline);
a king that costs more than a double is still the pick. A hotel with no king room is chosen over all its
rooms, and `selection.bed.note` and `defaultsApplied` say so (*"no king room offered; the selected room
is a room with 2 queen beds"*) - read that back. Say "a king room" in the read-back as a matter of
course; ask about beds only when the user mentions them. Pass `bedPreference` (`queen`, `double`,
`two-beds`, `any`) on `shop_hotels` and `book_hotel` only when the user WANTS a bed ("two queens for the
crew"); a user asking what the room options are has not chosen one - keep the default and read the
options from `get_hotel`. **"Book it, but tell me the room options" is a request for options first**:
present them (Step 5) and book on the user's pick - never book the default pick in that same turn. A booked room whose bed differs from
one the user named answers `BED_PREFERENCE_NOT_MET` - relay it. An accessible room (mobility or hearing) carries `bed.accessible: true` and is never the default pick when a regular room with the same bed costs no more; a crew that needs one picks it from `get_hotel`'s rows (they are all listed) and books that row.

**Read `roomDescription` for what a crew cares about.** Smoking rooms and "pay now" plans show only
there; say so when the pick you are offering is one of them. A **package** rate (`package: true`,
with `inclusions` such as "buffet breakfast for 4 guests daily, parking discount") sells something
besides the room: say so in one clause when you offer it - the price includes extras the crew may not
need.

**Describe the rates that came back, as what they are.** Tier `other` is a public rate, one of
FlightBridge's negotiated rates (open to every customer) or the best available rate: not one of this
buyer's own negotiated rates. **Never describe a tier-`other` rate as another company's corporate
rate, even when its room text names a company or a programme; read its cancellation terms out like
any other rate.** Never speculate about rates
that were not returned, never suggest the customer is missing out or that someone should "load the
rate", and do not bring up a FlightBridge rate that is configured but did not come back. That fact
rides in the row's `fbRateEvidence` (`configured`, `requestedThisShop`, `returned`) for one purpose:
answering when the user **asks** (see "Questions users ask" below). A client on an older tool list
may still see `OTHER_TIER_ONLY`; it now only says that no FlightBridge or company-negotiated rate was
returned. Relay it as written and add nothing.

**`paging.localPropertyCount` is not availability.** It counts properties FlightBridge holds near
that airport. Quote `paging.propertiesExamined` (what was searched) and `paging.returned` (the
rows shown, at most `maxRows`): *"I searched 12 properties near the field; here are the 9 with
rooms."* Two warnings name the hotels left out, and they mean different things:
- **`HOTELS_TIMED_OUT`**: those hotels did not answer in time. They are not full; offer to ask again
  (a fresh `shop_hotels` with the same arguments; `continuationToken` is only for hotels still in
  `pending`).
- **`HOTELS_NO_RATES`**: those hotels answered, with nothing available for these dates. Offer other
  dates or a wider radius, and do not offer to ask again as if they had been slow.
(`HOTELS_NOT_PRICED` is the older single warning, kept for older clients; it now says which part was
which.)

**"No hotels" is almost always the wrong sentence.** It is true only when `paging.truncated` is
false **and** `availabilityUnconfirmed` is false. Otherwise say what was searched and offer to
widen the radius or move the dates. `availabilityUnconfirmed` means every property came back
empty, which also happens when the vendor is not answering.

**Read `summary` back close to verbatim.** It is written on the server and carries these caveats
in the right proportion; paraphrasing is how they get dropped.

## Step 3 — cancellation terms, which is where the real risk sits

Every rate carries `reversal`. Give the deadline **in words** from `freeCancelUntilText` — *"free
to cancel until 6:00 PM EDT on Oct 6"* — which is placed in the hotel's own time zone; the bare
`freeCancelUntil` has no zone.

- `cancelFree: true` — a real deadline still ahead. Say it.
- **`cancelTimeUncertain: true`** — the hotel's terms gave the **day, not a time**, and FlightBridge
  assumed the hour (usually 6:00 PM local). `freeCancelUntilText` prints that hour as if it were
  the hotel's, so give the day and say the time is assumed. The shop `summary` adds *"(the hotel
  gave the day; the hour is FlightBridge's assumption)"* — never drop that parenthesis — and the
  booking words it *"Oct 11 (the hotel's terms give the day, not a time; FlightBridge assumes
  6:00 PM EDT)"*.
- `cancelFree: null`, `policyParsed: false` — *"the cancellation terms aren't confirmed for that
  rate"*. Never translate `policyText` yourself.
- `cancelFree: null`, `policyParsed: true` — the zone is unknown (`zoneResolved: false`); give the
  deadline as approximate.
- `cancelFree: false` — non-refundable (`isNonRefundableRoom: true`: the plan calls itself that) or
  the free window has closed; `policyText` says which.
- **Decide from `cancelFree`, and only from `cancelFree`.** A vendor's "refundable" with no deadline
  behind it is not a promise — some prepaid non-refundable plans are marked refundable — which is
  why such a rate arrives with `cancelFree: null`, never `true`.
- **Non-refundable wins.** If `isNonRefundableRoom` is true or the row carries `CANCEL_TERMS_CONFLICT`, treat the rate as non-refundable whatever `cancelFree` or a deadline says, and tell the user the hotel's terms disagree. Some plans name themselves non-refundable
  ("Member Prepay Non-Refundable") and also send a free-cancel deadline; the server drops that deadline
  (kept in `reversal.conflict.deadlineAsSent`), never selects such a rate as flexible, and
  `book_hotel` always asks before booking it. Read the warning's `message`: it quotes both statements.
  A plan that says only *prepaid* and also gives a deadline arrives as `cancelFree: null` with the
  same warning — unconfirmed, not free.

A change of dates, room or property is a cancel and a rebook: price it as a new stay, and say the
old booking is cancelled under whatever terms it carries.

## Step 4 — restricted rates: never assume the traveller qualifies

Government, military, AAA/CAA (and association) and senior rates are for travellers who show ID
at check-in. **They take no part in any pick, ranking, recommendation or booking unless the user
has said the traveller qualifies** and you pass `eligibleFor: ["government" | "military" | "aaa" |
"senior"]`. A rate on the buyer's own negotiated code is exempt.

- **Do not volunteer them.** Membership and restricted rates (AAA, AARP, government, senior,
  military) are NOT mentioned unless the user asks for such a rate, or asks whether a cheaper rate
  exists - never "AAA rates ~$3 lower exist (not shown)" unasked (the same posture as negotiated-rate
  availability, which is not stated unless asked).
- **`RESTRICTED_RATES_EXCLUDED`** lists in `excluded` the categories that exist in this search. Use
  it only when the user asked: then ask once, about exactly those - *"Some of these hotels also have
  military and AAA rates — does anyone on the crew qualify?"* If the user asked for a category the
  warning does not list, say this search found none.
- Once let in, a restricted rate shows `restricted` on its row. Name it — *"the military rate,
  $104.98 all in"* — and say the hotel can ask for proof.
- **`book_hotel` refuses a restricted rate without `eligibleFor`**: it answers `needs:
  eligibleFor` with one question and books nothing. Ask it; pass `eligibleFor` only on a yes (a yes
  the user already gave for this traveller, earlier in this conversation, counts).

Only the user's words count — for this traveller, in this conversation. Never infer eligibility
from a title, a tail number or an earlier trip.

## Step 5 — one property in detail, and re-pricing

**Every row of a normal shop is already priced, all in**: `ratesLoadedFor` equals `hotelCount` and
each rate's `pricing` is `"confirmed"`. The shop ships only the rates a mode can pick (`rateCount`
is the property's true count), so **`get_hotel` is for one property's full list** — "tell me about
the Hilton", "do they have a king". `get_hotel { hotelId, airportId, checkIn, nights }` reads every
rate through the same shopping service; pass the shop's `stay.airportId` and exact window, or the
rates will not match. Its `rates` list one row per **bed type, rate code and price** - the rooms that
differ, cheapest first - and `ratesShown` says how many of how many; pass `includeAllRates: true` only
when the user wants every rate. That list is what to read the room options from: group it by `bed`
("king rooms from $650.90, two queens from $650.90, a king with breakfast at $788.90").

**The older path** still serves some accounts: there only the top
rows have rates, `ratesLoadedFor` is smaller than `hotelCount`, and you call `get_hotel` before
quoting any row past it.

`get_hotel { ..., priceCheck: [bookingRateKey, ...] }` re-prices up to three rates against the
vendor's booking system and returns the policy text. It is the check `book_hotel` runs, so it
tells you whether a rate still prices — for some hotels it cannot read terms the shop showed, and
on test inventory it can disagree with the shop. A check failing on one rate is ordinary: offer
another and leave the hotel on the list.

**Carry `bookingRateKey`.** It identifies the room and rate across calls and is what `book_hotel` and
`get_hotel` take. The compact answer no longer shows the short-lived `rateKey` at all; the connector
keeps it server-side.

**Booking terms.** When the hotel's booking-time text does not parse, the booking's `reversal` carries
the shop's structured terms for that same rate, marked `source: "shop"` (the hotel's text stays in
`policyText`); `source: "policyText"` means the connector read the text itself ("NO CANCELLATIONS" is
non-refundable). Read the deadline from `freeCancelUntilText` either way.

## Step 6 — the recommendation

`recommendation` names the top row with `autoBookable` and `reasons[]`. **`topRateConfirmed: true`
means the shop priced the rows itself** — no separate vendor check ran (`topRateCheckMs` is empty;
`confirmTopRate` changes nothing). Each rate's own `totalIncludesTaxesAndFees` still decides
whether its total is all in. On the older path it means the
recommended rate alone was re-priced at the vendor.

`autoBookable: true` means every condition for booking without asking again is met: a priced
rate, a parsed and time-zoned policy, free cancellation at least 24 hours out, within the
company's thresholds and its own price limit, guests known, and one clear winner. When it is false,
each reason is worth a different sentence:

**Favourites and the recommended hotel are chosen for quality and value - crews must be rested to fly
safely. Never present a hotel's price against the cheapest hotel as a reason against it** ("costs
2.5 times the La Quinta" is never said as a mark against the pick); show a cheaper option only when
the user asks for cheaper.

| reason | what to say |
|---|---|
| `GUESTS_UNKNOWN` | ask who is staying, then re-shop with `guests` |
| `POLICY_UNKNOWN` | the terms could not be read; the user must accept that to book |
| `POLICY_TIMEZONE_UNKNOWN` | the deadline's time zone is unknown; treat it as approximate |
| `INSIDE_FREE_CANCEL_WINDOW` | free cancellation ends within a day, so it is a commitment |
| `NON_REFUNDABLE` | say so before anything else |
| `CANCEL_TERMS_CONFLICT` | the hotel's terms disagree (declared non-refundable or prepaid, yet a free-cancel deadline was sent); treat it as non-refundable and say so |
| `CANCEL_TIME_UNCERTAIN` | the hotel gave only the day of its deadline, and that day starts within 24 hours; treat it as a commitment |
| `OVER_RATE_THRESHOLD` / `OVER_TOTAL_THRESHOLD` | above this company's own warning level |
| `PRICE_OUTLIER` | above the company's OWN price limit (its "Max % Above Lowest Rate Allowed" setting); name that limit, never "cheaper hotels exist". There is no default: with no company limit it never fires |
| `NO_CLEAR_CHOICE` | two are equivalent; put both to the user |
| `AVAILABILITY_UNCONFIRMED` | the search may not have completed; offer to retry |
| `NO_BOOKABLE_RATE` | nothing at that property priced |
| `HOTELS_STILL_PRICING` | some hotels were still pricing when the answer went out (`stillPricing.message` names them); collect the continuation before booking |
| `MORE_HOTELS_OFFERED` | enough hotels have rooms and more can be checked on request (`moreHotels`); offer them, and book a hotel shown when the user names it |
| `TERMS_UNCONFIRMED` | the rate shown has cancellation terms that are not confirmed yet; it was shown because it is much cheaper, or nothing there is confirmed |

## Step 7 — money, said correctly

**Read `totalIncludesTaxesAndFees` on every rate.** On the shopping service it is true on every
row: `total` is all in, `taxesAndFees` sits beside it, `confirmedTotal` equals it, and every
preview and booking total is all in too. Where it is `false` — an offer that did not
itemise its taxes, or a shop row on the older path — `total` is room only and only a price check's
`confirmedTotal` is all in. Quoting one as the other understates the cost by the tax and invents a
price change that never happened. `total` is **one room for the whole stay**: with several rooms
say "per room" (a preview also gives `totalAllRooms`), and when a rate's `availableQuantity` is
below `stay.rooms`, say that rate may not hold for every room.

## Step 8 — when the shop cannot run

`SHOP_UNAVAILABLE` (the shopping service) or `SHOP_CONTEXT_UNAVAILABLE` (FlightBridge's side of
it) is HTTP 503 with `Retry-After: 5` and *"The hotel shopping service didn't answer; try again in
a moment. Nothing was booked."* `shop_hotels` has already retried a quick failure once; `get_hotel`,
and a booking that needs a fresh read, can answer it too, without a retry of their own.

Wait about five seconds and **retry once**, without telling the user yet. If it fails again, tell the user the hotel shopping
service is not answering right now, that nothing was booked, and offer to try again shortly.
**Answer only from what a tool returned**: no hotel names, prices or availability from memory, and
never "there are no hotels".

## Step 9 — booking

**Book without asking only when all six hold.** Otherwise recap and get a yes.

1. **The user asked for a room** — "book it", "get the crew into the Hilton" — not options or a
   price.
2. **The stay is known**: the dates, from the leg or said out loud.
3. **The place is known**: one property, one rate.
4. **The guests are known**: travelerIds from the manifest or the user's words, one room each.
5. **The choice is determined**: by the user's words, the guest's profile, or the shop's
   `recommendation` with `autoBookable: true`. Never by your own preference between near-equals.
6. **Nothing blocks it**: the recommendation's `reasons[]` is empty — or, for a hotel the user
   picked themselves, that row's own `warnings` and terms raise nothing. Say anything they do raise
   first; the server will ask about it anyway.

```
book_hotel { hotelId, bookingRateKey, checkIn, checkOut, guests: [{travelerId}, ...],
             confirmedTotal?, confirm?, eligibleFor?, clientRequestId }
```

**`confirmedTotal` is PER ROOM**: the per-room all-in total the user was shown - the row's or `preview.total`. For
several rooms pass that per-room figure, or `confirmedTotalAllRooms` with the group's total (`preview.totalAllRooms`); either
is compared per room. The booking echoes `total` (per room) and `totalAllRooms`; say both when there is more than one room
(*"$275.28 a room, $550.56 for both"*).

The server re-reads and **re-prices the rate at the vendor** before booking and judges **those
fresh terms**, which can differ from what the shop showed. So it may answer instead of booking:

- **`blocked: CONFIRMATION_REQUIRED`** carries `reasons[]`, a `preview` and one `question`. Put
  exactly that question to the user; on a yes, call again with the same arguments plus
  `confirm: true` and `confirmedTotal: preview.total`. For some hotels the fresh check cannot read
  the terms, so `POLICY_UNKNOWN` here is common: ask; do not argue from the shop's deadline.
- **`PRICE_CHANGED` in those reasons** — the fresh price is above the total the user agreed by more
  than the greater of $5 or 2 %. `confirm: true` does not override it: give the new all-in `preview.total`, ask
  again, and only on a yes call with `confirmedTotal` set to it.
- **`needs: eligibleFor`** — a restricted rate (Step 4); ask its question.
- **`RATE_NO_LONGER_AVAILABLE`** / **`VENDOR_REFUSED`** — the rate went away or the hotel's system
  said no; nothing was booked. Offer the next rate.
- **`GUEST_NAME_INVALID`** — a last name needs two letters, and some hotel bookings need a first
  name. Refused before any booking; ask for the full name.
- **`BOOKING_UNCERTAIN`** — the booking call failed in transit, so it is **not known** whether a
  room was booked. Say exactly that, do not book again, and read `get_existing_orders`
  (`orderType: "hotel"`) for those dates first.

**A guest with no email gets no confirmation email.** Email is optional but recommended:
always ask, never require it.
- **"Book" means book.** When the user has already said to book ("book Bob at the Sheraton"),
  book as soon as the gates allow - do NOT stop to ask about the email first. Then, after reading the
  booking back, say the email is missing and offer to add it (`GUEST_EMAIL_MISSING` names the guest).
- When the user is still choosing (no instruction to book yet) and a guest's profile has no email
  (`hasEmail: false` from `operator_search_travelers`, `fbo_search_travelers` or `get_traveler_profile`),
  you may ask **once**, in the same message as the recap - *"Bob has no email on file - want me
  to add one, so he gets his confirmation?"* - and save it: an operator with
  **`operator_update_traveler { travelerId, email }`**, an FBO rep with
  **`fbo_update_traveler { travelerId, email }`** (the customer's crew member, from
  `fbo_search_travelers`).
- If the user skips it, book anyway and say plainly that this person will not get the confirmation
  email. Never hold a booking for it.
- **`GUEST_EMAIL_MISSING`** (a warning, `audience: user`) names each guest without an email, on the
  booking's question and on the booking itself; the booking goes ahead regardless. Relay it in one
  sentence and offer to add the email.
- FBO reps use `fbo_update_traveler` - it changes only the email and phone on one of your
  customers' travelers, never a card. `TRAVELER_NOT_FOUND` means the person is not on any of your
  customers' lists: nothing changed.

**Never say a card is or is not on file from memory; use the latest reply's payment status or call
`card_request_status`.** A card can be saved through the link at any moment between two turns. Every
`shop_hotels` answer for a reservation, or for an operator's leg with named guests - the cached rate-card
answer included - carries `paymentStatus`: `known` with `paymentPreview` (`cardOnFile`, the wallet's `cards`
by label, each guest's `willUse`, `latestCardRequest` and a one-line `words`, read at that moment), or
`unknown` - then say nothing about the card until `card_request_status` has answered.

**Ask "which card?" with the hotel choice, not after a refused booking.** A reservation shop and
`fbo_get_reservation` carry `paymentPreview`: each guest's `defaultCard` (a label, or null), `willUse` (the card the
booking will take if you name none - their default, else the customer's only wallet card) and the wallet's `cards`
`[{vaultCardId, label}]`. When it carries a `question` (someone has no default card and the wallet holds several), put that
question IN THE SAME MESSAGE as the hotel list - *"Brendan and Tony have no default card; ACME's wallet holds the Amex ending
2004 and the Visa ending 1111. Which hotel, and which card?"* - then book with `payment: { vaultCardId }` (it covers every
guest without a default; a guest WITH a default keeps it), or `payment: { perGuest: [{travelerId, vaultCardId}] }` when the
user names different cards for different people. When `willUse` is filled for everyone, there is nothing to ask. A card the
user chooses for a guest with no default is saved as that guest's default by the booking (as the web app's "make
preferred" does); the summary then says *"Saved American Express ending 2004 as Brendan Johnson's default card for next
time."* - read it as written, and only when it is there.

**The visit's checklist after a booking.** A booking made off a reservation ends its `summary` with the Hotel
checklist item as FlightBridge filed it - *"Checklist: Hotel - All Confirmed (2 orders)."* - or, when it has not caught up
yet, *"...picks up the new booking when FlightBridge processes it, usually within a minute."* Read it as written; never set
the Hotel item yourself after booking (the platform does it). The same goes for a car and the Rental Car item.

**Payment is by reference only.** Each room is guaranteed to that guest's stored card on file, or
to a company-wallet card named by `payment.vaultCardId`. **A card number is never typed into this
conversation** — there is no field for one; if one appears, do not repeat it and point at the secure link. `PAYMENT_METHOD_REQUIRED` means the guest has no card on
file. **When it carries `walletCards`** - the company wallet's cards, each `{vaultCardId, label}`,
the label being the wallet's nickname or "<brand> ending <last 4>" - offer them by label: *"Bob has
no card on file. Use the Amex ending 1234 from the company wallet?"* On a yes, call `book_hotel`
again with the same arguments plus `payment: { vaultCardId }` from that entry (the retry reuses the price check the refusal
already ran - `priceCheck: "reused"` - so it is quicker). The refusal's `message` is the one sentence to say; its
`guestsWithoutCard` lists everyone without a card. Read the label as
it is; never a number beyond it. With no `walletCards`, say a card has to be added in FlightBridge
(the response may carry `walletPage`). If the same refusal says the guest has no email either,
offer to add the email (above) in the same breath. Read `payment.sentence` back as written.

**No card on file, or a card to put on file:** read `card-link.md` beside this file before the next call. It holds
the secure card link, whose card it is, the note, and the no-card-on-file flow.

**Read the booking `summary` back verbatim.** It names who is booked, the confirmation numbers,
how it is paid, and **ends with the undo sentence worded from the rate's own terms** — *"Not what
you wanted? It's free to cancel until 6:00 PM EDT on Oct 6. Just say so and I'll cancel it."*
When the terms did not parse, it says so instead. Never replace it with a blanket "no penalty".

**The hotel's address and phone.** The booking's `hotel` carries `address` {line1, line2, city,
stateProvince, postalCode, country} and `phone`, and the summary already names the phone beside the
confirmation. After the summary, give the address in one line (*"Sheraton Harrisburg Hershey, 4650
Lindle Rd, Harrisburg, Pennsylvania 17111"*) when the crew will drive there or the user asks. Leave
out a part that is null; never fill one in. `get_existing_orders` hotel rows carry the same `address`,
`phone` and `hotelName`, so "where is the crew staying?" is answered from the row.

`groupResult: partial` means some rooms are held and some are not; `rooms[]` names which. Say
both halves. `specialInstructions` stay on the FlightBridge booking and **do not reach the hotel**
— if a note matters (late arrival, a rollaway), tell the user to call the hotel. Always send a
`clientRequestId`: a retry with the same one returns the original booking (`replayed: true`). A NEW room - even
another guest at the same hotel, dates and card - is a new booking and takes a NEW `clientRequestId`; reusing
the last one would replay the old booking and book nothing.

**`PRODUCTION_GDS_BLOCKED`** (the code's name is historical) means booking is disabled for this
account in this test environment. When it carries no `reason` (or `reason: policy-blocked`), say exactly
that — *"booking is disabled for this account in this test environment"* — and do not retry or look for
another way.

**Live hotel inventory from a test environment.** Some test accounts shop and book the hotels'
LIVE inventory, and booking is allowed only on a refundable rate whose free cancellation runs 48 hours
or more. You can tell from `productionPolicy` (`applies: true`) on a `book_hotel` preview or answer:
- **Before asking for the yes**, when the preview carries `preview.productionPolicy.applies`, say the
  cancel-by date and time in words from `productionPolicy.cancelByText` (else `cancelBy`, hotel time) and
  that this is live inventory: *"Free cancellation until Mon Oct 20, 6 PM local — this is live inventory."*
  When `dayOnly` is true the hotel gave the day, not a time: say so. Such a booking is never made without
  an explicit yes (the preview's reason `LIVE_INVENTORY`), even when it would otherwise book unasked.
- **When refused** — `PRODUCTION_GDS_BLOCKED` with a `reason` — say the reason in plain words and name the
  deadline that was found (`cancelBy`): `non-refundable` (non-refundable or paid in advance),
  `cancel-terms-unknown` (no free-cancellation deadline with a date is stated), `deadline-under-48-hours`
  (free cancellation ends less than 48 hours from now; the number is in the reason), `deadline-passed`. Then **offer a rate that
  qualifies**: the one with the LONGEST free-cancellation window shown (`reversal.freeCancelUntilText`),
  and book that one only on a new yes. Never retry the refused rate, and never present the refusal as an
  outage.
- After the booking, read `summary` as always; `cancelBy` repeats the deadline (ISO, hotel time). Every
  such booking is on FlightBridge's list of live bookings to cancel before the deadline unless kept, so
  if the user says it is a real stay they want to keep, tell them to say so to the FlightBridge team.

## Step 10 — cancelling

Cancel with `cancel_order { orderId }`, the orderId coming from `get_existing_orders` with
`orderType: "hotel"`. **Recap first and get a yes** — hotel, guest, dates, confirmation.

- **`CANCEL_MAY_INCUR_PENALTY`** — the free-cancellation deadline has passed or the terms are
  unknown. The platform itself would cancel without checking and the penalty would arrive on the
  bill. Read the terms to the user; call again with `confirm: true` only if they still want it.
  With **`reasons: [CANCEL_TIME_UNCERTAIN]`** it is the deadline's own day and the hotel gave the
  day, not a time: FlightBridge's 6:00 PM is only an assumption, so the hotel may already charge.
  Say exactly that - *"Today is the last day to cancel free, but the hotel didn't give a time, so
  it may already cost 249 USD"* - and confirm only if they still want it. Before that day it
  cancels without asking, as before.
- **`cancelled`** — done; quote the `cancellationNumber`. It can differ from the confirmation
  number, so never read one as the other. The confirmation number is `confirmationNumber` on each
  booked room.
- **`alreadyCancelled`** — nothing else was done.
- **`CANCEL_DECLINED`** — the room is **still booked**. Say so.
- **`CANCEL_UNCERTAIN`** — never say it is cancelled. Say what the message says, and check
  `get_existing_orders` before any retry.

## FBO reps: a room for a visiting crew

**The first call names the location.** When a rep with several locations has said where they work ("working
out of MDT"), pass it on the FIRST location-bound call - `fbo_rate_card {airportCode: "MDT", kind: "hotels"}`,
`shop_hotels {airportCode}`, `shop_rentals {airportCode}`, `fbo_search_reservations {airportCode}` - and the connector
remembers it. Never call a location-bound tool with nothing when the rep already told you the airport.

**Quick answers first.** An FBO rep's "what hotels do you recommend?" is answered from the location's rate
card with no tool call: the speakable line in the instructions, `me.rateCard`, or `fbo_rate_card` (pass `kind: "hotels"`
for a hotel question, `kind: "cars"` for a car question; the visits' pre-shops only with `includeReservations: true`) (cache only - the
FBO's Favorites first, then FlightBridge-wide popularity; one night, king-first, 14 days; every figure "as of" its age).
Recommended hotel, two alternatives, the typical price and the spread, the age, then *"Do you want me to look into any
of those?"*. The recommended hotel is the card's first hotel that HOLDS a price (`recommended: true`); never lead with,
or name as an alternative, a hotel showing "no rates on any of the 14 days held". `tonight.termsClass` `deadlinePassed`
means a free-to-cancel rate whose time for tonight has passed - say that, not "non-refundable". That is the whole default answer: **60 words at most**; tables and notes only when asked. **One date, one number:** for a question about ONE date ("what are the hotels running on Oct 20?", "how much is the Hilton Thursday?") call `fbo_rate_card {date}` and say ONE number per hotel: `quote.total` (all-in, one king room, one night) with its terms, as of its age; add "or $X non-refundable" (`alternatives.nonRefundable`) only when the rep asks for the cheapest; a hotel with no `quote` says its `unavailable` ("sold out", "did not answer as of ..."), never a typical figure. Never present two of a hotel's figures as a range. A range ("usually $245-266") is only for "what does it usually cost" and comes from the overview's `typicalRange`. Then offer to check live. When the caller wants to book or a
firm price for their dates, shop it live (`shop_hotels {hotelIds: [it], checkIn, checkOut, rooms}` or `get_hotel`) and
book from that; never quote or book a cached figure as live. `rate-card.md` has the full script.

**A reservation's pre-shop.** `shop_hotels {reservationId}` with nothing else answers from the visit's
pre-shop when it is under 6 hours old: `cached: true`, `asOf`, `ageWords`, totals per room for the whole stay, and no
booking keys. Read it as "as of" figures; to quote a firm price or book, call `shop_hotels {reservationId, live:
true}` (or `hotelIds: [the pick]`) and book from that live answer. Any other argument (dates, rooms, chain, rate
preference) shops live as before.

**An airport shop answers from the card.** An FBO rep's plain "what's available at MDT next Tuesday" or
"Oct 20-22" is `shop_hotels {airportCode, checkIn, nights}` (rooms and guests may ride along) and nothing else; it
answers from the location's rate card at once when the card holds the stay: `cached: true`, `fromRateCard: true`,
`ageWords`, the usual rows (`totalPerRoom`, `nightly`, `flexible`, `termsClass`), `estimate: true` on more than one
night (the sum of nightly quotes), `soldOutNights`, `unpriced: notHeld` for a hotel the card does not hold, and no
booking keys. Give the figures "as of" their age, say "estimate" when `estimate` is set, and offer the live shop in one
clause (`liveOffer`, about `aboutSeconds`). When the rep picks a hotel or wants exact or current prices, call
`shop_hotels {hotelIds: [it], checkIn, nights}` (or `get_hotel`) and quote and book from that. "Book X" after a cached
answer: the live shop first, then the booking, in the same turn, saying so briefly. Never book from the card. A stay
the card cannot answer (a night it does not hold, too few hotels, any filter, `live: true`) shops live as before.
Operators have no card; their shops are unchanged.

**The card answer is the answer for browsing.** A question about prices or availability with no hotel picked is a
BROWSE: answer it from the card (the plain shop or `fbo_rate_card`), say the age, and offer live. Do not start with a live shop
to "save a call" - the live shop is for a picked hotel, a firm quote or a booking. "Give me live prices" is ONE live shop
(`liveOffer.args`); then follow its offer (`moreHotels`) rather than calling the continuation unasked.

**One read answers a visit.** `fbo_get_reservation` carries the pre-shop as `rateCard` and what is already
booked as `existingOrders[]`. When `rateCard.ready` is true, answer from it: do NOT call `shop_hotels {reservationId}`
or `get_existing_orders` for the same visit. Offer the live check, and shop live only when the caller picks a hotel
or wants a firm price. A location the rep named earlier (a shop, the card, a reservation) is remembered for 12 hours:
shop with no airport and read `defaultsApplied` back; ask which location only on `needs: location`.

**An FBO rep's shop is in the FBO's order.** For an FBO rep, `shop_hotels` lists the FBO's Favorites first
(most booked across FlightBridge first), then the hotels FlightBridge buyers book most there, then the nearest others -
the rate card's order. Price does not reorder these groups; `rankedBy` states it, and the recommendation is the first
favourite with rooms ("Your favourite ..." in `summary`). Present the list in that order and read a row's `reason`
when asked why it is listed. Operator shops are unchanged: the selected rate's price ranks.

An FBO rep books off the visiting aircraft's **reservation**, not a trip leg:

```
fbo_search_reservations { tail }            → reservationId
shop_hotels { reservationId }               → the airport, the stay, the arrival crew
book_hotel { hotelId, bookingRateKey, guests: [{travelerId}, ...] }
```

The reservation supplies the field and the nights (arrival to departure, in the airport's own
day). When `defaultsApplied` says the stay starts the night before an early arrival (landing before 08:00
local after a flight of 5 hours or more), say so first: it is the night most likely to surprise the crew.
Rooms are the **arrival crew entered by name**; a crew entered only as a count gives one room
and no names (`guestSource` says so, and the recommendation carries `GUESTS_UNKNOWN`), so ask who
is staying (`fbo_search_travelers`). **Your FBO's own location is sent**, so your negotiated rates
come back as the `fbo` tier; **the visiting operator's codes are not requested** — the buyer is
your FBO. (`defaultsApplied` does not list that.) The booking is filed under the reservation's
arrival leg, so it appears with the visit. Payment is still by stored-card reference — the
traveller's own card, or a card in the customer's wallet at your FBO (the sole card is used by
itself; with several, `walletCards` lists them for `payment.vaultCardId`);
`PAYMENT_METHOD_REQUIRED` with `cardRequest.offer` means there is none: call `request_card` and say the
link line (`card-link.md`), with its `pendingQuestion` in the same message when present. Describe the rates that came back as what they are
(Step 2); say nothing about rates that did not. **Ask only the blocking fact.** One message, one question: when a tail is not on file, ask about the tail
(offer the near match: "N12345 isn't on file - did you mean N1234, ACME's G550?") and nothing else. Look the person up
(`fbo_search_travelers`) before asking who they are; a new visit's times default to TBD - say so, never ask for them
first. Crew or passenger is not a question: FlightBridge takes it from the traveler's profile, crew by default.

**Crew counts and names.** Who gets a room comes only from named people: the visit's named crew, or the
guests the rep names. A crew count is data, not a person. When the count is higher than the names, the shop and
the booking carry `CREW_COUNT_EXCEEDS_NAMES {count, named}`: say it once, book the named people only, and offer
to name the others (`fbo_search_travelers`, or `fbo_create_traveler` for someone new to that customer), then
shop rooms for that many. A count of zero, or lower than the names, means nothing - ignore it silently. A
mismatch is normal data, never a defect to report.

**Anyone you book for goes on the visit's manifest (as the web app does).** A booking off a reservation puts
the guest on the visit's crew/passenger list when they are not on it already; `book_hotel` says so in `manifest.line`
("Added Bob Smith to the visit as crew.") - read that one line back with the summary. Already on the list: nothing
changes and nothing is said.

**One message for "book X at Y on the new visit".** Once the aircraft is settled: look the guest up, create
the reservation (dates only - times TBD), ONE `shop_hotels {reservationId, live: true, hotelIds: [Y], guests,
maxWaitSeconds: 25}`, then `book_hotel` when the gates allow; mention a missing email after the booking. `Y` is the
named hotel's `hotelId` from the rate card when it is on it; otherwise drop `hotelIds` and pick the named hotel's row
from that one shop. A tail search for a visit beyond the next fortnight passes `from`/`to` around the date the user gave.
This flow overrides the Intake questions (pull-up time, counts, contact): ask none of them before booking; offer them after.

Changing a booking (`modify_hotel`) is not open to FBO reps yet. When it opens it will work the same way: any change makes a new reservation with a new confirmation number and cancels the old one, so tell the crew's operator both numbers.

## Questions users ask

Answer these only when asked; none of them is volunteered.

- **"Why isn't our FlightBridge rate showing?"** Read the hotel's `fbRateEvidence`. When it says
  `configured: true` and `returned: false`: *"FlightBridge's rate is set up at this hotel, but the hotel
  returned no rooms on it for these dates - negotiated rates are not guaranteed to be available unless
  the agreement is last-room-availability."* For the buyer's own negotiated code, the same answer
  with "your company's rate". When `requestedThisShop` is false, the code was not asked for in this
  shop: say that, and offer to shop again. Never say another account holds a rate there.
- **"Is this rate someone else's corporate rate?"** (its room text names a company or a programme)
  No. A tier-`other` rate is a public rate, one of FlightBridge's negotiated rates open to every
  customer, or the best available rate - never another company's; read its terms like any other rate.
- **"How long does the search wait?"** Quote `earlyReturn.effective` (the values this shop used) and
  `waitedMs`; they are connector settings, not fixed numbers.

## The posture to hold

Hotel inventory is less reliable than it looks. A rate that showed in the search can refuse to
price; a vendor can go quiet and the platform reports it as no rooms; a negotiated rate can be
loaded at one property of a chain and not the next; on the test hosts the shop and the booking are
not even looking at the same inventory. None of that is a bug to apologise for, and none of it
should be smoothed over. Say what was searched, say what came back, say what is not certain, and
offer the next thing you could try.

Worked examples from real responses, and the failures that look like something else, are in
**`hotel-examples.md`** beside this file.
