## SchedAppCore - Detailed Design (Iteration 2-5)

### 1. Introduction
SchedAppCore is an intelligent laboratory automation system designed to orchestrate and manage complex, multi-step tasks within a dynamic laboratory environment. It aims to improve efficiency, throughput, and reliability by automating workflows involving diverse laboratory devices such as AGVs, robotic arms, CNC machines, and analytical instruments. The system is built upon the .NET 6.0 framework, utilizing WPF for the user interface, and incorporates advanced scheduling, dynamic scripting, and configurable communication capabilities.

### 2. System Architecture (High-Level)
The system employs a layered, event-driven, and increasingly service-oriented architecture. Core functionalities are distributed across several key modules, emphasizing modularity, separation of concerns, and extensibility.

-   **CcdqDispatchingCommon:** The core engine, housing the `MainController` for overall coordination, `ShareDataContainer` for state management, advanced scheduling algorithms (`CTaskSubVSec`), an instruction execution engine (`InsUnitCommunication`), dynamic scripting (`ScriptExecuteTools`), process template management (`InstructCombinationManager`), and sophisticated device management (`DevManager`, `DeviceAllocationHelper`).
-   **CcdqDispatchingPlatform:** The WPF-based GUI for system monitoring, control, configuration, and debugging.
-   **CcdqT13Task:** This module has evolved. While initially for specific task execution logic (CTaskMain, CTaskSub), its core task step execution is now largely embodied by `CTaskSubVSec` (conceptually part of the common scheduling and execution engine). The original CTaskMain/CTaskSub structure is a subject for refactoring.
-   **TestLib02 (CcdqCommLib):** The communication library providing abstractions and concrete implementations for various device protocols.

**Textual Representation of High-Level Architecture:**

```
[External Systems (LIS, WMS, WCS)] <--> [CcdqDispatchingPlatform (WPF UI)]
                                        |
                                        V
[CcdqDispatchingCommon (Core Engine, MainController, Scheduling, Scripting, Templates)] <--> [CcdqT13Task (Evolving Task Definitions, CTaskSubVSec logic)]
                                        |
                                        V
[TestLib02/CcdqCommLib (Device Communication Abstraction & Protocols)] <--> [Physical Devices (AGVs, Robots, Analyzers, CNCs, Conveyors)]
```

-   **Refinement Note on Task Management:** Analysis from Document 7 highlighted complexities in the existing `CTaskController` -> `CTaskMain` -> `CTaskSub` hierarchy, including deeply nested structures, complex state management, and confusing task execution modes. A proposed "Unified Single Task List" architecture, featuring Task Producers (`ITaskProducer`), a `CUnifiedTaskController`, and `CUnifiedTask` objects, aims to simplify task creation, management, and execution by flattening the hierarchy and standardizing task representation. This represents a significant potential evolution for the system's core task handling.

### 3. Core Components - Detailed Design

#### 3.1 CcdqDispatchingCommon
-   **MainController:**
    -   **Responsibilities:** Central coordinator, initializes and manages services (MQTT, HTTP, WebSocket, Quartz), `DevManager`, `InstructCombinationManager`, and orchestrates task execution flow (potentially via the existing or proposed task control architecture).
    -   **Key managed components:** WebSocket client, Quartz.NET, MQTT Controller, HTTP services, `DevManager`, `ShareDataContainer`, `InstructCombinationManager`, and the core task execution engine components.
-   **ShareDataContainer:**
    -   **Purpose:** Thread-safe, system-wide data cache for shared state.
    -   **Key data managed:** `taskMap` (active tasks, potentially evolving to `CUnifiedTask` instances), device states (`DevUnitVo`), AGV permissions, `dictTaskInfo` (task-specific context passed around), script cache (`DictScript`), process combination templates.
