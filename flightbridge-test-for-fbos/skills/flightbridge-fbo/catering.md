# Catering

Catering is **ordered, not shopped**. There is no availability, no rate and no price when the order is
placed. The user picks a caterer at the airport, lists the items, names a delivery place and time and
says how the caterer will be paid. The caterer then confirms and quotes, or declines, and FlightBridge
shows that on the order. Say so every time, without being asked: "Nothing is priced until the caterer
answers."

`book_*` creates an order at a vendor, so ordering catering is `book_catering`, even though people say
"order catering".

## 1. Read `caterers` first

- An aircraft operator passes the trip leg and which end of it: `caterers { legId, side }`, where side
  is `departure` or `arrival`. When the user names the airport, pass it as `airport` and the side is
  worked out. Otherwise the reply asks which end.
- An FBO rep passes the visit: `caterers { reservationId }`. The departure is the default end.
- Ad hoc, with no leg: `caterers { airport }`.

The reply lists each caterer at that airport: its phone, whether it has a menu, its menu restriction,
whether it is the company's **preferred caterer**, and how the order reaches it (straight into the
caterer's system, or by e-mail). **Offer the preferred caterer first, by name.**

With a leg or a visit, the reply also has:

- `deliveryDefault`: the FBO on that side, which is where the order is delivered unless the user says
  otherwise.
- `prerequisites.fboMissing`: the leg has no FBO on that side. An operator sets it first with
  `operator_manage_trip` (`update_leg`); FlightBridge does not take a catering order without it.
- `travelersOnLeg` and `preferencesSummary`: each person's catering preferences and instructions for
  the caterer. **Read these back before composing anything**, for example: "Jane's profile says
  vegetarian, and Bob has a nut allergy."

## 2. The menu

`caterers { catererId, menu: true }` (with the same leg, visit or airport) returns the menu by
category: each item's `menuItemId`, name, description and **unit**. Read the menu back in the user's
words. Never invent items, and never quote a price: menus have none.

Units matter. "1 x Fruit platter (serves 10)" is not "10 fruit platters". Read the unit back with the
quantity.

A caterer's menu restriction decides what can be ordered:

| restriction | meaning |
|---|---|
| `open` | will take anything; off-menu items go as free text |
| `menuPreferred` | has a menu but takes off-menu requests |
| `menuOnly` | menu items only: an off-menu item is refused, with the closest menu items to offer instead |

## 3. Compose and preview: `book_catering`

Pass the leg and side (operator) or the visit (FBO rep), the `catererId`, and the items:
`{ name or menuItemId, quantity, instructions?, forTraveler? }`. Item instructions carry preparation
and dietary notes. `forTraveler` ties an item to someone on the leg.

- **Delivery time** is the local time at the airport. Say it relative to the flight, in the user's
  words: "90 minutes before wheels-up" is the departure time less 90 minutes. Omit it to use
  FlightBridge's default for that end of the leg; the reply names every default it applied.
- **Who it is for** is `crew`, `passengers` or `both`. It is set when the order is created and cannot be
  changed later.
- **Payment is a label** that tells the caterer how they will be paid: `directBill` (the default, and
  the common case), `callForCard`, `addToFuelBill`, `cardOnFile` (the card **the caterer** already
  holds for the company) or `none`. FlightBridge tells the caterer how you will pay; it does not pay
  them, and it never takes or passes a card for catering.

The first call always **previews**: nothing is sent. It shows each item on or off the menu, the
delivery place and time, who it is for, the payment label and `travelerNotes`, the preferences on file
for the people on the leg. Those print on the caterer's copy of the order automatically. Read them back
and ask whether the items honour them. Then ask the preview's `question` exactly, and call again with
`confirm: true` and the preview's `confirmedSummaryHash` after a yes.

`draft: true` saves the order in FlightBridge without sending it.

## 4. After it is sent

The reply gives the order number and says the caterer has it. Statuses, in plain words:

| status | meaning |
|---|---|
| sent, not yet seen | the caterer has not opened it |
| Pending | the caterer has it in hand |
| Confirmed, In position | the caterer confirmed; a quote, a confirmation number and a total may follow |
| Declined | the caterer turned it down: order from another caterer |

`orders { orderId }` reads one order: its items, delivery, the caterer's status, confirmation number and
the quote once the caterer sends one. `orders { legId }` (or `reservationId`) lists the leg's catering
orders with the other orders.

## 5. Change or cancel

- To **change** an order, call `book_catering { orderId, ... }` with only what changes. An item with its `orderItemId`
  updates that line; `remove: true` deletes it; an item without `orderItemId` is added. Every line not
  named is kept. The caterer is notified of every change. The same preview and confirm apply.
- To **cancel**, call `manage_order { action: "cancel", orderId }`. The caterer is notified. While the caterer
  has the order in hand (Pending), FlightBridge does not cancel it: tell the user to call the caterer,
  with the phone number. A draft that was never sent is discarded, and the caterer hears nothing.

## Catering preferences on a traveler's profile

`travelers { travelerId }` shows each person's catering preferences and instructions for the caterer.
`manage_traveler { action: "update", travelerId, cateringPreferences?, cateringVendorInstructions? }`
saves them, so "note that Bob is allergic to nuts" is one call and appears on every future catering
order for him. A preference cannot be cleared from here; that is done in FlightBridge.

## Not the same as asking the FBO for catering

An operator's FBO notification can ask the FBO to arrange catering (`operator_manage_fbo_notification`
with catering in `otherServices`). Then the FBO arranges it. Ordering from a caterer is this page.
