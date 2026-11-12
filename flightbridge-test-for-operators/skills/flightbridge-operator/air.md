# Airline tickets for the crew and passengers

Commercial airline tickets, mostly to **position crew** to or from a trip: a pilot flying to meet the
aircraft, or home after the last leg. Training trips are the other common case; personal travel is the
operator's own call. Air must be switched on for the company in FlightBridge: if it is not, every air
tool answers `AIR_NOT_ENABLED` and the user's administrator can ask FlightBridge to enable it.

Four tools: **`air_search`** and **`air_itinerary`** read; **`book_air`** books;
**`manage_air_order`** voids or cancels. An airline order is read with **`orders { orderId }`**. One traveler per search and per booking.

## Searching - `air_search`

**Search from the trip leg first.** When the travel is for a trip, pass the `legId` and the `purpose`:

```
air_search { travelerId: 81908, legId: 146501, purpose: "to_trip" }
```

- `to_trip`: the traveler arrives before the leg departs, from their home base to the airport with
  airline service nearest the leg's departure field.
- `from_trip`: the traveler leaves after the leg lands, back to their home base.
- `to_training` / `from_training`: the same for training.

FlightBridge works out the airports, the date and the time window from the leg; `search` in the reply says
what it used. Read that back ("flying Atlanta to Harrisburg Tuesday morning, arriving before the 2 PM
departure"). Without a trip, pass `from`, `to` and `date` (and `time` with `timeType` `depart_after` or
`arrive_by` when the user gives one). An airport with no airline service answers `needs` with the nearest
that have it: offer those.

- **One-way unless it is clearly a round trip.** Positioning is one-way. Pass `returnDate` only when the
  user asks for a return or the trip is plainly there and back (a training course, for example).
- `travelerId` is required: fares, loyalty and the identity on the ticket depend on who flies. Take it from
  the trip manifest or `travelers`.
- Filters when the user asks: `cabin`, `refundableOnly`, `maxStops`, `carrier` (two-letter code),
  `alternateAirports`, `flexDays` (the day before and after).

The reply ranks itineraries by price including taxes, then flight time (`rankedBy`). Present the top few
as a short table: flights and times, stops, duration, refundable or not, total. **Say FlightBridge's booking
fee with the price**: `bookingFee` is per ticket and is not refunded once the ticket is issued. When
`credits` is present the traveler has unused airline credits; mention them, and say that applying one
to this booking is not available here yet.

## Seats

When seat information is available, each flight in an economy or premium economy search carries `seats`:
a short `label` in words (for example "Plenty of aisle & window") and an `icon` (`glyph`, and an `svgUrl`
picture of a window, middle and aisle seat, filled when open). Both are given: show the glyph or icon
beside each flight when it helps, and use the label as its words. `status` `pending` means call
**`seat_labels`** with the `seatLabelsToken`; `unavailable` means say the seat map is unavailable, never
guess. Business and First are not labelled. For one flight's whole cabin, First included, use
**`seat_map`**: which seats are left, aisle or window, exit rows, extra legroom, paid seats, and how many
seats are open up front. Seats are information only: a seat cannot be chosen or held while shopping.

## One itinerary in detail - `air_itinerary`

`air_itinerary { itineraryKey }` re-prices one itinerary now (`priceChanged` says if it moved) and lists
every fare option for it: cabin, refundability, change and refund penalties in words, bags. Use it when the
user wants a different fare, or the search is more than a few minutes old. `ITINERARY_EXPIRED` means search
again.

## Booking - `book_air`

`book_air` **always previews first**. The first call answers `CONFIRMATION_REQUIRED` with a `question`
that states:

- the flights;
- fare plus taxes plus the FlightBridge booking fee, and the total;
- whether the fare is refundable;
- the airline's window to undo it at no cost, or that this airline issues the ticket immediately, in
  which case it cannot be undone at no cost and cancelling forfeits the fare;
