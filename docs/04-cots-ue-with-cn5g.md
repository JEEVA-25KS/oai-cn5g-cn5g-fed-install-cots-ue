# 4. Connecting a COTS UE (Galaxy M13 5G) to OAI CN5G

## Objective
Connect a commercial-off-the-shelf (COTS) Android UE to the OAI SA testbed running the plain `oai-cn5g` core.

## Subscriber provisioning

Subscriber records come from `oai_db.sql` (OAI tutorial resources). IMSI/OPc/Ki for the SIM cards used:
```
k   = "465B5CE8B199B49FAA5F0A2EE238A6BC"
opc = "E8ED289DEBA952E4283B54E88E6183CA"

IMSI SIM 1: 001010010002559   # UE 1
IMSI SIM 2: 001010010002589   # UE 2
IMSI SIM 3: 001010010002568   # CPE
```
These are added to the OAI database, a static IP is assigned per subscriber, and the user is listed in `users.conf`:
- `doc/tutorial_resources/oai-cn5g/conf/users.conf`
- IMSI/OPc/Ki added to `targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf`
- UE DNS set in `config.yaml`:
  ```yaml
  ue_dns:
    primary_ipv4: "8.8.8.8"
  ```

## Launch core + gNB
```bash
cd ~/oai-cn5g && docker compose up -d

cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb.sa.band78.fr1.106PRB.usrpb210.conf -E --continuous-tx
```

## UE-side configuration

On the Android phone, before starting core/gNB, configure the APN:
```
Settings → Connections → Mobile networks → Network mode (5G only) → Access Point Names → Add
  Name: oai
  APN:  oai
  MCC:  001
  MNC:  01
```
Save, select the APN, then restart or toggle airplane mode.

## Connect and verify

```bash
# Watch AMF logs to confirm UE registration
sudo docker logs -f oai-amf
```
Check the UE's assigned IP matches the core's database entry. To capture the full attach/registration sequence:
```bash
sudo tcpdump -i any sctp -w rec.pcap
# or, broader capture:
sudo tcpdump -i any sctp or udp port 2152 or udp port 38472 or udp port 38462 -w rec.pcap
```
Sequence: start tcpdump → start gNB → connect phone → wait for 5G icon → browse to generate traffic → disconnect phone → stop gNB → stop tcpdump.

## Result
UE registered successfully to the OAI core and connected to the network, achieving a **maximum throughput of 120 Mbps**.

<img width="1248" height="699" alt="image" src="https://github.com/user-attachments/assets/9fbd2540-8a2b-4691-8799-7486ea3da21b" />

## Database Configuration
The database initialization file used by the OAI CN5G deployment is available below:

- [oai_db.sql](../oai-cn5g/oai_db.sql)


## References
- `doc/tutorial_resources/oai-cn5g/database/oai_db.sql`
- `doc/tutorial_resources/oai-cn5g/conf/users.conf`
- `targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf`