-   **Task Control Architecture (Existing):**
    -   **Role of interfaces:** `ICTaskController`, `ICTaskMain`, `ICTaskSub` define the current hierarchical task management system.
    -   **Event-driven nature:** `CTaskEvent` signals task state changes.
    -   **Critique & Proposed Refinement (from Doc 7):**
        -   **Identified Issues:**
            -   Complex nesting of task objects (`CTaskController` > `CTaskMain` > `CTaskSub`) leads to difficult state tracking and debugging.
            -   State management across these levels is overly complex.
            -   Task execution modes (Single, Multi, etc.) are confusing and add to complexity.
            -   Difficult to manage a unified view of all tasks in the system.
        -   **Proposed "Unified Task Architecture":**
            -   `ITaskProducer`: Interface for components that generate tasks (e.g., `MqttTaskProducer` for MQTT messages, `HttpTaskProducer` for API calls).
            -   `CUnifiedTaskController`: A centralized controller managing a single list/queue of `CUnifiedTask` objects. Responsible for dispatching tasks to an executor.
            -   `CUnifiedTask`: A standardized representation of a task, containing a list of `InstructUnitVo` (instructions) directly or derived from a template. Simplifies state (`TaskStatusEnum`) and execution flow. This aims to flatten the hierarchy and streamline task processing.
-   **Scheduling Algorithm (CTaskSubVSec - from Doc 5):**
    -   **Evolution:** Replaces the original `CTaskSub` with a more advanced, non-recursive instruction sequence executor. Conceptually, this is the core of instruction execution within any task (current `CTaskMain` or future `CUnifiedTask`).
    -   **Non-recursive Layered Parsing:**
        -   `ParseScheduledLevelOne`: Parses the main sequence of `InstructUnitVo` objects.
        -   `ParseScheduledLevelTwo`: Parses sub-sequences, particularly for loop control structures.
    -   **Intelligent Breakpoint Resume:**
        -   `UnitExecutedPathList`: Stores the execution path (indices of executed instructions).
        -   `dpIdx` (dynamic programming index): Allows O(1) positioning to the last executed instruction or loop iteration upon resume, crucial for recovering from pauses or communication failures.
    -   **Advanced Loop Control:**
        -   Basic loops (fixed iterations).
        -   By count: e.g., `mqCarryQuantity` from `dictTaskInfo` determining loop count for AGV carrying tasks.
        -   By condition: `condition_if` instructions evaluated by the script engine, controlling loop continuation.
        -   Dynamic loops: Loop termination based on external events, often via MQTT messages changing a flag in `dictTaskInfo`.
    -   **Performance Optimizations:**
        -   Skip optimization: Already completed instructions are skipped efficiently during resume.
        -   Memory Management: Potential use of `AssemblyLoadContext` for dynamically loaded scripts, allowing them to be unloaded to save memory (though details of its application need careful consideration for shared scripts).
-   **Instruction Execution Engine (`InsUnitCommunication` - from Doc 6 & 8):**
    -   **Core Logic:** This class is central to executing individual `InstructUnitVo` instructions. It handles timeout mechanisms (e.g., `insUnitTimeout`), looping retries (`insUnitRetry`), and dispatches execution based on `InstructTypeEnum`.
    -   **`InstructTypeEnum`:**
        -   `device_operate`: General device command.
        -   `device_status_check`: Query device status.
        -   `binding_by_status`: Device binding based on status and product matching.
        -   `binding_by_code`: Device binding based on pickup/putdown codes.
        -   `condition_if`: Conditional execution using scripting.
        -   `http_req`: HTTP request.
        -   `mqtt_publish`: Publish MQTT message.
        -   `normal`: General purpose, often used for script execution or setting values.
        -   `delay`: Introduce a delay.
        -   `custom_script`: Execute a custom C# or Python script.
        -   Flow control types (e.g., for loops, breaks).
    -   **Detailed execution flow for key instructions:**
        -   `binding_by_status`: Iterates through candidate devices, checks their status (potentially using scripts like `seerAgvStatusScript.cs`), matches product/material information (e.g., for CNC tasks, ensuring correct material is processed), and uses an atomic lock (`lockObj`) during the binding process to prevent race conditions.
        -   `binding_by_code`: Retrieves `mqPickUpCode` or `mqPutDownCode` from `dictTaskInfo` (task-specific data dictionary) and uses this to identify/reserve a specific item or location for AGV operations.
        -   `condition_if`: Utilizes the `ScriptExecuteTools` to evaluate a boolean condition. If no specific device is involved, the script provides the logic directly.
        -   `http_req`: Uses `CcdqCommHttp` from `TestLib02` to send HTTP requests. Parameters for the request are often sourced from `dictTaskInfo`.
        -   `normal`: Primarily passes context (`dictTaskInfo`, loop variables, timeout settings) to sub-logics, often script execution via `ScriptExecuteTools`.
