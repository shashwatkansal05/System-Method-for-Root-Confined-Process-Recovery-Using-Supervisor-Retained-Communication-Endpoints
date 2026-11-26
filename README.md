# POSIX-OTP ⚙️

## Root-Confined Process Recovery Using Supervisor-Retained IPC Endpoints for Embedded Linux Systems

POSIX-OTP is an **Embedded Linux fault-recovery system** designed to restore failed processes in sensor-processing pipelines without unnecessarily terminating and recreating unaffected downstream processes.

The system combines **dependency-aware fault attribution, process supervision, supervisor-retained IPC endpoints, root-confined recovery, post-restart verification, and resource observability** into a single userspace architecture.

---

## 🚀 Overview

Traditional process-recovery mechanisms may restart multiple downstream processes when an upstream process fails, even when those processes have not independently terminated.

POSIX-OTP takes a root-confined approach by identifying the actual failed process and replacing only that process while preserving existing downstream process instances.

The system combines:

- ⚙️ POSIX-based process management
- 🔍 Process failure detection
- ❤️ Heartbeat-based liveness monitoring
- 🔗 Dependency-aware fault attribution
- 🔄 Root-confined process recovery
- 🔌 Supervisor-retained IPC endpoints
- 🛡️ Downstream process continuity
- ✅ Post-restart recovery verification
- 📊 Linux resource observability
- 🔌 ESP32 sensor integration
- 📝 Structured fault and recovery records
- 🛑 Graceful process shutdown

The complete recovery mechanism operates in userspace using standard POSIX/Linux facilities, without requiring redundant hardware, kernel modifications, or external orchestration.

---

## 🎯 Objectives

The main objectives of POSIX-OTP are:

1. Monitor cooperating processes in an Embedded Linux sensor-processing pipeline.
2. Identify the actual failed process using declared dependency relationships.
3. Distinguish root failures from downstream dependent failures.
4. Replace only the identified root process instead of restarting the entire pipeline.
5. Preserve inter-process communication channels through supervisor-retained pipe descriptors.
6. Maintain unaffected downstream process instances and their execution state.
7. Verify process liveness and external sensor-interface reacquisition before declaring recovery successful.
8. Provide structured fault records and an inspectable recovery lifecycle.

---

## 🏗️ System Architecture

The system consists of an Ubuntu Linux host connected to an ESP32 sensor node through USB serial communication.

### Hardware Architecture

              ESP32 SENSOR NODE
         ┌─────────────────────────┐
         │                         │
         │        DHT11            │
         │   Temperature/Humidity  │
         │                         │
         │         LDR             │
         │      Light Level        │
         │                         │
         │         PIR             │
         │     Motion Detection    │
         │                         │
         │      SSD1306 OLED       │
         │                         │
         └────────────┬────────────┘
                      │
                 USB Serial
                      │
                      ▼
               /dev/ttyUSB0
                      │
                      ▼
             UBUNTU LINUX HOST

### Software Architecture

                    SUPERVISOR
            ┌────────────────────────┐
            │                        │
            │  Process Monitoring    │
            │  Heartbeat Monitoring  │
            │  Fault Attribution     │
            │  Recovery Management   │
            │  Resource Monitoring   │
            │                        │
            └────────────┬───────────┘
                         │
                         ▼

ESP32 ─────► INGESTION ─────► FUSION ─────► NETWORK
               │                │              │
               └────────────────┴──────────────┘
                  Supervisor-retained
                   POSIX Pipe Endpoints

### Process Responsibilities

| Component | Responsibility |
| --- | --- |
| **Supervisor** | Process supervision, fault attribution and recovery coordination |
| **INGESTION** | Acquires sensor telemetry from the ESP32 |
| **FUSION** | Receives and processes sensor telemetry |
| **NETWORK** | Receives processed telemetry |
| **Communication Manager** | Creates and manages inter-stage POSIX pipes |
| **Process Manager** | Handles process creation, termination and replacement |
| **Fault Manager** | Maintains structured process fault information |
| **Recovery Manager** | Coordinates the recovery lifecycle |
| **Resource Manager** | Monitors process and system resource information |

