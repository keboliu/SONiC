# New SFP Platform API Design

## 1. APIs for reading SFP info and DOM info

Considering to migrate from reading eeprom directly via access sysfs to using ethtool, cause hw-mgmt provided sysfs symbols for SFP eeprom and not standard Linux practice, maybe obsoleted in the future.

ethtool can provide a full parsed human readable eeprom content, also can return the raw hex dump starting from a certain offset.   

### a. Functions to get transceiver info (type, serial num, verdor name,...)

All the needed fields of this function can be found from the output of "ethtool -m", and already been well interpreted.
We can  make use of ethtool to get the needed items from it, to fill the return value for this function. 

for example:

       
        root@arc-mtbc-1001:/home/admin# time ethtool -m sfp56
        Identifier                                : 0x11 (QSFP28)
        Extended identifier                       : 0x8c
        Extended identifier description           : 2.5W max. Power consumption
        Extended identifier description           : CDR present in TX, CDR present in RX
        Extended identifier description           : High Power Class (> 3.5 W) not enabled
        Connector                                 : 0x23 (No separable connector)
        Transceiver codes                         : 0x81 0x00 0x00 0x00 0x00 0x00 0x00 0x00
        Transceiver type                          : 40G Ethernet: 40G Active Cable (XLPPI)
        Transceiver type                          : 100G Ethernet: 100G AOC or 25GAUI C2M AOC with worst BER of 5x10^(-5)
        Encoding                                  : 0x05 (64B/66B)
        BR, Nominal                               : 25500Mbps
        Rate identifier                           : 0x00
        Length (SMF,km)                           : 0km
        Length (OM3 50um)                         : 0m
        Length (OM2 50um)                         : 0m
        Length (OM1 62.5um)                       : 0m
        Length (Copper or Active cable)           : 3m
        Transmitter technology                    : 0x00 (850 nm VCSEL)
        Laser wavelength                          : 850.000nm
        Laser wavelength tolerance                : 15.000nm
        Vendor name                               : Mellanox
        Vendor OUI                                : 00:02:c9
        Vendor PN                                 : MFA1A00-C003
        Vendor rev                                : AC
        Vendor SN                                 : MT1706FT02064
        Revision Compliance                       : SFF-8636 Rev 2.0
        Module temperature                        : 32.67 degrees C / 90.81 degrees F
        Module voltage                            : 3.2803 V
        Alarm/warning flags implemented           : Yes
        Laser tx bias current (Channel 1)         : 6.750 mA
        Laser tx bias current (Channel 2)         : 6.750 mA
        Laser tx bias current (Channel 3)         : 6.750 mA
        Laser tx bias current (Channel 4)         : 6.750 mA
        Transmit avg optical power (Channel 1)    : 0.0000 mW / -inf dBm
        Transmit avg optical power (Channel 2)    : 0.0000 mW / -inf dBm
        Transmit avg optical power (Channel 3)    : 0.0000 mW / -inf dBm
        Transmit avg optical power (Channel 4)    : 0.0000 mW / -inf dBm
        Receiver signal OMA(Channel 1)            : 0.7231 mW / -1.41 dBm
        Receiver signal OMA(Channel 2)            : 0.7231 mW / -1.41 dBm
        Receiver signal OMA(Channel 3)            : 0.8890 mW / -0.51 dBm
        Receiver signal OMA(Channel 4)            : 1.0711 mW / 0.30 dBm
        Laser bias current high alarm   (Chan 1)  : Off
        Laser bias current low alarm    (Chan 1)  : Off
        Laser bias current high warning (Chan 1)  : Off
        Laser bias current low warning  (Chan 1)  : Off
        Laser bias current high alarm   (Chan 2)  : Off
        Laser bias current low alarm    (Chan 2)  : Off
        Laser bias current high warning (Chan 2)  : Off
        Laser bias current low warning  (Chan 2)  : Off
        Laser bias current high alarm   (Chan 3)  : Off
        Laser bias current low alarm    (Chan 3)  : Off
        Laser bias current high warning (Chan 3)  : Off
        Laser bias current low warning  (Chan 3)  : Off
        Laser bias current high alarm   (Chan 4)  : Off
        Laser bias current low alarm    (Chan 4)  : Off
        Laser bias current high warning (Chan 4)  : Off
        Laser bias current low warning  (Chan 4)  : Off
        Module temperature high alarm             : Off
        Module temperature low alarm              : Off
        Module temperature high warning           : Off
        Module temperature low warning            : Off
        Module voltage high alarm                 : Off
        Module voltage low alarm                  : Off
        Module voltage high warning               : Off
        Module voltage low warning                : Off
        Laser tx power high alarm   (Channel 1)   : Off
        Laser tx power low alarm    (Channel 1)   : Off
        Laser tx power high warning (Channel 1)   : Off
        Laser tx power low warning  (Channel 1)   : Off
        Laser tx power high alarm   (Channel 2)   : Off
        Laser tx power low alarm    (Channel 2)   : Off
        Laser tx power high warning (Channel 2)   : Off
        Laser tx power low warning  (Channel 2)   : Off
        Laser tx power high alarm   (Channel 3)   : Off
        Laser tx power low alarm    (Channel 3)   : Off
        Laser tx power high warning (Channel 3)   : Off
        Laser tx power low warning  (Channel 3)   : Off
        Laser tx power high alarm   (Channel 4)   : Off
        Laser tx power low alarm    (Channel 4)   : Off
        Laser tx power high warning (Channel 4)   : Off
        Laser tx power low warning  (Channel 4)   : Off
        Laser rx power high alarm   (Channel 1)   : Off
        Laser rx power low alarm    (Channel 1)   : Off
        Laser rx power high warning (Channel 1)   : Off
        Laser rx power low warning  (Channel 1)   : Off
        Laser rx power high alarm   (Channel 2)   : Off
        Laser rx power low alarm    (Channel 2)   : Off
        Laser rx power high warning (Channel 2)   : Off
        Laser rx power low warning  (Channel 2)   : Off
        Laser rx power high alarm   (Channel 3)   : Off
        Laser rx power low alarm    (Channel 3)   : Off
        Laser rx power high warning (Channel 3)   : Off
        Laser rx power low warning  (Channel 3)   : Off
        Laser rx power high alarm   (Channel 4)   : Off
        Laser rx power low alarm    (Channel 4)   : Off
        Laser rx power high warning (Channel 4)   : Off
        Laser rx power low warning  (Channel 4)   : Off
        Laser bias current high alarm threshold   : 0.000 mA
        Laser bias current low alarm threshold    : 0.000 mA
        Laser bias current high warning threshold : 0.000 mA
        Laser bias current low warning threshold  : 0.000 mA
        Laser output power high alarm threshold   : 0.0000 mW / -inf dBm
        Laser output power low alarm threshold    : 0.0000 mW / -inf dBm
        Laser output power high warning threshold : 0.0000 mW / -inf dBm
        Laser output power low warning threshold  : 0.0000 mW / -inf dBm
        Module temperature high alarm threshold   : 0.00 degrees C / 32.00 degrees F
        Module temperature low alarm threshold    : 0.00 degrees C / 32.00 degrees F
        Module temperature high warning threshold : 0.00 degrees C / 32.00 degrees F
        Module temperature low warning threshold  : 0.00 degrees C / 32.00 degrees F
        Module voltage high alarm threshold       : 0.0000 V
        Module voltage low alarm threshold        : 0.0000 V
        Module voltage high warning threshold     : 0.0000 V
        Module voltage low warning threshold      : 0.0000 V
        Laser rx power high alarm threshold       : 0.0000 mW / -inf dBm
        Laser rx power low alarm threshold        : 0.0000 mW / -inf dBm
        Laser rx power high warning threshold     : 0.0000 mW / -inf dBm
        Laser rx power low warning threshold      : 0.0000 mW / -inf dBm

