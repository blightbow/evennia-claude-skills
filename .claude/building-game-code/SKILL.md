---
name: building-game-code
description: Task-driven guide for backend game development: objects, scripts, commands, and entity operations
---

# Building Game Code

## Creating Custom Objects

Build custom game objects by inheriting from Default typeclasses.

### Basic Object Pattern

```python
 mygame/typeclasses/objects.py
from evennia.objects.objects import DefaultObject

class Object(DefaultObject):
    def at_object_creation(self):
        super().at_object_creation()
        self.db.weight = 1.0
        self.db.condition = 100

    def at_get(self, getter):
        getter.msg(f"You pick up {self.key}.")

    def return_appearance(self, looker, **kwargs):
        text = super().return_appearance(looker, **kwargs)
        if self.db.condition < 50:
            text += "\n|rIt looks damaged.|n"
        return text
```

**Common hooks**: `at_object_creation()`, `at_get()`, `at_drop()`, `return_appearance()`, `at_desc()`

**See**: `extract_api.py evennia/objects/objects.py --class DefaultObject` for full hook list

### Character Pattern

```python
 mygame/typeclasses/characters.py
from evennia.objects.objects import DefaultCharacter

class Character(DefaultCharacter):
    def at_object_creation(self):
        super().at_object_creation()
        self.db.health = 100
        self.db.stamina = 100
        self.db.skills = {"combat": 0, "crafting": 0}

    def at_before_move(self, destination, **kwargs):
        """Return False to prevent movement."""
        if self.db.stamina < 10:
            self.msg("You're too tired to move.")
            return False
        return True

    def at_after_move(self, source_location, **kwargs):
        self.db.stamina -= 5
```

### Room Pattern

```python
 mygame/typeclasses/rooms.py
from evennia.objects.objects import DefaultRoom

class Room(DefaultRoom):
    def at_object_creation(self):
        super().at_object_creation()
        self.db.temperature = 20
        self.db.weather = "clear"

    def return_appearance(self, looker, **kwargs):
        text = super().return_appearance(looker, **kwargs)
        if self.tags.get("outdoor", category="room_type"):
            text += f"\n|cThe weather is {self.db.weather}.|n"
        return text
```

---

## Creating Custom Scripts

Build game systems with periodic behavior, timers, and event handling.

### Periodic Script Pattern

```python
 mygame/typeclasses/scripts.py
from evennia.scripts.scripts import DefaultScript

class WeatherScript(DefaultScript):
    def at_script_creation(self):
        self.key = "weather_system"
        self.interval = 300  # Seconds between calls
        self.persistent = True  # Survives server reload
        self.db.current_weather = "clear"

    def at_repeat(self):
        """Called every self.interval seconds."""
        import random
        from evennia.utils import search

        self.db.current_weather = random.choice(["clear", "cloudy", "rainy"])
        outdoor_rooms = search.search_tag("outdoor", category="room_type")

        for room in outdoor_rooms:
            room.db.weather = self.db.current_weather
            room.msg_contents(f"|cWeather changes to {self.db.current_weather}.|n")
```

### Initialization Pattern

**Critical**: Set defaults unconditionally in `at_script_creation()` - no early returns or attribute checks.

**Why**: Attributes passed via `create_script(attributes=[...])` are applied AFTER `at_script_creation()` runs (line 463 in scripts.py). Early returns skip initialization.

```python
 ✓ CORRECT - Unconditional defaults
def at_script_creation(self):
    self.key = "my_script"
    self.db.my_attribute = None  # Set by attributes= after creation
    self.db.counter = 0

 ✗ WRONG - Early return skips initialization
def at_script_creation(self):
    if not self.db.my_attribute:  # Not set yet!
        return  # Skips ALL remaining initialization
    self.db.counter = 0  # Never runs for new scripts
```

**Pattern**: Kwargs (`key=`, `interval=`) override defaults automatically. Trust the framework.

**Creating scripts**:

```python
from evennia import create_script

 Create and start
weather = create_script("typeclasses.scripts.WeatherScript")

 Attach to an object
ai_script = create_script("typeclasses.scripts.AIScript", obj=npc)

 One-shot delayed task
from evennia.scripts.taskhandler import delay
delay(5, callback=my_function, arg1, arg2)
```

### Async Script Pattern (API Calls)

**For scripts making async API calls**, use Twisted's `@inlineCallbacks`:

