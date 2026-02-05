# Architecture Documentation

## Overview

This is an open-source anti-cheat system with a kernel-mode driver and user-mode module architecture designed to detect and prevent various cheating techniques in games and applications.

## System Components

### 1. Driver (Kernel Mode) - `driver.sys`

**Language:** C  
**Privilege Level:** Ring 0 (Kernel Mode)  
**Device Name:** `\\\\.\\DonnaAC`

The kernel driver performs low-level system integrity checks and monitoring. It runs with the highest privileges and has direct access to system internals.

**Key Responsibilities:**
- Process and thread monitoring via kernel callbacks
- System module integrity validation
- NMI/APC/DPC stackwalking for detecting hooks
- EPT hook detection
- Hardware information extraction
- Hypervisor detection
- Driver object validation
- Handle table enumeration

**Main Components:**
- `DeviceControl()` - IOCTL request handler
- `SessionManager` - Manages active protection sessions
- `IRP Queue` - Cancel-safe queue for asynchronous reporting
- `Shared Mapping` - Memory-mapped buffer for timer-based operations
- `Callbacks` - Process/thread observation callbacks
- `Integrity Checks` - Module and driver validation routines

### 2. Module (User Mode) - `user.dll`

**Language:** C++  
**Privilege Level:** Ring 3 (User Mode)  
**Deployment:** Injected into protected process

The user-mode module acts as the orchestrator and bridge between the protected application and the kernel driver.

**Key Responsibilities:**
- Initiating and managing driver communication
- Dispatching periodic security checks
- Receiving and processing threat reports
- Managing encryption for secure communication
- Coordinating with optional server component

**Main Components:**
- `Dispatcher` - Orchestrates security operations and timing
- `Kernel Interface` - Handles all driver communication
- `Message Queue` - Named pipe client for server communication
- `Crypto Module` - AES-256 encryption/decryption

### 3. Server (Optional) - Go Service

**Language:** Go  
**Status:** Minimal implementation (not required)  
**Purpose:** Remote telemetry and reporting

Currently a placeholder that can be extended for centralized logging, analytics, or remote administration. The system can be built with the `NO_SERVER` flag to exclude server communication.

## Communication Mechanisms

The driver and module communicate through three distinct channels, each optimized for different use cases:

### 1. IOCTL (I/O Control) - Synchronous Commands

**Purpose:** Direct, synchronous command execution  
**Direction:** Module → Driver (request/response)

**How It Works:**
1. Module opens a handle to the driver using `CreateFileW("\\\\.\\DonnaAC")`
2. Module sends commands via `DeviceIoControl()` with specific IOCTL codes
3. Driver's `DeviceControl()` function processes requests in a large switch statement
4. Driver returns results immediately in the same call

**Session Initialization Flow:**
```
Module                                  Driver
  |                                       |
  |-- CreateFileW("\\\\.\\DonnaAC") ---->|
  |<-------- Handle Returned ------------|
  |                                       |
  |-- IOCTL_NOTIFY_DRIVER_ON_PROCESS_LAUNCH -->|
  |    (includes: PID, AES keys, module info)  |
  |                                       |
  |                                  [Creates Session]
  |                                  [Registers Callbacks]
  |                                  [Hashes Module]
  |<-------- STATUS_SUCCESS ----------|
```

**Common IOCTL Codes:**
- `IOCTL_NOTIFY_DRIVER_ON_PROCESS_LAUNCH` (0x20004) - Initiates protection session
- `IOCTL_RUN_NMI_CALLBACKS` (0x20001) - Triggers NMI-based stackwalking
- `IOCTL_VALIDATE_DRIVER_OBJECTS` (0x20002) - Validates system drivers
- `IOCTL_SCAN_FOR_UNLINKED_PROCESS` (0x20011) - Detects hidden processes
- `IOCTL_DETECT_ATTACHED_THREADS` (0x20014) - Scans for attached threads
- `IOCTL_CHECK_FOR_EPT_HOOK` (0x20018) - EPT hook detection
- `IOCTL_VALIDATE_SYSTEM_MODULES` (0x20020) - System module validation
- `IOCTL_INSERT_IRP_INTO_QUEUE` (0x20021) - Queues IRP for async reports

**Security:**
- Session validation required (except for process launch IOCTL)
- Access denied if no active session exists
- Encrypted data transmission using AES-256

### 2. I/O Completion Port - Asynchronous Reports

**Purpose:** Efficient, asynchronous threat reporting  
**Direction:** Driver → Module (push notifications)

**How It Works:**

1. **Initialization Phase:**
   - Module creates an I/O Completion Port associated with the driver handle
   - Module pre-queues 5 IRPs (I/O Request Packets) with buffers
   - Driver stores these IRPs in a cancel-safe queue

