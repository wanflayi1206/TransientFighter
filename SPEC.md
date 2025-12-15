# Technical Specification Document - TransientFighter Control System

## 1. Document Information

- **Version**: 1.0.0
- **Last Updated**: 2025-12-15
- **Related Documents**: DESIGN.md
- **Status**: Draft

## 2. Module Specifications

### 2.1 device_interface.py

#### 2.1.1 Module Overview
Device communication layer providing abstraction over hardware API.

#### 2.1.2 Classes

##### DeviceInterface
Wraps existing device API for sensor data collection and parameter transmission.

**Initialization:**
```python
def __init__(self, config: Dict[str, Any]) -> None:
    """
    Initialize device interface.
    
    Args:
        config: Dictionary containing:
            - timeout_seconds: float (0.1 - 1.0, default: 0.8)
            - retry_attempts: int (1 - 5, default: 3)
            - device_address: str (optional, device-specific)
    
    Raises:
        DeviceConnectionError: If device cannot be initialized
        ConfigurationError: If config parameters are invalid
    """
```

**Public Methods:**

```python
def collect_data(self) -> Dict[str, float]:
    """
    Collect sensor readings from device.
    
    Returns:
        Dict with keys:
            - timestamp: float (Unix timestamp)
            - sensor_1: float (range: 0.0 - 100.0)
            - sensor_2: float (range: -50.0 - 50.0)
            - sensor_3: float (range: 0.0 - 1000.0)
            - status: str ('ok' | 'warning' | 'error')
    
    Raises:
        CommunicationTimeoutError: If device doesn't respond within timeout
        DeviceNotReadyError: If device is not in ready state
        DataValidationError: If received data fails validation
    
    Performance:
        - Execution time: < 300ms (target: 200ms)
        - Success rate: > 99% under normal conditions
    """

def send_params(self, params: Dict[str, float]) -> bool:
    """
    Send control parameters to device.
    
    Args:
        params: Dictionary containing:
            - control_param_1: float (range: 0.0 - 10.0)
            - control_param_2: float (range: -5.0 - 5.0)
            - control_param_3: float (range: 0.0 - 100.0)
    
    Returns:
        bool: True if parameters were successfully sent and acknowledged
    
    Raises:
        CommunicationTimeoutError: If device doesn't acknowledge within timeout
        ParameterValidationError: If params are outside allowed ranges
        DeviceNotReadyError: If device cannot accept parameters
    
    Performance:
        - Execution time: < 200ms (target: 150ms)
        - Success rate: > 99.5%
    """

def disconnect(self) -> None:
    """
    Gracefully disconnect from device.
    
    Raises:
        DisconnectionError: If cleanup fails (non-critical)
    """
```

#### 2.1.3 Error Codes

| Error Code | Exception Class | Description | Recovery Action |
|------------|-----------------|-------------|-----------------|
| DI-001 | CommunicationTimeoutError | Device response timeout | Retry with exponential backoff |
| DI-002 | DeviceNotReadyError | Device in invalid state | Wait 100ms, retry up to 3 times |
| DI-003 | DataValidationError | Invalid sensor data received | Log and use last known good data |
| DI-004 | ParameterValidationError | Invalid control parameters | Reject and log error |
| DI-005 | DeviceConnectionError | Cannot establish connection | Abort initialization |

---

### 2.2 optimizer.py

#### 2.2.1 Module Overview
Core optimization engine implementing 5-step algorithm for control parameter calculation.

#### 2.2.2 Classes

##### OptimizationLoop

**Initialization:**
```python
def __init__(self, config: Dict[str, Any]) -> None:
    """
    Initialize optimization loop.
    
    Args:
        config: Dictionary containing:
            - step_1_params: Dict (preprocessing parameters)
            - step_2_params: Dict (feature calculation parameters)
            - step_3_params: Dict (optimization algorithm parameters)
            - step_4_params: Dict (constraint checking parameters)
            - step_5_params: Dict (output formatting parameters)
            - output_range: Tuple[float, float] (min, max allowed output)
    """
```

**Public Methods:**