---

## 🔗 Dependency Relationship

The processing pipeline follows a directed dependency relationship:

        INGESTION
             │
             ▼
           FUSION
             │
             ▼
          NETWORK

- **INGESTION** provides telemetry to **FUSION**.
- **FUSION** processes the telemetry and forwards it to **NETWORK**.
- **NETWORK** is transitively dependent on **INGESTION** through **FUSION**.

The supervisor uses this declared dependency relationship to identify the recovery root and classify downstream processes as dependent failures. This prevents downstream processes from being unnecessarily restarted solely because of an upstream failure.

---

## ⚙️ Core Recovery Mechanism

The main technical focus of POSIX-OTP is the combination of root-confined process recovery and supervisor-retained inter-stage communication.

### 1. Supervisor-Retained IPC

The supervisor creates the inter-stage POSIX pipes before starting the pipeline processes and retains the required pipe descriptors.

             SUPERVISOR
                  │
           Creates POSIX Pipes
                  │
          Retains Pipe Descriptors
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼

     INGESTION           FUSION
          │                │
          └────── Pipe 1 ──┘

             FUSION
                │
                └──────── Pipe 2 ───────► NETWORK

When an upstream process terminates, the supervisor-retained endpoint keeps the corresponding communication channel available for downstream operation.

### 2. Process Replacement

When a process fails, the replacement process inherits the relevant retained descriptor through the Linux process-creation path using `fork()`.

The descriptor is connected to the required standard input or output stream using `dup2()` before executing the stage program. This allows the replacement process to reuse the existing underlying communication channel without recreating the unaffected downstream process.

### 3. Downstream Process Continuity

Instead of restarting the complete pipeline:

          INGESTION FAILURE
                  │
                  ▼
          Identify Root Process
                  │
                  ▼
          Replace INGESTION
                  │
                  ▼
       Preserve Existing IPC Channel
                  │
                  ▼
       FUSION and NETWORK Continue

The existing downstream process instances are preserved during root-process recovery.

---

## 🔍 Working Principle

The complete operating sequence is:

       Linux Supervisor Initializes
                    │
                    ▼
       Creates Inter-Stage POSIX Pipes
                    │
                    ▼
       Retains Required Pipe Descriptors
                    │
                    ▼
       Starts INGESTION, FUSION, NETWORK
                    │
                    ▼
       Process and Heartbeat Monitoring
                    │
                    ▼
             Process Failure
                    │
                    ▼
       Identify Failed Recovery Root
                    │
                    ▼
       Classify Downstream Dependents
                    │
                    ▼
       Withhold Downstream Restart
                    │
                    ▼
       Replace Only the Root Process
                    │
                    ▼
       Inherit Retained IPC Descriptor
                    │
                    ▼
       Reuse Existing Communication Channel
                    │
                    ▼
          Post-Restart Verification
                    │
                    ▼
       Verify Liveness and Hardware Interface
                    │
                    ▼
           RECOVERY SUCCESS

---

## 🔄 Recovery Lifecycle

POSIX-OTP uses an explicit recovery lifecycle to distinguish process replacement from successful recovery.

             PLANNED
                │
                ▼
           IN_PROGRESS
                │
                ▼
            VERIFYING
                │
          ┌─────┴─────┐
          ▼           ▼
       SUCCESS       FAILED

| State | Description |
| --- | --- |
| `PLANNED` | Recovery root identified and recovery action selected |
| `IN_PROGRESS` | Replacement of the failed process initiated |
| `VERIFYING` | Waiting for required post-restart liveness evidence |
| `SUCCESS` | Required verification conditions satisfied |
| `FAILED` | Required verification evidence not received within the configured condition |

For the sensor-attached `INGESTION` process, successful recovery requires reacquiring `/dev/ttyUSB0` and re-establishing communication with the ESP32 before producing the required heartbeat.

---

## ❤️ Process Health Monitoring