2. **Detection Phase:**
   - Driver detects a threat (e.g., invalid APC stackwalk)
   - Driver calls `IrpQueueCompletePacket()` with report data
   - If IRPs are available, driver completes one immediately
   - If no IRPs available, report is stored in deferred queue

3. **Reception Phase:**
   - Module's completion port thread receives notification via `GetQueuedCompletionStatus()`
   - Module processes the report (logs, forwards to server, etc.)
   - Module immediately queues a new IRP to maintain the pool

**Flow Diagram:**
```
Module                          Driver
  |                               |
  |-- Queue 5 IRPs -------------->|
  |                          [Stores in IRP Queue]
  |                               |
  |                          [Detects Threat]
  |                          [Completes IRP with Report]
  |                               |
  |<-- Report Data --------------|
  |                               |
[Process Report]                  |
  |                               |
  |-- Queue New IRP ------------->|
  |                          [Stores in Queue]
```

**Report Types:**
- `report_nmi_callback_failure` (50) - NMI callback validation failure
- `report_module_validation_failure` (60) - Module integrity check failed
- `report_illegal_handle_operation` (70) - Suspicious handle manipulation
- `report_invalid_process_allocation` (80) - Invalid process memory allocation
- `report_hidden_system_thread` (90) - Hidden thread detected
- `report_illegal_attach_process` (100) - Illegal process attachment
- `report_apc_stackwalk` (110) - APC stackwalk violation
- `report_dpc_stackwalk` (120) - DPC stackwalk violation
- `report_data_table_routine` (130) - HAL dispatch table violation
- `report_ept_hook` (180) - EPT hook detected

**Deferred Reports:**
- If all IRPs are in use, reports are queued in a deferred list (max 256)
- When new IRPs arrive, deferred reports are completed first
- Ensures no reports are lost even under heavy load

### 3. Shared Memory Mapping - Timer-Based Operations

**Purpose:** Low-overhead periodic security checks
**Direction:** Bidirectional (Module writes operation ID, Driver executes and updates status)

**How It Works:**

1. **Initialization:**
   - Module sends `IOCTL_INITIATE_SHARED_MAPPING` request
   - Driver allocates a non-paged kernel buffer (PAGE_SIZE = 4KB)
   - Driver creates an MDL (Memory Descriptor List) for the buffer
   - Driver maps the buffer to user-mode address space
   - Driver returns the user-mode pointer to the module

2. **Timer-Based Execution:**
   - Driver initializes a kernel timer with 15-second intervals
   - Timer triggers a DPC (Deferred Procedure Call)
   - DPC queues a work item to run at PASSIVE_LEVEL
   - Work item checks the shared buffer's `operation_id` field
   - If operation is set, driver executes the corresponding check
   - Driver updates the `status` field when complete

3. **Module Interaction:**
   - Module writes an operation ID to the shared buffer using atomic operations
   - Module can poll the status field to check completion
   - Module writes new operation IDs as needed

**Shared Buffer Structure:**
```c
typedef struct _SHARED_STATE {
    volatile UINT32 status;        // Operation status/result
    volatile UINT16 operation_id;  // Operation to execute
} SHARED_STATE;
```

**Supported Operations:**
- `ssRunNmiCallbacks` (0) - Execute NMI-based stackwalking
- `ssValidateDriverObjects` (1) - Validate all loaded drivers
- `ssEnumerateHandleTables` (2) - Enumerate process handle tables
- `ssScanForUnlinkedProcesses` (3) - Detect hidden processes
- `ssPerformModuleIntegrityCheck` (4) - Validate module integrity
- `ssScanForAttachedThreads` (5) - Detect attached threads
- `ssScanForEptHooks` (6) - Scan for EPT hooks
- `ssInitiateDpcStackwalk` (7) - Perform DPC stackwalking
- `ssValidateSystemModules` (8) - Validate system module integrity
- `ssValidateWin32kDispatchTables` (9) - Validate Win32k dispatch tables

