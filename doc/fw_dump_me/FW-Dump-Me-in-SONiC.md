# FW Dump Me in SONiC HLD

# High Level Design Document

#### Rev 0.1

# Table of Contents
- [FW Dump Me in SONiC HLD](#fw-dump-me-in-sonic-hld)
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
  - [3.1 FW Dump Me build and runtime dependencies](#31-fw-dump-me-build-and-runtime-dependencies)
  - [3.2 FW Dump Me daemon in SONiC](#32-fw-dump-me-daemon-in-sonic)
  - [3.3 FW Dump Me in SONiC overview](#33-fw-dump-me-in-sonic-overview)
  - [3.4 FW Dump Me provided data](#34-fw-dump-me-provided-data)
  - [3.5 FW Dump Me output file](#35-fw-dump-me-output-file)
  - [3.6 CLI](#36-cli)
  - [3.7 FW Dump Me daemon](#37-fw-dump-me-daemon)
    - [3.7.1 Main thread](#371-main-thread)
    - [3.7.2 Use-Cases](#372-use-cases)
    - [3.7.3 FW Dump Me communication with CLI](#373-fw-dump-me-communication-with-cli)
  - [3.8 Integration to "show techsupport" command](#38-integration-to-show-techsupport-command)
- [4 Flows](#4-flows)
  - [4.1 FW dump taking logic "trap handling"](#41-fw-dump-taking-logic-trap-handling)
  - [4.2 FW trap generation](#42-fw-trap-generation)
  - [4.3 SDK trap generation](#43-sdk-trap-generation)
  - [4.4 Error flow - Dump taking failed (or timed out)](#44-error-flow---dump-taking-failed-or-timed-out)
  - [4.5 FW Dump Me init flow](#45-fw-dump-me-init-flow)
- [5 Manual testing plan](#5-manual-testing-plan)

# List of Tables
* [Table 1: Abbreviations](#definitionsabbreviation)

# List of Figures
* [FW Dump Me in SONiC overview](#33-fw-dump-me-in-sonic-overview)
* [FW Dump Me general flow](#4-flows)
* [FW Dump Me init flow](#45-fw-dump-me-init-flow)

# Revision
| Rev | Date     | Author          | Change Description                 |
|:---:|:--------:|:---------------:|------------------------------------|
| 0.1 | 08/11    | Shlomi Bitton   | Initial version                    |

# About this Manual
This document provides an overview of the implementation and integration of FW Dump Me feature in SONiC.

# Scope
This document describes the high level design of the FW Dump Me feature in SONiC.

# Definitions/Abbreviation
| Abbreviation  | Description                               |
|---------------|-------------------------------------------|
| SONiC         | Software for open networking in the cloud |
| API           | Application programming interface         |
| SAI           | Switch Abstraction Interface              |
| SDK           | Software Developement Kit                 |
| FW            | Firmware                                  |

# 1. Overview

FW Dump Me feature gives the user a way to generate a full FW dump, even if the FW is stuck, as such we would like to "communicate" directly with the HW and not use the FW to get the info

# 2 Requirements Overview

## 2.1 Functional requirements

FW Dump Me feature in SONiC should meet the following high-level functional requirements:

- FW Dump Me feature provides functionality as an additional daemon running within mellanox syncd container.
- FW Dump Me feature is a dubugging tool in SONiC for Mellanox customers and it is enabled by default.
- FW Dump Me will create the dump by three possible cases:
  - FW originated trap.
  - SDK originated trap.
  - User specifically requested for a dump.

# 3 Modules design

## 3.1 FW Dump Me build and runtime dependencies

####################################

## 3.2 FW Dump Me daemon in SONiC

A new daemon will be added to mellanox 'syncd' container and will be included in Mellanox SONiC build by default.<p>
Build rules for FW Dump Me docker will reside under *platform/mellanox/docker-fw-dump-me.mk*.

* SDK Unix socket needs to be mapped to container (for CLI support).
* */var/log/mellanox/* mounted inside container (used for writing dump files)
* */var/run/fw_dump_me/* mounted inside container (used for Unix domain socket)

## 3.3 FW Dump Me in SONiC overview 

![FW Dump Me in SONiC overview](/doc/fw_dump_me/overview_syncd.svg)

## 3.4 FW Dump Me provided data

- The generating cause, severity & time (the cause of the “FW dump me now trap” or the reason the high level decided to take the dump).
- FW trace, I.E the FW configuration done to the HW from the beginning of time (boot). 
- GDB core files of all irisc’s.
- As much of the CR space according to priority defined in the ADB file.
- SDK dump.

## 3.5 FW Dump Me output file

The FW output files will be generated in "/var/log/mellanox/fw_dump_me" if triggered by FW or SDK cause.
If the user will trigger the dump taking, it will be generated under the path provided by the user.
If no path provided, the default location will be used.
The new dump files name will be: <device_id>/<module_name>_<time_stamp>

The SDK dump output file will be generated in "/var/log/mellanox/fw_dump_me".

## 3.6 CLI

Since the FW Dump Me daemon is running on a seperate container (syncd), CLI will be provided by an additional process communicating with the daemon with a socket. The CLI will start by sending a command with the host CLI ('show' script), it will recieve and send the requests to the daemon to trigger dump generating.

Command to create a new dump:
```
admin@sonic:~$ show platform mlnx fw-dump-me <desired_path>
```
CLI output examples:
```
admin@sonic:~$ show platform mlnx fw-dump-me /var/log/mellanox/example
FW Dump Me
path  = "/var/log/mellanox/example"
Generating dump...
Finished successfully 
Output = /var/log/mellanox/example/fw_dump_r-tigon-04_20200813_013434
admin@sonic:~$
```
```
admin@sonic:~$ show platform mlnx fw-dump-me /var/log/mellanox/example
FW Dump Me
path  = "/var/log/mellanox/example"
Generating dump...
Failed, timeout reached
admin@sonic:~$
```
```
admin@sonic:~$ show platform mlnx fw-dump-me
FW Dump Me
path  = "/var/log/mellanox/fw_dump"
Generating dump...
Finished successfully 
Output = /var/log/mellanox/fw_dump/fw_dump_r-tigon-04_20200813_013434
admin@sonic:~$
```

## 3.7 FW Dump Me daemon

### 3.7.1 Main thread

The main thread will register to the relevant trap group and listen to events generated by FW/SDK/User.
Upon event arrival, a proper message will be logged in the system log containing all event information.
The dump will be automaticaly generated on event.

Log messages example as they appear on 'systemlog':

```
Sep 30 13:15:42.525034 arc-switch1038 NOTICE fwdumpd: :- handle_sdk_health_event: Health event captured, Severity: 'Notice' Cause: 'FW health issue'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE fwdumpd: :- handle_sdk_health_event: Dump taking started, Severity: 'Notice' Cause: 'FW health issue'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE fwdumpd: :- handle_sdk_health_event: Dump taking started, Severity: 'Notice' Cause: 'gobit not cleared'
Sep 30 13:15:42.525034 arc-switch1038 NOTICE fwdumpd: :- handle_sdk_health_event: Dump taking finished, File path: /var/log/mellanox/fw_dump_me_arc-switch1038_20200813_013434
Sep 30 13:15:42.525034 arc-switch1038 NOTICE fwdumpd: :- handle_sdk_health_event: Dump taking failed, Timeout reached
```

### 3.7.2 Use-Cases

There are 4 severity levels a "health" event can get:
* Critical
* Error
* Warning
* Notice

1. During dump taking SDK is **blocked**, thus no events can arrive during this operation.

2. If the user request for a dump and at the same time a FW dump is being taken, a proper message will display and the dump taking by the user will result as failure.
The same result will be for a user requesting for a dump after a FW fatal event occur.

3. After each event, when a dump taking is finished, a "re-arm" operation will enable recieving more "health" events and generate more dumps.
But, if the trigger was by the FW as a result of a fatal case the daemon will log the events, if there are any, and **skip** the dump generation.

4. The decision to take a dump will be by the 'severity' of the event. In this case the SDK will be blocked only if it is "severe enough" ('critical' and 'error' levels).
For 'warning' and 'notice' a log entry will be added but no dump.

5. If the event requires taking a dump, FW dump will be taken first as this is real time sensitive.
A delay of 1-2 seconds will start to allow the SDK continue working and avoid blocking for too long period.
After this delay a SDK dump will be triggered.

6. The number of dumps will be limited to X dumps to avoid any scenario of dump request flood.

7. The number of log entries will be limited to X to avoid log flooding.

### 3.7.3 FW Dump Me communication with CLI

In order to avoid producing additional load in Redis DB or bringing another Redis instance specifically for FW Dump Me another IPC mechanism will be used.
A suggested alternative is a Unix domain socket. It may be placed under */var/run/fw-dump-me/dump.sock* which will be mapped to syncd container.

On CLI request FW Dump Me daemon trigger dump generation from FW and save it on the requested path.

CLI/FW Dump Me daemon communication protocol will be text based in the following format:

```
path    = string ; location for saving output files
```

Since the design is focused on one CLI client, only one connection will be handled at the time.

A considerable timeout has to be set on socket so that send/recv will not block CLI or daemon if one side unexpectedly terminates.

## 3.8 Integration to "show techsupport" command

Running the "show techsupport" command will trigger a new dump taking and include it in the output file.
In addition, it will include the last dump created (if there is one) as it could be a dump created as a result of a real issue.

```
drwxr-xr-x  2 root root  4096 Sep 17 08:41 core
drwxr-xr-x  2 root root 12288 Sep 17 08:41 dump
drwxr-xr-x 77 root root  4096 Sep 17 04:47 etc
drwxr-xr-x  2 root root  4096 Sep 17 08:42 fw_dump   <-----
-rwxr-xr-x  1 root root 17946 Sep 16 00:13 generate_dump
drwxr-xr-x  2 root root  4096 Sep 17 08:41 hw-mgmt
drwxr-xr-x  2 root root  4096 Sep 17 08:41 log
drwxr-xr-x  2 root root  4096 Sep 17 08:41 mstdump
drwxr-xr-x  4 root root  4096 Sep 17 04:55 proc
drwxr-xr-x  2 root root  4096 Sep 17 08:41 sai_sdk_dump
```

# 4 Flows

![FW Dump Me general flow](/doc/fw_dump_me/flow.svg)

## 4.1 FW dump taking logic "trap handling"

SONiC will handle the "fw dump me trap" as well as the action from the CLI by activating the "sx_fw_dump" flow.
1.	set a timer
2.	call: "sx_fw_dump <device_id> <file_path> <extra_info>"
3.	move the "fw dump file" to a designated (new) location: /var/opt/tms/fwdumps/system/<file> and add the info from the trap (cause + buffer) into the dump file
4.    generate SDK dump.

SDK will handle the "sx_fw_dump <device_id> <file_path> <extra_info>" generated by OS and will implement the "mstdump"+"gdb_dump"+"mlxtrace" logic to the "dump" logic

## 4.2 FW trap generation

The FW will generate the MFDE trap for internal FW causes (see FW ARCH)

## 4.3 SDK trap generation

The SDK will generate the MFDE trap for internal SDK causes when it encounters:
"go bit not cleared", FW timeout

## 4.4 Error flow - Dump taking failed (or timed out)

The "sx_fw_dump" will be executed with timeout that will close the application if it took too long.
in such a case a log error will be added.

## 4.5 FW Dump Me init flow

![FW Dump Me init flow](/doc/fw_dump_me/init_syncd.svg)

# 5 Manual testing plan

All test cases will run on all asics SPC1-3.

| Test name                | Description                                                                                                                    |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| test_cli_request         | This test will check if user request(cli) for a dump functionality is working.                                                 |
| test_fw_fatal_event      | This test will check if FW fatal event triggering a dump and avoiding further dump taking requests.                            |
| test_sdk_sever_event     | This test will check if SDK health event is triggering a dump taking functionality.                                            |
| test_sdk_normal_event    | This test will check if a not sever SDK health event is logged and a system dump is skipped.                                   |
| test_config_reload       | This test will check if reloading configuration not breaking the feature functionality.                                        |
| test_reboot              | This test will check if rebooting the switch is not breaking the feature functionality (will include fast/warm reboot also).   |
| test_cpu_mem_performance | This test will check CPU and Memory consumption.                                                                               |
| test_cli_time            | This test will check how long a CLI execution and dump taking of FW and SDK cost.                                              |
