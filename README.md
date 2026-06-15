# Multi-UE 5G Testbed Using Open5GS, OCUDU, ZeroMQ, GNU Radio, and Raspberry Pis

## Demo Video

<iframe src="[https://google.com](https://drive.google.com/file/d/1E4vb3TunYGgHGL87wx9UEeWyqkx5X8yH/view?usp=sharing)" width="640" height="480" allow="autoplay"></iframe>

## Overview

This repository documents the setup of a software-defined 5G network using:

* Open5GS 5G Core (Docker)
* srsRAN Project OCUDU gNB
* GNU Radio ZeroMQ broker
* Three Raspberry Pi devices acting as UEs
* srsRAN_4G `srsUE` operating in NR mode

The objective is to establish a multi-UE 5G environment suitable for:

* UE-to-Core communication
* UE-to-UE communication
* Traffic generation and monitoring
* Post-Quantum Cryptography (PQC) experiments
* TLS performance evaluation over 5G

---

# System Architecture

```text
                    Ubuntu VM
 ┌────────────────────────────────────────────┐
 │                                            │
 │  Open5GS 5GC                              │
 │  OCUDU gNB                                │
 │  GNU Radio ZMQ Broker                     │
 │                                            │
 └────────────────────────────────────────────┘
                  │
                  │ Ethernet LAN
                  │
 ┌────────────┬────────────┬────────────┐
 │ Raspberry  │ Raspberry  │ Raspberry  │
 │ Pi UE1     │ Pi UE2     │ Pi UE3     │
 │ .10        │ .20        │ .30        │
 └────────────┴────────────┴────────────┘
```

---

# Network Addresses

## Physical LAN

| Device | Address      |
| ------ | ------------ |
| VM     | 192.168.0.4  |
| UE1    | 192.168.0.10 |
| UE2    | 192.168.0.20 |
| UE3    | 192.168.0.30 |

## 5G Tunnel Addresses

| Device  | Tunnel IP |
| ------- | --------- |
| UE1     | 10.45.1.2 |
| UE2     | 10.45.1.3 |
| UE3     | 10.45.1.4 |
| Gateway | 10.45.1.1 |

---

# Software Components

## VM

### Open5GS

```bash
docker compose up 5gc
```

### OCUDU gNB

```bash
cd ~/ocudu/build/apps/gnb
sudo ./gnb -c ~/configs/gnb_ocudu_zmq_pi2.yaml
```

### GNU Radio Broker

```bash
python3 multi_ue_scenario.py
```

---

# Subscriber Configuration

Edit:

```bash
~/ocudu/docker/open5gs/subscriber_db.csv
```

Example:

```csv
ue01,001010123456780,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.2
ue02,001010123456781,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.3
ue03,001010123456782,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,9001,9,10.45.1.4
```

Verify:

```bash
cat ~/ocudu/docker/open5gs/open5gs.env
```

```text
SUBSCRIBER_DB=subscriber_db.csv
```

Restart Open5GS:

```bash
docker compose down
docker compose up --build 5gc
```

---

# GNU Radio Broker Port Mapping

## gNB

```text
TX -> 2000
RX -> 2001
```

## UE1

```text
RX -> 2100
TX -> 2101
```

## UE2

```text
RX -> 2200
TX -> 2201
```

## UE3

```text
RX -> 2300
TX -> 2301
```

---

# UE Configuration

## UE1

```ini
device_args = tx_port=tcp://0.0.0.0:2101,rx_port=tcp://192.168.0.4:2100,base_srate=11.52e6

imsi = 001010123456780

[gw]
netns = ue1
```

## UE2

```ini
device_args = tx_port=tcp://0.0.0.0:2201,rx_port=tcp://192.168.0.4:2200,base_srate=11.52e6

imsi = 001010123456781

[gw]
netns = ue2
```

## UE3

```ini
device_args = tx_port=tcp://0.0.0.0:2301,rx_port=tcp://192.168.0.4:2300,base_srate=11.52e6

imsi = 001010123456782

[gw]
netns = ue3
```

---

# Startup Procedure

## Step 1: Start Open5GS

```bash
cd ~/ocudu/docker
docker compose up 5gc
```

Verify:

```bash
docker logs -f open5gs_5gc
```

---

## Step 2: Start gNB

```bash
cd ~/ocudu/build/apps/gnb

sudo ./gnb \
-c ~/configs/gnb_ocudu_zmq_pi2.yaml
```

Expected:

```text
gNB-N2 accepted
```

---

## Step 3: Start GNU Radio Broker

