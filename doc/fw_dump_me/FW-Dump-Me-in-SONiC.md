# Dump Me Now in SONiC HLD

# High Level Design Document

#### Rev 0.1

# Table of Contents
- [Dump Me Now in SONiC HLD](#dump-me-now-in-sonic-hld)
- [High Level Design Document](#high-level-design-document)
      - [Rev 0.1](#rev-01)
- [Table of Contents](#table-of-contents)
- [List of Tables](#list-of-tables)
- [List of Figures](#list-of-figures)
- [Revision](#revision)
- [About this Manual](#about-this-manual)
- [Scope](#scope)
- [Definitions/Abbreviation](#definitionsabbreviation)
- [1. Overview](#1-overview)
- [2 Requirements Overview](#2-requirements-overview)
  - [2.1 Functional requirements](#21-functional-requirements)
- [3 Modules design](#3-modules-design)
  - [3.1 Dump Me Now build and runtime dependencies](#31-dump-me-now-build-and-runtime-dependencies)
  - [3.2 Dump Me Now daemon in SONiC](#32-dump-me-now-daemon-in-sonic)
  - [3.3 Dump Me Now in SONiC overview](#33-dump-me-now-in-sonic-overview)
  - [3.4 Dump Me Now provided data](#34-dump-me-now-provided-data)
  - [3.5 Dump Me Now output files](#35-dump-me-now-output-file)
  - [3.6 CLI](#36-cli)
  - [3.7 Dump Me Now daemon](#37-dump-me-now-daemon)
    - [3.7.1 Main thread](#371-main-thread)
    - [3.7.2 Use-Cases](#372-use-cases)
  - [3.8 Events types](#38-events-types)
  - [3.9 Integration to "show techsupport" command](#39-integration-to-show-techsupport-command)
- [4 Flows](#4-flows)
  - [4.1 Dump taking logic "trap handling"](#41-dump-taking-logic-trap-handling)
  - [4.2 FW trap generation](#42-fw-trap-generation)
  - [4.3 SDK trap generation](#43-sdk-trap-generation)
  - [4.4 Error flow - Dump taking failed (or timed out)](#44-error-flow---dump-taking-failed-or-timed-out)
  - [4.5 Dump Me Now init flow](#45-dump-me-now-init-flow)
- [5 Manual testing plan](#5-manual-testing-plan)

# List of Tables
* [Table 1: Abbreviations](#definitionsabbreviation)

# List of Figures
* [Dump Me Now in SONiC overview](#33-fw-dump-me-in-sonic-overview)
* [Dump Me Now general flow](#4-flows)
* [FW Dump Me init flow](#45-fw-dump-me-init-flow)

# Revision
| Rev | Date     | Author          | Change Description                 |
|:---:|:--------:|:---------------:|------------------------------------|
| 0.1 | 08/11    | Shlomi Bitton   | Initial version                    |

# About this Manual
This document provides an overview of the implementation and integration of Dump Me Now feature in SONiC.

# Scope
This document describes the high level design of the Dump Me Now feature in SONiC.

# Definitions/Abbreviation
| Abbreviation  | Description                               |
|---------------|-------------------------------------------|
| SONiC         | Software for open networking in the cloud |
| API           | Application programming interface         |
| SAI           | Switch Abstraction Interface              |
| SDK           | Software Developement Kit                 |
| FW            | Firmware                                  |

# 1. Overview

Dump Me Now feature gives the user a way to observe interrupts asserted from lower layers (SDK/FW).   
When such interrupt occur, all ASIC component dumps will be created even if the FW is stuck, as such we would like to "communicate" directly with the HW and not use the FW to get the info.   
The first release will include SDK and CR space dumps, in the future GDB core files and FW trace will be provided as well.

# 2 Requirements Overview

## 2.1 Functional requirements

Dump Me Now feature in SONiC should meet the following high-level functional requirements:

- Dump Me Now feature provides functionality as an additional daemon running within mellanox syncd container.
- Dump Me Now feature is a dubugging tool in SONiC for Mellanox customers and it is enabled by default.
- Dump Me Now will call SAI API to generate ASIC components dumps by three possible cases:
  - FW originated trap.
  - SDK originated trap.
  - User specifically requested for a dump.

# 3 Modules design

## 3.1 Dump Me Now build and runtime dependencies

* SDK libs
* SWSS common libs
* SAI libs

## 3.2 Dump Me Now daemon in SONiC

A new daemon will be added to mellanox 'syncd' container and will be included in Mellanox SONiC build by default.<p>
Build rules for Dump Me Now debian package will reside under *platform/mellanox/mlnx-fw-dump-me.mk*.

* */var/log/mellanox/dump_me_now* mounted inside the container (used for writing dump files)

## 3.3 Dump Me Now in SONiC overview 

![Dump Me Now in SONiC overview](/doc/dump_me_now/overview_syncd.svg)

## 3.4 Dump Me Now provided data

- The generating cause, severity & time (the cause of the “dump me now trap” or the reason the high level decided to take the dump).
- As much of the CR space according to priority defined in the ADB file.
- SDK dump.   
**In the future:**
- FW trace, I.E the FW configuration done to the HW from the beginning of time (boot). 
- GDB core files of all irisc’s.


## 3.5 Dump Me Now output files

CR/FW/GDB/SDK output files will be generated in "*/var/log/mellanox/dump_me_now*" if triggered by FW or SDK cause.
If the user will trigger the dump (show techsupport), it will be generated in "*/var/dump*" directory under sai_sdk directory
The new dump files name will be: <module_name>_<time_stamp>

## 3.6 CLI

The CLI provided will show all dumps currently on the disk.

Command:
```
admin@sonic:~$ show platform mlnx dumps
```
CLI output example:
```
admin@sonic:~$ show platform mlnx dumps
Dump Me Now
Dumps path = /var/log/mellanox/dump_me_now

-rw-r--r-- 1 root root 7057409 Oct 11 10:47 sdkdump_11_10_2020-10_47_20
-rw-r--r-- 1 root root 4883478 Oct 11 10:47 sdkdump_ext_cr_001-11_10_2020-10_47_19.udmp
-rw-r--r-- 1 root root     288 Oct 11 10:47 sdkdump_ext_meta_001-11_10_2020-10_47_19

It is recommended to delete old dumps after running "show techsupport"
admin@sonic:~$
```

## 3.7 Dump Me Now daemon

### 3.7.1 Main thread

The main thread will register to the relevant trap group and listen to events generated by FW/SDK/User.
Upon event arrival, a proper message will be logged in the system log containing all event information.
The dump will be automaticaly generated on event.

Log messages example as they appear on 'systemlog':

```
Sep 30 13:15:42.525034 arc-switch1038 NOTICE dumpd: :- handle_sdk_health_event: Health event captured, Severity: 'Notice' Cause: 'FW health issue'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE dumpd: :- handle_sdk_health_event: Dump taking started, Severity: 'Notice' Cause: 'FW health issue'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE dumpd: :- handle_sdk_health_event: Dump taking started, Severity: 'Notice' Cause: 'gobit not cleared'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE dumpd: :- handle_sdk_health_event: Dump taking finished
Sep 30 13:15:42.525034 arc-switch1038 NOTICE dumpd: :- handle_sdk_health_event: Dump taking failed, Timeout reached
```

### 3.7.2 Use-Cases

1. During dump taking SDK is **blocked**, thus no events can arrive during this operation.

2. If the user request for a dump and at the same time a dump is being taken, a proper message will display and the dump taking by the user will result as failure.

3. If a FW event occur and a dump was taken, the daemon will log all events from this point, if there are any, and **skip** the dump generation.

4. The number of dumps will be limited to X dumps to avoid any scenario of dump request flood.

5. The number of log entries will be limited to X to avoid log flooding.

## 3.8 Events types

There are 4 severity levels a "health" event can get:
* Critical
* Error
* Warning
* Notice

There are 4 causes a "health" event can get:
* FW health issue
* SDK go bit not cleared
* command interface completion timeout
* timeout in FW response

## 3.9 Integration to "show techsupport" command

Running the "show techsupport" command will trigger the SAI API to generate, in addition to the SDK dump, additional ASIC component dumps.
In addition, it will include all the dumps created (if there are any) as it could be dumps created as a result of a real issue.
For example, if a FW event occur some other events might happen in the SDK as a result of this FW issue.
In this case the first dump is the most important one and it should be included, regardless of other dumps taken.
The amount of dumps on the disk will be limited to X too prevent large disk usage. 

```
drwxr-xr-x  2 root root  4096 Sep 17 08:41 core
drwxr-xr-x  2 root root 12288 Sep 17 08:41 dump
drwxr-xr-x 77 root root  4096 Sep 17 04:47 etc
drwxr-xr-x  2 root root  4096 Sep 17 08:42 dump_me_now   <-----
-rwxr-xr-x  1 root root 17946 Sep 16 00:13 generate_dump
drwxr-xr-x  2 root root  4096 Sep 17 08:41 hw-mgmt
drwxr-xr-x  2 root root  4096 Sep 17 08:41 log
drwxr-xr-x  2 root root  4096 Sep 17 08:41 mstdump
drwxr-xr-x  4 root root  4096 Sep 17 04:55 proc
drwxr-xr-x  2 root root  4096 Sep 17 08:41 sai_sdk_dump
```

# 4 Flows

![Dump Me Now general flow](/doc/dump_me_now/flow.svg)

## 4.1 Dump taking logic "trap handling"

SONiC will handle the "dump me trap" as well as the action from the CLI by activating the "sx_fw_dump" flow.
1.	call: "sx_fw_dump"
2.	move the "dump files" to a designated (new) location: /var/log/mellanox/dump_me_now/<file> and add the info from the trap (cause + buffer) into the dump file
3.    generate SDK dump.

SDK will handle the call for "sx_fw_dump" generated by OS and will implement the "mstdump"+"gdb_dump"+"mlxtrace" logic to the "dump" logic

## 4.2 FW trap generation

The FW will generate the MFDE trap for internal FW causes (see FW ARCH)

## 4.3 SDK trap generation

The SDK will generate the MFDE trap for internal SDK causes when it encounters:
"go bit not cleared", FW timeout

## 4.4 Error flow - Dump taking failed (or timed out)

The "sx_fw_dump" will be executed with timeout that will close the application if it took too long.
in such a case a log error will be added.

## 4.5 Dump Me Now init flow

![Dump Me Now init flow](/doc/dump_me_now/init_syncd.svg)

# 5 Manual testing plan

All test cases will run on all asics SPC1-3.

| Test name                | Description                                                                                                                              |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| test_cli_request         | This test will check if a user request(techsupport) for a dump functionality is working.                                                 |
| test_fw_event            | This test will check if FW event triggering a dump and avoiding further dump taking requests.                                            |
| test_sdk_event           | This test will check if SDK health event is triggering a dump taking functionality.                                                      |
| test_config_reload       | This test will check if reloading configuration not breaking the feature functionality.                                                  |
| test_reboot              | This test will check if rebooting the switch is not breaking the feature functionality (will include fast/warm reboot).                  |
| test_cpu_mem_performance | This test will check CPU and Memory consumption.                                                                                         |
| test_cli_time            | This test will check how long a CLI execution and dump taking of FW and SDK cost (only dumps and not the whole "show techsupport" time)  |
