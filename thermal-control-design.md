# SONiC Thermal Control Design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Liu Kebo      | Initial version                   |


  
## 1. Overview

Thermal control is to keep the switch at the proper temperature by using cooling devices, e.g., fan.
Thermal control need to monitor the temperature of devices (CPU, ASIC, ports, etc) to know the thermal status of the switch and to manipulate the fan speed accordingly.

## 2. Thermal device monitoring

Thermal monitoring function need to retrieve the switch temperatures via platform APIs, platform APIs not only provide the value of the temperature, but also provide the threshold value, so can know the thermal object status by comparing the current temperature value against the threshold. If it above and under the threshold, alarm shall be raised.

Besides device temperature, should also monitoring the fan speed to make sure if the speed is a proper value according to current thermal condition 

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

an example for a temperature object in DB

    
    key                     = TEMPERATURE_INFO|CPU   
    ; field                 = value
    temperature             = "75"                                             
    high_threshold          = "110"                         
    low_threshold           = "-10"                          
    warning_status          = "true"                        
    

### 2.2 Fan monitoring 

In most case fan is the device to cool down the switch if the temperature rising. Thus to make sure fan running at a proper speed is the key for thermal control.

Fan target speed and speed tolerance was defined, make use of them we can know whether the fan reached and running at the desired speed.

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

If there is temperature above threshold and warning flag set to DB, log also shall be generated to the syslog.

## 3 Thermal control framework proposal for SONiC

Thermal control is quite vendor specific, maybe vendors already have their own implementation, in below appendix chapter described a Mellanox implementation and which can be take as an reference for a common platform framework. 

Use similar methodology to implement one PMON daemon container, to monitor the thermal device and manipulate the cooling devices.

The temperature and fan speed defined for the trip point in below chapter shall be configurable, so different vendors can adapt to the framework. 

And the thermal control itself can be disabled if the vendor have a in house thermal control implementation.


## 4. Open Questions

## Appendix

## Mellanox thermal control implementation

Mellanox thermal monitoring measure temperature from the ports and ASIC core. It operates in kernel space and binds PWM(Pulse-Width Modulation) control with Linux thermal zone for each measurement device (ports & core). The thermal algorithm uses step_wise policy which set FANs according to the thermal trends (high temperature = faster fan; lower temperature = slower fan).

### 1.1 Mellanox thermal Control framework 

The kernel provides a framework for thermal control. It defines a set of concepts, such as thermal zones, trip points, cooling devices, thermal instances, thermal governors. The kernel thermal algorithm uses step_wise policy which set FANs according to the thermal trends (high temperature = faster fan; lower temperature = slower fan). 

### 1.2 Components

- The cooling device is an actual functional unit for cooling down the thermal zone: Fan.

- Thermal instance describes how cooling devices work at certain trip point in the thermal zone.

- Governor handles the thermal instance not thermal devices. Step_wise governor sets cooling state based on thermal trend (STABLE, RAISING, DROPPING, RASING_FULL, DROPPING_FULL). It allows only one step change for increasing or decreasing at decision time. Framework to register thermal zone and cooling devices:

- Thermal zone devices and cooling devices will work after proper binding. Performs a routing function of generic cooling devices to generic thermal zones with the help of very simple thermal management logic.

### 1.3 Algorithm

Use step_wise policy for each thermal zone. Set the fan speed according to different trip points. 

### 1.4 Trip points

a series of trip point is defined to trigger fan speed manipulate.

	Cold,      t < 75 C,       fan pwm speed at 20%                 Do nothing
	
	Normal,    75 <= t < 85,   fan speed at pwm speed 20% - 40%     keep minimal speed
	
	High,      85 <= t < 105   fan speed at pwm speed 40% - 100%    adjust the fan speed according to the trends
	
	Hot        105 <= t < 110  fan speed at 100%                    produce warning
	
	Critical   t >= 110        fan speed at 100%                    shutdown

### 1.5 User space daemon

Mellanox hw-management package include a user space daemon to change the fan speed according to some external conditions:

1. set fan to full speed if one PSU not present and disable the thermal monitoring in kernel until PSU recovered.
2. set fan to full speed if one fan not present and disable the thermal monitoring in kernel until fan recovered.
3. If thermal control was disabled then set the fan speed to 60%.
	
