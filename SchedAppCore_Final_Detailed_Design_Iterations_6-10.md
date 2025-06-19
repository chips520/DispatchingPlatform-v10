## SchedAppCore - Final Detailed Design (Iterations 6-10)

### 1. Introduction
SchedAppCore represents a paradigm shift in intelligent laboratory and industrial automation, architected as a world-class Industrial 4.0 system. It is engineered for unparalleled performance, including nanosecond-level response times in critical sections and robust, high-throughput task orchestration. The system provides a comprehensive, extensible, and highly configurable platform for managing complex automated workflows, integrating a diverse array of robotic systems, AGVs, CNC machinery, and analytical instruments. Its design philosophy emphasizes extreme optimization, zero-conflict resource scheduling, dynamic adaptability through scripting, and comprehensive observability, positioning it as a leading solution for demanding automation challenges.

### 2. System Architecture (High-Level)
The system maintains its layered, event-driven, and service-oriented architecture, now further refined for maximum efficiency and scalability.
-   **CcdqDispatchingCommon:** The hyper-optimized core engine. It governs all system operations, featuring the `MainController` for supreme coordination, `ShareDataContainer` for high-speed state management, the advanced `CTaskSubVSec` scheduling algorithm, the ultra-fast `InsUnitCommunication` instruction execution engine, `ScriptExecuteTools` for dynamic C#/Python execution, `InstructCombinationManager` for reusable process templates, and the sophisticated `DevManager` with its "Triple Locking Mechanism" and "Zero-Conflict Scheduling." A high-performance, asynchronous eventing system underpins its operation.
-   **CcdqDispatchingPlatform:** The WPF-based GUI, providing comprehensive tools for system monitoring, real-time control, advanced configuration, and in-depth debugging.
-   **CcdqT13Task:** This module's core execution logic is embodied by `CTaskSubVSec`. The original CTaskMain/CTaskSub hierarchical structure remains a candidate for refactoring towards the "Unified Single Task List" model to further streamline task management at the highest level.
-   **TestLib02 (CcdqCommLib):** An exceptionally comprehensive and high-performance communication library, providing a unified abstraction over a vast range of device protocols and ensuring industrial-grade reliability and speed. Claimed to encompass ~500k lines of code dedicated to device communication.

**Textual Representation of High-Level Architecture:**
```
[External Systems (LIS, WMS, WCS, ERP)] <--> [CcdqDispatchingPlatform (WPF UI)]
                                            |
                                            V
[CcdqDispatchingCommon (Hyper-Optimized Core: MainController, Scheduling, Execution, Locking, Eventing, Scripting, Templates)] <--> [CcdqT13Task (Task Definitions, CTaskSubVSec Logic)]
                                            |
                                            V
[TestLib02/CcdqCommLib (Four-Layer Communication Architecture, ~18 Protocols, Dynamic Loading, High Reliability & Performance)] <--> [Physical Devices (AGVs, Robots, Analyzers, CNCs, PLCs, Conveyors)]
```
-   **Note on Task Management:** The existing `CTaskController` -> `CTaskMain` -> `CTaskSub` hierarchy provides a structured approach, while the proposed "Unified Single Task List" architecture (with `ITaskProducer`, `CUnifiedTaskController`, `CUnifiedTask`) promises enhanced simplicity and centralized control for future iterations, potentially simplifying overall task lifecycle management and state tracking.

### 3. Core Components - Detailed Design

#### 3.1 CcdqDispatchingCommon
-   **MainController:** (As before) Orchestrates all system services and core components with a focus on high availability and responsiveness.
-   **ShareDataContainer:** (As before) Provides system-wide, thread-safe data access with optimized concurrent collections for critical state information.
-   **Task Control Architecture (Existing & Proposed Refinement):** (As before) The proposed unified model remains a key strategic refinement for simplifying task orchestration.
-   **Scheduling Algorithm (CTaskSubVSec):**
    -   (As in Iteration 2-5) Its non-recursive, layered parsing (`ParseScheduledLevelOne`, `ParseScheduledLevelTwo`), O(1) breakpoint resume (`UnitExecutedPathList`, `dpIdx`), advanced loop control (count, condition, dynamic MQTT-driven), and skip optimizations are fundamental to the system's efficient task execution. Performance claims emphasize its ability to handle complex, multi-step processes with minimal overhead.
