# OCUDU-srsRAN-5G-ZMQ-Testbed-with-Three-Raspberry-Pi-UEs

OCUDU / srsRAN 5G ZMQ Testbed with Three Raspberry Pi UEs

This repository documents the setup of a software-based 5G testbed using:

Open5GS 5GC in Docker
OCUDU gNB with ZeroMQ RF
srsRAN_4G srsUE as 5G NR UE
Three Raspberry Pis as remote UE nodes
GNU Radio ZMQ broker for multi-UE RF emulation

The setup was developed for UE-to-UE communication experiments and future PQC/TLS testing over a controlled 5G software network.

1. Final Network Architecture
                 Ubuntu VM
        --------------------------------
        Open5GS 5GC + OCUDU gNB
                  |
                  | ZMQ RF
                  |
            GNU Radio Broker
          /          |           \
         /           |            \
   Pi UE1        Pi UE2        Pi UE3

Example LAN addresses:

VM / gNB / Broker: 192.168.0.4
Pi UE1:            192.168.0.10
Pi UE2:            192.168.0.20
Pi UE3:            192.168.0.30

UE tunnel IPs assigned by Open5GS:

UE1: 10.45.1.2
UE2: 10.45.1.3
UE3: 10.45.1.4
Gateway: 10.45.1.1
2. Important Lessons Learned

A direct ZMQ gNB configuration supports only one UE at a time.

This works for one UE:

gNB <---- ZMQ ----> Pi UE1

This does not work for multiple UEs:

gNB <---- same ZMQ ports ----> UE1, UE2, UE3

For multiple UEs, a GNU Radio ZMQ broker is required. The broker copies the downlink signal from the gNB to all UEs and combines uplink signals from the UEs back to the gNB.

3. Open5GS Subscriber Database

On the VM:

cd ~/ocudu/docker/open5gs
nano subscriber_db.csv

Use:

ue01,001010123456780,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.2
ue02,001010123456781,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.3
ue03,001010123456782,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.4

Check:

nano ~/ocudu/docker/open5gs/open5gs.env

Set:

SUBSCRIBER_DB="subscriber_db.csv"

Restart Open5GS:

cd ~/ocudu/docker
docker compose down
docker compose up --build 5gc
4. gNB Configuration

Example file:

~/configs/gnb_ocudu_zmq_pi2.yaml

Use this for multi-UE broker mode:

cu_cp:
  amf:
    addr: 10.53.1.2
    port: 38412
    bind_addr: 10.53.1.1
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  inactivity_timer: 7200

cu_up:
  ngu:
    socket:
      - bind_addr: 10.53.1.1
        bind_interface: auto
        ext_addr: 10.53.1.1

ru_sdr:
  device_driver: zmq
  device_args: tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=11.52e6
  srate: 11.52
  tx_gain: 0
  rx_gain: 0

cell_cfg:
  dl_arfcn: 368500
  band: 3
  channel_bandwidth_MHz: 10
  common_scs: 15
  plmn: "00101"
  tac: 7
  pdcch:
    common:
      ss0_index: 0
      coreset0_index: 6
    dedicated:
      ss2_type: common
      dci_format_0_1_and_1_1: false
  prach:
    prach_config_index: 1
    total_nof_ra_preambles: 64
    nof_ssb_per_ro: 1
    nof_cb_preambles_per_ssb: 64
  pdsch:
    mcs_table: qam64
  pusch:
    mcs_table: qam64

log:
  filename: /tmp/gnb.log
  all_level: info
  hex_max_size: 0

pcap:
  mac_enable: false
  mac_filename: /tmp/gnb_mac.pcap
  ngap_enable: false
  ngap_filename: /tmp/gnb_ngap.pcap

Important: for broker mode, the gNB ZMQ address must use 127.0.0.1, because the GNU Radio broker runs on the same VM.

5. GNU Radio Broker Configuration

The broker uses:

gNB TX: 2000
gNB RX: 2001

UE1 RX: 2100
UE1 TX: 2101

UE2 RX: 2200
UE2 TX: 2201

UE3 RX: 2300
UE3 TX: 2301

In multi_ue_scenario.py, the key ZMQ lines should look like:

