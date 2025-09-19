# RFC: Generic Payload Protocol for MAVLink

- Start Date: 2025-09-19
- Contributors: Thomas Watters (Skydio)
- Related Issues: N/A

## Summary

This RFC proposes a standardized Generic Payload Protocol for MAVLink that provides a flexible interface for controlling and monitoring arbitrary payloads attached to UAVs. The protocol supports various data types, control modes, and telemetry streaming through a set of five new MAVLink messages.

## Motivation

Current MAVLink lacks a standardized way to interface with generic payloads that don't fit into existing message categories. While MAVLink does provide some [payload services](https://mavlink.io/en/services/payload.html), these have significant limitations for generic payload integration:

### Limitations of Existing MAVLink Payload Interface

1. **Payload-Specific Design**: Current commands are tailored to specific payload types (grippers, winches, illuminators) rather than providing a generic framework. The documentation explicitly recommends "use the commands for known payload types where possible as they are more intuitive for users."

2. **Limited Generic Commands**: The few generic commands available (`MAV_CMD_DO_SET_ACTUATOR`, `MAV_CMD_DO_SET_SERVO`) are:

   - Rudimentary in functionality (basic PWM/position control only)
   - Inconsistently implemented ("only implemented on PX4" for actuator commands)
   - Lack metadata and discovery mechanisms
   - Provide no type safety or validation

3. **No Discovery Protocol**: Existing interfaces require prior knowledge of payload capabilities - there's no way for a GCS to discover what functions a payload provides or their acceptable value ranges.

4. **Missing Telemetry Framework**: No standardized way for payloads to stream sensor data or status information back to the GCS.

5. **Integration Complexity**: Each new payload type requires either:
   - Creating custom MAVLink messages (requiring message ID allocation and standardization)
   - Using inadequate generic commands that lack proper discovery and metadata
   - Implementing proprietary protocols outside MAVLink

### Benefits of a Comprehensive Generic Payload Protocol

A standardized generic payload protocol would address these limitations by providing:

- **Self-Describing Capabilities**: Payloads announce their functions and telemetry channels with metadata
- **Type-Safe Operations**: Strong typing prevents value interpretation errors
- **Plug-and-Play Discovery**: Zero-configuration payload detection and capability enumeration
- **Consistent UI/UX**: Standardized interface across different GCS implementations
- **Extensible Framework**: Support for future payload types without protocol changes
- **Reduced Integration Time**: Manufacturers can implement once and work with any compliant GCS
- **MAVLink Ecosystem Compatibility**: Maintains compatibility with existing infrastructure

## Detailed Design

### Protocol Architecture

The protocol consists of three main components:

1. **Status Reporting**: Periodic broadcast of payload presence and capabilities
2. **Function Control**: Command/response interface for payload functions
3. **Telemetry Monitoring**: Streaming of payload sensor data

### Message Definitions

#### Core Messages:

- `GENERIC_PAYLOAD_STATUS` (60000): Discovery and status broadcast
- `GENERIC_PAYLOAD_FUNCTION_STATUS` (60001): Function metadata and state
- `GENERIC_PAYLOAD_FUNCTION_CONTROL` (60002): Function command interface
- `GENERIC_PAYLOAD_TELEMETRY_STATUS` (60003): Telemetry channel metadata
- `GENERIC_PAYLOAD_TELEMETRY_DATA` (60004): High-rate telemetry streaming

#### Key Features:

- **Type Safety**: Strongly typed value system supporting 6 data types (INT32/UINT32/REAL32/INT64/UINT64/REAL64)
- **Extensibility**: Extension fields for additional metadata without breaking compatibility
- **Discovery**: Self-describing payloads that announce their capabilities
- **Flexibility**: Supports logical, continuous, and discrete function types
- **Control Modes**: Latching and momentary control with configurable timeouts

### Protocol Operation

