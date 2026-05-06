---
tags:
  - 吉利
---


QNX模拟车身信号，收( -r)发(-w)。carmode切到0，usgmode（ -p 557849093）切到driving(-d 13).
VehicleTool -r -s 1 -p 557849092 -a 0 -d 0 && VehicleTool -r -s 1 -p 557849093 -a 0 -d 13
# VehicleTool -h
[Debug] ch = 104

Usage: VehicleTool [-i num] [-c] [-v] [-h] [-b] [-w/r] [-s] [-t time|interval] [-p prop] [-a area] [-d val1|val2,val3...]
                     [-f key] [-P opt]
    ==== Options for vehicle service tool ====
    -i (info): show vehicle property statistics, 'num' represents the top 'num' properties, and slog details by batch.
    -c (clear): clear vehicle property statistics buffer
    -v (variant): read current car type(variant)
    -p (prop): set the property ID
    -a (area): set the area ID
    -d (data): set the value, format: 'value1|value2|..., value3|value4|value5|..., ...'
    -b (buffer): read property's current value, could support multiple properties separated by '|'
                 e.g.  show Usgmod status:  VehicleTool -b -p 557849093
    -t (time and interval): set the repeat times(default 1 time) and interval(default is 100ms, unit is ms) of sending value, format: VehicleTool -t 'repeat_times|interval'
    -w (write): write specific prop values to the GW, format: VehicleTool -w -p ID -a area -d 'value1|value2|...'
                 e.g. set CM=0 and UM=13
                      VehicleTool -r -s 1 -p 557849092 -a 0 -d 0 && VehicleTool -r -s 1 -p 557849093 -a 0 -d 13
    -r (read): write specific canoe values to the QNX and Android, format: VehicleTool -r -p ID -a area -d 'value1|value2|...'
    -s (style): set the simulating style, 0 - PDU (by default), 1 - Signal'
    -f (fold): fold slog by the 'key' word assigned
    -P (id): control the VehicleService process, e.g.
                         stop   :  VehicleTool -P0
                         suspend:  VehicleTool -P1
                         start  :  VehicleTool -P2
    -U e.g. VehicleTool -U
    -D Open the debug log in serial of vs. e.g.  open  : VehicleTool -D 1
                                                 close : VehicleTool -D 0
    -h (help): Print usage.