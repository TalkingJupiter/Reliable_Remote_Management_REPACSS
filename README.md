# Reliable Remote Management of REPACSS Cluster (Telemetry v0.1.0)

This project implements a **rack-level telemetry subsystem** using **two ESP32 controllers per rack**:
- **Controller A (Primary)**: sends telemetry during normal operation
- **Controller B (Standby)**: monitors A via heartbeat and **takes over telemetry sending** if A fails

Telemetry is delivered to the Radxa cluster over **wired Ethernet (W5500)** as a **UDP JSON payload**.

---

## Requirements: Heartbeat

### Purpose
The heartbeat mechanism ensures fault tolerance by continuously monitoring the health of Controller A and enabling Controller B to take over telemetry transmission when A is no longer healthy.

### Requirements
- Controller A shall transmit a heartbeat signal at a fixed interval of **500 ms**.
- Controller B shall continuously monitor the heartbeat signal from Controller A.
- A heartbeat failure shall be declared when **no heartbeat is received within 2000 ms**.
- Upon detecting heartbeat failure, Controller B shall assume responsibility for telemetry transmission without manual intervention.
- Heartbeat failure and failover state shall be reported as part of the telemetry payload.

---

## Requirements: Network Communication

### Purpose
This version of the project uses **Ethernet only** for deterministic, reliable telemetry transport.

### Requirements
- The system shall use a **W5500 Ethernet module** for network communication.
- Telemetry shall be transmitted to the Radxa cluster using **UDP**.
- All ESP32 devices shall transmit to a **single Radxa IP address**.
- Each telemetry message shall include a unique device identifier (**MAC address**).

---

## Requirements: Telemetry Payload

### Purpose
The telemetry payload provides a single structured message that Radxa can ingest and store.

### Requirements
Each telemetry message shall include:
- `message_type` set to `"telemetry"`
- `device.mac`
- `timestamp_device_ms`
- `items[]` containing heartbeat, sensors, and failover event data

---

## Failover Telemetry Rule (Critical)

### Requirements
- Controller A shall transmit telemetry while it is operational.
- Controller B shall not transmit telemetry while Controller A is healthy.
- Controller B shall begin transmitting telemetry only after Controller A failure.
- Controller B shall stop transmitting when Controller A recovers.

---

## Data Structure

```json
{
  "message_type": "telemetry",
  "device": { "mac": "AA:BB:CC:DD:EE:FF" },
  "timestamp_device_ms": 0,
  "items": [
    { "kind": "heartbeat", "controller_a_alive": true, "controller_b_alive": true },
    {
      "kind": "sensors",
      "buses": [
        { "bus": "cool", "temperatures_c": [0.0, 0.0, 0.0] },
        { "bus": "exhaust", "temperatures_c": [0.0, 0.0, 0.0] }
      ]
    },
    { "kind": "event", "type": "failover", "occurred": false, "details": "" }
  ]
}
```

---

## Not in This Version
- No relay control
- No SPDT switch control
- No sensor-bus switching


### Requirements: Sensor Bus Configuration
Purpose: The sensor bus architecture provides redundant access to environmental sensors by implementing two independent buses (front and back) while ensuring electrical isolation and controlled selection using analog SPDT switches.

- The system shall implement two sensor buses, designated as the front bus and the back bus.

- Both sensor buses shall share the same ESP32 GPIO pins, with bus selection performed through SPDT switches.

- Each sensor bus shall be connected to the ESP32 through a dedicated SPDT switch to ensure electrical isolation.

- Only one sensor bus shall be connected to the ESP32 at any time, preventing bus contention and undefined electrical states.

- SPDT switch control shall be managed by system logic and coordinated with heartbeat and failover mechanisms.

- Upon failover, the active SPDT switch shall select the appropriate sensor bus without requiring firmware pin reconfiguration.

- A failure or short on one sensor bus shall not electrically affect the other bus due to SPDT-based isolation.

- The active sensor bus state shall be observable and reportable for monitoring and diagnostics.

### Requirements: Network Communication
Purpose: The network communication subsystem ensures reliable data delivery by prioritizing wired Ethernet connectivity while providing automatic wireless failover to maintain system availability during network faults.

