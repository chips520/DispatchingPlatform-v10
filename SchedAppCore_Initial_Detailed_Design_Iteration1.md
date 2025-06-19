## SchedAppCore - Initial Detailed Design (Iteration 1)

### 1. Introduction
SchedAppCore is an intelligent laboratory automation system designed to orchestrate and manage tasks within a laboratory environment. It aims to improve efficiency and reliability by automating workflows involving various laboratory devices, including AGVs, robotic arms, and analytical instruments. The system is built upon the .NET 6.0 framework, utilizing WPF for the user interface.

### 2. System Architecture (High-Level)
The system employs a layered and event-driven architecture. The core functionalities are distributed across several key modules, enabling modularity and separation of concerns.

-   **CcdqDispatchingCommon:** This is the core engine of the system. It houses the `MainController`, which coordinates all system activities, manages shared data, and handles communication interfaces. It's responsible for task dispatching logic, state management, and overall system orchestration.
-   **CcdqDispatchingPlatform:** This module provides the WPF-based graphical user interface (GUI) for system monitoring, control, and debugging. It allows users to interact with the system, view task statuses, and manage device operations.
-   **CcdqT13Task:** This module is responsible for the execution logic of specific tasks. It defines how tasks are broken down into sub-tasks and how individual steps are performed. It interacts closely with `CcdqDispatchingCommon` for task management and `TestLib02` for device interactions.
-   **TestLib02 (CcdqCommLib):** This library provides the communication layer for interacting with various physical devices. It abstracts different communication protocols (TCP, HTTP, MQTT, etc.) and offers a standardized interface for sending commands and receiving data from laboratory equipment.

**Textual Representation of High-Level Architecture:**

```
[External Systems (LIS, WMS, WCS)] <--> [CcdqDispatchingPlatform (WPF UI)]
                                        |
                                        V
[CcdqDispatchingCommon (Core Engine, MainController)] <--> [CcdqT13Task (Task Execution & Logic)]
                                        |
                                        V
[TestLib02/CcdqCommLib (Device Communication)] <--> [Physical Devices (AGVs, Robots, Analyzers, Conveyors)]
```

### 3. Core Components - Detailed Design

#### 3.1 CcdqDispatchingCommon
-   **MainController:**
    -   **Responsibilities:** Acts as the central coordinator of the entire system. It initializes and manages other key controllers and services. It orchestrates task assignments, monitors system status, and handles external system interactions.
    -   **Key managed components:**
        -   WebSocket client (for real-time UI updates and potential external communication).
        -   Quartz.NET (for scheduled tasks and timed operations).
        -   MQTT Controller (for message-based communication, primarily task commands and status updates).
        -   HTTP services (for API interactions with external systems).
        -   Device Manager (`DevManager`): Manages device connections, status, and communication interfaces.
        -   Task Controllers (instances derived from `ICTaskController` managed within `CcdqT13Task` but orchestrated by `MainController`).
-   **ShareDataContainer:**
    -   **Purpose:** Provides a thread-safe cache for shared data and system state. This allows different components to access and modify data concurrently without race conditions.
    -   **Key data managed:**
        -   `taskMap` (e.g., `ConcurrentDictionary<string, CTaskMain>`): Stores active tasks and their states.
        -   Device states and configurations (`DevUnitVo`).
        -   AGV permissions and status.
        -   Real-time system parameters and logs.
-   **Task Control Architecture (ICTaskController, ICTaskMain, ICTaskSub):**
    -   **Role of interfaces:**
        -   `ICTaskController`: Defines the contract for managing a collection of related tasks, often for a specific device or workflow.
        -   `ICTaskMain`: Represents a primary task, which can be composed of multiple sub-tasks. It manages the lifecycle of this primary task.
        -   `ICTaskSub`: Represents an individual step or sub-task within a `ICTaskMain`. It executes a specific action, often involving device interaction.
    -   **Event-driven nature:** The task execution framework is event-driven, utilizing `CTaskEvent` objects to signal changes in task status, completion of steps, or errors. This allows for asynchronous operation and decoupled components.
