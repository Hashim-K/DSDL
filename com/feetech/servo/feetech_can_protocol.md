# Feetech CAN Protocol Reference

This document describes the CAN commands used by the Feetech Instruction GUI and the corresponding protocol for sending and receiving messages to Feetech servos over DroneCAN (UAVCAN v0).

## Message Types

All commands use the following message types from DSDL:

- `com.feetech.servo.Instruction` (ID: 20110) — Command messages (send)
- `com.feetech.servo.Config` (ID: 20111) — Configuration feedback (receive)
- `com.feetech.servo.Status` (ID: 20112) — Status feedback (receive)
- `com.feetech.servo.Debug` (ID: 20113) — Debug feedback (receive)

## Instruction Message (Send)

### Common Fields

- `actuator_id`: Target servo (0-255)
- `message_type`: Command type (see below)
- `data`: 2 bytes, command-specific payload

### Message Types

| Name                | Value | Description                   |
| ------------------- | ----- | ----------------------------- |
| ARM_DISARM          | 0     | Arm/disarm servo              |
| SET_TARGET_ANGLE    | 1     | Set target wing angle         |
| SET_SERIAL_FREQ     | 2     | Set serial feedback frequency |
| SET_CAN_FREQ        | 3     | Set CAN feedback frequency    |
| SET_LOG_LEVEL       | 4     | Set log verbosity             |
| SET_TARGET_POSITION | 5     | Set target servo position     |

### Data Encoding

- **ARM_DISARM:**
  - `data[0] = 1` (Arm), `data[0] = 0` (Disarm)
  - `data[1] = 0`
- **SET_TARGET_ANGLE:**
  - Two-byte signed little-endian int16 (two's complement) in centidegrees: `data[0] = LSB`, `data[1] = MSB`.
- **SET_SERIAL_FREQ / SET_CAN_FREQ:**
  - `data[0] = LSB`, `data[1] = MSB` of frequency in Hz (unsigned 16-bit)
- **SET_LOG_LEVEL:**
  - `data[0] = 1` (Long), `data[0] = 0` (Short)
  - `data[1] = 0`
- **SET_TARGET_POSITION:**
  - Two-byte signed little-endian int16 (two's complement) raw servo position: `data[0] = LSB`, `data[1] = MSB`.

### Examples

- **Arm:** `message_type=0, data=[1, 0]`
- **Disarm:** `message_type=0, data=[0, 0]`
- **Set Target Angle to +90.00° (9000 cdg):** `message_type=1, data=[0x28, 0x23]`
- **Set Target Angle to -15.00° (-1500 cdg):** `message_type=1, data=[0x24, 0xFA]` (two's complement of 0x05DC)
- **Set Serial Freq to 50Hz:** `message_type=2, data=[50, 0]`
- **Set CAN Freq to 10Hz:** `message_type=3, data=[10, 0]`
- **Set Log Level to Long:** `message_type=4, data=[1, 0]`
- **Set Log Level to Short:** `message_type=4, data=[0, 0]`
- **Set Target Position to +1024 (raw):** `message_type=5, data=[0x00, 0x04]`
- **Set Target Position to -1024 (raw):** `message_type=5, data=[0x00, 0xFC]` (two's complement of 0x0400)

---

## Feedback Messages (Receive)

### Config Message (`com.feetech.servo.Config`, ID: 20111)

- Contains current configuration:
  - `actuator_id`: Target servo
  - `gear_ratio`: uint8
  - `min_angle`: int16
  - `max_angle`: int16
  - `serial_frequency`,: float32 (Hz)
  - `can_frequency`: float32 (Hz)
  - `max_speed`: uint16

### Status Message (`com.feetech.servo.Status`, ID: 20112)

- Contains current status:
  - `actuator_id`: Target servo
  - `current_angle`: int16 (current wing angle, implementation uses centidegrees)

### Debug Message (`com.feetech.servo.Debug`, ID: 20113)

- Contains extended debug info:
  - `actuator_id` (uint8): Target servo ID
  - `armed` (bool): Servo armed state
  - `calculation_offset` (int16): Offset for angle calculation
  - `target_wing_angle` (int16, cdg): Target wing angle
  - `current_angle` (int16, cdg): Current wing angle
  - `revolution_count` (int32): Revolution count for multi-turn tracking
  - `target_servo_position` (int16): Target servo position (raw steps)
  - `current_position_global` (int32): Current global position
  - `current_position` (int16): Current local servo position
  - `previous_servo_position` (int16): Previous local servo position
  - `current_speed` (int16): Current servo speed
  - `latest_load` (int16): Current load reading
  - `voltage` (uint8): Supply voltage
  - `temp` (uint8): Temperature
  - `errcode` (uint8): Servo error code
  - `health_state` (uint16): Health state counter

---

## How to Read Target Angle

- Listen for `Status` or `Debug` messages from the servo node.
- `Status.current_angle` gives the current wing angle.
- `Debug.target_wing_angle` gives the target wing angle.
- No explicit read command; enable feedback by setting serial/CAN frequency.

---

## Summary Table (Instruction Message)

| Command             | message_type | data[0]         | data[1]   | Description                               |
| ------------------- | ------------ | --------------- | --------- | ----------------------------------------- |
| Arm                 | 0            | 1               | 0         | Enable servo                              |
| Disarm              | 0            | 0               | 0         | Disable servo                             |
| Set Target Angle    | 1            | angle LSB       | angle MSB | Signed int16 (two's complement), cdg      |
| Set Serial Freq     | 2            | freq LSB        | freq MSB  | Serial report freq (Hz, uint16)           |
| Set CAN Freq        | 3            | freq LSB        | freq MSB  | CAN report freq (Hz, uint16)              |
| Set Log Level       | 4            | 0=Short, 1=Long | 0         | Log verbosity                             |
| Set Target Position | 5            | pos LSB         | pos MSB   | Signed int16 (two's complement), raw step |

---

## Notes

- Multi-byte values are little-endian. Angles and positions on UAVCAN use standard C int formats (two's complement int16). The firmware converts these to the Feetech serial format internally when talking to the servo.
- Refer to the DSDL definitions:
  - `firmware/lib/DSDL/com/feetech/servo/20110.Instruction.uavcan`
  - `firmware/lib/DSDL/com/feetech/servo/20111.Config.uavcan`
  - `firmware/lib/DSDL/com/feetech/servo/20112.Status.uavcan`
  - `firmware/lib/DSDL/com/feetech/servo/20113.Debug.uavcan`