```python
@with_retry(max_attempts=3, delay=0.1)
@validate_output(validator_func=is_dict_with_required_keys)
@log_execution(log_level='INFO')
def step_1(self, raw_data: Dict[str, float]) -> Dict[str, float]:
    """
    Data preprocessing step.
    
    Args:
        raw_data: Raw sensor data from device
            - timestamp: float
            - sensor_1: float
            - sensor_2: float
            - sensor_3: float
            - status: str
    
    Returns:
        Dict with processed data:
            - normalized_sensor_1: float (0.0 - 1.0)
            - normalized_sensor_2: float (-1.0 - 1.0)
            - normalized_sensor_3: float (0.0 - 1.0)
            - delta_t: float (time since last reading)
    
    Raises:
        PreprocessingError: If normalization fails
        InvalidInputError: If required keys missing
    
    Performance:
        - Execution time: < 50ms
    """

@with_retry(max_attempts=3, delay=0.1)
@validate_output(validator_func=is_dict_with_numeric_values)
@log_execution(log_level='INFO')
def step_2(self, processed_data: Dict[str, float]) -> Dict[str, float]:
    """
    Feature calculation step.
    
    Args:
        processed_data: Output from step_1
    
    Returns:
        Dict with calculated features:
            - feature_1: float (derived metric)
            - feature_2: float (derived metric)
            - feature_3: float (derived metric)
            - stability_index: float (0.0 - 1.0)
    
    Raises:
        FeatureCalculationError: If calculation fails
    
    Performance:
        - Execution time: < 100ms
    """

@with_retry(max_attempts=3, delay=0.1)
@validate_output(validator_func=is_valid_optimization_output)
@log_execution(log_level='INFO')
def step_3(self, features: Dict[str, float]) -> Dict[str, float]:
    """
    Optimization algorithm step.
    
    Args:
        features: Output from step_2
    
    Returns:
        Dict with optimized parameters (unconstrained):
            - param_1_opt: float
            - param_2_opt: float
            - param_3_opt: float
            - objective_value: float (optimization metric)
    
    Raises:
        OptimizationError: If optimization fails to converge
        NumericalError: If calculations result in NaN/Inf
    
    Performance:
        - Execution time: < 400ms (most compute-intensive step)
    """

@with_retry(max_attempts=3, delay=0.1)
@validate_output(validator_func=is_within_constraints)
@log_execution(log_level='INFO')
def step_4(self, optimized_params: Dict[str, float]) -> Dict[str, float]:
    """
    Constraint checking step.
    
    Args:
        optimized_params: Output from step_3
    
    Returns:
        Dict with constrained parameters:
            - param_1: float (within device limits)
            - param_2: float (within device limits)
            - param_3: float (within device limits)
            - constraints_violated: List[str] (list of violated constraints)
    
    Raises:
        ConstraintError: If constraints cannot be satisfied
    
    Performance:
        - Execution time: < 50ms
    """

@with_retry(max_attempts=3, delay=0.1)
@validate_output(validator_func=is_device_compatible_format)
@log_execution(log_level='INFO')
def step_5(self, checked_params: Dict[str, float]) -> Dict[str, float]:
    """
    Output formatting step.
    
    Args:
        checked_params: Output from step_4
    
    Returns:
        Dict with final control parameters (device-ready format):
            - control_param_1: float (0.0 - 10.0)
            - control_param_2: float (-5.0 - 5.0)
            - control_param_3: float (0.0 - 100.0)
    
    Raises:
        FormattingError: If output format conversion fails
    
    Performance:
        - Execution time: < 50ms
    """

def execute_full_cycle(self, raw_data: Dict[str, float]) -> Dict[str, float]:
    """
    Execute all 5 optimization steps in sequence.
    
    Args:
        raw_data: Raw sensor data from device
    
    Returns:
        Final control parameters ready for device
    
    Raises:
        OptimizationCycleError: If any step fails after retries
    
    Performance:
        - Total execution time: < 650ms (sum of all steps)
    """
```

#### 2.2.3 Data Schemas

**Raw Data Schema:**
```json
{
  "timestamp": 1702650000.123,
  "sensor_1": 45.67,
  "sensor_2": -12.34,
  "sensor_3": 567.89,
  "status": "ok"
}
```

**Final Output Schema:**
```json
{
  "control_param_1": 5.5,
  "control_param_2": 2.3,
  "control_param_3": 78.9
}
```

---

### 2.3 error_handler.py

#### 2.3.1 Module Overview
Error detection, classification, and recovery orchestration.

#### 2.3.2 Classes

##### ErrorHandler

**Initialization:**
```python
def __init__(self, optimizer: OptimizationLoop, config: Dict[str, Any]) -> None:
    """
    Initialize error handler with optimizer instance.
    
    Args:
        optimizer: OptimizationLoop instance to manage
        config: Dictionary containing:
            - max_retry_attempts: int (1 - 10, default: 3)
            - strike_threshold: int (1 - 10, default: 3)
            - enable_auto_restart: bool (default: True)
            - emergency_params: Dict (safe fallback parameters)
    """
```

**Public Methods:**

