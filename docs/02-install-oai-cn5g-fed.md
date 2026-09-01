# 2. Installing OAI CN5G-Fed + RAN Stack

## System info (as deployed)

| Item | Value |
|------|-------|
| OS | Ubuntu 24.04.4 LTS (Noble) |
| Docker | Docker Engine 29.4.0, Docker Compose v5.1.1 |
| Machine | VI19000 — 5G Core + gNB (co-located) |
| USRP | NI USRP-2901 (B210), serial 30DBC30 |

## 1. UHD installation (same as plain CN5G — see [doc 1](01-install-oai-cn5g.md#1-uhd-usrp-hardware-driver--prerequisites-for-gnbue))

```bash
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils \
  doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev \
  libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy \
  python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

git clone https://github.com/EttusResearch/uhd.git ~/uhd
cd ~/uhd && git checkout v4.8.0.0
cd ~/uhd/host && mkdir build && cd build
cmake ../ && make -j$(nproc)
sudo make install && sudo ldconfig
sudo uhd_images_downloader
```

Verified: `sudo uhd_find_devices` → `B200 / NI2901 / serial 30DBC30`.

## 2. OAI CN5G-Fed core deployment

```bash
git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed.git ~/oai-cn5g-fed
cd ~/oai-cn5g-fed
git checkout v2.1.0
git submodule update --init --recursive

cd docker-compose
sudo apt install docker-compose-plugin -y
sudo ln -s /usr/libexec/docker/cli-plugins/docker-compose /usr/bin/docker-compose

python3 core-network.py --type start-basic --scenario 1
```

**Core network functions status (all healthy):** MySQL, oai-nrf, oai-udr, oai-udm, oai-ausf, oai-amf, oai-smf, oai-upf, oai-ext-dn

**NRF registration verified:**
```
AMF  → 192.168.70.138       SMF  → 192.168.70.137
UPF  → 192.168.70.133       AUSF → 192.168.70.134
UDM  → 192.168.70.135       UDR  → 192.168.70.136
SMF <-> UPF N4 Association: Active
UPF Heartbeats to SMF: Confirmed
```

## 3. OAI gNB build

```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g && git checkout develop
cd cmake_targets && ./build_oai -I
sudo apt install -y libforms-dev libforms-bin
./build_oai -w USRP --ninja --nrUE --gNB --build-lib 'nrscope' -C
```
Build result: `[10884/10884] Linking CXX executable nr-softmodem` — successful. Produces `nr-softmodem`, `nr-uesoftmodem`, `oai_usrpdevif`, `nrscope`, `libparams_libconfig.so`.

## 4. Running the stack

```bash
# Core
cd ~/oai-cn5g-fed/docker-compose
python3 core-network.py --type start-basic --scenario 1
python3 core-network.py --type stop-basic     # to stop

# gNB
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O .../gnb.sa.band78.fr1.106PRB.usrpb210.conf -E --continuous-tx
```

At this point the federated core and gNB are both operational — see
[doc 3](03-cn5g-fed-troubleshooting.md) for the real issues hit getting from "containers up" to
a fully working NG Setup, and [doc 5](05-cots-ue-with-cn5g-fed.md) for connecting COTS UEs on top of this core.

## References
- [oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed)
- [OpenAirInterface5G GitLab](https://gitlab.eurecom.fr/oai/openairinterface5g)