-   **Dynamic Scripting Engine (`ScriptExecuteTools` - from Doc 8):**
    -   **Supported Languages:** C# (via Roslyn) and Python (via IronPython).
    -   **Script Caching:** `DictScript` (a `ConcurrentDictionary<string, Script>`) caches compiled scripts to avoid recompilation overhead. Scripts are keyed by a unique identifier (e.g., script content hash or name).
    -   **`ExecuteScriptType` in `InstructUnitVo`:**
        -   `ScriptContent`: The script code is directly embedded in the instruction.
        -   `PathFileName`: The instruction points to a .cs or .py file.
        -   `Context`: Uses the result from the previous instruction (`LastResult`) as input or script content.
    -   **Script Context (`ScriptVarsContext`):** Provides scripts with access to:
        -   `LastResult`: Output of the previously executed instruction.
        -   `PersistentData`: A dictionary for sharing data across script executions within the same task.
        -   `TaskInfo`: The `dictTaskInfo` providing task-level parameters.
        -   `DeviceInfo`: Information about the device associated with the current instruction.
        -   `Log`: A logging interface.
-   **Process Combination Templates (`InstructCombinationVo` & Manager - from Doc 8):**
    -   **Purpose:** Define and manage reusable sequences of instructions (`InstructUnitVo`) as named templates. This standardizes common processes, especially for complex device interactions like those of Wu-AGVs.
    -   **Structure (`InstructCombinationVo`):**
        -   `CombinationName`: Unique name for the template (e.g., "WuAGV_LoadMaterial").
        -   `List<InstructUnitVo>`: The sequence of instructions defining the process.
    -   **Wu-AGV Templates:** Document 8 mentions 8 standard templates for Wu-AGV operations (e.g., loading, unloading, charging). These templates heavily utilize scripts for dynamic decision-making and interaction with the AGV controller.
    -   **`InstructCombinationManager`:**
        -   **Registration:** Loads templates from configuration (e.g., JSON files) or code into a dictionary.
        -   **Retrieval:** Provides deep clones of `InstructCombinationVo` instances to ensure that modifications to a retrieved template for a specific task do not affect the original template or other tasks using it.
-   **Device Management (`DevManager` - further details from Doc 8):**
    -   Manages `DevUnitVo` (device information) and `AbstractComm` (communication channels).
    -   **`DeviceAllocationHelper`:**
        -   **Intelligent Binding:** Provides logic for selecting the most suitable device for a task step. Considers device status, capabilities, current workload (if available), and custom logic (e.g., proximity for AGVs).
        -   **Status Checks:** Performs comprehensive status checks before binding, potentially using customizable scripts (e.g., `seerAgvStatusScript.cs` for Seer AGVs to check if they are idle, charging, or in error).
        -   **Candidate Lists:** Can generate a list of suitable candidate devices based on criteria, allowing for fallback or selection strategies.
    -   **`TaskDeviceConfigService`:**
        -   **Default Mode:** Automatically maps tasks to devices based on pre-configured compatibility or device types.
        -   **Flexible Mode:** Allows tasks to specify preferred devices or groups, or use dynamic binding logic defined in scripts or `DeviceAllocationHelper`. Enables overriding default mappings for specific scenarios.
    -   **AGV-Specific Status Checks:** The system allows defining custom scripts (e.g., `seerAgvStatusScript.cs`) that `DevManager` or `DeviceAllocationHelper` can execute to determine the detailed status of an AGV beyond basic connectivity, tailoring checks to specific AGV vendor capabilities.

#### 3.2 CcdqDispatchingPlatform (WPF Application)
-   **Purpose:** User interface for system monitoring, control, configuration, and debugging.
-   **Key UI Features:** (As before: MainWindowV3, MQTT/Cache/Task monitors). Potential new UI elements would be needed for managing `InstructCombinationVo` templates, viewing `CUnifiedTask` lists, and interacting with the dynamic scripting engine (e.g., uploading/editing scripts if supported).
-   **Core Dependencies:** `CcdqDispatchingCommon`, `CcdqT13Task` (or its evolved/refactored equivalent).

