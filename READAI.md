# APMONITOR.PY v1.2.10 — ON-PREMISES NETWORK AVAILABILITY & PORT MONITORING WITH GUARANTEED ALERT DELIVERY

Hey — welcome to APMonitor.py. You're a software engineering CS graduate working with a senior computer scientist on a mature, production Python monitoring tool. You've done good work here — let me bring you up to speed on exactly where we are and what matters.

---

## Purpose

APMonitor monitors on-premises network resources (ping/HTTP/HTTPS/QUIC/TCP/UDP/SNMP/ports) with guaranteed alert delivery via external heartbeat integration (Site24x7 / Healthchecks.io). It tracks interface status, bandwidth, MAC address changes, and system metrics — storing history in RRD for MRTG graphing, notifying via email and webhooks.

---

## What APMonitor Does

- Loads YAML/JSON configuration defining site, monitors, email/webhook notifications, and timing parameters
- Validates configuration comprehensively before any monitoring begins — fail fast on bad config
- Checks resource availability: ICMP ping, HTTP/HTTPS GET/POST, QUIC/HTTP3 GET/POST, TCP connection/banner, UDP send/receive
- SNMP monitors auto-discover device interfaces via IF-MIB::ifDescr walk, poll per-interface byte/packet counters, aggregate totals, TCP retransmits, CPU/memory
- SNMP vendor detection via sysObjectID (Cisco/HP/Juniper/Ubiquiti) drives CPU/memory OID selection with HOST-RESOURCES-MIB universal fallback
- `ports` monitors poll IF-MIB for per-interface oper/admin status changes via SNMPv2c
- `ports` monitors poll Q-BRIDGE-MIB (dot1qTpFdbTable) for learned MAC addresses per port — fires `PORT MAC CHANGE` alerts on appeared/disappeared MACs
- `ports` monitors fire `PORT CHANGE` alerts on oper/admin status changes (appeared/disappeared interfaces, up/down transitions)
- `ports` baseline established silently on first poll; all subsequent polls diff against committed baseline
- Enforces per-monitor check intervals with site-level defaults and immediate check on config change (SHA-256 checksum detection)
- Tracks persistent state in JSON statefile with atomic rotation (.new → current, .old backup)
- Sends email notifications via SMTP with per-recipient control flags (outages/recoveries/reminders)
- Sends webhook notifications (GET/POST with URL/HTML/JSON/CSVQUOTED encoding)
- Enforces notification throttling with escalating delays via quadratic Bezier curve
- Pings heartbeat URLs when resources are up, with configurable intervals
- Validates SSL certificates via SHA-256 fingerprints and expiration checks (HTTP/QUIC only)
- Stores SNMP metrics in single RRD per device; availability monitors get separate RRD per resource
- Generates MRTG configs (4 targets per SNMP monitor: bandwidth/packets/retransmits/system) and unified index.html with responsive 4-column grid
- Uses PID lockfiles per config file to prevent duplicate instances; supports concurrent monitoring of multiple sites
- Multithreaded with thread-local prefix storage for clean, filterable log output
- Explicit stdout flushing at synchronization points for correct pipe-captured output ordering

---

## Key Architecture

### `ports` Monitor — The Most Recent Major Feature (v1.2.9/v1.2.10)

`ports` is a distinct monitor type using SNMP transport but with completely different semantics from `snmp` monitors. It does not feed RRD/MRTG.

**State model:** A single `ports_state` key per monitor in the statefile holds the committed baseline: `{if_index: {name, oper, admin, macs}}`. This advances to current state on every successful poll. There is no separate pending/silence window dict — the throttle is handled by the existing `prev_last_notified` / `prev_notified_count` / `prev_last_alarm_started` state in `check_and_heartbeat_r()`.

**Change detection:** Two orthogonal checks per interface per poll:
1. Status diff: compare `(name, oper, admin)` tuple — `macs` deliberately excluded. Fires `##### PORT CHANGE: #####`.
2. MAC diff: set arithmetic on `curr_iface['macs']` vs `prev_iface['macs']`. Only when interface exists both sides. Fires `##### PORT MAC CHANGE: #####`.

