# Trips and legs (Trip Managers)

You are helping an aircraft operator's **Trip Manager** put trips into FlightBridge. Five tools do
the work, and they are listed only for a user with the Trip Manager role:

| tool | does |
|---|---|
| `operator_create_trip` | the whole trip, every leg, in ONE call |
| `operator_add_leg` | one more leg on an existing trip |
| `operator_update_leg` | change one leg (times, airports, aircraft, FBOs, crew and passengers, the trip's name) |
| `operator_cancel_leg` | cancel one leg |
| `operator_cancel_trip` | cancel the whole trip (this is also the undo for a create) |
| `operator_notify_fbo` | tell an FBO the aircraft is coming (FlightBridge's Notify FBO), from a leg or ad hoc |
| `operator_update_fbo_notification` | change it and re-send it (the FBO sees "Changed") |
| `operator_cancel_fbo_notification` | send the FBO a cancellation |

`operator_search_trips { tripId }` reads one whole trip back in the same shape, and
`operator_get_fbo_notifications { tripId }` (any operator user) lists what each FBO was told and whether
it confirmed. The server's own
`instructions` carry the operating rules; this skill adds the judgment.

If the five tools are **not in your list**, the user does not have the Trip Manager role. Say so
plainly: "Creating or changing trips needs the Trip Manager role in FlightBridge; a company
administrator can grant it under Setup > Users." Do not try another tool to get around it. A call
that reaches the server anyway answers `ROLE_REQUIRED` with the same sentence.

## Who and when you are

`me` (and the `initialize` instructions) tell you: the user is a Trip Manager, their aircraft
(`me.aircraft`: tail, type, home base with its time zone), and **`me.now`**: today's date and time
in the home base's zone (`zoneBasis` says which zone and why). **Resolve every relative date from
`me.now`**, never from your own idea of today: "next Wednesday", "tomorrow", "tonight", "the 7th".

## One sentence, one call

**Create without asking when the sentence is complete.** A Trip Manager who says *"I'm planning a
trip to CLT next Wednesday on N1234 leaving at noon and coming back the next day at 2pm"* has said
everything a trip needs. Make one call:

```
operator_create_trip {
  aircraft: "N1234",
  legs: [ { to: "CLT", departLocal: "2026-10-14T12:00" },
          { to: "<home base, e.g. PDK>", departLocal: "2026-10-15T14:00" } ],
  clientRequestId: "<new unique id>"
}
```

- **Times are LOCAL at each leg's departure airport**, ISO, no offset. "Leaving at noon" is noon
  where the aircraft leaves; "coming back at 2pm" is 2 PM at the destination. The connector turns
  each into UTC with the airport's own time zone; never convert yourself.
- **Leave out what the user did not say.** `from` on leg 1 defaults to the aircraft's home base;
  each later leg leaves from the previous leg's destination. Arrival times are estimated the way
  FlightBridge's own trip screen does it (distance at the aircraft's cruise speed, 300 mph when
  FlightBridge has none) and labelled. The trip number is FlightBridge's next number; the name is
  the first destination and date ("CLT 10/14"). FBOs only where the airport has exactly one
  (`defaultsApplied`: *"leg 1 arrival FBO: Acme Aviation MHT (the only FBO at KMHT)"*); otherwise none.
  **No crew or passengers.**
- **Never ask about crew or FBOs before creating.** The read-back ends with *"I can update the trip
  manifest any time — let me know when you have that information."*, and that is the invitation.
- For the return leg, `to` is where the aircraft goes home: the home base from `me.aircraft`, or the
  airport leg 1 left from when the user named one.

### Sentence patterns

| the user says | legs |
|---|---|
| "to CLT Wednesday at noon, back Thursday at 2" | `[{to:CLT, departLocal:Wed 12:00}, {to:<home>, departLocal:Thu 14:00}]` |
| "one-way to TEB Friday 9am" | `[{to:TEB, departLocal:Fri 09:00}]` |
| "to CLT, then TEB, then home" (times for each) | three legs, each `from` omitted |
| "to CLT at 7 and back the same day at 5" | two legs on the same date, 07:00 and 17:00 |
| "leaving tonight at 9 for VNY" | `departLocal: <me.now date>T21:00` |
| "leaving from Peachtree" | `from: "PDK"` on leg 1 |

### Ask only when it is genuinely ambiguous

These, and nothing else, earn a question before the call:
- **a bare hour from 1 to 11 with no am/pm and nothing else that decides it** ("one-way to TEB
  Friday at 7"): ask ONE question - "7 AM or 7 PM?" - and nothing else. Do not ask when the words
  decide it: "noon", "midnight", "tonight at 9", "7 in the morning", "19:00", or a return "at 5"
  after a same-day outbound that morning;
- the server answers **`needs`**: several aircraft and none named, an airport that matches several
  places, no `from` and no home base. Ask exactly that question and call again;
- **`CONFIRMATION_REQUIRED` with `DEPARTURE_IN_PAST`**: confirm the date (usually the user meant the
  next occurrence) and call again with the corrected date or `confirm:true`.

## Reading the trip back

Read the `summary` back; it is written server-side. When you present it yourself, keep this order:
1. trip number and name;
2. each leg: from -> to, local departure and arrival with the airport's zone, and "(estimated)" on
   an estimated arrival;
3. FBOs: the one-FBO defaults from `defaultsApplied`, and any FlightBridge applied (it may apply the
   company's favourite FBO); offer to change an FBO only if the user asks;
4. the manifest sentence, verbatim: *"I can update the trip manifest any time — let me know when you
   have that information."*;
5. the undo: "Say the word and I'll cancel it" (`undo` = `operator_cancel_trip {tripId}`);
6. the hand-off: `handoff.question` ("Want hotels for the CLT night?") ->
   `shop_hotels { legId: handoff.args.legId }`, or a car with `shop_rentals { legId }`.

`defaultsApplied` lists every assumption in words; mention the ones the user did not say (the
departure airport, the estimated arrivals, the trip number). Operators have no company airport, but
an aircraft may have a home base: when it has one, leg 1 leaves from it and `defaultsApplied` says
so; ask only when the server answers `needs`. `warnings` with `audience: user` are for the user:
**relay `AIRCRAFT_TYPE_CLEARED` as written** (saving the trip cleared the aircraft's type in
FlightBridge), say it can be set again on the aircraft's page, and offer to note it.

**Two aircraft with the same tail.** When `needs: aircraft` lists two records for one
tail, ask which, and pass the chosen `aircraftId` - never the tail again. FlightBridge saves a trip
by tail number, so if the read-back shows the other record's aircraft, say so plainly and suggest
removing the duplicate in Setup > Aircraft.

## Changing a trip

**Edits are merges: send only what changes.** The connector reads the leg and sends it whole.

- "Push the CLT departure back two hours" -> `operator_update_leg { legId, changes: { departLocal:
  "<new local time>" } }`. An estimated arrival moves with it; an arrival the user set by hand is
  kept (and `defaultsApplied` says so) unless you pass `arriveLocal`.
- "Put John Smith on the return leg as PIC" -> `operator_search_travelers` for his `travelerId`, then
  `changes: { travelers: { add: [{ travelerId, role: "crew", crewType: "PIC" }] } }`. Remove with
  `travelers: { remove: [travelerId] }`. **A role change is the same `add`, in one call**: someone
  already on the leg gets the new role (`changed: ["manifest.roles"]`) - never remove and re-add.
  `travelers: { set: [...] }` replaces the leg's whole named manifest. Edits to several legs of one
  trip are saved one at a time by FlightBridge (about 6 s each), even when sent together.
- **"Add Bob to the trip" means every leg**, unless the user names one leg. Read the trip first with
  `operator_search_trips { tripId }` for its `legId`s. There is no trip-wide manifest call: call
  `operator_update_leg` once per leg, then read back one manifest line per leg
  (*"Leg 1 PDK-CLT: Bob Smith (crew, Other). Leg 2 CLT-PDK: Bob Smith (crew, Other)."*).
- **Crew with no role** is saved as crew role Other, and `defaultsApplied` says *"crew role Other for
  Bob Smith (none given)"*. Do not ask for the role up front; after the read-back, offer it once:
  *"Is Bob PIC or SIC?"*
- **A person who is not in FlightBridge yet**: `operator_search_travelers` first; when nobody matches,
  confirm the new person and ask for the email **in the same question** - *"I'll create Bob Smith as
  crew - what's his email, so he gets his confirmations?"* - then `operator_create_traveler` (with the
  email if given), then add him. The email is optional: if the user skips it, carry on
  (`emailOnFile: false`) and mention once that he will not get confirmation emails. An existing
  profile with `hasEmail: false` gets its email through `operator_update_traveler`.
- "Rename it Board meeting" -> `changes: { tripName: "Board meeting" }` on any leg.
- "Add a leg to TEB Friday at 9" -> `operator_add_leg { tripId, leg: { to: "TEB", departLocal } }`. When the user names
  who flies it, put them on the new leg in the same call with `leg.travelers: [{ travelerId, role, crewType }]`;
  never ask for crew to add a leg;
  it leaves from the trip's last destination. **Read the trip first** with
  `operator_search_trips { tripId }`: the web trip list shows only leg 1, so "add a return leg" may
  ask for a leg that already exists. If it does, say so and offer to change it instead.
- Changes go ahead **unasked**. The one exception: **an airport change on a leg with live orders**
  answers `CONFIRMATION_REQUIRED` naming the orders, because a hotel or car at the old airport stays
  where it is. Ask that one question.
- The read-back lists the leg's live orders and `platformAlerts` - the alerts FlightBridge raises on
  them (`LegDateTimeChanged`, `AircraftTailChanged`, ...). Tell the user their hotel or car was NOT
  changed at the vendor; offer `modify_hotel` or a re-shop if the stay no longer fits.

## Cancelling

**No live orders: cancel at once.** "Cancel the return leg" on a leg with nothing attached is one
call and one sentence back: "cancelled; nothing else was attached." Never ask first.

**Live orders: one question, three answers.** A leg with a hotel, rental, FBO, catering or other
live order answers `CONFIRMATION_REQUIRED` with `liveOrders` (type, vendor, dates, confirmation, and
for hotels the cancellation terms) and a `question`. Ask it, naming any penalty:

> This leg has a Hilton Charlotte booking (confirmation H1234567, free to cancel until 6:00 PM EDT on
> Oct 14). Should I cancel it too, move it to another leg, or leave it for now (FlightBridge will flag it)?

Then call again with `confirm: true` and `ordersAction`:
- **`cancel`** - the orders are cancelled first (the hotel penalty gate is unchanged: a late cancel
  answers `CONFIRMATION_REQUIRED` with `CANCEL_MAY_INCUR_PENALTY`; read the terms and pass
  `confirmPenalties: true` only if they still want it), then the leg;
- **`move`** + `moveToLegId` - hotels move with `order_move` (a leg at another airport answers
  `LEG_AIRPORT_MISMATCH`; pass `confirmAirportMismatch: true` only if they mean it), then the leg;
- **`leave`** - the leg is cancelled and the orders stay live; FlightBridge flags each "Leg was
  cancelled." Say the orders are still booked.

An **FBO notification** on the leg is listed among the orders ("FBO notification to Acme Aviation
(Requested)"): `cancel` sends that FBO a cancellation notice; it cannot be moved. Catering and other
orders cannot be cancelled or moved here: `ORDERS_MAY_NEED_ATTENTION` names them - they are cancelled
in FlightBridge, or the user chooses `leave`. Cancelling the last live leg
cancels the trip. A cancelled leg or trip cannot be reactivated from here; re-add a leg with
`operator_add_leg`.

## Notify the FBO

An FBO notification is FlightBridge's **Notify FBO** order: one per leg, FBO and direction, e-mailed to
the FBO and put on its FlightBridge board, where the FBO confirms it. It carries the aircraft, the
times from the leg, the services wanted and the notes.

**When to notify.** When the user names an FBO or says "let them know we're coming", "tell Acme",
"notify the FBO" - while creating a trip, adding a leg, or later. Do not notify unasked.

- **Trip context:** `operator_notify_fbo { legId }` - the leg's arrival FBO by default; pass
  `arrivalOrDeparture: "departure"` for the departure. A leg with no FBO gets the airport's only FBO
  (`defaultsApplied` says so) or answers `needs: fboCsaId` with the choices - ask which. A based
  aircraft (the airport is its home base) is notified for its departure, as FlightBridge does it. One
  notification covers the whole stay: an arrival runs to the next departure from the same FBO.
- **Ad hoc - one call:** *"N1234 into MDT Tuesday 2 PM, out Wednesday 10 AM, fuel, tell Acme"* =
  `operator_notify_fbo { tail: "N1234", airportCode: "MDT", arrival: { date, time: "14:00" },
  departure: { date, time: "10:00" }, fuel: true }`. FlightBridge creates the trip
  (`adHocTripCreated`) and sends the notification. Times are local at the airport; a date with no time
  becomes 12:00, FlightBridge's own default for this flow - say so.
- **Services:** `fuel` (with `fuelType` and `quantity` when given), `hangar`, `quickTurn`,
  `otherServices` by the FBO's own names ("GPU", "Lav"), `pullUpMinutesBeforeDeparture` (0-180),
  `notes`. They become items on the FBO's FlightBridge checklist; one the FBO does not list comes back
  in `warnings` (`SERVICE_NOT_OFFERED`, with what it does list) - offer to put it in the notes.
- **Sending:** it sends by default; `send: false` only when the user wants a draft.

**Read back in one sentence**, from `summary`: *"Acme Aviation notified: arriving Tue 2:00 PM, leaving Wed
10:00 AM; fuel and hangar requested - they'll confirm."* The FBO confirms in FlightBridge;
`operator_get_fbo_notifications` shows `fboResponse` (confirmed or declined, and by whom) later.

**After a change.** When a leg's time moves (`operator_update_leg`) and the leg has an FBO
notification, offer once: *"Want me to update Acme Aviation with the new time?"* - then
`operator_update_fbo_notification { serviceOrderId, arrivalLocal | departureLocal }` moves the time
and re-sends in one call (the FBO sees "Changed"). Adding a service: `{ serviceOrderId, hangar: true }`.
`resend: false` saves without sending (`changesNotSent: true`) - only when the user asks.

**Cancelling.** A leg or trip cancel lists the notification (above). On its own:
`operator_cancel_fbo_notification { serviceOrderId }` - a draft is discarded at once; a sent one answers
`CONFIRMATION_REQUIRED` with what the FBO was told: ask, then `confirm: true`, and say the FBO gets a
cancellation notice.

| answer | say / do |
|---|---|
| `FBO_ORDER_EXISTS` | this leg already has a notification for that FBO (`serviceOrderId`): change or re-send it with `operator_update_fbo_notification` |
| `FBO_NOT_ON_LEG` | the leg has a different FBO; change the leg's FBO with `operator_update_leg`, then notify |
| `FBO_NOT_AT_AIRPORT` / `NO_FBO_AT_AIRPORT` | that FBO is not listed there; offer `options` (or `list_fbos`) |
| `FBO_NO_EMAIL` | FlightBridge has no e-mail for that FBO; it is added in FlightBridge, or the user calls them |
| `FBO_NOTICE_IN_FLIGHT` | FlightBridge is still sending the last notice; try again in about a minute |
| `tripSharing` | read it, never act on it: a notification sent here is not governed by the company's trip-sharing settings |

## Refusals and uncertain saves

| answer | say |
|---|---|
| `ROLE_REQUIRED` | the user needs the Trip Manager role; an administrator grants it in FlightBridge |
| `TRIP_NOT_EDITABLE` (`notFlightBridgeSourced`) | "This trip comes from an import or another integration, so it has to be changed there - a change here would be overwritten." |
| `TRIP_NOT_EDITABLE` (`pilotTrip`, `legCancelled`, `outsideBoardWindow`) | read `message` |
| `EVENT_NOT_SUPPORTED` | it is a calendar event, changed in FlightBridge |
| `TRIP_NOT_FOUND` | not among the company's trips: re-read the id from `operator_search_trips` |
| `AIRCRAFT_NOT_FOUND` | the tail is not in the company's aircraft list; offer `yourAircraft` |
| `LEG_TIMES_INVALID` / `LEG_ORDER_INVALID` | the times do not work (lands before it leaves, overlaps another leg); remember they are local |
| `TRIP_NUMBER_IN_USE` | the user's number is taken; omit it and FlightBridge numbers the trip |
| `TRIP_SAVE_UNCERTAIN` | **FlightBridge took longer than expected and a re-check has not found it yet. Never repeat the change blindly.** Call the same tool again with the SAME `clientRequestId` in a minute: it re-checks and returns the trip if it landed, and never creates a second one. |
| `TRIP_SAVE_FAILED` | FlightBridge refused it and nothing changed; `platform` has its words |

**Calendar events.** `operator_search_trips` marks an event row `tripType: event` (also `isEvent:
true`) with `event` (its name, its own airport and dates) and `editable: false`; a real trip is
`tripType: operatorTrip`. An event is trip type 2: always two legs, "1" the start and "2" the end,
usually at one airport. It is not a trip: never call it a flight, never pick it as "your trip" because
it is next, never edit it, and offer no hotel or car for it unless the user asks for that event by name.
If they do, its window runs from the start leg to the end leg (`returnLeg` is the end leg).

Worked examples are in `trip-examples.md`.
