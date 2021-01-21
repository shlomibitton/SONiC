# Query statistics capability Design

# Motivation

- Currently flex counters are initializing the supported counters on SYNCD docker by accessing the HW for the first port and keep this data to use for all other ports.
- The HW access consume a lot of time during boot of the system and add ~0.6 seconds to the process.
- The goal of this design is to find a better way of initializing the counters without consume time on startup, a main feature enhancement is for fast-reboot.


# Enhancement overview

- This enhancement will have two implementations:
    - Adding new SAI API which is not accessing the HW and update the supported counters in ~0 seconds.
    - Delay the update supported counters flow of flex counters after boot has finished.
- New SAI API will be used once it is implemented.
- Delay the flow with 2 suggested approaches introduced in this design document.

## Option 1

- Keep all data related to flex counters on ‘initPort’ function from the ‘orchagent’ process.
- Start a timer to delay the counters flow, sample if the boot process finished and when it does, populate the DB with the flex counters information.
- Once the data is available in the DB, SYNCD will consume this information and update the supported counters.
- Following function need to be added:
void PortsOrch::doTask(SelectableTimer &timer)
- Following variable need to be added:
vector m_port_stat_list

![Option 1](/doc/query_statistics_capability/query.svg)

## Option 2

- Keep all data related to flex counters on ‘initPort’ function from the ‘orchagent’ process.
- Listen to events from ‘Config DB’ related to enabling the counters.
There is a script running on SWSS docker (enable_counters.py) which delay enabling the counters on startup.
- Once ‘PORT’ counters are enabled, populate the DB with the flex counters information.
SYNCD will consume this information and update the supported counters.
- Following function need to be added:
void PortsOrch::doTask(Consumer &consumer)
- Following variable need to be added:
vector m_port_stat_list

![Option 2](/doc/query_statistics_capability/query2.svg)