**MAC resolution chain (Q-BRIDGE-MIB, RFC 2674):**
- Walk `dot1qTpFdbPort` (1.3.6.1.2.1.17.7.1.2.2.1.2) — OID tail is `<vlan_id>.<6 MAC octets>`, value is bridge port number (= ifIndex directly on this switch family)
- Walk `dot1qTpFdbStatus` (1.3.6.1.2.1.17.7.1.2.2.1.3) — filter to status=3 (learned) only
- MAC decoded from OID tail: 7-octet tail, strip first octet (vlan_id), convert remaining 6 decimal octets to `AA:BB:CC:DD:EE:FF`
- The `dot1dTpFdbTable` (classic BRIDGE-MIB) was tried first but returns 0 entries on VLAN-aware switches — Q-BRIDGE-MIB is the correct table for these devices
- `bridge_port_to_ifindex` join (via `dot1dBasePortIfIndex`) is NOT needed — Q-BRIDGE-MIB value is already ifIndex on the target switch hardware
- MAC walk failure is non-fatal: sets `macs: []` for all interfaces, monitoring continues

**Numeric interface sort:** `sorted(all_indices, key=lambda x: int(x))` — interface indices are strings from SNMP OID tails; lexicographic sort gives wrong order (e.g. "10" < "2").

### SNMP Vendor Detection & Metrics (v1.2.5–v1.2.7)

Query sysObjectID first → match known enterprise OID prefixes (Cisco/HP/Juniper/Ubiquiti) → try vendor-specific CPU/memory OIDs → fall back to HOST-RESOURCES-MIB if any step fails. CPU/memory failures store 'U' in RRD rather than failing the check. Aggregate bandwidth/packets computed in Python before RRD storage.

### Notification Throttling (all monitor types)

Quadratic Bezier curve escalation: intervals start short, grow to `notify_every_n_secs` over `after_every_n_notifications` notifications, then plateau. `ports` monitors share this machinery via `prev_last_notified` / `prev_notified_count`. Note: for `ports`, both `PORT CHANGE` and `PORT MAC CHANGE` events advance the same throttle counters — they share one notification window per monitor (not per interface).

### Unified URL Resource Architecture

HTTP/QUIC/TCP/UDP share `check_url_resource()` entry point returning `(error_msg, status_code, headers, response_text)`. TCP/UDP return 200 by convention for success. Expect checking lives once in `check_url_resource()`. SNMP and ports bypass this entirely — different return signature, no expect checking.

### Thread Safety & Output Ordering

`STATE` dict protected by `STATE_LOCK`. `thread_local.prefix` stores `[T#XXXX Site/Resource]` per thread, set once at start of `check_and_heartbeat_r()`. Explicit `sys.stdout.flush()` inside `update_state()` ensures thread output visible atomically with state mutations — critical for pipe-captured output (systemd journal).

---

## Important Modules & Code Sections

### `check_ports_resource(resource)`
Polls IF-MIB (ifDescr/ifOperStatus/ifAdminStatus) and Q-BRIDGE-MIB (dot1qTpFdbPort/dot1qTpFdbStatus). Returns `(error_msg, current_ports_state)` where `current_ports_state` is `{if_index: {name, oper, admin, macs}}`. MAC walk is non-fatal — exception → `macs: []` for all interfaces. Interface state sorted by numeric ifIndex. No RRD involvement.

### `check_and_heartbeat_r(resource, site_config)`
Main per-resource orchestration. For `ports` type with `is_up=True`: diffs current vs prev `ports_state`, fires `PORT CHANGE` on status tuple mismatch and `PORT MAC CHANGE` on MAC set difference, advances throttle state. Saves updated `ports_state` to statefile. For all other types: standard up/down/recovery logic. The `ports` branch and the normal `else` branch both write `last_notified`, `notified_count`, `down_count`, `last_alarm_started` to ensure `new_state` dict is always fully defined. Status comparison uses explicit `(name, oper, admin)` tuple to exclude `macs` from triggering `PORT CHANGE`.

