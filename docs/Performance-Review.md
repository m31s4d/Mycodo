# Performance Review Notes

## Daemon Recurring Workloads
- The main daemon loop runs every second (`time.sleep(1)`) and performs checks for timelapses, output-usage reports, anonymous stats, and upgrade checks, each protected by their own timers or schedule calculations.【F:mycodo/mycodo_daemon.py†L156-L190】
- Timelapse maintenance queries all cameras on every loop iteration; if a timelapse is active and due, it advances `timelapse_next_capture`, commits, and spawns a worker thread to capture images.【F:mycodo/mycodo_daemon.py†L1010-L1042】
- Timed statistics and upgrade checks rely on configurable intervals (`STATS_INTERVAL`, `UPGRADE_CHECK_INTERVAL`), with the next trigger timestamp advanced in `while now > timer` loops to avoid drift.【F:mycodo/mycodo_daemon.py†L175-L185】

## Sensor Read/Controller Loops
- Each input controller tracks `next_measurement` and `period`; when the current time exceeds the scheduled window the controller sets `get_new_measurement` and advances `next_measurement` in a `while` loop to catch up after delays.【F:mycodo/controllers/controller_input.py†L138-L152】
- Measurements may activate a pre-output, then call `update_measure()` before queuing actions and writing results to InfluxDB; success resets the measurement flag.【F:mycodo/controllers/controller_input.py†L188-L237】
- Controller configuration (including `period` and `start_offset`) is loaded from the database during initialization along with global controller sample-rate settings.【F:mycodo/controllers/controller_input.py†L246-L266】

## Blocking HTTP Endpoints (mycodo_flask)
- Login keypad routes use `time.sleep(2)` when no code is provided or on failed codes, blocking the request thread to slow brute-force attempts.【F:mycodo/mycodo_flask/routes_authentication.py†L309-L363】
- The camera management page stops an active stream before capturing a still, adding a `time.sleep(2)` delay before calling `camera_record`, which stalls the request until completion.【F:mycodo/mycodo_flask/routes_page.py†L173-L199】
- Forcing input measurements from the UI triggers a synchronous RPC (`input_force_measurements`) against the daemon; errors bubble back to the request context, indicating the request waits for the daemon response.【F:mycodo/mycodo_flask/utils/utils_input.py†L846-L866】

## Metrics and Logging Hooks
- The daemon logs lifecycle events and errors through the `mycodo` logger and records statistics at configurable intervals, with support for regenerating missing stats CSV files inside `send_stats`.【F:mycodo/mycodo_daemon.py†L120-L190】【F:mycodo/mycodo_daemon.py†L1064-L1074】

## Caching/Batching Candidates
- Timelapse checks query all `Camera` rows every second; caching only active timelapse IDs or event-driven scheduling could reduce database churn.【F:mycodo/mycodo_daemon.py†L1010-L1042】
- Input controllers read configuration from the database at startup; warm caches for device metadata or precomputed measurement dictionaries could shorten controller spin-up and reduce repeated DB lookups across many inputs.【F:mycodo/controllers/controller_input.py†L246-L266】
- UI flows that sleep (`routes_authentication`, `routes_page`) block the WSGI worker during delays; moving cooldowns to async tasks or client-side timers would keep request threads free.【F:mycodo/mycodo_flask/routes_authentication.py†L309-L363】【F:mycodo/mycodo_flask/routes_page.py†L173-L199】