#### 3.3 CcdqT13Task (Task Module - primarily CTaskSubVSec now)
-   **CTaskSubVSec (Second Version Task Execution - from Doc 5 & 6):**
    -   This component, or its underlying logic, represents the refined instruction execution mechanism. It's less a standalone "task module" in the old sense and more the core engine for processing sequences of `InstructUnitVo` objects, whether they come from an old `CTaskMain` or a new `CUnifiedTask`.
    -   **Layered Parsing:** `ParseScheduledLevelOne` and `ParseScheduledLevelTwo` handle the structured execution of instructions, including loops.
    -   **`InsUnitCommunication` as Core Executor:** Details of `InsUnitCommunication` are now primarily within `CcdqDispatchingCommon` as it's a shared resource. `CTaskSubVSec` is the client or orchestrator of `InsUnitCommunication` for a given instruction sequence.
    -   **State Management:** Manages a multi-dimensional state matrix for each task/instruction sequence:
        -   Execution State (Running, Paused, Error, etc.).
        -   Communication State (Sending, Receiving, Idle, Error).
        -   Resume State (tracking `dpIdx` and `UnitExecutedPathList` for breakpoint resume).
    -   **Exception Handling:** Implements layered exception handling:
        -   Instruction Level: Handled by `InsUnitCommunication` (retries, timeouts).
        -   Task Level (`CTaskSubVSec`): Catches exceptions from `InsUnitCommunication`, attempts recovery actions (e.g., breakpoint resume, invoking error handling scripts), and manages overall task state.
-   **Note on CTaskMain/CTaskController:** With the advent of `CTaskSubVSec` and the proposed "Unified Task Architecture," the original roles of `CTaskMain` and `ICTaskController` are diminished or subject to significant refactoring. `CTaskMain` might become a simple wrapper that feeds a list of instructions to `CTaskSubVSec`, or it could be replaced entirely by `CUnifiedTask`.

#### 3.4 TestLib02 / CcdqCommLib (Communication Library)
-   **Purpose:** (As before) Provides abstracted and concrete communication protocol implementations.
-   **Supported Protocols:** (As before: TCP, Modbus TCP, HTTP, MQTT, WebSocket).
-   **Key Communication Modules:** (As before: `AbstractComm`).
    -   **Specific Protocol Implementations (from Doc 8):**
        -   `CcdqCommModbusTcp`: Uses the `ModbusTcpNet` library (or a similar third-party library) for Modbus communication. Provides utility methods in `ModbusTools` for common read/write operations (e.g., holding registers, coils).
        -   `CcdqCommHttp`: Leverages .NET's `HttpClient` for making HTTP/HTTPS requests. Manages client instances and request/response handling.
-   **Configuration (`Cfg/commdef.cfg`):** A configuration file (likely JSON or XML) that maps protocol identifier strings (e.g., "ModbusTCP", "Http") to the fully qualified class names of their respective `AbstractComm` implementations within `TestLib02`. This allows for dynamic loading and extensibility of communication protocols.

### 4. Data Flow
A refined typical data flow, incorporating new elements:
1.  **Task Initiation:** External system (WMS) sends a `TaskCommandMessage` via MQTT, specifying a `CombinationName` (process template) and initial parameters in `dictTaskInfo`.
2.  **Reception & Task Creation (Proposed Unified Model):**
    -   `MqttController` (in `CcdqDispatchingCommon`) receives the message.
    -   An `MqttTaskProducer` processes the message, retrieves the corresponding `InstructCombinationVo` (template) from `InstructCombinationManager` (getting a deep clone).
    -   A `CUnifiedTask` is created, populated with the `InstructUnitVo` list from the template and the initial `dictTaskInfo`.
    -   The `CUnifiedTask` is added to the queue managed by `CUnifiedTaskController`.
