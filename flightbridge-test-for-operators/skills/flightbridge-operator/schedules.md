# Schedules and leg checklists

A company's **schedules** are the views its administrators set up in FlightBridge (Schedules in the web
app): which legs show, over what window, with which columns and which checklist **rollups** (a rollup is
a group of checklist items, such as "Catering" or "Ground", shown as one status). Each leg also has a
**checklist**: the items to arrange for that leg, each with a status and a note.

These tools appear only for users with the Checklist role, in a company with checklists switched on.
`me.roles.checklistUser` says whether this user has the role.

## Reading a schedule

1. **List the schedules once**: `operator_schedule {}` returns each schedule's `scheduleId`, title,
   window, columns (`key`, `label`) and rollups. Open a schedule by the id from this list; never guess one.
2. **Open it**: `operator_schedule { scheduleId }` returns the legs the web page shows now, in the
   schedule's own order. Each row has `tripNumber`, `legNumber`, `tail`, the airports, the departure,
   `columns` by key (each value exactly as the page shows it) and `rollups` with a status in words.
3. **Ask only for what the user wants.** "When does N123 leave and is catering done?" is
   `columns: ["DepartureTime"], rollups: ["Catering"]` with `tail: "N123"`. Narrow with `tripNumber`,
   `tail` or `legId`, and with `from`/`to` (dates in the schedule's own time zone) when the user names
   days outside the schedule's window. The same toggles as the page are available: `hideCompleted`,
   `hideLanded`, `hidePostFlight`, `hideReleased`, `onlyDelayed`.
4. **Drill down only when asked.** By default a rollup carries its status and counts.
   `includeRollupItems: "full"` adds the items behind each rollup (the default for one leg).

A rollup's status sums up its items: **Not Needed** when every item is not needed; **Completed** when
every item is completed or not needed; **Review** when any item needs review; **In Progress** when any
item has started; otherwise **Not Started**. Order-driven words (Confirmed, Delivered, All Confirmed,
All Canceled) come from the orders behind the items.

When the reply says `isMaxResults`, the schedule has more legs than one answer carries: narrow the
window or the scope instead of reading on.

## One leg's checklist

`operator_checklist { legId }` is the direct way to answer about one leg, with no schedule: every item
with its status, note, delegation, side (arrival or departure) and orders; every rollup for the leg;
the leg note, the trip note and the text notes. `tripId` instead returns every leg of the trip.
`includeHistory: true` adds each item's status history (who, when, the note) and the note histories;
ask for it only when the user asks what changed or who changed it.

## Changing a checklist or a note

`operator_manage_checklist` makes one change per call, through the same paths as the web page, so
FlightBridge's integrations, alerts and open schedules update just as they do from the page:

| The user wants | Call |
|---|---|
| an item's status | `{ action: "set_item_status", itemId, status, note? }` |
| an item's note | `{ action: "set_item_note", itemId, note }` |
| every item of a rollup on a leg | `{ action: "set_rollup_status", legId, rollupName, status }`, then the same with `confirm: true` |
| the leg note | `{ action: "set_leg_note", legId, note }` |
| the trip note | `{ action: "set_trip_note", tripId, note }` |
| a schedule text note | `{ action: "set_text_note", legId, column, note }` |

- **Statuses a person can set** are the item's `canSetTo` (from `operator_checklist`): usually Not
  Needed, Not Started, In Progress, Completed, Delivered and Review. Confirmed, All Confirmed and All
  Canceled are set by the item's orders; the tool refuses them with STATUS_SET_BY_ORDER.
- **A rollup change previews first.** It changes every item in the rollup on that leg, so the reply asks
  one question naming the items. Ask exactly that, and call again with `confirm: true` only after a yes.
- **Only the named field changes.** Setting a text note leaves the leg note, the trip note and the other
  text notes as they are. An empty `note` clears that one note.
- A text note's `column` is TextOne, TextTwo or TextThree, or the label a schedule shows for it
  (`textNoteLabels` in `operator_checklist`).
- An item under operational control needs the Operational Control role; the tool says so when it
  refuses.
- After a change the reply reads the leg back (`now`). Quote the item's new status from there.

## What these tools do not do

They do not create or delete checklist items, change templates or delegation, set up schedules, or
send a schedule's follow-up e-mail. Those stay in the FlightBridge web app.
