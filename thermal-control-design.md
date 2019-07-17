# SONiC Thermal Control Design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Liu Kebo      | Initial version                   |


  
## 1. Overview

The purpose of Thermal Control is to keep the switch at a proper temperature by using cooling devices, e.g., fan.
Thermal control daemon need to monitor the temperature of devices (CPU, ASIC, optical modules, etc) and the running status of fan, at the same time it need to manipulate the cooling device according to the temperature and related device status.

## 2. Thermal device monitoring

Thermal monitoring function will retrieve the switch device temperatures via platform APIs, platform APIs not only provide the value of the temperature, but also provide the threshold value. The thermal object status can be deduced by comparing the current temperature value against the threshold. If it above the high threshold or under the low threshold, alarm shall be raised.

Besides device temperature, shall also monitoring the fan speed to make sure it is running at a proper speed according to current thermal condition. 

Thermal device monitoring will loop at a certain period, 60s can be a good value since usually temperature don't change much in certain short period.

## 2.1 Temperature monitoring 

In new platform API ThermalBase() class provides get_temperature(), get_high_threshold(),  get_low_threshold(), get_critical_high_threshold() and get_critical_low_threshold() functions, values for a thermal object can be fetched from them. Warning status can also be deduced. 

For the purpose of feeding CLI/SNMP or telemetry functions, these values and warning status can be stored in the state.  DB schema can be like this:

    ; Defines information for a thermal object
    key                     = TEMPERATURE_INFO|object_name   ; name of the thermal object(CPU, ASIC, optical modules...)
    ; field                 = value
    temperature             = FLOAT                          ; current temperature value                        
    high_threshold          = FLOAT                          ; temperature high threshold
    critical_high_threshold = FLOAT                          ; temperature critical high threshold
    low_threshold           = FLOAT                          ; temperature low threshold
    critical_low_threshold  = FLOAT                          ; temperature critical low threshold
    warning_status          = BOOLEAN                        ; temperature warning status

These devices shall be included to the temperature monitor list but not limited to: CPU core, CPU pack, ASIC, PSU, Optical Modules, etc.

TEMPERATURE_INFO Table key object_name convention can be "device_name + index"  or device_name if there is no index, like "cpu_core_0", "asic", "psu_2".  Appendix 1 listed all the thermal sensors that supported on Mellanox platform.
    
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

If there was warning raised or warning cleared, log shall be generated:

	High temperature warning: PSU 1 current temperature 85C, high threshold 80C！
	High temperature warning cleared, PSU1 temperature restore to 75C, high threshold 80C

If fan broken or become up present, log shall be generated：

	Fan removed warning: Fan 1 was removed from the system, potential overheat hazard!
	Fan removed warning cleared: Fan 1 was inserted.

## 3. Cooling device control framework proposal for SONiC

Another part of the Thermal Control daemon is to handle the cooling device.

this part could be very vendor specific, maybe some vendors already have their own implementation. In below Appendix chapter describes a Mellanox implementation, it's including functions inside kernels and policies in user space. For the user space policies can be extracted out and make it as part of a generic cooling device control framework. 

The user space functions is mainly periodically checking the fan and PSU status and take action accordingly. If some fan or PSU was broken or removed, it will suspend the thermal control functions inside kernel and set the fan speed to max, until the fan or PSU restored. 

Besides these polices, as a generic cooling device control framework, there should be some functions which check the device temperature periodically and take action(speed up, slow down the fan or keep the current fan speed) according to some algorithm. The function shall leave to vendors to implement, it's hard to invent a common one since the system hardware cooling design are various.

This cooling device control function can be disabled if the vendor have their own implementation in the kernel or somewhere else.

### 3.1 Cooling device control framework

It will be a routing function to check the policies and the perform the algorithm to adjust the fan speed according real-time temperatures.


Two policy check functions will check the FAN and PSU presence and return actions need to take. For example, return value {"fan_1":"60", "fan_2":"None"} means set fan_1 speed to 60% and keep fan_2 current speed.