- The system shall use Ethernet as the primary communication protocol for all normal data transmission and control traffic.

- The system shall support Wi-Fi as a secondary communication path used only when Ethernet connectivity is unavailable or degraded.

- Ethernet link status shall be continuously monitored to detect loss of connectivity or failure conditions.

- Upon detection of an Ethernet failure, the system shall automatically switch communication to Wi-Fi without requiring manual intervention.

- Failover from Ethernet to Wi-Fi shall occur within a bounded and predictable time window to minimize data loss.

- When Ethernet connectivity is restored and verified as stable, the system shall transition back to Ethernet as the primary communication channel.

- Network failover events shall be logged and reported to the monitoring or alerting system for diagnostics and maintenance awareness.

- The failover mechanism shall operate independently of the heartbeat and relay logic but remain coordinated to preserve overall system consistency.

### Requirements: Rack Identification
Purpose: Rack identity shall be assigned and managed centrally by the Radxa cluster to support scalable deployment without requiring firmware changes. The system shall also detect and safely handle telemetry received from unregistered (unknown) ESP32 devices.

- The ESP32 firmware shall transmit telemetry containing a unique device identifier (e.g., MAC address) with every message.

- The Radxa cluster shall maintain a configuration mapping of MAC address → rack_id (and optionally role, location, and metadata).

- The Radxa cluster shall be the source of truth for rack identification and labeling.

- Upon receiving telemetry from a MAC address not present in the Radxa mapping, the system shall:

    - Mark the data as unassigned/unknown (e.g., rack_id = "unknown" or unassigned=true).

    - Store the unknown MAC address and associated metadata (first-seen time, last-seen time, message count).

    - Continue ingesting the data without dropping it, while ensuring it is clearly labeled as unassigned.

**Alerting Policy for Unknown MAC Addresses**

- Unknown MAC address detection shall generate a warning severity event (not an error).

- The system shall send a summarized notification to the administrator once every 24 hours if one or more unknown MAC addresses were observed during that period.

- The daily warning notification shall include, at minimum:

    - List of unknown MAC addresses

    - First-seen and last-seen timestamps (per MAC)

    - Count of messages received (per MAC)

    - Any available network/source details (e.g., IP, interface: Ethernet/Wi-Fi)

**Non-Functional Requirement**

- The unknown-device warning mechanism shall be rate-limited to at most one notification per 24-hour window, to prevent alert fatigue while ensuring visibility for maintenance.

---

# Radxa Cluster -  STILL IN DEVELOPMENT

## Purpose
The Radxa cluster shall act as the **central ingestion, identification, storage, and alerting authority** for the rack-level telemetry system. It shall provide scalable rack identification, reliable **long-term storage of environmental telemetry**, operational event tracking, and controlled administrator notifications without requiring firmware changes when the system scales.

---

## Requirements: Database

### Purpose
A centralized SQL database shall be used to persist **environmental telemetry, system configuration, operational state, and alert metadata** required for reliable long-term monitoring and visualization through Grafana.

### Requirements
- The Radxa cluster shall maintain a **PostgreSQL-based database** for:
  - **Environmental telemetry storage** (temperature and humidity time-series at 5-second resolution)
  - **Device-to-rack identity mapping** (MAC → rack_id)
  - **Device status tracking** (last seen timestamp, last network interface, last known IP when available)
  - **Unknown device registry** (first seen, last seen, message counts)
  - **System event history** (failover, relay switching, aisle switching, network failover)
  - **Alert history and rate-limiting state** (e.g., unknown-device summary once per 24 hours)

- The database shall be the **source of truth** for rack identification and telemetry persistence.
- The database shall allow updates to rack mappings without requiring firmware modifications.
- Database write operations shall not block telemetry ingestion; ingestion shall continue in a fail-safe manner if the database is temporarily unavailable.
- The database shall be compatible with **Grafana SQL data sources** for dashboarding and alerting.

### Recommended Implementation
- PostgreSQL shall be used as the primary database.
- Use of **TimescaleDB extensions** is recommended to optimize time-series storage, indexing, compression, and retention management.

