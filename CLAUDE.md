# Project Overview
Evennia is an open-source Python-based framework, codebase and server for creating text-based multiplayer online games (aka MUD/MU* etc) using modern technologies and tools.

Assume that the Evennia server is running on a remote computer unless the user specifies otherwise. (local python3/evennia commands don't run against the remote server)

# Knowledge Domains (skills)

Load skills based on **task** or **error** you encounter:

## Task Cheatsheet

- **Building backend game code** → building-game-code skill
  - Creating custom objects, characters, rooms
  - Creating custom scripts (timers, AI, weather systems)
  - Creating custom commands with MuxCommand
  - Entity operations (search, create, tags)
  - Prototypes, CmdSets, locks, testing
  - **Prerequisites**: Understand Architecture Overview (this file) first
  - TIPS:
    - This is the one skill you can often get away with loading on its own. Start here, then add more skills if needed.

- **Building web features** → building-web-features skill
  - Templates and UI (Bootstrap 4.6, blocks, context variables)
  - JavaScript integration and data flow
  - AppConfig, lazy imports, URL configuration
  - TIPS:
    - Not relevent unless specifically working with Django views or REST APIs (DRF, ViewSets, serializers)
    - The building-game-code skill is needed as well when web elements integrate with in-game systems

- **Analyzing tracebacks and errors** → analyzing-tracebacks skill
  - Interpreting Python tracebacks
  - Common Evennia error patterns (AttributeError, TypeError, ImportError)
  - Tracing errors back to source code
  - Log analysis
  - TIPS:
    - Use this first when you encounter errors - no debugger required
    - Covers most common error scenarios without needing remote access

## Error Cheatsheet
Start with the **analyzing-tracebacks** skill for error diagnosis.

- `AttributeError: 'arglist'` → building-game-code skill → Command Development
- `TypeError: _SaverDict is not JSON serializable` → building-web-features skill → SaverDict Serialization
- `AppRegistryNotReady` → building-web-features skill → Lazy Imports
- Commands with "=value" fail → building-game-code skill → Critical: lhs vs arglist
- Wrong template blocks break CSS → building-web-features skill → Template Block Names
- `'NoneType' has no attribute` → analyzing-tracebacks skill → AttributeError
- `DoesNotExist` / `MultipleObjectsReturned` → analyzing-tracebacks skill → DoesNotExist
- Search returns empty (silent failure) → Entity Operations (see below)

---

## Contributing to Evennia Core

When contributing code to Evennia:
1. Read relevant domain files to understand patterns
2. Follow **Code Style Requirements** (below)
3. Always run `make format` before committing
4. Ensure tests pass with `make test`

---

## Quick Reference

### Development Commands

```bash
# Testing
make test          # Run all tests
make testp         # Run tests in parallel (faster)
make TESTS=evennia.commands.tests test  # Specific test module

# Code Formatting (ALWAYS run before committing)
make format        # Auto-format with black and isort
make lint          # Check formatting without changes

# Installation
make install       # Install in development mode
make installextra  # Install with extra dependencies
```

### Code Style Requirements
- **100 character line width**
- **4-space indentation (NO TABS)**
- **PEP 8 compliant** (enforced via black and isort)
- **Google-style docstrings** with Markdown formatting
- **Import order**: Python builtins → Twisted → Django → Evennia → Evennia contrib
- **Naming**: CamelCase for classes; lowercase_with_underscores for functions/variables

### Evennia CLI Commands
```bash
evennia --init mygame      # Initialize new game directory
evennia migrate            # Run database migrations
evennia start/stop/reload  # Manage server
evennia test --keepdb evennia  # Run tests
evennia shell              # Django shell
```

---

## Architecture Overview

### Core Design Philosophy

**Evennia doesn't prescribe game mechanics** - it provides infrastructure (networking, persistence, command handling, object management) and lets developers build their game their way using standard Python.

### Two-Process Architecture (Critical Design)

Evennia runs as **two separate Twisted processes** communicating via AMP:

```
Portal (evennia/server/portal/)     Server (evennia/server/)
├─ Stateless connection handler     ├─ Game engine ("mud driver")
├─ Protocol abstraction              ├─ All game logic & database
│  (Telnet, WebSocket, SSH, etc.)   ├─ Command processing
├─ Can reload without disconnect     ├─ Object/account management
└─ No database access                └─ Can reload while Portal holds connections
```

**Why this matters**: Server can be fully reloaded (code changes) while Portal keeps players connected. This is the foundation for hot-reload functionality.

### Typeclass System (Critical Design Pattern)

**Two-layer pattern**: Django models (persistence) + Typeclasses (game logic)

```
Database Layer (evennia/*/models.py)
├─ ObjectDB, AccountDB, ScriptDB, ChannelDB
└─ Each has db_typeclass_path field → points to Python class

Typeclass Layer (evennia/*/objects.py, accounts.py, scripts.py)
├─ DefaultObject, DefaultAccount, DefaultScript
├─ Instantiated when loaded from DB
└─ Game directory inherits these (mygame/typeclasses/*.py)
```

**Why this matters**: Multiple Python classes share the same database table. Different behavior without schema changes. The `db_typeclass_path` determines which Python class gets instantiated.

**Key concept**:
- `obj.db.attribute` = persistent (stored in Attribute table)
- `obj.ndb.attribute` = non-persistent (in-memory only)
- Override `at_*` hooks for custom behavior (see API docs for full list)

### Command System Architecture

**Unique feature**: CmdSet merging with set theory operations

```
User Input → CmdHandler:
  1. Gather CmdSets from: obj.location, obj, obj.account
  2. Merge with priorities (Union/Intersect/Replace/Remove)
  3. Find matching command
  4. Execute: parse() → func()
```

**Why this matters**: Different game states (darkness, combat, unconscious) can modify available commands dynamically via CmdSet priority and merge types. A higher-priority CmdSet can override/remove/filter lower-priority commands.

**Key files**:
- `evennia/commands/command.py` - Command base class
- `evennia/commands/cmdset.py` - CmdSet container
- `evennia/commands/cmdhandler.py` - Parser and dispatcher

---

## File Structure Reference

### Core Evennia (evennia/)
```
evennia/
├── objects/         # ObjectDB model, DefaultObject typeclass
├── accounts/        # AccountDB model, DefaultAccount typeclass
├── scripts/         # ScriptDB model, DefaultScript, event handlers
├── commands/        # Command system, default commands
├── comms/           # ChannelDB, Msg models
├── typeclasses/     # TypedObject base, attributes, tags
├── locks/           # Lock system
├── prototypes/      # Prototype/spawner system
├── server/          # Server, portal, session management, AMP
├── utils/           # Search, create functions, signals, utilities
└── web/             # Django web, webclient, REST API
```

### Game Directory (created by `evennia --init mygame`)
```
mygame/
├── server/conf/
│   ├── settings.py      # Main config (typeclass paths, cmdsets, etc.)
│   ├── lockfuncs.py     # Custom lock functions
│   └── inputfuncs.py    # Custom input handlers
├── typeclasses/         # Your game logic (inherit from Default*)
│   ├── objects.py, characters.py, rooms.py, exits.py
│   ├── accounts.py, scripts.py, channels.py
├── commands/            # Your custom commands
│   ├── command.py       # Custom Command base
│   └── default_cmdsets.py
├── world/               # Game content
│   ├── prototypes.py    # Object templates
│   └── help_entries.py
└── web/                 # Web customization
```

---

## Key Systems Quick Reference

### Objects & Entity Operations
- **Search**: `from evennia import search_object; obj = search_object("name")`
- **Create**: `from evennia import create_object; obj = create_object(typeclass, key=, location=)`
- **Attributes**: `obj.db.key = value` (persistent), `obj.ndb.key = value` (temporary)
- **Tags**: `obj.tags.add("tag", category="cat")` - searchable categorization
- **Locks**: `obj.locks.add("edit:perm(Builder)")` - access control
- **See**: BUILDING_GAME_CODE.md for detailed patterns, hooks, custom typeclasses

### Scripts & Commands
- **Scripts**: Time-based behavior - `create_script(typeclass, obj=, key=)`
- **Commands**: User input - `from evennia.commands.default.muxcommand import MuxCommand`
- **Command attributes**: `self.switches`, `self.lhs`, `self.rhs`, `self.arglist`
- **See**: BUILDING_GAME_CODE.md for detailed patterns, CmdSets, testing

### Critical: Type-Specific Entity Operations

**CRITICAL**: Dbrefs (`#N`) exist in **separate namespaces** per entity type. Object #5, Script #5, Account #5, and Channel #5 are **four completely different entities**. Using the wrong search function returns empty results without error.

| Entity Type | Tag Search | Dbref Search | Create | Delete |
|-------------|------------|--------------|--------|--------|
| **Objects** | `search.search_object_by_tag()` | `search.search_object("#123")` | `create_object(typeclass, key=, location=)` | `obj.delete()` |
| **Scripts** | `search.search_script_tag()` | `search.search_script("#789")` | `create_script(typeclass, obj=, key=)` | `script.delete()` |
| **Accounts** | `search.search_account_tag()` | `search.search_account("#456")` | `create_account(key, email, password)` | `account.delete()` |
| **Channels** | `search.search_channel_tag()` | `search.search_channel("#012")` | `create_channel(key, desc=)` | `channel.delete()` |

**Rules**:
1. Always use type-specific search functions
2. Document entity type in variable names (`target_script_id` not `target_id`)
3. Wrong function = silent failure (returns empty results)

**CRITICAL for Scripts**: `search.search_tag()` is **ONLY for Objects**. Scripts are NOT Objects (they're ScriptDB, a different database model). Using `search.search_tag()` for scripts will **never find any scripts** - you MUST use `search.search_script_tag()`.

**See**: BUILDING_GAME_CODE.md → Entity Operations for detailed patterns and examples.

---

## Looking Up API Details

When you need specific API information (method signatures, parameters, hook lists, etc.):

**Primary method**: Use `/evapi` to extract API docs locally using Sphinx Napoleon:

```
/evapi evennia/objects/objects.py --class DefaultObject --method move_to
/evapi evennia/objects/objects.py --class DefaultObject
/evapi evennia/utils/search.py --function search_object
/evapi evennia/utils/search.py
```

**Common module paths**:
| Module | Path |
|--------|------|
| Objects | `evennia/objects/objects.py` |
| Scripts | `evennia/scripts/scripts.py` |
| Accounts | `evennia/accounts/accounts.py` |
| Channels | `evennia/comms/comms.py` |
| Commands | `evennia/commands/command.py` |
| CmdSet | `evennia/commands/cmdset.py` |
| Search Utils | `evennia/utils/search.py` |
| Create Utils | `evennia/utils/create.py` |

**Workflow**:
1. Understand the design pattern (this guide)
2. Identify which system to use (typeclass, command, script, etc.)
3. Extract API details with `extract_api.py`
4. Implement in game directory (mygame/)

**Additional resources**:
- Online docs: https://www.evennia.com/docs/latest/
- Tutorials: Check docs for beginner tutorials (5-part series)
- Contrib systems: Community-contributed game systems in `evennia/contrib/`

---

## Testing & Quality

- **Always run `make format`** before committing
- **Run tests**: `make test` or `make testp` (parallel)
- **Specific tests**: `make TESTS=path.to.test test`
- **Keep tests passing** - never commit broken tests
- **Line length**: 100 characters (black enforced)
