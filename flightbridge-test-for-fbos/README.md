# FlightBridge for FBOs (Test)

For FBO customer service reps: put visiting aircraft on the schedule, answer rate questions from your location's rate card, and book hotel rooms and rental cars for visiting crews through your FlightBridge account. This plugin bundles the `flightbridge-fbo` skill, which teaches the assistant how FlightBridge shops, books, changes and cancels safely, together with the FlightBridge connector (a remote MCP server).

## Connector

```
https://testagent.flightbridge.com/mcp
```

Sign-in is OAuth with your own FlightBridge credentials; the assistant acts as you, inside your existing permissions. Nothing is booked without the rules in the skill: bookings, cancellations and customer emails are recapped and confirmed, card numbers are never typed into the chat, and restricted rates need the user's word that the traveller qualifies.

## Setup

Claude Code: the plugin starts the connector itself; sign in when the first tool call asks.

```
/plugin marketplace add FlightBridge-llc/ai-plugins
/plugin install flightbridge-test-for-fbos@flightbridge-plugins
```

Claude on the web or Desktop: add the plugin, then add the connector it lists (FlightBridge Test, the URL above) and sign in. Installing a plugin there does not add its connector for you.

Every assistant, step by step: https://developer.flightbridge.com/connect

## License

Proprietary: see LICENSE. For use with your own FlightBridge account only.