-   **Instruction Execution Engine (`InsUnitCommunication`):**
    -   (As in Iteration 2-5) This engine executes `InstructUnitVo` instructions with extreme efficiency, claiming execution times often less than 10ms per instruction (excluding actual device I/O or lengthy script execution).
    -   **State Machine (Doc 9):** Internally, `InsUnitCommunication` implements a sophisticated state machine for each instruction, managing states like:
        -   `InstructionPending`: Awaiting execution.
        -   `PreExecuteScriptRunning`: Executing pre-condition scripts.
        -   `DeviceBinding`: Acquiring and locking device via `DevManager`.
        -   `CommunicationSending`: Actively sending command via `CcdqCommLib`.
        -   `CommunicationReceiving`: Awaiting response.
        -   `PostExecuteScriptRunning`: Executing post-condition/validation scripts.
        -   `InstructionCompleted`: Successful execution.
        -   `InstructionFailed`: Error state, triggering retry or exception.
        -   `Retrying`: Managing retry attempts.
        -   `TimeoutWaiting`: Handling timeout conditions.
    -   This stateful approach ensures robust handling of each step in an instruction's lifecycle.
-   **Device Locking & Scheduling (`DevManager` - `GetDevUnitVoAndLock`, `PredicateFindDev`):**
    -   **Triple Locking Mechanism (Doc "10th Ultimate"):** Ensures "Zero-Conflict Scheduling" and data integrity during device allocation and use.
        -   `lockDevObj`: A general lock for coarse-grained synchronization within `DevManager` during device list manipulations or broad state changes.
        -   `lockMergerDevObj` (per device or device group): A finer-grained lock used specifically when a task is binding to a device (`DevUnitVo.IsLock = true`). This prevents multiple tasks from attempting to acquire the same device simultaneously.
        -   `lockDevComm` (per communication channel object in `AbstractComm`): A lock within the communication instance itself to protect against concurrent send/receive operations on the same communication channel if the underlying protocol/library is not inherently thread-safe for such operations.
    -   **Four-Layer Filtering in `PredicateFindDev` (Doc "10th Ultimate"):** Used by `GetDevUnitVoAndLock` to find a suitable and available device:
        1.  **Model Match:** Filters by `DevUnitSettingVo.DevModelName` and `DevUnitSettingVo.DevModelGroup`.
        2.  **Group Flag:** Checks `DevUnitSettingVo.DevUnitGroupFlag` for task-specific device grouping.
        3.  **Global Lock & Free Status:** Verifies `DevUnitVo.IsLock == false` (device not locked by another task) and `DevUnitVo.DevStatus == DevStatusEnum.Free`.
        4.  **Group Free Status:** For some scenarios, ensures other devices in a related group also meet certain status criteria.
    -   **`RefreshDevList` Intelligent Incremental Update:** An optimized algorithm to update the list of available devices (`DevUnitVoListAll`) without full re-initialization, reacting to device status changes events for minimal latency.
    -   **Performance Claims:** These mechanisms collectively aim for "Zero-Conflict Scheduling," drastically reducing wait times and maximizing resource utilization, contributing to the system's high throughput.
-   **Dynamic Scripting Engine (`ScriptExecuteTools`):** (As in Iteration 2-5) C#/Python support, caching, and rich context remain key for flexibility.
-   **Process Combination Templates (`InstructCombinationVo` & Manager):** (As in Iteration 2-5) Crucial for standardizing complex operations.
-   **Event-Driven Architecture (`FireEventMainController`, `FireDevStatusEvent`, `FireDevLockEvent` - Doc 9):**
    -   **Asynchronous Event Queues:** Utilizes `ConcurrentQueue<T>` for storing events and `SemaphoreSlim` for managing worker threads that process these queues. This decouples event producers from consumers, ensuring non-blocking event publication.
    -   **Event Types:** A comprehensive set of event types for granular tracking:
        -   `CTaskEvent`: Task lifecycle events.
        -   `DevStatusChangeEvent`: Device status updates (e.g., Free, Busy, Error).
        -   `DevLockChangeEvent`: Device lock status changes.
        -   `CommStatusChangeEvent`: Communication channel status (Connected, Disconnected).
        -   `SystemLogEvent`: For logging significant system occurrences.
        -   `MqttMessageEvent`: For MQTT message arrivals.
    -   **High-Performance Dispatch:** The architecture is designed for extremely fast event dispatch, with claims of <3ms from event firing to handler invocation, enabling near real-time system responsiveness. Each major event type may have its own dedicated processing queue and thread pool managed by `FireEventMainController`.
