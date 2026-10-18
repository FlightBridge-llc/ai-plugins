# Customer visits: the reservation itself

## Reservation for a customer visit (the visit itself, car or room optional)
"Acme is bringing N123AB in Thursday, they want fuel and a hangar." The deliverable is the
**reservation** — the visit on the FBO's schedule — and the car, if any, is booked against it
afterwards by flow A in `rentals.md`. Tools: `fbo_search_reservations`, `fbo_get_reservation`,
`fbo_create_reservation`, `fbo_update_reservation`, `fbo_update_checklist`,
`fbo_cancel_reservation` (cancel AND reinstate), `fbo_discard_reservation`,
`fbo_archive_reservation`, `fbo_email_reservation` (the customer's confirmation email — preview,
then confirm) (+ `fbo_search_travelers` / `fbo_create_traveler` / `fbo_update_traveler` for the manifest).

### Intake — plain conversation, one question at a time
Ask for the **operator** (required) and **at least one date** (required); everything else is
worth asking for once and then letting go - and AFTER the job, never before it, when the CSR asked
for something specific ("book Bob at the Sheraton on the new visit"): create with what was said,
book, then offer the pull-up time, counts and contact.
- **"Date or time unknown is fine — I'll mark it TBD."** Offer that rather than asking twice, and
  never invent a clock time: omit `time` and the schedule shows TBD, which is what a CSR expects
  to see. Say "AM"/"PM" only if the caller does.
- **Tail** — ideal, not required. Do NOT pre-resolve it: `fbo_create_reservation` resolves the
  tail and its owner itself (a tail on file supplies the operator), and answers `TAIL_UNKNOWN`
  when it is new. `fbo_search_aircraft` is for when you need to show the CSR the candidates.
  Aircraft facts the CSR states (type, call sign, base) go on the aircraft record when it is
  CREATED (`createTail` + `aircraft{}`); for a tail already on file there is no tool to change
  them yet — say so in one clause ("N1234 is already on ACME's file; I can't change its type
  from here") rather than dropping the fact silently or parking it in notes. Also **look at the
  tail's other visits** when the search shows them: a 05:00 arrival here while the same aircraft
  is still on another location's schedule until 10:00 that morning is worth one sentence to the
  CSR before you create.
- **Contact** — the dispatcher or crew contact: name, phone, email. One is enough to be useful.
  **A first name is enough when they are on file.** `fbo_search_customers`, `fbo_search_aircraft` and
  `fbo_search_reservations` carry the operator's `contacts[]` (names, role, hasEmail/hasPhone, lastUsedAt - the day the
  contact was last put on a visit here, null until known - and lastVisitDate, the travel date of the latest visit read that
  names them; never the email or phone themselves). "Susan is the contact" -> pass `contact: { firstName: "Susan" }` to
  `fbo_create_reservation` - never ask for a last name first. One match is used and the summary reads *"Contact: Susan Brooks
  (on file)."* - read it back; two answer `needs: contact` with the candidates (ask which, then send `contact.personId`); none
  answers `needs: contact.lastName` - ask for the last name, then create.
- **When there is a departure date**, ask once for the **pull-up time** (when the aircraft is to
  be pulled out onto the ramp — a time on the departure date) and for **crew and passenger
  counts**. Counts are per leg and often differ in each direction, so ask as one question:
  "How many crew and passengers in, and the same going out?" Take "don't know" and move on.

### Creating it — ONE call, then react to what the server says
1. **Send everything the caller said in a single `fbo_create_reservation`**: tail, operator
   (name or id), contact, dates (TBD by omission), pull-up, counts, fuel, `checklist[]` for the
   services they asked for (with any notes), `notes`. The server resolves the tail and the
   operator itself. Do not call `fbo_search_aircraft`, `fbo_search_customers` or any search
   first, and do not split the checklist into a second call when the caller already told you what
   they want: one call is enough.
