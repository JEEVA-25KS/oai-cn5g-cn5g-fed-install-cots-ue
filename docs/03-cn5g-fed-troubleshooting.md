# OAI 5G Testbed (with oai-cn5g-fed) — Troubleshooting & Resolution Report

## Executive Summary

This report documents the complete troubleshooting process encountered during the
deployment of the OpenAirInterface (OAI) 5G Standalone (SA) testbed. The deployment
involved setting up the OAI CN5G federated core network (`oai-cn5g-fed`) using Docker
Compose, building and running the OAI gNB (`nr-softmodem`) with a NI USRP-2901 (B210)
software-defined radio, and establishing a successful NG interface connection between
the gNB and the AMF.

A total of five distinct issues were encountered during the deployment. Each issue was
systematically diagnosed using log analysis, configuration inspection, and network
verification commands. All five issues were successfully resolved, culminating in a
fully operational gNB connected to the 5G core network with the NG Setup procedure
completing successfully.

| # | Issue | Component | Root Cause | Status |
|---|-------|-----------|------------|--------|
| 1 | Missing Config Files (`sip.conf`, `users.conf`) | Docker / oai-cn5g-fed | Files not present before container start | ✅ Resolved |
| 2 | PLMN Mismatch between gNB and AMF | Config / gNB | Mismatched MCC/MNC values | ✅ Resolved |
| 3 | TAC Mismatch between gNB and AMF | Config | Mismatched TAC values | ✅ Resolved |
| 4 | Dynamic Docker Container IPs | Docker Networking | IPs unset, reassigned on restart | ✅ Resolved |
| 5 | Wrong AMF IP in gNB Config | gNB Config | IP updated each restart due to Issue 4 | ✅ Resolved |

---

## Detailed Issue Analysis

### Issue 1: Missing Config Files (`sip.conf` & `users.conf`) — Critical

#### 1.1 Background & Context
During the initial deployment attempt of the `oai-cn5g-fed` stack, the
`docker-compose-basic-nrf.yaml` file was used to bring up the 5G core containers. The
deployment script `python3 core-network.py --type start-basic --scenario 1` was
executed to launch all containers.

#### 1.2 Symptom & Error
The following error was thrown by the Docker runtime during container initialization:
```
Error response from daemon: failed to create task for container:
failed to create shim task: OCI runtime create failed:
runc create failed: unable to start container process:
error during container init: error mounting
"/home/basestation/oai-cn5g-fed/docker-compose/conf/config.yaml"
to rootfs at "/openair-nrf/etc/config.yaml":
mount src=...config.yaml, dst=...config.yaml,
flags=MS_BIND|MS_REC: not a directory:
Are you trying to mount a directory onto a file (or vice-versa)?
Check if the specified host path exists and is the expected type
```
The same error then recurred for `users.conf`:
```
error mounting "/home/basestation/oai-cn5g-fed/docker-compose/conf/users.conf"
to rootfs at "/etc/asterisk/users.conf":
not a directory: Are you trying to mount a directory onto a file?
```

#### 1.3 Root Cause Analysis
When Docker performs a bind mount from a host path to a container path, it requires
the source (host) path to already exist with the correct type — either a file or a
directory. If the source file does not exist at the time Docker attempts the mount,
Docker automatically creates it as an **empty directory**.

In this case, the `conf/` directory was present but `config.yaml` and `users.conf`
did not exist as files. Docker created them as empty directories instead. When the
container then tried to mount these paths expecting files, it encountered a type
conflict — a directory cannot be mounted as a file.

Verification confirmed the problem:
```bash
$ ls -la ~/oai-cn5g-fed/docker-compose/conf/config.yaml
total 8
drwxr-xr-x 2 root root 4096 Apr 7 15:24 .
drwxrwxr-x 5 basestation basestation 4096 Apr 7 15:24 ..
# config.yaml is a DIRECTORY, not a file!
```