```python
def validate_output(self, output: Dict[str, float]) -> Tuple[bool, Optional[str]]:
    """
    Validate optimization output against expected ranges.
    
    Args:
        output: Control parameters to validate
    
    Returns:
        Tuple of (is_valid: bool, error_message: Optional[str])
        - (True, None) if valid
        - (False, "error description") if invalid
    
    Validation Rules:
        - All required keys present
        - All values are numeric (float/int)
        - All values within configured ranges
        - No NaN or Inf values
    """

def check_control_effectiveness(self, 
                                 sent_params: Dict[str, float],
                                 received_data: Dict[str, float]) -> bool:
    """
    Verify control parameters had expected effect on device.
    
    Args:
        sent_params: Parameters sent to device in previous cycle
        received_data: Sensor data received in current cycle
    
    Returns:
        bool: True if control is effective, False otherwise
    
    Logic:
        - Compare expected vs actual device response
        - Allow for measurement noise and delays
        - Track effectiveness over rolling window
    """

def handle_step_error(self, 
                      step_num: int,
                      exception: Exception,
                      context: Dict[str, Any]) -> str:
    """
    Classify and handle step execution errors.
    
    Args:
        step_num: Step number (1-5) that failed
        exception: Exception that was raised
        context: Additional context (attempt number, input data, etc.)
    
    Returns:
        str: Recovery action code ('RETRY' | 'RERUN' | 'RESTART' | 'LOGOUT' | 'EMERGENCY')
    
    Decision Logic:
        - Check exception type and severity
        - Check attempt count vs max_retry_attempts
        - Check strike count vs strike_threshold
        - Return appropriate recovery action
    """

def rerun_step(self, 
               step_num: int,
               input_data: Dict[str, float],
               modified_params: Optional[Dict[str, Any]] = None) -> Dict[str, float]:
    """
    Retry specific optimization step with optional parameter modifications.
    
    Args:
        step_num: Step number (1-5) to rerun
        input_data: Input data for the step
        modified_params: Optional modified parameters for retry
    
    Returns:
        Step output if successful
    
    Raises:
        StepRetryFailedError: If retry fails
    """

def logout_and_quit(self, reason: str, save_data: bool = True) -> None:
    """
    Graceful shutdown with data preservation.
    
    Args:
        reason: Reason for shutdown (logged)
        save_data: Whether to save current state before exit
    
    Side Effects:
        - Saves current state to logs/state_checkpoint.json
        - Writes shutdown reason to logs
        - Closes device connection
        - Exits process with code 0
    """

def restart_cycle(self) -> None:
    """
    Reset optimizer state and start from step 1.
    
    Side Effects:
        - Clears intermediate calculation results
        - Resets step counters
        - Logs restart event
    """

def emergency_stop(self) -> None:
    """
    Immediate device safe state and shutdown.
    
    Side Effects:
        - Sends emergency safe parameters to device
        - Logs critical event
        - Calls logout_and_quit with save_data=True
        - Exits process with code 1
    """

def decide_action(self, error_type: str, context: Dict[str, Any]) -> str:
    """
    Map error type to recovery strategy.
    
    Args:
        error_type: One of ERROR_TYPES (see table below)
        context: Additional context for decision making
    
    Returns:
        str: Action code ('RETRY' | 'RERUN' | 'RESTART' | 'LOGOUT' | 'EMERGENCY')
    """
```

#### 2.3.3 Error Type Decision Matrix

| Error Type | Condition | Recovery Action | Max Attempts |
|------------|-----------|-----------------|--------------|
| STEP_EXECUTION_ERROR | Exception in step 1-5 | RETRY → RESTART | 3 |
| INVALID_OUTPUT_RANGE | Numbers outside bounds | RERUN (step 4, relaxed) | 2 |
| CONTROL_INEFFECTIVE | Device response poor | Log + Continue (strikes) | 3 strikes |
| COMMUNICATION_TIMEOUT | Device not responding | EMERGENCY | 1 |
| CRITICAL_EXCEPTION | Unhandled error | LOGOUT | 0 |
| NUMERICAL_ERROR | NaN/Inf in calculations | RESTART | 1 |
| CONSTRAINT_VIOLATION | Cannot satisfy constraints | RERUN (step 3, modified) | 2 |

#### 2.3.4 Strike System Specification

```python
# Strike tracking
strike_count: int = 0  # Incremented on CONTROL_INEFFECTIVE
STRIKE_THRESHOLD: int = 3  # From config

# Strike logic
if error_type == 'CONTROL_INEFFECTIVE':
    strike_count += 1
    if strike_count >= STRIKE_THRESHOLD:
        return 'LOGOUT'
    else:
        return 'CONTINUE'

# Reset strikes on successful cycle
if cycle_successful:
    strike_count = 0
```

---

### 2.4 decorators.py

#### 2.4.1 Decorator Specifications

##### @with_retry

```python
def with_retry(max_attempts: int = 3, delay: float = 0.1) -> Callable:
    """
    Retry decorator with exponential backoff.
    
    Args:
        max_attempts: Maximum retry attempts (1-10)
        delay: Initial delay between retries in seconds (0.01 - 1.0)
    
    Behavior:
        - Attempt 1: No delay
        - Attempt 2: delay seconds
        - Attempt 3: delay * 2 seconds
        - Attempt N: delay * 2^(N-2) seconds (exponential backoff)
    
    Exceptions Retried:
        - All exceptions except SystemExit, KeyboardInterrupt
    
    Logging:
        - INFO: Retry attempt N/max_attempts
        - ERROR: All attempts failed
    
    Example:
        @with_retry(max_attempts=3, delay=0.1)
        def unstable_function():
            # May fail intermittently
            pass
    """
```

##### @validate_output

```python
def validate_output(validator_func: Callable[[Any], bool]) -> Callable:
    """
    Output validation decorator.
    
    Args:
        validator_func: Function that takes output and returns bool
    
    Behavior:
        - Calls validator_func on function return value
        - Raises OutputValidationError if validator returns False
        - Logs validation failures
    
    Example:
        def is_positive(x):
            return x > 0
        
        @validate_output(validator_func=is_positive)
        def get_value():
            return 42
    """
```

