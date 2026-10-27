# Linux Firewall Lab — Containerlab

A network security lab built with Containerlab, Alpine Linux, and iptables. This project simulates a segmented network environment with a LAN, a DMZ web server, external WAN clients, and a host representing a potential attacker.

## Overview

The goal of this lab is to configure and test a Linux-based firewall that controls traffic between different network segments using iptables.

The firewall uses a default-deny policy for incoming and forwarded traffic, allowing only explicitly permitted connections.

## Network Topology

The lab consists of five containers:

| Node | Role | IP Address |
|---|---|---|
| `router` | Linux firewall and router | Multiple interfaces |
| `lan-client` | Internal LAN client | `10.1.1.10/24` |
| `dmz-server` | Nginx web server | `192.168.10.10/24` |
| `wan-client` | External WAN client | `172.16.1.100/24` |
| `wan-attacker` | Simulated attacker / blacklisted host | `172.16.1.200/24` |

### Network Segments

- **LAN:** `10.1.1.0/24`
- **DMZ:** `192.168.10.0/24`
- **WAN client segment:** `172.16.1.0/24`
- **WAN attacker segment:** `172.16.1.0/24` (separate link in the current topology)

### Topology Diagram

```text
                    Linux Firewall
                       (Alpine)
                          |
          +---------------+---------------+
          |               |               |
         LAN              DMZ         WAN Connections
          |               |             /       \
    LAN Client       Nginx Server   WAN Client  Attacker
    10.1.1.10       192.168.10.10  172.16.1.100 172.16.1.200
```

## Technologies Used

