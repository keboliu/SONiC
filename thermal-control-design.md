# SONiC Thermal Control Design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Liu Kebo      | Initial version                   |


  
## 1. Overview

The purpose of Thermal Control is to keep the switch at a proper temperature by using cooling devices, e.g., fan.
Thermal control daemon need to monitor the temperature of devices (CPU, ASIC, ports, etc) and the running status of fan, at the same time it need to manipulate the cooling device according to the temperature and related device status.

## 2. Thermal device monitoring

Thermal monitoring function will retrieve the switch device temperatures via platform APIs, platform APIs not only provide the value of the temperature, but also provide the threshold value. The thermal object status can be deduced by comparing the current temperature value against the threshold. If it above the high threshold or under the low threshold, alarm shall be raised.

Besides device temperature, shall also monitoring the fan speed to make sure it is running at a proper speed according to current thermal condition. 

Thermal device monitoring will loop at a certain period, 15s can be a good value according the implementation on Mellanox platform.

## 2.1 Temperature monitoring 

In new platform API ThermalBase() class provides get_temperature(), get_high_threshold() and get_low_threshold() functions, values for a thermal object can be fetched from them. Warning status can also be deduced. 

For the purpose of feeding CLI/SNMP or telemetry functions, these values and warning status can be stored in the state.  DB schema can be like this:

    ; Defines information for a thermal object
    key                     = TEMPERATURE_INFO|object_name   ; name of the thermal objec(CPU, ASIC, Ports...)
    ; field                 = value
    temperature             = FLOAT                          ; current temerature value                        
    high_threshold          = FLOAT                          ; temperature high threshold
    low_threshold           = FLOAT                          ; temperature low threshold
    warning_status          = BOOLEAN                        ; temperature warning status
    
### 2.2 Fan device monitoring 

In most case fan is the device to cool down the switch when the temperature is rising. Thus to make sure fan running at a proper speed is the key for thermal control.

Fan target speed and speed tolerance was defined, by examining them we can know whether the fan reached at the desired speed.

same as the temperature info, a [table for fan](https://github.com/Azure/SONiC/blob/master/doc/pmon/pmon-enhancement-design.md#153-fan-table) also defined as below:

	; Defines information for a fan
	key                     = FAN_INFO|fan_name              ; information for the fan
	; field                 = value
	presence                = BOOLEAN                        ; presence of the fan
	model                   = STRING                         ; model name of the fan
	serial                  = STRING                         ; serial number of the fan
	status                  = BOOLEAN                        ; status of the fan
	change_event            = STRING                         ; change event of the fan
	direction               = STRING                         ; direction of the fan 
	speed                   = INT                            ; fan speed
	speed_tolerance         = INT                            ; fan speed tolerance
	speed_target            = INT                            ; fan target speed
	led_status              = STRING                         ; fan led status

### 2.3 Syslog for thermal control

If there was warning raised inside Thermal Control system, or some significant action performed, log shall be generated to the syslog.

## 3. Cooling device manipulate logic proposal for SONiC

Another part of the Thermal Control daemon is to handle the cooling device.

this part could be very vendor specific, maybe some vendors already have their own implementation, in below Appendix chapter described a Mellanox implementation. It can be taken as an reference for a common cooling device control framework. 

If take Mellanox similar methodology to implement the common framework, the temperature and fan speed defined for the trip point in below Appendix chapter shall be specific to each vendor's device, so different vendors can adapt to the common framework. 

And the cooling device manipualte funtion can be disabled if the vendor have their own implementation.

## 4. CLI show command for temperature design

 adding a new sub command to the "show platform":
 
	admin@sonic# show platform ?
	Usage: show platform [OPTIONS] COMMAND [ARGS]...

	  Show platform-specific hardware info

	Options:
	  -?, -h, --help  Show this message and exit.

	Commands:
	  mlnx         Mellanox platform specific configuration...
	  psustatus    Show PSU status information
	  summary      Show hardware platform information
	  syseeprom    Show system EEPROM information
	  temperature  Show fan status information
	  
out put of the new CLI

	admin@sonic# show platform temperature
	NAME    Temperature    High Threshold    Low Threshold     Warning Status
	-----  -------------  ----------------  ---------------   ----------------
	CPU        85              110              -10                false
	ASIC       75              100               0                 false

## 5. Open Questions


## Appendix

## Mellanox thermal control implementation

### 1.1 Mellanox thermal Control framework

Mellanox thermal monitoring measure temperature from the ports and ASIC core. It operates in kernel space and binds PWM(Pulse-Width Modulation) control with Linux thermal zone for each measurement device (ports & core). The thermal algorithm uses step_wise policy which set FANs according to the thermal trends (high temperature = faster fan; lower temperature = slower fan). 

More detail information can refer to Kernel documents https://www.kernel.org/doc/Documentation/thermal/sysfs-api.txt
and Mellanox HW-management package documents: https://github.com/Mellanox/hw-mgmt/tree/master/Documentation

### 1.2 Components

- The cooling device is an actual functional unit for cooling down the thermal zone: Fan.

- Thermal instance describes how cooling devices work at certain trip point in the thermal zone.

- Governor handles the thermal instance not thermal devices. Step_wise governor sets cooling state based on thermal trend (STABLE, RAISING, DROPPING, RASING_FULL, DROPPING_FULL). It allows only one step change for increasing or decreasing at decision time. Framework to register thermal zone and cooling devices:

- Thermal zone devices and cooling devices will work after proper binding. Performs a routing function of generic cooling devices to generic thermal zones with the help of very simple thermal management logic.

### 1.3 Algorithm

Use step_wise policy for each thermal zone. Set the fan speed according to different trip points. 

### 1.4 Trip points

a series of trip point is defined to trigger fan speed manipulate.

 |state   |Temperature value(Celsius) |PWM speed                  |Action                                    |
 |:------:|:-------------------------:|:-------------------------:|:-----------------------------------------|
 |Cold    |      t < 75 C    | 20%        |   Do nothing                             |
 |Normal  |    75 <= t < 85  | 20% - 40%  |   keep minimal speed|
 |High    |  85 <= t < 105   | 40% - 100% | adjust the fan speed according to the trends|
 |Hot     |  105 <= t < 110  | 100%       | produce warning message                     |
 |Critical|  t >= 110        | 100%       |  shutdown |

	
