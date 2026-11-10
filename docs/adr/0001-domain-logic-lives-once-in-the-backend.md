# Domain logic lives once in the backend

Outfit generation, search and scoring are needed by both the Flutter app and the MCP server, so the domain logic is implemented exactly once in the backend (Firebase / Google Cloud) and both are thin clients of it. We accepted that generation needs a network round-trip (at most cached on the device for offline use) rather than maintaining a second implementation in the app that could drift from the one the MCP server uses.