### `check_snmp_resource(resource)`
Vendor detection → per-interface byte/packet polls → aggregate calculation → TCP retransmits → CPU/memory → RRD update. Returns `Optional[str]` (None = success). All metric failures graceful — store 'U' not error.

### `create_snmp_rrd(path, interval, interfaces)` / `update_snmp_rrd(...)`
Single RRD per SNMP device. DS schema: `if{index}_in/out` (COUNTER) per interface + `total_bits_in/out`, `total_pkts_in/out` (COUNTER) + `tcp_retrans` (COUNTER) + `cpu_load`, `memory_pct` (GAUGE). Stable sort on `sorted(interfaces.keys())` in both create and update — DS order must match exactly. Template parameter always used in `rrdtool.update()`.

### `get_rrd_path(monitor_name, metric_type='availability')`
Resolves to `{statefile_dir}/{statefile_stem}.rrd/{safe_name}-{metric_type}.rrd`. Metric types: `'availability'` (ping/http/quic/tcp/udp) or `'snmp'`. `ports` monitors: no RRD path, not called.

### `generate_mrtg_config(...)` / `generate_mrtg_index(...)`
4 targets per SNMP monitor (bandwidth/packets/retransmits/system). Target deduplication in index HTML by suffix detection (`-bandwidth`, `-packets`, `-retransmits`, `-system`) → base name → dict (ordered, auto-deduplicates) → one row per device with 4-column graph grid. Site name extracted from `config['site']['name']` in `main()`, passed as parameter — never parsed from MRTG comments.

### `check_url_resource(resource)` / Protocol checkers
Unified tuple contract: `(error_msg, status_code, headers, response_text)`. TCP/UDP return 200 for success. Expect checking once in `check_url_resource()`. SNMP/ports not routed here.

### `print_and_exit_on_bad_config(config)`
Comprehensive validation before any monitoring. `ports` validation: `snmp://` scheme required, `expect`/`ssl_fingerprint`/`ignore_ssl_expiry`/`send`/`content_type`/`percentile` all forbidden, `community` optional, `heartbeat_url` allowed.

### `update_state(updates)` / `load_state()` / `save_state()`
Thread-safe: `STATE_LOCK` → in-memory update → write `.new` → `flush()`. Atomic rotation on exit. `ports_state` persisted per-monitor as committed baseline. No `ports_pending` key — it was designed but not implemented; throttling uses existing `last_notified`/`notified_count` fields.

### `main()`
PID lock → load config/state → thread pool (`executor.submit()`) → `future.result()` loop (blocks + propagates exceptions) → flush → save state → release lock. `site_name` extracted once from config, passed to `generate_mrtg_index()`.

---

## Technical Tactics

**Q-BRIDGE-MIB over BRIDGE-MIB:** `dot1dTpFdbTable` returns 0 entries on VLAN-aware switches because the FDB is partitioned per VLAN. `dot1qTpFdbTable` encodes VLAN in the OID tail and returns all learned MACs regardless of VLAN. Target switch hardware also maps bridge port directly to ifIndex, eliminating the `dot1dBasePortIfIndex` join step.

**7-octet OID tail for MAC decoding:** Q-BRIDGE-MIB OID tail is `<vlan_id>.<6 MAC octets>`. Split last 7 elements, discard element 0 (vlan_id), convert elements 1–6 from decimal to hex: `':'.join(f'{int(o):02X}' for o in mac_octets)`.

**Status tuple excludes macs:** `(name, oper, admin)` comparison for `PORT CHANGE` — never compare full iface dict — prevents a MAC change from spuriously triggering a status alert with misleading "was oper=up admin=up" messaging.

**Non-fatal MAC walk:** Wrap Q-BRIDGE-MIB walks in try/except. On any failure: log if VERBOSE, set `macs_by_ifindex = {}`, proceed. Interface status monitoring unaffected. Better to monitor ports without MACs than fail the entire check.

**Vendor detection via sysObjectID prefix match:** `startswith()` on known enterprise OID prefixes. Detection failure acceptable — log and fall back to HOST-RESOURCES-MIB. Never skip vendor detection for potential match.

