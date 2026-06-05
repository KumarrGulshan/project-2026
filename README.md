# lfw – Linux Firewall

![Project](https://img.shields.io/badge/lfw-purple.svg)
![Language](https://img.shields.io/badge/C11-blue.svg)

`lfw` is a high-performance stateful Linux firewall daemon that intercepts and filters packets in-kernel using eBPF at the Traffic Control (TC) subsystem, evaluating them against a human-readable ruleset.


## 1. Features

* **eBPF/TC-based filtering**: Intercepts packets in-kernel at the Traffic Control (TC) ingress/egress hooks and issues high-performance ACCEPT or DROP/SHOT verdicts.
* **Stateful connection tracking**: Tracks active 5-tuple connections (Source IP, Destination IP, Source Port, Destination Port, Protocol) for both IPv4 and IPv6, with a background thread that periodically purges expired connections.
* **Subnet/CIDR Matching**: Supports bitwise subnet masking for both IPv4 and IPv6 rule definitions (e.g. `/24`, `/64`, `/32`, or `any`).
* **Port Range Support**: Allows matching destination ports by ranges (e.g. `67-68` or `546-547`) or single ports.
* **Thread-Safe Architecture**: Full concurrency protection utilizing reader-writer locks (`pthread_rwlock_t`) for rules evaluation/reload and mutexes (`pthread_mutex_t`) for connection tracking.
* **On-the-fly Config Reload (SIGHUP)**: Dynamic reload of rulesets without terminating the daemon or dropping active connection tracking states.
* **Operational Metrics (SIGUSR1)**: Real-time statistics dump of rule hits, throughput bytes, and connection counts (both IPv4 and IPv6) directly to syslog.
* **Production Logging**: Integration with `syslog` for structured, JSON-based telemetry.
* **Dual-Stack Support**: Full stateful filtering support for IPv4 and IPv6 TCP, UDP, and ICMP/ICMPv6.


## 2. Requirements

To build and run `lfw`, you need the following:

* **Linux** kernel supporting eBPF and Traffic Control (TC) clsact.
* **GCC** with C11 support.
* **Libraries**: `libbpf` and `libpcap` (for the test tool).
* **Compilers**: `clang` and `llvm` (to compile the eBPF kernel program).

On Debian/Ubuntu:

```bash
sudo apt install build-essential clang llvm libbpf-dev libpcap-dev
```


## 3. Build & Installation

Clone the repo & navigate to the `lfw` repo:

```bash
git clone https://github.com/saurabh-857/lfw.git
cd lfw
```

From the project root:

```bash
make
```

This compiles:
- The main firewall daemon: `build/lfw`
- The eBPF kernel program: `build/lfw_bpf.o`

To install the daemon, configuration rules, and systemd service globally on the system:

```bash
sudo make install
```

### Running Unit Tests

The codebase includes a suite of unit tests verifying raw packet parsing, CIDR subnet matching, connection tracking (including thread-safety and concurrency), and stateful behavior:

```bash
make test
```

### Running Offline PCAP Tests

You can run the rule engine offline against a pcap or pcapng packet capture file to verify verdicts without attaching to live interfaces:

```bash
# Build the pcap test utility
make pcap-test

# Run the tester with the sample pcapng file in the repo root
./build/lfw_pcap_test wireshark_packet_capture.pcapng lfw.rules
```

To clean build artifacts:

```bash
make clean
```


## 4. Configuration

By default, `lfw` reads rules from:

- `/etc/lfw/lfw.rules`

You can also pass a custom rules file path as the second CLI argument after the interface:

```bash
sudo build/lfw <interface> /path/to/custom.rules
```

### 4.1 Syntax

One rule per line:

```text
ACTION [PROTO] [PORT] [from SRC] [to DST]
```

- **ACTION**: `allow` | `deny` (or `drop`)
- **PROTO**: `any` | `tcp` | `udp` | `icmp` (optional, default: any)
- **PORT**: single port (e.g. `22`), port range (e.g. `67-68`), or `PORT/PROTO` (e.g. `53/udp`) (optional; matches destination port/range)
- **SRC/DST**: `any`, IPv4 address (e.g. `192.168.1.10`), IPv6 address (e.g. `2001:db8::1`), IPv4 CIDR (e.g. `192.168.1.0/24`), or IPv6 CIDR (e.g. `2001:db8::/32`)

Lines starting with `#` or empty lines are ignored.

### 4.2 Examples

```text
# Deny by default
default deny

# Allow loopback interface traffic
allow tcp from 127.0.0.1 to 127.0.0.1
allow tcp from ::1 to ::1

# Allow DHCPv4 and DHCPv6 client ports
allow udp 67-68
allow udp 546-547

# Allow HTTPS from anywhere
allow tcp 443

# Allow DNS queries to a specific resolver
allow udp 53 to 8.8.8.8

# Allow HTTP from a local IPv6 subnet
allow tcp 80 from 2001:db8::/32

# Allow ICMP (Ping)
allow icmp
```

Place your rules into `/etc/lfw/lfw.rules` (or another file you pass on the command line).


## 5. Running the firewall

### 5.1 Prepare the rules file

Create the directory and copy your rules:

```bash
sudo mkdir -p /etc/lfw
sudo cp lfw.rules /etc/lfw/lfw.rules
```

Edit `/etc/lfw/lfw.rules` as needed (see examples above).

### 5.2 Start the daemon

Run `lfw` as root and specify the network interface to attach to:

```bash
cd /path/to/lfw
sudo build/lfw <interface>
```

For example, to run on the loopback (`lo`) interface or ethernet (`eth0`):

```bash
sudo build/lfw lo
```

If you want to use a custom rules file:

```bash
sudo build/lfw <interface> /path/to/custom.rules
```

### 5.3 Syslog logs and signals

Operational events and logs are sent to the system logger (`syslog`). You can view them using:

```bash
tail -f /var/log/syslog | grep lfw
# or using journalctl
journalctl -t lfw -f
```

The daemon supports operational control signals:

- **Reload Config**: Reload the rules configuration file dynamically without restarting:
  ```bash
  sudo kill -HUP $(pgrep lfw)
  ```
- **Dump Statistics**: Output active connections table size, rule hit counts, and byte counters:
  ```bash
  sudo kill -USR1 $(pgrep lfw)
  ```

## 5.4 Systemd Integration

For integration with the host system, you can use the systemd template service unit (installed globally via `sudo make install`) to manage the firewall on a specific network interface (e.g., `eth0`):

```bash
sudo systemctl enable lfw@eth0 --now
```

This starts the daemon and configures it to run automatically on boot. To check status:

```bash
sudo systemctl status lfw@eth0
```


## 6. Internal Architecture

* **eBPF Filter**: Intercepts packets directly in the kernel's TC ingress and egress pipelines, parsing packet headers (L3/L4) and matching them against active rules and connections for sub-microsecond filtering.
* **State/Conntrack Maps**: 
  - `conntrack_map`: A BPF Hash Map tracking active IPv4 connections.
  - `conntrack_map_v6`: A BPF Hash Map tracking active IPv6 connections.
  - Both track stateful TCP connections (SYN-SENT, SYN-RECV, ESTABLISHED, FIN-WAIT) and UDP flows. Out-of-state TCP packets (e.g., non-SYN packets arriving before connection establishment) are dropped.
* **Rules Map**: A BPF Array Map (`rules_details_map`) populated by the userspace daemon containing up to 256 compiled rules.
* **Config Map**: A BPF Array Map (`config_map`) storing runtime configuration parameters (e.g., default action and rule count).
* **Trie Maps**: LPM (Longest Prefix Match) Trie maps (`src_ip_trie`, `dst_ip_trie`, `src_ip6_trie`, `dst_ip6_trie`) populated by the userspace daemon for high-performance bitwise subnet/CIDR matching.
* **Background Housekeeper**: A userspace thread that periodically sweeps `conntrack_map` and `conntrack_map_v6` in the kernel and deletes expired connections using state-specific timeouts (e.g. shorter timeouts for unfinished TCP handshakes).
* **Config Loader**: Parses text-based rules files in userspace and synchronizes compiled rule structures and policies to the BPF maps.


## 7. Quick start (TL;DR)

```bash
# 1) Install dependencies (Debian/Ubuntu)
sudo apt install -y build-essential clang llvm libbpf-dev libpcap-dev

# 2) Clone the repo & navigate to the folder
git clone https://github.com/saurabh-857/lfw.git
cd lfw

# 3) Build & Install system-wide
make
sudo make install

# 4) Enable and start the firewall on your network interface (e.g. wlan0 or eth0)
sudo systemctl enable lfw@wlan0 --now
```

After this, incoming and outgoing packets on the specified interface will be filtered according to `/etc/lfw/lfw.rules`.

