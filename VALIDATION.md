# Validation record

Checked on 2026-11-06. Browser, simulation, and mock-Bluetooth results are separate from physical robot validation.

- The app opens without activation; Python export is editable; Developer Studio opens for every visitor.
- The editor preview loaded with parent storage and Bluetooth isolated. Applying custom code still requires a stopped mission and disconnected robots.
- A complete local outbound and return simulation restored Robot A's start tile and heading.
- 240 rectangular/custom A* routes matched an independent turn-first Dijkstra implementation.
- Mock GATT covered distinct connections, duplicate-device rejection, touch cancellation, bump recovery, late replies, and CRC rejection.
- Pose checks covered four headings, imperfect reset poses, 16 outbound/return moves, and excessive-drift stops.
- Tile-reservation checks covered adjacent starts, swaps, shared tiles, yielding, parked-robot detours, and blocked states.

No real robot was paired or moved in this update. BLE stop latency, sensor timing, connection capacity, and Python import still need physical validation. Tile reservations do not model robot body overlap. Stock Root hardware uses a shared top light rather than independently addressable left/right lights.