##### @log_execution

```python
def log_execution(log_level: str = 'INFO') -> Callable:
    """
    Automatic function execution logging.
    
    Args:
        log_level: Logging level ('DEBUG' | 'INFO' | 'WARNING' | 'ERROR')
    
    Logged Information:
        - Function name
        - Input arguments (sanitized, no sensitive data)
        - Execution time (milliseconds)
        - Success/failure status
        - Return value type
    
    Example:
        @log_execution(log_level='INFO')
        def process_data(data):
            return processed_data
    
    Log Format:
        "[INFO] process_data(data=<dict>) executed in 45ms - SUCCESS"
    """
```

##### @timeout

```python
def timeout(seconds: float) -> Callable:
    """
    Timeout decorator to prevent hanging operations.
    
    Args:
        seconds: Timeout in seconds (0.1 - 10.0)
    
    Behavior:
        - Raises TimeoutError if function exceeds time limit
        - Uses signal.alarm on Unix systems
        - Uses threading.Timer on Windows systems
    
    Limitations:
        - Not thread-safe on Unix systems
        - May not interrupt blocking I/O operations
    
    Example:
        @timeout(seconds=0.5)
        def may_hang():
            # Complex calculation
            pass
    """
```

---

### 2.5 controller.py

#### 2.5.1 Module Overview
Main orchestration layer managing system state machine and control loop.

#### 2.5.2 Classes

##### ControlSystem

**Initialization:**
```python
def __init__(self, config_path: str) -> None:
    """
    Initialize control system.
    
    Args:
        config_path: Path to settings.yaml file
    
    Initialization Steps:
        1. Load configuration from YAML
        2. Setup logging system
        3. Initialize DeviceInterface
        4. Initialize OptimizationLoop
        5. Initialize ErrorHandler
        6. Set state to IDLE
    
    Raises:
        ConfigurationError: If config file invalid
        InitializationError: If any component fails to initialize
    """
```

**State Machine:**

```python
# States
IDLE = 'IDLE'
COLLECTING = 'COLLECTING'
OPTIMIZING = 'OPTIMIZING'
SENDING = 'SENDING'
VALIDATING = 'VALIDATING'
ERROR = 'ERROR'
SHUTDOWN = 'SHUTDOWN'

# Valid transitions
TRANSITIONS = {
    'IDLE': ['COLLECTING', 'SHUTDOWN'],
    'COLLECTING': ['OPTIMIZING', 'ERROR'],
    'OPTIMIZING': ['SENDING', 'ERROR'],
    'SENDING': ['VALIDATING', 'ERROR'],
    'VALIDATING': ['IDLE', 'ERROR'],
    'ERROR': ['IDLE', 'SHUTDOWN'],
    'SHUTDOWN': []
}
```

**Public Methods:**

```python
def run(self) -> None:
    """
    Main control loop (1Hz operation).
    
    Loop Structure:
        while not shutdown_requested:
            1. Wait for 1Hz tick (precise timing)
            2. Execute state machine cycle
            3. Handle any errors
            4. Update metrics
    
    Timing:
        - Target cycle time: 1000ms
        - Time budget breakdown:
            - Data collection: 200ms
            - Optimization: 650ms
            - Parameter sending: 150ms
        - Total: 1000ms (no overhead budget)
    
    Error Handling:
        - Catches all exceptions
        - Delegates to ErrorHandler
        - May exit loop on critical errors
    
    Shutdown:
        - Graceful on SIGINT, SIGTERM
        - Immediate on critical error
    """

def _wait_for_tick(self) -> None:
    """
    Precise 1Hz timing control.
    
    Implementation:
        - Track last tick timestamp
        - Calculate sleep time to next 1s boundary
        - Use time.sleep() with drift compensation
    
    Accuracy:
        - Target: ±5ms jitter
        - Acceptable: ±20ms jitter
    """

def _execute_cycle(self) -> None:
    """
    Execute single control cycle through state machine.
    
    State Machine Flow:
        IDLE → COLLECTING: Start data collection
        COLLECTING → OPTIMIZING: Data received successfully
        OPTIMIZING → SENDING: Optimization complete
        SENDING → VALIDATING: Parameters sent to device
        VALIDATING → IDLE: Validation passed
        
        Any state → ERROR: On exception
        ERROR → IDLE: After recovery action
        ERROR → SHUTDOWN: On critical error
    """

def shutdown(self) -> None:
    """
    Graceful system shutdown.
    
    Shutdown Steps:
        1. Set shutdown_requested flag
        2. Complete current cycle if safe
        3. Save final state
        4. Disconnect from device
        5. Close log files
        6. Exit cleanly
    
    Timeout:
        - Maximum shutdown time: 5 seconds
        - Force exit after timeout
    """
```

#### 2.5.3 Performance Requirements