#### 1.4 Investigation Steps
- Inspected the `conf/` directory using `find` to distinguish files from directories
- Confirmed that `config.yaml` and `users.conf` were both created as directories by Docker
- Reviewed the `docker-compose-basic-nrf.yaml` to determine which mounts were needed
- Searched the repository for template config files using `find`
- Determined that the basic NRF compose file does not mount `sip.conf` or `users.conf`
  at all — those belonged to an IMS container that is not part of the basic scenario

#### 1.5 Resolution
The cleanest fix was a complete reinstallation of the `oai-cn5g-fed` repository. The
entire directory was deleted and re-cloned at a stable tagged version:
```bash
# Remove broken installation
cd ~
rm -rf ~/oai-cn5g-fed

# Fresh clone at stable version
git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git
cd ~/oai-cn5g-fed
git checkout v2.1.0
git submodule update --init --recursive
```
The new clone came with the correct `basic_nrf_config.yaml` already present in
`conf/`, and the basic NRF compose file only mounts that single file — no `sip.conf`
or `users.conf` required.

#### 1.6 Lesson Learned
Always ensure that all files referenced in Docker bind mounts exist on the host as
actual files before running `docker-compose up`. Docker will silently create missing
paths as directories, causing a type conflict at runtime. This is a well-known Docker
behavior that is easy to overlook.

---

### Issue 2: PLMN Mismatch Between gNB and AMF — Critical

#### 2.1 Background & Context
After the fresh installation and successful startup of the 5G core, the OAI gNB was
launched using the default configuration file
`gnb.sa.band78.fr1.106PRB.usrpb210.conf`. The gNB attempted to register with the AMF
through the NG Setup procedure over SCTP.

#### 2.2 Symptom & Error
The gNB log showed the NG Setup being explicitly rejected by the AMF:
```
[NGAP] Send NGSetupRequest to AMF
[NGAP] 3584 -> 0000e000
[NGAP] Received NG setup failure for AMF... please check your parameters
```
The AMF log provided the specific reason for the rejection:
```
[amf_n2] [error] [gNB ID 57344] No common PLMN between gNB and AMF,
encoding NG_SETUP_FAILURE with cause (Unknown PLMN)
value: 4 (unknown-PLMN-or-SNPN)
```

#### 2.3 Root Cause Analysis
The gNB default configuration file uses PLMN values of MCC=001, MNC=01, which are
standard test network identifiers. However, the AMF's `plmn_support_list` in
`basic_nrf_config.yaml` only contained the entry for MCC=208, MNC=95 (the French
Orange network identifiers used as defaults in the OAI configuration).

During NG Setup, the gNB sends its supported PLMN list to the AMF. The AMF checks
whether any of the gNB's PLMNs match its own `plmn_support_list`. Since 001/01 was
not in the AMF's list, the AMF rejected the request with cause **Unknown PLMN**.

| Parameter | Value |
|-----------|-------|
| gNB PLMN | MCC=001, MNC=01 (test network) |
| AMF `plmn_support_list` | MCC=208, MNC=95 (French Orange) |
| Result | No common PLMN — NG Setup **REJECTED** |

#### 2.4 Investigation Steps
- Compared PLMN values between gNB config and AMF config using `grep`
- Confirmed the AMF `plmn_support_list` only contained 208/95
- Noted that `served_guami_list` already had both 208/95 **and** 001/01, but
  `plmn_support_list` did not

#### 2.5 Resolution
Since the testbed is designed to use the standard test PLMN (001/01), the AMF
configuration was updated rather than the gNB. The `plmn_support_list` in
`basic_nrf_config.yaml` was changed from 208/95 to 001/01:
```yaml
# In ~/oai-cn5g-fed/docker-compose/conf/basic_nrf_config.yaml
# BEFORE:
plmn_support_list:
  - mcc: 208
    mnc: 95
    tac: 0xa000

# AFTER:
plmn_support_list:
  - mcc: 001
    mnc: 01
    tac: 0xa000   # TAC still to be fixed in Issue 3
```
The core was then restarted to apply the new configuration.

---

### Issue 3: TAC Mismatch Between gNB and AMF — High