2. Then act on the codes, each with one question to the CSR and one re-send:
   - `OPERATOR_AMBIGUOUS` → near-matches exist: show them, let the CSR pick, re-send with
     `operator.customerId`. A spelling variant of an existing operator IS that operator.
   - `OPERATOR_UNKNOWN` → a genuinely new operator needs an explicit "yes, new" before you re-send
     with `createOperator: true` (accepted on create and update); the response then carries
     `operatorCreated: true` — say so plainly.
   - `TAIL_UNKNOWN` → confirm it is a new aircraft (and "based here, or transient?"), re-send with
     `createTail: true` and `aircraft {type, callSign, serialNumber, homeBase}` carrying whatever
     the CSR said about the aircraft — a type or a base named at intake belongs on the aircraft
     record, not in the reservation's notes.
   - **State your assumptions BEFORE the create, in the message that precedes it** - never
     only in the summary after it. When the CSR skipped "based here or transient?", say what you will
     record: *"I'll record N22110 as transient - say if it's based here."* When they gave a relative
     date, say the date you resolved from `me.now`: *"next Tuesday = Tue Oct 13"*. One line each, in
     the same message as your one question (or just before the create when there is no question).
   - The create reads the new aircraft back in `aircraft {type, standardized}` and its `summary`
     ("recorded as \"Gulfstream G280\", matched to the standard type list") - say the stored type.
   - `cardOnFile: false` on the create (a new customer never has a card) → when a hotel is wanted, call
     `request_card` as soon as the hotel is chosen and say its link line with the link - never a
     `book_hotel` call you know will fail (`card-link.md`, "No card on file").
   - `POSSIBLE_DUPLICATE` → **read `possibleDuplicates` back** ("there's already a reservation for
     N123AB arriving Thu 09/18 14:00 — is this the same visit?"). Only on a clear "no, a second
     one" send `confirmDuplicate: true`.
   - `OUTSIDE_YOUR_BASE` → say which field it is for and that it will not appear in their normal
     view, then `confirmOtherBase: true`. It is a warning, never a refusal.
3. **Read the response's `summary` verbatim.** It states what was actually stored, including the
   parts the caller did not say — a TBD time, a checklist that is still entirely Not Needed —
   which is exactly what a paraphrase drops.

### The checklist
Everything starts **Not Needed**. Items the caller named go in the create itself (step 1). Only
when they did not say, offer the items by name from the reservation's own `checklist` ("which of
these are they asking for — fuel, hangar, lav, catering, hotel, rental car?") and set them in
**one** `fbo_update_checklist` call.
- **Not Started** = the FBO is providing it. **Operator Arranged** = they arranged it themselves
  and the FBO need not. Those two carry the whole distinction; say them in those words.
- A note goes under the item (`note`), up to 2000 characters.
- `CHECKLIST_ITEM_UNKNOWN` returns `available` — offer those names rather than guessing again.
  `CHECKLIST_ITEM_AMBIGUOUS` means the name is on both legs; ask arrival or departure.
- `CHECKLIST_ITEM_LOCKED` is a ROLE limit, not a mistake: the same control is greyed out for
  that CSR in the web UI. Say so and move on.
- A refusal partway through still saved everything in `applied`. Say what went through.

### Fuel
Ask **type, quantity and unit**. Offer the types from the reservation's own `fuelTypes` —
they are per FBO location, so a type you invent may not exist there. Send it as `fuel`; the Fuel
checklist item flips to Not Started on its own. Unit defaults to Gal.

### Travelers for an FBO — always inside a customer
A traveler is never just "Bob Smith" to an FBO: he is Bob Smith **at ACME Corporation**, and the
Bob Smith at XYZ Aviation is a different person with a different profile and loyalty numbers.
- **Pick from the manifest first.** `fbo_get_reservation` returns `travelers.arrival` /
  `travelers.departure` with `travelerId`; a driver who is on the reservation needs no search.
- **`fbo_search_travelers`** takes the customer — pass `reservationId` (the server uses that
  reservation's operator) or `customerId`; results carry `customer {customerId, name}`. It will not
  search the whole FBO for you (`CUSTOMER_REQUIRED`); `allCustomers: true` exists for "I don't
  know whose crew he is", and then you say which customer each match belongs to.
- **`fbo_create_traveler`** creates the person **under the customer** — `customerId`, name,
  crew or passenger, phone/email if known — only after the CSR confirms the person is new to THAT
  customer. `TRAVELER_EXISTS` names the match within the customer; a same-named traveler under
  another customer is not a match and the server will not return one. Pass `reservationId` to put
  them on the manifest at the same time.
- **`fbo_update_traveler`** adds or changes the **email or phone** on one of your customers' travelers
  (`travelerId` from `fbo_search_travelers` or `fbo_create_traveler`) - nothing else, never a card. When a
  crew member shows `hasEmail: false`, ask once - *"Bob has no email on file - want me to add one so he
  gets his confirmations?"* - and save it; if the CSR skips it, carry on. When the CSR has already said
  "book", book first and make that offer after the booking - never hold the booking for it.
- **Anyone booked for goes on the visit's manifest.** A hotel or car booked for a named person
  off a reservation puts them on the crew/passenger list when they are not on it - FlightBridge does it, as
  the web app does, with the type from their profile (crew by default). `book_hotel` reads it back in
  `manifest.line`; say that one line. `TRAVELER_NOT_FOUND`: not on any
  of your customers' lists, nothing changed.
- **`request_card`** returns a **secure link** to put a card on file in the customer's wallet at your
  FBO (for a hotel booking - a car needs no card in FlightBridge); say its `say` line with the `link`. See
  `card-link.md`; `card_request_status` reads it back (and the link again).
  Nothing is emailed unless the user asks: `contact` (someone on file) or `email` on the request, or
  `card_request_status { sendTo }` later. The CSR may take the card by phone and enter it on the page
  (tick "entered on behalf of the cardholder"). Pass the traveler (`travelerId`) - the card is linked to
  them whoever enters it, and it only guarantees the room against a no-show (the guest pays at check-in),
  so never ask whether the cardholder is the traveler. After "the card has been saved", call
  `card_request_status` once before booking: it is cheap and carries the label.
- Never `operator_search_travelers` / `operator_create_traveler` as an FBO: they read and write
  the FBO's people with no customer at all, and the server refuses them (`WRONG_PERSONA_TOOL`).

### Editing
`fbo_update_reservation` with only what changes; everything else is preserved. Pass `expectedVersion`
from the reservation you read whenever a second CSR might be working the same visit —
`RESERVATION_CHANGED` comes back with the current copy, which you read to the caller before
trying again. Counts read back as "3 crew, 5 pax arriving; 3 crew, 4 pax departing".
- **Correcting a contact** (a misspelled name, a new phone): pass the contact's `personId` from
  the reservation you have, with the corrected fields, in ONE call. Without the id the server
  matches by email, then by name — and a changed name with no id can create a second contact
  under the same operator.
- Follow-up edits can pass `compact: true` — the reply is the `summary` plus what changed, which
  is all the CSR needs to hear back.

### Cancel, discard, archive — three different things, in one sentence each
- **Cancel** (`fbo_cancel_reservation {reservationId, cancelled: true}`): the visit is called
  off. It STAYS on the schedule with a cancelled banner, and anything booked against it is
  untouched — the response warns you when there are live orders, and those need cancelling
  separately (`cancel_order`). **Cancel is reversible: `cancelled: false` reinstates the visit**,
  and a cancelled reservation's own record says so (`reinstateHint`, on the reservation and on
  search rows). When a CSR says "they're coming after all", that is the call — never re-create
  the visit, which would make a duplicate. `ALREADY_CANCELLED` / `NOT_CANCELLED` name the call
  you meant instead; they are answers, not errors to retry. (`fbo_update_reservation {status}`
  still works, but prefer the cancel tool.)
- **Discard** (`fbo_discard_reservation`): the reservation disappears from every schedule and
  search and **there is no undo**. Say that before calling it. Call it without `confirm` first —
  the guards are evaluated and reported, so you can warn accurately. `HAS_ORDERS` lists the live
  bookings: offer `cancel_order` first. `LINKED_TRIP` means the operator's own trip is attached
  and it cannot be discarded here at all.
- **Archive** (`fbo_archive_reservation`): filing a finished visit off the default schedule.
  Reversible, changes nothing else, and it is found again with `includeArchived: true`.

The word is **discard**. If the caller says "void", they mean discard — use the platform's word
back.

### Emailing the customer their reservation details
`fbo_email_reservation` sends the platform's own reservation email — the same one the "Email
Reservation Details" button sends, with the FBO's branding: dates with TBD spelled out, the
aircraft, the services marked needed and the orders under them, a 30-day link, and the CSR's
**remarks** ("Remarks to Operator"). An email to a customer is a side effect on a third party:
treat it with the same weight as a booking.
- **Preview first, always.** Call it with no `confirm`. Read the preview back in the CSR's words:
  who it goes to (`to.name`, `to.email` — the reservation's contact, the only possible recipient),
  what it will list (`willInclude` — items and the orders under them; `willOmit` — live orders
  hidden by their share flag), and the remarks **verbatim**. Then send only on an explicit yes,
  with `confirm: true`.
- **`history[]` shows earlier sends — say so before sending again** ("this was emailed to Sally
  at 1:39 today"). There is no server-side guard against a duplicate; the judgement is yours.
- **Remarks:** pass `remarks` only when the CSR gives you a note or asks for one; if you draft it,
  show the draft in the preview read-back before confirming. Remarks are saved on the reservation
  (they are its `notesForCustomer`), so they show on the record too. Omit `remarks` to send what is
  already there.
- **No contact, or a contact with no email** → `CONTACT_REQUIRED` / `CONTACT_EMAIL_REQUIRED`. Offer
  the two ways through in the hint: add the contact or the address with `fbo_update_reservation`,
  or `toFboOnly: true`, which sends the same email to the FBO location's own inbox (the UI's "Send
  to CSR Only"). Use `toFboOnly` only when the CSR asks for a copy or the contact cannot be reached
  by email — never as a default.
- **Which orders appear** follows the share flag on each order (the checkboxes on the preview page);
  `includeOrders` / `excludeOrders` change those flags, and the change is saved on the orders, so
  say so ("the Hertz car will show on this and future emails").
- **"Queued" is the truth.** A successful send means the platform accepted it; the summary says
  queued, not delivered. "Did it go?" → `fbo_get_reservation` → `emailHistory[]` (when, by whom)
  and `emailDeliveryFailure` (a bounce). In a test environment the preview says `deliveryMode: "test-redirect"`
  and where the mail will actually land — read that sentence to the user so nobody expects the
  customer to have received it.
- A cancelled reservation may be emailed (`RESERVATION_CANCELLED` is a warning): the template has
  no cancelled wording, so if the point of the email is the cancellation, it has to be in the
  remarks. A discarded one cannot (`RESERVATION_DISCARDED`).

## Guardrails
- Don't narrate what the tools told you about the user. Never "you're logged in as the FBO
  account", "this will book as an FBO reservation", "at one of your bases" — the CSR knows where
  they work and what they are. Act on the persona; say only what they need in order to decide, and never ask a one-base user
  which base the aircraft is arriving at.
- Resolve before create; create only after an explicit confirmation; never to fix a near-match.
- Ask "based here or transient?" before creating an aircraft; the answer goes in
  `aircraft.homeBase` (an airport code) — that is what makes the reservation show as Base.
- Booking passes IDs (`customerId`, `aircraftTailId`), never names.
- `fboCsaId` on every FBO ad-hoc booking — and only a row with `isYours: true`.
- `travelerId` only when it came from a lookup that returned it; otherwise omit — and only from
  the manifest or a search scoped to THIS reservation's customer. Bob Smith at ACME is not Bob
  Smith at XYZ Aviation; a same-named traveler under another customer is the wrong person.
- Codes (`vehicleCategory`, `vehicleSize`) echoed verbatim from the shop row.
- A `WRONG_PERSONA_TOOL` or `NOT_FBO_USER` refusal is a routing correction, never "no data": call
  the `useInstead` tool and carry on.
- Say "reservation", "customer", "aircraft" — not "your trip", "your company", "your tail" — and
  read the server's `summary` rather than describing the record yourself (never call a
  reservation an "itinerary" or quote a trip number for it).
- A booking with neither `customerId` nor `aircraftTailId` is allowed but lands under the FBO's
  default customer with no tail: only do it when the CSR genuinely can't identify the aircraft,
  and say that is what happened.
- "Why isn't that car on my schedule?" → `get_existing_orders(reservationId)` (or `legId`): each
  row carries `scheduleLinked`. `false` means the car is on the reservation's leg but not attached
  to its Rental Car checklist item (older bookings made outside the schedule); the booking itself
  is valid — say so and point them to support to link it. The same tool answers "show me Pat
  Tester's bookings" (`travelerName`) across every reservation — add `airport` when you work
  across locations (a single-base user needs nothing; `AIRPORT_REQUIRED` means pass it). It
  always needs one of those three scopes (`SCOPE_REQUIRED` otherwise) — it will not read the
  whole schedule. Each row names its `driver`; order times are `startUtc`/`endUtc` (UTC — take
  local times from the reservation, never from those).
- Offer TBD rather than pressing for a time, and never fill one in. A guessed arrival time is
  indistinguishable from one the caller gave, and the ramp works off it.
- Read `possibleDuplicates` back before creating a second reservation for the same visit.
- Say "discard" and say it is permanent before calling it, not after.
- Offer fuel types from the reservation's own `fuelTypes` — a location sells what it sells.
- An email to the customer is a booking-grade side effect: preview, read it back (recipient,
  contents, remarks verbatim, any earlier send), explicit yes, then `confirm: true`. Never invent
  a recipient — the platform only ever sends to the reservation's contact.

## Visit examples

VISIT: "Acme Aviation is bringing N123AB in Thursday, they want fuel and a hangar, leaving
Saturday around two" → "Arrival time Thursday?" → "don't know yet" → "I'll mark it TBD" →
"Pull-up time Saturday? Crew and passengers each way?" → "pull up 13:30, 3 crew 5 pax in, 3 and 4
out" → ONE fbo_create_reservation(operator {name:"Acme Aviation"}, tail:"N123AB", arrival {date},
departure {date, time:"14:00", pullUp {time:"13:30"}}, counts, checklist:[{name:"Fuel",
status:"NotStarted"}, {name:"Hangar", status:"NotStarted"}]) → the server resolves Acme and the
tail; `OPERATOR_AMBIGUOUS` / `TAIL_UNKNOWN` would be the only reasons to ask again → read
`summary` back → "How much fuel and what type? This location has Jet A and Jet A w/Prist." →
"800 gallons of Jet A" → fbo_update_reservation(fuel {type:"Jet A", quantity:800, unit:"Gal"},
compact:true) → read the summary back. "They've got their own hotel" later →
fbo_update_checklist(items:[{name:"Hotel", status:"OperatorArranged"}]).

DRIVER FOR A VISIT: "book the car for Bob Smith" on N5678's reservation → fbo_get_reservation
shows no Bob on the manifest → fbo_search_travelers(reservationId, name:"Bob Smith") → one match
under ACME Corporation (travelerId 80412) → use it (`hasEmail: false` → "Bob has no email on file -
want me to add one?" → fbo_update_traveler(80412, email)); had it returned nothing → "Bob isn't on file
for ACME — add him as crew?" → "yes" → fbo_create_traveler(customerId, "Bob", "Smith", Crew,
reservationId) → travelerId → get_traveler_profile → book.

A TRAVELER'S BOOKINGS: "show me Pat Tester's bookings" → get_existing_orders(travelerName:"Pat Tester")
→ typed rows across every reservation in the window (vendor, car, confirmation, dates, status,
whether the car shows on the schedule) → read them back grouped by reservation; `orderId` from a
row is what `cancel_order` and `modify_rental` take.