1. **Discovery Phase**: Payload broadcasts `GENERIC_PAYLOAD_STATUS` at 1Hz
2. **Capability Query**: Vehicle requests function/telemetry metadata via `MAV_CMD_REQUEST_MESSAGE`
3. **Operation**: Vehicle sends control commands, payload streams telemetry and status updates

### Enumerations

#### Control Modes

- `GENERIC_PAYLOAD_CONTROL_MODE`: Single-value enumeration for selecting control mode
  - `LATCHING` (1): Function maintains state until changed
  - `MOMENTARY` (2): Function auto-deactivates after timeout
- `GENERIC_PAYLOAD_CONTROL_MODE_FLAGS`: Bitmask for supported control modes
  - `SUPPORTS_LATCHING` (1): Function supports latching mode
  - `SUPPORTS_MOMENTARY` (2): Function supports momentary mode

#### Value Types

Support for 6 data types covering most payload needs:

- 32-bit: INT32, UINT32, REAL32
- 64-bit: INT64, UINT64, REAL64

#### Function Types

- `LOGICAL`: Boolean controls (on/off, enable/disable)
- `CONTINUOUS`: Analog controls with min/max ranges (servo angles, speeds)
- `DISCRETE`: Digital controls with enumerated values (modes, frequencies)

### Message ID Allocation

Proposed message IDs in the 60000-60004 range (vendor-specific range) to avoid conflicts during development. Final standardization would require official MAVLink message ID allocation.

### Extension Fields

All messages include extension fields for future enhancements:

- Power consumption monitoring
- Temperature sensing
- Mass/inertia properties
- 64-bit value support

## Alternatives

### Alternative 1: Extend Existing Payload Commands