| Metric | Requirement | Target | Measurement |
|--------|-------------|--------|-------------|
| Cycle Time | < 1000ms | 950ms | Per-cycle timer |
| Collection Time | < 300ms | 200ms | DeviceInterface.collect_data |
| Optimization Time | < 650ms | 600ms | OptimizationLoop.execute_full_cycle |
| Sending Time | < 200ms | 150ms | DeviceInterface.send_params |
| Timing Jitter | < ±20ms | ±5ms | Tick-to-tick variance |
| Success Rate | > 95% | > 99% | Successful cycles / total cycles |
| CPU Usage | < 50% | < 30% | Average per cycle |
| Memory Usage | < 100MB | < 50MB | Process RSS |

---

### 2.6 utils.py

#### 2.6.1 Utility Functions

```python
def setup_logging(config: Dict[str, Any]) -> logging.Logger:
    """
    Configure logging system.
    
    Args:
        config: Logging configuration dict:
            - level: str ('DEBUG' | 'INFO' | 'WARNING' | 'ERROR')
            - file: str (path to log file, supports {timestamp} placeholder)
            - console: bool (enable console output)
            - format: str (optional, log format string)
    
    Returns:
        Configured logger instance
    
    Log Format:
        Default: "[%(asctime)s] %(levelname)-8s %(name)s - %(message)s"
    
    Handlers:
        - File handler (rotating, max 10MB, 5 backups)
        - Console handler (if enabled)
    """

def load_config(config_path: str) -> Dict[str, Any]:
    """
    Load and validate configuration from YAML file.
    
    Args:
        config_path: Path to settings.yaml
    
    Returns:
        Configuration dictionary
    
    Raises:
        ConfigurationError: If file not found or invalid
        ValidationError: If required keys missing or values out of range
    
    Validation:
        - All required sections present (device, optimization, error_handling, logging)
        - All numeric values within acceptable ranges
        - All paths are valid (for file-based configs)
    """

def save_state(state: Dict[str, Any], filepath: str) -> None:
    """
    Save system state to JSON file.
    
    Args:
        state: State dictionary to save
        filepath: Destination file path
    
    State Contents:
        - timestamp: float
        - current_cycle: int
        - last_sensor_data: Dict
        - last_control_params: Dict
        - strike_count: int
        - error_history: List[Dict]
    
    Format:
        JSON with indent=2 for readability
    """

def is_in_range(value: float, min_val: float, max_val: float) -> bool:
    """
    Check if value is within range (inclusive).
    
    Args:
        value: Value to check
        min_val: Minimum allowed value
        max_val: Maximum allowed value
    
    Returns:
        bool: True if min_val <= value <= max_val
    
    Special Cases:
        - Returns False for NaN
        - Returns False for Inf/-Inf
    """

def is_valid_output(output: Dict[str, float], 
                    required_keys: List[str],
                    ranges: Dict[str, Tuple[float, float]]) -> bool:
    """
    Comprehensive output validation.
    
    Args:
        output: Dictionary to validate
        required_keys: List of required keys
        ranges: Dict mapping keys to (min, max) tuples
    
    Returns:
        bool: True if all validations pass
    
    Validations:
        - All required keys present
        - All values are numeric (float/int)
        - All values within specified ranges
        - No NaN or Inf values
    """
```

---

## 3. Configuration Specification

### 3.1 settings.yaml Schema

```yaml
# Device Interface Configuration
device:
  timeout_seconds: 0.8        # float, range: [0.1, 1.0]
  retry_attempts: 3           # int, range: [1, 5]
  device_address: null        # str or null (optional, device-specific)
  safe_params:                # Emergency fallback parameters
    control_param_1: 5.0
    control_param_2: 0.0
    control_param_3: 50.0

# Optimization Configuration
optimization:
  # Step 1: Preprocessing
  step_1_params:
    sensor_1_range: [0.0, 100.0]    # Normalization range
    sensor_2_range: [-50.0, 50.0]
    sensor_3_range: [0.0, 1000.0]
  
  # Step 2: Feature Calculation
  step_2_params:
    window_size: 10                  # Rolling window for features
    stability_threshold: 0.8
  
  # Step 3: Optimization Algorithm
  step_3_params:
    algorithm: "gradient_descent"    # or "newton_method", "bayesian"
    max_iterations: 100
    convergence_tolerance: 1e-6
    learning_rate: 0.01
  
  # Step 4: Constraint Checking
  step_4_params:
    param_1_bounds: [0.0, 10.0]
    param_2_bounds: [-5.0, 5.0]
    param_3_bounds: [0.0, 100.0]
    relaxation_factor: 1.1           # For constraint relaxation on retry
  
  # Step 5: Output Formatting
  step_5_params:
    decimal_precision: 2
    unit_conversion: true
  
  # Global optimization settings
  output_range: [0.0, 100.0]        # Overall output validity range

# Error Handling Configuration
error_handling:
  max_retry_attempts: 3              # int, range: [1, 10]
  strike_threshold: 3                # int, range: [1, 10]
  enable_auto_restart: true          # bool
  emergency_params:                  # Safe parameters for emergency stop
    control_param_1: 5.0
    control_param_2: 0.0
    control_param_3: 50.0

# Logging Configuration
logging:
  level: "INFO"                      # "DEBUG" | "INFO" | "WARNING" | "ERROR"
  file: "logs/control_{timestamp}.log"
  console: true
  max_file_size_mb: 10
  backup_count: 5
  format: "[%(asctime)s] %(levelname)-8s %(name)s - %(message)s"

# Performance Monitoring
monitoring:
  enable_metrics: true
  metrics_file: "logs/metrics_{timestamp}.json"
  log_cycle_times: true
  alert_on_slow_cycle: true
  slow_cycle_threshold_ms: 1050
```