Vendor specific init function and clean up function will be performed at the beginning of the task and at then end of the task.

Above vendor specific functions will be added to a thermal_control base class and leave them for vendor implementation.

![](https://github.com/keboliu/SONiC/blob/master/images/thermal-control.svg)



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
	NAME    Temperature    High Threshold    Low Threshold    Critical High Threshold   Critical Low Threshold   Warning Status
	-----  -------------  ----------------  ---------------  ------------------------- ------------------------ ----------------
	CPU        85              110              -10                 120                        -20                    false
	ASIC       75              100               0                  110                        -10                    false

An option '--major' provided by this CLI to only print out major device temp, if don't want show all of sensor temperatures.
Major devices are CPU pack, cpu cores, ASIC and optical modules.

## 5. Open Questions


## Appendix

## 1.Mellanox platform thermal sensors list

On Mellanox platform we have below thermal sensors that will be monitored by the thermal control daemons, not all of the Mellanox platform include all of them, some platform maybe only have a subset of these thermal sensors.

        cpu_core_x : "CPU Core x Temp", 
        cpu_pack : "CPU Pack Temp",
        modules_x : "xSFP module x Temp",
        psu_x : "PSU-x Temp",
        gearbox_x : "Gearbox x Temp"
        asic : "Ambient ASIC Temp",
        port : "Ambient Port Side Temp",
        fan : "Ambient Fan Side Temp",
        comex : "Ambient COMEX Temp",
        board : "Ambient Board Temp"
 

## 2.Mellanox thermal control implementation

### 2.1 Mellanox thermal Control framework

Mellanox thermal monitoring measure temperature from the ports and ASIC core. It operates in kernel space and binds PWM(Pulse-Width Modulation) control with Linux thermal zone for each measurement device (ports & core). The thermal algorithm uses step_wise policy which set FANs according to the thermal trends (high temperature = faster fan; lower temperature = slower fan). 

More detail information can refer to Kernel documents https://www.kernel.org/doc/Documentation/thermal/sysfs-api.txt
and Mellanox HW-management package documents: https://github.com/Mellanox/hw-mgmt/tree/master/Documentation

### 2.2 Components

- The cooling device is an actual functional unit for cooling down the thermal zone: Fan.

- Thermal instance describes how cooling devices work at certain trip point in the thermal zone.

- Governor handles the thermal instance not thermal devices. Step_wise governor sets cooling state based on thermal trend (STABLE, RAISING, DROPPING, RASING_FULL, DROPPING_FULL). It allows only one step change for increasing or decreasing at decision time. Framework to register thermal zone and cooling devices:

- Thermal zone devices and cooling devices will work after proper binding. Performs a routing function of generic cooling devices to generic thermal zones with the help of very simple thermal management logic.

### 2.3 Algorithm

Use step_wise policy for each thermal zone. Set the fan speed according to different trip points. 

### 2.4 Trip points

a series of trip point is defined to trigger fan speed manipulate.

 |state   |Temperature value(Celsius) |PWM speed                  |Action                                    |
 |:------:|:-------------------------:|:-------------------------:|:-----------------------------------------|
 |Cold    |      t < 75 C    | 20%        |   Do nothing                             |
 |Normal  |    75 <= t < 85  | 20% - 40%  |   keep minimal speed|
 |High    |  85 <= t < 105   | 40% - 100% | adjust the fan speed according to the trends|
 |Hot     |  105 <= t < 110  | 100%       | produce warning message                     |
 |Critical|  t >= 110        | 100%       |  shutdown |

### 2.5 Thermal control Policies 

Besides kernel’s feature, Mellanox hw-mgmt package provides a userspace daemon to check the status of thermal zone and change the fan speed according to the following policies:

- Set PWM to full speed if one of PS units is not present 

- Set PWM to full speed if one of FAN drawers is not present or one of tachometers is broken present 

- Set the fan speed to a consant value (60% of full speed) thermal control was disabled.

