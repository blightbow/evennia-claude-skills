Evennia is an open-source Python-based framework, codebase and server for
creating text-based multiplayer online games (aka MUD/MU* etc) using modern
technologies and tools. It's pretty sexy.
Link: https://github.com/evennia/evennia

This repo contains Claude skills designed to facilitate designing in-game
features while minimizing the amount of time you spend steering Claude away
from the more common traps agentic coders run into when developing for Evennia.
Skills are more efficient than trying to embed all of this logic inside of
CLAUDE.md because the logic is only loaded into context when needed. A central
CLAUDE.md is still provided for semi-ubiquitous logic and common failure modes.

Familiarity with the following Claude Code concepts are assumed:
- https://code.claude.com/docs/en/memory#how-claude-looks-up-memories
- https://code.claude.com/docs/en/skills

Example of the types of problems this skill tries to avoid:

- proper deserialization of SaverDict() and similar to JSON
- web features should be built against Bootstrap 4
- understanding the difference between game entity types
  - avoids attempts to delete script #1 that delete character #1 instead
  - avoids attempts to use search_tag() for non-object entities

PRs are welcome. I assume no liability if Claude deletes the superuser or finds
new and creative ways to make your life miserable that I haven't yet discovered.