self.zeromq_req_source_1 = zeromq.req_source(..., 'tcp://192.168.0.10:2101', ...)
self.zeromq_req_source_1_0 = zeromq.req_source(..., 'tcp://192.168.0.20:2201', ...)
self.zeromq_req_source_0_0 = zeromq.req_source(..., 'tcp://192.168.0.30:2301', ...)

self.zeromq_req_source_0 = zeromq.req_source(..., 'tcp://127.0.0.1:2000', ...)
self.zeromq_rep_sink_0_1 = zeromq.rep_sink(..., 'tcp://127.0.0.1:2001', ...)

self.zeromq_rep_sink_0 = zeromq.rep_sink(..., 'tcp://0.0.0.0:2100', ...)
self.zeromq_rep_sink_0_0 = zeromq.rep_sink(..., 'tcp://0.0.0.0:2200', ...)
self.zeromq_rep_sink_0_2 = zeromq.rep_sink(..., 'tcp://0.0.0.0:2300', ...)

Recommended broker settings:

Time Slow Down Ratio = 1
UE1 Pathloss = 0 dB
UE2 Pathloss = 0 dB
UE3 Pathloss = 0 dB

Run broker:

python3 -u ~/multi_ue_scenario.py

Verify ports:

ss -lntup | grep -E "2100|2200|2300"
ss -ntp | grep -E "2101|2201|2301"

Expected:

0.0.0.0:2100 LISTEN
0.0.0.0:2200 LISTEN
0.0.0.0:2300 LISTEN

192.168.0.10:2101 ESTAB
192.168.0.20:2201 ESTAB
192.168.0.30:2301 ESTAB
6. UE Configurations
UE1 on Pi 192.168.0.10
[rf]
freq_offset = 0
tx_gain = 50
rx_gain = 40
srate = 11.52e6
nof_antennas = 1

device_name = zmq
device_args = tx_port=tcp://0.0.0.0:2101,rx_port=tcp://192.168.0.4:2100,base_srate=11.52e6

[rat.eutra]
dl_earfcn = 2850
nof_carriers = 0

[rat.nr]
bands = 3
nof_carriers = 1
max_nof_prb = 52
nof_prb = 52

[pcap]
enable = none
mac_filename = /tmp/ue_mac.pcap
mac_nr_filename = /tmp/ue_mac_nr.pcap
nas_filename = /tmp/ue_nas.pcap

[log]
all_level = info
phy_lib_level = none
all_hex_limit = 32
filename = /tmp/ue.log
file_max_size = -1

[usim]
mode = soft
algo = milenage
opc  = 63BFA50EE6523365FF14C1F45F88737D
k    = 00112233445566778899aabbccddeeff
imsi = 001010123456780
imei = 353490069873319

[rrc]
release = 15
ue_category = 4

[nas]
apn = internet
apn_protocol = ipv4

[gw]
netns = ue1
ip_devname = tun_srsue
ip_netmask = 255.255.255.0

[gui]
enable = false
UE2 on Pi 192.168.0.20

Change:

device_args = tx_port=tcp://0.0.0.0:2201,rx_port=tcp://192.168.0.4:2200,base_srate=11.52e6
imsi = 001010123456781

[gw]
netns = ue2
UE3 on Pi 192.168.0.30

Change:

device_args = tx_port=tcp://0.0.0.0:2301,rx_port=tcp://192.168.0.4:2300,base_srate=11.52e6
imsi = 001010123456782

[gw]
netns = ue3
7. Starting the System

Use this clean start procedure.

On the VM

Stop old processes:

sudo pkill -9 gnb
pkill -9 -f multi_ue_scenario.py

Start Open5GS:

cd ~/ocudu/docker
docker compose up 5gc

Start gNB:

cd ~/ocudu/build/apps/gnb
sudo ./gnb -c ~/configs/gnb_ocudu_zmq_pi2.yaml

Start GNU Radio broker:

python3 -u ~/multi_ue_scenario.py
On Pi UE1
sudo pkill -9 srsue
sudo ip netns delete ue1 2>/dev/null
sudo ip netns add ue1
cd ~/srsRAN_4G/build/srsue/src
sudo ./srsue ./ue1_zmq.conf
On Pi UE2
sudo pkill -9 srsue
sudo ip netns delete ue2 2>/dev/null
sudo ip netns add ue2
cd ~/srsRAN_4G/build/srsue/src
sudo ./srsue ./ue2_zmq.conf
On Pi UE3
sudo pkill -9 srsue
sudo ip netns delete ue3 2>/dev/null
sudo ip netns add ue3
cd ~/srsRAN_4G/build/srsue/src
sudo ./srsue ./ue3_zmq.conf
8. Expected Successful UE Attach

