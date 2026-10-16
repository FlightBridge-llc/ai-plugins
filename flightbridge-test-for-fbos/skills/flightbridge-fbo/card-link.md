# Cards: the secure card link

Read this when a hotel booking needs a card that is not on file, or the user asks about a card. A card number is
never typed into the conversation. The payment-status and "which card?" rules are in `hotels.md`, Step 9.

**The secure card link (FBO reps and aircraft operators).** A card number is never typed into
this conversation. On an FBO booking off a reservation the card comes from the **customer's wallet at
your FBO** (the visiting operator's cards held at your location): when that wallet holds exactly one
card, `book_hotel` uses it by itself - no `vaultCardId` needed, and the guest's `paymentSource` reads
`customerWallet`. An aircraft operator's card comes from the **company's own wallet** or the guest's
stored card. When a booking needs a card that is not on file (`PAYMENT_METHOD_REQUIRED` with
`cardRequest.offer`, `paymentPreview.cardNeeded`, or `cardOnFile: false`), make ONE call and ask
nothing first: `request_card { reservationId, travelerId }` (FBO rep) or `request_card { legId | tripId,
travelerId }` (operator; ad hoc with neither is fine). The reply carries `link` and `say`; say exactly:

*"Click this link to securely enter the credit card to be used for this booking, or let me know an email address if you want me to email this link to the traveler or other arranger."*

and give the link. **Ask for an address only when the user wants the link emailed.** The CSR may take
the card by phone and enter it on that page themselves. When the user says *"email it to Sherry"*, call
`card_request_status { requestId, sendTo: { contact: { firstName: "Sherry" } } }` (the same link, no new
one) - or `request_card { ..., contact: { firstName: "Sherry" } }` when there is no link yet; with an
address they give, `sendTo: { email }`. `CONTACT_AMBIGUOUS` / `CONTACT_LAST_NAME_REQUIRED` ask one
question; a failed email still leaves the link working (`emailRefused` says why). Read the `summary`
back; it says who got it (in a test environment a non-whitelisted address goes to a test inbox
instead). The page asks for the aircraft's tail number or the traveler's last name (3 attempts), then
sends the card straight to FlightBridge's card vault - nobody in the chat sees it. Lost the tab?
`card_request_status` hands the same link back while it is `ready`, `sent`, `opened` or `verified`.
Progress, from `statusWords`: "ready" (made, not emailed) / "sent" / "the link was opened" / "identity
confirmed" / "card saved" (also locked, expired, cancelled). Say "the link was opened" - never that the
traveler opened it: the CSR may be the one on the page. A CSR entering the card for the cardholder ticks
"Entered on behalf of the cardholder with their verbal authorization" on the page; the status then says
`enteredOnBehalf: true`.
When the user says the card is in but the status is still `verified`, say *"identity confirmed, card not
yet saved - submit the form"* and do not book yet. A page fetch seconds after an email is a mail
scanner, not a person - it only shows in `linkFetchedAt`. `fbo_get_reservation` also shows `cardOnFile`
and `cardRequest`. When the status is `completed` (or `cardOnFile` turns true), book as agreed:
`book_hotel` again with no `vaultCardId` picks up the traveler's new stored card (and, for an FBO, a
customer wallet's sole card), and you name it by its label ("Visa ending 1234"). **If a card number
appears in the chat, do not repeat it** - point at the link. `locked` (3 wrong answers) and `expired`
need a new link; a new link for the same visit cancels the old one. `CARD_REQUEST_LIMIT` means 3 links
already made for this visit or trip in the last 24 hours (it names them - re-use one with
`card_request_status`, or wait); `RATE_LIMITED` means 5 links in the last hour, or the daily cap, from you; `CARD_LINK_NOT_SUPPORTED` means this operator company's card setting cannot use a
stored card - say the card has to be added in FlightBridge. (`fbo_request_card` / `fbo_card_request_status`
are frozen aliases of the two tools.)

**Whose card it is - never a question.** Whoever enters it (the dispatcher, the
CSR on the phone, or the traveler), an FBO rep's link puts the card into the OPERATOR's wallet at your
FBO and an operator's link into the company's own wallet, linked to the traveler either way - pass
`travelerId` (with one crew member on the visit or leg it is linked by default). The card only
guarantees the room against a no-show; the guest pays at check-in with their own card. So never ask
"is the cardholder the traveler?", and read the booking's `payment.sentence` as written - it names
the wallet owner, the traveler and the label: *"Guaranteed for no-show to the card on file for Riley
Johnson in Acme Aviation's wallet (Visa ending 1234); the guest pays at check-in with their own
card."*

**The note.** `note` is optional. A note naming the tail or an accepted last name is still sent, with
those words taken out (they are the page's security question) - relay the reply's one sentence about
it. A note with a long digit run is refused (`PAN_REJECTED`, `needs: note`): rephrase or drop it.

**No card on file: one call, never make a booking call you know will fail.** A new customer has no
card by definition, and `fbo_create_reservation` / `fbo_get_reservation` say `cardOnFile: false`; an
operator's shop with named guests says `paymentPreview.cardNeeded`. As soon as the hotel is chosen, call
`request_card` and say the link line with the link - do NOT call `book_hotel` first, and do not ask who
should get an email. In the same message ask the one remaining OK the chosen row's own terms need (a
day-only or under-24-hours free-cancel deadline, a non-refundable rate) - *"Once the card is in, the
booking also needs your OK: the Residence Inn's free-cancel deadline is 6 PM tomorrow and the hotel gave
only the day. Book it then?"* When `book_hotel` does answer `PAYMENT_METHOD_REQUIRED`, it has already
re-priced the room: when it carries `pendingReasons` and `pendingQuestion`, put the link line and that
question in ONE message. After "the card has been saved", call `card_request_status` once (cheap, and it
carries the label). Then book: with `confirm: true` and `confirmedTotal` (the preview's total, or the
shop row's all-in total when there was no preview) ONLY if the user actually said yes to that remaining
OK; with no such yes, call `book_hotel` without `confirm` and ask its `question`. The server re-checks
price and terms; if they changed, it answers `CONFIRMATION_REQUIRED` with `changedSinceYes` - ask that
again.
