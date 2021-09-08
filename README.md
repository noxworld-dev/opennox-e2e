# OpenNox end-to-end tests

To run the test, you must have OpenNox v1.8+ and English GoG game data folder.

```bash
NOX_E2E="$PATH_TO_E2E/e2e.yaml" opennox
```

It will run the client which will execute the E2E test script. Steps marked with `screen` will be compared to expected
frames from `testdata` folder. If there's a mismatch, the test will fail and emit `*_got.png`, so you can compare it.

## Writing scripts

Scripts must be written in YAML format, see [e2e.yaml](./e2e.yaml) for an example.

Conceptually, it consists of a sequence of actions that either add some input to the game, or read the results (usually a screenshot).
Each command can be executed instantly or delayed for a given number of frames.

Actions can be named, which helps to see what's currently being executed in the console. Names can be arbitrary,
except the names for `screen` steps which should include prefix which helps sort the test frames in the directory.

All the input expect closing the engine is ignored until the script execution completes.

Now, let's consider available script actions that can be used in steps.

### quit

Closes the engine and stops the test with successful exit status. If omitted, the test script will stop and will
let the user control game input again, which might be helpful for reproducing issues with some preconditions. 

**Example:**

```yaml
  - action: quit
```

### wait

Waits for a given number of frames. Most of the actions have an integrated `dt` parameter that can be used instead.
But sometimes this may come handy, for example for commands that use `dt` for other purposes.

**Example:**

```yaml
  - name: "Wait for save"
    action: wait
    dt: 20
```

### screen

Takes screenshot and checks it against expected frames from `testdata` directory.
If there's no frame, it will write the image automatically. If the frame doesn't match, the test will fails and file
with `*_got.png` suffix will be generated for comparison.

**Example:**

```yaml
  - name: "menu-000 Main menu"
    action: screen
  - name: "menu-001 Main after one frame"
    action: screen
    dt: 1
```

This will check or create a screenshot named `menu-000_main_menu.png`, wait for one frame and take/check another
screenshot named `menu-001_main_after_one_frame.png`. As you may notice, `dt` is optional and sets a delay for this action.

By convention, the screenshot name must contain a test ID (`menu`), followed by a test frame number (`001`), followed
by a short description of that frame.

### slow

Intentionally delays the gameloop so that it's easier to debug steps. Can be placed between any steps, making it easy
to skip good part of the test and focus on a buggy one. The delay between game loop frames can be controlled with `dur`.

Setting non-zero `dur` sets the delay, setting it to `0ms` removes it again.

**Example:**

```yaml
  - name: "DEBUG START"
    action: slow
    dur: 300ms
  - name: "menu-000 Main menu"
    action: screen
  - name: "DEBUG STOP"
    action: slow
    dur: 0ms
```

### click

Moves cursor to a given point and clicks with a left mouse button. It does this instantly, unless delayed by `dt`.

Game window is assumed to always be `1024x768`, X coordinate goes right, Y coordinate goes down the screen.

To know which coordinate to use, run the engine in E2E mode, click st the right moment, and you will see logs
of your click event

**Example:**

```yaml
  - name: "Open options"
    action: click
    x: 368
    y: 731
```

If you use this in-game and notice that the click doesn't work, use [`interact`](#interact) instead.

### interact

Similar to [`click`](#click), but delays click after moving the mouse a bit for the game to detect hover event.
This works best for in-game objects, where `click` might be too fast. Still `click` is preferred for the menu, since
it doesn't waste frames by waiting for a hover.

**Example:**

```yaml
  - name: "Pickup apple"
    action: interact
    x: 534
    y: 351
```

### melee

Make character hit with a current melee weapon in a given direction.
Direction is given as a floating point value where `0.0` is up the screen, `0.5` - right, `1.0` - down, `1.5` - left (or `-0.5`).

**Example:**

```yaml
  - name: "Hit wall"
    action: melee
    ang: 0.25
```

### cast

Make character cast spell or ability in a quick slot. The slot number is given by `slot` parameter.

**Example:**

```yaml
  - name: "Cast pixies"
    action: cast
    slot: 1
```

### walk

Similar to [`run`](#run). Makes the character walk in a given direction for a certain amount of time.
Direction is given as a floating point value where `0.0` is up the screen, `0.5` - right, `1.0` - down, `1.5` - left (or `-0.5`).

In this command, the `dt` parameter specifies the amount of time that the character will walk, _not_ the delay before.

After walking for a given duration, the character will slow down and stop before executing the next command.
To remove this delay consider [`walk-start`](#walk-start).

**Example:**

```yaml
  - name: "Walk into the elevator"
    action: walk
    ang: 0.70
    dt: 20
```

### run

Similar to [`walk`](#walk). Makes the character run in a given direction for a certain amount of time.
Direction is given as a floating point value where `0.0` is up the screen, `0.5` - right, `1.0` - down, `1.5` - left (or `-0.5`).

In this command, the `dt` parameter specifies the amount of time that the character will run, _not_ the delay before.

After running for a given duration, the character will slow down and stop before executing the next command.
To remove this delay consider [`run-start`](#run-start).

**Example:**

```yaml
  - name: "Run into the bear cave"
    action: run
    ang: 0.65
    dt: 50
```

### walk-start

Similar to [`walk`](#walk), but the command is split in three parts, allowing change direction while walking.
The `walk-start` will make the character walk in a given direction, then `walk-dir` allows to change direction,
and finally `walk-stop` will make the character stop.

It is possible to start walking with `walk-start` and then switch to running with [`run-dir`](#run-start),
but not the other way around.

**Example:**

```yaml
  - name: "Walk out of the bear cave"
    action: walk-start
    ang: -0.35
  - name: "Walk to the elevator"
    action: walk-dir
    ang: -0.20
    dt: 50
  - action: walk-dir
    ang: -0.30
    dt: 60
  - action: walk-stop
    dt: 17
```

### run-start

Similar to [`run`](#run), but the command is split in three parts, allowing change direction while running.
The `run-start` will make the character run in a given direction, then `run-dir` allows to change direction,
and finally `run-stop` will make the character stop.

**Example:**

```yaml
  - name: "Run out of the bear cave"
    action: run-start
    ang: -0.35
  - name: "Run to the elevator"
    action: run-dir
    ang: -0.20
    dt: 50
  - action: run-dir
    ang: -0.30
    dt: 60
  - action: run-stop
    dt: 17
```

### inventory

Toggles inventory on or off. There might be some delay before it fully opens or closes.

**Example:**

```yaml
  - name: "Open inventory"
    action: inventory
  - name: "con01-018 Inventory"
    action: screen
    dt: 5
  - name: "Close inventory"
    action: inventory
```

### esc

Simulates Esc key, which might be useful to close current UI elements.

**Example:**

```yaml
  - name: "Close book"
    action: esc
```