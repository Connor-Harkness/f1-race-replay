# Live Mode Work Plan (OpenF1 via MQTT)

Goal: render a session in real time using data pushed by the OpenF1 MQTT broker, reusing the existing replay UI and telemetry stream.

## Background
- Replay today: `src/f1_data.py` builds a full list of `frames` from FastF1 up front; `src/interfaces/race_replay.py` (`F1RaceReplayWindow`) plays them back by time index.
- Live data is incremental and unordered across topics, so the window needs a growing frame buffer rather than a precomputed one.
- `src/services/stream.py` already rebroadcasts state to secondary windows; it should work unchanged if frames keep the same shape.

## Phases

### 0. Discovery (spike)
- [ ] Confirm OpenF1 MQTT connection details (host, port, TLS, auth/token, topic names) from the official docs.
- [ ] Capture a recording of messages from a live/test session to JSONL for fixtures.
- [ ] Map OpenF1 topics (e.g. `v1/location`, `v1/car_data`, `v1/position`, `v1/laps`, `v1/intervals`, `v1/stints`, `v1/pit`, `v1/race_control`, `v1/weather`, `v1/sessions`, `v1/drivers`) to the fields in the current frame schema (see `README.md` / `telemetry.md`). List gaps (e.g. tyre info, track status, safety car position).

### 1. Dependencies and config
- [ ] Add an MQTT client (`paho-mqtt`) to `requirements.txt`; run the advisory check first.
- [ ] Add settings (broker URL, credentials via env var, never committed) in `src/lib/settings.py`.

### 2. MQTT ingest layer (`src/services/openf1_mqtt.py`)
- [ ] Client wrapper: connect, reconnect with backoff, subscribe, thread-safe queue of raw messages.
- [ ] Message parsing/validation per topic into typed dicts; tolerate missing/late fields.
- [ ] Record/replay mode: write raw messages to JSONL and read them back, so everything is testable offline.

### 3. Live state aggregator (`src/live/`)
- [ ] Maintain per-driver latest state (position x/y, speed, gear, DRS, lap, tyre, status, race position).
- [ ] Resample on a fixed tick (e.g. 10-25 Hz) into frames matching the replay frame schema.
- [ ] Rolling buffer of frames with bounded memory; session metadata (drivers, colours, circuit) fetched from OpenF1 REST at start.
- [ ] Derive track outline from early location data (or FastF1 circuit fallback) since no full-lap reference exists at start.
- [ ] Handle race control, track status and weather updates.

### 4. UI integration
- [ ] Refactor `F1RaceReplayWindow` to accept a frame source interface (static list vs live buffer).
- [ ] Live mode behaviour: follow "now", small display delay buffer, "LIVE" indicator, disable/adapt seek and progress bar, allow scrubbing back within buffer.
- [ ] Reconnect/stale-data indicators.

### 5. CLI entry point
- [ ] Add `--live` flag in `main.py` (plus `--live-file` to play a recorded JSONL), selecting current/most recent session.

### 6. Testing and docs
- [ ] Unit tests (pytest, see `docs/Testing.md`) for parsers, aggregator, frame resampling using recorded fixtures.
- [ ] Integration test with a mock MQTT broker or the JSONL replayer.
- [ ] Document usage in README and `telemetry.md`.

## Risks / open questions
- OpenF1 MQTT may require a paid/authenticated account during live sessions.
- Location data rate and clock sync across topics (use message timestamps, not arrival time).
- Tyre/compound and gap data availability and latency.
- Missing safety car GPS; existing simulation approach can be reused.

## Suggested first milestone
Recorded-JSONL source -> aggregator -> frames -> existing window rendering cars on track, then swap in the real MQTT client.
