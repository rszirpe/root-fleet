# Root Fleet: custom maps, touch stop, and dynamic robots

`index.html` is the complete front-end app. CSS, JavaScript, the simulator and Web Bluetooth are embedded.
It needs no packages, external scripts, MCP integration or application backend.

## Use it

1. Open [Root Fleet on GitHub Pages](https://rszirpe.github.io/root-fleet/) in a compatible browser. No account or code is needed.
2. Start with Robot A, or use **Load crossing demo** for A and B. Use Chrome or Edge for physical Bluetooth.
3. Press **Add Robot** at the bottom to create B, C, and beyond. No Bluetooth chooser opens.
4. Select a robot's mode. Tap its start, tap that start again to rotate, then tap another tile for its target.
5. Choose **Wall** or **Hazard** and drag to paint. Choose **Floor** to erase.
   Use **Robot Start** / **Robot Target** to place endpoints explicitly for the selected robot.
6. Press **Simulate Fleet** without connecting hardware.
7. Later, connect each placed robot using its card's **Connect Robot [Letter]** button, then press **GO - Run Fleet**.

Everyone can edit generated Python and use Developer Studio to customize a local copy of the app.

Place real robots at their marked tile centers, facing their arrows. Put bench robots off the physical board.
The app rejects assigning one Bluetooth device to multiple cards.

**Virtual fleet size has no fixed application cap.** IDs continue A through Z, AA, AB, and onward.
The available map tiles, browser resources and Bluetooth adapter impose practical limits.
This is not a promise that infinitely many physical robots fit or connect simultaneously.

Unplaced cards stay on the virtual bench and do not block a configured mission or require a GO connection.
A robot with a start but no target is parked: its current tile is a shared map obstacle.
It participates in physical connection checks.
Use **Move to bench** or **Remove** to take a robot out of the deployed fleet. Remove its real robot from the board too.
IDs are not reused after removal. Robot A's card remains available as the priority identity.

## Floor plans and scale

1. Set rows/columns and tile width/height, then **Apply dimensions**.
2. Use **Upload floor plan** for PNG, JPG, or SVG. The grid covers the full image.
3. For an image with a known measurement, select **Calibrate from image**, tap the two measurement endpoints,
   enter their real distance in centimeters, and apply. The preview shows the resulting physical map size.
4. Trace image walls with **Wall**. **Hazard** marks prohibited tiles; **Floor** erases either kind.
   Both walls and hazards are excluded from every robot’s path. Mission endpoints cannot be painted over.
5. **Save map file** exports the image, dimensions, and obstacle tiles as JSON. **Open map file** restores them.
   Robot missions are configured separately. Copy JSON is available when a browser does not support downloads.
   Hazard types are saved; older map files without hazards still open.

Images are backgrounds, not automatic wall detectors. Saved map files do restore explicit obstacle cells.
Uploads stay in the browser; SVG content is sanitized and rasterized before display/storage.
Images must be under 10 MB and 32 million pixels; stored backgrounds are downsampled to a maximum side of 2048 pixels.
No floor plan, image, or robot state is sent to a server. The application has no external script/CDN dependency.

```text
Image points p1,p2 + measured length L
              |
       cm/pixel = L / pixel span
              |
       full image width/height in cm
              |
       divide by grid columns/rows -> tile width/height
```

Calibration uses image pixels, so resizing the browser does not change physical distances.
Use an undistorted top-down plan. Two-point scale cannot correct a perspective photograph.
**Upload reference image** works separately: select the ends of a measured span, enter its physical length
and the number of tiles in that span. It sets square tiles to `length / tile count`, preserving the background.
Manual width/height inputs remain available for rectangular tiles.
Changing dimensions preserves valid endpoints and removes endpoints/obstacles outside the new map.
Map controls are locked while a mission runs or an emergency remains latched.

## Capacitive touch emergency pause

Top touch events arrive on the existing UART TX notification stream, with device 17 / command 0.
Device 17 is explicitly enabled in the 128-bit event mask. Packet byte 7 uses its upper nibble:
front-left `0x80`, front-right `0x40`, rear-right `0x20`, rear-left `0x10`.
Corrupt CRC packets are ignored. Release updates the cached touch state without resuming anything.

```text
Any top pad pressed during a run
              |
     abort ALL active commands and pending planning
              |
     zero wheel speeds -> cancel firmware actions -> flashing red
              |
     EMERGENCY PAUSE (no movement, no bumper rollback)
              |
     release pads + restore marked starts/headings + manual reset
              |
     idle; a NEW Simulate/GO action is required
```

The fleet stops together because a partial move invalidates traffic reservations.
The UI freezes at the interrupted position; it does not claim the current tile was reached.
**Reset touch stop - starts restored** acknowledges manual repositioning, turns off the red warning,
and clears the latch. It refuses reset if a connected robot still reports a pressed pad.
The latch persists through a page reload in that tab; maps/missions should be saved separately.
Test this flow without hardware using **Test touch [Letter]** during simulation.

BLE stops are sent as soon as event handling runs and the current short GATT write completes.
This is a software stop, not a certified or latency-guaranteed emergency-stop circuit.
A disconnected robot may not receive it. Physical capacitive-stop behavior remains untested.

## Auto-Wait architecture

```text
Virtual registry: A, B, C, ... Z, AA, AB, ...
        |
        +-- unplaced cards -> bench, no execution
        |
        +-- placed robots -> shared obstacle map
                                  |
                         A* + traffic reservations
                                  |
             +--------------------+--------------------+
             |                    |                    |
         Clear route         B/C interference     Map blocked
         Red motion          Orange Auto-Wait     Yellow blinking
             |                    |                    |
         Per-robot BLE queue      +-- stay at exact current pose
             |
         All commands finish -> next reserved step

Any bumper -> halt the fleet -> measured pose recovery -> shared map update -> cascade replan
```

A reserves first. Subsequent robots follow creation order. Every route is checked against every other robot.
Higher-priority robots have planned schedules; lower-priority robots conservatively reserve their current positions.
That means a lower robot can always fall back to stationary Auto-Wait without invalidating a higher route.

Lower-priority robots without an available traffic route wait at their exact current tile with orange lights.
Scheduled waits resume automatically as higher-priority robots clear their reservations.
Robot A never receives a traffic-yield command or orange traffic-wait lights.

**Parked and finished robots are static map obstacles at their actual live tile.** A runs A* around them.
If no alternate path exists, A is blocked with flashing yellow lights; it never waits for a finished robot to move.
The occupancy layer is derived separately from drawn/bump obstacles. Benching or restarting a robot cannot leave
an old robot tile behind, and removing occupancy cannot erase a permanent obstacle.

Target arrival starts a fresh return search. Final arrival adds that robot's current tile to the shared map.
Both events, and every bump recovery, trigger a fleet replan before the next movement is dispatched.
A completed round trip normally parks at the original start; the map always uses live pose, not that assumption.

If all remaining routes are blocked or waiting, the controller stays idle without repeated BLE writes or busy polling.
Use **Stop all**, change the layout and restart to resolve permanent occupancy. **Recheck waiting routes** repeats
planning with the current known map; it does not guess that a robot has moved.

Obstacle-map blockage is distinct from traffic interference: a map-blocked robot flashes yellow and stays put.
Other safe missions may continue. An idle controller can resume when a map update calls `wake()`.
Normal UI map editing is disabled during a run; stop before editing.

## Collision model and planning

Default: 4 x 4, 16 cm tiles. Custom maps support 1-32 rows/columns and 1-200 cm tile widths/heights.
Dimensions round to 0.1 cm (1 mm). Origin is bottom left; headings are N=0, E=90, S=180, W=270.
Relative clockwise rotation is `(desired - current + 360) % 360`; North to East is right 90.

Single-robot A* searches `(x, y, heading)`. Actions are one calibrated tile, left 90 and right 90.
North/south moves use tile height; east/west moves use tile width. Fewer turns wins, then physical distance.
The turn weight exceeds the distance of any simple path in the heading-state graph. At the default scale,
the cost reduces to `64 * quarter_turns + tile_steps`.
The admissible heuristic ignores obstacles and uses the better unobstructed axis order.

Space-time A* adds mission leg and reservation time. Cost is the lexicographic tuple `[turns, normalized physical
distance, waits]`.
Time saturates after higher schedules finish, leaving a finite search graph. Failure to find a traffic schedule
produces Auto-Wait for lower-priority robots or a yellow blocked state for A. A fresh return search includes
the exact initial heading as the final goal.
The planner minimizes turns; it does not claim calibrated execution-time optimality in seconds.
It is a conservative prioritized planner, not a complete solver for every possible joint maneuver.

The collision model uses tile ownership only. Adjacent starts and adjacent movement are allowed.
There is no minimum center-distance test and no Euclidean calculation between robots.

Every step reserves its source AND destination tile until all dispatched commands finish. Two robots cannot
enter the same tile, swap tiles, or enter a tile another robot has not yet confirmed leaving. Independent
Bluetooth delays therefore cannot release an occupied tile early. Parallel movement on adjacent tiles is allowed.
Unresolved interference returns a stationary state rather than a priority-schedule exception.

```text
A: (0,0) -> (0,1)     B: (1,0) -> (1,1)     Allowed: distinct endpoint tiles
A: (0,0) -> (1,0)     B: (1,0) -> (0,0)     Blocked: tile swap
A: (0,0) -> (0,1)     B: parked at (0,1)     A replans around B
```

This discrete model does not represent physical robot bodies. Own-robot encoder checks remain for validating
movement and bump recovery; they do not measure separation between robots. Physical fleet motion is untested.

## Pose checks and the two-tile stop fix

The previous checker accepted a reset heading within 5 degrees of 90, but then assumed exactly 90 degrees
for every absolute pose. A simulated 88-degree reset passed the first 16 cm step and failed the second,
even with perfect relative movement. It also imposed a 1 cm accumulated-error limit while accepting
up to 2 cm of error within one individual move. These inconsistent checks could halt an otherwise valid route.

The controller now records each robot's measured, settled starting pose and transforms grid coordinates
into that fixed frame. It never resets/rebases the reference after individual moves.
Absolute error is checked separately along the board's X/Y axes, within a quarter of that axis's tile size,
capped at 5 cm. The heading bound is 10 degrees. Default 16 cm tiles therefore allow 4 cm per axis.
These are bounded engineering tolerances, not a claim of measured physical accuracy or clearance.
Per-command odometry, bumper, touch, timeout, and connection checks remain active.

```text
Measured reset pose + configured start heading
                    |
         fixed grid-to-encoder transform
                    |
         await each movement-finished reply
                    |
     per-step check + bounded absolute tile check
                    |
           commit tile -> next command
```

There is no two-step limit. Target arrival triggers the independent return search, then every remaining
command runs until home, Stop, a sensor interruption, or a genuine error.
A pose failure now reports measured X/Y/heading errors and their limits.
**Movement diagnostics** retains the last 200 checks, measured origins, expected/reported poses, and outcome.
Use **Copy diagnostics** if a physical run still stops. This data is local and contains no Bluetooth device IDs.

Regression tests reproduce the old two-tile failure and complete 16 movement commands for each of four
initial headings using real transport code with mock GATT, rectangular tiles, reset offsets, and
6 mm of per-move distance variation. Excessive drift still halts before committing the unconfirmed tile.
Physical hardware has not been validated by these tests.

## Independent Bluetooth objects

Each card owns a `RootBLE` object with its own device, GATT server, RX writer, TX subscription, packet sequence,
pending-reply map and write queue. The controller owns shared traffic and obstacle state.
Adding a virtual card does not request or acquire a Bluetooth connection.
Browser A* searches run inside a local Blob Worker, keeping notification handling and Stop responsive.
Editing a map cancels stale preview work; emergency stop aborts an outstanding run search.

- Identifier: `48c5d828-ac2a-442d-97a3-0c9822b04979`
- UART: `6e400001-b5a3-f393-e0a9-e50e24dcca9e`
- RX, host writes: `6e400002-b5a3-f393-e0a9-e50e24dcca9e`
- TX, robot notifications: `6e400003-b5a3-f393-e0a9-e50e24dcca9e`

Notifications start before commands. Bumpers use UART events: device 12, command 0, absolute byte 7,
left bit `0x80`, right bit `0x40`. Held presses are debounced until release.

Packets are 20 bytes: device, command, sequence ID, 16 payload bytes, CRC-8/ATM (`0x07`, initial zero).
Multibyte values are big-endian. Response matching uses device, command and sequence ID.

- Move: 1/8, signed int32 millimeters; distance is calibrated tile centimeters times 10.
- Turn: 1/12, signed int32 decidegrees; right 90 is 900, left 90 is -900.
- Halt: 1/4, two zero int32 wheel speeds; this preserves odometry during recovery.
- Position reset/read: 1/15 and 1/16.
- LEDs: 3/2, `[animation, R, G, B]`, then zero padding.
- Note: 5/0, uint32 Hz and uint16 milliseconds.

Only short writes are queued; waiting for motor completion does not block a stop packet.
A delivery acknowledgment alone never advances a robot's grid pose.

## LEDs

- Drive: solid red `#FF0000`, payload `[1,255,0,0]`.
- Turn: red, turning side blinks in the simulator; opposite side stays solid.
- Auto-Wait: solid orange `#FFA500`, payload `[1,255,165,0]`, after motors halt.
- Touch stop: flashing red `#FF0000`, payload `[2,255,0,0]`, latched until reset.
- Blocked/error: flashing yellow `#FFFF00`, payload `[2,255,255,0]`.
- Completion: yellow, orange, green `#00FF00`; each on 150 ms, off 50 ms, then a short chime.
- Completion is shown at target arrival and final home arrival, then lights turn off.

**The documented stock Root API controls the entire top LED cross together.** It exposes no independent side mask.
Directional turn signals are exact in simulation. Hardware turns use shared red pulses, nominally 4 Hz.
No invented left/right BLE fields or unsupported SDK calls are sent. Wireless and timer latency affect pulse timing.

## Reactive recovery

A forward bump aborts all in-flight operations, requests all motor halts and marks the attempted tile as blocked.
Stable native odometry measures partial movement. Interrupted moves and peer turns roll back to confirmed poses.
Every source/destination tile reservation remains held throughout recovery. All remaining routes then replan together.
There is no fixed bump-retry cap; map and physical constraints determine whether progress is possible.

A bump outside a forward step, contact during recovery, unreliable odometry, persistent bumper press, stall,
cliff, connection fault or command timeout stops the fleet. Physical execution requires usable native odometry.
After a physical stop, reposition the deployed robots and confirm their start positions before GO.
Simulation does not clear that physical reposition requirement.

Keep the page visible on a flat supported surface. Hiding it stops a run. A disconnected robot may not receive
its stop request; the connected peers are stopped when the fault is reported. Physical fleet motion is untested.

## Python fallback

The drawer exports a static map snapshot for any configured robot. Run exports one robot at a time with other
robots removed. They contain no fleet coordination, live bump recovery, or capacitive touch-stop listener. Level 3 event
code and full Python
SDK/Playground scripts use supported shared-light APIs. Raw `.py` import into the Coding App remains unverified.

## Validation

- 11,128 heading-aware A* cases checked against independent Dijkstra results.
- 160 layouts with 1-5 robots; 10,073 source/destination reservation checks, including adjacent starts.
- Parked/finished robots at live tiles, immediate completion replans, no ghost occupancy and A never yielding checked.
- Same-tile arrivals, swaps, no-write idle, wake, Stop and stable IDs beyond Z checked.
- Three-robot missions restore every original pose; lower-priority yielding and completion colors checked.
- A C bumper stops all peers and updates the shared map; all three complete after the cascade replan.
- Three independent mock GATT servers complete a fleet with unequal reply delays and duplicate-device rejection.
- Browser checks cover a parked-B detour, adjacent routes, Add Robot controls and independent connection cards.
- 240 custom/rectangular A* routes checked against independent turn-first Dijkstra.
- Calibration math, axis-specific BLE distances, Python export, and scaled bump recovery checked.
- Mock GATT covers all four pads, touch event mask, CRC rejection, red flash packets and cancellation.
- Browser checks cover SVG upload, two-point calibration, map JSON round trip, and latched touch reset.
- Physical fleet motion and touch response have not been tested by these checks.

## Primary references

- [Root BLE protocol](https://github.com/PoweredBySAM/sam-root-ble-protocol)
- [Root hardware documentation](https://github.com/PoweredBySAM/RootDocs)
- [Touch notification implementation](https://github.com/iRobotEducation/irobot-edu-python-sdk/blob/main/irobot_edu_sdk/robot.py)
- [Official Python SDK](https://github.com/iRobotEducation/irobot-edu-python-sdk)