### 3.2 Configuration Validation Rules

| Parameter | Type | Range/Values | Default | Required |
|-----------|------|--------------|---------|----------|
| device.timeout_seconds | float | [0.1, 1.0] | 0.8 | Yes |
| device.retry_attempts | int | [1, 5] | 3 | Yes |
| optimization.step_3_params.algorithm | str | See algorithms | "gradient_descent" | Yes |
| error_handling.max_retry_attempts | int | [1, 10] | 3 | Yes |
| error_handling.strike_threshold | int | [1, 10] | 3 | Yes |
| logging.level | str | DEBUG/INFO/WARNING/ERROR | "INFO" | Yes |

---

## 4. API Contracts

### 4.1 Internal Module Interfaces

#### DeviceInterface ↔ Controller
```python
# Controller calls
data = device_interface.collect_data()  # Returns Dict[str, float]
success = device_interface.send_params(params)  # Returns bool
```

#### OptimizationLoop ↔ Controller
```python
# Controller calls
result = optimizer.execute_full_cycle(raw_data)  # Returns Dict[str, float]
```

#### ErrorHandler ↔ Controller
```python
# Controller calls
is_valid, error_msg = error_handler.validate_output(output)
action = error_handler.handle_step_error(step_num, exception, context)
effectiveness = error_handler.check_control_effectiveness(sent, received)
```

### 4.2 Data Flow Contracts

**Contract 1: Device → Controller**
- Format: Dict[str, float] with keys: timestamp, sensor_1, sensor_2, sensor_3, status
- Guarantees: All values numeric, timestamp is Unix epoch, status is string
- Timing: Data becomes stale after 1.5 seconds

**Contract 2: Controller → Optimizer**
- Format: Same as Device → Controller
- Guarantees: Data validated before passing, no missing keys
- Timing: Must be fresh (< 100ms old)

**Contract 3: Optimizer → Controller**
- Format: Dict[str, float] with keys: control_param_1, control_param_2, control_param_3
- Guarantees: All values within device-acceptable ranges, no NaN/Inf
- Timing: Generated within current 1Hz cycle

**Contract 4: Controller → Device**
- Format: Same as Optimizer → Controller
- Guarantees: Validated before sending, acknowledged by device
- Timing: Sent within 150ms of optimization completion

---

## 5. Error Handling Specification

### 5.1 Error Hierarchy

```
BaseControlError
├── DeviceError
│   ├── CommunicationTimeoutError
│   ├── DeviceNotReadyError
│   ├── DataValidationError
│   ├── ParameterValidationError
│   └── DeviceConnectionError
├── OptimizationError
│   ├── PreprocessingError
│   ├── FeatureCalculationError
│   ├── OptimizationConvergenceError
│   ├── ConstraintError
│   ├── FormattingError
│   └── NumericalError
├── ConfigurationError
└── SystemError
    ├── TimeoutError
    ├── InitializationError
    └── StateTransitionError
```

### 5.2 Error Recovery Workflows

#### Workflow 1: Step Execution Error
```
1. Exception raised in step N
2. @with_retry catches exception
3. If attempts < max_attempts:
   a. Wait (exponential backoff)
   b. Retry step N
   c. If success: Continue
   d. If fail: Repeat from 3
4. If attempts >= max_attempts:
   a. Log failure
   b. Call error_handler.handle_step_error()
   c. Execute returned action (RESTART/LOGOUT)
```

#### Workflow 2: Invalid Output
```
1. Optimizer produces output
2. error_handler.validate_output() returns False
3. Increment retry counter
4. If retries < 2:
   a. Call optimizer.step_4() with relaxed constraints
   b. Validate again
5. If retries >= 2:
   a. Use emergency_params
   b. Log critical error
   c. Increment strike count
```

#### Workflow 3: Communication Timeout
```
1. Device doesn't respond within timeout
2. CommunicationTimeoutError raised
3. error_handler.decide_action() returns 'EMERGENCY'
4. Execute emergency_stop():
   a. Send safe_params to device (if possible)
   b. Save state
   c. Logout and quit
```

---

## 6. Performance Metrics

### 6.1 Timing Budgets (per 1Hz cycle)