---

## Requirements: Telemetry Ingestion

- The Radxa cluster shall ingest telemetry messages from ESP32 devices using a **minimal required schema**.
- Each message shall include a **device identifier (MAC address)** and a **message type**.
- The Radxa cluster shall timestamp all messages at ingestion time.
- Telemetry ingestion shall continue even if the sending device is unknown or not mapped to a rack.
- **Temperature and humidity telemetry shall be persisted to the database as the primary system output.**

---

## Requirements: Telemetry Sampling Rate

- Environmental telemetry shall be reported at a fixed interval of **5 seconds** per rack.
- The database schema and indexing strategy shall support sustained ingestion at this rate without data loss.
- Long-term retention of telemetry data shall be supported (minimum **1 year**).

---

## Requirements: Rack Identification

- The Radxa cluster shall maintain a **MAC → rack_id** mapping in the database.
- Rack identity shall be resolved and assigned during telemetry ingestion.
- Telemetry received from unmapped devices shall be labeled as `rack_id = unknown`.
- Scaling the system shall require **Radxa-side database updates only**.

---

## Requirements: Heartbeat Awareness

- The Radxa cluster shall ingest periodic heartbeat messages.
- Device liveness shall be determined using a **last-seen timestamp**, not by storing every heartbeat as a time-series.
- Missing heartbeat beyond a configured timeout shall be interpreted as a device failure.
- Heartbeat failures and failover-related events shall be recorded as system events.
- Heartbeat failures shall trigger administrator notifications.

---

## Requirements: Unknown Device Detection

- The Radxa cluster shall detect telemetry from MAC addresses not present in the MAC → rack_id mapping.
- Unknown devices shall be labeled as `rack_id = unknown` and marked as unassigned.
- Unknown device metadata shall be stored in the database, including:
  - First-seen timestamp
  - Last-seen timestamp
  - Message count (rolling 24-hour window)
  - Network metadata when available (IP address and interface)
- Unknown-device detection shall generate a **warning-level condition** (not an error).
- A summarized unknown-device notification shall be sent **at most once every 24 hours**.
- The 24-hour rate limit shall be enforced using database state.

---

## Data Structure (What Radxa Receives)

### Required Telemetry Envelope
```json
{
  "schema_version": "1.0",
  "message_type": "heartbeat|sensors|event",
  "device": { "mac": "AA:BB:CC:DD:EE:FF" },
  "timestamp_device_ms": 0
}
```

---

### Heartbeat Message
```json
{
  "message_type": "heartbeat",
  "device": { "mac": "AA:BB:CC:DD:EE:FF" },
  "heartbeat": { "alive": true },
  "timestamp_device_ms": 0
}
```

---

### Sensor Readings Message
```json
{
  "message_type": "sensors",
  "device": { "mac": "AA:BB:CC:DD:EE:FF" },
  "sensors": [
    { "aisle": "cool|exhaust", "temperature_c": 0.0, "humidity_rh": 0.0 }
  ],
  "timestamp_device_ms": 0
}
```

---

### Event Message
```json
{
  "message_type": "event",
  "device": { "mac": "AA:BB:CC:DD:EE:FF" },
  "event": {
    "type": "failover|relay_switch|aisle_switch|network_failover",
    "value": "string"
  },
  "timestamp_device_ms": 0
}
```

---

## Radxa Database Schema (PostgreSQL)

### device_map
- mac (PRIMARY KEY)
- rack_id
- expected_role
- notes

### device_status
- mac (PRIMARY KEY)
- last_seen_ts
- last_ip
- last_interface
- last_rack_id_resolved

### telemetry_samples
- ts
- mac
- rack_id
- unassigned
- aisle
- temperature_c
- humidity_rh

### unknown_devices
- mac (PRIMARY KEY)
- first_seen_ts
- last_seen_ts
- message_count_24h

### events
- id
- ts
- mac
- rack_id
- type
- value

---

## Storage Provisioning
- The system shall provision **around 1 TB of persistent storage** to support multi-year retention of 5-second telemetry data and future system scaling.