-   **Device Management (`DevManager` - General):**
    -   (As in Iteration 2-5, including `DeviceAllocationHelper` and `TaskDeviceConfigService`).
    -   **"Five Core Data Structures" (Doc "10th Ultimate" - combined view for DevManager & CTaskController context):**
        1.  `DevUnitVoListAll`: Master list of all device configurations and dynamic states.
        2.  `TaskMainList` (or equivalent `CUnifiedTask` list): All active tasks.
        3.  `DevCommList` (within DevManager): Manages active communication channel instances.
        4.  `TaskToDevBindingMap`: Tracks which tasks are bound to which devices.
        5.  `DevGroupToTaskAffinity`: Defines preferred or restricted device-task pairings.
        (These conceptual groupings highlight the critical data managed for scheduling and execution.)

#### 3.2 CcdqDispatchingPlatform (WPF Application)
-   (As before) Provides the essential human-machine interface for this high-performance system.

#### 3.3 CcdqT13Task (Task Module - CTaskSubVSec)
-   (As in Iteration 2-5) `CTaskSubVSec` remains the workhorse for executing instruction sequences, refined for performance and reliability as part of the `CcdqDispatchingCommon`'s execution core.

#### 3.4 TestLib02 / CcdqCommLib (Communication Library - from Doc "11th Ultimate")
-   **Project Structure & Dependencies:**
    -   Project Name: `CcdqCommLib`.
    -   Framework: .NET 6.0.
    -   Key External Libraries: Utilizes robust, industry-proven libraries like `HslCommunication` (for broad protocol support, especially PLC and Modbus) and `TouchSocket` (for high-performance TCP/UDP and custom socket programming).
-   **Extensive Protocol Support (~18 claimed classes, ~500k LOC):**
    -   Provides a vast array of concrete communication classes, including but not limited to: `CcdqSeerAgvCommModbusTcp`, `CcdqFanucFocas1Comm` (for Fanuc CNCs using FOCAS1/Ethernet), `CcdqHikAgvComm` (Hikvision AGVs), `CcdqStdModbusTcpComm`, `CcdqStdTcpClientComm`, `CcdqStdTcpServerComm`, `CcdqOmronFinsTcpComm`, `CcdqMelsecFxSerialComm`, `CcdqSiemensS7TcpComm`, `CcdqWebApiHttpComm`, `CcdqMqttComm`. This extensive list underscores the library's goal of near-universal device compatibility. The ~500k LOC claim, if accurate, indicates significant depth and feature richness.
-   **Four-Layer Communication Architecture:**
    1.  **`ICommInterface` (Uniform Abstraction Layer):** Defines a standardized contract for all communication types (e.g., `Connect`, `Disconnect`, `Send`, `Receive`, `Request`, `Subscribe`). This ensures that `CcdqDispatchingCommon` interacts with all devices uniformly.
    2.  **`AbstractComm` (Template Method & Strategy Layer):**
        -   Implements `ICommInterface`.
        -   Uses the Template Method pattern for common sequences (e.g., connect -> send -> receive -> disconnect).
        -   Supports strategy pattern for interchangeable behaviors (e.g., different parsing strategies for responses).
        -   Provides common error handling, logging, and retry logic.
        -   Manages `lockDevComm` for thread-safe operations on the communication channel.
    3.  **Concrete Implementation Classes (Protocol Layer):** Specific classes like `CcdqFanucFocas1Comm`, `CcdqHikAgvComm`, `CcdqStdModbusTcpComm`. Each encapsulates the unique logic for a particular protocol or device type, leveraging third-party libraries where appropriate. For example, `CcdqHikAgvComm` would handle Hikvision's specific AGV control API.
    4.  **`ICommControllerInterface` (Model-Level Management Layer):** (Less explicitly detailed but implied for higher-level concerns) Manages collections of `ICommInterface` instances, potentially handling resource pooling (like TCP connection pools), dynamic instantiation, and configuration of communication channels at a system level, often part of `DevManager`.
