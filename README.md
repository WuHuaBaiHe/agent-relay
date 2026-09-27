# agent-relay

Private relay for the `agent` polling client (the agent project's SPEC.md is the
design record). One-directional by design: this repository publishes
instructions, the target machine pulls them, verifies them, executes each one at
most once, and records the outcome locally -- optionally publishing it back here.

## Layout

| path | role |
|---|---|
| `run.json` | the only command channel; read on every poll |
| `artifacts/<name>` | binaries referenced by a command whose `exe` is `artifacts/...` |
| `STOP` | if this *file* exists, nothing is executed and `POLL-STOPPED` is logged |
| `results/<machine>.json` | last run outcome per machine, written by `--post-result` |

## run.json

    {
      "seq": 4,
      "commands": [
        { "id": "collect-1",
          "exe": "artifacts/collector.exe",
          "args": ["--out", "report.json"],
          "timeout": 300 },
        { "id": "local-tool",
          "exe": "C:\\Windows\\System32\\cmd.exe",
          "args": ["/c", "echo hi"],
          "timeout": 30 }
      ]
    }

Enforced by the client:

* **`seq` must increase.** It is persisted after a command completes, so a
  command runs at most once. A crash mid-command leaves an
  `interrupted_seq` marker that the next poll reports instead of silently
  repeating or forgetting.
* **Bytes are proven before they run.** The Contents API envelope is
  base64-decoded, then size and git blob id (SHA-1 over `blob <len>\0<content>`)
  are checked locally against what the API reported. A mismatch refuses the run.
* **`exe` must be absolute or `artifacts/...`.** A bare name would be resolved
  through PATH, which is the ambient dependency the tool exists to avoid. An
  `artifacts/...` reference is downloaded and verified first, then run from the
  local work directory.
* **Every command runs as a child process with a hard timeout.** The poller
  survives whatever the child does; a killed child is reported as
  `timed_out=yes` with the process exit code.
* **A non-zero exit from a command is data, not a failure.** The poller exits 0
  whenever the cycle itself succeeded (fetched, verified, ran once); non-zero
  means the poll could not do its job at all.

## Exit codes

| code | meaning |
|---|---|
| 0 | cycle succeeded (or there was nothing to do: `POLL-NO-CHANGE` / `POLL-SKIP` / `POLL-STOPPED`) |
| 1 | cycle failed: network, parse, or verification |
| 2 | configuration refused: bad `--repo`, unreadable token, missing curl |

## Change log

* seq 3: idle. The command channel is empty so a poll cannot execute anything.
* seq 2: result-post check (verified the `results/` write path).
* seq 1: connectivity self-test (echo) plus a 5-second timeout probe that proved
  the hard kill.