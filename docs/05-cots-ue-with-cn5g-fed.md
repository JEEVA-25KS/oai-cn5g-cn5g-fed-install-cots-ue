# 5. Connecting COTS UEs to OAI CN5G-Fed — Configuration & Troubleshooting

Two Android phones (plus a third CPE SIM) were connected to the federated core
(`oai-cn5g-fed`). This required six distinct fixes across the MySQL subscriber database,
SMF config, and phone-side APN setup.

## Summary of changes

| # | Change | File/Component | What changed | Result |
|---|--------|-----------------|---------------|--------|
| 1 | Switch MySQL database file | `docker-compose-basic-nrf.yaml` | `oai_db2.sql` → `oai_db-basic.sql` | ✅ Fixed |
| 2 | Fix static IPs in database | MySQL (live + `oai_db-basic.sql`) | `10.0.0.x` → `12.1.1.13x` | ✅ Fixed |
| 3 | Fix DNS in SMF config | `basic_nrf_config.yaml` | `172.21.3.100` → `8.8.8.8` | ✅ Fixed |
| 4 | Change security algorithm priority | `basic_nrf_config.yaml` | NIA0 first → NIA2 first | ✅ Fixed |
| 5 | Update SQL file for persistence | `oai_db-basic.sql` | `10.0.0.x` IPs updated in file | ✅ Fixed |
| 6 | Android APN configuration | Phone settings | APN set to `oai`, IPv4 | ✅ Fixed |

## 1. Wrong MySQL database file (PLMN mismatch)

**Problem:** MySQL was initialized from `oai_db2.sql`, which contains only PLMN 208/95 subscribers. The core was running PLMN 001/01, so no UE could register — its IMSI simply wasn't in the database.

**Discovery:**
```bash
docker exec mysql mysql -u test -ptest oai_db -e "SELECT ueid FROM AuthenticationSubscription;"
# 131 entries, all 20895xxxxxxxx — wrong PLMN entirely
```
```yaml
# Wrong in docker-compose-basic-nrf.yaml:
volumes:
  - ./database/oai_db2.sql:/docker-entrypoint-initdb.d/oai_db.sql
```

**Fix:**
```bash
sed -i 's/oai_db2.sql/oai_db-basic.sql/' \
  ~/oai-cn5g-fed/docker-compose/docker-compose-basic-nrf.yaml

cd ~/oai-cn5g-fed/docker-compose
python3 core-network.py --type stop-basic
docker volume prune -f
python3 core-network.py --type start-basic --scenario 1
```
Verified subscribers now included the correct PLMN 001/01 IMSIs, including the three COTS UEs (`001010010002559`, `001010010002568`, `001010010002589`).

## 2. Static IPs outside the UPF's routable subnet

**Problem:** COTS UE subscribers had static IPs in `10.0.0.x`, but the UPF's N6 interface subnet is `12.1.1.128/25` — an unroutable IP means PDU session establishment fails (UPF can't create a GTP tunnel for an address it can't route).

**Fix:**
```bash
docker exec mysql mysql -u test -ptest oai_db -e "
UPDATE SessionManagementSubscriptionData
SET dnnConfigurations = REPLACE(dnnConfigurations, '10.0.0.2', '12.1.1.130')
WHERE ueid = '001010010002589';
-- (repeated for the other two IMSIs → 12.1.1.131, 12.1.1.132)
"
```
Final assignment: `.2589 → 12.1.1.130`, `.2559 → 12.1.1.131`, `.2568 → 12.1.1.132` (all within `12.1.1.128/25`).

## 3. Unreachable DNS server in SMF config

**Problem:** `basic_nrf_config.yaml` had `primary_ipv4: "172.21.3.100"` — a nonexistent internal IP sent to UEs via PCO. Result: UE gets a data connection but can't resolve domain names.

**Fix:**
```bash
sed -i 's/primary_ipv4: "172.21.3.100"/primary_ipv4: "8.8.8.8"/' \
  ~/oai-cn5g-fed/docker-compose/conf/basic_nrf_config.yaml
```

## 4. NAS security algorithm priority (the critical blocker)

**Problem:** AMF listed `NIA0` (null integrity) first in `supported_integrity_algorithms`. Modern Android phones **refuse NIA0** — they silently drop the Security Mode Command instead of returning Security Mode Complete, so registration times out repeatedly.

**Evidence:**
```
[amf_n1] Authentication successful by network!   <- auth OK
[amf_n1] Selected AMF NIA: 0x0                     <- NIA0 selected
... (silence, no Security Mode Complete) ...
[amf_n2] UE Context Release Complete               <- timeout, UE released
```

**Fix** — reorder so NIA2/NEA2 come first:
```yaml
# Before:
supported_integrity_algorithms: ["NIA0", "NIA1", "NIA2"]
supported_encryption_algorithms: ["NEA0", "NEA1", "NEA2"]
# After:
supported_integrity_algorithms: ["NIA2", "NIA1", "NIA0"]
supported_encryption_algorithms: ["NEA2", "NEA1", "NEA0"]
```
After this fix, the AMF selects NIA2 (AES-128-CMAC) / NEA2 (AES-128-CTR), which Android accepts, and registration completes.

## 5. Persisting the IP fix across restarts
The live database `UPDATE` in fix #2 only survives until the MySQL volume is recreated. Applied the same substitution to the source SQL file so it survives restarts:
```bash
sed -i 's/10\.0\.0\.2/12.1.1.130/g; s/10\.0\.0\.3/12.1.1.131/g; s/10\.0\.0\.4/12.1.1.132/g' \
  ~/oai-cn5g-fed/docker-compose/database/oai_db-basic.sql
```

## 6. Android APN configuration

| Setting | Value |
|---------|-------|
| APN Name | OAI |
| APN (DNN) | oai |
| APN Type | default |
| APN Protocol | IPv4 |
| Bearer | NR/5G |

Plus: preferred network type set to NR/5G only, WiFi disabled to force cellular, airplane mode toggled to force re-search, SIM programmed with the matching IMSI/Ki/OPc.

## Final result
Two COTS UEs successfully registered and established active data-plane connectivity — visible as live traffic on LCID 4 (the data radio bearer) for both:
```
UE RNTI 322f: LCID 4: TX 24,578,066 RX 2,073,862 bytes   <- data plane active
UE RNTI 89ad: LCID 4: TX  5,948,426 RX 1,249,516 bytes   <- data plane active
```

| Component | Status |
|-----------|--------|
| 5G Core (9 NFs) | ✅ Running, PLMN 001/01 |
| MySQL DB | ✅ `oai_db-basic.sql`, correct subscribers |
| gNB | ✅ Transmitting (Band 78, 106 PRB, USRP B210) |
| NG Interface | ✅ Connected |
| COTS UE 1 (...2559) | ✅ Registered, PDU session active, 12.1.1.131 |
| COTS UE 2 (...2589) | ✅ Registered, PDU session active, 12.1.1.130 |
| NAS Security | ✅ NIA2 + NEA2 (AES-128) |
| DNS | ✅ Primary/secondary 8.8.8.8 |


## Database Configuration

The database initialization file used by the OAI CN5G-FED deployment is available below:

- [oai_db_basic.sql](../oai-cn5g-fed/oai_db_basic.sql)
