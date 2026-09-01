# 1. Installing OAI CN5G + RAN Stack

## System info (as deployed)

| Item | Value |
|------|-------|
| OS | Ubuntu 24.04.4 LTS (Noble) |
| Docker | Docker Engine 29.4.0, Docker Compose v5.1.1 |
| Machine | VI19000 — 5G Core + gNB (co-located) |
| USRP | NI USRP-2901 (B210), serial 30DBC30 |
## 1. UHD (USRP Hardware Driver) — prerequisites for gNB/UE

**Install dependencies:**
```bash
sudo apt update
sudo apt install -y autoconf automake build-essential ccache cmake \
  cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev \
  libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev \
  python3-mako python3-numpy python3-requests python3-scipy \
  python3-setuptools python3-ruamel.yaml
```

**Clone and build UHD v4.8.0.0:**
```bash
cd ~
git clone https://github.com/EttusResearch/uhd.git ~/uhd
cd ~/uhd
git checkout v4.8.0.0

cd host && mkdir build && cd build
cmake ../
make -j$(nproc)
make test          # optional
sudo make install
sudo ldconfig
```

**Download USRP firmware/FPGA images:**
```bash
sudo uhd_images_downloader     # places images in /usr/local/share/uhd/images/
```

**Verify device connectivity** (USRP over USB 3.0 — USB 2.0 will limit performance or fail):
```bash
sudo uhd_find_devices
# Expected:
# [INFO] [UHD] Device found: device: B200, name: NI USRP-2901, serial: XXXXXXXX, product: B210

sudo uhd_usrp_probe   # optional deeper check: firmware, FPGA image, capabilities
```

## 2. Setting up the 5G Core (oai-cn5g)

**Basic tools + Docker Engine:**
```bash
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo ${UBUNTU_CODENAME:-$VERSION_CODENAME}) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Run Docker without sudo:**
```bash
sudo usermod -a -G docker $(whoami)
sudo reboot     # or log out/in
docker run hello-world   # should print "Hello from Docker!"
```

**Download OAI CN5G config files:**
```bash
wget -O ~/oai-cn5g.zip \
  "https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g"

unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
```
This gives a `~/oai-cn5g` directory with `docker-compose.yaml`, a `.env` file, and CN5G config templates.

**Pull and launch the core:**
```bash
cd ~/oai-cn5g
docker compose pull    # downloads AMF, SMF, UPF, NRF, AUSF, UDM, MySQL, WebUI, etc.
docker compose up -d
docker compose down     # to stop
```

## 3. Setting up the 5G gNB and UE

**Clone OpenAirInterface5G (`develop` branch — required for latest 5G SA support):**
```bash
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop
```

**Install build dependencies:**
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai -I
```

**(Optional) NR Scope GUI dependencies** for real-time IQ visualization:
```bash
sudo apt install -y libforms-dev libforms-bin
```

**Build gNB + UE with USRP (UHD) support:**
```bash
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

At this point the system has a working core (`~/oai-cn5g`) and compiled `nr-softmodem`/`nr-uesoftmodem` binaries ready to launch, as used throughout the [multi-gNB/UE testbed experiments](https://github.com/<your-username>/oai-5g-sa-multi-gnb-ue-testbed).


## Database Configuration

The database initialization file used by the OAI CN5G deployment is available below:

- [oai_db.sql](../oai-cn5g/oai_db.sql)


## References
- [oai-cn5g tutorial resources](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/doc/tutorial_resources/oai-cn5g)
- [OpenAirInterface5G GitLab](https://gitlab.eurecom.fr/oai/openairinterface5g)