| Phase | Budget | Target | Monitored By |
|-------|--------|--------|--------------|
| Wait for tick | Variable | ~0ms | controller._wait_for_tick() |
| Data collection | 300ms | 200ms | device_interface.collect_data() |
| Step 1 (preprocess) | 50ms | 40ms | optimizer.step_1() |
| Step 2 (features) | 100ms | 80ms | optimizer.step_2() |
| Step 3 (optimize) | 400ms | 350ms | optimizer.step_3() |
| Step 4 (constraints) | 50ms | 40ms | optimizer.step_4() |
| Step 5 (format) | 50ms | 40ms | optimizer.step_5() |
| Output validation | 10ms | 5ms | error_handler.validate_output() |
| Send parameters | 200ms | 150ms | device_interface.send_params() |
| **TOTAL** | **1160ms** | **905ms** | controller.run() |

**Note:** Total budget exceeds 1000ms, so some overlap/parallelization may be needed.

### 6.2 Success Metrics

| Metric | Calculation | Target |
|--------|-------------|--------|
| Cycle Success Rate | successful_cycles / total_cycles | > 95% |
| Optimization Convergence Rate | converged_cycles / optimization_attempts | > 98% |
| Device Communication Success | successful_sends / total_sends | > 99.5% |
| Average Cycle Time | mean(cycle_times) | < 950ms |
| P99 Cycle Time | percentile(cycle_times, 99) | < 1050ms |

### 6.3 Resource Limits

| Resource | Limit | Monitoring |
|----------|-------|------------|
| CPU Usage (avg) | < 50% | Per-cycle measurement |
| Memory RSS | < 100MB | Per-cycle measurement |
| Disk I/O | < 1MB/s | Log file writes |
| Log File Size | < 100MB/day | Rotating file handler |

---

## 7. Testing Requirements

### 7.1 Unit Test Coverage

| Module | Functions | Target Coverage | Critical Tests |
|--------|-----------|-----------------|----------------|
| device_interface.py | 3 | 90% | Timeout handling, data validation |
| optimizer.py | 6 | 95% | All 5 steps + full cycle |
| error_handler.py | 7 | 95% | All recovery actions |
| decorators.py | 4 | 100% | Retry logic, validation |
| controller.py | 4 | 85% | State transitions, timing |
| utils.py | 5 | 90% | Config loading, validation |

### 7.2 Integration Test Scenarios

1. **Happy Path**: Full cycle with no errors (10 consecutive cycles)
2. **Transient Error Recovery**: Random step failures, verify retries work
3. **Communication Timeout**: Device stops responding, verify emergency stop
4. **Invalid Output**: Optimizer produces out-of-range values, verify recovery
5. **Performance Test**: 100 cycles, verify all < 1000ms
6. **Strike System**: Trigger 3 control ineffective errors, verify logout
7. **Graceful Shutdown**: Send SIGTERM during cycle, verify clean exit

### 7.3 Error Injection Tests

```python
# Test: Step 3 fails twice then succeeds
mock_optimizer.step_3.side_effect = [
    OptimizationError("Test failure 1"),
    OptimizationError("Test failure 2"),
    {"param_1_opt": 5.0, ...}  # Success on 3rd attempt
]

# Expected: 2 retries, then success
# Verify: Total attempts = 3, final output valid
```

---

## 8. Data Schemas (JSON)

### 8.1 State Checkpoint Format
```json
{
  "version": "1.0.0",
  "timestamp": 1702650000.123,
  "current_cycle": 1234,
  "state": "IDLE",
  "last_sensor_data": {
    "timestamp": 1702650000.123,
    "sensor_1": 45.67,
    "sensor_2": -12.34,
    "sensor_3": 567.89,
    "status": "ok"
  },
  "last_control_params": {
    "control_param_1": 5.5,
    "control_param_2": 2.3,
    "control_param_3": 78.9
  },
  "strike_count": 0,
  "error_history": [
    {
      "timestamp": 1702649950.0,
      "error_type": "STEP_EXECUTION_ERROR",
      "step_num": 3,
      "action": "RETRY",
      "resolved": true
    }
  ],
  "metrics": {
    "total_cycles": 1234,
    "successful_cycles": 1200,
    "avg_cycle_time_ms": 920.5
  }
}
```

### 8.2 Metrics Log Format
```json
{
  "timestamp": 1702650000.123,
  "cycle_num": 1234,
  "timings": {
    "collection_ms": 198.5,
    "step_1_ms": 42.1,
    "step_2_ms": 87.3,
    "step_3_ms": 356.8,
    "step_4_ms": 38.9,
    "step_5_ms": 41.2,
    "validation_ms": 4.5,
    "sending_ms": 145.6,
    "total_ms": 914.9
  },
  "success": true,
  "errors": [],
  "sensor_data_summary": {
    "sensor_1": 45.67,
    "sensor_2": -12.34,
    "sensor_3": 567.89
  },
  "control_params_summary": {
    "control_param_1": 5.5,
    "control_param_2": 2.3,
    "control_param_3": 78.9
  }
}
```

---

## 9. Deployment Specification

### 9.1 System Requirements
- **Python**: 3.8 or higher
- **OS**: Linux (preferred), Windows, macOS
- **Memory**: Minimum 256MB, Recommended 512MB
- **CPU**: 1 core minimum, 2 cores recommended
- **Disk**: 1GB for logs and checkpoints