- [Containerlab](https://containerlab.dev/) — Network lab deployment and management
- Docker — Container runtime
- Alpine Linux — Lightweight Linux environment
- iptables — Firewall rules and packet filtering
- Nginx — DMZ web server
- curl and netcat — Connectivity testing

## Repository Structure

```text
.
├── physical-firewall.clab.yml
├── config/
│   ├── iptables.rules
│   └── index.html
└── README.md
```

- `physical-firewall.clab.yml`: Defines the lab nodes, network links, IP configuration, and startup commands(Including Firewall).
- `config/iptables.rules`: Contains the firewall filtering rules (But on this lab all the firewall rules are set manually on the iptables-firewall-lab.clab.yml).
- `config/index.html`: Custom HTML page served by the Nginx container.
- `README.md`: Project documentation.

## Firewall Policy

The firewall uses a **default-deny policy for incoming and forwarded traffic**, allowing only explicitly permitted connections.

| Chain | Default Policy | Purpose |
|---|---|---|
| `INPUT` | `DROP` | Blocks incoming traffic destined for the firewall unless explicitly allowed |
| `FORWARD` | `DROP` | Blocks traffic routed between networks unless explicitly allowed |
| `OUTPUT` | `ACCEPT` | Allows traffic originating from the firewall |

### Allowed Traffic

- **Established and related connections:** Existing and related connections are allowed through the `INPUT` and `FORWARD` chains.
- **Invalid packets:** Packets classified as `INVALID` are dropped.
- **LAN to DMZ:** LAN clients (`10.1.1.0/24`) can initiate TCP connections to ports 80 (HTTP) and 443 (HTTPS) on the DMZ subnet (`192.168.10.0/24`).
- **LAN to WAN:** LAN clients (`10.1.1.0/24`) can initiate TCP connections to ports 80 (HTTP) and 443 (HTTPS) on the WAN subnet (`172.16.1.0/24`).
- **WAN to DMZ:** WAN clients (`172.16.1.0/24`) can initiate TCP connections to ports 80 (HTTP) and 443 (HTTPS) on the DMZ web server (`192.168.10.10`).

### Blacklist and Threat Mitigation

- Traffic forwarded from the blacklisted IP address `172.16.1.200` is denied by a dedicated firewall rule.
- The firewall drops invalid packets to reduce unwanted or malformed traffic.
- All other forwarded traffic is denied by the default `FORWARD` policy unless it matches an explicitly permitted rule.


## Prerequisites

Before deploying the lab, ensure that the following tools are installed:

- Docker
- Containerlab
- A Linux environment with the necessary privileges to manage containers and network interfaces

## Deployment

Clone the repository:

```bash
git clone <https://github.com/AvJar1138/networking-lab-physical-firewalls>
cd <your-repository-directory>
```

Deploy the topology:

```bash
sudo containerlab deploy -t physical-firewall.clab.yml
```

## Verification & Expected Test Results

The following tests verify the firewall's access control policies between the WAN, LAN, and DMZ network segments.

### Test 1: Web Access from WAN Client to DMZ

**Objective:** Verify that the WAN client can access the DMZ web server over HTTP.

**Command:**

```bash
docker exec -it clab-iptables-firewall-lab-wan-client \
  curl -I http://192.168.10.10
```

**Expected Result:**

```text
HTTP/1.1 200 OK
```

**Status:** Pass if the server returns HTTP 200.

### Test 2: SSH Access from WAN Client to DMZ (Blocked)

**Objective:** Verify that SSH access to the DMZ server is blocked.

**Command:**

```bash
docker exec -it clab-iptables-firewall-lab-wan-client \
  nc -zv -w 3 192.168.10.10 22
```

**Expected Result:**

```text
Connection timed out or connection failed
```

**Status:** Pass if the SSH connection cannot be established.

### Test 3: Blacklisted Attacker Access to DMZ (Blocked)

**Objective:** Verify that the blacklisted attacker host cannot access the DMZ web server.

**Command:**

```bash
docker exec -it clab-iptables-firewall-lab-wan-attacker \
  curl --connect-timeout 3 http://192.168.10.10
```

**Expected Result:**

```text
Connection timed out
```

**Status:** Pass if the attacker cannot establish an HTTP connection.

### Test 4: Direct Unsolicited WAN Access to Internal LAN (Blocked)

**Objective:** Verify that the WAN client cannot reach the internal LAN workstation through unsolicited ICMP traffic.

**Command:**

```bash
docker exec -it clab-iptables-firewall-lab-wan-client \
  ping -c 3 10.1.1.10
```

**Expected Result:**

```text
100% packet loss
```

**Status:** Pass if no ICMP echo replies are received.

### Test 5: LAN Workstation Access to DMZ

**Objective:** Verify that the internal LAN workstation can access the DMZ web server over HTTP.

**Command:**

```bash
docker exec -it clab-iptables-firewall-lab-lan-client \
  curl -I http://192.168.10.10
```

**Expected Result:**

```text
HTTP/1.1 200 OK
```

**Status:** Pass if the server returns HTTP 200.

### Test Summary

| Test | Description | Expected Outcome |
|---|---|---|
| 1 | WAN to DMZ HTTP | HTTP 200 OK |
| 2 | WAN to DMZ SSH | Blocked |
| 3 | Blacklisted attacker to DMZ HTTP | Blocked |
| 4 | WAN to internal LAN ICMP | No replies |
| 5 | LAN to DMZ HTTP | HTTP 200 OK |

**Note:** These are expected results, not confirmed test results. Actual outcomes depend on the firewall rules, routing configuration, and services running in the containers. Record each test as passed or failed after running it.

Packet and byte counters can help verify which rules are matching traffic.

> Note: Test results depend on correct routing, interface configuration, and successful firewall initialization.

## Cleanup

Remove the lab containers and network resources:

```bash
sudo containerlab destroy -t iptables-firewall-lab.clab.yml
```

## Learning Objectives

- Understand Linux packet filtering with iptables.
- Implement a default-deny firewall policy.
- Control traffic between LAN, DMZ, and WAN segments.
- Deploy network infrastructure using Containerlab.
- Test connectivity and troubleshoot firewall rules.
- Understand the importance of network segmentation and access control.


## Disclaimer

This project is intended for educational purposes in an isolated lab environment. The attacker node is a simulated test host used to evaluate firewall behavior.