-   **Communication Interface Layer (AbstractComm, ICommInterface):**
    -   **Purpose:** Provides an abstraction layer for device communication, decoupling the core logic from the specifics of communication protocols. `ICommInterface` defines the standard methods for communication, while `AbstractComm` likely provides a base implementation with common functionalities.
-   **MQTT System (within Common):**
    -   **Role:** Facilitates message-based communication for task command reception from external systems (like WMS/LIS) and broadcasting system status or task updates. It's a key component for integrating SchedAppCore into a larger automated environment.
    -   **Key message types:** `TaskCommandMessage` (used to initiate tasks), device status messages, AGV control messages.

#### 3.2 CcdqDispatchingPlatform (WPF Application)
-   **Purpose:** Serves as the primary user interface for operators and administrators to monitor, control, and debug the SchedAppCore system.
-   **Key UI Features:**
    -   **MainWindowV3:** The main application window, designed with a modular approach using UserControls for different functionalities (e.g., task monitoring, device status, AGV control).
    -   **Debugging windows:**
        -   MQTT Message Monitoring Window: To inspect MQTT messages.
        -   System Cache Viewer: To view the contents of `ShareDataContainer`.
        -   Task Information Monitor: To track the status and details of ongoing tasks.
        -   AGV Debugging Interface: For specific AGV control and status viewing.
-   **Core Dependencies:** `CcdqDispatchingCommon` (for accessing core logic, data, and services) and `CcdqT13Task` (for task-specific UI elements or interactions).

#### 3.3 CcdqT13Task (Task Module)
-   **Purpose:** Encapsulates the logic for executing specific types of laboratory tasks. It defines the workflows, steps, and device interactions required to complete these tasks. This module often contains implementations of `ICTaskMain` and `ICTaskSub`.
-   **CTaskMain:**
    -   **Responsibilities:** Manages the overall lifecycle of a complex task. This includes initializing the task, breaking it down into `CTaskSub` instances, managing their concurrent or sequential execution, handling errors, and reporting final task status.
    -   **Execution modes:** While not explicitly detailed as "Single, Multi" in docs 1-4, the architecture implies support for concurrent execution of multiple `CTaskMain` instances and concurrent execution of `CTaskSub` within a `CTaskMain` where appropriate (e.g., managing multiple devices simultaneously for a complex workflow). The design supports various task execution patterns through the flexible `CTaskMain`/`CTaskSub` structure.
-   **CTaskSub:**
    -   **Role:** Represents an atomic step within a `CTaskMain`. It executes a specific, well-defined action, such as sending a command to a device, waiting for a device response, or performing a data processing step.
-   **Task Queuing:** Tasks are typically initiated via messages (MQTT, HTTP) or UI commands. These requests are received by `MainController`, which then instantiates and manages the appropriate `CTaskMain` objects. `ShareDataContainer` (`taskMap`) holds these active tasks. Prioritization and sequencing logic would reside within `MainController` or dedicated task scheduling components if further detailed.

#### 3.4 TestLib02 / CcdqCommLib (Communication Library)
-   **Purpose:** Provides a robust and extensible library for handling low-level communication with a variety of laboratory devices and systems. It abstracts protocol details, allowing other modules to interact with devices through a consistent interface.
-   **Supported Protocols (as per docs 1-4):**
    -   TCP/IP (for direct device communication)
    -   Modbus TCP (common industrial protocol)
    -   HTTP/HTTPS (for web-based APIs and external system integration)
    -   MQTT (for message queuing and event-driven communication)
    -   WebSocket (for real-time bidirectional communication, often with UI or specific devices)
