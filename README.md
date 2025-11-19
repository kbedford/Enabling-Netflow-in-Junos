---

# Enabling NetFlow/IPFIX Export on Junos (inline jflow → Collector)



In this lab I enabled inline jflow (NetFlow/IPFIX) on a Junos vMX router and exported flows to a Netflow collector (NorthStar Analytics `netflowd`).

This post walks through:

* The Junos configuration for inline jflow/IPFIX
* Enabling sampling on interfaces
* Basic verification on the router
* What to expect on the collector (logs + tcpdump)
* Common troubleshooting steps

---

## 1. Lab Overview

**Router**

* Platform: vMX (`vmx101`)
* Export source `11.0.0.101`
* Collector `172.16.18.72`
* Export protocol IPFIX (inline jflow)
* Export port `9000`

**Collector**

* NorthStar Analytics `netflowd`
* Listening on UDP/9000
* Publishing flows to Elasticsearch and AMQP

---

## 2. Junos Configuration

### 2.1 Enable inline jflow on the FPC

```bash
set chassis network-services enhanced-ip
set chassis fpc 0 sampling-instance inline_jflow
```

### 2.2 Configure the sampling instance & flow-server

```bash
set forwarding-options sampling instance inline_jflow input rate 1000
set forwarding-options sampling instance inline_jflow family inet output flow-server 172.16.18.72 port 9000
set forwarding-options sampling instance inline_jflow family inet output flow-server 172.16.18.72 version-ipfix template vgn_ipfix_v4
set forwarding-options sampling instance inline_jflow family inet output inline-jflow source-address 11.0.0.101
```

Notes:

* `input rate 1000` = 1 in every 1000 packets is sampled.
* `source-address` must be reachable by the collector.

### 2.3 Define the IPFIX template

```bash
set services flow-monitoring version-ipfix template vgn_ipfix_v4 flow-active-timeout 60
set services flow-monitoring version-ipfix template vgn_ipfix_v4 flow-inactive-timeout 15
set services flow-monitoring version-ipfix template vgn_ipfix_v4 template-refresh-rate seconds 30
set services flow-monitoring version-ipfix template vgn_ipfix_v4 ipv4-template
```

This template controls:

* How long active flows are exported (`flow-active-timeout`)
* How quickly idle flows are aged out (`flow-inactive-timeout`)
* How often the template itself is refreshed

### 2.4 Enable sampling on the ingress interfaces

```bash
set interfaces ge-0/0/5 unit 0 family inet sampling input
set interfaces ge-0/0/6 unit 0 family inet sampling input
set interfaces ge-0/0/7 unit 0 family inet sampling input
set interfaces ge-0/0/8 unit 0 family inet sampling input
set interfaces ge-0/0/9 unit 0 family inet filter input filter-11.255
set interfaces ge-0/0/9 unit 0 family inet sampling input
```

Any traffic ingressing these interfaces will now be subject to the sampling instance `inline_jflow`.

---

## 3. Verifying on the Router

You can use `show services accounting flow inline-jflow` to confirm that flows are being created and exported by the PFE.

Example output:

```bash
root@vmx101> show services accounting flow inline-jflow fpc-slot 0
  Flow information
    FPC Slot: 0
    Flow Packets: 1942, Flow Bytes: 2566738
    Active Flows: 2, Total Flows: 123
    Flows Exported: 171, Flow Packets Exported: 169
    Flows Inactive Timed Out: 25, Flows Active Timed Out: 96
    Total Flow Insert Count: 27

    IPv4 Flows:
    IPv4 Flow Packets: 1942, IPv4 Flow Bytes: 2566738
    IPv4 Active Flows: 2, IPv4 Total Flows: 123
    IPv4 Flows Exported: 171, IPv4 Flow Packets exported: 169
    IPv4 Flows Inactive Timed Out: 25, IPv4 Flows Active Timed Out: 96
    IPv4 Flow Insert Count: 27
```

Key fields to watch:

* **Flow Packets / Flow Bytes** – volume of sampled traffic
* **Flows Exported** – should be increasing over time
* **Flows Active/Inactive Timed Out** – confirms aging is happening as per your timers

If `Flows Exported` stays at 0, either sampling is not active, or export to the collector is failing.

---

## 4. Collector Configuration (NorthStar `netflowd`)

On the analytics / collector side, the relevant part of `northstar.cfg` for `netflowd` looks like this:

```ini
# netflowd settings
netflow_collector_address=
netflow_port=9000
netflow_ssl=0
netflow_log_level=debug
netflow_sampling_interval=1
netflow_publish_interval=60
netflow_workers=1
netflow_ageout=0
netflow_aggregate_by_prefix=disabled
netflow_stats_interval=-1
```

* `netflow_port=9000` must match the port configured under `forwarding-options sampling ... flow-server` on the router.
* Leaving `netflow_collector_address` empty binds to all local addresses.

### 4.1 netflowd logs (`/opt/northstar/logs/netflow.msg`)

Once the router is exporting, you should see messages like:

```text
2021-06-22 14:05:24,220 netflowd.NetflowExporter sent 1 documents to elasticsearch, 0 errors
2021-06-22 14:05:24,222 netflowd.NetflowExporter 1 ns_demand sent to controller.wan.stats
2021-06-22 14:06:25,211 netflowd.NetflowExporter sent 1 documents to elasticsearch, 0 errors
2021-06-22 14:06:25,213 netflowd.NetflowExporter 1 ns_demand sent to controller.wan.stats
2021-06-22 14:07:26,209 netflowd.NetflowExporter sent 1 documents to elasticsearch, 0 errors
2021-06-22 14:07:26,209 netflowd.NetflowExporter 1 ns_demand sent to controller.wan.stats
```

