# Orders: what is booked, and order history

One tool, `orders`, answers both questions. Which one it answers depends on what you pass:

- **One leg** - `orders { legId }`: what is already ordered on that leg, with every detail (a hotel
  row's address and cancellation terms, a car's vendor and class, the leg's FBO notification).
  "What's booked on my Harrisburg trip?" is this call, with the leg from `operator_trips`.
- **One person, coming up** - `orders { travelerName }`: their orders on the schedule from a week back
  to a month ahead.
- **History** - no leg, no visit, no `travelerName`: a search of every order you can see over a window,
  with optional filters. Use it for anything that is not one leg or visit: "all our hotel bookings at
  MDT last month", "what did we book for N123AB this year", "Jane's bookings since September".

With a leg, a visit or a name, cancelled orders are left out unless you pass `includeCancelled: true`. The
history search always includes them (below).

## The history search

```
orders { startDate, endDate, orderType, airport, tail, travelerId, status, mine, page, pageSize }
```

Every argument is optional. Send everything the user said in one call.

- **The date basis is the SERVICE date**: hotel check-in, rental pickup, FBO arrival. A booking made
  last week for next month shows under next month. Every row carries `createdAt` (when it was booked)
  and `createdBy`: mention when it was booked when that matters. For "what did we book last week",
  search without dates and pick the rows whose `createdAt` falls in that week - say that is what you
  did.
- **Dates** are local, `YYYY-MM-DD`, resolved from `me.now`: "last month" on October 26 is
  `startDate: "2026-09-01", endDate: "2026-09-30"`.
- **No dates: the default window.** With nothing else, 90 days back to 2 years ahead. With an
  airport, a traveler or an aircraft and no dates, 1 year back to 2 years ahead. The reply's
  `window {from, to, reason}` says which window was used (`reason` is `default`, `default-filtered`
  or `given`); when the user gave no dates, say the window in the answer ("from July 28 on, including
  everything booked ahead").
- **Order type, in the user's words**: hotels or rooms -> `hotel`; cars or rentals -> `rental`; FBO
  notifications -> `fbo-notification`; catering -> `catering`; limo or ground transport -> `limo`;
  fuel -> `fuel`; FBO services -> `fboServices`; trip support -> `tripSupport`; airline tickets ->
  `air`. One type per call; for "hotels and cars", leave the type out and read `byType`.
- **Airport**: `airport` as a code ("MDT", "KMDT") or `airportId`.
- **Aircraft**: `tail`, or `aircraftId` from `operator_aircraft`.
- **A person**: `travelerId` from `travelers` - the guest, driver or passenger, matched exactly.
  Bookings where the user is the guest: their own profile (`me.linkedTraveler`).
- **`mine` is false by default**: who booked an order does not matter to the user, so every order the
  company can see is listed. Pass `mine: true` only when the user asks for the orders THEY booked
  ("what did I book?").
- **Status**: `status` (`requested`, `confirmed`, `fulfilled`, `cancelled`) only when the user asks
  for one state.

## Reading the answer

- **Read `summary` back.** It counts the orders in the window by type and says how many are
  cancelled: *"14 orders with a service date between Oct 1 and Oct 26: 9 hotels, 4 rental cars, 1 FBO
  notification; 2 cancelled."*
- **Cancelled orders are shown with their status, never hidden.** A history is what happened. Leave
  them out (`includeCancelled: false`) only when the user asks for that.
- **Paging.** 50 rows a page (`pageSize` up to 200). `matchedCount` is how many orders matched;
  `hasMore: true` means another page is waiting - say "showing 50 of 132" and offer the next one
  (`page: 2`), or pass a larger `pageSize` when the user wants them all at once.
- **`truncated: true`** means more than 2,000 orders were in the window before the aircraft,
  traveler or status filter, so the list may be incomplete. Say so, and offer a narrower window; never
  present a truncated list as complete.
- **Present a table** when there are several rows: service dates, type, provider (the hotel or the
  car vendor), traveler, tail, status and total, with the
  trip
  each row belongs to. Order times are service dates; give them as dates, not as UTC times.
- **The row's name is its link**, as in every other list: a hotel row's `links.hotel.markdown` in
  place of the hotel name,
  and `links.trip.markdown` in place of the trip.
  No separate link column; a row without `links` is plain text.
- A history row names the traveler by name only and carries the `orderId`. For one order's full
  detail, read it by id: `orders { orderId }` works for every type (a hotel's terms, a car's class,
  an FBO notification, a catering order's items and the caterer's status, an airline ticket and what
  can still be undone).

## Acting on a row

The row's `orderId` is what every change takes: `manage_order { action: "cancel", orderId }` (recap and
a yes first, [hotels.md](hotels.md) Step 10 and [rentals.md](rentals.md)), `book_hotel { replacesOrderId }`
for a hotel rebook (hotels.md Step 11), `book_rental { orderId }` for a car change and
`book_catering { orderId }` for a catering change ([catering.md](catering.md)).
