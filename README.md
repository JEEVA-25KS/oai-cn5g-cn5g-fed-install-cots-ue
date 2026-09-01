# OAI CN5G & CN5G-Fed Installation + COTS UE Connectivity

End-to-end  installation of both OAI 5G Core variants — plain **oai-cn5g** and the
federated **oai-cn5g-fed** — alongside the OAI RAN stack (gNB/UE with USRP support),
and successful connection of commercial-off-the-shelf (COTS) Android smartphones to
each core, including full troubleshooting logs for real issues hit along the way.

## Contents

| # | Doc | What it covers |
|---|-----|-----------------|
| 1 | [Installing OAI CN5G + RAN](docs/01-install-oai-cn5g.md) | UHD/USRP setup, Docker, OAI CN5G core deployment, gNB/UE build |
| 2 | [Installing OAI CN5G-Fed + RAN](docs/02-install-oai-cn5g-fed.md) | Federated core deployment via `core-network.py`, NRF registration, gNB build |
| 3 | [CN5G-Fed troubleshooting (5 real issues resolved)](docs/03-cn5g-fed-troubleshooting.md) | Missing config files, PLMN/TAC mismatches, dynamic Docker IPs, wrong AMF IP |
| 4 | [COTS UE → OAI CN5G](docs/04-cots-ue-with-cn5g.md) | Connecting a Samsung Galaxy M13 5G to the plain CN5G core — subscriber provisioning, APN config, 120 Mbps result |
| 5 | [COTS UE → OAI CN5G-Fed](docs/05-cots-ue-with-cn5g-fed.md) | Connecting multiple Android phones to CN5G-Fed — PLMN/database mismatch, IP subnet fix, critical NIA0→NIA2 security fix |

## Testbed hardware/software

- **RF hardware**: NI USRP-2901 (B210), UHD v4.8.0.0 built from source
- **RAN**: OpenAirInterface5G (`develop` branch) — `nr-softmodem` (gNB), `nr-uesoftmodem` (UE)
- **Cores tested**: `oai-cn5g` (plain Docker Compose deployment) and `oai-cn5g-fed` v2.1.0 (federated, deployed via `core-network.py`)
- **COTS UEs**: Samsung Galaxy M13 5G and additional Android phones, each with a 5G-programmable SIM (custom IMSI/Ki/OPc)
- **Band/config**: Band 78, FR1, 106 PRB, `gnb.sa.band78.fr1.106PRB.usrpb210.conf`

## Why two cores?

`oai-cn5g` is the simpler, tutorial-style core (single Docker Compose stack) — good for
quick bring-up and single-machine testing. `oai-cn5g-fed` is EURECOM's federated
core distribution, deployed via a Python helper script (`core-network.py`) with more
production-like NF separation and NRF-based service registration. Testing COTS UE
connectivity against both validates the RAN stack against two independent core
implementations and surfaces different classes of integration issues (see the
[troubleshooting doc](docs/03-cn5g-fed-troubleshooting.md) and the
[CN5G-Fed COTS UE doc](docs/05-cots-ue-with-cn5g-fed.md) for real examples).

## Key results

- OAI CN5G + COTS UE (Galaxy M13 5G): registered successfully, **120 Mbps** peak throughput
- OAI CN5G-Fed + COTS UE (×2 phones): both registered and passed active data-plane traffic on LCID 4 after resolving PLMN/database, DNS, and NAS security algorithm issues
- CN5G-Fed troubleshooting: 5 distinct issues diagnosed and resolved end-to-end, from Docker bind-mount errors to dynamic container IP instability

## References
- [OAI CN5G](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/doc/tutorial_resources/oai-cn5g)
- [oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed)
- [OpenAirInterface5G](https://gitlab.eurecom.fr/oai/openairinterface5g)
- [UHD (Ettus Research)](https://github.com/EttusResearch/uhd)

---
*Note: QoS-flow and PCF/eBPF-related experiments performed on top of these cores are
documented in a separate repository, since they represent a distinct line of work
(dynamic PCC rule enforcement, per-QFI datapath).*