Build upon the current [MAVLink payload services](https://mavlink.io/en/services/payload.html) by adding more generic commands and standardizing implementation.

**Analysis**: The existing `MAV_CMD_DO_SET_ACTUATOR` and `MAV_CMD_DO_SET_SERVO` commands could theoretically be extended with additional parameters for type information and metadata.

**Key Limitations**:

- Commands are stateless and don't provide discovery mechanisms
- Inconsistent implementation across flight stacks ("only implemented on PX4")
- Limited parameter space in command messages prevents rich metadata
- No built-in telemetry streaming capabilities
- Would require significant breaking changes to existing command semantics

### Alternative 2: Parameter-Only Approach

Use MAVLink's parameter system (`PARAM_*` messages) to expose payload functions as parameters.

**Analysis**: Parameters provide get/set functionality and some metadata (min/max, type). Could potentially expose payload functions as settable parameters.

**Key Limitations**:

- Parameters are designed for configuration, not real-time control
- No standardized discovery of which parameters belong to payloads
- Limited data types (mostly float32)
- No support for momentary operations or complex control modes
- Parameter namespace pollution with payload-specific entries
- No built-in telemetry streaming for sensor data

### Alternative 3: Extend Component Information Service

Build upon the [MAVLink Component Information Service](https://mavlink.io/en/services/component_information.html) to include payload metadata and add control messages.

**Analysis**: The component information service provides sophisticated metadata discovery through JSON files and supports actuator configuration. This could potentially be extended to describe payload capabilities.

**Key Limitations**:

- Currently "work in progress" with limited adoption
- Designed for static configuration metadata, not dynamic control
- Requires external file hosting or on-device storage for metadata
- JSON parsing overhead for simple operations
- No real-time control or telemetry streaming mechanisms
- Metadata assumed invariant after boot, limiting dynamic payloads

## Unresolved Questions

1. **Message ID Range**: Should we request official MAVLink message IDs or remain in vendor-specific range?
2. **Backward Compatibility**: How should payloads indicate protocol version support?
3. **Performance Limits**: What are reasonable limits for number of functions/telemetry channels per payload?
4. **Rate Control**: How should high-rate telemetry interact with existing MAVLink rate limiting mechanisms?
5. **Error Recovery**: Should we define standard error recovery procedures for failed commands?

## Implementation Considerations

### Bandwidth Management

- Status messages at low rate (1Hz) for discovery
- Telemetry data messages can be rate-controlled per channel
- Function status only sent on state changes or when requested

### Payload Discovery

- Payloads self-announce via periodic status broadcasts
- Vehicle can request detailed capability information on-demand
- Supports hot-plug scenarios where payloads are connected during flight

### Multi-Payload Systems

- Each payload uses unique component ID
- Messages include payload-specific addressing
- Supports heterogeneous payload mixes on single vehicle

### Validation and Safety

- Strong typing prevents value interpretation errors
- Range checking via min/max fields
- Error flags provide diagnostic information
- Timeout mechanisms prevent stuck states

## Appendix: Complete Message Definitions

### Enumerations

#### GENERIC_PAYLOAD_CONTROL_MODE

Control mode for generic payload functions (enumeration - mutually exclusive values).

| Value | Name                                   | Description                                                                                                                                            |
| ----- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1     | GENERIC_PAYLOAD_CONTROL_MODE_LATCHING  | Function maintains its state until explicitly changed (toggle on/off).                                                                                 |
| 2     | GENERIC_PAYLOAD_CONTROL_MODE_MOMENTARY | Function is active when commanded and automatically deactivates after timeout_ms milliseconds. If timeout_ms is 0, a default timeout of 100ms is used. |

#### GENERIC_PAYLOAD_CONTROL_MODE_FLAGS

Bitmask of supported control modes for generic payload functions.

| Value | Name                               | Description                               |
| ----- | ---------------------------------- | ----------------------------------------- |
| 1     | GENERIC_PAYLOAD_SUPPORTS_LATCHING  | Function supports latching control mode.  |
| 2     | GENERIC_PAYLOAD_SUPPORTS_MOMENTARY | Function supports momentary control mode. |

#### GENERIC_PAYLOAD_VALUE_TYPE

Data type for generic payload function values.

| Value | Name                              | Description                     |
| ----- | --------------------------------- | ------------------------------- |
| 0     | GENERIC_PAYLOAD_VALUE_TYPE_INT32  | 32-bit signed integer.          |
| 1     | GENERIC_PAYLOAD_VALUE_TYPE_UINT32 | 32-bit unsigned integer.        |
| 2     | GENERIC_PAYLOAD_VALUE_TYPE_REAL32 | 32-bit IEEE-754 floating point. |
| 3     | GENERIC_PAYLOAD_VALUE_TYPE_INT64  | 64-bit signed integer.          |
| 4     | GENERIC_PAYLOAD_VALUE_TYPE_UINT64 | 64-bit unsigned integer.        |
| 5     | GENERIC_PAYLOAD_VALUE_TYPE_REAL64 | 64-bit IEEE-754 floating point. |

#### GENERIC_PAYLOAD_FUNCTION_TYPE

Type of generic payload function.

| Value | Name                                     | Description                                                                |
| ----- | ---------------------------------------- | -------------------------------------------------------------------------- |
| 0     | GENERIC_PAYLOAD_FUNCTION_TYPE_LOGICAL    | Logical state control (true/false, on/off, high/low).                      |
| 1     | GENERIC_PAYLOAD_FUNCTION_TYPE_CONTINUOUS | Continuous value control (e.g., servo angle, motor speed). Uses min/max.   |
| 2     | GENERIC_PAYLOAD_FUNCTION_TYPE_DISCRETE   | Discrete value selection (e.g., mode number, PWM frequency). Uses min/max. |

#### GENERIC_PAYLOAD_ERROR_FLAGS

Error flags for a generic payload. This is a bitmask enumeration.

| Value | Name                                       | Description                                                                                                              |
| ----- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| 1     | GENERIC_PAYLOAD_ERROR_FLAGS_SOFTWARE_ERROR | There is an error with the generic payload's software.                                                                   |
| 2     | GENERIC_PAYLOAD_ERROR_FLAGS_HARDWARE_ERROR | There is an error with the generic payload's hardware.                                                                   |
| 4     | GENERIC_PAYLOAD_ERROR_FLAGS_OVERTEMP_FAULT | A specified limit has been reached by the generic payload's temperature sensor. Applies if payload monitors temperature. |
| 8     | GENERIC_PAYLOAD_ERROR_FLAGS_CUSTOM         | Generic payload custom failure. See custom error flag bitmask for details.                                               |

### Messages

#### GENERIC_PAYLOAD_STATUS (Message ID: 60000)

Status and discovery message for a generic payload. This should be broadcast at a low rate (nominally 1 Hz) to announce the payload's presence and capabilities.

| Field Name             | Type     | Description                                                          |
| ---------------------- | -------- | -------------------------------------------------------------------- |
| uptime_ms              | uint32_t | Time since payload boot (in milliseconds).                           |
| error_flags            | uint32_t | Current error flags (bitmask). See GENERIC_PAYLOAD_ERROR_FLAGS.      |
| custom_error_flags     | uint32_t | Bitmap for custom error flags specific to this payload type.         |
| num_functions          | uint16_t | Number of functions this payload provides.                           |
| num_telemetry_channels | uint16_t | Number of telemetry channels this payload provides.                  |
| name                   | char[32] | Name of the generic payload for UI display (NULL-terminated string). |

**Extensions:**

| Field Name  | Type        | Description                                                                          |
| ----------- | ----------- | ------------------------------------------------------------------------------------ |
| power_draw  | uint16_t    | Power consumption of the payload in mW. 0 if not available.                          |
| temperature | uint16_t    | Temperature of the payload (0.01 °C per unit, 0 = 0°C). UINT16_MAX if not available. |
| mass        | uint16_t    | Mass of the payload in grams. 0 if not available.                                    |
| torque_arm  | uint16_t[3] | Torque arm in Payload Frame (mm)                                                     |

#### GENERIC_PAYLOAD_FUNCTION_STATUS (Message ID: 60001)

Generic payload function status.

| Field Name    | Type       | Description                                                                      |
| ------------- | ---------- | -------------------------------------------------------------------------------- |
| index         | uint16_t   | Index of this function on the generic payload.                                   |
| type          | uint8_t    | Type of function. See GENERIC_PAYLOAD_FUNCTION_TYPE.                             |
| value_type    | uint8_t    | Data type for value/min/max fields. See GENERIC_PAYLOAD_VALUE_TYPE.              |
| enabled       | uint8_t    | 0: Disabled, 1: Enabled                                                          |
| value_low     | uint8_t[4] | Lower 32 bits of current value. For 32-bit types, this is the entire value.      |
| min_low       | uint8_t[4] | Lower 32 bits of minimum value (for CONTINUOUS/DISCRETE types).                  |
| max_low       | uint8_t[4] | Lower 32 bits of maximum value (for CONTINUOUS/DISCRETE types).                  |
| control_modes | uint16_t   | Bitmask of supported control modes. See GENERIC_PAYLOAD_CONTROL_MODE_FLAGS.      |
| timeout_ms    | uint32_t   | Timeout in milliseconds for MOMENTARY mode. 0 means use default timeout (100ms). |
| name          | char[32]   | Name of the function for UI display. NULL-terminated string.                     |
| units         | char[16]   | Units for the value field (optional). NULL-terminated string.                    |

**Extensions:**

| Field Name | Type       | Description                                                 |
| ---------- | ---------- | ----------------------------------------------------------- |
| value_high | uint8_t[4] | Upper 32 bits of current value. Only used for 64-bit types. |
| min_high   | uint8_t[4] | Upper 32 bits of minimum value. Only used for 64-bit types. |
| max_high   | uint8_t[4] | Upper 32 bits of maximum value. Only used for 64-bit types. |

#### GENERIC_PAYLOAD_FUNCTION_CONTROL (Message ID: 60002)

Command to control a generic payload function.

| Field Name   | Type       | Description                                                                                |
| ------------ | ---------- | ------------------------------------------------------------------------------------------ |
| index        | uint16_t   | Index of this function on the generic payload.                                             |
| value_type   | uint8_t    | Data type for value field. See GENERIC_PAYLOAD_VALUE_TYPE.                                 |
| control_mode | uint8_t    | Desired Control Mode. See GENERIC_PAYLOAD_CONTROL_MODE.                                    |
| enable       | uint8_t    | 0: Disable, 1: Enable                                                                      |
| value_low    | uint8_t[4] | Lower 32 bits of value to set. For 32-bit types, this is the entire value.                 |
| timeout_ms   | uint32_t   | Timeout for momentary operation if control_mode is GENERIC_PAYLOAD_CONTROL_MODE_MOMENTARY. |

**Extensions:**

| Field Name | Type       | Description                                                |
| ---------- | ---------- | ---------------------------------------------------------- |
| value_high | uint8_t[4] | Upper 32 bits of value to set. Only used for 64-bit types. |

#### GENERIC_PAYLOAD_TELEMETRY_STATUS (Message ID: 60003)

Generic payload telemetry channel status.

| Field Name  | Type       | Description                                                                 |
| ----------- | ---------- | --------------------------------------------------------------------------- |
| index       | uint16_t   | Index of this telemetry channel on the generic payload.                     |
| value_type  | uint8_t    | Data type for value/min/max fields. See GENERIC_PAYLOAD_VALUE_TYPE.         |
| update_rate | uint8_t    | Rate at which the value is updated (in Hz). 0 if unknown.                   |
| value_low   | uint8_t[4] | Lower 32 bits of current value. For 32-bit types, this is the entire value. |
| min_low     | uint8_t[4] | Lower 32 bits of minimum value. Used for scaling/display.                   |
| max_low     | uint8_t[4] | Lower 32 bits of maximum value. Used for scaling/display.                   |
| name        | char[32]   | Name of the telemetry channel for UI display. NULL-terminated string.       |
| units       | char[16]   | Units for the value field (optional). NULL-terminated string.               |

**Extensions:**

| Field Name | Type       | Description                                                 |
| ---------- | ---------- | ----------------------------------------------------------- |
| value_high | uint8_t[4] | Upper 32 bits of current value. Only used for 64-bit types. |
| min_high   | uint8_t[4] | Upper 32 bits of minimum value. Only used for 64-bit types. |
| max_high   | uint8_t[4] | Upper 32 bits of maximum value. Only used for 64-bit types. |

#### GENERIC_PAYLOAD_TELEMETRY_DATA (Message ID: 60004)

Lightweight telemetry data message for streaming single values.

| Field Name | Type       | Description                                                                 |
| ---------- | ---------- | --------------------------------------------------------------------------- |
| index      | uint16_t   | Index of this telemetry channel on the generic payload.                     |
| value_low  | uint8_t[4] | Lower 32 bits of current value. For 32-bit types, this is the entire value. |

**Extensions:**

| Field Name | Type       | Description                                                 |
| ---------- | ---------- | ----------------------------------------------------------- |
| value_high | uint8_t[4] | Upper 32 bits of current value. Only used for 64-bit types. |

## References

- [MAVLink Message Definition Standard](https://mavlink.io/en/guide/define_xml_element.html)
- [MAVLink Common Message Set](https://mavlink.io/en/messages/common.html)
- [Existing Payload Interfaces in MAVLink](https://mavlink.io/en/messages/common.html#camera-protocol)
- [MAVLink Extension Fields](https://mavlink.io/en/guide/message_definitions.html#extension_fields)
- [MAVLink Message Rate Control](https://mavlink.io/en/messages/common.html#MAV_CMD_SET_MESSAGE_INTERVAL)