```python
from twisted.internet.defer import inlineCallbacks

 ✓ CORRECT - Uses @inlineCallbacks WITH yield
@inlineCallbacks
def at_repeat(self):
    response = yield self.call_external_api()  # async call
    self.process_response(response)

 ✗ WRONG - @inlineCallbacks WITHOUT yield
@inlineCallbacks
def build_data(self):
    data = {"key": "value"}
    return data  # TypeError!

 ✓ CORRECT - No @inlineCallbacks for sync functions
def build_data(self):
    data = {"key": "value"}
    return data
```

**Rule**: Only use `@inlineCallbacks` if function contains `yield` statements.

**Error**: Using `@inlineCallbacks` without `yield` will raise `TypeError: inlineCallbacks requires ... to produce a generator`.

**See**: `extract_api.py evennia/scripts/scripts.py --class DefaultScript`

---

## Non-Persistent Attributes (NDB)

**Critical**: NDB attributes are wiped on server reload. Always use defensive initialization.

**Use `ndb` for**: Temp caching, network connections (non-serializable), message buffers, working memory
**Use `db` for**: Persistent state, configuration, long-term memory

### Defensive Patterns

```python
 ✓ Lazy initialization (NDB)
def at_msg_receive(self, text=None, **kwargs):
    # NDB: Temporary buffer, wiped after batch processing
    if not hasattr(self.ndb, 'message_buffer') or self.ndb.message_buffer is None:
        self.ndb.message_buffer = []
    self.ndb.message_buffer.append(text)

 ✓ hasattr() check (NDB)
if not hasattr(script.ndb, 'rag_client') or not script.ndb.rag_client:
    return {"error": "Client not initialized"}

 ✓ getattr() with default (NDB)
provider = getattr(self.ndb, 'last_provider', 'default')

 ✓ getattr() with explicit None check (DB) - CRITICAL for optional config
buffer = getattr(character.db, "context_buffer", [])
buffer = buffer if buffer is not None else []  # Attribute exists but is None
count = len(buffer)

 ✗ WRONG - Crashes after reload (NDB)
self.ndb.buffer.append(data)  # AttributeError: 'NoneType' has no attribute 'append'

 ✗ WRONG - Crashes if attribute exists but is None (DB)
buffer = getattr(character.db, "context_buffer", [])
count = len(buffer)  # TypeError: object of type 'NoneType' has no len()
```

**Critical**: `getattr(obj.db, 'attr', default)` returns `None` if attribute EXISTS but is `None`, not the default. Always check for None on optional DB attributes.

### Recreate Pattern (Scripts)

```python
class MyScript(DefaultScript):
    def at_script_creation(self):
        # NDB: Connection recreated via at_start() on reload
        self.ndb.api_client = None

    def at_start(self):
        """Called on startup AND reload."""
        super().at_start()
        if self.db.api_enabled:
            self.ndb.api_client = APIClient(
                host=self.db.api_host,  # Config in db
                port=self.db.api_port
            )
```

**Rules**: Always comment why ndb chosen; use `at_start()` to recreate connections; store config in db, connections in ndb

