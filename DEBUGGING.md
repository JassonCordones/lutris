# Debugging Lutris

This guide covers all the ways to debug Lutris during development. Whether you're
investigating a crash, tracing a game launch failure, or stepping through installer
code — you'll find the right tooling here.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Logging System](#logging-system)
- [VS Code Debugging](#vs-code-debugging)
- [Python Debugger (pdb/breakpoint)](#python-debugger-pdbbreakpoint)
- [gdb for Segfaults](#gdb-for-segfaults)
- [Debugging Game Launches](#debugging-game-launches)
- [Debugging Installers](#debugging-installers)
- [Environment Variables](#environment-variables)
- [Test Suite](#test-suite)
- [Static Analysis & Linting](#static-analysis--linting)
- [Common Pitfalls](#common-pitfalls)

---

## Quick Start

The fastest way to see what Lutris is doing:

```bash
# Run with debug output (verbose console logging)
./bin/lutris -d

# Tail the log file (always DEBUG level, even without -d)
tail -f ~/.cache/lutris/lutris.log
```

The log file is a rotating log at `~/.cache/lutris/lutris.log` (5 backups).
It always captures `DEBUG`-level messages — no flag needed.

---

## Logging System

Lutris uses Python's `logging` module with a **dual-handler** setup defined in
[`lutris/util/log.py`](lutris/util/log.py):

### Two Handlers

| Handler | Output | Default Level | Format |
|---|---|---|---|
| **File handler** | `~/.cache/lutris/lutris.log` | `DEBUG` (always) | `[LEVEL:TIMESTAMP:MODULE]: message` |
| **Console handler** | `stderr` | `INFO` (`DEBUG` with `-d`) | Short timestamp format |

### Debug Format (with `-d`)

When you pass `-d`, the console output switches to a detailed format:

```
DEBUG    2026-06-27 00:40:00,123 [module.function:123]:message
```

### Logger Usage

All Lutris modules should use the shared logger:

```python
from lutris.util.log import logger

logger.debug("Detailed info for development")
logger.info("General information")
logger.warning("Something unexpected")
logger.error("Something went wrong: %s", ex)
```

### Adding Logging to New Code

Always use `%s`-formatting with the logger — never use f-strings or `.format()`:

```python
# ✅ Correct
logger.debug("Processing game %s with runner %s", game_id, runner)

# ❌ Wrong
logger.debug(f"Processing game {game_id} with runner {runner}")
```


---

## VS Code Debugging

> **Prerequisites**: Install the [Python extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python)
> for VS Code.

This project includes a `.vscode/launch.json` with several pre-configured debug
profiles. Open the **Run and Debug** view (`Ctrl+Shift+D` / `Cmd+Shift+D`) to
see them.

### Configurations

| Configuration | What It Does |
|---|---|
| **Lutris (Debug Mode)** | Runs Lutris with `-d`, verbose logging, all breakpoints active |
| **Lutris (Normal Mode)** | Runs Lutris without verbose console logging (INFO level) |
| **Lutris: Install Game** | Prompts for a YAML/JSON installer file path and runs `-i` |
| **Lutris: Launch Game by Slug** | Prompts for a game slug and launches via `lutris:rungame/<slug>` |
| **Lutris (with Experimental Features)** | Sets `LUTRIS_EXPERIMENTAL_FEATURES_ENABLED=1` |
| **Debug Current Test File** | Runs the currently open test file under the debugger (nose2) |
| **Run All Tests** | Runs the full nose2 test suite |
| **Run Tests with Coverage** | Runs tests and generates HTML coverage report |
| **Python: Attach to Running Lutris Process** | Attaches to an already-running process by PID |
| **Python: Debug with gdb** | Launches Lutris under gdb for C-level crash analysis |

### Using Breakpoints

1. Open the file you want to debug
2. Click in the gutter (left of line numbers) to set a breakpoint (red dot)
3. Select a configuration from the dropdown in the Run & Debug view
4. Press `F5` to start debugging
5. Use the debug toolbar to step (`F10`), step into (`F11`), or continue (`F5`)

**Note on GTK main loops**: The GTK main loop (`Gtk.main()`) blocks the event
loop. When the debugger hits a breakpoint, the GTK UI will freeze — this is
normal. The UI resumes when you continue execution.

### Attaching to a Running Instance

If Lutris is already running and you want to inspect a specific scenario:

1. Run Lutris normally: `./bin/lutris`
2. In VS Code, select **"Python: Attach to Running Lutris Process"** from the
   debug configuration dropdown
3. Press `F5` — a process picker will appear
4. Search for and select the `lutris` process (or `python3 ./bin/lutris`)
5. Set breakpoints and continue debugging

> **Tip**: You may need elevated permissions to attach to processes you didn't
> launch from VS Code, depending on your kernel's `ptrace` settings.

---

## Python Debugger (pdb/breakpoint)

You can use Python's built-in debugger anywhere in the code:

```python
# Python 3.7+ — insert anywhere in lutris source code
breakpoint()
```

When Lutris hits this line, execution will pause in the terminal where Lutris
was launched and you'll get a `(Pdb+)` prompt.

### Useful pdb Commands

| Command | Description |
|---|---|
| `n` (next) | Execute next line, stay in current function |
| `s` (step) | Step into the function call |
| `c` (continue) | Continue execution until next breakpoint |
| `l` (list) | Show source code around current line |
| `p variable` | Print a variable's value |
| `pp variable` | Pretty-print a complex variable |
| `w` (where) | Show the call stack |
| `q` (quit) | Exit the debugger and terminate |

### Using pdb with GTK Apps

The GTK main loop makes pdb slightly trickier — but still works:

```bash
# Run with debug output and breakpoint() available
./bin/lutris -d
```

When `breakpoint()` is hit:
1. The GTK window will freeze (this is normal)
2. Switch to the terminal where you ran Lutris — the `(Pdb+)` prompt is there
3. Use pdb commands, then `c` to continue — the UI resumes

### Conditional Breakpoints

```python
if game_slug == "quake":
    breakpoint()  # Only stops for Quake
```



---

## gdb for Segfaults

If Lutris crashes with a segmentation fault (often caused by GTK, GStreamer,
or Wine native libraries), catch it with gdb:

```bash
gdb -ex r --args "/usr/bin/python3" "./bin/lutris"
```

Or with arguments:

```bash
gdb -ex r --args "/usr/bin/python3" "./bin/lutris" -d
```

When the crash happens, gdb will show the backtrace. Useful commands at the
`(gdb)` prompt:

| Command | Description |
|---|---|
| `bt` (backtrace) | Show the call stack at crash point |
| `bt full` | Show backtrace with local variables |
| `frame N` | Switch to frame N in the backtrace |
| `info locals` | Show local variables in current frame |
| `list` | Show source code around current instruction |

### Getting a Stack Trace from a Core Dump

```bash
# Enable core dumps
ulimit -c unlimited

# Run Lutris until it crashes
./bin/lutris

# Analyze the core dump
gdb /usr/bin/python3 core -ex bt -ex quit
```


---

## Debugging Game Launches

Game launches involve a **subprocess** (`lutris-wrapper`) that monitors the game
process tree. This makes debugging more complex.

### The Launch Chain

```
Lutris Application → MonitoredCommand → lutris-wrapper → Game Process
```

### Logging Game Launches

```bash
# Run Lutris in debug mode first
./bin/lutris -d

# Then launch your game from the GUI — all subprocess output
# is captured in the log
tail -f ~/.cache/lutris/lutris.log
```

### The lutris-wrapper Script

The wrapper at [`share/lutris/bin/lutris-wrapper`](share/lutris/bin/lutris-wrapper)
is a Python script that:

- Sets itself as a **subreaper** to catch orphaned processes
- Monitors for the game process to start
- Gently closes game processes when the main game exits

When running from a dev source tree, it automatically sets its own log level to
`DEBUG` (line 44). You can also edit it to add `breakpoint()` calls if needed.

### Return Codes

Game return codes are written to `/tmp/lutris-<GAME_UUID>` (or the path in
`LUTRIS_RETURN_CODE_FILE`). Check this file after a game exits:

```bash
cat /tmp/lutris-*
```


---

## Debugging Installers

Installers are YAML/JSON scripts that automate game installations. To debug an
installer:

```bash
# Run with debug logging and install a specific script
./bin/lutris -d -i /path/to/installer.yml
```

Or use the VS Code **"Lutris: Install Game"** configuration which prompts for
the installer file path.

### Installer Script Debugging

The installer interpreter lives in [`lutris/installer/`](lutris/installer/). Key
files:

| File | Purpose |
|---|---|
| `interpreter.py` | Main installer script interpreter |
| `commands.py` | Individual installer commands (mkdir, extract, etc.) |
| `installer.py` | High-level installer orchestration |

Add `breakpoint()` calls in these files to step through installation logic.

---

## Environment Variables

Useful environment variables for debugging:

| Variable | Effect |
|---|---|
| `LUTRIS_EXPERIMENTAL_FEATURES_ENABLED=1` | Enable experimental/under-development features |
| `LUTRIS_ALLOW_LOCAL_PYTHON_PACKAGES=1` | Allow loading Python packages from `/home` directories |
| `LUTRIS_RETURN_CODE_FILE=/path/to/file` | Custom path for game return codes |
| `LUTRIS_GAME_UUID=<uuid>` | Game UUID (set automatically by Lutris for subprocess) |
| `PYTHONPATH=/path/to/lutris` | Ensure Python finds the Lutris source tree |
| `DISPLAY=:0` | X11 display to use |
| `WEBKIT_DISABLE_DMABUF_RENDERER=1` | Disabled by Lutris automatically for WebKit compatibility |

---

## Test Suite

### Running Tests

```bash
# Run all tests
make test

# Run a specific test file
nose2 tests._test_utils

# Run with verbose output
nose2 -v

# Run with coverage
make cover
# Then open tests/coverage/index.html in your browser
```

### Writing Tests

Test files follow the pattern `_test_*.py` in the `tests/` directory (configured
in `unittest.cfg`). Existing test files include:

- `_test_api.py`
- `_test_installer.py`
- `_test_utils.py`
- `_test_pga.py`
- `_test_wine.py`
- `_test_resources.py`

### Test Configuration

- Test runner: **nose2** (configured in [`unittest.cfg`](unittest.cfg))
- DB fixture: `tests/fixtures/pga.db` (sqlite, auto-created/removed)
- Plugins: `tests/nose2_plugins/gtk_version` and `ci_exclude_test`

### Debugging Tests

Use the VS Code **"Debug Current Test File"** or **"Run All Tests"**
configurations. Set breakpoints in test code or production code that tests
exercise.


---

## Static Analysis & Linting

Run before submitting code:

```bash
# Style check only
make sc              # or `make style` or `make styles`

# Format your code (auto-fix imports and formatting)
make format

# Full static analysis (ruff lint + mypy + syntax check + translation check)
make check

# Or individually:
make ruff_lint       # ruff check
make syntax-compat   # compileall syntax check
make mypy            # type checking

# For mypy baseline (when adding new typing errors intentionally)
make mypy-reset-baseline
```


---

## Common Pitfalls

### 1. PyGObject / GTK Import Failures

PyGObject is **not** installable via `pip` for the C bindings — you must install
it from your system package manager:

```bash
# Arch
sudo pacman -S python-gobject gtk3

# Debian/Ubuntu
sudo apt install python3-gi python3-gi-cairo gir1.2-gtk-3.0

# Fedora
sudo dnf install python3-gobject gtk3
```

### 2. Virtualenv Limitations

A virtualenv is **not recommended** for running Lutris itself because PyGObject
lives outside the venv. If you must use a venv, symlink the system gi module:

```bash
ln -s /usr/lib/python3/dist-packages/gi* ~/venvs/lutris/lib/python*/site-packages/
```

However, the project's `.venv` already exists and should work with the setup
already in place.

### 3. Display / Wayland Issues

If Lutris crashes on launch with display errors:

```bash
# For Wayland sessions, Lutris sets WEBKIT_DISABLE_DMABUF_RENDERER=1
# If you still have issues, force XWayland:
env WAYLAND_DISPLAY= ./bin/lutris -d

# Or use X11 explicitly:
env DISPLAY=:0 ./bin/lutris -d
```


### 4. Debug Mode Randomness

The `-d` flag affects timing — the additional log output can mask or alter
race conditions. If a bug only reproduces without `-d`, try adding targeted
`logger.debug()` calls instead.

### 5. Subprocess Relationship

Lutris launches game processes as subprocesses. The Python debugger can't
step into a subprocess unless you:

- Attach VS Code to the child process PID (use the **Attach** configuration)
- Or use `PYTHONBREAKPOINT` / edit `lutris-wrapper` to insert `breakpoint()`

### 6. Missing Dependencies

The project's dependencies are listed in:

- **Debian/Ubuntu**: `debian/control`
- **RPM/Fedora**: `lutris.spec`
- **All platforms**: `Makefile` target `req-python` (`make req-python`)

Run `make dev` to install development tools (ruff, mypy, nose2).

---

## Quick Reference

```bash
# I just want to see what's happening
tail -f ~/.cache/lutris/lutris.log

# I want verbose terminal output
./bin/lutris -d

# I want to step through code
./bin/lutris -d   # then insert breakpoint() calls

# I want VS Code debugging
# → Open Run & Debug (Ctrl+Shift+D) → Select "Lutris (Debug Mode)" → F5

# I want to debug a segfault
gdb -ex r --args "/usr/bin/python3" "./bin/lutris"

# I want to run tests
make test

# I want to run linting
make sc
make check
```