#### 3.1 Background & Context
After fixing the PLMN mismatch in Issue 2, the NG Setup was still being rejected.
Even with matching PLMNs, the AMF performs additional validation of the Tracking Area
Code (TAC) during the NG Setup procedure.

#### 3.2 Symptom & Error
The same NG Setup failure persisted after the PLMN fix, with the AMF still rejecting
the gNB registration. Deeper log analysis revealed the TAC discrepancy.

#### 3.3 Root Cause Analysis
The gNB configuration file had TAC set to 1 (decimal), which is `0x0001` in
hexadecimal. However, the AMF's `plmn_support_list` still had the original TAC value
of `0xa000` (decimal 40960) from the default French Orange configuration. These
values did not match, causing continued rejection.

| Parameter | Value |
|-----------|-------|
| gNB TAC (from log) | TAC 1 (decimal) = `0x0001` |
| AMF `plmn_support_list` TAC | `0xa000` (decimal 40960) |
| Result | TAC mismatch — NG Setup **REJECTED** |

Evidence from gNB startup log:
```
[GNB_APP] F1AP: gNB idx 0 gNB_DU_id 3584, gNB_DU_name gNB-OAI,
TAC 1 MCC/MNC/length 1/1/2 cellID 12345678
# TAC=1 clearly shown here
```

#### 3.4 Investigation Steps
- Re-examined AMF logs after PLMN fix — still showing rejection
- Used `grep` to extract TAC values from both the gNB config and the AMF config
- Identified the `0xa000` vs `0x0001` mismatch

#### 3.5 Resolution
The TAC in the AMF's `plmn_support_list` was updated to match the gNB's TAC value of `0x0001`:
```yaml
# In ~/oai-cn5g-fed/docker-compose/conf/basic_nrf_config.yaml
# BEFORE:
plmn_support_list:
  - mcc: 001
    mnc: 01
    tac: 0xa000   # Wrong TAC

# AFTER:
plmn_support_list:
  - mcc: 001
    mnc: 01
    tac: 0x0001   # Matches gNB TAC=1
```
The core was restarted again to apply this fix.

---

### Issue 4: Dynamic Docker Container IPs — High

#### 4.1 Background & Context
With PLMN and TAC now matching, a new class of instability appeared: the IP
addresses of core network containers (particularly the AMF) were changing between
restarts of the `oai-cn5g-fed` stack.

#### 4.2 Symptom & Error
Repeated restarts of the core produced different container IPs each time, which in
turn broke any gNB configuration that hardcoded a specific AMF IP:
```
# After 1st start: oai-amf 192.168.70.138
# After 2nd start: oai-amf 192.168.70.135
# After 3rd start (static IPs enabled): oai-amf 192.168.70.132   <- now permanent
```

#### 4.3 Root Cause Analysis
The `docker-compose-basic-nrf.yaml` file had static IP assignments **commented out**
by default — intentionally, for CI (Continuous Integration) pipeline purposes, where
fixed IPs can cause conflicts in shared environments:
```yaml
# From docker-compose-basic-nrf.yaml:
networks:
  public_net:
    # For CI purposes, we are keeping the line commented
    # ipv4_address: 192.168.70.132
```
Without fixed IPs, Docker's internal IPAM (IP Address Management) assigns addresses
dynamically from the subnet pool, and the assignment order can change between
restarts depending on which containers start first.

#### 4.4 Investigation Steps
- Used `docker inspect` with a Go template to get IP addresses of all containers at once
- Confirmed IPs changed between restarts by comparing multiple `docker inspect` outputs
- Searched the compose file for IP configuration and found the commented-out static IP lines
- Verified the subnet was `192.168.70.0/26` with gateway at `192.168.70.129` (host)

#### 4.5 Resolution
The commented-out static IP assignments were uncommented using `sed`, making all
container IPs permanent:
```bash
# Uncomment all static IP assignments in one command
sed -i 's/# ipv4_address:/ ipv4_address:/g' \
  ~/oai-cn5g-fed/docker-compose/docker-compose-basic-nrf.yaml

# Verify all IPs are now active
grep 'ipv4_address' ~/oai-cn5g-fed/docker-compose/docker-compose-basic-nrf.yaml
```
The resulting permanent IP assignments are:

