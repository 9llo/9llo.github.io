---
title: "VRMS Connection refused: tracking down an hms.service file descriptor leak"
date: 2026-04-20 16:00:00 -0300
categories: VMware Troubleshooting
tags: vsphere vsphere-replication vrms vmware vlsr troubleshooting # TAG names should always be lowercase
description: Intermittent VRMS API failures traced through systemd watchdog timeouts, "Too many open files" errors, and a single XML parameter — schedule-health-checks — that was slowly exhausting file descriptors until hms crashed.
---

If you manage vSphere Replication through the API and are hitting intermittent `Connection refused` errors on port 8043, this is likely what's happening. The error looks like this:

```
Unable to retrieve pairs from extension server at https://vrms-appliance.example.local:8043.
Unable to connect to vSphere Replication Management Server at https://vrms-appliance.example.local:8043.
Reason: https://vrms-appliance.example.local:8043 invocation failed with
"java.net.ConnectException: Connection refused", ErrorCode=2.99.2,
OpID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

`hms.service` (the vSphere Replication Management Server Java process) listens on port 8043. When it's not there, API calls fail. The process wasn't missing because of a network problem — it was restarting.

---

## Following the restart counter

The first useful number came from `systemctl show`:

```bash
root@vrms-appliance [ ~ ]# systemctl show hms.service --property=NRestarts
NRestarts=13
```

13 restarts since the last VRMS reboot 13 days prior — roughly one crash per day. `hbrsrv.service` (the replication engine) had been running continuously for those same 13 days, so the appliance itself was stable. `hms` was the one cycling.

The journal had the mechanism:

```
Apr 14 11:16:28 vrms-appliance systemd[1]: hms.service: Watchdog timeout (limit 5min)!
Apr 14 11:16:28 vrms-appliance systemd[1]: hms.service: Killing process 1725075 (java) with signal SIGABRT.
Apr 14 11:16:29 vrms-appliance systemd-coredump[2582430]: Process 1725075 (java) of user 980 dumped core.
Apr 14 11:16:29 vrms-appliance systemd[1]: hms.service: Main process exited, code=dumped, status=6/ABRT
Apr 14 11:16:29 vrms-appliance systemd[1]: hms.service: Failed with result 'watchdog'.
Apr 14 11:16:34 vrms-appliance systemd[1]: hms.service: Scheduled restart job, restart counter is at 13.
```

systemd's watchdog kills the process if it stops responding to health checks within the configured interval (5 minutes here) and restarts it. `hms` was consistently missing that window.

---

## Too many open files

The kernel audit log pointed at the java process:

```
Apr 14 11:16:28 vrms-appliance kernel: audit: type=1701 audit(1776176188.958:11942): auid=4294967295
uid=980 gid=980 ses=4294967295 subj=unconfined pid=1725075 comm="java"
exe="/usr/java/jre-vmware/bin/java" sig=6 res=1
```

And `hms-stderr.log` had the actual error, repeating on a roughly 2-second interval for several minutes before the crash:

```
Apr 14, 2026 11:16:08 AM org.apache.tomcat.util.net.Acceptor run
SEVERE: Socket accept failed
java.io.IOException: Too many open files
    at java.base/sun.nio.ch.Net.accept(Native Method)
    at java.base/sun.nio.ch.ServerSocketChannelImpl.implAccept(Unknown Source)
    at java.base/sun.nio.ch.ServerSocketChannelImpl.accept(Unknown Source)
    at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:449)
    at org.apache.tomcat.util.net.Acceptor.run(Acceptor.java:129)
```

The Tomcat acceptor thread was trying to accept new connections and hitting the file descriptor limit. Once you can't accept connections, the process can't respond to systemd's watchdog health check either — so systemd kills it.

About 2.5 hours after the last restart, the process was already at 14,472 open file descriptors against a limit of 20,000:

```bash
root@vrms-appliance [ ~ ]# cat /proc/2582480/limits | grep -i "open files"
Max open files            20000                20000                files

