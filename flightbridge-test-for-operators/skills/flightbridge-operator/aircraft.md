# Your company's aircraft

The company's aircraft list (Aircraft in FlightBridge) is read with **`operator_aircraft`** and changed
with **`operator_manage_aircraft`**, actions `save`, `delete` and `save_owner`. Aircraft records change
rarely; most questions are reads.

## Reading the list

`operator_aircraft {}` answers "which aircraft do we have?": every aircraft with its `tail`, type
(`typeName`, and `standardType` when it is matched to FlightBridge's standard list), `callSign`,
`cruiseSpeed`, `homeBase`, `tripPurpose` and the operator or owner it flies for (`owner`, or
`ownedByThisCompany: true`). Pass `tail` or `aircraftId` for one aircraft. The reply also carries
`owners[]`, the company's list of operators and owners, and with `includeStandardTypes: true` the
standard aircraft types - so a change needs no second read.

**Read the list before guessing a tail.** When the user names a tail you have not seen, or a trip
create answers `AIRCRAFT_NOT_FOUND`, check `operator_aircraft { tail }`: a near match on the list
("N123AB isn't on file - did you mean N123AD?") is offered first, and only a genuinely new aircraft is
added.

## Adding or changing one - action `save`

```
operator_manage_aircraft { action: "save", aircraft: { tail: "N123AB", typeName: "Citation XLS",
                           standardTypeName: "Citation XLS", homeBase: "KMDT" } }
```

- **Add or edit by tail.** `tail` is required to add one. A tail already on file is CHANGED, never
  duplicated - the reply says so ("N123AB already exists; updated it"). To change an aircraft you have
  read, pass its `aircraftId`.
- **Pass only what changes.** Every other field is kept as it is.
- **It saves without a preview** when the user said what to add or change. `confirm: false` previews
  the change field by field and writes nothing: use it when the user asks what would change, or before
  a change they did not spell out. Undo is another `save` with the old values.
- **Type: free text and the standard type.** `typeName` is the type as the user says it ("Citation
  XLS+"); `standardTypeName` (or `standardTypeId`) matches it to FlightBridge's standard list. When the
  user names a model, pass both. An unclear standard type answers `needs` with the candidates: ask
  which, or save the free-text type alone.
- **Home base**: an airport code (`homeBase: "KMDT"`). New trips leave from it by default
  ([trips.md](trips.md)). **Cruise speed** is in knots; trip arrival estimates use it.
- **The owner**: the operator or owner this aircraft flies for - leave it blank if it is your own.
  Set it with `ownerCustomerId` from `owners[]`; `ownerCustomerId: null` sets it back to your own
  company. An owner not on the list is added first with `save_owner`.
- When the user says an aircraft's type is missing or wrong after a trip change, setting it back is one
  `save` with the type.

Read `summary` back. A new aircraft can be used on a trip straight away.

## Adding or renaming an owner - action `save_owner`

`operator_manage_aircraft { action: "save_owner", owner: { name: "Northwind Charter" } }` adds an operator
or owner to the company's list (Setup > Operators in FlightBridge); pass `owner.customerId` from
`owners[]` to rename one. A name already on the list is not added twice - the reply says so. The new
owner is in `owners[]` and can then be set on an aircraft with `save`.

## Deleting one - action `delete`

- **Company Admins only.** Without that role the call is refused and the refusal names the role - say
  it as written.
- **It is FlightBridge's soft delete.** Trips, legs and orders that use the aircraft keep it and its
  details; it only leaves the aircraft list. Nothing is refused because of upcoming trips.
- **It always previews first.** `operator_manage_aircraft { action: "delete", aircraftId }` (the id from
  `operator_aircraft`) answers `CONFIRMATION_REQUIRED` with a `question` that names the aircraft and how
  many upcoming legs use it (*"N123AB has 3 upcoming legs; they keep the aircraft. Deleting only takes it
  off your aircraft list. Delete N123AB?"*). Ask it; on a yes, call again with `confirm: true`.
- To bring a deleted aircraft back, add the tail again with `save`.

A refusal names what is missing (a role, an aircraft or owner that is not the company's): say it as
written, and do not try another way around it.