-   **`ModbusTools` Deep Dive:** A utility class within `CcdqCommLib` (likely used by `CcdqStdModbusTcpComm` and related classes).
    -   **Standard Functions:** Encapsulates easy-to-use methods for all common Modbus functions (Read Coils, Read Discrete Inputs, Read Holding Registers, Read Input Registers, Write Single Coil, Write Single Register, Write Multiple Coils, Write Multiple Registers).
    -   **Data Types:** Handles various data conversions (e.g., float, int32, string) to/from Modbus register formats.
    -   **Performance Optimizations:** May include features like connection pooling (if not handled at a higher level by `ICommControllerInterface`), batch operations (reading/writing multiple registers in one request to reduce network overhead), and optimized byte manipulation.
-   **Dynamic Loading (`CommunicationHelper`):**
    -   A factory component, likely static, `CommunicationHelper.CreateInstance<ICommInterface>(DevModeVo deviceConfig)`.
    -   Uses reflection (`System.Reflection.Assembly.CreateInstance`) to instantiate concrete communication classes based on `DevModeVo.CommClass` (string specifying the fully qualified class name) and `DevModeVo.CommParams` (connection parameters). This allows adding new device protocols without recompiling the core system.
-   **Industrial Reliability:**
    -   **Connection Management:** Sophisticated retry policies for connection attempts (e.g., exponential backoff).
    -   **Health Monitoring:** Built-in mechanisms for heartbeats (if supported by protocol/device), keep-alives, and automatic detection of disconnections. Implements auto-recovery sequences to re-establish communication.
-   **Performance Claims:**
    -   **Response Time:** Aims for 20-50ms response times for typical device interactions (protocol dependent).
    -   **Concurrency:** Engineered to support 1000+ concurrent device connections reliably.
    -   **Superiority:** Positioned as superior to many commercial SCADA/MES communication drivers in terms of flexibility, performance, and integration depth with SchedAppCore.

### 4. Data Flow
-   (As in Iteration 2-5) The data flow is now understood to be highly optimized, with events, locking, and communication layers designed for minimal latency. A task initiated via MQTT might trigger: `MqttTaskProducer` -> `CUnifiedTaskController` -> `CUnifiedTask` (populated from `InstructCombinationVo`) -> `CTaskSubVSec` -> `InsUnitCommunication` -> `DevManager` (with "Triple Lock" & "Four-Layer Filter") -> `CommunicationHelper` -> `ICommInterface` (e.g., `CcdqFanucFocas1Comm`) -> Physical Device. Events are fired asynchronously throughout this process.

### 5. Key Design Principles Observed
-   (As before: Modularity, Event-Driven Architecture, Abstraction, Concurrency Control, Enhanced Configurability, Template-Based Design, Intelligent Automation, Layered Exception Handling)
-   **Extreme Optimization:** Performance is a paramount concern, evident in claims of nanosecond/millisecond timings, efficient algorithms, and optimized data structures.
-   **Zero-Conflict Resource Management:** The sophisticated locking and filtering in `DevManager` aim to eliminate contention for devices.
-   **Comprehensive Instrumentation & Observability:** Implied by the detailed event system, facilitating real-time monitoring and diagnostics.
-   **Self-Healing Characteristics:** Auto-recovery in communication, breakpoint resume in tasks, and robust error handling contribute to system resilience.
-   **Scalability & Extensibility:** Dynamic loading of communication protocols, scriptable logic, and template-based processes allow the system to adapt and grow.

### 6. Final Architecture Diagram (Textual Representation)