This confirms that:

* IPFIX records are being received
* They’re being translated into documents for Elasticsearch
* They’re being published on AMQP (`controller.wan.stats`)

---

## 5. Packet Capture on the Collector

To double-check that the IPFIX/NetFlow packets are hitting the collector interface, you can run `tcpdump`:

```bash
[root@ana2-sitea-q-pod21 ~]# tcpdump -i ens3f2 port 9000 and host 11.0.0.101 -vv
tcpdump: listening on ens3f2, link-type EN10MB (Ethernet), capture size 262144 bytes
13:43:45.921306 IP (tos 0x0, ttl 250, id 32772, offset 0, flags [none], proto UDP (17), length 138)
    11.0.0.101.50121 > ana2-sitea-q-pod21.cslistener: [no cksum] UDP, length 110
13:43:59.212970 IP (tos 0x0, ttl 250, id 263, offset 0, flags [none], proto UDP (17), length 168)
    11.0.0.101.50121 > ana2-sitea-q-pod21.cslistener: [no cksum] UDP, length 140
13:44:29.227260 IP (tos 0x0, ttl 250, id 265, offset 0, flags [none], proto UDP (17), length 168)
    11.0.0.101.50121 > ana2-sitea-q-pod21.cslistener: [no cksum] UDP, length 140
13:44:30.926935 IP (tos 0x0, ttl 250, id 2423, offset 0, flags [none], proto UDP (17), length 138)
    11.0.0.101.50121 > ana2-sitea-q-pod21.cslistener: [no cksum] UDP, length 110
```

If you see periodic UDP packets from the router’s export source IP to port 9000, the network path is good and the focus can move to parsing/collector issues if needed.

---

## 6. Wrap-Up

In summary, enabling NetFlow/IPFIX export on Junos with inline jflow boils down to:

1. Enabling `enhanced-ip` and binding a `sampling-instance` to the FPC
2. Configuring a `forwarding-options sampling` instance pointing at your collector
3. Defining an IPFIX template under `services flow-monitoring`
4. Enabling `sampling input` on the relevant interfaces
5. Confirming flow export on both the router (`show services accounting flow`) and the collector (`netflow.msg` + `tcpdump`)

From here you can:

* Tune sampling rates and timeouts
* Extend templates for IPv6 and additional fields
* Build dashboards on top of the data in Elasticsearch / your TSDB

---

## 7. Troubleshooting

### 7.1 Symptom: `Flows Exported` stays at 0 on the router

Things to check:

* **Sampling enabled on the right interfaces**

  ```bash
  show configuration interfaces | display set | match "sampling input"
  ```

  Make sure traffic is actually *ingressing* those interfaces and using `family inet`.

* **Sampling instance attached to the FPC**

  ```bash
  show configuration chassis | display set | match sampling-instance
  show configuration forwarding-options sampling instance inline_jflow
  ```

  The `sampling-instance inline_jflow` must be configured under both `chassis fpc` and `forwarding-options sampling`.

* **Correct family / template**

  Ensure you have `family inet` under the sampling instance and an IPv4 IPFIX template (`ipv4-template`) defined.

* **Platform / feature support**

  Inline jflow is PFE-based. On some platforms/linecards you may need specific licenses or minimum Junos versions – worth checking release notes if it simply refuses to export.

---

### 7.2 Symptom: Router shows `Flows Exported`, but collector sees nothing

On the router:

* Confirm the export configuration:

  ```bash
  show configuration forwarding-options sampling instance inline_jflow family inet output
  ```

  Validate:

  * `flow-server` IP matches the collector
  * `port` matches `netflow_port` (9000)
  * `source-address` is correct and routable

On the network / collector host:

* **Ping / routing check**

  From the router, confirm reachability to the collector:

  ```bash
  ping 172.16.18.72 routing-instance <if-applicable>
  ```

* **Firewalls / ACLs**

  Make sure UDP/9000 is allowed end-to-end (no filter / firewall dropping it).

* **tcpdump**

  Run `tcpdump` on the collector interface. If you don’t see packets, the issue is network / routing / firewall. If you do see packets, move on to `netflowd` checks.

---

### 7.3 Symptom: Packets visible in tcpdump, but `netflowd` not processing flows

If `tcpdump` shows traffic, check:

* **`northstar.cfg` matches the Junos config**

  * `netflow_port=9000`
  * `netflow_collector_address` is empty (bind all) or set to the correct local IP

* **Restart `netflowd` after config changes**

  Any change in `northstar.cfg` typically requires restarting the `netflowd` process for it to pick up new settings.

* **Log level and messages**

  With `netflow_log_level=debug`, check `/opt/northstar/logs/netflow.msg` for clues, e.g.:

  * “Dropping flows with an invalid egress interface”
  * Template / version mismatch errors
  * Database connection issues

  If you see something like *“Dropping flows with an invalid egress interface”*, it usually means:

  * The device/IF mapping tables are not aligned with what the router is exporting
  * The interface indices in the flow records don’t match known interfaces in the analytics DB

  In that case, refresh the inventory or check that the router is properly discovered in NorthStar so the interface tables are populated.

---