We can extract the needed items from the result, no need to read the eeprom and parse it any more.

### b. Functions for get monitor values (power, voltage, temperature, bias...)
since this function will be called periodically, use ethtool to read out the raw data of the monitored items, then parse it with original sfputilbase.

for example:

    root@arc-mtbc-1001:/home/admin# time ethtool -m sfp37 offset 0 length 10
    Offset          Values
    ------          ------
    0x0000:         03 04 0b 00 00 00 00 40 08 00 


## 2. Set/Get lpmode, TX enable/disable functions

### Solution 1.
This is done by calling SX_APIs inside syncd, plan to extend the current mlnx_sfpd daemon inside syncd.
which is currently listening to the SDK for sfp change event and send the event to pmon daemon xcvrd via DB.
We can add reverse message channel from pmon to syncd, and mlnx_sfpd will listen to the message. 
When pmon platform APIs are called to set/get lpmode, or enable/disable TX, message will be sent from pmon to syncd.
mlnx-sfpd will listen to these message and do the action accordingly.

#### Open Question: 
how to convey the return value from syncd SX_API to pmon? 

### Solution 2.
start a sx_sdk task inside pmon, so can call SX_API directly inside pmon.

#### Open Questions
a. what kind if parameters needed for the init of sx_sdk? 

b. any conflict with current sx_sdk inside syncd?

c. non-tech issue: persuade MSFT/community to accept installing Mellanox specific SDK to common pmon docker.