The supervisor monitors process health using:

- `waitpid()` for detecting child-process termination.
- Heartbeat reports for monitoring expected process liveness.
- Linux `/proc` for process CPU usage, memory usage, and process state.
- System memory information for resource observability.

Resource measurements are used for monitoring and post-incident diagnosis, not for dependency attribution or recovery-root selection.

---

## 🔬 Experimental Validation

The project was demonstrated using an Ubuntu Linux host connected to an ESP32 sensor node.

### Test 1 — INGESTION Process Failure
`INGESTION` was terminated during normal pipeline operation. The supervisor identified `INGESTION` as the recovery root, replaced it, and preserved the existing downstream processes. After the replacement reconnected to the ESP32 and reported liveness, recovery was declared successful.

### Test 2 — FUSION Process Failure
`FUSION` was terminated. The supervisor identified `FUSION` as the recovery root and classified `NETWORK` as a downstream dependent process. No restart action was issued for `NETWORK` solely because of the upstream failure.

### Test 3 — Graceful Shutdown
A `SIGINT` signal was sent to the supervisor. The supervisor sent termination requests to the managed processes and waited for their orderly termination.

### Test 4 — Downstream Process Survival
The process IDs were recorded before and after `INGESTION` recovery.

| Process | Before Recovery | After Recovery |
| --- | --- | --- |
| **INGESTION** | Original PID | New PID |
| **FUSION** | `1980` | `1980` |
| **NETWORK** | `1981` | `1981` |

The unchanged `FUSION` and `NETWORK` PIDs demonstrate that the downstream process instances were preserved during `INGESTION` recovery.

### Architectural Comparison

| Parameter | Full Cascade Restart | POSIX-OTP |
| --- | --- | --- |
| **Failed root process replaced** | Yes | Yes |
| **Downstream processes recreated** | Yes | No |
| **Number of processes recreated** | 3 | 1 observed |
| **Downstream process continuity** | Not preserved | Preserved |
| **Existing IPC topology** | Reconstructed | Retained |

> **Note:** This is an architectural comparison, not a measured performance benchmark.

---

## 🖥️ Hardware and Software Requirements

### Hardware

| Component | Purpose |
| --- | --- |
| **ESP32 Development Board** | Sensor data generation |
| **DHT11** | Temperature and humidity sensing |
| **LDR** | Ambient light measurement |
| **PIR Sensor** | Motion detection |
| **SSD1306 OLED** | Sensor information display |
| **USB Connection** | ESP32-to-Linux communication |

### Software and Technologies

| Technology | Application |
| --- | --- |
| **C** | System implementation |
| **Ubuntu Linux** | Host operating system |
| **POSIX APIs** | Process management and IPC |
| `fork()` / `exec()` | Process creation and execution |
| `waitpid()` | Child-process monitoring |
| **POSIX Pipes** | Inter-process communication |
| `dup2()` | File descriptor redirection |
| **Signals** | Process control and fault injection |
| **Heartbeats** | Process liveness monitoring |
| **Linux `/proc`** | Resource observability |

---

## 📈 Future Improvements

Possible future improvements include:

- Expanded fault-injection testing across all pipeline stages.
- Quantitative recovery-time and resource-overhead measurements.
- More extensive external-interface failure testing.
- Improved dependency configuration and validation.
- Testing with additional Embedded Linux hardware platforms.
- Extended recovery verification under different failure conditions.
- Evaluation with larger multi-process sensor pipelines.

---

## 📄 Patent Publication

This work is associated with a published Indian patent application.

**Title:** System and Method for Root-Confined Process Recovery Using Supervisor-Retained Communication Endpoints

| Detail | Information |
| --- | --- |
| **Application Number** | 202641126952 A |
| **Applicant** | Vellore Institute of Technology |
| **Publication Date** | 30 October 2026 |
| **Patent Journal** | No. 44/2026 |
| **Inventors** | Konguvel Elango, Vidhya S., Shashwat Kansal |
| **Status** | Published Indian Patent Application |
