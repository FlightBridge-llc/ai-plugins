# The location's rate card - quick answers

A seasoned CSR answers "what hotels do you recommend?" or "what cars can you get me?" from memory. The rate card is
that memory: for the rep's location, 10 hotels (the FBO's own Favorites first - the stars in FlightBridge's hotel search -
then the hotels FlightBridge buyers book most there; hidden hotels never appear) and the vendors that deliver to the FBO,
each quoted for one night or one day on each of the next 14 days, refreshed in the background. It costs no tool call:
a one-location rep's **instructions carry the speakable line**, and `me.rateCard` carries the summary.
`fbo_rate_card` (cache only, never shops) gives a day (`date`), one hotel's 14 days (`hotelId`) or one vendor's (`vendor`).
Pass `kind: "hotels"` for a hotel question and `kind: "cars"` for a car question; the visits' pre-shops come
only with `includeReservations: true`. A rep with several locations who said where they work ("working out of MDT")
gets `airportCode: "MDT"` on the FIRST call - never a call with nothing that comes back `needs: location`.

**The script.** Recommended hotel first (the card's first hotel that holds a price - never one with no held rates), two alternatives, the typical price and the spread,
the age, then the offer. **The default answer is that line and nothing more: 60 words at most.** Tables,
every hotel, "things you should know" and analysis only when the caller asks for them. A CSR on the phone reads your
answer aloud; a long one is slower than no answer.

> *"Our recommended hotel is the Hilton Garden Inn - usually about $139 a night with our rate. The Courtyard and the
> Hampton Inn are also close, around $149 and $129. Those are as of this morning. Do you want me to look into any of
> those?"*

Cars the same way: *"Hertz delivers here, from about $62 a day; Enterprise about $58. Want me to check one for your
dates?"*

- **Exceptions as facts**, when they touch the caller's dates: *"Thursday the Hilton is $189, and the Courtyard is sold
  out; Friday it's back to $139."* Never a guessed cause.
- **The live shop uses the card's order.** An FBO rep's `shop_hotels` (by airport, leg or reservation) lists
  the FBO's Favorites first, most booked across FlightBridge first, then the hotels FlightBridge buyers book most there,
  then the nearest others; price does not reorder them (`rankedBy` says so). The recommendation is the first favourite
  with rooms, and `summary` opens "Your favourite ...". Each row's `reason` says why it is listed (`favourite`,
  `popular`, `pad`): read it back when the caller asks. Operator shops still rank by price.
- **A price question is still a card answer.** "How much is the Hilton Thursday?" is answered from the card's row for
  that hotel and date (`fbo_rate_card {date}` or `{hotelId}` - cache only), said as "about $X, as of this morning",
  with the offer to check it live. Do not shop live just to answer a price question.
- **Narrowing = one live shop**: when the caller wants to BOOK, or asks for a firm price for their dates, shop that one live -
  `shop_hotels {hotelIds: [that hotel], checkIn, checkOut, rooms}` (or `get_hotel`) and `shop_rentals {vendors: [that
  vendor], ...}` - then the normal booking path. The live price is the one to quote and book.
- **Never** read a cached figure as a live price, never say a cached figure is "available now", never book from the card.
- "Usually" means the median over the days the card holds; say so if asked. `typical: null` = fewer than 7 days held:
  say "no typical price yet" and give tonight's.
- `RATE_CARD_NOT_READY`: no card for that location yet - shop live as before.
- **A reservation's own pre-shop.** Each upcoming reservation at a card location is shopped ahead for its exact
  stay (arrival night to departure date, one room per arrival crew member) and car window, within a minute of
  `fbo_create_reservation` or a date change, and by a 15-minute scan for reservations made in the web app.
  `fbo_get_reservation` carries it as `rateCard`: `line`, the full `hotels` list (the card's 10, Favorites first, each
  with its total per room for the whole stay, free-cancel deadline and bed) and `cars` (each vendor's intermediate,
  full-size, standard SUV and minivan for `carWindow`, cheapest first). `fbo_search_reservations` rows carry
  `rateCardReady` and `rateCardTop`. **"What options do they have?" is answered from `rateCard.hotels` and
  `rateCard.cars` with no further call**, saying the age. Shop live only when the caller picks one or wants a firm
  price: `shop_hotels {reservationId, live: true}` or `{hotelIds: [the pick]}`, and `shop_rentals {vendors: [the pick],
  startDate, endDate}` with the `carWindow` times (a reservation's leg carries no rental window, so pass them).
  `ready: false` with `queued: true` right after a create means it is still shopping: say so, or shop live.
- **When `fbo_get_reservation` carries a ready `rateCard`, do NOT call `shop_hotels {reservationId}` too.**
  The read already holds the pre-shopped hotels and cars; answer from it and offer the live check. A second call
  returns the same figures and costs the caller seconds.
- **"What about the Oct 20 visit?" is ONE call.** `fbo_get_reservation` carries the visit, its pre-shop
  (`rateCard`) and what is already booked against it (`existingOrders[]`: type, vendor, confirmation, status, leg;
  cancelled orders left out). Do not follow it with `get_existing_orders` or `shop_hotels` for the same visit.
  `existingOrders: []` means nothing is booked yet; `existingOrdersErrors` means a read failed, so say that you could
  not check, never that nothing is booked.
- **The location the rep named is remembered.** Once a rep with several locations names
  one - on any shop, the rate card, or by creating or opening a reservation there - shops and the card default to it
  for 12 hours. Call them with no airport and read `defaultsApplied` back ("your location, as chosen earlier"). Ask
  "which location?" only when the answer says `needs: location`.
- A tonight figure the line marks **non-refundable** (`tonight.flexible: false`) is the only rate held for tonight: say
  "non-refundable" with it, and that a flexible rate may exist on a live shop. `termsClass: deadlinePassed` is a
  free-to-cancel rate whose time for tonight has passed - say that, never "non-refundable".
- A vendor's identical days are one fact (`exceptionRanges`: "Enterprise: setup error, all 14 days, all classes") - say
  it once, never per day.
- **An error is not a fact about the cars.** An exception marked `errorRow` - a vendor or setup error, a hotel
  that did not answer - carries when it was seen (`asOfText`; the words read *"Hertz: error: vendorSystem - as of yesterday
  11:15 AM EDT, Mon Oct 12, all classes"*). Say it with that age ("Hertz's system did not answer when we last checked,
  yesterday at 11:15 AM") and never as "Hertz has no cars"; the card re-checks an error within the hour, and a live shop
  answers now.
- A vendor whose card days all read unavailable (for example the vendor does not take online bookings at this
  location) is not one to offer; name it only if the caller asks for it, and say a live shop can confirm.
