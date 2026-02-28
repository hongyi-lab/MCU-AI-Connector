# MCU-AI-Connector
This can help you connect your AI agent with API to hardware MCU.
# MCU_AI_Connector

A toolkit for automatically tuning PID parameters using Claude AI. Connects to an STM32 via UART serial, collects step response data, and lets the AI iteratively optimize Kp/Ki/Kd until the system converges.

```
PC (Python)                    UART                    MCU (STM32)
┌──────────────────┐   $PID/$STEP ──────────►   ┌─────────────────┐
│  tuner.py        │                             │  PID Controller  │
│  ┌────────────┐  │   ◄────────── $TEL Telem.  │  Plant           │
│  │ Claude API │  │                             └─────────────────┘
│  └────────────┘  │
└──────────────────┘
```

## Directory Structure

```
MCU_AI_Connector/
├── pc/
│   ├── main.py          # CLI entry point
│   ├── config.py        # All configuration options
│   ├── tuner.py         # Main loop orchestration
│   ├── analyzer.py      # Performance metric calculation + Claude API calls
│   ├── safety.py        # Parameter clipping and divergence detection
│   ├── serial_conn.py   # Real serial port + FakeSerial simulator
│   ├── protocol.py      # UART protocol encoding/decoding
│   └── tuning_log.json  # Tuning log (auto-generated at runtime)
└── mcu/
    ├── pid_uart_bridge.c  # STM32-side implementation
    └── pid_uart_bridge.h  # STM32-side header file
```

## Quick Start

### 1. Install Dependencies

```bash
pip install anthropic pyserial
```

### 2. Set API Key

```bash
# Windows
set ANTHROPIC_API_KEY=sk-ant-...

# Linux / macOS
export ANTHROPIC_API_KEY=sk-ant-...
```

### 3. Run

```bash
cd pc

# Pure software simulation, no hardware required
python main.py --simulate

# Connect to a real MCU
python main.py --port COM3

# Specify initial parameters and maximum iterations
python main.py --simulate --kp 2.0 --ki 0.05 --kd 0.5 --max-iter 10
```

## Tuning Workflow

Each iteration executes the following 4 steps:

```
[1/4] Send $STEP command, collect 3 seconds of telemetry data
[2/4] Calculate overshoot, rise time, steady-state error, oscillation count
[3/4] Send metrics to Claude, receive new Kp/Ki/Kd suggestions
[4/4] Safety clipping (bounds + divergence detection), apply new parameters
```

Automatically stops when all three criteria are met:

| Metric | Default Target |
|--------|----------------|
| Overshoot | ≤ 5% |
| Settling time | ≤ 1.0 s |
| Steady-state error | ≤ 1% |

## Main Configuration (`pc/config.py`)

| Option | Default | Description |
|--------|---------|-------------|
| `SERIAL_PORT` | `"COM3"` | Serial port |
| `SERIAL_BAUDRATE` | `115200` | Baud rate |
| `INITIAL_KP/KI/KD` | `1.0 / 0.01 / 0.1` | Initial PID parameters |
| `PID_BOUNDS` | Kp∈[0.05,20] Ki∈[0,5] Kd∈[0,10] | Parameter safety bounds |
| `MAX_CHANGE_RATIO` | `0.5` | Maximum adjustment per iteration (±50%) |
| `STEP_TARGET` | `100.0` | Step target value |
| `DATA_COLLECTION_SEC` | `3.0` | Data collection duration per iteration (seconds) |
| `MAX_ITERATIONS` | `20` | Maximum number of iterations |

## MCU Integration (STM32 + Keil)

Copy `mcu/pid_uart_bridge.c` and `pid_uart_bridge.h` into your Keil project, then:

**`main.c` — Initialization**

```c
#include "pid_uart_bridge.h"

// Call after HAL_Init()
PID_Bridge_Init(&huart1);  // Replace with your UART handle
```

**Interrupt Callback — Forward each byte to the parser**

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    PID_Bridge_RxCallback(rx_byte);
    HAL_UART_Receive_IT(huart, &rx_byte, 1);  // Re-arm interrupt
}
```

**PID Control Loop — Read parameters / send telemetry**

```c
// Read the latest PID parameters sent from PC
PID_Params_t p = PID_Bridge_GetParams();

// Check for a new step command
uint8_t has_new;
float target = PID_Bridge_GetStepTarget(&has_new);
if (has_new) setpoint = target;

// ... your PID calculation ...

// Report telemetry
PID_Bridge_SendTelemetry(HAL_GetTick(), setpoint, actual, error, output);
```

**CubeMX Configuration Requirements**
- UART: 115200 baud, 8N1, enable interrupt receive
- Modify the HAL header on line 19 of `pid_uart_bridge.h` to match your chip (default: `stm32f1xx_hal.h`)

## UART Protocol

All frames are ASCII text terminated with `\n`.

| Direction | Frame Format | Description |
|-----------|--------------|-------------|
| PC → MCU | `$PID,<kp>,<ki>,<kd>` | Set PID parameters |
| PC → MCU | `$STEP,<target>` | Send step command |
| PC → MCU | `$QUERY` | Query current parameters |
| MCU → PC | `$TEL,<tick_ms>,<setpoint>,<actual>,<error>,<pwm>` | Telemetry report |
| MCU → PC | `$PARAM,<kp>,<ki>,<kd>` | Parameter query reply |

## Safety Mechanisms

- **Per-iteration change limiting**: Each parameter adjustment is capped at ±50% of the current value (`MAX_CHANGE_RATIO`)
- **Absolute bounds clipping**: Parameters are always constrained within `PID_BOUNDS`
- **Divergence detection**: If the error in the second half of a collection window persistently exceeds 3× the setpoint, the system automatically rolls back to the previous parameter set

## Tuning Log

After each run, the complete history is saved to `pc/tuning_log.json`:

```json
{
  "final_params": {"kp": 0.95, "ki": 0.012, "kd": 0.21},
  "iterations": [
    {
      "iteration": 0,
      "status": "ok",
      "old_params": {"kp": 1.0, "ki": 0.01, "kd": 0.1},
      "metrics": {"overshoot_pct": 23.5, "rise_time_sec": 0.42, ...},
      "claude_suggestion": {"kp": 0.8, "ki": 0.012, "kd": 0.18, "reasoning": "..."},
      "applied_params": {"kp": 0.8, "ki": 0.012, "kd": 0.18},
      "safety_warnings": []
    }
  ]
}
```