```bash
python3 multi_ue_scenario.py
```

Recommended settings:

```text
Slow Down Ratio = 1

UE1 Pathloss = 0 dB
UE2 Pathloss = 0 dB
UE3 Pathloss = 0 dB
```

---

## Step 4: Start UE1

```bash
sudo pkill -9 srsue

sudo ip netns delete ue1 2>/dev/null
sudo ip netns add ue1

sudo ./srsue ./ue1_zmq.conf
```

---

## Step 5: Start UE2

```bash
sudo pkill -9 srsue

sudo ip netns delete ue2 2>/dev/null
sudo ip netns add ue2

sudo ./srsue ./ue2_zmq.conf
```

---

## Step 6: Start UE3

```bash
sudo pkill -9 srsue

sudo ip netns delete ue3 2>/dev/null
sudo ip netns add ue3

sudo ./srsue ./ue3_zmq.conf
```

---

# Verification

## Verify Subscriber Registration

Open5GS should show:

```text
UE SUPI[imsi-001010123456780]
IPv4[10.45.1.2]

UE SUPI[imsi-001010123456781]
IPv4[10.45.1.3]

UE SUPI[imsi-001010123456782]
IPv4[10.45.1.4]
```

---

## Verify gNB

Expected:

```text
rnti=0x4601
rnti=0x4602
rnti=0x4603

PUSCH crc=OK
PUCCH ack=1
```

---

## Verify Routes

```bash
sudo ip netns exec ue1 ip route
```

Expected:

```text
10.45.1.0/24 dev tun_srsue
default via 10.45.1.1 dev tun_srsue
```

---

## Verify Gateway Connectivity

```bash
sudo ip netns exec ue1 ping 10.45.1.1

sudo ip netns exec ue2 ping 10.45.1.1

sudo ip netns exec ue3 ping 10.45.1.1
```

---

# Troubleshooting

## UE Stuck at "Attaching UE"

### Cause

ZMQ connection not established.

### Verify

```bash
ss -ntp | grep -E "2000|2001|2101|2201|2301"
```

Expected:

```text
ESTAB
ESTAB
ESTAB
```

---

## gNB Shows

```text
Waiting for reading samples.
Completed 0 of 11520 samples
```

### Cause

Broker not forwarding RF samples.

### Check

```bash
python3 multi_ue_scenario.py
```

and

```bash
ss -lntup
```

---

## No Tunnel Interface Created

Verify successful attach:

```bash
tail -f /tmp/ue.log
```

Expected:

```text
PDU Session Establishment successful
```

---

## Subscriber Not Found

Verify:

```bash
cat subscriber_db.csv
```

and IMSI matches UE config.

---

## Error Reading Subscriber Database

```text
not enough values to unpack
```

Cause:

Incorrect CSV format.

Verify:

```csv
name,imsi,key,opc,...
```

contains no blank lines.

---

## libsrsran_rf.so.0 Missing

Rebuild:

```bash
rm -rf build

mkdir build
cd build

cmake ../ -DENABLE_RF_PLUGINS=OFF

make -j$(nproc)

sudo make install

sudo ldconfig
```

---

## Raspberry Pi SSH Connectivity Issues

### Symptoms

SSH becomes unreachable.

### Solution

```bash
sudo ip link set eth0 down
sleep 5
sudo ip link set eth0 up
```

If necessary:

```bash
sudo reboot
```

---

## Recommended Startup Order

The most reliable sequence observed was:

1. Open5GS
2. gNB
3. GNU Radio Broker
4. UE1
5. UE2
6. UE3

Wait approximately 10–15 seconds between UE launches.

This sequence consistently produced successful multi-UE attachment and stable operation.

---



# UE-to-UE Communication

Once all UEs are attached:

```bash
sudo ip netns exec ue1 ping 10.45.1.3

sudo ip netns exec ue1 ping 10.45.1.4
```

Verify connectivity before attempting PQC experiments.

---


# PQC Experiments

Recommended:

## Baseline

```text
TLS 1.3 + X25519
```

## PQC

```text
TLS 1.3 + ML-KEM-768
```

## Hybrid

```text
TLS 1.3 + X25519 + ML-KEM-768
```

Suggested libraries:

* wolfSSL
* BoringSSL

Run client/server inside UE namespaces:

```bash
sudo ip netns exec ue1 ...
sudo ip netns exec ue2 ...
```

and measure:

* Handshake latency
* Packet count
* Handshake size
* CPU utilization
* Throughput
* Re-keying cost