- the legal name as it will be sent;
- the card.

Ask that question as written. After the yes, call again with `confirm: true` and
`confirmedTotal` = `preview.total`.

```
book_air { itineraryKey: "...", travelerId: 81908, legId: 146501, purpose: "to_trip" }
book_air { itineraryKey: "...", travelerId: 81908, legId: 146501, purpose: "to_trip", confirm: true, confirmedTotal: 412.60 }
```

- **The card.** The traveler's own default card is used. With none, `PAYMENT_METHOD_REQUIRED` lists the
  company-wallet cards by label: offer them by name on card and last four, and book again with
  `payment.vaultCardId`. With no wallet card, offer the secure card link ([card-link.md](card-link.md)).
  Never take a card number in chat.
- **Identity.** The ticket needs the legal name, date of birth and gender exactly as on the traveler's ID.
  They come from the profile. When something is missing the reply asks (`needs`, one question per field):
  ask once, pass the answers in `identity`, and tell the user they are saved to the traveler's profile for
  next time. Never guess a gender or a date of birth.
- **Loyalty, Known Traveler and redress numbers** come from the profile; a number the user gives wins.
- **After booking** the reply says the booking is **confirmed and queued for ticketing**. The e-ticket
  number usually follows within about 10 minutes: check with `orders { orderId }`. Never quote a ticket number
  the reply does not have.
- `PRICE_CHANGED`: quote the new total and book only if the user agrees. `BOOKING_UNCERTAIN`: it is not known
  whether a ticket was booked; never book again, read `orders { orderId }` or `orders { orderType: "air" }` first.
  `PRODUCTION_GDS_BLOCKED`: airline booking is not available in this environment; say exactly that.
- With `legId` and `purpose` the order is filed under that trip leg, so it shows on the trip.
- **What the flight is for.** A booking needs a `purpose`. A search from a trip leg already carries it. After a
  search between two airports, `book_air` asks (`PURPOSE_REQUIRED`): to or from a trip, training, positioning,
  personal or other.
- **Phone and email.** The airline gets the traveler's phone and email from the profile. When one is missing the
  reply asks; pass it as `phone` or `email` and it is saved to the profile.

## The order - `orders { orderId }`

`orders { orderId }` gives the status in words (confirmed and queued for ticketing, ticketed, under
review, cancelled, voided), the record locator, the e-ticket once issued, the segments, the totals with the
booking fee and the card label. `eligibility` says what can be done now: `canVoid` with `voidUntilCentral`,
or `canCancel` with what cancelling means (`cancelOutcome`). Each row links to the order page, the receipt
and the confirmation in FlightBridge: show the order number as the link. A company's airline orders over a
period are `orders { orderType: "air" }` ([orders.md](orders.md)).

## Undoing a booking - `manage_air_order`

Read `orders { orderId }` first; its `eligibility` decides which word applies.

- **Void** (`action: "void"`) inside the airline's no-cost window: the fare comes back. The window runs
  to 11:59 PM Central a few days after ticketing and never past the end of the departure day.
- **Cancel** (`action: "cancel"`) after it: a refund under the fare rules when the fare allows one,
  otherwise the fare is forfeited.
- Both preview first; ask the preview's question, then call again with `confirm: true`. Neither can be
  reversed, and FlightBridge's booking fee is not refunded. A refund's timing is the airline's.
- `VOID_WINDOW_CLOSED`: say when it closed and offer the cancel. `NON_ARC_CANNOT_VOID`: this airline issued
  the ticket immediately; cancelling forfeits the fare. `NOT_TICKETED_YET`: a void before ticketing is a
  cancel, and the reply says so. `CHECKED_IN`: the traveler has checked in; it cannot be undone here.

## Not here yet

Changing a ticket (dates or flights), applying an unused credit, and choosing a seat from a seat map are
not available through this connector yet. Say so plainly and give the order link: the order page in
FlightBridge has them.