Each UE should show:

Random Access Complete
RRC Connected
PDU Session Establishment successful. IP: 10.45.1.x
RRC NR reconfiguration successful

Open5GS should show:

UE SUPI[imsi-001010123456780] IPv4[10.45.1.2]
UE SUPI[imsi-001010123456781] IPv4[10.45.1.3]
UE SUPI[imsi-001010123456782] IPv4[10.45.1.4]

gNB log may show:

rnti=0x4601
rnti=0x4602
rnti=0x4603
PUSCH crc=OK
PUCCH ack=1
9. UE Routing

After attach, check each UE:

sudo ip netns exec ue1 ip a
sudo ip netns exec ue1 ip route

Expected UE1:

tun_srsue: 10.45.1.2/24
10.45.1.0/24 dev tun_srsue src 10.45.1.2

Add default route if needed:

sudo ip netns exec ue1 ip route replace default via 10.45.1.1 dev tun_srsue

Similarly:

sudo ip netns exec ue2 ip route replace default via 10.45.1.1 dev tun_srsue
sudo ip netns exec ue3 ip route replace default via 10.45.1.1 dev tun_srsue
10. Connectivity Tests

Gateway test:

sudo ip netns exec ue1 ping -c 4 10.45.1.1
sudo ip netns exec ue2 ping -c 4 10.45.1.1
sudo ip netns exec ue3 ping -c 4 10.45.1.1

UE-to-UE test:

sudo ip netns exec ue1 ping -c 4 10.45.1.3
sudo ip netns exec ue1 ping -c 4 10.45.1.4
sudo ip netns exec ue2 ping -c 4 10.45.1.2
sudo ip netns exec ue3 ping -c 4 10.45.1.2

If pings are very slow, reduce GNU Radio slow-down ratio:

Time Slow Down Ratio = 1
11. Known Issues and Fixes
Problem: UE stuck at Attaching UE...

Check:

ss -ntp | grep -E "2000|2001|2101|2201|2301"

Make sure broker has established connections to all Pis.

Problem: gNB shows Waiting for reading samples. Completed 0 of 11520 samples

This means the gNB is not receiving RF samples from the broker. Check:

gNB uses 127.0.0.1:2000/2001
broker is running
all three UE uplink connections are established
GNU Radio pathloss is not too high
slow-down ratio is set to 1
Problem: UE gets IP but cannot ping gateway

Check gNB cu_up binding:

cu_up:
  ngu:
    socket:
      - bind_addr: 10.53.1.1
        bind_interface: auto
        ext_addr: 10.53.1.1

Also check VM bridge:

ip a | grep -A3 10.53
Problem: scp: local "srsRAN_4G" is not a regular file

Use recursive copy:

scp -rp ~/srsRAN_4G ue3@192.168.0.30:/home/ue3/
Problem: libsrsran_rf.so.0 missing

Rebuild and install on the Pi:

cd ~/srsRAN_4G
rm -rf build
mkdir build
cd build
cmake ../ -DENABLE_RF_PLUGINS=OFF
make -j$(nproc)
sudo make install
sudo ldconfig
12. PQC/TLS Experiment Layer

Once UE-to-UE IP communication works, PQC is tested at the application layer.

Example:

UE1: TLS/PQC client
UE2: TLS/PQC server

The 5G network provides IP connectivity. PQC/TLS runs between UE tunnel IP addresses such as:

10.45.1.2 <-> 10.45.1.3

Recommended first test:

Baseline: TLS 1.3 X25519
PQC: TLS 1.3 Hybrid X25519 + ML-KEM-768

wolfSSL is recommended for Raspberry Pi experiments because it is lightweight and easier to use with standalone client/server examples.

13. Final Notes

The most stable tested path was:

Verify one UE direct ZMQ attach first.
Add GNU Radio broker.
Attach three UEs.
Confirm Open5GS allocates three IPs.
Confirm gateway ping.
Confirm UE-to-UE ping.
Run PQC/TLS experiments.

The main technical challenge was not Open5GS registration, but correct ZMQ RF sample routing among gNB, GNU Radio broker, and multiple remote Raspberry Pi UEs.