| Container | Static IP Address |
|-----------|-------------------|
| oai-nrf | 192.168.70.130 |
| mysql | 192.168.70.131 |
| oai-amf | 192.168.70.132 |
| oai-smf | 192.168.70.133 |
| oai-upf | 192.168.70.134 |
| oai-ext-dn | 192.168.70.135 |
| oai-udr | 192.168.70.136 |
| oai-udm | 192.168.70.137 |
| oai-ausf | 192.168.70.138 |

---

### Issue 5: Incorrect AMF IP Address in gNB Configuration — High

#### 5.1 Background & Context
This issue was a direct consequence of Issue 4. Because Docker was assigning
different IP addresses to the AMF container on each restart, the gNB configuration
file had to be manually updated multiple times throughout the debugging session.
Additionally, determining the correct AMF IP was complicated by conflicting outputs
from different `docker inspect` commands.

#### 5.2 Symptom & Error
The gNB would start but fail to establish an SCTP connection to the AMF:
```
[SCTP] Connect failed: Connection refused
[NGAP] Received unsuccessful result for SCTP association (3),
instance 0, cnx_id 1
```
Even after the PLMN and TAC fixes, the gNB was connecting to the wrong IP address
for the AMF. The confusion arose because different inspection methods returned
different results:
```
# docker inspect oai-amf gave:
IPAddress: 192.168.70.138

# But actual network assignment was:
oai-amf 192.168.70.135 (at one point)
oai-amf 192.168.70.132 (after static IPs enabled)
```

#### 5.3 Root Cause Analysis
Three compounding factors caused this issue:
- Issue 4 (dynamic IPs) meant the AMF IP changed on every core restart
- The `docker inspect` command with different format flags sometimes returned cached
  or stale network information
- The gNB log line `Parsed IPv4 address for NG AMF: 192.168.70.129` was misleading —
  it was actually printing the gNB's **own host IP**, not the AMF IP, causing initial
  confusion about which address was being used

The correct approach to identify the AMF IP was to list all containers at once:
```bash
docker inspect $(docker ps -aq --filter 'name=oai') \
  --format '{{.Name}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# Output:
# /oai-amf 192.168.70.132   <- Definitive answer
```

#### 5.4 Investigation Steps
- Used multiple `docker inspect` approaches to cross-verify the AMF IP
- Tested SCTP connectivity using `ss -lnp` inside the AMF container
- Confirmed the host could ping the AMF container successfully
- Verified the AMF was listening on SCTP port 38412 by checking AMF startup logs
- Determined the final correct IP after enabling static IPs in Issue 4

#### 5.5 Resolution
After enabling static IPs (Issue 4), the AMF was permanently assigned
`192.168.70.132`. The gNB configuration was updated to use this permanent address:
```bash
# Update AMF IP in gNB config
sed -i 's/amf_ip_address = ({ ipv4 = "192.168.70.135"/amf_ip_address = ({ ipv4 = "192.168.70.132"/' \
  ~/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf

# Verify
grep 'amf_ip_address' \
  ~/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf
# amf_ip_address = ({ ipv4 = "192.168.70.132"; });
```

---

## Final Result — Successful NG Setup