### 9.2 Dependencies
```txt
# Core dependencies
PyYAML >= 5.4
numpy >= 1.20.0
scipy >= 1.6.0  # For optimization algorithms

# Development dependencies
pytest >= 6.0
pytest-cov >= 2.10
black >= 21.0
mypy >= 0.900
```

### 9.3 Installation Steps
```bash
# 1. Clone repository
git clone https://github.com/wanflayi1206/TransientFighter.git
cd TransientFighter

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure settings
cp config/settings.example.yaml config/settings.yaml
# Edit settings.yaml with device-specific parameters

# 5. Run tests
pytest tests/

# 6. Start system
python main.py --config config/settings.yaml
```

### 9.4 Runtime Directory Structure
```
TransientFighterFCVer/
├── config/
│   └── settings.yaml          # Active configuration
├── logs/
│   ├── control_20251215.log   # Daily log files
│   ├── metrics_20251215.json  # Metrics data
│   └── state_checkpoint.json  # Last saved state
├── src/                       # Source code
├── tests/                     # Test suite
├── venv/                      # Virtual environment (gitignored)
├── main.py                    # Entry point
├── requirements.txt           # Python dependencies
├── DESIGN.md                  # Design document
├── SPEC.md                    # This specification
└── README.md                  # User guide
```

---

## 10. Validation Checklist

### 10.1 Implementation Validation

- [ ] All modules implement specified APIs
- [ ] All error types are properly defined and raised
- [ ] All decorators function as specified
- [ ] Configuration validation enforces all rules
- [ ] State machine follows transition rules
- [ ] Timing budgets are monitored and logged
- [ ] All error recovery workflows are implemented
- [ ] Performance metrics are collected and logged

### 10.2 Testing Validation

- [ ] Unit test coverage meets targets (>90%)
- [ ] All integration scenarios pass
- [ ] Error injection tests verify recovery
- [ ] Performance tests validate <1s cycles
- [ ] Load test: 1000 consecutive cycles successful
- [ ] Shutdown test: Clean exit on SIGTERM/SIGINT

### 10.3 Documentation Validation

- [ ] README.md provides user guide
- [ ] DESIGN.md matches implementation
- [ ] SPEC.md (this document) is accurate
- [ ] Code comments explain complex logic
- [ ] Configuration examples are provided
- [ ] API docstrings follow specification

---

## 11. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-12-15 | Generated | Initial specification from DESIGN.md |

---

## 12. Appendices

### Appendix A: Common Validator Functions

```python
def is_dict_with_required_keys(data: Any, required_keys: List[str]) -> bool:
    """Check if data is dict with all required keys."""
    if not isinstance(data, dict):
        return False
    return all(key in data for key in required_keys)

def is_dict_with_numeric_values(data: Any) -> bool:
    """Check if data is dict with all numeric values."""
    if not isinstance(data, dict):
        return False
    return all(isinstance(v, (int, float)) and not math.isnan(v) and not math.isinf(v) 
               for v in data.values())

def is_valid_optimization_output(data: Any) -> bool:
    """Check if data is valid optimization step 3 output."""
    required_keys = ['param_1_opt', 'param_2_opt', 'param_3_opt', 'objective_value']
    return is_dict_with_required_keys(data, required_keys) and \
           is_dict_with_numeric_values(data)

def is_within_constraints(data: Any) -> bool:
    """Check if data satisfies constraint bounds."""
    required_keys = ['param_1', 'param_2', 'param_3']
    if not is_dict_with_required_keys(data, required_keys):
        return False
    
    bounds = {
        'param_1': (0.0, 10.0),
        'param_2': (-5.0, 5.0),
        'param_3': (0.0, 100.0)
    }
    
    return all(
        bounds[key][0] <= data[key] <= bounds[key][1]
        for key in required_keys
    )

def is_device_compatible_format(data: Any) -> bool:
    """Check if data is in device-compatible format."""
    required_keys = ['control_param_1', 'control_param_2', 'control_param_3']
    return is_dict_with_required_keys(data, required_keys) and \
           is_dict_with_numeric_values(data) and \
           is_within_constraints({
               'param_1': data['control_param_1'],
               'param_2': data['control_param_2'],
               'param_3': data['control_param_3']
           })
```

### Appendix B: Timing Measurement Template

```python
import time
from contextlib import contextmanager

@contextmanager
def measure_time(operation_name: str):
    """Context manager for timing operations."""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed_ms = (time.perf_counter() - start) * 1000
        logger.info(f"{operation_name} took {elapsed_ms:.2f}ms")

# Usage
with measure_time("step_1_preprocessing"):
    result = optimizer.step_1(raw_data)
```

### Appendix C: Emergency Parameters Rationale

The emergency safe parameters are chosen to:
- `control_param_1 = 5.0`: Mid-range, neutral setting
- `control_param_2 = 0.0`: Zero point, no bias
- `control_param_3 = 50.0`: 50% of maximum, safe operating point

These values should be customized based on specific device characteristics to ensure safety during emergency stops.

---

**END OF SPECIFICATION**
