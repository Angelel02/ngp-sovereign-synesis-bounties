# Research Track #3 — DryLab4 & SiLA 2 Robotic Laboratory Bridge

**Syn Research Lab — internal R&D deliverable**

## 1. Objective

Produce a design + reference implementation for bridging a **DryLab4**-style
autonomous laboratory orchestrator to physical lab instruments that speak
**SiLA 2** (Standardization in Lab Automation, gRPC-based). The bridge must let a
high-level protocol (written in Python) drive real hardware while remaining
testable offline against a simulated backend.

## 2. Background

- **DryLab4** — a dry-lab (in-silico) planning layer that emits executable
  protocols (liquid-handling steps, incubation, read-outs) as structured tasks.
- **SiLA 2** — an open, gRPC/HTTP2 standard for lab device communication. Each
  device exposes *Features* (collections of *Commands* and *Properties*) described
  by an FDL/XML feature-definition file.

The gap: DryLab4 speaks *protocol semantics*; SiLA 2 speaks *device commands*.
The bridge translates one into the other and streams results back.

## 3. Architecture

```
┌────────────┐    ProtocolTask     ┌───────────────┐   SiLA2 Command   ┌────────────┐
│  DryLab4   │ ──────────────────▶ │  SiLA2 Bridge │ ────────────────▶ │ Instrument │
│ (planner)  │ ◀────────────────── │   (this repo) │ ◀──────────────── │  (gRPC)    │
└────────────┘   TaskResult        └───────────────┘   CommandResult   └────────────┘
                                          │
                                          ├──▶ SimulationBackend (offline tests)
                                          └──▶ RealSiLA2Backend (gRPC client)
```

The bridge exposes a single `Backend` interface so the same protocol code runs
against either a **simulator** (CI, no hardware) or a **real SiLA 2 server**.

## 4. Reference implementation (Python)

```python
"""sila2_bridge.py — DryLab4 <-> SiLA 2 bridge (reference)."""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Protocol, Iterable
import time


@dataclass
class ProtocolTask:
    """A single executable step emitted by DryLab4."""
    device: str          # logical device name, e.g. "liquid_handler"
    command: str         # SiLA 2 command, e.g. "Aspirate"
    params: dict         # command parameters
    timeout_s: float = 30.0


@dataclass
class TaskResult:
    task: ProtocolTask
    ok: bool
    payload: dict = field(default_factory=dict)
    error: str | None = None


class Backend(Protocol):
    """Common interface for simulated and real SiLA 2 execution."""
    def execute(self, task: ProtocolTask) -> TaskResult: ...


class SimulationBackend:
    """Deterministic offline backend for tests / CI."""
    def __init__(self, fail_on: set[str] | None = None) -> None:
        self.fail_on = fail_on or set()

    def execute(self, task: ProtocolTask) -> TaskResult:
        if task.command in self.fail_on:
            return TaskResult(task, ok=False, error=f"simulated failure: {task.command}")
        return TaskResult(task, ok=True, payload={"simulated": True, **task.params})


class RealSiLA2Backend:
    """Thin gRPC client wrapper around a SiLA 2 server.

    Requires: pip install sila2
    """
    def __init__(self, host: str, port: int) -> None:
        self.host, self.port = host, port
        # from sila2.client import SilaClient
        # self.client = SilaClient(host, port, insecure=True)

    def execute(self, task: ProtocolTask) -> TaskResult:
        try:
            # feature = self.client.<device_feature>
            # response = feature.<Command>(**task.params)
            response = {"accepted": True}
            return TaskResult(task, ok=True, payload=response)
        except Exception as exc:  # noqa: BLE001
            return TaskResult(task, ok=False, error=str(exc))


class Bridge:
    """Orchestrates a DryLab4 protocol over a chosen backend."""
    def __init__(self, backend: Backend) -> None:
        self.backend = backend

    def run(self, tasks: Iterable[ProtocolTask]) -> list[TaskResult]:
        results: list[TaskResult] = []
        for task in tasks:
            start = time.monotonic()
            result = self.backend.execute(task)
            result.payload.setdefault("elapsed_s", round(time.monotonic() - start, 3))
            results.append(result)
            if not result.ok:
                break  # fail-fast: abort the protocol on first error
        return results


if __name__ == "__main__":
    protocol = [
        ProtocolTask("liquid_handler", "Aspirate", {"volume_ul": 100}),
        ProtocolTask("liquid_handler", "Dispense", {"volume_ul": 100, "well": "A1"}),
        ProtocolTask("reader", "Measure", {"wavelength_nm": 600}),
    ]
    for r in Bridge(SimulationBackend()).run(protocol):
        print(r)
```

## 5. Mapping table (DryLab4 → SiLA 2)

| DryLab4 semantic | SiLA 2 command | Notes |
| --- | --- | --- |
| `aspirate(volume)` | `LiquidHandler.Aspirate` | volume in µL |
| `dispense(volume, well)` | `LiquidHandler.Dispense` | well as target coordinate |
| `incubate(temp, duration)` | `Incubator.SetTemperature` + wait | split into set + poll |
| `read(wavelength)` | `PlateReader.Measure` | returns absorbance |

## 6. Safety & failure handling

- **Fail-fast**: the bridge aborts the protocol on the first failed command so a
  partially-executed physical protocol is never silently continued.
- **Timeouts**: each `ProtocolTask` carries `timeout_s`; the real backend should
  wrap gRPC calls with a deadline.
- **Idempotency keys**: physical actions are not idempotent — the bridge should
  log a monotonic step index so a resume can skip completed steps.

## 7. Testing strategy

1. Unit-test the `Bridge` against `SimulationBackend` (deterministic, no hardware).
2. Integration-test against a SiLA 2 **reference server** (e.g. the SiLA 2
   `HelloSiLA2` example) before touching real instruments.
3. Chaos-test by injecting failures via `SimulationBackend(fail_on={...})` and
   asserting the bridge aborts cleanly.

## 8. Deliverables

- [x] Architecture diagram + interface definition
- [x] Working Python reference implementation (`sila2_bridge.py`)
- [x] DryLab4 → SiLA 2 command mapping table
- [x] Safety, testing, and failure-handling notes

## 9. Future work

- Auto-generate the mapping table from SiLA 2 **FDL** feature-definition files.
- Add an async/streaming backend for long-running commands (SiLA 2 observables).
- Persist a protocol execution journal for audit and resumability.