After resolving all five issues, the gNB was restarted and successfully completed
the NG Setup procedure with the AMF. The following log output confirms the
successful connection:
```
[NGAP] Send NGSetupRequest to AMF
[NGAP] 3584 -> 0000e000
[NGAP] Served GUAMIs for AMF OAI-AMF (assoc_id=14):
[NGAP] GUAMI:
[NGAP] PLMN: MCC=208, MNC=95
[NGAP] GUAMI:
[NGAP] PLMN: MCC=001, MNC=01
[NGAP] Supported PLMN 0: MCC=001 MNC=01   <-- PLMN MATCHED
[NGAP] Supported slice (PLMN 0): SST=0x01 SD=000
[NGAP] Received NGSetupResponse from AMF   <-- SUCCESS
[GNB_APP] [gNB 0] Received NGAP_REGISTER_GNB_CNF: associated AMF 1
[NR_RRC] cell PLMN 001.01 Cell ID 12345678 is in service
```
The USRP B210 hardware was also successfully initialized and transmitting:
```
[HW] Found USRP b200
[INFO] [B200] Detected Device: B210
[INFO] [B200] Operating over USB 3.
[HW] Actual master clock: 46.080000MHz...
[PHY] RU 0 RF started
[NR_MAC] Frame.Slot 384.0   <-- gNB transmitting frames
TYPE <CTRL-C> TO TERMINATE
```

### Final System State

| Component | Detail | Status |
|-----------|--------|--------|
| 5G Core (oai-cn5g-fed) | All 9 NFs healthy, NRF registered | ✅ Running |
| AMF | 192.168.70.132, PLMN 001/01, TAC 1 | ✅ Healthy |
| SMF | 192.168.70.133, N4 with UPF active | ✅ Healthy |
| UPF | 192.168.70.134, GTP-U ready | ✅ Healthy |
| gNB (nr-softmodem) | Band 78, 106 PRB, 3619.2 MHz | ✅ Transmitting |
| USRP B210 | USB 3.0, 46.08 MSps, TX/RX active | ✅ Active |
| NG Interface | gNB ↔ AMF SCTP + NGSetup | ✅ Established |
| PLMN | MCC=001, MNC=01 | ✅ Matched |
| TAC | 0x0001 (decimal 1) | ✅ Matched |

---

## Recommendations & Best Practices

### R1: Always Pre-create Bind-Mounted Files
Before running `docker-compose up`, verify that all files referenced in volume bind
mounts exist on the host as actual files. Use a pre-flight check script:
```bash
# Pre-flight check for all bind-mounted config files
grep -E './conf/' docker-compose-basic-nrf.yaml | awk '{print $1}' | while read f; do
  [ -f "$f" ] || echo "MISSING FILE: $f"
done
```

### R2: Always Use Static IP Addresses in Docker Compose
For any testbed or lab deployment (non-CI), always uncomment the static IP
assignments in the docker-compose file. Dynamic IPs introduce fragility into
configurations that reference container IPs directly.

### R3: Ensure PLMN and TAC Consistency Across All Components
Before launching the gNB, always verify that MCC, MNC, and TAC values match exactly
between the gNB config and the AMF `plmn_support_list`. A simple comparison command
can save hours of debugging:
```bash
# Compare PLMN/TAC between gNB and core
echo '=== gNB ===' && grep -i 'mcc\|mnc\|tac\|plmn' gnb.sa.band78.fr1.106PRB.usrpb210.conf
echo '=== AMF ===' && grep -i 'mcc\|mnc\|tac\|plmn' conf/basic_nrf_config.yaml
```

### R4: Use a Single Reliable Method for Container IP Discovery
Always use the following command to get container IPs — it shows all containers at
once and avoids the confusion of multiple `docker inspect` invocations:
```bash
docker inspect $(docker ps -aq --filter 'name=oai') \
  --format '{{.Name}} {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

### R5: Monitor AMF Logs During gNB Startup
Always watch AMF logs in a second terminal while starting the gNB. The AMF logs
provide the definitive reason for any NG Setup rejection:
```bash
# Terminal 1: Start gNB
sudo ./nr-softmodem -O \
  ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  --gNBs.[0].min_rxtxtime 6 -E --continuous-tx

# Terminal 2: Watch AMF logs
docker logs oai-amf -f 2>&1 | grep -i 'sctp\|ngap\|setup\|fail\|plmn\|tac'
```



## Database Configuration

The database initialization file used by the OAI CN5G-FED deployment is available below:

- [oai_db_basic.sql](../oai-cn5g-fed/oai_db_basic.sql)

