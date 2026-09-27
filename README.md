# agent-relay

Private relay used by the `agent` polling client (see the agent project's SPEC.md).
One-directional by design: this repository publishes instructions and artifacts,
the target machine pulls them, verifies them, executes them once, and logs the
result locally.

## Protocol (v1)

| path | role |
|---|---|
| `run.json` | the only command channel. The machine reads it each poll. |
| `artifacts/<name>` | binaries/scripts referenced by a command. |
| `STOP` | if this file exists, the machine executes nothing and logs POLL-STOPPED. |

`run.json` shape:

    {
      "seq": 1,
      "commands": [
        { "id": "step-1", "exe": "C:\\Windows\\System32\\cmd.exe",
          "args": ["/c", "echo hi"], "timeout": 60 }
      ]
    }

Rules the machine enforces:

* `seq` must be greater than the locally recorded seq; every command is executed
  at most once, and the state is written to disk before the next poll.
* An optional `"sha256"` per command is verified against the downloaded artifact
  before anything runs.
* A command whose `exe` points into `artifacts/` is downloaded through the
  GitHub Contents API, base64-decoded, size-checked and hash-checked first.
* Each command runs as a child process with a hard timeout; the poller itself
  must survive whatever the child does.

## Change log

* seq 1: connectivity self-test (cmd.exe echo) plus a 5-second timeout probe.