3.  **Task Execution Orchestration:**
    -   `CUnifiedTaskController` (or the existing `MainController` adapting `CTaskSubVSec`) picks up the task.
    -   The core instruction execution logic (conceptually `CTaskSubVSec`) begins processing the `InstructUnitVo` list.
4.  **Instruction Execution (`InsUnitCommunication`):** For each `InstructUnitVo`:
    -   **Device Binding (if needed):** If the instruction requires a device (e.g., `binding_by_status`, `device_operate`), `InsUnitCommunication` may call `DevManager` and `DeviceAllocationHelper`. `DeviceAllocationHelper` uses status scripts and `TaskDeviceConfigService` to select and allocate a suitable `DevUnitVo`.
    -   **Script Execution:** If the instruction is `custom_script` or `condition_if`, `InsUnitCommunication` invokes `ScriptExecuteTools`. The script engine executes C# or Python code, potentially using `LastResult`, `PersistentData`, `TaskInfo`, and `DeviceInfo` from the script context.
    -   **Device Communication:** For `device_operate`, `http_req`, etc., `InsUnitCommunication` obtains the appropriate `AbstractComm` instance from `DevManager` (for the bound device) and calls its methods (e.g., `SendCmd`, `Request`).
5.  **Data Exchange & State Updates:** `CcdqCommLib` handles protocol-level communication. Responses are returned to `InsUnitCommunication`. `dictTaskInfo` may be updated by scripts or device responses. Task status and `dpIdx` are updated by `CTaskSubVSec`/`InsUnitCommunication`. All critical states are reflected in `ShareDataContainer`.
6.  **Looping & Conditional Logic:** `CTaskSubVSec` manages loops based on instruction parameters and script evaluations.
7.  **UI Updates:** `CcdqDispatchingPlatform` reflects progress from `ShareDataContainer`.
8.  **Task Completion:** The task finishes, status is updated, and notifications may be sent (e.g., MQTT).