**See**: [Evennia Attributes Docs](https://www.evennia.com/docs/latest/Components/Attributes.html#in-memory-attributes-nattributes)

---

## Creating Custom Commands

Build commands with MuxCommand for automatic parsing of switches, arguments, and delimiters.

### MuxCommand Basics

**Use MuxCommand for**:
- Commands with switches (`/create`, `/delete`)
- Commands with `=` delimiter (`set key = value`)
- Commands needing `arglist`, `lhs`, `rhs` parsing

**Critical**: Import from `evennia.commands.default.muxcommand import MuxCommand`, NOT `evennia.commands.command import Command`. Using base `Command` when you need `arglist`/`switches`/`lhs`/`rhs` will cause `AttributeError`.

**Command syntax**: `name[/switch[/switch..]] arg1[,arg2,...] [[=] arg[,..]]`

**Parsing attributes**:
- `self.switches` - List of switches (e.g., `["create", "verbose"]`)
- `self.lhs` - Left side of `=` delimiter
- `self.rhs` - Right side of `=` delimiter
- `self.lhslist` - LHS split by commas
- `self.rhslist` - RHS split by commas
- `self.arglist` - Space-separated arguments

**Example parsing**:
```
Input: channel/create mychannel;mc = A new channel
→ switches = ["create"]
→ lhs = "mychannel;mc"
→ rhs = "A new channel"
→ lhslist = ["mychannel", "mc"]
```

**Warning**: Don't manually parse with `self.args.split()` or `"=" in self.args`. Use MuxCommand's built-in attributes (`lhs`, `rhs`, `arglist`, etc.) - manual parsing breaks switch and delimiter handling.

### Basic Command Template

```python
 mygame/commands/command.py
from evennia.commands.default.muxcommand import MuxCommand

class CmdCustom(MuxCommand):
    """
    Custom command with switch support.

    Usage:
      mycommand <target>
      mycommand/verbose <target>
      mycommand <name> = <value>
    """
    key = "mycommand"
    aliases = ["mc"]
    locks = "cmd:all()"

    def func(self):
        if "verbose" in self.switches:
            self.caller.msg("Verbose mode enabled")

        if self.rhs:
            # Format: mycommand name = value
            self.caller.msg(f"Setting {self.lhs} to {self.rhs}")
        elif self.arglist:
            # Format: mycommand target
            target = self.arglist[0]
            self.caller.msg(f"Target: {target}")
```

### Common Command Patterns

**Pattern 1: Switch-based subcommands**

```python
def func(self):
    if not self.switches:
        self.show_default()
    elif "create" in self.switches:
        self.create_object()
    elif "delete" in self.switches:
        self.delete_object()
```

**Pattern 2: Key=value syntax**

```python
def func(self):
    if self.rhs:
        # Format: command key = value
        key = self.lhs.strip()
        value = self.rhs.strip()
        self.caller.db.settings[key] = value
    else:
        # Format: command key (show value)
        key = self.lhs.strip()
        self.caller.msg(f"{key}: {self.caller.db.settings.get(key)}")
```

**Pattern 3: Comma-separated lists**

```python
def func(self):
    # Format: command item1, item2, item3
    items = self.lhslist  # Auto-split by commas
    for item in items:
        self.process(item)
```

### Critical: lhs vs arglist

**For commands with `=` delimiter syntax, use `self.lhs`, NOT `self.arglist[0]`**

```python
 ✗ WRONG - arglist contains unparsed string
def func(self):
    target = self.arglist[0]  # "senku=confirm" (unparsed!)

 ✓ CORRECT - lhs is parsed
def func(self):
    target = self.lhs.strip()  # "senku" (parsed!)
    confirmation = self.rhs.strip() if self.rhs else None
```

**Why**: MuxCommand parses `=` delimiter BEFORE space-splitting:
- Input: `"senku=confirm"` → `lhs="senku"`, `rhs="confirm"`, `arglist=["senku=confirm"]`

---

## Entity Operations

Search, create, and manipulate game entities with type-specific functions.

### Type-Specific Search Functions

**Critical**: Dbrefs exist in separate namespaces per type. Object #5 ≠ Script #5 ≠ Account #5. Always use type-specific search functions.

### Search Patterns

**Error**: Using wrong search function (e.g., `search_object("#789")` for a script) returns empty results without error - silent failure.

```python
from evennia.utils import search

 ✓ CORRECT - Type-specific search
obj = search.search_object("#123")
script = search.search_script("#789")

 ✓ SAFE - Document type in variable names
self.db.target_script_id = 123
script = search.search_script(f"#{self.db.target_script_id}")

 ✗ WRONG - Generic variable name
self.db.target_id = 123  # Which table?
obj = search.search_object(f"#{self.db.target_id}")  # Assumes ObjectDB!
```

### Tag Patterns

```python
 Add tags
obj.tags.add("weapon", category="item_type")
obj.tags.add("sword", category="weapon_type")

 ✓ CORRECT - Type-specific tag searches
weapons = search.search_object_by_tag("weapon", category="item_type")
scripts = search.search_script_tag("ai_assistant", category="system")
accounts = search.search_account_tag("admin", category="user_type")

 ✗ WRONG - search.search_tag() is ONLY for Objects
from evennia.scripts.models import ScriptDB
tagged = search.search_tag("ai_assistant", category="system")
scripts = [obj for obj in tagged if isinstance(obj, ScriptDB)]  # WILL NOT FIND SCRIPTS!
```

**CRITICAL**: `search.search_tag()` is **ONLY for Objects**. Scripts are NOT Objects (they're ScriptDB, a different model).

**ALWAYS use type-specific tag search functions**:
- **Scripts**: `search.search_script_tag()` - Scripts are NOT Objects, generic search won't find them
- **Objects**: `search.search_object_by_tag()` (or `search.search_tag()` - same function)
- **Accounts**: `search.search_account_tag()`
- **Channels**: `search.search_channel_tag()`

**Common mistake**: Trying to use `search.search_tag()` + isinstance filtering for scripts. This cannot work because `search.search_tag()` only searches ObjectDB.

---

## Prototypes & Spawning

Create object templates for spawning multiple similar objects.

### Prototype Pattern

```python
 mygame/world/prototypes.py

SWORD = {
    "prototype_key": "sword",
    "prototype_parent": "weapon",  # Inherit from other prototypes
    "typeclass": "typeclasses.objects.Weapon",
    "key": "iron sword",
    "aliases": ["sword"],
    "desc": "A simple iron sword.",
    "damage": 10,
    "weight": 3.0,
    "prototype_tags": [("weapon", "item_type"), ("melee", "weapon_type")],
    "locks": "get:all();equip:has_skill(combat, 1)",
}

MAGIC_SWORD = {
    "prototype_key": "magic_sword",
    "prototype_parent": "sword",  # Inherit and override
    "key": "enchanted blade",
    "damage": 15,
    "magical_damage": 5,
}
```

### Spawning

```python
from evennia.prototypes import spawner

 Spawn single object
sword = spawner.spawn("sword", location=character)

 Spawn multiple
swords = spawner.spawn("sword", 5, location=room)

 Spawn with overrides
custom_sword = spawner.spawn({
    "prototype_parent": "sword",
    "key": "custom blade",
    "damage": 20,
}, location=character)
```

---

## CmdSet Pattern

Register commands and control availability with CmdSet merging.

### Basic CmdSet

```python
 mygame/commands/default_cmdsets.py
from evennia import CmdSet
from commands.combat import CmdAttack, CmdDefend

class CharacterCmdSet(CmdSet):
    """Commands available to all characters."""
    key = "DefaultCharacter"
    priority = 0
    mergetype = "Union"  # Add to existing commands

    def at_cmdset_creation(self):
        self.add(CmdAttack())
        self.add(CmdDefend())
```

### Dynamic CmdSet (Combat Example)

```python
class CombatCmdSet(CmdSet):
    """Higher priority commands during combat."""
    key = "CombatCmdSet"
    priority = 10  # Overrides lower priority commands
    mergetype = "Replace"

    def at_cmdset_creation(self):
        from commands.combat import CmdCombatAttack, CmdFlee
        self.add(CmdCombatAttack())
        self.add(CmdFlee())

 Add/remove dynamically
def enter_combat(character):
    character.cmdset.add(CombatCmdSet)

def leave_combat(character):
    character.cmdset.remove(CombatCmdSet)
```

**Merge types**: `Union` (add), `Intersect` (filter), `Replace` (override), `Remove` (subtract)

---

## Custom Lock Functions

Create custom access control logic.

```python
 mygame/server/conf/lockfuncs.py

def has_skill(accessing_obj, accessed_obj, *args, **kwargs):
    """
    Check if character has minimum skill level.
    Usage: locks = "equip:has_skill(combat, 5)"
    """
    if not args or len(args) < 2:
        return False
    skill_name = args[0]
    try:
        min_level = int(args[1])
    except (ValueError, TypeError):
        return False
    skills = accessing_obj.db.skills or {}
    return skills.get(skill_name, 0) >= min_level

def in_combat(accessing_obj, accessed_obj, *args, **kwargs):
    """Check if in combat. Usage: locks = "cmd:in_combat()" """
    return accessing_obj.ndb.in_combat or False
```

**Using locks**:

```python
obj.locks.add("equip:has_skill(combat, 5)")

class CmdAdvancedAttack(MuxCommand):
    locks = "cmd:has_skill(combat, 10)"

room.locks.add("enter:has_skill(climbing, 3)")
```

---

## Testing Patterns

### Database-First Testing (Preferred)

**Always use real database objects to test Evennia code**. Using `MagicMock()` for characters/scripts hides SaverDict behavior bugs.

```python
from evennia.utils.test_resources import BaseEvenniaTest
from evennia.utils import create

class TestCustomObject(BaseEvenniaTest):
    """
    BaseEvenniaTest provides: self.char1, self.char2, self.room1, self.room2, self.account
    """

    def setUp(self):
        super().setUp()
        from typeclasses.objects import CustomObject
        self.obj = create.create_object(CustomObject, key="test", location=self.room1)

    def tearDown(self):
        if self.obj:
            self.obj.delete()
        super().tearDown()

    def test_attribute_persistence(self):
        """Test SaverDict properly persists nested data."""
        self.obj.db.config = {"key": "value"}
        # Verify via fresh lookup
        self.assertEqual(self.obj.db.config["key"], "value")
```

### Anti-Pattern: MagicMock for Evennia Objects

```python
# ✗ WRONG - MagicMock hides SaverDict behavior
class BadTest(TestCase):
    def setUp(self):
        self.character = MagicMock()
        self.character.db.profiles = {}  # Not a real SaverDict!

# ✓ CORRECT - Real objects test real behavior
class GoodTest(BaseEvenniaTest):
    def setUp(self):
        super().setUp()
        self.character = create.create_object(MyCharacter, key="Test", location=self.room1)
```

**When mocks ARE appropriate**: External services (HTTP APIs, Qdrant, LLM providers)

### SaverDict Gotchas

**Nested modification requires reassignment to trigger save**:

```python
# ✗ WRONG - Modification not persisted
profile = character.db.entity_profiles["#123"]
profile["attributes"]["key"] = "value"  # Lost on reload!

# ✓ CORRECT - Reassign parent to trigger save
profiles = character.db.entity_profiles
profiles["#123"]["attributes"]["key"] = "value"
character.db.entity_profiles = profiles  # Triggers save
```

**isinstance(x, dict) fails for SaverDict proxies**:

```python
# ✗ WRONG - SaverDict proxy is not isinstance(dict)
if isinstance(profile["attributes"], dict):  # False!
    profile["attributes"]["key"] = value

# ✓ CORRECT - Check for dict-like behavior
if hasattr(profile.get("attributes"), "__setitem__"):
    profile["attributes"]["key"] = value
```

### Async Testing with Twisted

For async code using `@inlineCallbacks`, use `trial_unittest.TestCase` with real DB objects:

```python
import pytest
from twisted.internet import defer, reactor
from twisted.trial import unittest as trial_unittest
from evennia.utils import create

@pytest.mark.django_db(transaction=True)  # Required for DB access!
class AsyncTestBase(trial_unittest.TestCase):
    """Base for async tests with real database objects."""

    def setUp(self):
        from evennia.objects.objects import DefaultRoom
        # nohome=True required - DEFAULT_HOME doesn't exist in fresh test DB
        self.room = create.create_object(DefaultRoom, key="TestRoom", nohome=True)
        self.character = create.create_object(MyCharacter, key="Test", location=self.room)

    def tearDown(self):
        # Delete in reverse creation order
        if self.character:
            try:
                self.character.delete()
            except Exception:
                pass
        if self.room:
            try:
                self.room.delete()
            except Exception:
                pass

        # Cancel pending reactor calls (prevents DirtyReactorAggregateError)
        for call in reactor.getDelayedCalls():
            if call.active():
                try:
                    call.cancel()
                except Exception:
                    pass

class TestAsyncFeature(AsyncTestBase):
    @defer.inlineCallbacks
    def test_async_operation(self):
        result = yield my_async_function(self.character)
        self.assertTrue(result["success"])
```

**Key**: `trial_unittest.TestCase` works with Django's test database - no special integration needed.

### Required Configuration

**pytest.ini** - Django settings must be configured:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = evennia.settings_default
python_files = test_*.py
```

**conftest.py** - SESSION_HANDLER must be initialized for `BaseEvenniaTest`:

```python
from unittest.mock import MagicMock
import pytest
import evennia
from evennia.server.sessionhandler import ServerSessionHandler

@pytest.fixture(scope="session", autouse=True)
def init_session_handler():
    """BaseEvenniaTest requires SESSION_HANDLER with data_out/disconnect."""
    if evennia.SESSION_HANDLER is None:
        evennia.SESSION_HANDLER = ServerSessionHandler()
        evennia.SESSION_HANDLER.data_out = MagicMock()
        evennia.SESSION_HANDLER.disconnect = MagicMock()
    yield
```

**Without conftest.py**: `AttributeError: 'NoneType' has no attribute 'data_out'` in `BaseEvenniaTest.setUp()`

### Run Tests

```bash
make test                                    # All tests
make testp                                   # Parallel (faster)
make TESTS=mygame.tests.test_objects test    # Specific module
.venv/bin/python -m pytest path/to/test.py -v  # pytest directly
```

---

## API References

```
/evapi evennia/commands/command.py --class Command
/evapi evennia/objects/objects.py --class DefaultObject
/evapi evennia/scripts/scripts.py --class DefaultScript
/evapi evennia/utils/search.py
```
