# Control Algorithm System Design Document

## 1. System Overview

A real-time control algorithm that interfaces with an experimental device operating at 1Hz frequency. The system collects device data, performs optimization calculations, and sends control parameters back to the device in a closed-loop fashion (similar to Reinforcement Learning agent loop).

## 2. Core Requirements

- **Real-time operation**: Respond within 1 second cycle time
- **Robustness**: Handle errors gracefully without stopping the experiment
- **Modularity**: Separate concerns for maintainability
- **Logging**: Track all operations for debugging and analysis
- **Configurability**: Easy to adjust thresholds and parameters without code changes

## 3. Architecture Components

### 3.1 Project Structure

```
TransientFighterFCVer/
├── config/
│   └── settings.yaml              # All configurable parameters
├── src/
│   ├── __init__.py
│   ├── device_interface.py        # Part 1: Device API wrapper (existing)
│   ├── optimizer.py               # Part 2: Core optimization logic
│   ├── error_handler.py           # Part 3: Exception handling logic
│   ├── decorators.py              # Reusable decorators for robustness
│   ├── controller.py              # Main orchestrator
│   └── utils.py                   # Logging, validators, helpers
├── logs/                          # Runtime logs
├── tests/
│   ├── test_optimizer.py
│   ├── test_error_handler.py
│   └── test_controller.py
├── main.py                        # Entry point
├── DESIGN.md                      # This file
└── README.md                      # User guide
```

### 3.2 Module Responsibilities

#### **device_interface.py** (~50 lines)
- Wraps existing device API
- `collect_data()`: Get sensor readings from device
- `send_params()`: Send control parameters to device
- Handles communication timeout

#### **optimizer.py** (~150 lines)
- `OptimizationLoop` class with 5 sequential steps:
  - `step_1()`: Data preprocessing
  - `step_2()`: Feature calculation
  - `step_3()`: Optimization algorithm
  - `step_4()`: Constraint checking
  - `step_5()`: Output formatting
- `execute_full_cycle()`: Orchestrates all 5 steps
- Each step decorated with `@with_retry` and `@validate_output`

#### **error_handler.py** (~120 lines)
- `ErrorHandler` class (uses composition, receives optimizer instance)
- **Error Detection**:
  - `validate_output()`: Check if numbers are in expected range
  - `check_control_effectiveness()`: Verify control meets requirements
  - `handle_step_error()`: Catch and classify exceptions
- **Error Recovery Actions**:
  - `rerun_step()`: Retry specific step with modified params
  - `logout_and_quit()`: Graceful shutdown with data saving
  - `restart_cycle()`: Reset and start from step 1
  - `emergency_stop()`: Immediate device safe state
- **Decision Logic**:
  - `decide_action()`: Maps error type to recovery strategy

#### **decorators.py** (~80 lines)
- `@with_retry(max_attempts, delay)`: Automatic retry on failure
- `@validate_output(validator_func)`: Output range/type checking
- `@log_execution(log_level)`: Automatic logging of function calls
- `@timeout(seconds)`: Prevent hanging operations

#### **controller.py** (~100 lines)
- `ControlSystem` class (main orchestrator)
- Initializes device, optimizer, error_handler
- `run()`: Main control loop (runs at 1Hz)
- State machine: IDLE → COLLECTING → OPTIMIZING → SENDING → VALIDATING
- Graceful shutdown handling

#### **utils.py** (~60 lines)
- `setup_logging()`: Configure logging to file and console
- `load_config()`: Read YAML configuration
- `save_state()`: Checkpoint current state
- `is_in_range()`, `is_valid_output()`: Common validators

## 4. Data Flow

```
┌─────────────────────────────────────────────────────┐
│               1Hz Control Loop (1 second)           │
└─────────────────────────────────────────────────────┘

Device (1Hz) 
    │
    │ sensor_data (dict)
    ↓
DeviceInterface.collect_data()
    │
    │ raw_data
    ↓
OptimizationLoop.execute_full_cycle()
    │
    ├─> step_1(raw_data) → processed_data
    ├─> step_2(processed_data) → features
    ├─> step_3(features) → optimized_params
    ├─> step_4(optimized_params) → checked_params
    └─> step_5(checked_params) → final_output
    │
    │ final_output (dict)
    ↓
ErrorHandler.validate_output(final_output)
    │
    ├─> Valid → continue
    └─> Invalid → decide_action() → rerun/logout/restart
    │
    ↓
DeviceInterface.send_params(final_output)
    │
    ↓
Device (1Hz)
```

## 5. Error Handling Strategy

### 5.1 Error Types

| Error Type | Trigger Condition | Recovery Action |
|------------|-------------------|-----------------|
| **Step Execution Error** | Exception in step 1-5 | Retry up to 3 times → Restart cycle |
| **Invalid Output Range** | Numbers outside bounds | Rerun step 4 with relaxed constraints |
| **Control Ineffective** | Device response not as expected | Log warning → Continue (3 strikes → Logout) |
| **Communication Timeout** | Device not responding | Emergency stop → Logout |
| **Critical Exception** | Unhandled error | Save state → Logout immediately |

### 5.2 Recovery Decision Tree

```
Error Detected
    │
    ├─> Retriable? (< max_attempts)
    │   └─> Yes → Rerun step
    │   └─> No → Escalate
    │
    ├─> Critical?
    │   └─> Yes → Emergency stop + Logout
    │   └─> No → Continue
    │
    └─> Repeated failure? (strike_count > threshold)
        └─> Yes → Logout and quit
        └─> No → Log + Continue
```

## 6. Configuration Parameters (settings.yaml)

```yaml
device:
  timeout_seconds: 0.8
  retry_attempts: 3

optimization:
  step_1_params: {...}
  step_2_params: {...}
  output_range: [min, max]

error_handling:
  max_retry_attempts: 3
  strike_threshold: 3
  enable_auto_restart: true

logging:
  level: INFO
  file: logs/control_{timestamp}.log
```

## 7. Execution Flow

1. **Initialization**: Load config → Setup logging → Connect device
2. **Main Loop**: 
   - Wait for 1Hz tick
   - Collect data
   - Optimize (with error handling at each step)
   - Validate output
   - Send to device
3. **Shutdown**: Save final state → Close device connection → Write summary log

## 8. Testing Strategy

- **Unit Tests**: Each optimization step independently
- **Integration Tests**: Full cycle with mocked device
- **Error Injection Tests**: Simulate various failure scenarios
- **Performance Tests**: Ensure < 1 second execution time

## 9. Future Considerations

- Add data recording for post-experiment analysis
- Web dashboard for real-time monitoring
- Machine learning model integration in optimization steps
- Multi-device support

---

**Document Version**: 1.0  
**Last Updated**: 2025-12
