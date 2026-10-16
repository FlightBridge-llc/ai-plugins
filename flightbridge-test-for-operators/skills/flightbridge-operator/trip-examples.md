# Trip planning - worked examples (test responses, shortened)

All from a test environment: an operator company with a Trip Manager and aircraft N4455. Names and numbers are test data.

## 1. The CLT sentence -> one call

User: *"I'm planning a trip to CLT next Wednesday on N4455 leaving at noon and coming back the next day at 2pm."*
(`me.now` = Monday, October 5, 2026.)

When the aircraft has no home base on file, the first call answers one question:

```
operator_create_trip { aircraft: "N4455", legs: [ { to: "CLT", departLocal: "2026-10-14T12:00" },
                                                  { to: "PDK", departLocal: "2026-10-15T14:00" } ] }
-> needs: [{ field: "legs[0].from", question: "Where does the trip leave from? N4455 has no home base in FlightBridge." }]
```

With a home base on file that question never comes. After "from Peachtree":

```
operator_create_trip { aircraft: "N4455", clientRequestId: "...",
  legs: [ { from: "PDK", to: "CLT", departLocal: "2026-10-14T12:00" }, { to: "PDK", departLocal: "2026-10-15T14:00" } ] }
```

Answered in about 13 s:

> Created trip 0000144 "CLT 10/14" on N4455: KPDK Wed, Oct 14, 12:00 PM EDT -> KCLT Wed, Oct 14, 12:42 PM EDT
> (estimated); KCLT Thu, Oct 15, 2:00 PM EDT -> KPDK Thu, Oct 15, 2:42 PM EDT (estimated). No crew or passengers on
> the manifest yet. I can update the trip manifest any time — let me know when you have that information. Want hotels
> for the CLT night?

- `legs[0]`: `departLocal 2026-10-14T12:00:00`, `departUtc 2026-10-14T16:00:00`, `arriveLocal 12:42`,
  `arrivalEstimated: true`, `estimatedFrom: "estimated from 42 m flight time"`, manifest empty, no FBO.
- `defaultsApplied`: leg 2 leaves from KCLT (where leg 1 lands); both arrivals estimated; trip number 0000144
  (FlightBridge's next number); trip name "CLT 10/14" (destination and date); manifest empty; FBOs not set.
- `undo: operator_cancel_trip { tripId: 98232 }`; `handoff: shop_hotels { legId: 144917 }`.

Say it back as the summary does, then offer the undo and the hotels.

With a home base on file (say FTY), the bare sentence creates the trip at once, and `defaultsApplied` says *"leg 1
leaves from KFTY (the aircraft's home base)"* - no question. Ask only on `needs`.

## 2. Add a leg

```
operator_add_leg { tripId: 98232, leg: { to: "TEB", departLocal: "2026-10-16T09:00" } }
```
> ... KPDK Fri, Oct 16, 9:00 AM EDT -> KTEB Fri, Oct 16, 11:28 AM EDT (estimated).
`defaultsApplied`: "leg 3 leaves from KPDK (where leg 2 lands)", the estimate (2 h 28 m). About 8 s.

## 3. Move a departure two hours later

```
operator_update_leg { legId: 144918, changes: { departLocal: "2026-10-15T16:00" } }
```
> Updated leg 144918 of trip 0000144: KCLT Thu, Oct 15, 4:00 PM EDT -> KPDK Thu, Oct 15, 4:42 PM EDT (estimated).
> arrival re-estimated: Thu, Oct 15, 4:42 PM EDT (42 m flight time).

The arrival moved because it was an estimate. With a hotel on the leg the read-back adds `liveOrders` and
`platformAlerts: ["LegDateTimeChanged"]`: tell the user the hotel itself was not changed.

## 4. Rename the trip

```
operator_update_leg { legId: 144917, changes: { tripName: "Board meeting" } }
```
> Updated leg 144917 of trip 0000144 (trip name now "Board meeting") ...

## 5. Crew with no role, on every leg

"Add Bob Smith to the trip as crew": `operator_update_leg` once per leg, `travelers.add [{travelerId, role: crew}]`.
Each answers *"crew role Other for Bob Smith (none given; PIC or SIC can be set any time)"*, and the manifest reads back
`crewType: Other` on both legs. Then offer: "Is he PIC or SIC?"

## 6. A new person

`operator_create_traveler` without an email answers `emailOnFile: false` and *"No email on file: this person will not
get confirmation emails. Add one any time with operator_update_traveler."*; `operator_update_traveler {travelerId,
email}` answers `hasEmail: true` and *"Saved the email on Bob Smith's profile; confirmation emails will reach them."*

## 7. Cancel a leg with nothing attached - no question

```
operator_cancel_leg { legId: 144919 }
```
> Cancelled leg 144919 of trip 0000144. Nothing else was attached. ...

## 8. Cancel a leg with a hotel - the three answers

First call, no `confirm` -> `CONFIRMATION_REQUIRED`, `reasons: ["LEG_HAS_LIVE_ORDERS"]`, `liveOrders` (type, vendor,
dates, confirmation, `cancelTerms`), `options` (cancel / move / leave) and a `question` like:
> This leg has 1 live order: Hotel at Hilton Hasbrouck Heights from 2026-10-26 15:00 (confirmation H1234567) - free
> to cancel until 6:00 PM EDT on Oct 26. Should I cancel it too, move it to another leg, or leave it for now
> (FlightBridge will flag it)?

Then `confirm: true, ordersAction: "cancel"` (or `"move"` + `moveToLegId`, or `"leave"`). Nothing is sent before the
answer.

## 9. Refusals

- Another company's trip: `TRIP_NOT_FOUND`, for read, add, update and both cancels.
- An imported trip of the user's own company: `TRIP_NOT_EDITABLE`, `reason: notFlightBridgeSourced` - "comes from an
  import or another integration, not from FlightBridge, so it must be changed in the system it came from".
- An FBO user forcing a create: `ROLE_REQUIRED`; the five tools are not in an FBO user's list at all.

## 10. Calendar events

`operator_search_trips {airport: TEB}` can return a calendar event: `isEvent: true`, `editable: false`, summary
*"Event 0000107 at TEB, Oct 9 (leg 143833; a calendar event, not editable here)."* - offer the next real leg, not a
hotel. A `tripId` read of a normal trip reads *"Trip 0000150: leg 145917 at CLT"*.

## 11. A save FlightBridge has not confirmed

When FlightBridge is slow to confirm a save:
> TRIP_SAVE_UNCERTAIN - FlightBridge took longer than expected to save trip 0000145, and a re-read has not found it
> yet. The trip may still appear; check in a minute ...

The same call again with the SAME `clientRequestId` answered `created`, `replayed: true`, `savedAfterWait: true`,
trip 0000145 - one trip, not two.