```
+---------------------------------+      +---------------------------------------+      +--------------------------------+
| External Systems (LIS, WMS, ERP)|<---->| CcdqDispatchingPlatform (WPF)         |<---->| User / Operator                |
| - Task Cmds (MQTT/HTTP)         |      | - UI for Monitoring, Config, Debug    |      +--------------------------------+
+---------------------------------+      +-------------------|---------------------+
                                                           | (UI Actions, Data Display via Events/Polling)
                                                           V
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| CcdqDispatchingCommon                                                                                                                                             |
| +---------------------------------+  (Manages)  +---------------------------------------+      +--------------------------------------+      +---------------------+
| | MainController                  |-----------> | EventFireController (Async Queues)    |      | InstructCombinationManager           |      | ShareDataContainer  |
| | - Orchestrates all services     |             | - Dispatches CTaskEvent, DevEvents    |      | - Manages InstructCombinationVo Tmpls|      | - TaskMap, DevMap   |
| | - High Availability Focus       |             | - <3ms Dispatch Claim                 |      | - Provides Deep Clones               |      | - ScriptCache       |
| +---------------------------------+             +---------------------------------------+      +-----------------|--------------------+      | - dictTaskInfo pool |
|         | (Coordination)                                                                         | (Templates)                         +---------------------+
|         V                                                                                          |                                         ^ (Global State)
| +-------------------------------------------------+  (Uses)  +-------------------------------------+      +--------------------------------------+ |
| | DevManager (Device & Resource Scheduling)       |<-------- | Task Control / Execution Engine       |----->| InsUnitCommunication (State Machine) | |
| | - DevUnitVoListAll (Device States)              |          | (CTaskController/CTaskMain/CTaskSubVSec|      | - Executes InstructUnitVo (<10ms)    | |
| | - GetDevUnitVoAndLock (Triple Lock, 4-L Filter) |          |  or CUnifiedTaskController/CUnifiedTask)|      | - Timeout, Retry, Dispatch by Type   | |
| | - RefreshDevList (Incremental Update)           |          | - Uses ITaskProducer (MQTT, HTTP)     |      +-----------------|--------------------+ |
| | - DeviceAllocationHelper, TaskDeviceConfigSvc   |          | - Task Lifecycle & State Management   |                    | (Scripting)             |
| | - "Zero-Conflict Scheduling"                    |          +-------------------------------------+                    |                         |
| +-------------------|-----------------------------+                                                                       V                         |
|                     | (Device Binding Req/Resp + Events)                                          +--------------------------------------+           |
|                     |                                                                             | ScriptExecuteTools                   | <---------
|                     | (Request ICommInterface instance)                                           | - C# (Roslyn), Python (IronPython)   |
|                     V                                                                             | - Script Caching (DictScript)        |
| +-----------------------------------------------------------------------------------------------+ | - Rich Script Context                |
| | TestLib02 / CcdqCommLib (Communication Library) - ~500k LOC, ~18 Protocols                  | |                                      |
| | +-------------------------------------------------------------------------------------------+ | +--------------------------------------+
| | | CommunicationHelper (Factory)                                                             | |
| | | - CreateInstance<ICommInterface>(DevModeVo) via Reflection (from DevModeVo.CommClass)     | |
| | +-------------------------------------------------------------------------------------------+ |
| | +-------------------------------------------------------------------------------------------+ |
| | | ICommInterface (Uniform Abstraction Layer)                                                | |
| | | - Define Connect, Disconnect, Send, Receive, Request, Subscribe, etc.                     | |
| | +-------------------------------------------------------------------------------------------+ |
| | +-------------------------------------------------------------------------------------------+ |
| | | AbstractComm (Template Method & Strategy Layer)                                           | |
| | | - Implements ICommInterface, common error/retry/log, `lockDevComm`                      | |
| | +-------------------------------------------------------------------------------------------+ |
| | +-------------------------------------------------------------------------------------------+ |
| | | Concrete Protocol Implementations (Protocol Layer)                                        | |
| | | - e.g., CcdqFanucFocas1Comm, CcdqStdModbusTcpComm (uses ModbusTools), CcdqHikAgvComm      | |
| | | - Utilizes HslCommunication, TouchSocket                                                  | |
| | +-------------------------------------------------------------------------------------------+ |
| | +-------------------------------------------------------------------------------------------+ |
| | | ICommControllerInterface (Model-Level Management - conceptual, part of DevManager)        | |
| | | - Resource Pooling (Connection Pools), Dynamic Instantiation Management                   | |
| | +-------------------------------------------------------------------------------------------+ |
| | - Industrial Reliability (Connection Mgt, Health Monitoring, Auto-Recovery)                 |
| | - Performance: 20-50ms response, 1000+ concurrent connections claim                         |
| +---------------------------------------------------|-----------------------------------------+
|                                                     | (Protocol-specific commands/data)
|                                                     V
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Physical Devices                                                                                                                                                  |
| - AGVs (Seer, Wu, Hikvision), CNCs (Fanuc), PLCs (Siemens, Omron, Mitsubishi), Robots, Conveyors, Analyzers                                                        |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
