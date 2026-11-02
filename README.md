# ops-report

Error reporting and heartbeats for a Python service. A service installs it once at startup; from then
on its uncaught exceptions, `logging` records at ERROR and above, and the beats it sends go to an ops
responder over HTTP. Standard library only.

## Install

Pin a tag by its archive URL, which needs neither git nor credentials at install time. In
`requirements.txt`:

    ops-report @ https://github.com/JJGantt/ops-report/archive/refs/tags/v1.0.0.tar.gz

Moving a service to a new version is changing that tag and reinstalling (rebuilding its venv or its
image).

## Use

```python
import ops_report as ops

ops.install(machine="pi", service="voice-worker")  # first thing at startup
ops.report(exc_or_message, context={...})           # an error caught and handled
ops.beat(60)                                         # from the loop that does the work, every <= 60 s
```

- `install(machine=, service=, spool=)` hooks `sys.excepthook`, `threading.excepthook` and the root
  logger at ERROR. A `KeyboardInterrupt` reaching the top is treated as a shutdown, not a crash.
- `report(err, context=None, stack=None)` sends a caught failure. The same first line repeated within
  30 s is sent once, with a count.
- `beat(every_s=60, info=None)` says the service is alive and how often to expect it; the responder
  treats a missing beat as the failure.
- `flush(timeout=5.0)` waits for queued sends; it also runs at exit.

Sending happens on one background thread, so a caller never waits on the network. An error that cannot
be sent is appended to a spool file and goes out with the next successful send; a missed beat is not
kept.

A service that names its machine `fly` but runs where `FLY_APP_NAME` is unset (the same code imported
somewhere else) sends nothing, so a local script cannot report as the live service.

## Configuration

| variable | meaning |
|---|---|
| `OPS_URLS` | comma-separated responder base URLs; the first that answers wins. Required. |
| `OPS_TOKEN` | bearer token the responder accepts. Required. |
| `OPS_MACHINE`, `OPS_SERVICE` | defaults for `install()`'s `machine` and `service` |
| `OPS_SPOOL` | file for errors that could not be sent (default: the temp dir) |

`OPS_URLS` and `OPS_TOKEN` are read from the environment first and then from `~/.config/ops/env`
(`KEY=value` lines), so one file configures every service on a host. Without both, `install()` prints
one line to stderr saying it is not reporting.

The responder receives `POST <url>/error` and `POST <url>/beat`, each a JSON array of items
(`machine`, `service`, `message`, `stack`, `context`, `at` for an error; `machine`, `service`,
`every_s`, `info` for a beat).
