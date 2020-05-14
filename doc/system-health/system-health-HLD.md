# SONiC system health monitor Design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Liu Kebo      | Initial version                   |


  
## 1. Overview of the system health monitor

System health monitor is intend to monitor both of critical services and peripheral device status and leverage system health LED to indicate the status.

In current SONiC implementation, already have Monit which is monitoring the critical services status and also have a set of daemons(psud, thermaltcld) inside PMON collecting the peripheral devices status.

System health will not monitor the critical services or devices directly, it will reuse the result of Monit and PMON daemons to summary the current status and decide the color of the system health LED.

### 1.1 Services under Monit monitoring

For the Monit, now below services and file system is under monitoring:

	root@r-tigris-13:/home/admin# monit summary -B
	Monit 5.20.0 uptime: 1h 6m
	 Service Name                     Status                      Type          
	 sonic                            Running                     System        
	 rsyslog                          Running                     Process       
	 telemetry                        Running                     Process       
	 dialout_client                   Running                     Process       
	 syncd                            Running                     Process       
	 orchagent                        Running                     Process       
	 portsyncd                        Running                     Process       
	 neighsyncd                       Running                     Process       
	 vrfmgrd                          Running                     Process       
	 vlanmgrd                         Running                     Process       
	 intfmgrd                         Running                     Process       
	 portmgrd                         Running                     Process       
	 buffermgrd                       Running                     Process       
	 nbrmgrd                          Running                     Process       
	 vxlanmgrd                        Running                     Process       
	 snmpd                            Running                     Process       
	 snmp_subagent                    Running                     Process       
	 sflowmgrd                        Running                     Process       
	 lldpd_monitor                    Running                     Process       
	 lldp_syncd                       Running                     Process       
	 lldpmgrd                         Running                     Process       
	 redis_server                     Running                     Process       
	 zebra                            Running                     Process       
	 fpmsyncd                         Running                     Process       
	 bgpd                             Running                     Process       
	 staticd                          Running                     Process       
	 bgpcfgd                          Running                     Process       
	 root-overlay                     Accessible                  Filesystem    
	 var-log                          Accessible                  Filesystem 


Any above services or file systems is not in good status will be considered as fault status.

### 1.2 Peripheral devices status which could impact the system health status

- 1. Some fan is missing/broken 
- 2. not sufficient number or working fans in the system; 
- 3. Fan speed is below minimal range
- 4. PSU power voltage is our of range
- 5. PSU temperature is too hot
- 6. PSU is in bad status
- 7. ASIC temperature is too hot 

### 1.3 system status LED color definition

 | Color            |     Status    |       Description       |
 |:----------------:|:-------------:|:-----------------------:|
 | Off              |  off          |   no power              |
 | Blinking amber   |  off          |   in fault status       |
 | Solid green      |  Normal       |   in normal status      |


## 2. System health monitor service business logic

System health monitor daemon will running on the host, periodically check the monit summary and PSU, fan, thermal status which stored in the state DB, if anything wrong with the services monitored by monit or peripheral devices, system status LED will be set to fault status. When fault condition relieved, system status will be set to normal status.

this service will be started after....?

## 3. Platform API and PMON related change to support this new service

To have system LED can be set by this new service, system LED object need to be added to Chassis class. 
psud need to collect more PSU data to the DB to satisfy the requirement of this new service. more specifically, psud need to collect psu output voltage and it's threshold; psu temperature and it's threshold. 

	; Defines information for a psu
	key                     = PSU_INFO|psu_name              ; information for the psu
	; field                 = value
	presence                = BOOLEAN                        ; presence of the psu
	model                   = STRING                         ; model name of the psu
	serial                  = STRING                         ; serial number of the psu
	status                  = BOOLEAN                        ; status of the psu
	change_event            = STRING                         ; change event of the psu
	fan                     = STRING                         ; fan_name of the psu
	led_status              = STRING                         ; led status of the psu
    temp                    = INT                            ; tempeture of the PSU
    temp_th                 = INT                            ; tempeture threshold
    voltage                 = INT                            ; out voltage of the PSU
    voltage_th              = INT                            ; threshold of the output voltage

## 5. System health monitor CLI

Add a new "show system-health" command line to the system

	admin@sonic# show ?
	Usage: show [OPTIONS] COMMAND [ARGS]...
	
	  SONiC command line - 'show' command
	
	Options:
	  -?, -h, --help  Show this message and exit.
	
	Commands:
	  ...
	  startupconfiguration  Show startup configuration information
	  subinterfaces         Show details of the sub port interfaces
	  system-memory         Show memory information
      system-health         Show system health status
      ...

output is like below:

when everything is OK

	admin@sonic# show system-health
	Services OK   
	Hardware OK 

When something is wrong

	admin@sonic# show system-health
	Services Fault
        orchagent is not running   
	Hardware Fault
        PSU 1 temp 85C and threshold is 70C
        FAN 2 is broken
