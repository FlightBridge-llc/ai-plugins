# Trips and legs (Trip Managers)

You are helping an aircraft operator's **Trip Manager** put trips into FlightBridge. Two tools do
the work, each with an `action`, and they are listed only for a user with the Trip Manager role:

| tool and action | does |
|---|---|
| `operator_manage_trip` `create` | the whole trip, every leg, in ONE call |
| `operator_manage_trip` `create_positioning` | a crew-positioning trip: one crew member travelling to or from the aircraft |
| `operator_manage_trip` `add_leg` | one more leg on an existing trip |
| `operator_manage_trip` `update_leg` | change one leg (times, airports, aircraft, FBOs, crew and passengers, the trip's name) |
| `operator_manage_trip` `cancel_leg` | cancel one leg |
| `operator_manage_trip` `cancel_trip` | cancel the whole trip (this is also the undo for a create) |
| `operator_manage_fbo_notification` `send` | tell an FBO the aircraft is coming (FlightBridge's Notify FBO), from a leg or ad hoc |
| `operator_manage_fbo_notification` `update` | change it and re-send it (the FBO sees "Changed") |
| `operator_manage_fbo_notification` `cancel` | send the FBO a cancellation |

`operator_trips { tripId }` reads one whole trip back in the same shape, and
`operator_fbo_notifications { tripId }` (any operator user) lists what each FBO was told and whether
it confirmed. For what was booked on past or future trips over a period, use `orders` without a leg
([orders.md](orders.md)). The server's own `instructions` carry the operating rules; this skill adds
the judgment.

If `operator_manage_trip` and `operator_manage_fbo_notification` are **not in your list**, the user
does not have the Trip Manager role. Say so
plainly: "Creating or changing trips needs the Trip Manager role in FlightBridge; a company
administrator can grant it under Setup > Users." Do not try another tool to get around it. A call
that reaches the server anyway answers `ROLE_REQUIRED` with the same sentence.

**When the user says the role was just granted**, call `me` again: it reads the account fresh every time, so
`me.roles.tripManager` is true at once. Your tool list only changes when your client lists the tools again, so when
`me.roles.note` is present, say it as written - *"The role is now on your account; refresh the FlightBridge tools in
your client (or reconnect) to use the trip tools."* - and carry on once the trip tools appear.

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
operator_manage_trip {
  action: "create",
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
- **Never ask about crew or FBOs before creating.** When nobody was put on the legs, the read-back ends with
  *"I can update the trip manifest any time — let me know when you have that information."*, and that is the
  invitation. When the user named crew, the summary reads back what was saved instead (*"Both legs: John Doe PIC,
  Robert Smith SIC."*) and says nothing about updating the manifest later.
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

An FBO name FlightBridge does not list at that airport does not stop the trip. The leg keeps the
name as text, FlightBridge maps it later, and `defaultsApplied` names the FBOs that are listed
there. Read that back and offer one of them if the user wants it set now. A name that matches
several FBOs still asks which.

## Reading the trip back

Read the `summary` back; it is written server-side. When the reply carries `links`, end with
`links.label`: "See the updated trip here: <url>" after a change, "Open it in FlightBridge: <url>" when
the user only asked about the trip ("tell me about my CLT trip next week"). Each row of
`operator_trips` carries its own `links.trip`: in a list of trips, the trip is the link, written as
`links.trip.markdown`, with no separate link column; the "Open it in FlightBridge" sentence is for a one-trip
answer. When you present it yourself, keep this order:
1. trip number and name;
2. each leg: from -> to, local departure and arrival with the airport's zone, and "(estimated)" on
   an estimated arrival;
3. FBOs: the one-FBO defaults from `defaultsApplied`, and any FlightBridge applied (it may apply the
   company's favourite FBO); offer to change an FBO only if the user asks;
4. the manifest, as the summary says it: the crew saved on each leg (*"Both legs: John Doe PIC, Robert Smith
   SIC."*), or, only when nobody is on the legs, *"I can update the trip manifest any time — let me know when you
   have that information."*;
5. the undo: "Say the word and I'll cancel it" (`undo` = `operator_manage_trip { action: "cancel_trip", tripId }`);
6. the hand-off: `handoff.question` ("Want hotels for the CLT night?") ->
   `shop_hotels { legId: handoff.args.legId }`, or a car with `shop_rentals { legId }`.

`defaultsApplied` lists every assumption in words; mention the ones the user did not say (the
departure airport, the estimated arrivals, the trip number). Operators have no company airport, but
an aircraft may have a home base: when it has one, leg 1 leaves from it and `defaultsApplied` says
so; ask only when the server answers `needs`. `warnings` with `audience: user` are for the user: relay them.

**Two aircraft with the same tail.** When `needs: aircraft` lists two records for one
tail, ask which, and pass the chosen `aircraftId` - never the tail again. FlightBridge saves a trip
by tail number, so if the read-back shows the other record's aircraft, say so plainly and offer to
delete the duplicate (`operator_manage_aircraft` action `delete`, [aircraft.md](aircraft.md)).

**A tail that is not on file.** When the user names a tail you have not seen, or a create answers
`AIRCRAFT_NOT_FOUND`, read the list with `operator_aircraft` before guessing; a genuinely new aircraft
is added with `operator_manage_aircraft` action `save` ([aircraft.md](aircraft.md)), then the trip is
created. When the user says an aircraft's type is missing or wrong after a trip change, setting it back
is one `save` with the type.

## Changing a trip

**Edits are merges: send only what changes.** The connector reads the leg and sends it whole.

- "Push the CLT departure back two hours" -> `operator_manage_trip { action: "update_leg", legId, changes: { departLocal:
  "<new local time>" } }`. An estimated arrival moves with it; an arrival the user set by hand is
  kept (and `defaultsApplied` says so) unless you pass `arriveLocal`.
- "Put John Smith on the return leg as PIC" -> `travelers` for his `travelerId`, then
  `changes: { travelers: { add: [{ travelerId, role: "crew", crewType: "PIC" }] } }`. Remove with
  `travelers: { remove: [travelerId] }`. **A role change is the same `add`, in one call**: someone
  already on the leg gets the new role (`changed: ["manifest.roles"]`) - never remove and re-add.
  `travelers: { set: [...] }` replaces the leg's whole named manifest. Edits to several legs of one
  trip are saved one at a time by FlightBridge (about 6 s each), even when sent together.
- **"Add Bob to the trip" means every leg**, unless the user names one leg. Read the trip first with
  `operator_trips { tripId }` for its `legId`s. There is no trip-wide manifest call: call
  `operator_manage_trip` action `update_leg` once per leg, then read back one manifest line per leg
  (*"Leg 1 PDK-CLT: Bob Smith (crew, Other). Leg 2 CLT-PDK: Bob Smith (crew, Other)."*).
- **Crew with no role** is saved as crew role Other, and `defaultsApplied` says *"crew role Other for
  Bob Smith (none given)"*. Do not ask for the role up front; after the read-back, offer it once:
  *"Is Bob PIC or SIC?"*
- **A person who is not in FlightBridge yet**: `travelers` first; when nobody matches,
  confirm the new person and ask for the email **in the same question** - *"I'll create Bob Smith as
  crew - what's his email, so he gets his confirmations?"* - then `manage_traveler` action `create` (with the
  email if given), then add him. The email is optional: if the user skips it, carry on
  (`emailOnFile: false`) and mention once that he will not get confirmation emails. An existing
  profile with `hasEmail: false` gets its email through `manage_traveler` action `update`.
- "Rename it Board meeting" -> `changes: { tripName: "Board meeting" }` on any leg.
- "Add a leg to TEB Friday at 9" -> `operator_manage_trip { action: "add_leg", tripId, leg: { to: "TEB", departLocal } }`. When the user names
  who flies it, put them on the new leg in the same call with `leg.travelers: [{ travelerId, role, crewType }]`;
  never ask for crew to add a leg;
  it leaves from the trip's last destination. **Read the trip first** with
  `operator_trips { tripId }`: the web trip list shows only leg 1, so "add a return leg" may
  ask for a leg that already exists. If it does, say so and offer to change it instead.
- Changes go ahead **unasked**. The one exception: **an airport change on a leg with live orders**
  answers `CONFIRMATION_REQUIRED` naming the orders, because a hotel or car at the old airport stays
  where it is. Ask that one question.
- The read-back lists the leg's live orders and `platformAlerts` - the alerts FlightBridge raises on
  them (`LegDateTimeChanged`, `AircraftTailChanged`, ...). Tell the user their hotel or car was NOT
  changed at the vendor; offer a rebook (`book_hotel` with `replacesOrderId`) or a re-shop if the stay no longer fits.

## Crew-positioning trips

A crew-positioning trip moves ONE crew member to the aircraft, or home after a trip, usually on an
airline. It has one leg, that one crew member, no passengers and no FBOs. `operator_trips` marks
it: `tripType` is `crewPositioning`, the leg has `isPlaceholder: true` and
`placeholderType: "CrewActivity"`, and `positioningTraveler` names the crew member.
`positioningDirection` is `out` (to the aircraft) or `return` (home), read from the trip's name.

- Hotels and airline tickets for that leg are for the positioning traveller. Do not ask about
  passengers or FBOs; they do not apply.
- To make one: `operator_manage_trip` action `create_positioning` with the crew member (`traveler`:
  a name or a `travelerId`), `from`, `to`, `departLocal` and `arriveLocal` (both local at their own
  airport), and `aircraft` only when the user names one. `direction` is `out` unless the user says
  the crew member is going home (`return`).
- FlightBridge names it "POS" (or "POST" for a return), the traveller number and the local date,
  and numbers it POS- plus its next trip number. A number the user gives must start with POS-.
- The reply reads the trip back. Say it as a positioning trip for that crew member, then offer the
  undo (`cancel_trip`).
- On a positioning trip, `update_leg` may change the times, the airports, the aircraft and the crew
  member (remove the old one and add the new one in the same call). Adding a leg, a passenger, a
  second traveller or an FBO is refused (`POSITIONING_SHAPE`).
- Other refusals, each said plainly: `POSITIONING_PURPOSE_MISSING` (a company administrator adds a
  trip purpose with external identifier CrewActivity in Company Admin), `POSITIONING_TRAVELER_NOT_CREW`
  (the profile is not a crew profile), `POSITIONING_TRAVELER_NUMBER_MISSING` (add the traveller
  number to the profile first).

## A trip from a scheduling system

Some companies' trips come whole from their scheduling system. FlightBridge shows them, but the
scheduling system owns the dates, airports, aircraft and crew. Read the leg's `editableFields`
before you promise any change:

- every field: a trip made in FlightBridge; change it as above.
- `arrivalFbo` and `departureFbo` only: the scheduling system does not send FBOs, so the FBO on each
  leg can be chosen here and nothing else. `tripSource.name` says which system it is.
- empty: the scheduling system owns everything, the FBOs included. It is changed there.

`me.tripSource` says the same for the company's trips in general, so you can tell the user early
("your trips come from your scheduling system; you can choose the FBOs here").

To choose an FBO: `operator_manage_trip` action `update_leg` with `changes.arrivalFbo` or
`changes.departureFbo` only (an `fboCsaId` from `search_airports`, a name, or `favorite`). A leg
with no FBO may carry `fboSuggestion`, the company's favourite FBO at that field; offer it first.
In `search_airports`, the favourite has `favorite: true`.

- The first call previews and asks one question. Read it as it is: the old and new FBO, the
  stop's other leg when it changes too (the arrival FBO of one leg is the departure FBO of the next
  when they share the field), and any order that still names the old FBO. Then call again with
  `confirm: true`.
- The choice is marked as set in FlightBridge, so the scheduling system's next update keeps it.
- Any other field in the same call is refused and nothing changes, the FBO included.
- `TRIP_SAVE_UNCERTAIN` means FlightBridge took the change but has not shown it yet. Read the trip
  again in a minute; never send the change twice.
- After the change, FlightBridge raises alerts on orders tied to the old FBO ([alerts.md](alerts.md)).
  The reply's `followUps` offers the next step: cancel the notice to the old FBO and notify the new
  one, or notify the new FBO when there was no notice. Ask its question; do it only on a yes.

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
- **`move`** + `moveToLegId` - hotels are moved to that leg, as `manage_order` action `move` does it (a leg at another airport answers
  `LEG_AIRPORT_MISMATCH`; pass `confirmAirportMismatch: true` only if they mean it), then the leg;
- **`leave`** - the leg is cancelled and the orders stay live; FlightBridge flags each "Leg was
  cancelled." Say the orders are still booked.

An **FBO notification** on the leg is listed among the orders ("FBO notification to Acme Aviation
(Requested)"): `cancel` sends that FBO a cancellation notice; it cannot be moved. After a leg changes,
FlightBridge raises alerts on its orders; read and fix them with `alerts` ([alerts.md](alerts.md)). A catering order
is cancelled with `manage_order { action: "cancel", orderId }` ([catering.md](catering.md)) before the
leg. Other orders cannot be cancelled or moved here: `ORDERS_MAY_NEED_ATTENTION` names them - they are
cancelled in FlightBridge, or the user chooses `leave`. Cancelling the last live leg
cancels the trip. A cancelled leg or trip cannot be reactivated from here; re-add a leg with
`operator_manage_trip` action `add_leg`.

## Notify the FBO

An FBO notification is FlightBridge's **Notify FBO** order: one per leg, FBO and direction, e-mailed to
the FBO and put on its FlightBridge board, where the FBO confirms it. It carries the aircraft, the
times from the leg, the services wanted and the notes.

**When to notify.** When the user names an FBO or says "let them know we're coming", "tell Acme",
"notify the FBO" - while creating a trip, adding a leg, or later. Do not notify unasked.

- **Trip context:** `operator_manage_fbo_notification { action: "send", legId }` - the leg's arrival FBO by default; pass
  `arrivalOrDeparture: "departure"` for the departure. A leg with no FBO gets the airport's only FBO
  (`defaultsApplied` says so) or answers `needs: fboCsaId` with the choices - ask which. A based
  aircraft (the airport is its home base) is notified for its departure, as FlightBridge does it. One
  notification covers the whole stay: an arrival runs to the next departure from the same FBO.
- **Ad hoc - one call:** *"N1234 into MDT Tuesday 2 PM, out Wednesday 10 AM, fuel, tell Acme"* =
  `operator_manage_fbo_notification { action: "send", tail: "N1234", airportCode: "MDT", arrival: { date, time: "14:00" },
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
`operator_fbo_notifications` shows `fboResponse` (confirmed or declined, and by whom) later.

**After a change.** When a leg's time moves (`operator_manage_trip` action `update_leg`) and the leg has an FBO
notification, offer once: *"Want me to update Acme Aviation with the new time?"* - then
`operator_manage_fbo_notification { action: "update", serviceOrderId, arrivalLocal | departureLocal }` moves the time
and re-sends in one call (the FBO sees "Changed"). Adding a service: `{ serviceOrderId, hangar: true }`.
`resend: false` saves without sending (`changesNotSent: true`) - only when the user asks.

**Cancelling.** A leg or trip cancel lists the notification (above). On its own:
`operator_manage_fbo_notification { action: "cancel", serviceOrderId }` - a draft is discarded at once; a sent one answers
`CONFIRMATION_REQUIRED` with what the FBO was told: ask, then `confirm: true`, and say the FBO gets a
cancellation notice.

| answer | say / do |
|---|---|
| `FBO_ORDER_EXISTS` | this leg already has a notification for that FBO (`serviceOrderId`): change or re-send it with `operator_manage_fbo_notification` action `update` |
| `FBO_NOT_ON_LEG` | the leg has a different FBO; change the leg's FBO with `operator_manage_trip` action `update_leg`, then notify |
| `FBO_NOT_AT_AIRPORT` / `NO_FBO_AT_AIRPORT` | that FBO is not listed there; offer `options` (or the airport's `fbos[]` from `search_airports`) |
| `FBO_NO_EMAIL` | FlightBridge has no e-mail for that FBO; it is added in FlightBridge, or the user calls them |
| `FBO_NOTICE_IN_FLIGHT` | FlightBridge is still sending the last notice; try again in about a minute |
| `LEG_FBO_REQUIRED` | the leg comes from a scheduling system and has no FBO on that side, and its FBOs are set there; say so. When the FBO can be chosen here, notifying chooses it first by itself (the one named, else the company's favourite, else the airport's only FBO) and the reply says both steps |
| `tripSharing` | read it, never act on it: a notification sent here is not governed by the company's trip-sharing settings |

## Refusals and uncertain saves

| answer | say |
|---|---|
| `ROLE_REQUIRED` | the user needs the Trip Manager role; an administrator grants it in FlightBridge |
| `TRIP_NOT_EDITABLE` (`notFlightBridgeSourced`) | read `message`: it names the system the trip comes from, and `editableFields` says what can still be changed here (often the FBOs) |
| `TRIP_NOT_EDITABLE` (`pilotTrip`, `legCancelled`, `outsideBoardWindow`) | read `message` |
| `EVENT_NOT_SUPPORTED` | it is a calendar event, changed in FlightBridge |
| `TRIP_NOT_FOUND` | not among the company's trips: re-read the id from `operator_trips` |
| `AIRCRAFT_NOT_FOUND` | the tail is not in the company's aircraft list; offer `yourAircraft`, or add it ([aircraft.md](aircraft.md)) |
| `LEG_TIMES_INVALID` / `LEG_ORDER_INVALID` | the times do not work (lands before it leaves, overlaps another leg); remember they are local |
| `TRIP_NUMBER_IN_USE` | the user's number is taken; omit it and FlightBridge numbers the trip |
| `TRIP_SAVE_UNCERTAIN` | **FlightBridge took longer than expected and a re-check has not found it yet. Never repeat the change blindly.** Call the same tool again with the SAME `clientRequestId` in a minute: it re-checks and returns the trip if it landed, and never creates a second one. |
| `TRIP_SAVE_FAILED` | FlightBridge refused it and nothing changed; `platform` has its words |

**Calendar events.** `operator_trips` marks an event row `tripType: event` (also `isEvent:
true`) with `event` (its name, its own airport and dates) and `editable: false`; a real trip is
`tripType: operatorTrip`. An event is trip type 2: always two legs, "1" the start and "2" the end,
usually at one airport. It is not a trip: never call it a flight, never pick it as "your trip" because
it is next, never edit it, and offer no hotel or car for it unless the user asks for that event by name.
If they do, its window runs from the start leg to the end leg (`returnLeg` is the end leg).

Worked examples are in [trip-examples.md](trip-examples.md).
