---
name: analyzing-tracebacks
description: Analyzing Python tracebacks and error messages in Evennia
---

# Analyzing Tracebacks

This skill covers interpreting Python tracebacks and error messages from Evennia without using the remote debugger. Use this for understanding exceptions, diagnosing errors from logs, and tracing issues back to source code.

---

## Reading Python Tracebacks

### Traceback Structure

```
Traceback (most recent call last):        ← Header (read BOTTOM to TOP for cause)
  File "/path/to/file.py", line 42, in function_name
    some_code_here()                      ← Code that was executing
  File "/path/to/another.py", line 15, in caller
    function_name()                       ← Previous frame in call stack
ExceptionType: Error message here         ← The actual error (START HERE)
```

**Reading order**: Start at the BOTTOM (exception), then read UP to trace the call chain.

### Key Information Per Frame

- **File path**: Which module/file
- **Line number**: Exact location
- **Function name**: Which function was executing
- **Code snippet**: The line that raised (or called the next frame)

---

## Common Evennia Error Patterns

### AttributeError

```python
AttributeError: 'NoneType' object has no attribute 'key'
```

**Meaning**: Variable is `None` when you expected an object.

**Common causes in Evennia**:
- `search_object()` returned empty list, accessed `[0]` on it
- `obj.location` is `None` (object not in any room)
- `obj.account` is `None` (character not connected)
- Database object was deleted but reference still held

**Fix pattern**: Check for None before accessing attributes:
```python
obj = search_object("sword")
if obj:
    obj = obj[0]
    # use obj.key
```

---

### TypeError

**Signature mismatch**:
```python
TypeError: func() takes 2 positional arguments but 3 were given
```
→ Check function signature, count arguments

**Wrong type**:
```python
TypeError: 'int' object is not subscriptable
```
→ Trying `obj[0]` on something that's not a list/dict

**JSON serialization** (common with SaverDict):
```python
TypeError: Object of type _SaverDict is not JSON serializable
```
→ Convert to dict: `dict(obj.db.some_attr)` or use `evennia.utils.dbserialize.from_db()`

---

### ImportError / ModuleNotFoundError

```python
ModuleNotFoundError: No module named 'mymodule'
```

**Common causes**:
- Typo in import path
- Module not in `INSTALLED_APPS` (for Django apps)
- Missing `__init__.py` in package directory
- Not installed in virtualenv

**Evennia-specific**: Check `mygame/server/conf/settings.py` for correct paths.

---

### AppRegistryNotReady

```python
django.core.exceptions.AppRegistryNotReady: Apps aren't loaded yet.
```

**Cause**: Importing Django models before `django.setup()` completes.

**Fix**: Use lazy imports in module-level code:
```python
# BAD - at module level
from evennia.objects.models import ObjectDB

# GOOD - inside function
def my_function():
    from evennia.objects.models import ObjectDB
```

---

### KeyError

```python
KeyError: 'missing_key'
```

**Common causes**:
- Accessing `dict["key"]` when key doesn't exist
- Accessing `obj.db.attr` that was never set (returns `None`, but accessing nested returns KeyError)

**Fix**: Use `.get()` with default:
```python
value = mydict.get("key", default_value)
```

---

### DoesNotExist / MultipleObjectsReturned

```python
evennia.objects.models.ObjectDB.DoesNotExist: ObjectDB matching query does not exist
```

**Cause**: Using `.get()` on queryset when no match (or multiple matches).

**Fix**: Use `.filter()` and check results, or wrap in try/except:
```python
# Option 1: filter
results = ObjectDB.objects.filter(db_key="sword")
if results.exists():
    obj = results.first()

# Option 2: try/except
try:
    obj = ObjectDB.objects.get(db_key="sword")
except ObjectDB.DoesNotExist:
    obj = None
```

---

## Evennia-Specific Traceback Patterns

### Command Execution Errors

Look for frames in:
- `evennia/commands/cmdhandler.py` - Command dispatch
- `evennia/commands/command.py` - Command base class
- `mygame/commands/*.py` - Your custom commands

**Key variables to check**:
- `self.args`, `self.lhs`, `self.rhs` - Parsed arguments
- `self.caller` - Who executed command
- `self.obj` - Object command is attached to

### Script Execution Errors

Look for frames in:
- `evennia/scripts/scripts.py` - Script base class
- `evennia/scripts/tickerhandler.py` - Ticker system
- `mygame/typeclasses/scripts.py` - Your scripts

**Key context**:
- `at_repeat()` - Called on each tick
- `at_start()` / `at_stop()` - Lifecycle hooks

### Typeclass Instantiation Errors

```python
evennia.typeclasses.attributes.AttributeError: Cannot access attributes on non-saved object
```

**Cause**: Accessing `obj.db.*` before object is saved to database.

**Fix**: Ensure object is created properly with `create_object()` or saved before attribute access.

---

## Tracing Errors to Source

### Step 1: Identify the Exception Type

The last line tells you WHAT went wrong. Common types:
- `AttributeError` - Missing attribute
- `TypeError` - Wrong type or signature
- `ValueError` - Right type, wrong value
- `KeyError` - Missing key in dict
- `ImportError` - Import failed

### Step 2: Find YOUR Code in the Traceback

Scan for paths containing `mygame/` or your project directory. This is usually where the bug is (not in Evennia core).

### Step 3: Examine the Context

Look at the code line shown. What variables are involved? Trace them back up the call stack to find where they got their (wrong) value.

### Step 4: Reproduce Minimally

If possible, isolate the issue:
```python
# In evennia shell
from mygame.typeclasses.objects import MyObject
# Reproduce the exact call that failed
```

---

## Using `/evapi` for Error Context

When you see an error in Evennia code, use `/evapi` to understand what the function expects:

```
/evapi evennia/objects/objects.py --class DefaultObject --method move_to
```

This shows:
- Expected parameters
- What the function does
- What it returns
- Common failure modes

---

## Log Analysis Tips

### Evennia Log Locations

- `mygame/server/logs/server.log` - Main server log
- `mygame/server/logs/portal.log` - Connection handling
- Console output when running `evennia start -l` (foreground)

### Searching Logs for Errors

```bash
# Find all tracebacks
grep -A 20 "Traceback" server.log

# Find specific error type
grep -B 5 -A 15 "AttributeError" server.log

# Recent errors (last 100 lines)
tail -100 server.log | grep -A 20 "Traceback"
```

### Timestamp Correlation

Match log timestamps to when the error occurred. Evennia logs include timestamps - use them to find related events before the error.

---

## Cheat Sheet: Error → Likely Cause

| Error Message Fragment | Likely Cause |
|------------------------|--------------|
| `'NoneType' has no attribute` | Search returned empty, or obj deleted |
| `is not JSON serializable` | SaverDict not converted to dict |
| `Apps aren't loaded yet` | Module-level Django import |
| `takes N arguments but M were given` | Function signature mismatch |
| `not subscriptable` | Trying to index non-sequence |
| `DoesNotExist` | Database query found nothing |
| `MultipleObjectsReturned` | `.get()` found too many |
| `Cannot access attributes on non-saved` | Accessing db.* before save |
| `CmdSet has no command` | Command not properly added to cmdset |
| `Lock string malformed` | Syntax error in lock definition |