-   **Key Communication Modules:**
    -   **AbstractComm:** A base class providing common functionality and defining the interface for specific communication protocol implementations.
    -   **Specific implementations:**
        -   `CcdqCommTcp`: Handles TCP/IP client/server communication.
        -   `CcdqCommModbusTcp`: Implements Modbus TCP client logic.
        -   HTTP client modules (likely using .NET's `HttpClient`).
        -   MQTT client integration (using libraries like MQTTnet).
        -   WebSocket client/server modules.
        -   Specialized modules for AGV communication (e.g., `CcdqAgvServer`, `CcdqAgvShellClient`), conveyor systems, and other specific device types.

### 4. Data Flow
A typical task execution data flow:
1.  **Task Initiation:** An external system (e.g., LIS, WMS) or a user via the `CcdqDispatchingPlatform` UI sends a task request. This request is often formatted as a `TaskCommandMessage` and transmitted via MQTT or an HTTP POST request.
2.  **Reception & Processing by MainController:** The `MainController` in `CcdqDispatchingCommon` receives the request. Its MQTT Controller or HTTP listener processes the incoming message.
3.  **Task Creation & Dispatch:** `MainController` validates the request, creates an instance of the appropriate `CTaskMain` (from `CcdqT13Task`), and stores it in `ShareDataContainer.taskMap`.
4.  **Task Execution (CTaskMain & CTaskSub):** The `CTaskMain` instance begins execution. It breaks the task into `CTaskSub` steps.
5.  **Device Interaction:** `CTaskSub` instances, when needing to interact with hardware, use communication modules from `TestLib02/CcdqCommLib`. For example, a `CTaskSub` might request `DevManager` (in `CcdqDispatchingCommon`) to get a communication channel (`AbstractComm` instance) for a specific device (`DevUnitVo`).
6.  **Command & Data Exchange:** The `AbstractComm` instance sends commands to the physical device (e.g., AGV, analyzer) and receives responses/data.
7.  **Status Updates & Events:** Device responses and internal task step completions generate `CTaskEvent`s. These events are propagated back through `CTaskSub` to `CTaskMain`. `MainController` and `ShareDataContainer` are updated with the latest task and device statuses.
8.  **UI Updates:** `CcdqDispatchingPlatform` listens for state changes (e.g., via WebSocket or by polling `ShareDataContainer`) and updates the UI to reflect task progress and device statuses.
9.  **Task Completion:** Once all `CTaskSub`s are complete, `CTaskMain` finalizes the task, updates its status in `ShareDataContainer`, and potentially notifies external systems via MQTT or HTTP.

Key data structures/models:
-   `TaskCommandMessage`: JSON structure for receiving task requests.
-   `DevUnitVo`: Value Object representing a device, its configuration, and status.
-   `CTaskEvent`: Used for signaling within the task execution framework.
-   `ShareDataContainer` stores various `ConcurrentDictionary` collections for tasks, devices, etc.

### 5. Key Design Principles Observed
-   **Modularity and Separation of Concerns:** The system is divided into distinct modules (`Common`, `Platform`, `T13Task`, `CommLib`) with well-defined responsibilities.
-   **Event-Driven Architecture:** Task execution (`CTaskEvent`) and MQTT messaging are core examples, enabling asynchronous and decoupled operations.
-   **Abstraction:**
    -   Communication: `AbstractComm` and `ICommInterface` hide protocol-specific details from the core logic.
    -   Tasks: `ICTaskController`, `ICTaskMain`, and `ICTaskSub` interfaces allow for flexible and extensible task definitions.
-   **Concurrency Control:** The use of `ConcurrentDictionary` in `ShareDataContainer` and the design of `CTaskMain` to manage multiple sub-tasks imply considerations for safe concurrent operations. Documents also mention thread safety for `ShareDataContainer` and careful management of shared resources.

### 6. Initial Architecture Diagram (Textual Representation)

```
+---------------------------------+      +---------------------------------------+      +--------------------------------+
| External Systems (LIS, WMS)     |<---->| CcdqDispatchingPlatform (WPF)         |<---->| User / Operator                |
| - Send Task Commands (MQTT/HTTP)|      | - MainWindowV3 (UserControls)         |      +--------------------------------+
| - Receive Status Updates        |      | - Debug Windows (MQTT, Cache, Task)   |
+---------------------------------+      | - AGV Control Interface               |
                                       +-------------------|---------------------+
                                                           | (User Commands, UI Updates via WebSocket/Polling)
                                                           V
+-------------------------------------------------------------------------------------------------------------+
| CcdqDispatchingCommon                                                                                       |
| +---------------------------------+      +---------------------------------------+      +-------------------+--------------+
| | MainController                  |----->| ShareDataContainer                    |<---->| CcdqT13Task Module             |
| | - Orchestrates all components   |      | - taskMap (ConcurrentDictionary)      |      | (Implementations of ICTask...)|
| | - Manages:                      |      | - deviceStateMap (ConcurrentDictionary) |      | +----------------------------+ |
| |   - MqttController (Broker conn)|      | - agvPermissionMap                    |      | | TaskController (ICTaskCtrl)| |
| |   - DevManager                  |      | - System Logs, Configs                |      | |  - Manages CTaskMain(s)    | |
| |   - Quartz Scheduler            |      +---------------------------------------+      | +----------------------------+ |
| |   - WebSocket Server/Manager    |                                                     |               |                |
| |   - HTTP Listeners/Clients      |                                                     |               V                |
| |   - Task Controllers (via T13)  |                                                     | +----------------------------+ |
| +-----------------|---------------+                                                     | | CTaskMain (ICTaskMain)     | |
|                   |                                                                     | |  - Task Lifecycle Mgmt     | |
|                   | (Task Cmds, Status)                                                 | |  - Manages CTaskSub(s)     | |
|                   |                                                                     | |  - Emits CTaskEvents       | |
|                   V                                                                     | +----------------------------+ |
| +---------------------------------------+                                               |               |                |
| | DevManager                      |                                                     |               V                |
| | - Manages DevUnitVo instances   |                                                     | +----------------------------+ |
| | - Provides Comm Interfaces      |                                                     | | CTaskSub (ICTaskSub)       | |
| +-----------------|---------------+                                                     | |  - Atomic Task Steps       | |
|                   | (Device Control Requests)                                           | |  - Device Interaction      | |
|                   V                                                                     | +----------------------------+ |
| +---------------------------------------------------------------------------------------+      +--------------------------------+ |
| | TestLib02 / CcdqCommLib                                                               |<-----| (Task Execution, Events)       |
| | - AbstractComm (Base for protocols)                                                   |      +--------------------------------+
| | - CcdqCommTcp, CcdqCommModbusTcp                                                      |
| | - HttpClient, MqttClient (e.g., MQTTnet), WebSocketClient                             |
| | - Specialized AGV communication modules (CcdqAgvServer, CcdqAgvShellClient)           |
| | - Conveyor, Analyzer specific communication modules                                   |
| +---------------------------------|-----------------------------------------------------+
|                                   | (Protocol-specific commands/data)
|                                   V
+-------------------------------------------------------------------------------------------------------------+
| Physical Devices                                                                                            |
| - AGVs, Robotic Arms, Conveyor Belts, Analyzers, Sensors, etc.                                              |
+-------------------------------------------------------------------------------------------------------------+

```

**Key Interactions in Diagram:**

*   **External Systems to Common:** Task initiation primarily via MQTT/HTTP to `MainController`.
*   **Platform to Common:** User commands trigger actions in `MainController`; UI updates based on data in `ShareDataContainer` (potentially pushed via WebSocket from `MainController`).
*   **MainController & ShareDataContainer:** `MainController` updates and reads shared state.
*   **MainController & T13Task:** `MainController` instantiates and oversees `TaskController`/`CTaskMain` instances from the `CcdqT13Task` module.
*   **T13Task & ShareDataContainer:** Task objects update their state in `ShareDataContainer`.
*   **T13Task & CommLib:** `CTaskSub` instances use `DevManager` (which in turn uses `TestLib02/CcdqCommLib`) to communicate with devices.
*   **CommLib & Physical Devices:** Direct communication using various protocols.
*   **Event Flow:** `CTaskEvent`s flow from `CTaskSub` upwards, influencing `CTaskMain` state, which is reflected in `ShareDataContainer` and potentially broadcast by `MainController`. Device events from `CommLib` can also trigger actions or state changes.
