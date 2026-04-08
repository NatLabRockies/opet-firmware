# OPET firmware introduction
This repository contains everything relating to the firmware of the OPET device including source code.  If you are looking for the hardware or control software repositories, links are below:

[OPET Hardware](https://github.com/NREL/opet-hardware)

[OPET Control Software](https://github.com/NREL/opet-control)

# OPET firmware functions
## PV device load functions
- The main application of OPET is long term performance measurements of PV devices in field or under controlled environmental conditions
- device performance degradation can be impacted by operation point, hence different load functions have been implemented
- the load mode is controlled with the `LOAD:MODE (N)` command, see also [Load Control Commands](#load-control-commands)
    - \(N\) is the desired operation mode ID as detailed in the following sections
    - set the load mode before switching the output on after first boot-up, as the standard EEPROM load mode at boot-up is "0 - none" and the output cannot be enabled
        - see AutoStart function in [Auto-Start System](#auto-start-system) to change this behaviour
- use the `OUTP 1` or `OUTP 0` commands to enable or disable the output
    - IV curve or transient measurements cannot be performed unless the output is enabled
    - if an invalid load mode is entered/requested, the output will switch off automatically
    - if only IV curve or transient measurements are required without active load control, enable the output with the open circuit (V<sub>oc</sub>) load mode
- use the `READ?` function to collect load measurement data and with system status, see [Read status and output measurements](#read-status-and-output-measurements) for more details and data formatting

### Open circuit load
- Use `LOAD:MODE 1` to enter V<sub>oc</sub> mode
- During Open circuit (V<sub>oc</sub>) load the output of OPET is kept at high impedance, with the driver MOSFET and controller effectively being disabled as in off-mode
- The only current that can flow is the current trough the voltage sense terminals, which is dependent on the voltage reading and range
- \~0.9 mA at 100V (see impedance specification for each range in documentation)
    - This leakage current is recorded on the PV current channel

### Short Circuit load
- Use `LOAD:MODE 2` to enter I<sub>sc</sub> mode
- The Short circuit (I<sub>sc</sub>) load mode controls the voltage at the PV device 4-wire connection junctions at zero volts, essentially I<sub>sc</sub>
- this is a special case of the static voltage load mode with fixed voltage control at 0.0 V
- The bias voltage supply is important here, as its voltage output is used to compensate for the voltage losses on the PV device cables
    - Without bias voltage, OPET can be operated, but cannot reach or operate at I<sub>sc</sub>

### Static Voltage load
- Use `LOAD:MODE 3` to enter static voltage load V<sub>SET</sub> mode
- Use `LOAD:SETVOLT (voltage)` to adjust the set-point, whereas (voltage) is the required voltage in \[V\]
- In static voltage load mode, the PV device load can be set to any desired voltage setpoint
- Load set-point range is between zero volts (I<sub>sc</sub>) and open circuit (V<sub>oc</sub>)
- If the setpoint voltage is above V<sub>oc</sub>, the device load will operate at V<sub>oc</sub> until the PV device output voltage can surpass the setpoint voltage, at which time set-point is held and current can flow

### Static Current tracking
- Use `LOAD:MODE 4` to enter static current lead C<sub>SET</sub> mode
- Use `LOAD:SETCURR (current)` to adjust the set-point, whereas (current) is the required current in \[A\]
- Static current tracking mode is similar to the static voltage mode, but controlling current instead of voltage
- set-point range is between zero amps (V<sub>oc</sub>) and short circuit (I<sub>sc</sub>),
- if the setpoint is above I<sub>sc</sub> of the PV device, the PV device operates at ISC until its supply current is able to exceed the setpoint current, at which the load voltage is increasing to keep the current set-point
- since the OPET uses a voltage driven PI-control to regulate the output load, the current load mode is a software implemented load control mode
- its reaction time is therefore slower and more typical to that of the maximum power point tracking mode dependent on the control options used
- the current load mode has a configurable "no-adjust-zone" that is around the current set-point, at which the load controller set-point is not adjusted
    - can lead to better control stability with high input noise

### Maximum power point tracking
- Use `LOAD:MODE 5` to enter maximum power point tracking P<sub>mp</sub> mode
- Maximum power point P<sub>mp</sub> tracking is realised using a simple hill-climbing algorithm
- Dependent if power output is decreasing or increasing, the climbing direction is changed or kept
- The voltage step width is adjustable and variable
    - Before P<sub>mp</sub> is found or if point is lost, the step size is larger/ increasing to reach the point faster
    - when P<sub>mp</sub> is found, the voltage step size is reduced to achieve a better tracking accuracy
- if the P<sub>mp</sub> load mode is active during an IV curve measurement, the tracker is automatically set to the global P<sub>mp</sub> of the IV curve thereafter
    - useful if the tracker is stuck at the local P<sub>mp</sub> different to the overall maximum power, that is caused by shading on the PV device
- the P<sub>mp</sub> tracker also has a so called "no-adjust-zone"
    - this is a power range at which no adjustments to the tracking direction of step size are made until change in power exceeds this range, at which a new adjustment is made
    - this improves tracking accuracy at high noise level
    - causes the P<sub>mp</sub> tracker to wander around the maximum point at better accuracy instead of drifting off to any side below P<sub>mp</sub>

## IV curve measurement function
- The IV curve measurement function traces the complete IV curve in direction 0 V to V<sub>oc</sub> or in the reverse direction, dependent on the option selected
- The OPET measurement routine is as follows in order:
    - Measure V<sub>oc</sub> and set optimal voltage measurement range
    - Measure I<sub>sc</sub> and set optimal current measurement range
    - Calculate voltage control set-points based on measurement options
    - Measure IV curve
    - Reset previous set-point and range conditions
    - Process data
- Current and voltage measurement ranges are only adjusted if the OPET is in auto-range mode, see [Measurement range control](#measurement-range-control) for more details
- During the IV curve measurement process following is done for each point in order:
    - Voltage reference set-point is transferred
    - Waiting settling time
    - Repeat measure voltage and current dependent on averaging options
    - Measure voltage one last time if asymmetric voltage option enabled
- Use commands in [IV Curve Tracing Commands](#iv-curve-tracing-commands) to adjust the measurement options
- To measure an IV curve, following program flow steps should be taken:
    - make sure output is on without active errors
    - Call `IV:MEAS` to start the IV curve measurement
    - OPET replies with `IV:MEAS (XXX)`, whereas (XXX) is the estimated IV curve measurement time in ms
        - This can be used to optimise a timeout function in program flow
        - If XXX is zero, the IV curve measurement internally aborted, usually because the output is disabled
    - Wait at least 20ms (one control loop time cycle, see [Control loop timer multiplier](#control-loop-timer-multiplier))
    - Send `\*OPC?` and wait for reply or wait until IV tracing is finished
        - A response to this command is send right after the IV finishes
    - Send `IV:DATA?` to collect the latest IV data
- the distribution of voltage set-points over the IV curve can be linear with equal point distribution or cosine with more dense voltage point distribution around VOC
    - see [IV Curve Tracing Commands](#iv-curve-tracing-commands) for command details
    - the cosine phase-angle maximum can be used to tweak the position of the maximum density of points or its point density
        - density near V<sub>oc</sub> when φ is below π/2, the higher the more points around V<sub>oc</sub>
        - position can be shifted towards P<sub>mp</sub> with π \> φ \> π/2, the higher the further towards I<sub>sc</sub>
- the maximum power point is also determined during the data processing, which is returned to the P<sub>mp</sub> load mode if active

## Transient measurement function
- The transient measurement function is used to measure the driver and PV device response to a step change in set-point voltage
    - Useful for PI-controller tweaking and debugging
- This function uses the voltage setpoint control value of the V<sub>SET</sub> load mode as start point and the Transient voltage end point as target step voltage
    - `LOAD:SETVOLT (voltage)` to adjust start point
    - `TRANS:ENDVOLT (voltage)` to adjust end point
- Following program flow executes in order:
    - Set optimal measurement ranges if in auto-range
    - Set start voltage
    - Settle for \~40ms
    - Set end voltage
    - Measure voltage at given intervals
    - Set start voltage
    - Settle for \~40ms
    - Set end voltage
    - Measure current at given intervals
    - Progress data in raw format
    - Reset previous load conditions
- As above, transient is measured twice, once for voltage and once for current
    - Done do achieve an adjustable measurement resolution down to \~12 µs, comprised of 4 µs conversion time and 8 µs data transfer and processing time
    - would otherwise need \~100 µs to change the input channel on the multiplexer
- Use commands in [Transient measurement control](#transient-measurement-control) to adjust the measurement options
    - The point delay time given in \[µs\] is a good approximation, but not exact
    - Below the minimum of \~12 µs the delay time is ignored
- Data is transferred in uncalibrated raw format
    - helps analysing noise distribution at the ADC, when using same start and end voltage
- To measure a transient curve do following:
    - make sure output is on without active errors
    - set start and end voltage using `LOAD:SETVOLT (voltage)` and `TRANS:ENDVOLT (voltage)`
    - Call `TRANS:MEAS` to start the transient measurement
    - Wait at least \~20ms (one control loop time cycle, see [Control loop timer multiplier](#control-loop-timer-multiplier))
    - Send `\*OPC?` and wait for reply or wait until measurement is finished
        - A response to this command is send right after the transient measurement is finished
    - Send `IV:DATA?` to collect the latest transient data

## System status and error handling
- When using the `READ?` command to request load measurement data the system status byte is also transferred (see [Read status and output measurements](#read-status-and-output-measurements))
- The system status byte mainly details the output status, calibration mode status and error indicators
- In normal operation, the status byte should just return "1", meaning output is on without any errors
- The calibration status bit is enabled only during calibration, at which the current and voltage readings are returned in raw format without calibration applied
- The voltage and current input error bits indicate at ADC over- or underflow, meaning the ADC is above maximum or below minimum value and cannot show the actual value
    - The error enables even when one measurement of all averaged measurements is out of range
    - The output will stay on as normal even when an input error is active
- The overcurrent bypass active bit enables when the current flow to a shunt resistor is getting too high and thus protects the shunt resistor from overheating and premature aging
    - The overcurrent error is cleared automatically with the auto-range handling routine that increases the measurement range; in manual range mode it has to be cleared manually
    - If the range is still too low for the current flow, the overcurrent bypass will reactivate again after clearing/resetting
    - The output state is not affected by the overcurrent bypass protection
    - The current reading is not representative of actual current flow when bypasses is engaged and only shows what is still going through the shunt, while the majority of current is bypassed
    - see [Overcurrent bypass reset](#overcurrent-bypass-reset) for more details
- the bias voltage error indicator bit enables when the bias voltage goes out of range
    - this error will temporarily disable the output of the OPET, it will still show enabled to indicate that it will come on again, but will not measure IV curves and only set to V<sub>oc</sub> loading
    - once the bias voltage is within range for \~5 s, the output will be activated again
    - the time delayed return to normal operations reduces the stress on the system in case of recurring errors, for example where the voltage output fails above a specific current flow due to an overcurrent dropout off the power supply
    - the bias voltage range can be adjusted, see [Bias Voltage Limits Protection](#bias-voltage-limits-protection) for more details
- the NTC 1 and NTRC 2 over Temperature errors indicate when the OPET electronics are overheating or its too cold
    - if the temperature is too high or too low and reaches the disconnect threshold, the output will be disabled and at V<sub>oc</sub> loading, IV curve can also not be measured
    - for as long as the temperature is between disconnect and reconnect threshold after a disconnect, the output is off and at V<sub>oc</sub>, but IV curves and transient curves can be measured
    - once the reconnect threshold is reached after cooling down / warming up, the output is re-enabled to the previous control state
    - the temperature threshold can be adjusted, see [Operating Temperature limits configuration](#operating-temperature-limits-configuration) for more details

## Cooling fan output control
- with the `FAN:MODE` command the cooling fan control can either be controlled manually to on/off or set to automatic control mode, see [Fan Control](#fan-control) for command details
- in auto mode, the fan is controlled based on:
    - the NTC1 and NTC 2 temperature
    - the power load on the MOSFET
- the auto mode uses threshold hysteresis on/off control which basically switches the fan on if the upper threshold is exceeded and switches off if the value is less than the lower threshold
- auto mode uses switched-on priority, meaning the fan is controlled on if any one control channel is requesting the fan to be on and it is only off if all channels request the fan to be off
- the switching frequency of the fan in auto control is also limited by a timer counter that limits how fast the fan can be switched off
- thresholds can be adjusted via the EEPROM setting, see [Operating Temperature limits configuration](#operating-temperature-limits-configuration)

## PI-Controller adjustments
- The driver MOSFET PI-controller on the OPET PCB is actively regulating the voltage at the PV device
- in case that the PI-controller gain and integration are inadequate for the PV device load and capacitance, the controller can get instable and oscillate
    - at this point the current and voltage readings may get very noisy
    - also the shape of the IV curve measurements can be affected
- due to the wide variety of PV devices with different load capacitances and stray induction, output voltages and currents, it is difficult to optimise the PI-controller to work for all devices at once
- hence the PI-controller can be adjusted electronically with a selection of 4 gain resistors and 4 integrator capacitors
- in general, increasing the control values for integrator capacitors and gain resistor makes the response of the OPET slower and more stable (see also details in [PI Driver Control](#pi-driver-control))
    - increasing the integrator capacitor ID value increases the integrator capacitance, which reduces the voltage slew rate of the driver MOSFET gain voltage
    - increasing the gain resistor ID value, reduces the proportional gain response to the error amplifier on the driver MOSFET and hence the speed at which a set-point is reached
- the PI-controller stability is also dependent on the voltage range setting, hence it is advised to check the response throughout the entire PV device IV curve using the manual control mode in auto-range setting, to check the stability of the controller
    - it helps plotting the voltage control value against the PV device measured voltage to determine stability, this should be a linear relationship
    - it also helps using the transient response measurement function to determine stability, a stable response should have at max a single overshoot and should settle quickly
        - in instable conditions the transient will overshoot significantly and oscillate around the set-point until maybe settling, basically the dampening is too low
- stability should also be checked when the PV device connected is under maximum light conditions, where it has its highest V<sub>oc</sub>, I<sub>sc</sub> and output power
    - the stability should then still be fine under low irradiance conditions
- when using a high integration capacitor ID value, it is possible that the IV curve does not finish at V<sub>oc</sub>
    - this is due to the reduced response time of the PI controller, which is slowest around V<sub>oc</sub>, where the controller needs to adjust the driver MOSFET voltage more to reduce current flow
    - in this case it is best to increase the IV-point settling time so that the controller has more time to reach the set-point (see 4.3.4.4)

## Auto-Start System
- in normal configuration the OPET system needs to be controlled by a computer first to enable the output and enter a load mode
- with the auto-start function enabled, the OPET will automatically enable the output and enter the selected load mode right after power-up
    - this helps getting the system going again after a power fault without having to check & start-up the control computer
- Auto-start can be enabled via the EEPROM control values, see [Auto-Start Load configuration](#auto-start-load-configuration) for more details

# Communication & Control

## IO Protocol

### Serial communication settings
- The RS485 bus uses following software communication settings:
    - Baud rate: 250 000
    - Data bits: 8
    - Stop bits: 1
    - Parity: None
    - Flow control: None
- Use those settings when initiating a com-port session to the USB to RS485 controller

#### Optimising communication time delays and reducing lag
- When communicating with many OPET devices on one RS485 bus, it is important that the response time from an OPET is as fast as possible to be able to communicate with as many units in as little time as possible
- Normal response time of OPET is \~10ms, with the exception after a reset command or during IV and transient measurements
- This can be lengthened by the USB buffer delay times (latency timers)
    - Basically the USB buffer waits before actually sending data, especially with small data packages, to see if more data is coming in and then sends it of the RS485 controller
- In windows, one can access the USB device latency timers using: Device Manager-\>Ports-\>COM Port-\>Port Settings-\>Advanced-\>Latency Timer
    - The com port that represents the USB to RS485 controller must be selected
    - Sorry no idea where that is on Linux or MAC or any other operating system
    - Set the latency times to 1-2 ms
    - the process is a little experimental, but can speed communication up significantly

### Basic command structure and addressing 
- The following basic command structure is used:
    - Write access: `address`#`command`\[TAB\]`value`\[LF\]
        - Example Reply: `command`\[TAB\]`value`\[LF\]
    - Read access: `address`#`command`\[?\]\[LF\]
        - Example Reply: `command`\[?\]\[TAB\]`value`\[LF\]
    - `address` is the address of the OPET to communicate to
    - \# is the address separator (ASCII 0x23)
    - `command` is the actual command
    - \[TAB\] (tabulator) is the command and value separator (ASCII 9 or 0x09)
    - `value` is the control value, number / text
    - \[LF\] (line feed) is the transmission termination character (ASCII 10 or 0x0A)
    - `?` is used for read access without extra value and ends directly with \[LF\]
    - Commands are case sensitive / all in upper case
- The `address` is formed by a single character rather than the actual address number
    - This is determined via ASCII starting with `@` (ASCII 64 or 0x40) as address 0 + address of device
    - Hence `@` is address 0, `A` is address 1, `B` is address 2, ..., `\_` (ASCII 95 or 0x5F) is address 31
- If the command requires a value to be read or does not need a value, the transmission can be terminated directly after the command without sending a value with TAP separator
- OPET devices, as of yet, do not recognise multiple commands send at the same time
    - This means that only one command can be sent at a time until a response has been received from the device, see details in following section
    - If multiple commands are sent only the first command is likely being processed and communication collisions on the RS485 bus are likely
- The input buffer of the OPET accepts maximum of 120 characters per command line
    - Sending more characters without a \[LF\] termination will result in the command being rejected without notification

### Command response / reply
- The OPET communication protocol sends a response echo for any command send after it has been processed
- This can be interpreted at the control software and indicates is the command is understood and transferred correctly, details to the response are given for each command in the following instruction set sections
- It is also used to control the command flow
- The response has the same structure as the send command but without the address at the beginning, it can contain multiple values separated by \[TAB\]
- If a read access command was given, the command is returned with the requested variable value added to the command separated by \[TAB\]
- If a write access command was given, the value is first written to the MCU memory and then read-back and returned
    - In certain circumstances the returned value will be different to what was send:
        - In case it is out of boundaries of the allowed range
        - In case the give resolution of the value is greater than single float (\~6-7 digits)
- If a command is not recognized, the device does just returns a question mark `?`

## Quick reference of Instruction Set

| Command            | Description                                                      |
| ------------------ | ---------------------------------------------------------------- |
|                    | **System commands**                                              |
| `*SBR?`            | Returns system status byte                                       |
| `*OPC?`            | Returns <1> when operation complete                              |
| `*IDN?`            | Returns the device identity string                               |
| `*RST`             | Runs a soft reset of the device and reloads EEPROM settings      |
|                    | **Main Access Commands**                                         |
| `READ?`            | Returns the current PV device load data and other readings       |
| `OUTP`             | Enables or disables the PV device load output                    |
|                    | **Load Control Commands**                                        |
| `LOAD:MODE`        | Controls the load type/mode applied to the PV device             |
| `LOAD:SETVOLT`     | Sets the manual voltage mode set-point and step start voltage    |
| `LOAD:SETCURR`     | Sets the manual current mode set-point                           |
| `LOAD:MPPT:DELAY`  | Sets the MPPT output update delay in cycles                      |
|                    | **IV Curve Tracing Control**                                     |
| `IV:MEAS`          | Initiates an IV measurement                                      |
| `IV:DATA?`         | Returns last IV or transient measurement data                    |
| `IV:POINTS`        | Sets the number of IV points for IV and transient measurements   |
| `IV:DELAY`         | Controls the measurement delay during IV measurements in (ms)    |
| `IV:AVR:VC`        | Controls number measurements averaged per set per point          |
| `IV:AVR:SETS`      | Controls number measurement sets averaged per point              |
| `IV:MODE`          | Sets the IV tracing mode / control options                       |
| `IV:PHASE`         | Controls the maximum phase angle of cosine distributed IV points |
| `IV:VOC:MULT`      | Voc measurement multiplier of IV control end voltage             |
|                    | **Transient Measurement**                                        |
| `TRANS:MEAS`       | Initiates a transient measurement                                |
| `TRANS:ENDVOLT`    | Sets the step end voltage                                        |
| `TRANS:DELAY`      | Sets the point-to-point delay time in (µs)                       |
|                    | **Range control**                                                |
| `RANGE:ACTVAL?`    | Returns Actual Voltage & Current range in absolute (V) and (A)   |
| `RANGE:IDVOLT`     | Sets voltage measurement range                                   |
| `RANGE:IDCURR`     | Sets current measurement range                                   |
| `RANGE:OCRST`      | Resets the over-current bypass clamp                             |
| `RANGE:ONVOLT`     | Controls voltage ranges to be used in auto range                 |
| `RANGE:ONCURR`     | Controls current ranges to be used in auto range                 |
|                    | **Range Calibration**                                            |
| `RANGE:CAL:MODE`   | Enters / exits the calibration mode (current or voltage)         |
| `RANGE:CAL:SCALE`  | Sets / returns the scale value of the calibration range          |
| `RANGE:CAL:OFFSET` | Sets / returns the offset value of the calibration range         |
|                    | **Fan Control**                                                  |
| `FAN:MODE`         | Sets the fan control mode                                        |
|                    | **PI Driver Control**                                            |
| `DRIV:VSET?`       | Returns the reference voltage set-point of the PI driver         |
| `DRIV:IDGAIN`      | Sets the proportional gain resistor                              |
| `DRIV:IDINT`       | Sets the integrator capacitor                                    |
|                    | **ADC control Config**                                           |
| `ADC:AVR:VC`       | Sets number of voltage and current meas. averaged per cycle      |
| `ADC:AVR:OTHER`    | Sets number of other meas. averaged per cycle                    |
| `ADC:CYCLES:VC`    | Sets number of cycles averaged for current and voltage           |
|                    | **EEPROM Access**                                                |
| `EEROM:WRITE`      | Writes data to the EEPROM at a given register address            |
| `EEROM:READ?`      | Returns the EEPROM value at a given register address             |

## Detailed Instructions Set
- OPET understands the following commands as detailed in this section

### System commands

#### Read System Status Byte
- Command: `\*SBR?`
- Example reply: \*SBR? \[TAB\] 1 \[LF\]
- This is a read only command and it returns the main status byte of the OPET device
- Each bit has following meaning, false is inactive, true is active:
    - Bit 0: Output enabled
        - Indicates if the output is actively controlled ON
    - Bit 1: Calibration mode
        - Indicates if calibration mode is active
    - Bit 2: Voltage input error
        - Indicates voltage reading is invalid
        - Active when ADC value of the voltage reading is out of bounds,
        - If anyone or multiple values during averaging are at `0` or `65535`
    - Bit 3: Current input error
        - Indicates current reading is invalid
        - Active when ADC value of the current reading is out of bounds, same as previous
    - Bit 4: Over current bypass active
        - Indicates if the overcurrent protection bypass is active or not
        - If active, the current reading is invalid as the current is bypasses through a MOSFET to protect the shunt resistor from overloading and overheating
        - If current range is set to auto range this will be reset automatically unless the current is above the rating (maximum range) of the OPET board
        - If current is in manual range this will have to be reset manually (see [Overcurrent bypass reset](#overcurrent-bypass-reset))
    - Bit 5: Bias Voltage error
        - Indicates if the bias voltage is out of specified boundaries
        - for example, when supply is switched off
        - this error forces a temporary output disabled until the bias voltage is back within the specified voltage range for at least a few seconds
    - Bit 6: NTC 1 Overtemperature
        - Indicates if the onboard NTC temperature sensor reading is out of specified boundaries
        - Over or under temperature
        - this error forces a temporary output disabled until the temperature reaches the specified re-enable threshold
        - the error shows active only for as long as the temperature is over the limit
    - Bit 7: NTC 2 Overtemperature
        - Indicates if the driver MOSFET NTC temperature sensor reading is out of specified boundaries, same as previous
	-  Bit 8: Main Loop Timer Overrun
		- Indicates that the main loop timer was not reset before the next interrupt
		- basically, means that the process in the loop did not finish on time and the next loop is skipped
		- this happens normally only after processing time extensive commands such as measuring an IV curve,
		- if overrun flag is triggered during normal operation, it is likely that the number of measurements averaged is too high, see [Nu. Voltage and Current measurements averaged per cycle](#nu-voltage-and-current-measurements-averaged-per-cycle) for further details
		- the flag is reset every time the status byte is read
	-  Bit 9: New IV data ready
		- As the name suggests, this flag indicates when new IV data is ready to read from the unit
		- It is reset after the IV data transfer has been requested
	-  Bit 10: Voltage range hold up
		- Means that the range switching frequency timer is running down preventing the voltage range from switching downwards
		- Does not affect switching voltage range upwards
		- A change in voltage range sets the switching frequency timer as well as when the output is disabled or the OPET has an active error
	-  Bit 11: Current range hold up
		- Same as bit 10, but for current range


#### Operation Complete Query
- Command: `\*OPC?`
- Example reply: \*OPC? \[TAB\] 1 \[LF\]
- This is a read only command, a reply is sent only if the device is ready and "idle" in load control mode or Off mode, after for example measuring an IV curve
- Use to check if an IV-curve or transient measurement is finished

#### Identify
- Command: `\*IDN?`
- Example reply: \*IDN? \[TAB\] OPET_R1.3 \[TAB\] R0.50-01/08/2021 \[TAB\] Not Sure! \[LF\]
- a read only command, that returns the device name with hardware revision, the firmware version with date and the board identifier as specified in EEPROM
- all separated by \[TAB\]

#### Reset
- Command: `\*RST`
- Example reply: \*RST \[TAB\] 1 \[LF\]
- A write only command that does not require a specific value
- The OPET device sends the reply and then immediately after triggers a software reset and returns to the state as specified in the EEPROM
- Use this command to reset the device after the EEPROM is written or after calibrating to refresh the internal memory with the new permanently stored values

### Main access commands

#### Read status and output measurements
- Command: `READ?`
- Example reply: READ? \[TAB\] `SBR` \[TAB\] `Vpv` \[TAB\] `Cpv` \[TAB\] `Voff` \[TAB\] `Vb` \[TAB\] `NTC1` \[TAB\] `NTC2` \[TAB\] `RTD` \[LF\]
- read only command that returns the latest ADC input readings and the system status byte
    - it does not initiate a new ADC reading, just returns the last reading results
    - readings are taken all the time with every measurement trigger, if requested or not
- this command should be used to transfer the reading of the OPET device to the logging software
- following data is reported:
    - `SBR`: system status byte, see previous section
    - `Vpv`: PV device voltage
    - `Cpv`: PV device current
    - `Voff`: internal reference voltage offset
    - `Vb`: bias supply voltage
    - `NTC1`: temperature of the onboard temperature sensor
    - `NTC2`: temperature of the MOSFET driver temperature sensor
    - `RTD`: [only if enabled, the temperature reading of the temperature sensor connected]{.mark}
- The internal reference voltage offset `Voff` is a reference signal that should always be the same
    - only drifts with excessive temperature change in the onboard operating temperature
    - an internal zero value ADC count determining zero volts and current, and enabling slightly negative current and voltage measurements on a unipolar ADC reading
    - this reading could be used for advanced drift corrections on readings and voltage control, but this has not been implemented as of yet
    - value given in raw ADC counts
- The bias supply offset voltage `Vb` should be more regarded as lower accuracy guide number rather that an accurate reading similar to load voltage and current
    - The reference voltage can change significantly with current load on the PV module

#### Output enable/ disable
- Write Command: `OUTP` \[TAB\] `value` \[LF\]
    - `0` -- to disable the output, `1` -- to enable the output
- Read Command: `OUTP?` \[LF\]
    - Example reply: OUTP? \[TAB\] 1 \[LF\]
- Read write access to switch on/off the load control output
    - in off-state the MOSFET driver is forced into high resistance state
    - in on-state the driver controls the desired output load of the PV device

### Load Control Commands

#### Load Mode control
- Write Command: `LOAD:MODE` \[TAB\] \[mode value\] \[LF\]
- Read Command: `LOAD:MODE?` \[LF\]
	    - Example reply: `LOAD:MODE?\t1\n`
- Controls the PV device load mode
- Following values are valid:
    - `0` None -- off-state mode, output will not enable and disable
    - `1` Voc -- open circuit load
    - `2` Isc -- short circuit load
    - `3` Vset -- constant voltage load at given voltage set-point see below
    - `4` Cset -- constant current load at given current set-point see below
    - `5` MPPT -- maximum power point tracker

#### Voltage Set-Point
- Write Command: `LOAD:SETVOLT` \[TAB\] `voltage value` \[LF\]
- Read Command: `LOAD:SETVOLT?` \[LF\]
    - Example reply: LOAD:SETVOLT? \[TAB\] 1.2 \[LF\]
- Voltage set-point value in volts can be anything, but must be positive, if the set-point is too high and cannot be reached, the system will operate at Voc
- This set-point is used when in constant voltage mode and as the static start value during voltage transient measurements (see [Transient measurement control](#transient-measurement-control))

#### Current Set-Point
- Write Command: `LOAD:SETCURR` \[TAB\] `current value` \[LF\]
- Read Command: `LOAD:SETCURR?` \[LF\]
    - Example reply: LOAD:SETCURR? \[TAB\] 1.2 \[LF\]
- Current set-point value in amperes can be anything, but must be positive, if the set-point is to high and cannot be reached, the system will operate at Isc

#### MPPT update delay
- Write Command: `LOAD:MPPT:DELAY\t` \[TAB\] `number cycles` \[LF\]
- Read Command: `LOAD:MPPT:DELAY?` \[LF\]
    - Example reply: LOAD:MPPT:DELAY? \[TAB\] 1 \[LF\]
- the MPPT update delay determines how often the MPPT tracer updates the output, it is given in number of internal OPET control cycles
    - the minimum is 1, meaning the MPPT is updated every control cycle
    - the maximum is 60000, meaning the MPPT waits 60000 cycles before updating again
        - at a 25ms cycle time this would be every 25minutes
- this delay is useful when testing very "slow responding" devices with metastable response, such as some perovskite solar cells

### IV Curve Tracing Commands

#### Start IV curve measurement
- Write only Command: `IV:MEAS` \[LF\]
    - Example reply: IV:MEAS \[TAB\] 900 \[LF\]
- This command will initiate the start of and IV curve measurement, no value needed
- The return value is the expected total IV measurement time in milliseconds including I<sub>sc</sub> and V<sub>oc</sub> pre-measurements
    - If active error found or the output is off, `0` is returned and measurement is not started

#### Read IV curve Data
- Read only command: `IV:DATA?` \[LF\]
- This function returns the last measured IV curve or transient data
    - IV curve and transient measurement share the same buffer on the micro controller
    - calling this function multiple times will return the same data unless a new measurement is taken in-between
- Example reply: IV:DATA? \[TAB\] `IVSB` \[TAB\] `V1` \[TAB\] `C1` \[TAB\] ... \[TAB\] `VN` \[TAB\] `CN` \[LF\]
- `IVSB`, the first parameter given is the IV curve measurement status byte
    - In principle, as long as its `0` the IV curve measurement was a success
    - has following bit definitions:
        - Bit 0: overcurrent bypass active at end of IV tracing
        - Bit 1: over- under temperature fault, IV cancelled
        - Bit 2: bias voltage out of range
        - Bit 3: none (always false)
        - Bit 4: voltage ADC input over-load at one or more measurements
        - Bit 5: voltage ADC input under-load at one or more measurements
        - Bit 6: current ADC input over-load at one or more measurements
        - Bit 7: current ADC input under-load at one or more measurements
- After the status byte, the voltage `VN` and current `CN` points are given for the entire IV curve, while N represents the point ID up to the configured number of IV points

#### Number of IV Points
- Write Command: `IV:POINTS` \[TAB\] `value` \[LF\]
- Read Command: `IV:POINTS?` \[LF\]
    - Example reply: IV:POINTS? \[TAB\] 100 \[LF\]
- This value specifies the number of IV points that are measured for every IV curve and transient measurement curve
- Minimum is `3` and maximum is `250`
- If the requested value is out of range, it will be cohered to the minim or maximum value

#### IV point settling time delay in ms
- Write Command: `IV:DELAY` \[TAB\] `value` \[LF\]
- Read Command: `IV:DELAY?` \[LF\]
    - Example reply: IV:DELAY? \[TAB\] 1 \[LF\]
- This specifies the time delay in milliseconds between setting the voltage of the PV device and start of measuring the voltage and current point
- value range accepted is between `1` and `60000` \[ms\], number fractions are ignored
- Default value would be 5-10 ms for normally responding PV devices

#### IV data Averaging control
- Commands:
    - Number Voltage and current averages per set: `IV:AVR:VC` \[TAB\] `value` \[LF\]
    - Number of averaging sets: `IV:AVR:SETS` \[TAB\] `value` \[LF\]
- Both commands support read-write functions
- `IV:AVR:VC` controls the number of measurements that are averaged in one set
    - voltage is N times measured and then current is N times measured to form one set
- `IV:AVR:SETS` controls the number of sets of data to average
    - For every set it will measure voltage and current again
- Averaging in general reduces noise
- With multiple sets of current and voltage measurements following can be achieved:
    - Averaging for longer without large time delays between measuring current and voltage
    - Correctly tuned multiple sets can be measured throughout one or more full power cycles to reduce noise in the measurement which is useful when operating under artificial light or in environments with high electrical noise
- default just one set is measured
- both settings allow integer numbers between `1` and `255`

#### IV measurement mode
- Write Command: `IV:MODE` \[TAB\] `mode byte` \[LF\]
- Read Command: `IV:MODE?` \[LF\]
    - Example reply: IV:MODE? \[TAB\] 3 \[LF\]
- The IV mode control byte allows changing the methodology of the IV curve measurement
- Following bit definitions are available:
    - Bit 0: Cosine IV point distribution
        - `1` for cosine IV point distribution (enabled by default), or `0` for linear IV point distribution
    - Bit 1: Asymmetric voltage measurements
        - `1` asymmetric mode, at the end of every IV point measurement an extra set of only voltage measurements is taken to partly compensate for incomplete settling of signals and changing voltage and current signals during measuring (enabled by default)
        - `0` symmetric mode, no extra voltage measurement at the end for each IV point
    - Bit 2: Reverse direction IV measurements
	    - `1` reverse direction IV measurement from open circuit voltage to 0V
	    - `0` forward direction IV measurement from 0V to open circuit voltage
    - Bit 3: none
    - Bit 4: none
    - Bit 5: none
    - Bit 6: none
    - Bit 7: none

#### Cosine maximum phase in radians
- Write Command: `IV:PHASE` \[TAB\] `phase in radians` \[LF\]
- Read Command: `IV:PHASE?` \[LF\]
    - Example reply: IV:PHASE? \[TAB\] 1.57 \[LF\]
- This parameter defines the maximum phase angle used for the nonlinear cosine point distribution
- It effectively controls where the maximum point distribution is
- Default is 1.57 or π/2, any value can be used here, but values below 1 or values larger than π may result in unexpected performance
- If a value between 1 and π/2 is used the point distribution is highest around the open circuit voltage and getting denser with increasing phase
- If a value between π/2 and π is used, the highest point density moves towards MPPT with increasing value and at V<sub>oc</sub> the point density reduces again

#### VOC overshoot control multiplier
- Write Command: `IV:VOC:MULT` \[TAB\] `multiplier` \[LF\]
- Read Command: `IV:VOC:MULT?` \[LF\]
    - Example reply: IV:VOC:MULT? \[TAB\] 1.01 \[LF\]
- Before any IV measurements, the I<sub>sc</sub> and V<sub>oc</sub> of the IV curve are measured for controlling the measurement input ranges and the voltage tracing control range
- This value determines the extra factor that is applied to the V<sub>oc</sub> measurement to calculate the voltage point distribution of the IV curve
- If the value is above 1, the control voltage will go above V<sub>oc</sub> which effectively leads to a few points of the IV curve being measured at V<sub>oc</sub>
    - Makes sure V<sub>oc</sub> is always present
- If the value is below 1, V<sub>oc</sub> will not be reached, and the IV curve will finish mid-way

### Transient measurement control

#### Start transient measurement
- Write only Command: `TRANS:MEAS` \[LF\]
- Example reply: TRANS:MEAS \[TAB\] 1 \[LF\]
- This command will initiate the start of a transient measurement, no value needed
- The transient measurement data is collected using the `IV:DATA?` command
- It will return `1` if there is no active error that prohibits transient measurements
    - If active error found `0` is returned and measurement is not started

#### Transient end voltage
- Write Command: `TRANS:ENDVOLT` \[TAB\] `voltage in V` \[LF\]
- Read Command: `TRANS:ENDVOLT?` \[LF\]
    - Example reply: TRANS:ENDVOLT? \[TAB\] 1.01 \[LF\]
- The parameter defines the transient end set-point voltage
- The start point is defined by the voltage load control set-point see [Voltage Set-Point](#voltage-set-point)

#### Transient point delay in us 
- Write Command: `TRANS:DELAY` \[TAB\] `delay in microseconds` \[LF\]
    - Example reply TRANS:DELAY \[TAB\] 10.1 \[LF\]
- Read Command: `TRANS:DELAY?` \[LF\]
    - Example reply: TRANS:DELAY? \[TAB\] 1.01 \[LF\]
- Defines the delay time between start of individual voltage or current measurements
- multiplied with the number of points measured gives the total measurement time
- This parameter recognises fractional numbers, however a finite resolution is given by the voltage or current point measurement time and delay loop controller cycles
- The minimum is \~12us for the measurement and \~0.6us per delay cycle[^1]
[^1]:(check, adjust that)

### Measurement range control

#### Read active measurement range value
- Read only command: `RANGE:ACTVAL?` \[LF\]
    - Example reply: RANGE:ACTVAL? \[TAB\] `Voltage range in V` \[TAB} `Current range in A` \[LF\]
- This read only command returns the active measurement ranges for voltage and current in volts and amperes as absolute values rather than an ID definition that is more difficult to recognise
- The current range values returned here may be different to the actual range size
    - i.e. a 320mA range is returned for a 340mA range
    - this is an adjustment made to improve the calibration, as the calibrator will lower range and achieve a higher total calibration accuracy at 320mA instead of 340mA

#### Voltage Range ID
- Write Command: `RANGE:IDVOLT` \[TAB\] `voltage range ID` \[LF\]
- Read Command: `RANGE:IDVOLT?` \[LF\]
    - Example reply: RANGE:IDVOLT? \[TAB\] auto \[LF\]
- This is used to manually control the voltage measurement range or to set it to auto ranging
- Following ranges are available:
    - `0` - 1V range
    - `1` - 4.2V range
    - `2` - 10V range
    - `3` - 30V range
    - `4` - 100V range
    - `any other` - Auto range
- Reading this value does return "auto" in auto-range mode to indicate that its auto-ranging

#### Current Range ID
- Write Command: `RANGE:IDCURR` \[TAB\] `current range ID` \[LF\]
- Read Command: `RANGE:IDCURR?` \[LF\]
    - Example reply: RANGE:IDCURR? \[TAB\] auto \[LF\]
- This command is used to manually control the current measurement range or to set it to auto ranging
- Following ranges are available (depending on the current rating of the OPET version):
    - `0` - 1.07mA or 50mA range
    - `1` - 3.2mA or 150mA range
    - `2` - 11mA or 500mA range
    - `3` - 33mA or 1.5A range
    - `4` - 110mA or 5A range
    - `5` - 340mA or 15A range
    - `any other` - Auto range
- Reading this value does return "auto" in auto-range mode to indicate that its auto-ranging

#### Overcurrent bypass reset
- Write Only Command: `RANGE:OCRST` \[LF\]
    - Example reply RANGE:OCRST \[TAB\] 1 \[LF\]
- This write only command does not require a value and manually resets the overcurrent bypass control of the OPET, which is used to protect the shunt resistors from over-voltage and over-heating
- In current auto-range mode this is done automatically after switching to a higher current range
- In manual mode it is advised to switch manually to a higher range before resetting the bypass unless it is known that the overcurrent is removed
- The bypass will immediately activate again if the over-current state is not dealt with beforehand
- The bypass activates within \~100 milliseconds of detecting overcurrent independently of the microcontroller and hence can be triggered by short current spikes

#### Control voltage ranges to be used in auto range mode
- Write Command: `RANGE:ONVOLT`\[TAB\]\<range enabled byte\>\[LF\]
- Read Command: `RANGE:ONVOLT?`\[LF\]
	- Example reply: RANGE:ONVOLT? \[TAB\] 255 \[LF\]
- This command is used to control which ranges can be accessed in auto ranging mode
	- If a range is disabled for auto range, the next higher enabled range is used instead
- The range enable byte, send as 8-bit unsigned number in write command, has for each voltage range a corresponding bit, given as follows:
	- Bit 0: 1V range
	- Bit 1: 4.2V range
	- Bit 2: 10V range
	- Bit 3: 30V range
	- Bit 4: 100V range, always enabled
	- All others, nothing
- If a bit is set to true the range is enabled for auto range and if set to false it is not used during auto range, but can still be accessed in manual range control
- The highest voltage range can not be disabled, as it is the default range in “off mode”

#### Control current ranges to be used in auto range mode
- Write Command: `RANGE:ONCURR` \[TAB\] \<range enabled byte\> \[LF\]
- Read Command: `RANGE:ONCURR?`\[LF\]
	- Example reply: `RANGE:ONCURR?\t255\n`
- This command is used to control which ranges can be accessed in auto ranging mode
	- If a range is disabled for auto range, the next higher enabled range is used instead
- The range enable byte, send as 8bit unsigned number in write command, has for each current range a corresponding bit, given as follows:
	- Bit 0: 1.07mA or 50mA range
	- Bit 1: 3.2mA or 150mA range
	- Bit 2: 11mA or 500mA range
	- Bit 3: 33mA or 1.5A range
	- Bit 4: 110mA or 5A range
	- Bit 5: 340mA or 15A range, always enabled
	- All others, nothing
- If a bit is set to true the range is enabled for auto range and if set to false it is not used during auto range, but can still be accessed in manual range control
- The highest current range cannot be disabled, as it is the default range in “off mode”

### Range calibration control
- With the range calibration commands the OPET device can be calibrated without having to directly access the EEPROM memory, which simplifies the procedure
- For specific calibration procedure details, please refer to [Basic 2-point calibration](#basic-2-point-calibration)

#### Calibration Mode Control
- Write Command: `RANGE:CAL:MODE` \[TAB\] `mode type` \[LF\]
- Read Command: `RANGE:CAL:MODE?` \[LF\]
    - Example reply: RANGE:CAL:MODE? \[TAB\] CURR \[LF\]
- This command is used to enter and exit the calibration mode of the OPET ADC system
- Calibration mode restricts the output to OFF only
- The Mode types that can be selected are following:
    - `VOLT` - for voltage calibration
    - `CURR` - for current calibration
    - `anything else` or `0` disables calibration mode
- Calibration mode, switches to direct raw count data output, which can be used to measure and calculate the calibration factors
    - Raw count data is still averaged as specified in the associated settings
    - only the calibration scale and offset are set to 1 and 0 respectively

#### Set Range Scale Value
- Write Command: `RANGE:CAL:SCALE` \[TAB\] `value` \[LF\]
- Read Command: `RANGE:CAL:SCALE?` \[LF\]
    - Example reply: RANGE:CAL:SCALE? \[TAB\] 1.2032 \[LF\]
- This command can be used to set or read the scale value of the selected range from the selected calibration mode (voltage or current)
- When writing this command it is advised to use the scientific number format (1.1234E-01) with up to 7 digits
- Command is only active for as long as the calibration mode is active
- If calibration mode is not activated the device will return `No Cal!` as value

#### Set Range Offset Value
- Write Command: `RANGE:CAL:OFFSET` \[TAB\] `value` \[LF\]
- Read Command: `RANGE:CAL:OFFSET?` \[LF\]
    - Example reply: RANGE:CAL:OFFSET? \[TAB\] 647.5 \[LF\]
- This command can be used to set or read the offset value of the selected range from the selected calibration mode (voltage or current)
- When writing this command it is advised to use the scientific number format (1.1234E-01) with up to 7 digits
- Command is only active for as long as the calibration mode is active
- If calibration mode is not activated the device will return `No Cal!` as value

### Fan Control

#### Fan control mode
- Write Command: `FAN:MODE` \[TAB\] `2` \[LF\]
- Read Command: `FAN:MODE?` \[LF\]
    - Example reply: FAN:MODE? \[TAB\] AUTO ON \[LF\]
- This command is used the control the fan and change the control mode, accepted values are following:
    - `0` - manual control mode, fan turned off
    - `1` - manual control mode, fan turned on
    - `2` - automatic control dependent on power and temperature readings
- When reading from this command the state of the fan control is returned in string format
    - `MAN ON`, `MAN OFF` - manual control fan turned on or off
    - `AUTO ON`, `AUTO OFF` - automated control with the current active state of the fan control (either it is turned on or off at that moment)
- The temperature and power threshold values for the automatic fan control can be configure in the EEPROM, see [Operating Temperature limits configuration](#operating-temperature-limits-configuration)

### PI Driver Control

#### Reference voltage DAC setpoint
- Read Command: `DRIV:VSET?` \[LF\]
    - Example reply: DRIV:VSET? \[TAB\] 10.45 \[LF\]
- This read only command returns the setpoint voltage that controls the input of the PI regulator driver circuit
- Useful for debugging and calibrating the DAC output

#### Driver Proportional Gain Control
- Write Command: `DRIV:IDGAIN` \[TAB\] `gain resistor ID` \[LF\]
- Read Command: `DRIV:IDGAIN?` \[LF\]
    - Example reply: DRIV:IDGAIN? \[TAB\] 1 \[LF\]
- This controls the proportional gain of the driver PI controller
- Gain resistor ID values from 0 to 3 are valid, if the value is outside of those bounds will be cohered to the minim or maximum value
- With increasing ID value, the proportional gain is reducing, which is in most cases increases the driver stability
- Adjust this value to stabilize driver regulation in case of oscillations and to optimise voltage control step response

#### Driver Integrator Capacitor Control
- Write Command: `DRIV:IDINT` \[TAB\] `gain resistor ID` \[LF\]
- Read Command: `DRIV:IDINT?` \[LF\]
    - Example reply: DRIV:IDINT? \[TAB\] 1 \[LF\]
- This controls the integrator capacitor of the driver PI controller
- Integrator capacitor ID values from 0 to 3 are valid, if the value is outside of those bounds will be cohered to the minim or maximum value
- With increasing ID value, the integrator capacitor is increasing, this reduces the response time of the driver and is in most cases increasing the driver stability
- Adjust this value to stabilize driver regulation in case of oscillations and to optimise voltage control step response

### ADC Control Commands

#### Number of Voltage and Current measurement averaged
- Write Command: `ADC:AVR:VC` \[TAB\] `nu average measurements` \[LF\]
- Read Command: `ADC:AVR:VC?` \[LF\]
    - Example reply: ADC:AVR:VC? \[TAB\] 1 \[LF\]
- This command controls how many consecutive current and voltage load measurements are taken and averaged at each measurement cycle (does not impact IV data point averaging)
- The value can range from `1` to `255` and is cohered to minimum or maximum if out of range
- This can be used to greatly reduce the measurement fluctuations due to noise
- When controlling this value, it is important to make sure the value is not too high to cause measurement cycle skipping, which happens, if the device is not "idle" before the measurement cycle time trigger is given

#### Number of Other measurements averaged
- Write Command: `ADC:AVR:OTHER` \[TAB\] `nu average measurements` \[LF\]
- Read Command: `ADC:AVR:OTHER?` \[LF\]
    - Example reply: ADC:AVR:OTHER? \[TAB\] 1 \[LF\]
- This command controls how many consecutive measurements of all other channels are taken and averaged at each measurement cycle
- The value can range from `1` to `255` and is cohered to minimum or maximum if out of range
- When controlling this value, it is important to make sure the value is not too high to cause measurement cycle skipping, which happens, if the device is not "idle" before the measurement cycle time trigger is given

#### Number of Voltage and Current cycles averaged
- Write Command: `ADC:CYCLES:VC` \[TAB\] `nu average measurements` \[LF\]
- Read Command: `ADC:CYCLES:VC?` \[LF\]
    - Example reply: ADC:CYCLES:VC? \[TAB\] 1 \[LF\]
- This command controls how many measurements over consecutive cycles are averaged in an continues stream for voltage and current only
- The value can range from `1` to `100` and is cohered to minimum or maximum if out of range
- Since this function averages over multiple measurement cycles at specific time intervals, this function can be used to reduce low frequency noise from power lines
    - For best performance average over 1 or more full power line cycles
    - OPETs default measurement cycle period 4.167ms, 4 times over one power cycle
    - Hence a value of 4 averages one full power cycle as 60Hz

### EEPROM Access Commands

#### Write to EEPROM
- Command: `EEROM:WRITE` \[TAB\] `register address` \[TAB\] `register value` \[LF\]
    - Example reply EEROM:WRITE \[TAB\] 4 \[TAB\] 10 \[LF\]
- This write only command directly accesses the EEPROM and should be used carefully, as there is only a finite amount of write cycles of each address space in the EEPROM memory (\~100,000)
- As shown above this command requires 2 variables, the register address and the associated register value
- Refer to [EEPROM Address Specifications](#eeprom-address-specifications) for details regarding the register address and register value type

#### Read from EEPROM
- Command: `EEROM:READ?` \[TAB\] `register address` \[LF\]
    - Example reply EEROM:READ? \[TAB\] 4 \[TAB\] 10 \[LF\]
- This read only command directly accesses the EEPROM, reading from the EEPROM can be done infinite amount of times
- As shown above this command requires the register address to be accessed and the reply will contain additionally the associated value at the register address
- Refer to [EEPROM Address Specifications](#eeprom-address-specifications) for details regarding the register address and register value type

# EEPROM Address Specifications

## Introduction
- The EEPROM (electrically erasable programmable read-only memory) is a non-volatile memory space of the OPET devices used to store the system configuration values
- The configuration is loaded during the short boot-up phase
- Memory space is accessible via read and write functions
    - write should not be used frequently, as there is only a finite amount of write cycle lifetime (\~100,000) of each address space in the EEPROM memory
- EEPROM values can be configured to auto-start PV device active loading functions (for example MPPT) autonomously right after powering up without the need to interface with a computer
    - Useful after for example power-outages or glitches

## Address Definitions Summary
- Below a list of EEPROM addresses
- Datatypes:
    - uint_8 = 8 bit unsigned integer 0 -- 255
    - uint_16 = 16 bit unsigned integer 0 -- 65536
    - single float = floating point \~7digits accuracy (0.000000+E00)
    - string = any text with signs, numbers... of specified length

| Register ID | Name                                          | Data Type      |
|:-----------:| --------------------------------------------- | -------------- |
|      0      | EEPROM valid (165 = load from EEPROM)         | uint_8         |
|      1      | Sample / Device Name                          | String 20 char |
|      2      | PCB hardware config byte                      | uint_8         |
|      3      | Temperature sensor type                       | uint_8         |
|      5      | CPU Frequency Value                           | uint_32        |
|      6      | Main Loop Timer Value                         | uint_16        |
|      7      | Control timer loop multiplier                 | uint_8         |
|      8      | Temperature measurement delay [cycles]        | uint_16        |
|     10      | ADC num. volt & curr meas. Averaged per cycle | uint_16        |
|     11      | ADC num. other meas. Averages per cycle       | uint_8         |
|     12      | ADC num. volt & curr meas. Cycles averaged    | uint_8         |
|     20      | Range control status                          | uint_8         |
|     21      | Manual voltage Range ID                       | uint_8         |
|     22      | Manual current Range ID                       | uint_8         |
|     23      | Voltage range switching freq. counter         | unit_16        |
|     24      | Current range switching freq. counter         | unit_16        |
|     25      | Volt R1 A0                                    | single float   |
|     26      | Volt R1 A1                                    | single float   |
|     27      | Volt R1 Range Val                             | single float   |
|     28      | Volt R1 leakage                               | single float   |
|     30      | Volt R2 A0                                    | single float   |
|     31      | Volt R2 A1                                    | single float   |
|     32      | Volt R2 Range Val                             | single float   |
|     33      | Volt R2 leakage                               | single float   |
|     35      | Volt R3 A0                                    | single float   |
|     36      | Volt R3 A1                                    | single float   |
|     37      | Volt R3 Range Val                             | single float   |
|     38      | Volt R3 leakage                               | single float   |
|     40      | Volt R4 A0                                    | single float   |
|     41      | Volt R4 A1                                    | single float   |
|     42      | Volt R4 Range Val                             | single float   |
|     43      | Volt R4 leakage                               | single float   |
|     45      | Volt R5 A0                                    | single float   |
|     46      | Volt R5 A1                                    | single float   |
|     47      | Volt R5 Range Val                             | single float   |
|     48      | Volt R5 leakage                               | single float   |
|     55      | Curr R1 A0                                    | single float   |
|     56      | Curr R1 A1                                    | single float   |
|     57      | Curr R1 Range Val                             | single float   |
|     60      | Curr R2 A0                                    | single float   |
|     61      | Curr R2 A1                                    | single float   |
|     62      | Curr R2 Range Val                             | single float   |
|     65      | Curr R3 A0                                    | single float   |
|     66      | Curr R3 A1                                    | single float   |
|     67      | Curr R3 Range Val                             | single float   |
|     70      | Curr R4 A0                                    | single float   |
|     71      | Curr R4 A1                                    | single float   |
|     72      | Curr R4 Range Val                             | single float   |
|     75      | Curr R5 A0                                    | single float   |
|     76      | Curr R5 A1                                    | single float   |
|     77      | Curr R5 Range Val                             | single float   |
|     80      | Curr R6 A0                                    | single float   |
|     81      | Curr R6 A1                                    | single float   |
|     82      | Curr R6 Range Val                             | single float   |
|     83      | Voltage Ranges Enabled State                  | uint_8         |
|     84      | Current Ranges Enabled State                  | uint_8         |
|     85      | Voltage range switching delay counter         | uint_8         |
|     86      | Current range switching delay counter         | uint_8         |
|     90      | Bias A0                                       | single float   |
|     91      | Bias A1                                       | single float   |
|     95      | RTD A0                                        | single float   |
|     96      | RTD A1                                        | single float   |
|     97      | RTD A2                                        | single float   |
|     100     | NTC 1 inverse gain                            | single float   |
|     101     | NTC 1 series resistance                       | single float   |
|     102     | NTC 1 inverse R25                             | single float   |
|     103     | NTC 1 inverse Beta                            | single float   |
|     105     | NTC 2 inverse gain                            | single float   |
|     106     | NTC 2 series resistance                       | single float   |
|     107     | NTC 2 inverse R25                             | single float   |
|     108     | NTC 2 inverse Beta                            | single float   |
|     110     | Control Voltage DAC Offset A0                 | single float   |
|     111     | Control Voltage DAC Scale A1                  | single float   |
|     115     | PI CTR Gain Resistor ID                       | uint_8         |
|     116     | PI CTR Integrator Capacitor ID                | uint_8         |
|     125     | NTC 1 Low Temp Disconnect                     | single float   |
|     126     | NTC 1 High Temp Disconnect                    | single float   |
|     127     | NTC 1 Low Temp Reconnect                      | single float   |
|     128     | NTC 1 High Temp Reconnect                     | single float   |
|     130     | NTC 2 Low Temp Disconnect                     | single float   |
|     131     | NTC 2 High Temp Disconnect                    | single float   |
|     132     | NTC 2 Low Temp Reconnect                      | single float   |
|     133     | NTC 2 High Temp Reconnect                     | single float   |
|     135     | Bias Volage Range Minimum                     | single float   |
|     136     | Bias Voltage Range Maximum                    | single float   |
|     140     | Number of IV points                           | uint_8         |
|     141     | IV tracing mode                               | uint_8         |
|     142     | Cos Sweep Max Phase                           | single float   |
|     143     | Voc Overshoot Factor                          | single float   |
|     144     | IV point settle time in \[ms\]                | uint_16        |
|     145     | IV point num. meas. Sets averaged             | uint_8         |
|     146     | IV point num. meas. Averaged per set          | uint_8         |
|     150     | MPPT max step size count                      | single float   |
|     151     | MPPT min step size count                      | single float   |
|     152     | MPPT step size increase factor                | single float   |
|     153     | MPPT step size reduction factor               | single float   |
|     154     | MPPT tolerance range count                    | single float   |
|     155     | MPPT update delay \[cycles\]                  | uint_16        |
|     160     | CurrT max step size count                     | single float   |
|     161     | CurrT min step size count                     | single float   |
|     162     | CurrT step size increase factor               | single float   |
|     163     | CurrT step size reduction factor              | single float   |
|     164     | CurrT tolerance range count                   | single float   |
|     165     | System Control Byte                           | uint_8         |
|     166     | Load Control Mode ID                          | uint_8         |
|     167     | PV voltage load setpoint                      | single float   |
|     168     | PV current load setpoint                      | single float   |
|     175     | Fan Control Mode                              | uint_8         |
|     176     | Fan NTC 1 Temp On                             | single float   |
|     177     | Fan NTC 1 Temp Off                            | single float   |
|     178     | Fan NTC 2 Temp On                             | single float   |
|     179     | Fan NTC 2 Temp Off                            | single float   |
|     180     | Fan Power On                                  | single float   |
|     181     | Fan Power Off                                 | single float   |
|182|Fan Switch Frequency Counter|uint_16|            |                                               |                |

## Detailed Address Definitions
- Default values are loaded if the EEPROM is not valid
- Standard values are the from software revision REV1.04 of the EEPROM after programming

### EEPROM Valid
- Register ID: `0`, default value `165`
- Must have the value `165` written to it for the OPET device to read the config from EEPROM
- Used to make sure that a OPET device without programmed EEPROM is not reading an invalid configuration
- At any other value the default configuration will be loaded, which has no calibration factors (i.e. works in and returns raw counts for signals)
    - May be useful for debugging purposes

### Hardware config & ID variables

#### Sample or Device Name
- Register ID: `1`,
- Value: default `NoErom!`, standard `Not Sure!`
- Recognises a maximum of 20 characters, any more are truncated/removed
- Can have signs, numbers, letters, pretty much anything, except of `@`, `:`, `TAB`, `NULL`, `CR`, `NL`

#### Hardware config byte
- Register ID: `2`,
- Value: default `byte 0000 0001`, standard `byte 0000 0001`
- The hardware config byte details the hardware configuration of OPET to the microcontroller
- Following bit configuration is available:
    - Bit 0: Temperature Sensor enable / disabled
        - This enables the temperature sensing option connected to the SPI or I2C extension ports
    - All others, not in use yet

#### Temperature sensor type
- Register ID: `3`,
- Value: default `2`, standard `2`
- This uint_8 value specifies the temperature sensor breakout board type connected to the SPI or I2C extension header
- Following values, devices are supported:
    - 0 -- no temperature sensor
    - 1 -- Max31865, Adafruit RTD PT100 amplifier connected to SPI interface
    - 2 -- MCP9600, Adafruit Thermocouple amplifier connected to I2C interface

### Timing control variables

#### CPU frequency Value
- Register ID: `5`,
- Value: default `16000000`, standard `16000000`
- EEPROM value of the microcontroller (MCU) frequency, it can be used to calibrate the exact value for each OPET device for timing purposes
- At current, this value is only used to calculate the time delays for transient measurements
- Other timings are currently not impacted by this value

#### Main measurement loop timer counter
- Register ID: `6`,
- Value: default `1041`, standard `1041`
- Value Range: `1–65535`
- This is the timer counter value for the main measurement loop
- The counter counts up to this value, resets to start again and sends a signal to the main measurement loop to start
- The exact time *t* the counter takes is calculated as follows, whereas *C<sub>val</sub>* is the loop timer value, *64* is the clock divider and *F<sub>MCU</sub>* is the frequency of the microcontroller:

$$t = \frac{C_{val}*64}{F_{MCU}} = \ \frac{1041*64}{16000000} = 4.164\ ms\  \approx \ \frac{1}{60\ Hz*4}$$

$$C_{val} = \frac{t\ *{\ F}_{MCU}}{64}$$
- It is highly recommended to keep the main measurement loop timer at exactly one quarter of the mains frequency to enable effective noise rejection of any mains electrical interference
- The main measurement loop timer controls the execution of communication functions, to enable effective measurement synchronisation, i.e. RS485 commands are executed during the remaining idle time after the measurements have completed
- This timer also controls the counter for the control loop (see [Control loop timer multiplier](#control-loop-timer-multiplier))

#### Control loop timer multiplier
- Register ID: `7`,
- Value: default `6`, standard `6`
- Value Range: `2–255`
- The control loop timer multiplier counts the amount of main measurement loops executed up to the given value, resets to start counting again and sends a signal to the main control loop to start control function execution
- The functions executed control the OPET load functions such as the maximum power point tracker and as such determine how often load control voltage values are updated
- This value must be at a minimum of 2, otherwise communication to the OPET unit will fail, as this is done outside of a control timer loop to reduce potential loop time overruns
- It is recommended to keep this multiplier value at a minimum of 5, which allows for averaging of over one power cycle (4 measurements) + some extra time to settle to the new voltage set-point (1 extra cycle)
- The normal value in the EEPROM of `6` (every \~25ms) is chosen to give the OPET more time to settle to the new control value before for example a new MPPT load cycle starts
    - i.e. 2 measurement cycles settling time + 4 measurement cycles averaging time
- The control functions also include the OPET health monitoring functions such as over temperature cut-out, hence it is advisable that the multiplier is not set too high to keep the execution time to at least once a second

#### Temperature measurement loop timer multiplier
- Register ID: `8`,
- Value: default `120`, standard `120`
- Value Range: `1 ... 65535`
- This temperature measurement timer multiplier functions the same as the control loop timer, but it solely controls temperature measurement interval of the connected temperature sensor
- as per default, the timer value in cycles is set to achieve a 500 ms measurement interval
- This does nothing if the temperature measurement functionality if disabled in the EEPROM configuration

### Measurement loop ADC control variables

#### Nu. Voltage and Current measurements averaged per cycle
- Register ID: `10`,
- Value: default `1`, standard `50`
- Value Range: `1 \... (65534 / Nu cycles averaged)`
- This value controls how many ADC PV current and voltage measurements are averaged during each measurement cycle
- `1` equals no averaging
- the maximum value that can be used here depends on the amount of cycles the data is averaged over `65535 / Nu cycles averaged`
    - If the averaging value is too high, the MCU may encounter an overflow on the 32bit averaging buffer and hence the final value read will be false
- Care should be taken that the number of measurements averaged is not too high to cause time synchronisation problems on the measurement loop timer
    - Each measurement takes \~20us, hence 50 measurements take 1ms to complete for each channel (voltage and current), \~2ms in total
    - Try to keep at least 2-3ms of idle time at each measurement loop for the control function execution and communication functions
    - If the main measurement loop is not complete before a new loop timing signal is received, the new loop is discarded which may cause de-synchronisation when averaging over many cycles to remove mains frequency noise influences

#### Nu. of other channels averaged per cycle
- Register ID: `11`,
- Value: default `1`, standard `10`
- Value Range: `1 \... 255`
- This value controls the averaging for all other input channels except of the PV voltage and current and RTD temperature
    - Channels included are Offset, NTC temperatures and bias voltage
- There is no multi-cycle averaging support for those channels
- the measurements taken here are not as critical as PV current and voltage, as such the number of averages should be kept low
- Again, make sure the main measurement loop has enough idle time to run periodic control functions and execute communication functions

#### Nu. of Cycles of Voltage and Current measurements averaged
- Register ID: `12`,
- Value: default `1`, standard `4`
- Value Range: `1 \... 100`
- This parameter controls over how many consecutive cycles the PV current and voltage signals are averaged
- Averaging over multiple measurement cycles enables low frequency noise reduction at the main power frequency if configured correctly
- The standard EEPROM value is set here to `4` because the measurement timer is set at 4 times the mains frequency, hence one full mains cycle is averaged

### Measurement range settings

#### Range Control Status
- Register ID: `20`,
- Value: default `byte 0000 0000`, standard `byte 0000 0000`
- This parameter sets the range control type at start-up
- Following bit configuration is available:
    - Bit 0: Voltage Range manual mode
        - Set this bit to one to manually control the voltage range
    - Bit 1: Current Range manual mode
        - Set this bit to one to manually control the current range
    - All other bits are reserved and must be left at 0

#### Voltage measurement range
- Register ID: `21`,
- Value: default `255`, standard `255`
- Value Range: `0 ... 4, 255`
- In case of a manually controlled range at start-up, this parameter controls the voltage range ID used, otherwise this parameter has no impact on the operation
- See [Voltage Range ID](#voltage-range-id) for voltage range ID definitions

#### Current measurement range
- Register ID: `22`,
- Value: default `255`, standard `255`
- Value Range: `0 ... 5, 255`
- In case of a manually controlled range at start-up, this parameter controls the current range ID used, otherwise this parameter has no impact on the operation
- See [Current Range ID](#current-range-id) for current range ID definitions

#### Voltage range switching frequency counter
- Register ID: `23`,
- Value: default `200`, standard `200`
- Value Range: `0 ... 65535`
- This parameter controls how frequently the voltage measurement range can be lowered in auto range mode:
    - the counter is bound to the main measurement cycles
    - it must count down to zero after lowering the range once to be able to lower it again
    - thus limits the frequency of range setting to reduce oscillation between ranges in non-ideal operation conditions, such as instability
    - increasing the range value is not prohibited, to make sure a valid reading can always be provided, albeit it may not be at the ideal range
- with the above standard EEPROM settings a value of 200 results in a delay time of \~0.83s
- setting this parameter to `0` disables the functionality
- setting this parameter to <0> disables the functionality, however voltage range control does wear out the mechanical relay slowly, so limiting switching can save the relay especially in conditions where the PI controller is instable at which point the current fluctuates fast

#### Current range switching frequency counter
- Register ID: `24`,
- Value: default `200`, standard `200`
- Value Range: `0 ... 65535`
- This parameter controls how frequently the current measurement range can be lowered in auto range mode; it works same as the above voltage range switching frequency counter
    - Important difference is that the current range switch also controls mechanical latching relays, that if the frequency is too high, can wear out quite quickly
- with the above standard EEPROM settings a value of 200 results in a delay time of \~0.83s
- setting this parameter to `0` disables the functionality, however current range control does wear out the mechanical relay slowly, so limiting switching can save the relay especially in conditions where the PI controller is instable at which point the current fluctuates fast

#### Voltage range switching delay counter
- Register ID: `84`
- Value: default `100`, standard `150`
- Value Range: `0–255`
- This parameter controls the delay time until the voltage measurement range is changed
	- This delay is used to prevent range changes after a brief voltage pulse caused by step changes in the load
	- The counter is bound to the main measurement cycle

#### Current range switching delay counter
- Register ID: `85`
- Value: default `100`, standard `150`
- Value Range: `0–255`
- This parameter controls the delay time until the current measurement range is changed
	- This delay is used to prevent range changes after a brief current pulse caused by step changes in the load
	- The counter is bound to the main measurement cycle

### Voltage Range calibration factors

#### Offset value
- Register ID: `25`, `30`, `35`, `40`, `45` for voltage range ID 0 to 4
- Value: default `0.0`, standard `-648.9`
- Value Range: `single floating point`
- This parameter specifies the signal offset calibration factor
- This value is not scaled and raw in ADC counts
- The standard value of -648.9 is set here, as the ADC voltage channel is deliberately offset to enable measurements a slightly less than zero (\~1% under range)

#### Scale value
- Register ID: `26`, `31`, `36`, `41`, `46` for voltage range ID 0 to 4
- Value: default `1.0`, standard `variable`
- Value Range: `single floating point`
- This parameter specifies the signal scale calibration factor
- It scales the voltage from ADC counts to absolute values after adding of the offset value

#### Nominal absolute range value
- Register ID: `27`, `32`, `37`, `42`, `47` for voltage range ID 0 to 4
- Value: default `0.0`, standard `variable`
- Value Range: `single floating point`
- This parameter specifies the absolute range value (the range maximum)
- It is used internally to control the range switching
- Important to note is that it also controls the set-points for the calibration routine, so it is important that this value represents the range maximum correctly, as otherwise the calibrator may inject too much voltage into the system (or too little to measure the range correctly)

#### Voltage input current leakage
- Register ID: `28`, `33`, `38`, `43`, `48` for voltage range ID 0 to 4
- Value: default `0.0`, standard `variable`
- Value Range: `single floating point`
- This parameter defines amount of current leakage into the voltage measurement terminals in \[A/V\], it is dependent on the series resistance in the voltage measurement path
- The calculated absolute leakage into the voltage terminals is added to the current measurement
    - hence it needs to be reasonable accurately specified to not cause current measurement inaccuracies

###  Current Range calibration factors

#### Offset value
- Register ID: `55`, `60`, `65`, `70`, `75`, `80` for current range ID 0 to 5
- Value: default `0.0`, standard `-648.9`
- Value Range: `single floating point`
- parameter specifies the signal offset calibration factor,
- This value is not scaled and raw in ADC counts
- The standard value of -648.9 is set here, as the ADC current channel is deliberately offset to enable measurements a slightly less than zero (\~1% under range)

#### Scale value
- Register ID: `56`, `61`, `66`, `71`, `76`, `81` for current range ID 0 to 5
- Value: default `1.0`, standard `variable`
- Value Range: `single floating point`
- This parameter specifies the signal scale calibration factor
- It scales the current from ADC counts to absolute values after adding of the offset value

#### Nominal absolute range value
- Register ID: `57`, `62`, `67`, `72`, `77`, `82` for current range ID 0 to 5
- Value: default `0.0`, standard `variable`
- Value Range: `single floating point`
- This parameter specifies the absolute range value (the range maximum)
- It is used internally to control the range switching
- Important to note is that it also controls the set-points for the calibration routine, so it is important that this value represents the range maximum correctly, as otherwise the calibrator may inject too much current into the system (or too little to measure the range correctly)

### Autorange mode range enable/disable state bytes for 
- The range disable and enabled bytes for current and voltage ranges is used to disable/enable particular ranges in auto range mode
- A disabled range will not be selected when auto range is active and the next available higher range will be selected instead
- The exception is the highest range, which is the only range that is always treated as enabled (selectable)
- The range selection in manual range selection mode is not impacted

#### Voltage Range Enabled State Byte
- Register ID: `82`
- Value: default `255`, standard `255`
- Value Range: `unsigned 8-bit integer`
- Each bit in the control byte belongs a difference range given as follows:
	- Bit 0: Voltage range 0, 1V
	- Bit 1: Voltage range 1, 4.2V
	- Bit 2: Voltage range 2, 10V
	- Bit 3: Voltage range 3, 30V
	- All other bits: does not matter, best keep as True

#### Current Range Enabled State Byte
- Register ID: `83`
- Value: default `255`, standard `255`
- Value Range: `unsigned 8-bit integer`
- Each bit in the control byte belongs to a difference range given as follows:
	- Bit 0: Current range 0, lowest
	- Bit 1: Current range 1
	- Bit 2: Current range 2
	- Bit 3: Current range 3
	- Bit 4: Current range 4
	- All other bits: does not matter, best keep as True

### Bias Voltage reading calibration
- Bias voltage readings can be calibrated with the following parameters
- however, bias voltage readings are more indicative and not designed for accuracy
    - the ±100V overvoltage protection within measurement circuit causes a small non-linearity
- following conversion formula is used, whereas x is the ADC reading: $V_{b} = A0 + x*A1$

#### Offset value (A0)
- Register ID: `90`
- Value: default `0.0`, standard `0.0`
- Value Range: `single floating point`
- parameter specifies the signal offset calibration factor
- the offset value is scaled on the bias voltage channel and added to the ADC measurement after the scale factor is applied

#### Scale value (A1)
- Register ID: `91`
- Value: default `1.0`, standard `0.000125`
- Value Range: `single floating point`
- This parameter specifies the signal scale calibration factor
- It scales the voltage from ADC counts, offset is added after

### RTD PV device temperature calibration factors
- Register ID: A0 `95`, A1 `96` and A2 `97`
- Value: default A0 `0.0`, A1 `1.0` and A2 `0.0`
- Value: standard A0 `-2.45681E+02`, A1 `3.09122E-02` and A2 `1.74219E-07`
- Value Range: `single floating point`
- The RTD calibration factors are used to convert the raw RTD ADC readings in counts to temperature in \[°C\]
- The calibration values are only used when the Adafruit RTD PT100 amplifier (Max31865) is used
- Following formula is used to calculate the absolute temperature: $T = A0 + x*A1 + x^{2}*A2$

### NTC temperature calibration factors

#### Inverse Gain Value
- Register ID: NTC1 `100`, NTC2 `105`
- Value: default ` 1.66945E-01`, standard ` 0.465116`
- Value Range: `single floating point`
- Defines the inverse / reciprocal gain of the NTC signal amplifier

#### Series Resistance Value
- Register ID: NTC1 `101`, NTC2 `106`
- Value: default `1E05`, standard `5E04`
- Value Range: `single floating point`
- Defines the series resistance between reference voltage output and NTC sensor

#### Inverse Resistance at 25°C
- Register ID: NTC1 `102`, NTC2 `107`
- Value: default `1.0E-04`, standard `1.0E-04`
- Value Range: `single floating point`
- Defines the inverse / reciprocal resistance of the NTC sensor at 25°C

#### Inverse Beta
- Register ID: NTC1 `103`, NTC2 `108`
- Value: default ` 2.91545E-04`, `2.50752E-04`, standard ` 2.91121E-04`, `2.50752E-04`
- Value Range: `single floating point`
- Defines the inverse / reciprocal beta value of the NTC sensor

### Control Voltage DAC calibration factors
- The DAC calibration factors can be used to improve the control accuracy between voltage setting and its actual measurement
- The actual scale of the DAC set-point reference depends on the actual voltage range, however for all voltage ranges only one DAC calibration is applied as follows:
    - ${Count}_{DAC} = \frac{V_{set}}{V_{range\ scale}}*A1 + A0$
    - This formula calculates the raw counts required at the DAC to reach the requested set-point

#### Offset value A0
- Register ID: `110`
- Value: default `648.0`, standard `648.9`
- Value Range: `single floating point`
- parameter specifies the offset between setpoint and measurement, it is in raw ADC counts
    - the voltage measurement channel is offset by \~648.9 ADC counts

#### Scale value
- Register ID: `111`
- Value: default `1.0`, standard `1.0`
- Value Range: `single floating point`
- This parameter specifies the scale calibration / correction factor

### PI-Controller configuration

#### Gain Resistor ID
- Register ID: `115`
- Value: default `0`, standard `1`
- Value Range: `0 ... 3`
- This parameter configures the select gain resistor of the PI controller at start-up, see [Driver Proportional Gain Control](#driver-proportional-gain-control) for more information

#### Integrator capacitor ID
- Register ID: `116`
- Value: default `0`, standard `1`
- Value Range: `0 ... 3`
- This parameter configures the selected integrator capacitor of the PI controller at start-up, see [Driver Integrator Capacitor Control](#driver-integrator-capacitor-control) for more details

### Operating Temperature limits configuration

#### Low temperature disconnect
- Register ID: NTC1 `125`, NTC2 `130`
- Value: default `0.0`, standard `0.0`
- Value Range: `single floating point`
- This is the automatic disconnect (switch off) temperature at low NTC1 and NTC2 temperature
- NOTE: At current version, any value below of \~−4°C cannot be reached as the NTC resistance is too high and overflowing the ADC input

#### High temperature disconnect
- Register ID: NTC1 `126`, NTC2 `131`
- Value: default NTC1 `40.0`, NTC2 `80.0`; standard NTC1 `45.0`, NTC2 `80.0`
- Value Range: `single floating point`
- This is the automatic disconnect (switch off) temperature at high NTC1 and NTC2 temperature
- IMPORTANT: the NCT temperature measured on the heatsink especially for NTC2 at the driver MOSFET does only measure the outer heatsink temperature and not the core MOSFET junction temperature.
    - Example for low current devices: NTC2 measures 80°C, the MOSFET junction is \~130°C
    - Hence do NOT set the temperate too high, as it may cause immediate damage or reduce lifetime of the MOSFET

#### Low temperature reconnect
- Register ID: NTC1 `127`, NTC2 `132`
- Value: default `5.0`, standard `5.0`
- Value Range: `single floating point`
- This is the automatic re-connect (switch back on) temperature after a low NTC1 and NTC2 temperature disconnect
- This parameter must be larger than the low temperature disconnect value
- NOTE: as detailed above, at current version any value below of \~−4°C cannot be reached as the NTC resistance is too high and overflowing the ADC input

#### High temperature reconnect
- Register ID: NTC1 `128`, NTC2 `133`
- Value: default NTC1 `35.0`, NTC2 `60.0`; standard NTC1 `40.0`, NTC2 `50.0`
- Value Range: `single floating point`
- This is the automatic reconnect (switch back on) temperature after a high NTC1 and NTC2 temperature disconnect
- This parameter must be lower than the high temperature disconnect value

### Bias Voltage Limits Protection

#### Minimum Bias voltage
- Register ID: `135`
- Value: default `15000` \[counts\], standard `1.0` \[V\]
- Value Range: `single floating point`
- This sets the minimum bias voltage at which the OPET output can operate
- If this threshold is undercut, the output will automatically turn off
- The device will turn automatically back on if the bias voltage is within range continuously for \~5s
- When changing this value to below 0 V, the function is effectively disabled

#### Maximum Bias voltage
- Register ID: `136`
- Value: default `45000` \[counts\], standard `6.0` \[V\]
- Value Range: `single floating point`
- This sets the maximum bias voltage at which the OPET output can operate
- Function is same as detailed above, when exceeding the threshold
- When changing this value to above 100 V, the function is effectively disabled

### IV curve measurement settings

#### Number of IV points
- Register ID: `140`
- Value: default `10`, standard `100`
- Value Range: `3 ... 250`
- This sets the number of IV point taken in IV curve and transient measurements

#### IV measurement mode byte
- Register ID: `141`
- Value: default `byte 0000 0000`, standard `byte 0000 0011`
- This parameter determines the IV measurement mode settings at start-up
    - Cos IV point distribution and asymmetric voltage readings are enabled by default
- See [IV measurement mode](#iv-measurement-mode) for the binary bit definitions and more details

#### COS sweep maximum phase angle
- Register ID: `142`
- Value: default ` 1.57079`, standard ` 1.57079`
- Value Range: `single floating point`
- As further detailed in [Cosine maximum phase in radians](#cosine-maximum-phase-in-radians) this parameter defines the maximum phase angle in radians used during cosine distributed IV curve measurements

#### V<sub>oc</sub> overshoot control multiplier
- Register ID: `143`
- Value: default ` 1.01`, standard ` 1.01`
- Value Range: `single floating point`
- As further detailed in [VOC overshoot control multiplier](#voc-overshoot-control-multiplier), this multiplier is applied to the measured V<sub>oc</sub> before IV measurements and sets the IV curve endpoint control voltage at the voltage control DAC

#### IV point settling time delay
- Register ID: `144`
- Value: default `5`, standard `5`
- Value Range: `1 ... 60000`
- This parameter sets the delay time in \[ms\] between setting the output voltage till measurement of the voltage and current of the PV device, see [IV point settling time delay in ms](#iv-point-settling-time-delay-in-ms) for more details

#### Nu. measurements averaged per set
- Register ID: `145`
- Value: default `1`, standard `50`
- Value Range: `1 ... 255`
- Controls the number of current and voltage measurements averaged for each IV point in a single set, see [IV data Averaging control](#iv-data-averaging-control) for more details

#### Nu. sets averaged per IV point
- Register ID: `146`
- Value: default `1`, standard `1`
- Value Range: `1 ... 255`
- Controls the number of current and voltage measurement sets averaged for each IV point, see [IV data Averaging control](#iv-data-averaging-control) for more details

### Maximum power point tracker control variables

#### Maximum step size 
- Register ID: `150`
- Value: default `1000.0`, standard `200.0`
- Value Range: `single floating point within a 16-bit integer range, 1 ... 65535`
- This parameter sets the maximum control voltage step size per control cycle in DAC counts and hence automatically scales with the active voltage measurement range
- It essentially controls the step size the MPPT tracer takes to reach the P<sub>MAX</sub> at the start
- Using a too high value here may cause oscillations around the maximum power point
- Using a too low value here means it will take a long time for the P<sub>MAX</sub> to be reached or it may get stuck due to signal noise
- This parameter must be larger than the minimum step size

#### Minimum step size 
- Register ID: `151`
- Value: default `10.0`, standard `3.0`
- Value Range: `single floating point within a 16-bit integer, 1 ... 65535`
- This parameter sets the minimum control voltage step size per control cycle in DAC counts and hence automatically scales with the active voltage measurement range
- This controls the step size applied after the MPPT has reached P<sub>MAX</sub> and is "hovering" around this point
- Using a too high value here causes large oscillations around the maximum power point and low tracking accuracy
- Using a too low value here may cause the MPPT to get stuck elsewhere due to signal noise
- This parameter must be lower than the maximum step size

#### Step size increase multiplier 
- Register ID: `152`
- Value: default `1.2`, standard `1.2`
- Value Range: `single floating point`
- The step size increase multiplier determines the increase in step size per control cycle when the MPPT tracer has lost its P<sub>MAX</sub> tracking, it essentially controls how fast it will get back to maximum step size
- The percentual increase in step size (here standard is 20%) should be lower than the percentual reduction in step size to assure optimal MPPT control

#### Step size reduction multiplier 
- Register ID: `153`
- Value: default `0.6`, standard `0.6`
- Value Range: `single floating point`
- The step size reduction multiplier determines the reduction in step size per control cycle when the MPPT tracer has found the P<sub>MAX</sub> point, it essentially controls how fast it will reduce to minimum step size
- The percentual reduction in step size (here standard is 40%) should be larger than the percentual increase in step size and less than 50% to assure optimal MPPT control

#### Power signal noise tolerance range (no adjust range)
- Register ID: `154`
- Value: default `1000.0`, standard `300.0`
- Value Range: `single floating point`
- Parameters is scaled in ADC counts, meaning it scales automatically with changing voltage and current ranges, this parameter is also squared internally to make up the full range of power noise (current \* voltage noise)
- The signal noise tolerance value determines over which range the MPPT tracer "ignores" new power measurements and continues moving the same direction and step size until the noise tolerance threshold range is either exceeded or succeeded and then decides again based on the resulting new power value
    - Basically, a no adjust range
- This parameter is designed the reduce the chance of the MPPT tracer getting stuck in noisy environments
- A large value results in the MPPT wandering around the P<sub>MAX</sub> point continuously
- A too low value may result in the tracker not tracking around the exact P<sub>MAX</sub>.

### Current tracker control variables

#### Maximum step size 
- Register ID: `160`
- Value: default `1000.0`, standard `200.0`
- Value Range: `single floating point within a 16-bit integer, 1 ... 65535`
- This parameter sets the maximum control voltage step size per control cycle in DAC counts and hence automatically scales with the active voltage measurement range
- It essentially controls the step size the current tracker takes to reach the current set-point at the start
- Using a too high value here may cause oscillations around the current set-point
- Using a too low value here means it will take a long time to reach the current set-point
- This parameter must be larger than the minimum step size

#### Minimum step size 
- Register ID: `161`
- Value: default `1.0`, standard `1.0`
- Value Range: `single floating point within a 16-bit integer, 1 ... 65535`
- This parameter sets the minimum control voltage step size per control cycle in DAC counts and hence automatically scales with the active voltage measurement range
- This controls the step size applied after the current tracker has reached current set-point and is "hovering" around this point
- Using a too high value here causes large oscillations around the current set-point and low tracking accuracy
- Using a value below 1.0 here, may cause too slow response to change PV output
- This parameter must be lower than the maximum step size

#### Step size increase multiplier 
- Register ID: `162`
- Value: default `1.2`, standard `1.2`
- Value Range: `single floating point`
- The step size increase multiplier determines the increase in step size per control cycle when the current tracer has lost its current set-point, it essentially controls how fast it will get back to maximum step size
- The percentual increase in step size (here standard is 20%) should be lower than the percentual reduction in step size to assure optimal control

#### Step size reduction multiplier 
- Register ID: `163`
- Value: default `0.6`, standard `0.6`
- Value Range: `single floating point`
- The step size reduction multiplier determines the reduction in step size per control cycle when the current tracer has reached the set-point current, it essentially controls how fast it will reduce to minimum step size
- The percentual reduction in step size (here standard is 40%) should be larger than the percentual increase in step size and less than 50% to assure optimal control

#### Current signal noise tolerance range 
- Register ID: `164`
- Value: default `10.0`, standard `5.0`
- Value Range: `single floating point`
- Parameters is scaled in ADC counts, meaning it scales automatically with changing current range
- The current signal noise tolerance value determines the current set-point tolerance
- If the current set-point is reached within the given tolerance, the current-tracer will stop updating the reference voltage on the DAC until it goes out of tolerance again
- This parameter is designed the reduce the number of DAC control updates
- A large value result in inaccurate current tracing
- A too low value should not have a negative impact, but means the apparent noise on voltage and current caused by constant voltage set-point updates is larger

### Auto-Start Load configuration

#### System Control Byte
- Register ID: `165`
- Value: default `byte 0000 0000`, standard `byte 0000 0000`
- Defines the operation mode entered after booting of the OPET device
- Following bit definitions are available:
    - Bit 0: Output on/off
        - Used to enable or disable the output after start-up, set to `1` to enable the output and enter a PV load mode without extra intervention
    - All other bits are reserved and should be left false

#### Load mode ID
- Register ID: `166`
- Value: default `0`, standard `0`
- Value Range: `0 ... 4`
- This defines the auto-start load mode ID, see details in [Load Mode control](#load-mode-control) for load mode ID definitions
- To make use of the auto-start function define the load mode desired and enable the output on bit 0 in the system control byte

#### Voltage set-point
- Register ID: `167`
- Value: default `0.0`, standard `0.0`
- Value Range: `single floating point`
- The voltage set-point in \[V\] used if the device is static voltage load mode

#### Current set-point 
- Register ID: `168`
- Value: default `0.0`, standard `0.0`
- Value Range: `single floating point`
- The current set-point in \[A\] if the device is controlled in static current load mode

### Fan control configuration

#### Fan Control Mode
- Register ID: `175`
- Value: default `byte 0000 0000`, standard `byte 0000 0000`
- Defines the fan operation mode after booting of the OPET device
- Following bit definitions are available:
    - Bit 0: Reserved, must be 0
    - Bit 1: Fan enable request: 1 -- on, 0 - off
    - Bit 2: Reserved, must be 0
    - Bit 3: Reserved, must be 0
    - Bit 4: Reserved, must be 0
    - Bit 5: Reserved, must be 0
    - Bit 6: Reserved, must be 0
    - Bit 7: Fan manual mode: 1 -- manual mode, 0 -- automatic controlled
- If all bits are left false, the fan control runs in automatic temperature control mode
- If bit 7 is true, it sets the fan into manual control mode
    - In manual mode, the state of bit 1 defines if the fan is left off or on until a new command changes state

#### Fan On Temperature 
- Register ID: NTC 1 `176`, NTC 2 `178`
- Value: default `30.0`, standard `35.0`
- Value Range: `single floating point`
- The temperature above which the fan will turn on if controlled in automatic mode
- This value must be above the fan off temperature value of the given NTC sensor
    - The temperature range between the on and off value of the given NTC sensor is the hysteresis
- If this value is set very high, it essentially disables automatic control of the fan dependent on that specific NTC sensor

#### Fan Off Temperature 
- Register ID: NTC 1 `177`, NTC 2 `179`
- Value: default `25.0`, standard `30.0`
- Value Range: `single floating point`
- The temperature below which the fan will turn off if controlled in automatic mode
- This value must be below the fan on temperature value of the given NTC sensor
    - The temperature range between the on and off value of the given NTC sensor is the hysteresis

#### Fan On Power 
- Register ID: `180`
- Value: default `10.0`, standard `10.0`
- Value Range: `single floating point`
- The PV load power above which the fan will turn on if controlled in automatic mode
- This value must be above the fan off power value
    - The range between the on and off value of the given NTC sensor is the hysteresis
- If this value is set very high, it essentially disables automatic control of the fan dependent on PV load power

#### Fan Off Power 
- Register ID: `181`
- Value: default `5.0`, standard `5.0`
- Value Range: `single floating point`
- The PV load power below which the fan will turn off if controlled in automatic mode
- This value must be below the fan on power value
    - The range between the on and off value of the given is the hysteresis

#### Fan switch frequency counter 
- Register ID: `182`
- Value: default `2400`, standard `4800`
- Value Range: `1 … 65535` 
- This controls how fast the automated fan control is allowed cycle the fan power on and off again and hence can reduce wear on the control relay
- Determines the minimum time until fan can be switched off
	- Counter value is counted down at every control cycle (default every \~25ms)
	- 2400 counter value is hence every \~60s

# Basic 2-point calibration
- In principle follow the steps detailed in the previous section as adequate, but take measurements and calculate calibration factors manually or with a suitably programmed software routine
- Following sections detail the setting, formulas and processed used in the calibration software as a guide

## OPET calibration measurement settings
- It is advised to use a very high number of voltage and current averages to lead to an estimated measurement completion time of \~1s (including voltage and current)
- Following setting are using in the calibration software:
    - use 100 current and voltage measurements average per cycle (`ADC:AVR:VC 100`, see [Number of Voltage and Current measurement averaged](#number-of-voltage-and-current-measurement-averaged) for more details)
    - use a 100 current and voltage cycle averaging (`ADC:CYCLES:VC 100`, see [Number of Voltage and Current cycles averaged](#number-of-voltage-and-current-cycles-averaged))
    - this leads to an update time of \~ 1s, the OPET cannot really operate under those conditions, but it measures at very low noise if there are no excessive external noise sources
    - settings do cause every 2nd cycle to be skipped as the averaging per cycle takes longer, which means a cycle is twice as long (with standard EEPROM settings 120Hz cycle update not 240Hz)

## Calibration and adjustment procedure of a single range
1. Connect the calibration cables required for the range and board type to be calibrated
1. Sent a reset command to the OPET `\*RST` and wait until is rebooted (\~1s)
1. Set the PV current and voltage measurement range ID to `0`, meaning you enter manual mode on both channels at the lowest range (`RANGE:IDVOLT 0` and `RANGE:IDCURR 0`)
1. Set the desired range (voltage or current) to the range ID that is to be calibrated
1. Enter calibration mode for the range type to be calibrated (current or voltage) using either `RANGE:CAL:MODE VOLT` or `RANGE:CAL:MODE CURR`
1. Read the active range values with `RAGE:ACTVAL?` and depending on the range type to be calibrated use the voltage or current range value as the maximum source value for the calibration (scale value)
    - i.e. for current range calibration with a OPET reply of ` RAGE:ACTVAL 1.0 0.15`, use 0.15 A as the scale value for the 0.15 A range to be calibrated
1. Update the measurement averaging configuration as required
1. First measure at maximum scale: Set the calibrating source to maximum, wait to stabilize and record the Calibrator/DMM and OPET readings
    - The OPET readings will be in ADC counts, meaning they may be as much as 63000 depending on range
    - Repeat measurements and average to achieve a better accuracy
1. Second measure the offset at zero: Set the calibrating source to zero, wait to stabilize and record the Calibrator/DMM and OPET readings
    - The OPET readings will be in ADC counts, a normal offset reading is \~650 counts
    - Repeat measurements and average to achieve a better accuracy
1. Calculate the calibration scale and offset factors
    - Use following formulas with *S* for scale measurement, *O* for offset measurement, *ref* from the reference calibrator / DMM and *dut* from the target OPET
    - *Scale* factor: $Scale = \frac{(S_{ref} - O_{ref})}{(S_{dut} - S_{dut})}$
    - *Offset* factor: $Offset = \frac{( - Scale*\ O_{dut} + O_{ref})}{Scale}$
1. Check the deviation between old and new scale factors
    - Read the original scale and offset calibration factors using `RANGE:CAL:SCALE?` and `RANGE:CAL:OFFSET?`
    - Calculate the percentual deviation to the scale of each
        - Scale deviation: ${DEV}_{scale} = 100*\frac{({Scale}_{old} - {Scale}_{new})}{{Scale}_{new}}$
        - Offset deviation: ${DEV}_{offset} = 100*\frac{({Offset}_{old} - {Offset}_{new})}{{Scale}_{new}}$
    - Both should not be out by more than 2% of scale in case of a first calibration from standard EEPROM values, and much lower in case of a recalibration
        - If the deviation is excessive, there may be a problem in measurement set-up, cabling, the current bypass may have activated (reset manually)
        - In case of an active current bypass, check the maximum scale is correct for the range and board type and only if OK try to reset the current clamp after setting the maximum scale value on the calibration source as there may be an initial short current spike causing it to trigger
1. If happy with the results send the new calibration factors to the OPET unit using `RANGE:CAL:SCALE \[*Scale*\]` and `RANGE:CAL:OFFSET \[*Offset*\]`
    - Make sure you use floating point formatting with exponent such as "6.489634E-02", up to \~ 7 digits accuracy can be represented with a single floating-point number this way
1. Exit calibration mode using `RANGE:CAL:MODE 0` to finish the calibration of this range
1. Reset the OPET device using `\*RST`

## Validation procedure of a single range
- The validation of the calibration is pretty much the same as the calibration with the following exceptions:
    - Do not enter calibration mode
    - Instead of reading the original calibration factors, use ideal factors as *Scale = 1* and *Offset = 0* and calculate the percentual deviations to scale with those.
    - Do not set any calibration factors
    - Deviation between ideal and calculated factors should be well within ±0.01%

# Firmware details
This document contains further information with regards to the microcontroller (MCU) source code structure and programming of the MCU on the OPET board
- Software and hardware requirements
- Source code overview
- MCU programming / debugging
If you just wish to load the stock firmware package, proceed to the MCU programming section below.

# Disclaimer

DISCLAIMER: NREL/ALLIANCE FOR SUSTAINABLE ENERGY, LLC/DOE DISCLAIM ALL WARRANTIES, EXPRESS OR IMPLIED, INCLUDING THE WARRANTIES OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE, AND MAKES NO WARRANTY AS TO THE ACCURACY, COMPLETENESS, OR USEFULNESS OF ANY INFORMATION PROVIDED HEREIN. USE OF THIS PACKAGE IS AT THE USER’S OWN RISK.