### 5. Key Design Principles Observed
-   (As before: Modularity, Event-Driven Architecture, Abstraction, Concurrency Control)
-   **Enhanced Configurability & Extensibility:** Achieved through dynamic scripting (C#/Python), process combination templates, and configurable device allocation logic.
-   **Template-Based Design:** `InstructCombinationVo` promotes reuse and standardization of common processes.
-   **Intelligent Automation:** Features like breakpoint resume, advanced loop control, and status-based device binding reflect a move towards more autonomous and resilient operation.
-   **Layered Exception Handling:** Improves robustness by managing errors at different levels (instruction, task).

### 6. Refined Architecture Diagram (Textual Representation)

```
+---------------------------------+      +---------------------------------------+      +--------------------------------+
| External Systems (LIS, WMS)     |<---->| CcdqDispatchingPlatform (WPF)         |<---->| User / Operator                |
| - Task Cmds (MQTT/HTTP)         |      | - UI for Monitoring, Config, Debug    |      +--------------------------------+
|   (May specify CombinationName) |      | - Views for Templates, Scripts (Pot.) |
+---------------------------------+      +-------------------|---------------------+
                                                           | (UI Actions, Data Display)
                                                           V
+---------------------------------------------------------------------------------------------------------------------------------+
| CcdqDispatchingCommon                                                                                                           |
| +---------------------------------+      +---------------------------------------+      +--------------------------------------+
| | MainController                  |----->| ShareDataContainer                    |      | InstructCombinationManager           |
| | - Orchestrates services         |      | - taskMap (Current/Unified Tasks)     |      | - Manages InstructCombinationVo Tmpls|
| | - Manages (or is part of):      |      | - deviceStateMap, agvPermissionMap    |      | - Loads from Config/Code             |
| |   - MqttController              |      | - dictTaskInfo instances              |      | - Provides Deep Clones               |
| |   - HttpServices                |      | - DictScript (Script Cache)           |      +-----------------|--------------------+
| |   - WebSocket Server            |      +-------------------|-------------------+                    | (Templates)
| |   - Quartz Scheduler            |                          | (Shared State)                         |
| |   - DevManager                  |                          |                                        V
| |   - Task Execution Engine       |      +-------------------|-------------------+      +--------------------------------------+
| +-----------------|---------------+      | DevManager                            |      | Task Control / Execution             |
|                   |                      | - Manages DevUnitVo, AbstractComm map |      | (Existing: CTaskCtrl/Main/Sub)       |
|                   |                      | - Uses TaskDeviceConfigService        |      | (Proposed: CUnifiedTaskController)   |
|                   |                      +----------|--------------------------+      |   - Manages Task Queue (UnifiedTask) |
|                   |                                 | (Device Allocation)             |   - Uses ITaskProducer (e.g. MqttProducer) |
|                   |                      +----------V--------------------------+      |   - Orchestrates CTaskSubVSec/Executor|
|                   |                      | DeviceAllocationHelper                |      +-----------------|--------------------+
|                   |                      | - Intelligent Device Binding          |                    | (Task Instructs)
|                   |                      | - Custom Status Scripts (e.g. AGV)    |                    V
|                   |                      +---------------------------------------+      +--------------------------------------+
|                   |                                                                     | CTaskSubVSec (Instruction Executor)  |
|                   V (Task Commands if not via Unified Producers)                        | - Layered Parsing (Lvl1, Lvl2)       |
| +-------------------------------------------------------------------------+             | - Breakpoint Resume (dpIdx, PathList)|
| | Instruction Execution Core (`InsUnitCommunication`)                     |<------------| - Advanced Loop Control              |
| | - Executes individual `InstructUnitVo` from CTaskSubVSec/UnifiedTask    |             | - State Mgt (Exec, Comm, Resume)   |
| | - Timeout, Retry Logic                                                  |             | - Exception Handling                 |
| | - Dispatch by `InstructTypeEnum`                                        |             +-----------------|--------------------+
| |   - Calls `ScriptExecuteTools` for `condition_if`, `custom_script`    |                               |
| |   - Calls `DevManager` for device-bound ops, then `AbstractComm`        |                               |
| |   - Interacts with `dictTaskInfo` (via context)                       |                               |
| +-------------------|--------------------|--------------------------------+                               |
|                     | (Script Exec)      | (Device I/O)                                                 |
|    +----------------V--------------+     |                                                               |
|    | ScriptExecuteTools            |     |                                                               |
|    | - C# (Roslyn), Python (IronPy)|     |                                                               |
|    | - Script Caching (DictScript) |     V                                                               V
|    | - Script Context (LastRes,   |  +---------------------------------------------------------------------------------------+
|    |   PersistData, TaskInfo, etc) |  | TestLib02 / CcdqCommLib                                                               |
|    +-------------------------------+  | - AbstractComm, ICommInterface                                                        |
|                                       | - CcdqCommTcp, CcdqCommModbusTcp (uses ModbusTcpNet, ModbusTools)                    |
|                                       | - CcdqCommHttp (uses HttpClient)                                                      |
|                                       | - MQTT Client, WebSocket Client/Server impls.                                         |
|                                       | - Config: cfg/commdef.cfg (maps strings to classes)                                   |
|                                       +------------------------------------------|----------------------------------------------+
|                                                                                  | (Protocol-specific commands/data)
|                                                                                  V
+---------------------------------------------------------------------------------------------------------------------------------+
| Physical Devices                                                                                                                |
| - AGVs (Seer, Wu), CNC Machines, Robotic Arms, Conveyors, Analyzers, etc.                                                       |
+---------------------------------------------------------------------------------------------------------------------------------+
```

**Key Interaction Updates in Diagram:**

*   `InstructCombinationManager` provides templates to the Task Control layer.
*   `DevManager` uses `DeviceAllocationHelper` for binding, which can use custom scripts.
*   The Task Control layer (specifically `CTaskSubVSec` or a similar executor for `CUnifiedTask`) uses `InsUnitCommunication`.
*   `InsUnitCommunication` uses `ScriptExecuteTools` for scriptable instructions and `DevManager`/`CcdqCommLib` for device interactions.
*   `ShareDataContainer` holds cached scripts (`DictScript`) and task-specific data (`dictTaskInfo`).
*   The proposed `CUnifiedTaskController` and `ITaskProducer` are shown as part of the evolving task control.
*   Details of `CcdqCommLib` implementations and configuration are noted.
