# How to configure Table size in SONiC

The customer may want to configure the table size of the FDB table, IPv4/IPv6 neighbor table from the SONiC level.

So on Mellanox platform we provided a way to change these tables through add configuration in the SAI profile.

## How to configure
The configuration are applied by changing the device SAI profile manually.

the path of the SAI profile on the SONiC DUT is  /usr/share/sonic/device/{platform name}/{sku name}/sai.profile,

for example, on ACS-MSN2700 the profile path is  /usr/share/sonic/device/x86_64-mlnx_msn2700-r0/ACS-MSN2700/sai.profile , 

in this file you need to add or edit(if it already exists) below keys, e.g.: 

    SAI_FDB_TABLE_SIZE=32768
    SAI_IPV4_ROUTE_TABLE_SIZE=102400
    SAI_IPV6_ROUTE_TABLE_SIZE=16384
    SAI_IPV4_NEIGHBOR_TABLE_SIZE=16384
    SAI_IPV6_NEIGHBOR_TABLE_SIZE=8192

above configuration intend to limit the max size of FDB to 32K, IPv4 route table to 100K, IPv6 route to 16K, IPv4 neighbor to 16K and IPv6 neighbor to 8K.

To make the configuration take effect need to cold reboot the switch.

## Limitation

### SPC-1 KVD single hash size issue
the original assumption is that the KVD single hash table size will be  `hash_single_size = fdb_num + ipv4_routes_num + ipv4_neighbors_num`,

but in current SDK implementation, for SPC-1,  `hash_single_size = fdb_num + ipv4_routes_num + ipv4_neighbors_num + ipv6_routes_num + VID_num`,

so currently on SPC-1, the real reserved FDB size is not as configured value.  


