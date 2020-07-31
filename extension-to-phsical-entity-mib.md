# SONiC Entity MIB Extension #

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Kebo Liu      | Initial version                   |



## 1. Overview 

The Entity MIB contains several groups of MIB objects, currently SONiC only implemented part of the entityPhysical group following RFC2737. The definition of "entityPhsical" group defined below.

	EntPhysicalEntry ::= SEQUENCE {
		entPhysicalIndex          PhysicalIndex,
		entPhysicalDescr          SnmpAdminString,
		entPhysicalVendorType     AutonomousType,
		entPhysicalContainedIn    INTEGER,
		entPhysicalClass          PhysicalClass,
		entPhysicalParentRelPos   INTEGER,
		entPhysicalName           SnmpAdminString,
		entPhysicalHardwareRev    SnmpAdminString,
		entPhysicalFirmwareRev    SnmpAdminString,
		entPhysicalSoftwareRev    SnmpAdminString,
		entPhysicalSerialNum      SnmpAdminString,
		entPhysicalMfgName        SnmpAdminString,
		entPhysicalModelName      SnmpAdminString,
		entPhysicalAlias          SnmpAdminString,
		entPhysicalAssetID        SnmpAdminString,
		entPhysicalIsFRU          TruthValue
	}
## 2. Current Entity MIB implementation in SONiC
Now in SONiC implemented part of the MIB objects listed as below:

	entPhysicalDescr          SnmpAdminString,
	entPhysicalClass          PhysicalClass, 
	entPhysicalName           SnmpAdminString,
	entPhysicalHardwareRev    SnmpAdminString,
	entPhysicalFirmwareRev    SnmpAdminString,
	entPhysicalSoftwareRev    SnmpAdminString,
	entPhysicalSerialNum      SnmpAdminString,
	entPhysicalMfgName        SnmpAdminString,
	entPhysicalModelName      SnmpAdminString,

Now only transceivers and it's DOM sensors(Temp, voltage, rx power, tx power and tx bias) are added to the MIB table.

## 3. New extension to Entity MIB implementation
In this extension aim to implement all the objects in entityPhysical group.

Also plan to add more physical entities to such as thermal sensors, fan, and it's tachometers, PSU, PSU fan, and some sensors contained in PSU.

Another thing need to highlight is that in the current implementation,  currently "entPhysicalContainedIn" object is not implemented, so there is no way to reflect the physical location of the components, this time it will be amended, by this all the IMB instances can be organized in a hierarchy manner, as described in below chart.

	Chassis -
	         |--MGMT (Chassis)
	         |              |--CPU package Sensor/T(x) (Temperature sensor)
	         |              |--CPU Core Sensor/T(x) (Temperature sensor)
	         |              |--Board AMB temp/T(x) (Temperature sensor)
	         |              |--Ports AMB temp/T(x) (Temperature sensor)
	         |              |--SPC (Switch device)
	         |                  |--SPC/T(x) (Temperature sensor)
	         |--FAN(x) (Fan)
	         |              |-- FAN/F(y) (Fan sensor)
	         |--PS(x) (Power supply)
	         |              |-- FAN/F(y) (Fan sensor)
	         |              |-- power-mon/T(y) (Temperature sensor)
	         |              |-- power-mon/ VOLTAGE  (Voltage sensor)
	         |--Ethernet x/y … cable (Port module)
                            |--DOM Temperature Sensor for Ethernet(x)  (Temperature sensor)
                            |--DOM Voltage Sensor for Ethernet(x)  (Voltage sensor)
                            |--DOM RX Power Sensor for Ethernet(x)/(y) (Power sensor)
                            |--DOM TX Bias Sensor for Ethernet(x)/(y) (Bias sensor)


## 4. The data source of the MIB entries

Thermalctl daemon, Xcvrd, psud, are collecting physical device info to state DB, now we have PSU_INFO tale, FAN_INFO table, and TEMPERATURE_INFO table which can provide information for MIB entries. 

Thermal sensors MIB info will come from TEMPERATURE_INFO, FAN_INFO will feed to FAN MIB instance and PSU_INFO will be the source of the PSU related instances.

The current already implemented cable and cable DOM sensors getting data from tables(TRANSCEIVER_INFO and TRANSCEIVER_DOM_SENSOR) which maintained by xcvrd.

### 4.1 Adding more data to state DB

As mentioned in Section 3, currently lack implementation of "entPhysicalContainedIn", and also there is no such kind of info stored in the related tables of state DB. 

To have this kind info available we need to extend the current platform API, can add a new function get_contained_in_device_name to DeviceBase class, this function will return the name of the device which it is contained in. This new field will be populated to state DB by PMON daemons.

## 5. Entity MIB extension test

1. Get temp sensor MIB info and cross-check with the TEMPERATURE_INFO table.
2. Get fan MIB info and cross-check with the FAN_INFO table.
3. Get PSU related MIB info and cross-check with PSU_INFO and related tables
3. Remove/Add DB entries from related tables to see whether MIB info can be correctly updated.

