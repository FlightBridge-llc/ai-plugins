# FlightBridge plugins for AI assistants

Each plugin brings the FlightBridge connector together with the skill for your business, so an AI assistant can search and book through your own FlightBridge account.

| Plugin | For |
|---|---|
| FlightBridge Test for FBOs (`flightbridge-test-for-fbos`) | FBO customer service reps |
| FlightBridge Test for Aircraft Operators (`flightbridge-test-for-operators`) | flight departments, charter schedulers and Trip Managers |

These plugins connect to FlightBridge's test environment, where bookings are test bookings. Production plugins will be listed here when they are available.

## Install in Claude Code

```
/plugin marketplace add FlightBridge-llc/ai-plugins
/plugin install flightbridge-test-for-fbos@flightbridge-plugins
/plugin install flightbridge-test-for-operators@flightbridge-plugins
```

Install the one for your business. The connector starts with the plugin; sign in with your FlightBridge account when the first tool call asks.

## Claude on the web or Desktop

Add this repository as a plugin marketplace, or upload a plugin, then add the connector the plugin lists and sign in. Step-by-step setup for every assistant: https://developer.flightbridge.com/connect

Have only one FlightBridge environment switched on in a conversation: the test and production connectors have the same tool names.

## License

Proprietary: see [LICENSE](LICENSE). For use with your own FlightBridge account only. Support: support@flightbridge.com