root@vrms-appliance [ ~ ]# ls /proc/2582480/fd | wc -l
14472
```

72% of the limit gone in under three hours. At that rate, the process hits the ceiling well before the end of the day.

---

## Tracing it to schedule-health-checks

A Broadcom engineer suggested looking at `schedule-health-checks` in `hms-configuration.xml`:

```bash
root@vrms-appliance [ ~ ]# cat /opt/vmware/hms/conf/hms-configuration.xml | grep "<schedule-health-checks>"
<schedule-health-checks>true</schedule-health-checks>
```

`schedule-health-checks` tells `hms` to periodically poll replication health on all protected VMs. Each cycle opens connections that are never properly closed — so file descriptors accumulate until the process runs out and crashes.

---

## How to check if you're affected

Run these commands on your VRMS appliance via SSH.

**1. Check if hms.service has been restarting:**

```bash
systemctl show hms.service --property=NRestarts
```

Any value above 0 means the service has crashed and been restarted since the last appliance reboot. Cross-reference with uptime:

```bash
uptime
```

If the appliance has been up for weeks but `NRestarts` is in the double digits, something is cycling `hms` regularly.

**2. Confirm the restarts are caused by a watchdog timeout:**

```bash
journalctl -u hms.service --no-pager | grep -i "watchdog\|Failed with result"
```

You're looking for lines like:

```
hms.service: Watchdog timeout (limit 5min)!
hms.service: Failed with result 'watchdog'.
```

If you see those, the process is hanging — not crashing for an unrelated reason.

**3. Check current file descriptor usage:**

Get the PID of the running hms process:

```bash
systemctl show hms.service --property=MainPID
```

The output will look like:

```
MainPID=2582480
```

Use that number in the next two commands — replacing `2582480` with whatever value you got:

```bash
cat /proc/2582480/limits | grep -i "open files"
ls /proc/2582480/fd | wc -l
```

The first command shows the configured limit (`Max open files`). The second shows how many file descriptors the process currently has open. If the count from the second command is already a large fraction of the limit — and the service restarted recently — the leak is active.

**4. Confirm the parameter is enabled:**

```bash
cat /opt/vmware/hms/conf/hms-configuration.xml | grep "<schedule-health-checks>"
```

If it returns `<schedule-health-checks>true</schedule-health-checks>`, you're hitting this issue.

---

## Fix

Steps:

1. **Take a snapshot of the appliance first.**

2. SSH into the appliance and stop `hms.service`:

    ```bash
    systemctl stop hms.service
    ```

3. Edit the configuration file:

    ```bash
    vi /opt/vmware/hms/conf/hms-configuration.xml
    ```

    Change:

    ```xml
    <schedule-health-checks>true</schedule-health-checks>
    ```

    To:

    ```xml
    <schedule-health-checks>false</schedule-health-checks>
    ```

4. Wait approximately two minutes for existing open sockets to time out and close.

5. Start the service:

    ```bash
    systemctl start hms.service
    ```

6. Confirm `NRestarts` is now 0:

    ```bash
    systemctl show hms.service --property=NRestarts
    ```

    You can also verify how long `hms` has been up without issue:

    ```bash
    systemctl status hms.service | grep Active
    ```

---

## After the fix

The following day, `NRestarts` was back to 0 and has stayed there.

```bash
root@vrms-appliance [ ~ ]# systemctl show hms.service --property=NRestarts
NRestarts=0
```

The Broadcom engineer on this case was genuinely helpful — they pointed us to the `schedule-health-checks` parameter and, after we confirmed the fix, took the time to document it officially. That kind of follow-through is worth calling out.

Broadcom published [KB 437468](https://knowledge.broadcom.com/external/article/437468) covering this issue. The KB confirms the root cause: health check connections for Enhanced Replication mappings are never properly closed, so file descriptors accumulate until the process crashes. Affected and fixed versions:

| | vSphere Replication | Live Site Recovery |
|---|---|---|
| **Affected** | 9.x with large-scale ESXi hosts using Enhanced Replication | 9.0.3, 9.0.4 |
| **Fixed** | 9.0.2.3 | 9.0.5 |

Upgrading to the fixed version removes the need for the `schedule-health-checks` workaround entirely.