**Aggregate metrics in Python:** Total bits/packets computed before RRD storage. Enables per-interface error handling (one failed interface doesn't kill the aggregate), flexible combinations, validation before storage. RRD COUNTER auto-calculates rates.

**Explicit flush at synchronization points:** `sys.stdout.flush()` inside `update_state()` (called by every thread), after banner, after startup messages. Systemd captures via pipes (fully buffered). Without flush, thread output accumulates and appears out-of-order in journal.

**Stable sort on interface indices:** `sorted(interfaces.keys(), key=lambda x: int(x))` everywhere — SNMP OID tail indices are strings, lexicographic sort is wrong. RRD DS order must match between create and update.

**SHA-256 config checksums:** JSON-serialize with `sort_keys=True`, compute SHA-256, store as `last_config_checksum`. Mismatch on load forces immediate check, bypassing `check_every_n_secs`.

**Bezier escalation for notifications:** `t = index / N`, delay = `(1-t)² × 0 + 2(1-t)t × base + t² × base`. After N notifications, plateau at `notify_every_n_secs`. Applies to all monitor types including `ports`.

**Thread-local prefix:** `threading.local()` eliminates prefix parameter threading. Set once in `check_and_heartbeat_r()`, retrieved via `getattr(thread_local, 'prefix', '')`. Main thread output stays unprefixed (banner, startup).

**COUNTER vs GAUGE selection:** COUNTER for cumulative metrics (bytes, packets, retransmits) — RRD calculates rate, handles 32/64-bit wraparound. GAUGE for instantaneous values (CPU %, memory %). Never mix.

---

## Engineering Principles for This Code

**Never delete code or diagnostic comments without explicit notification.** Comments may carry operational knowledge and hard-learned lessons (e.g., why Q-BRIDGE-MIB and not BRIDGE-MIB).

**Status comparison must exclude `macs`.** Always use explicit `(name, oper, admin)` tuple for `PORT CHANGE` detection. Full dict equality would conflate status and MAC changes.

**MAC walk is non-fatal, always.** Interface status monitoring is the primary function of `ports`. MAC changes are valuable but secondary. Never let Q-BRIDGE-MIB failure degrade port status monitoring.

**Q-BRIDGE-MIB is the correct MAC table for VLAN-aware switches.** `dot1dTpFdbTable` will return zero entries. If you're debugging MAC detection, verify with `snmpwalk -v2c -c <community> <host> 1.3.6.1.2.1.17.7.1.2.2` first.

**Numeric sort on interface indices everywhere.** String sort on SNMP OID tail indices is wrong. This applies in `check_ports_resource()`, `check_snmp_resource()`, and anywhere else interface indices are sorted.

**Ports throttle is per-monitor, not per-interface.** A MAC change and a status change on different interfaces both advance the same `prev_last_notified` / `prev_notified_count` counters. This is intentional — one notification window per monitor resource.

**`ports_state` is the only ports-related statefile key.** There is no `ports_pending`. The silence window semantics described in early design docs were simplified — throttling uses existing `last_notified`/`notified_count` machinery shared with all monitor types.

**Partial data over complete failure.** For SNMP, individual metric failures store 'U' rather than failing the check. For `ports`, MAC walk failure → `macs: []`, not check failure. Better to collect 9/10 metrics than nothing.

**Vendor detection first, HOST-RESOURCES-MIB fallback second.** Never assume HOST-RESOURCES-MIB on network hardware. Attempt vendor OIDs first; treat HOST-RESOURCES-MIB as safety net.

**SNMP monitors and `ports` monitors are orthogonal.** They share the `snmp://` transport scheme but have completely different check logic, return signatures, state models, and RRD involvement. Do not conflate them.

**All interfaces in a single RRD per SNMP device.** Atomic updates prevent timestamp skew. If interface list changes, stale DS stays in RRD (unused but harmless). Never recreate RRD on interface list change.

**Validate before execute, always.** Config validation in `print_and_exit_on_bad_config()` is comprehensive and fail-fast. Any new monitor type or field must have corresponding validation rules.

**Leave code cleaner than you found it.** This means preserving comments, not just not deleting code. Comments represent operational knowledge. Deleting them is a -100,000 util event.

---

Would you like to see the code?