**Advantages:**
- Minimal context switches (no IOCTL overhead)
- Asynchronous execution (module doesn't block)
- Timer-based scheduling (consistent intervals)
- Low CPU overhead (checks run at PASSIVE_LEVEL)

**Flow Diagram:**
```
Module                          Driver
  |                               |
  |-- IOCTL_INITIATE_SHARED_MAPPING -->|
  |                          [Allocate Buffer]
  |                          [Map to User Mode]
  |<-- User Mode Pointer --------|
  |                               |
[Write operation_id = 6]          |
  |                          [Timer Fires (15s)]
  |                          [DPC Queued]
  |                          [Work Item Runs]
  |                          [Reads operation_id = 6]
  |                          [Executes EPT Hook Scan]
  |                          [Updates status]
  |                               |
[Poll status field]               |
[Read completion status]          |
```

## Complete Initialization Sequence

Here's the complete flow when the module is injected into a protected process:

### Phase 1: Driver Startup (System Boot)

```
1. Windows loads driver.sys at boot (registered as system service)
2. DriverEntry() executes:
   - Creates device object "\Device\DonnaAC"
   - Creates symbolic link "\??\DonnaAC"
   - Initializes cryptographic providers (AES, SHA-256)
   - Resolves dynamic imports
   - Initializes driver configuration
3. Driver waits for user-mode connection
```

### Phase 2: Module Injection

```
1. Protected game/application starts
2. User injects user.dll into the process
3. DllMain() executes:
   - Creates new thread for initialization
   - Thread calls module::run()
```

### Phase 3: Module Initialization

```
1. Module allocates console for logging
2. Module retrieves its own information:
   - Base address
   - Module size
   - Full path
3. Module creates message_queue (named pipe to server - optional)
4. Module creates dispatcher with kernel_interface
```

### Phase 4: Kernel Interface Setup

```
1. kernel_interface constructor:
   - Opens handle to driver: CreateFileW("\\\\.\\DonnaAC")
   - Calls notify_driver_on_process_launch()
   - Calls initiate_completion_port()
```

### Phase 5: Session Establishment

```
1. notify_driver_on_process_launch():
   - Prepares session_initiation_packet:
     * Process ID
     * Session cookie (123)
     * AES-256 key (32 bytes)
     * AES IV (16 bytes)
     * Module information (base, size, path)
   - Sends IOCTL_NOTIFY_DRIVER_ON_PROCESS_LAUNCH

2. Driver's SessionInitialise():
   - Validates process handle
   - Stores session information
   - Copies module information
   - Initializes cryptographic objects
   - Hashes the module for integrity checks
   - Registers process observation callbacks
   - Marks session as active
```

### Phase 6: Completion Port Setup

```
1. initiate_completion_port():
   - Allocates 5 event_dispatcher objects with buffers (1000 bytes each)
   - Creates I/O Completion Port
   - Associates driver handle with completion port
   - Sends 5 pending IRPs to driver

2. Driver receives IRPs:
   - Stores in cancel-safe IRP queue
   - IRPs wait for threat reports
```

### Phase 7: Shared Memory Initialization

```
1. Module sends IOCTL_INITIATE_SHARED_MAPPING
2. Driver:
   - Allocates PAGE_SIZE non-paged buffer
   - Creates MDL and maps to user mode
   - Initializes timer (15-second intervals)
   - Returns user-mode pointer to module
3. Module stores pointer for future operations
```

### Phase 8: Dispatcher Execution

```
1. dispatcher::run():
   - Initializes cryptographic provider
   - Starts timer callback thread
   - Starts I/O port monitoring thread
   - Queues completion port job to thread pool
   - Enters main dispatch loop:
     * Issues periodic kernel jobs (IOCTLs)
     * Writes to shared memory for timer-based checks
     * Sleeps between iterations
```

### Phase 9: Active Monitoring

```
Driver continuously:
- Monitors process/thread creation via callbacks
- Executes timer-based integrity checks
- Validates system modules
- Detects hooks and anomalies
- Completes IRPs with threat reports

Module continuously:
- Receives reports via completion port
- Logs threats to console/debugger
- Re-queues IRPs to maintain pool
- Issues periodic IOCTL commands
- Updates shared memory operations
```

## Security Features

### Encryption

**AES-256 Encryption:**
- Session keys exchanged during initialization
- Sensitive data encrypted before transmission
- Keys stored in encrypted form in driver memory
- Separate IV (Initialization Vector) for each session

### Integrity Validation

**Module Hashing:**
- Driver computes SHA-256 hash of module on session start
- Periodic integrity checks compare current hash
- Detects runtime patching or modification

**Driver Self-Protection:**
- Dynamic import resolution (obfuscates API usage)
- Import table encryption
- Device extension pointer encryption
- XOR-based obfuscation of critical structures

### Session Management

**Active Session:**
- Only one active session per driver instance
- Session cookie validation
- Process handle verification
- Automatic cleanup on process termination

**Access Control:**
- Most IOCTLs require active session
- Session validation on every request
- Access denied without proper initialization

## Threat Detection Capabilities

### Kernel-Level Monitoring

1. **NMI Stackwalking:**
   - Uses Non-Maskable Interrupts to capture stack traces
   - Detects kernel-mode hooks and rootkits
   - Validates return addresses against known modules

2. **APC/DPC Stackwalking:**
   - Queues APCs/DPCs to all threads/CPUs
   - Captures stack traces at different IRQL levels
   - Detects inline hooks and trampolines

3. **Process/Thread Callbacks:**
   - Monitors process creation/termination
   - Detects thread injection
   - Validates thread start addresses

4. **Handle Table Enumeration:**
   - Scans process handle tables
   - Detects suspicious handle operations
   - Validates handle permissions

### Integrity Checks

1. **Module Validation:**
   - Verifies .text section integrity
   - Compares against known-good hashes
   - Detects runtime patching

2. **Driver Object Validation:**
   - Enumerates all loaded drivers
   - Validates dispatch routines
   - Detects driver object manipulation

3. **System Module Checks:**
   - Validates critical system modules (ntoskrnl, hal, etc.)
   - Checks .text section integrity
   - Detects kernel patches

4. **Dispatch Table Validation:**
   - Validates HAL dispatch tables
   - Checks Win32k dispatch tables
   - Detects SSDT hooks

### Advanced Detection

1. **EPT Hook Detection:**
   - Timing-based detection of hypervisor hooks
   - Compares execution times of hooked vs. unhooked functions
   - Detects Extended Page Table manipulation

2. **Hypervisor Detection:**
   - APERF MSR timing checks
   - INVD instruction emulation detection
   - Identifies VM environments

3. **PCI Device Scanning:**
   - Scans PCI configuration space
   - Detects malicious hardware devices
   - Validates device signatures

4. **Hidden Process Detection:**
   - Scans for unlinked EPROCESS structures
   - Detects processes hidden from task manager
   - Validates process list integrity

## Performance Considerations

### Efficient Design

**IRP Queue with Deferred Reports:**
- No polling required (event-driven)
- Reports queued if user-mode is busy
- Maximum 256 deferred reports
- Automatic overflow handling

**Shared Memory:**
- Eliminates IOCTL overhead for periodic checks
- Timer-based execution (15-second intervals)
- Work items run at PASSIVE_LEVEL (low priority)
- Minimal CPU impact

**Thread Pool:**
- Module uses thread pool for parallel operations
- Completion port thread dedicated to receiving reports
- Timer thread for periodic operations
- Main dispatch loop for orchestration

### Resource Management

**Memory:**
- Pre-allocated buffers (no runtime allocation in hot paths)
- Non-paged pool for kernel buffers
- Proper cleanup on session termination
- MDL-based memory mapping

**Synchronization:**
- Guarded mutexes for session data
- Spinlocks for IRP queue (minimal hold time)
- Atomic operations for shared memory
- Cancel-safe queue implementation

## Known Limitations

1. **Single Session:**
   - Only one protected process at a time
   - Driver must be reloaded for multiple sessions

2. **Test Signing Required:**
   - Driver is not production-signed
   - Requires test signing mode enabled

3. **Windows Version Specific:**
   - Tested on Win10 22H2 and Win11 22H2
   - May require adjustments for other versions

4. **Server Component:**
   - Currently minimal implementation
   - No remote reporting functionality

## Build Configurations

**Release - No Server - Win10:**
- Optimized build for Windows 10
- Server communication disabled
- Production-ready (except signing)

**Release - No Server - Win11:**
- Optimized build for Windows 11
- Server communication disabled
- Production-ready (except signing)

**Debug Builds:**
- Verbose logging enabled
- Additional validation checks
- Performance impact acceptable for testing

## Logging and Debugging

### Log Levels

**Driver (Kernel):**
- `LOG_ERROR_LEVEL` (0x3) - Critical errors only
- `LOG_WARNING_LEVEL` (0x7) - Warnings and errors
- `LOG_INFO_LEVEL` (0xf) - General information
- `LOG_VERBOSE_LEVEL` (0x1f) - Detailed debugging

**Module (User Mode):**
- Console output for all operations
- Real-time threat reporting
- Performance metrics

### Debug Output

**Kernel Debugger:**
- WinDbg with filter: `.ofilter donna-ac*`
- DebugView with include filter: `donna-ac*`

**User Mode:**
- Console window (AllocConsole)
- Standard output/input streams

## Conclusion

This architecture provides a robust, multi-layered anti-cheat system with:
- **Efficient communication** through three specialized channels
- **Comprehensive threat detection** at kernel and user levels
- **Low performance overhead** through careful design
- **Extensible framework** for adding new checks
- **Strong security** through encryption and integrity validation

The separation of concerns between synchronous commands (IOCTL), asynchronous reporting (completion port), and periodic checks (shared memory) creates a flexible and performant system suitable for real-time game protection.


