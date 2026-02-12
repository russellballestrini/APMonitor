# APMONITOR.PY v1.2.7 - NETWORK MONITORING WITH VENDOR-AWARE SNMP SYSTEM METRICS

Hey! Welcome back to APMonitor.py. We've just expanded SNMP monitoring with aggregate bandwidth/packet metrics, CPU/memory polling with vendor detection, TCP retransmit tracking, MRTG 4-target visualization, and 4-column index.html layout. The system now intelligently adapts to Cisco, HP, Juniper, and Ubiquiti devices with HOST-RESOURCES-MIB fallback. Let me bring you up to speed.

## Purpose

APMonitor monitors network resources (ping/HTTP/HTTPS/QUIC/TCP/UDP/SNMP) on-premises with guaranteed alert delivery via external heartbeat integration. SNMP monitors collect per-interface metrics, aggregate bandwidth/packets, TCP retransmits, and system resources (CPU/memory), storing in RRD for MRTG graphing with vendor-aware OID selection.

## What APMonitor Does

- Loads YAML/JSON configuration defining site, monitors (ping/http/quic/tcp/udp/snmp), email/webhook notifications, timing parameters
- Validates configuration including SNMP-specific rules (scheme, community string, forbidden fields)
- Checks resource availability via ICMP ping, HTTP/HTTPS GET/POST, QUIC/HTTP3 GET/POST, TCP connection/banner, UDP send/receive, SNMP polling
- SNMP monitors auto-discover device interfaces via IF-MIB::ifDescr walk
- SNMP monitors detect vendor via sysObjectID (Cisco/HP/Juniper/Ubiquiti) for CPU/memory OID selection
- SNMP polls per-interface byte/packet counters (ifInOctets, ifOutOctets, ifHC*Pkts for unicast/multicast/broadcast)
- SNMP calculates aggregate totals: total_bits_in/out (sum of interface octets × 8), total_pkts_in/out (sum of all packet types)
- SNMP polls system resources: CPU load (vendor-specific or hrProcessorLoad average), memory utilization (vendor-specific or hrStorage calculation)
- SNMP falls back to HOST-RESOURCES-MIB when vendor OIDs fail or device unrecognized
- SNMP stores 'U' (unknown) in RRD for unavailable metrics, enabling partial data collection
- Supports three-way SNMP community string specification: monitor `community` field, URL userinfo, or default `public`
- Stores SNMP metrics in single RRD per device with dynamic data sources: per-interface counters, aggregates, TCP retransmits, CPU/memory
- RRD COUNTER type auto-calculates rates (bytes/sec, packets/sec, retransmits/sec) with wraparound handling
- RRD GAUGE type stores instantaneous values (CPU %, memory %)
- Generates MRTG configs with 4 targets per SNMP monitor: bandwidth, packets, retransmits, system resources
- Creates index.html with unified 4-column layout for Network Monitoring (SNMP) and Availability Monitoring sections
- Extracts site name from config for index.html title/header
- Deduplicates MRTG targets by detecting suffixes (-bandwidth, -packets, -retransmits, -system) for single-row-per-device layout
- Supports HTTP/QUIC POST with configurable MIME types (application/json, application/x-www-form-urlencoded, text/plain)
- Supports TCP/UDP protocol checks with text/hex/base64 encoding for send data and optional expect validation
- TCP monitors auto-receive data after connecting for banner protocol support (SSH/SMTP/FTP)
- UDP monitors require send parameter (connectionless protocol), support fire-and-forget or expect-based validation
- Enforces per-monitor check intervals with site-level defaults and configuration change detection via SHA-256 checksums
- Tracks persistent state in JSON statefile with atomic rotation
- Sends email notifications via SMTP with per-recipient control flags (outages/recoveries/reminders)
- Sends webhook notifications (GET/POST with URL/HTML/JSON/CSVQUOTED encoding)
- Enforces notification throttling with escalating delays via quadratic Bezier curve
- Pings heartbeat URLs when resources up with configurable intervals
- Validates SSL certificates via SHA-256 fingerprints and expiration checks (HTTP/QUIC only)
- Uses PID lockfiles to prevent duplicate instances per config file
- Runs multi-threaded with explicit stdout flushing for proper log ordering
- Prefixes all thread output with thread ID and site/resource context

## Key Architecture

**Vendor Detection Architecture**: SNMP polling begins with sysObjectID query (1.3.6.1.2.1.1.2.0) to identify device manufacturer. OID prefix matched against known vendors: Cisco (1.3.6.1.4.1.9.*), HP (1.3.6.1.4.1.11.*), Juniper (1.3.6.1.4.1.2636.*), Ubiquiti (1.3.6.1.4.1.41112.*). Vendor detection determines CPU/memory OID selection, enabling device-appropriate polling before fallback to HOST-RESOURCES-MIB.

**Vendor-Specific OID Mappings**: Each vendor exposes CPU/memory via proprietary MIBs. Cisco: cpmCPUTotal5secRev (5-sec avg) with cpmCPUTotal1minRev fallback, ciscoMemoryPoolUsed/Free for percentage calc. HP: hpSwitchCpuStat (direct %), hpLocalMemTotalBytes/FreeBytes for percentage calc. Juniper: jnxOperatingCPU/Buffer (direct %). Ubiquiti: ubntSystemCpuLoad (direct %), ubntSystemMemTotal/Free for percentage calc. OID selection isolated per vendor with try/except wrapping each attempt.

**HOST-RESOURCES-MIB Fallback**: When vendor detection fails, vendor OIDs unavailable, or device unrecognized, system falls back to standard HOST-RESOURCES-MIB. CPU: walks hrProcessorLoad, averages all cores. Memory: walks hrStorageDescr to find physical memory entry (description contains "physical memory", "ram", or exact "memory"), retrieves hrStorageAllocationUnits/Size/Used, calculates percentage. Fallback triggered by `if cpu_load is None:` and `if memory_pct is None:` checks after vendor attempts.

**Graceful Degradation for System Metrics**: CPU/memory poll failures result in 'U' (unknown) stored in RRD rather than check failure. Individual interface packet counter failures logged at VERBOSE > 1, don't affect aggregates. Partial data philosophy: better 9/10 metrics than complete failure. Summary output shows "cpu=unavailable" or "memory=unavailable" when metrics missing. Enables continued monitoring when system MIBs unsupported (e.g., older switches).

**Aggregate Metric Calculation**: Total bandwidth calculated as sum of all interface octets × 8 (bits), stored as total_bits_in/out COUNTER. Total packets calculated as sum of ifHCInUcastPkts + ifHCInMulticastPkts + ifHCInBroadcastPkts (and Out equivalents) across all interfaces, stored as total_pkts_in/out COUNTER. Aggregates computed in Python before RRD storage, enabling flexible combinations and per-interface error handling. RRD automatically calculates rates (bits/sec, packets/sec) from cumulative COUNTER values.

**SNMP RRD Schema Evolution**: Single RRD file per SNMP monitor now contains: per-interface DS (if{index}_in, if{index}_out COUNTER), aggregate DS (total_bits_in, total_bits_out, total_pkts_in, total_pkts_out COUNTER), TCP retransmit DS (tcp_retrans COUNTER), system resource DS (cpu_load, memory_pct GAUGE). Total DS count: 2N + 7 where N = interface count. All metrics updated atomically to prevent timestamp skew.

**MRTG 4-Target Architecture**: Each SNMP monitor generates 4 MRTG targets for comprehensive visualization. Target 1 (bandwidth): total_bits_in&total_bits_out, MaxBytes=10Gbps, YLegend="Bits per second". Target 2 (packets): total_pkts_in&total_pkts_out, MaxBytes=10M pps, YLegend="Packets per second". Target 3 (retransmits): tcp_retrans&tcp_retrans (same DS twice for single line), MaxBytes=100K, YLegend="Retransmits per second". Target 4 (system): cpu_load&memory_pct, MaxBytes=100, YLegend="Utilization %". Non-SNMP monitors unchanged: single target with response_time&is_up.

**Index HTML Unified 4-Column Layout**: Both Network Monitoring (SNMP) and Availability Monitoring sections use 4-column grid layout. Network section displays one row per SNMP device with columns: Bandwidth | Packets | Retransmits | System. Target deduplication extracts base name from suffixes (-bandwidth, -packets, -retransmits, -system), stores unique device names in dict. Site name extracted from config['site']['name'], passed to generate_mrtg_index() for title/header. Responsive breakpoints: >1400px=4col, 1400-768px=2col, <768px=1col.

**SNMP Monitor Integration with Check Flow**: SNMP integrates into check_resource() dispatch but bypasses check_url_resource() unified entry point. SNMP polls multiple metrics per check (N interfaces + aggregates + TCP + CPU + memory) vs single up/down status. Function check_snmp_resource() returns Optional[str] (error message or None) matching check_ping_resource() pattern, not the (error_msg, status_code, headers, response_text) tuple used by URL resources. Keeps SNMP logic isolated from HTTP/QUIC/TCP/UDP expect-checking machinery.

**RRD File Naming Convention**: Availability monitors use {monitor-name}-availability.rrd, SNMP monitors use {monitor-name}-snmp.rrd. Pattern implemented in get_rrd_path(monitor_name, metric_type='availability'). Enables both availability tracking (ping/http/quic/tcp/udp) and performance tracking (SNMP) for same logical resource if needed. All RRD files in {statefile_dir}/{statefile_stem}.rrd/ directory.

**SNMP Community String Precedence**: Three-way lookup with explicit precedence: (1) monitor-level community field, (2) URL userinfo from parsed address (snmp://community@host), (3) default "public". Code: community = resource.get('community') or parsed.username or 'public'. Short-circuit evaluation ensures first non-None/non-empty wins. Allows per-monitor overrides with sensible defaults.

**Interface Name Sanitization**: SNMP interface names (e.g., "GigabitEthernet0/1") contain characters invalid for RRD DS names. Sanitization: re.sub(r'[^\w]', '_', if_name)[:15]. Truncation to 15 chars leaves 4 chars for _in/_out suffix within 19-char RRD DS limit. Ensures deterministic DS names while preventing RRD creation failures.

**SNMP Stable Sorting for Deterministic DS Order**: RRD update requires template string with DS names in exact order matching RRD creation. Code uses sorted(interfaces.keys()) to ensure consistent ordering across create and update operations. Without stable sort, DS order nondeterministic, causing RRD update failures. Critical for multi-interface devices.

**Unified URL Resource Architecture**: HTTP/QUIC/TCP/UDP share common entry point check_url_resource(). This function handles expect checking universally by extracting (error_msg, status_code, headers, response_text) tuples from protocol-specific checkers. HTTP/QUIC return actual status codes (200, 404, etc.), TCP/UDP return 200 for success by convention. Expect logic lives once in check_url_resource(), not duplicated per protocol.

**Thread-Aware Prefixing**: prefix_logline(site_name, resource_name) generates [T#XXXX Site/Resource] prefix using threading.get_native_id() for unique identification. Prefix created once at start of check_and_heartbeat(), stored in thread_local storage, passed to all functions via thread_local.prefix. Enables clean log filtering and audit trails without parameter threading bloat.

**Output Buffering Solution**: Python stdout line-buffered for terminals but fully buffered for pipes. Systemd captures via pipes, causing interleaved output. Solution: explicit sys.stdout.flush() after welcome banner, startup messages, and critically inside update_state() after every state write. The update_state() flush ensures thread output visible atomically with state mutations.

**Stateless Execution with Flush Discipline**: Each invocation loads state, performs checks, flushes output at synchronization points, saves state atomically, exits. Output ordering guaranteed by strategic flush placement.

**Per-Config PID Locking**: Creates /tmp/apmonitor-{hash}.lock where hash = SHA256(config_path)[:16]. Prevents duplicate processes per config, allows concurrent monitoring of different sites. Stale lockfile detection via os.kill(pid, 0). Unix-only.

**Thread-Safe State Management**: Global STATE dict protected by STATE_LOCK. Updates written immediately to .new file inside lock, flushed immediately after lock release.

**Configuration Change Detection**: SHA-256 checksum of JSON-serialized monitor config triggers immediate checks when configuration changes, bypassing timing intervals.

**Interval-Based Scheduling**: Each monitor tracks last_checked, last_notified, last_successful_heartbeat timestamps. Decisions made by comparing elapsed time against configured intervals.

**Bezier Curve Notification Escalation**: Notification delays follow quadratic Bezier curve over first N notifications, then plateau. Formula: t = (1/N) * index, delay = (1-t)² * 0 + 2(1-t)t * base + t² * base.

## Important Modules & Communication

**SNMP Resource Checker** (check_snmp_resource):
- Validates snmp:// scheme, extracts hostname and port (default 161)
- Resolves community string via precedence: monitor field → URL userinfo → default "public"
- Creates easysnmp Session with SNMPv2c, community, timeout=MAX_TRY_SECS, retries=MAX_RETRIES-1
- Queries sysObjectID (1.3.6.1.2.1.1.2.0) for vendor detection (Cisco/HP/Juniper/Ubiquiti)
- Walks IF-MIB::ifDescr (1.3.6.1.2.1.2.2.1.2) to discover all interfaces
- For each interface: polls ifInOctets (1.3.6.1.2.1.2.2.1.10), ifOutOctets (1.3.6.1.2.1.2.2.1.16)
- For each interface: polls packet counters ifHCInUcastPkts/MulticastPkts/BroadcastPkts (1.3.6.1.2.1.31.1.1.1.7/8/9) and Out equivalents (1.3.6.1.2.1.31.1.1.1.11/12/13)
- Aggregates interface totals: total_octets_in/out (sum octets), total_pkts_in/out (sum all packet types)
- Converts octets to bits: total_bits_in/out = total_octets_in/out × 8
- Polls TCP-MIB::tcpRetransSegs (1.3.6.1.2.1.6.12.0) for global TCP retransmit counter
- Polls CPU via vendor-specific OID based on detected vendor, falls back to hrProcessorLoad (1.3.6.1.2.1.25.3.3.1.2) walk with averaging
- Polls memory via vendor-specific OID based on detected vendor, falls back to hrStorage table walk (1.3.6.1.2.1.25.2.3.1.x) with physical memory search and percentage calculation
- Individual metric failures set value to None (partial data better than no data)
- CPU/memory unavailable results in 'U' stored in RRD
- If RRD_ENABLED: creates SNMP RRD if missing via create_snmp_rrd(), updates via update_snmp_rrd()
- Returns None on success, error message string on failure
- Verbose output shows vendor detection, per-metric SNMP GET operations with OIDs and values, fallback attempts, final metric summary

**SNMP RRD Creation** (create_snmp_rrd):
- Accepts RRD path, check interval, interfaces dict (keys=indices, values={name, ...})
- Sanitizes interface names: alphanumeric+underscore, max 15 chars
- Creates per-interface DS: if{index}_in (COUNTER), if{index}_out (COUNTER)
- Creates aggregate DS: total_bits_in (COUNTER), total_bits_out (COUNTER), total_pkts_in (COUNTER), total_pkts_out (COUNTER)
- Creates TCP DS: tcp_retrans (COUNTER)
- Creates system DS: cpu_load (GAUGE), memory_pct (GAUGE)
- COUNTER type auto-calculates rates from cumulative values, handles wraparound
- GAUGE type stores instantaneous values as-is
- Heartbeat = 2 × check interval (allows one missed update)
- Uses existing create_rrd_rras() for MRTG-compatible retention policy (1d/2d/12d/50d/2y aggregations)
- Stable sort on interface indices ensures deterministic DS order matching update operations

**SNMP RRD Update** (update_snmp_rrd):
- Accepts RRD path, timestamp, interfaces dict, tcp_retrans value, aggregate metrics (total_bits_in/out, total_pkts_in/out), system metrics (cpu_load, memory_pct)
- Stable sort on interface indices (sorted(interfaces.keys())) ensures DS order matches creation
- Builds template string with DS names: per-interface (if{index}_in, if{index}_out), aggregates (total_bits_in, total_bits_out, total_pkts_in, total_pkts_out), TCP (tcp_retrans), system (cpu_load, memory_pct)
- Collects values in same order as template, formats floats with 2 decimal places
- Missing values represented as 'U' (unknown) in RRD update
- Uses rrdtool.update() with --template parameter for explicit DS ordering
- Critical: DS order must be identical between create and update, hence stable sort

**RRD Path Generation** (get_rrd_path):
- Signature: get_rrd_path(monitor_name: str, metric_type: str = 'availability') -> str
- Sanitizes monitor name to filesystem-safe characters via re.sub(r'[^\w\-.]', '_', monitor_name)
- Constructs path: {statefile_dir}/{statefile_stem}.rrd/{safe_name}-{metric_type}.rrd
- metric_type values: 'availability' (ping/http/quic/tcp/udp) or 'snmp' (SNMP monitors)
- Enables both availability and performance tracking for same logical resource if needed

**MRTG Config Generation** (generate_mrtg_config):
- For each SNMP monitor, generates 4 targets:
  - {name}-bandwidth: total_bits_in&total_bits_out, MaxBytes=10Gbps, Options=gauge,nopercent,growright,bits, YLegend="Bits per second"
  - {name}-packets: total_pkts_in&total_pkts_out, MaxBytes=10M pps, Options=gauge,nopercent,growright, YLegend="Packets per second"
  - {name}-retransmits: tcp_retrans&tcp_retrans (same DS twice), MaxBytes=100K, Options=gauge,nopercent,growright, YLegend="Retransmits per second"
  - {name}-system: cpu_load&memory_pct, MaxBytes=100, Options=gauge,nopercent,growright, YLegend="Utilization %"
- For non-SNMP monitors, generates single target: response_time&is_up with dual-axis graphing
- Each target includes Title, PageTop with HTML headers, MaxBytes for Y-axis scale, Options for graph rendering behavior
- Legends differentiate inbound/outbound or CPU/memory for dual-line graphs
- Config written to {output_dir}/mrtg-{name}.cfg for each SNMP monitor

**MRTG Index HTML Generation** (generate_mrtg_index):
- Accepts all_config_files list, index_path, site_name parameter (default "Availability Monitoring")
- Extracts site_name from config['site']['name'] in main(), passed during --generate-mrtg-config
- Parses MRTG config files to extract targets and monitor types
- Separates SNMP monitors (type='snmp') from non-SNMP monitors
- For SNMP monitors: detects target suffixes (-bandwidth, -packets, -retransmits, -system), extracts base name, stores in dict for deduplication
- Network Monitoring section: host label outside grid, 4-column grid with links to {name}-{suffix}-day.png for each suffix
- Availability Monitoring section: 4-column grid with links to {name}-day.png
- CSS responsive breakpoints: >1400px=4col, 1400-768px=2col, <768px=1col
- Uses site_name in <title> and <h1> tags
- Target deduplication ensures one row per SNMP device with 4 graphs
- Non-SNMP monitors displayed in separate section with consistent 4-column layout

**Protocol Function Signatures** (unified tuple return for URL resources):
All URL resource checkers return: (error_msg: Optional[str], status_code: Optional[int], headers: Any, response_text: Optional[str])
- check_http_url_resource(): Returns actual HTTP status code, headers dict, decoded response text
- check_quic_url_resource(): Returns HTTP/3 status code, headers dict, decoded response text
- check_tcp_url_resource(): Returns 200 for success (HTTP-like convention), empty headers dict {}, received text
- check_udp_url_resource(): Returns 200 for success, empty headers dict {}, received text if any
This uniform interface enables check_url_resource() to handle expect checking identically across all protocols.

**Check Resource Dispatch** (check_resource):
- Top-level dispatcher based on resource['type']
- Ping: calls check_ping_resource() directly (returns Optional[str])
- HTTP/QUIC/TCP/UDP: calls check_url_resource() (returns Optional[str] after tuple processing)
- SNMP: calls check_snmp_resource() directly (returns Optional[str])
- Updates state atomically after each check with thread-safe locking
- Returns error message or None consistently across all protocol types

**Configuration Validation** (print_and_exit_on_bad_config):
SNMP-specific validation rules:
- SNMP monitors must use snmp:// scheme
- Address must include hostname/IP (validated via urlparse)
- Hostname must be valid IPv4, IPv6, or DNS name (regex patterns)
- community field optional, must be non-empty string if specified
- expect, ssl_fingerprint, ignore_ssl_expiry not allowed for SNMP
- send, content_type not allowed for SNMP
- heartbeat_url, heartbeat_every_n_secs allowed (same as other types)
All validation happens before any monitoring begins—fail-fast on config errors.

**State Management** (load_state, save_state, update_state):
- load_state: Reads JSON at startup
- update_state: Thread-safe in-memory update + immediate write to .new + immediate flush
- save_state: Atomic rotation (current → .old, .new → current)
- Per-monitor state includes: is_up, last_checked, last_response_time_ms, down_count, last_alarm_started, last_notified, last_successful_heartbeat, notified_count, error_reason, last_config_checksum
- SNMP monitors track response time as total poll duration (includes all interface queries, aggregation, TCP, CPU/memory polls)

**Main Orchestration** (check_and_heartbeat, main):
- main: Acquires PID lock, loads config/state, extracts site_name from config['site']['name'], spawns thread pool, waits for completion with explicit result retrieval, flushes output, records execution time, passes site_name to generate_mrtg_index(), saves state, releases lock
- check_and_heartbeat: Creates prefix once at start, stores in thread_local, performs checks, handles state transitions with proper notification types
- Thread pool uses executor.submit() to launch, then future.result() in sequential loop to wait for ALL threads and propagate exceptions
- All protocols (ping/http/quic/tcp/udp/snmp) flow through same orchestration

## Technical Tactics

**Vendor Detection via sysObjectID Prefix Match**: Query sysObjectID (1.3.6.1.2.1.1.2.0) returns enterprise OID like 1.3.6.1.4.1.9.1.2.3.4 for Cisco device. Use startswith() on known prefixes: 1.3.6.1.4.1.9. (Cisco), 1.3.6.1.4.1.11. (HP), 1.3.6.1.4.1.2636. (Juniper), 1.3.6.1.4.1.41112. (Ubiquiti). Detection stored in vendor variable, drives CPU/memory OID selection. Query failure or unknown prefix leaves vendor=None, triggers HOST-RESOURCES-MIB fallback immediately. Simple string prefix matching more reliable than complex OID tree parsing.

**Vendor-Specific OID Try/Except Isolation**: Each vendor's CPU/memory polls wrapped in separate try/except blocks. Example: Cisco CPU tries cpmCPUTotal5secRev first, catches exception, tries cpmCPUTotal1minRev fallback, catches exception. If both fail, cpu_load remains None, triggers HOST-RESOURCES-MIB attempt. Pattern enables vendor-specific graceful degradation (e.g., Cisco 1-min fallback when 5-sec unavailable) before cross-vendor fallback. Keep vendor logic isolated—don't intermix Cisco and HP OIDs in same try block.

**HOST-RESOURCES-MIB as Universal Fallback**: Standard MIB supported by servers/hosts (Linux, Windows, BSD), not always by network switches (vendor MIBs primary). CPU: hrProcessorLoad returns per-core values, average for aggregate metric. Memory: hrStorageDescr table walk finds physical memory entry by description matching ("physical memory", "ram", or exact "memory"), retrieves hrStorageAllocationUnits (bytes per unit), hrStorageSize (total units), hrStorageUsed (used units), calculates percentage: (used/size) × 100. Fallback triggered by if cpu_load is None and if memory_pct is None checks, ensuring attempt only when vendor OIDs failed.

**Aggregate Metric Calculation in Python**: Total bits/packets calculated before RRD storage, not via MRTG math. Enables flexible combinations (e.g., summing specific interface types), per-interface error handling (missing interface doesn't fail aggregate), and validation before storage. Pattern: iterate interfaces, accumulate octets_in/out and pkts_in/out, convert octets to bits (× 8), store aggregates as separate COUNTER DS in RRD. RRD automatically calculates rates (bits/sec, packets/sec) from cumulative values. Alternative MRTG CDEF approach limited to simple arithmetic, lacks error handling flexibility.

**Packet Counter Summation Across Types**: Total packets = sum of unicast + multicast + broadcast for all interfaces. Uses IF-MIB high-capacity 64-bit counters (ifHCInUcastPkts, ifHCInMulticastPkts, ifHCInBroadcastPkts, ifHCOutUcastPkts, ifHCOutMulticastPkts, ifHCOutBroadcastPkts). Individual counter failures logged at VERBOSE > 1, don't affect aggregate—partial sum better than no data. Pattern matches aggregate bandwidth calculation philosophy. Important: packet counters may not be supported by all devices (older switches), failure graceful.

**COUNTER vs GAUGE DS Type Selection**: COUNTER for cumulative metrics (interface bytes, packets, TCP retransmits)—RRD calculates rate automatically, handles 32/64-bit wraparound. GAUGE for instantaneous values (CPU %, memory %)—stored as-is without rate calculation. MaxBytes in MRTG config scales Y-axis: 10Gbps for bandwidth, 10M pps for packets, 100 for CPU/memory percentages. Type mismatch (e.g., GAUGE for bytes) would require application-level rate calculation and lose RRD wraparound handling—complexity not justified.

**4-Target MRTG Architecture for SNMP**: Separating bandwidth/packets/retransmits/system into distinct targets enables independent Y-axis scaling and focus. Bandwidth needs Gbps scale, packets need Mpps scale, retransmits need 100K scale, system needs 0-100 percentage scale. Single multi-DS target would force shared Y-axis, making some metrics invisible. Alternative would be 4 separate RRD files—complexity and atomicity loss not justified. Current approach: 4 targets reading same RRD file with different DS pairs.

**TCP Retransmits on Separate Graph**: Originally considered combining with packets, separated for visibility. Retransmit scale (0-100K) differs significantly from packet scale (0-10M pps), shared Y-axis would compress retransmit line to near-zero. Separate graph allows independent scaling, better highlights network quality issues. Pattern: system resources (CPU/memory) on same graph work because both 0-100 scale, similar semantics.

**MRTG Target Deduplication by Suffix Detection**: Index HTML generation walks all MRTG targets, detects suffixes (-bandwidth, -packets, -retransmits, -system), extracts base name. Stores base name as dict key (automatic deduplication), enables single row per SNMP device with 4 graph columns. Alternative would generate 4 separate rows—visually cluttered, breaks "one row per device" mental model. Uses dict instead of set to preserve discovery order (Python 3.7+ dicts ordered).

**Site Name Extraction for Index HTML**: generate_mrtg_index() accepts site_name parameter rather than parsing from MRTG config comments. Site name extracted in main() from config['site']['name'], passed during --generate-mrtg-config. Pattern separates config parsing from HTML generation, enables consistent site naming across all MRTG files. Alternative parsing # Site: comments from MRTG configs fragile (comment format changes, missing comments).

**Responsive Grid Breakpoints**: CSS media queries adjust column count by viewport width: >1400px shows 4 columns (optimal for large displays), 1400-768px shows 2 columns (2×2 grid for tablets), <768px shows 1 column (stacked for mobile). Breakpoint values empirically chosen to prevent graph squashing. Network Monitoring and Availability Monitoring sections use identical grid structure for visual consistency.

**Verbose Output Layering**: VERBOSE=1 shows operational summary (vendor detected, final metrics, interface counts). VERBOSE=2 shows per-operation detail (individual SNMP GETs with OIDs/values, packet counter failures, fallback attempts). Layering prevents log flood while enabling deep debugging when needed. Format: [T#XXXX Site/Resource] SNMP CPU (Cisco 5-sec): 15.2% at VERBOSE=1, [T#XXXX Site/Resource] SNMP GET 1.3.6.1.4.1.9.9.109.1.1.1.1.7.1 = 15 at VERBOSE=2.

**SNMP Library Choice**: Using easysnmp (wrapper around Net-SNMP C library) rather than pysnmp (pure Python). easysnmp provides simpler synchronous API with session.walk() and session.get() methods. pysnmp v6+ uses async-only API requiring asyncio integration, adds complexity without performance benefit for APMonitor's use case (sequential polling). easysnmp session-based design fits naturally with thread-per-monitor pattern.

**SNMP Walk with Sorted Results**: session.walk(OID_IF_DESCR) returns list of SNMPVariable objects. Extract interface index from each OID via item.oid.split('.')[-1]. Store in dict keyed by index for stable ordering. Critical: Interface indices may not be sequential (gaps, high values), sorting by index ensures deterministic DS order for RRD operations.

**Partial SNMP Poll Tolerance**: Individual interface poll failures set value to None rather than failing entire check. Reasoning: Better to collect 9/10 interfaces successfully than fail completely because one interface unreachable. Missing values represented as 'U' in RRD update. This matches RRD's design philosophy—gaps acceptable, complete data loss unacceptable. Pattern extended to CPU/memory: unavailable metrics stored as 'U', don't fail check.

**RRD Template Parameter Usage**: rrdtool.update() without template uses DS order from RRD file creation (stored in RRD metadata). With template, explicitly specifies DS order for current update. Code always uses template parameter with sorted DS list to ensure deterministic updates regardless of RRD internal ordering. Prevents "found extra data" and "expected N data values but got M" errors.

**Community String Precedence Chain**: resource.get('community') or parsed.username or 'public' provides three-tier lookup with explicit fallthrough. Short-circuit evaluation ensures first non-None/non-empty value wins. Alternative would require nested if/else blocks. Or-chain more Pythonic and clearer intent.

**RRD DS Name Truncation Strategy**: Truncate interface names to 15 chars before adding _in/_out suffix. Total = 15 + 4 = 19 chars (RRD limit). Alternative would truncate full DS name to 19 chars, risking collision if two interfaces differ only in trailing chars. Truncating base name preserves uniqueness better in practice. Example: "GigabitEthernet0/1" → "GigabitEtherne_in" (15 + 4 = 19).

**SNMP Response Time Calculation**: Response time for SNMP check includes: sysObjectID query + interface walk + (N interfaces × byte polls) + (N interfaces × packet polls) + aggregate calculation + TCP retrans poll + CPU poll + memory poll. This is total duration of all SNMP operations, not per-metric breakdown. Stored in last_response_time_ms for consistency with other monitor types. Useful for heartbeat timing adjustments and performance trending.

**Chained Ternary for Protocol Dispatch**: check_url_resource() uses chained ternary conditional to dispatch to protocol-specific checkers and handle unknown types. This enables unified expect checking after dispatch while handling protocol-specific connection logic separately. SNMP not included here—different return signature, no expect checking.

**Status Code Convention for Non-HTTP Protocols**: TCP/UDP return 200 for success (HTTP-like convention) to enable unified expect checking logic. Alternative would duplicate expect checking across protocols or require protocol-aware branches in check_url_resource(). Convention approach cleaner. SNMP doesn't use this—no status code concept, just error message or None.

**Thread-Local Storage for Prefix**: Using threading.local() eliminates prefix parameter threading through all function calls. Set once in check_and_heartbeat(), retrieved via getattr with default in output functions. Main thread never sets prefix (getattr returns empty string), keeping banner/startup output clean. Balances clean logs with implementation simplicity.

**Explicit Flush for Pipe-Captured Output**: Systemd captures stdout/stderr via pipes (fully buffered, not line-buffered). Without explicit flush, thread output accumulates and appears out-of-order. Strategic flush after state updates ensures output visibility aligns with state mutations. The update_state() flush critical—called by every thread, natural synchronization point.

**Future Result Retrieval Pattern**: After submitting all jobs to thread pool, loop through futures calling future.result(). This blocks until each future completes AND re-raises exceptions from worker threads. Without explicit result retrieval, thread exceptions silently swallowed. Sequential result retrieval ensures proper exception propagation while maintaining execution concurrency.

**Socket Timeout Coordination**: Socket timeout set to MAX_TRY_SECS to ensure operation completes within retry interval. For TCP, connection + send + receive must complete within timeout. For UDP, send + receive must complete. For SNMP, session timeout set to MAX_TRY_SECS, retries set to MAX_RETRIES-1 (easysnmp API). Timeout expiration raises exceptions caught in try/except blocks.

**Configuration Change Detection**: JSON-serialize monitor config with sort_keys=True, compute SHA-256, store as last_config_checksum. Compare on load—mismatch forces immediate check bypassing timing intervals. Detects any field change including protocol parameters, SNMP community string, vendor-specific settings.

**Atomic State File Rotation**: Write to .new, atomically rename to current. Keep .old as backup. Ensures state consistency even with kill -9. State mutations always: update memory → write .new → flush → rotate on exit.

## Engineering Principles for This Code

**Vendor Detection First, Fallback Second**: Always attempt vendor detection via sysObjectID before falling back to HOST-RESOURCES-MIB. Pattern: detect vendor → try vendor OIDs → check if metrics None → try HOST-RESOURCES-MIB → check if metrics None → store 'U'. Never skip vendor detection for devices that might support it. Fallback is safety net, not primary strategy. Detection failure (exception on sysObjectID query) acceptable—log and proceed to fallback.

**Isolate Vendor-Specific Logic with Try/Except**: Each vendor's CPU/memory polls must be in separate try/except blocks. Never combine vendor OID attempts in shared exception handler—obscures which vendor OID failed. Pattern enables vendor-specific fallbacks (e.g., Cisco 1-min after 5-sec fails) before cross-vendor fallback. Cisco fallback to Cisco alternative OID cleaner than jumping to HP OIDs.

**HOST-RESOURCES-MIB Is Universal Fallback, Not Primary**: Standard MIB designed for servers/hosts, may not be supported by network switches. Always attempt vendor-specific OIDs first when vendor detected. Fallback triggers only when vendor OIDs fail or device unrecognized. Never assume HOST-RESOURCES-MIB available—network gear often lacks it. Log fallback attempts at VERBOSE for troubleshooting.

**Partial Data Collection Over Complete Failure**: Individual metric failures (interface poll, packet counter, CPU, memory) should not fail entire SNMP check. Store 'U' (unknown) in RRD for missing values. Reasoning: 9/10 metrics successfully collected provides useful data. Complete failure provides nothing. RRD designed to handle gaps gracefully. User sees "cpu=unavailable" in verbose output, understands limitation, still gets bandwidth data.

**Aggregate Metrics Calculated in Application, Not MRTG**: Total bits/packets computed before RRD storage enables flexible combinations, per-interface error handling, validation before storage. Alternative MRTG CDEF approach limited to simple arithmetic, lacks error handling. Pattern: iterate interfaces, accumulate values with exception handling per interface, convert/calculate, store aggregates as COUNTER DS. RRD auto-calculates rates from cumulative values.

**COUNTER for Cumulative, GAUGE for Instantaneous**: Use COUNTER not GAUGE for interface bytes, packets, TCP retransmits. COUNTER calculates rate automatically, handles wraparound, matches SNMP/MRTG conventions. Use GAUGE for CPU %, memory %—instantaneous values stored as-is. Never mix types for similar metrics. Type mismatch creates confusion, breaks graphing, requires application-level workarounds.

**4 Targets Per SNMP Monitor for Independent Scaling**: Bandwidth needs Gbps scale, packets need Mpps scale, retransmits need 100K scale, system needs 0-100 scale. Separate targets enable independent Y-axis scaling. Alternative single multi-DS target forces shared Y-axis, makes some metrics invisible. Pattern: 4 targets reading same RRD with different DS pairs. Non-SNMP monitors unchanged—single target with response_time&is_up.

**Deduplicate MRTG Targets by Base Name for Clean Layout**: Index HTML must show one row per SNMP device with 4 graphs (bandwidth, packets, retransmits, system). Detect suffixes in target names, extract base name, store in dict for automatic deduplication. Never generate 4 separate rows—breaks "one row per device" mental model, clutters UI. Pattern: dict key = base name, value = discovered target suffixes.

**Site Name from Config, Not MRTG Comment Parsing**: Extract site name in main() from config['site']['name'], pass to generate_mrtg_index(). Never parse # Site: comments from MRTG configs—fragile, inconsistent, breaks when comments missing/malformed. Pattern separates config parsing from HTML generation, ensures consistent site naming. Parameter default "Availability Monitoring" for backward compatibility.

**Responsive Grid Must Scale Gracefully**: CSS breakpoints must prevent graph squashing on small screens. Pattern: >1400px=4col (large displays), 1400-768px=2col (tablets), <768px=1col (mobile). Test breakpoints with actual MRTG graph sizes—too many columns makes graphs unreadable. Network Monitoring and Availability Monitoring sections use identical breakpoints for visual consistency.

**Verbose Output Must Be Layered**: VERBOSE=1 for operational summary (vendor detected, final metrics, interface counts), VERBOSE=2 for per-operation detail (individual SNMP GETs, OID values, fallback attempts). Never flood logs with detail at VERBOSE=1—users want summary. Never hide critical info at VERBOSE=0—silent failures bad. Layer appropriately: operational events always shown, diagnostic details gated by VERBOSE level.

**SNMP Monitors Return Error String Not Tuple**: SNMP returns Optional[str] like ping, not (error_msg, status_code, headers, response_text) like URL resources. Reason: SNMP polls multiple metrics per check (N interfaces + aggregates + TCP + CPU + memory), doesn't fit single response paradigm. Expect checking not applicable—SNMP validates by successful metric retrieval, not content matching. Keep SNMP isolated from URL resource machinery.

**All Interfaces In Single RRD File**: Never create separate RRD file per interface. Single file enables atomic updates of all interface counters, prevents timestamp skew between interfaces, simplifies state management. Interface list discovered on first poll, RRD created once with all DS, then updated on subsequent polls. If interface list changes, current implementation keeps stale DS (unused but harmless). Alternative would recreate RRD—complexity not justified.

**Stable Sort Interface Indices for RRD Operations**: Always use sorted(interfaces.keys()) when building DS lists for RRD create/update. RRD template parameter requires exact DS order match. Without stable sort, DS order nondeterministic, causing RRD errors. Interface indices may not be sequential—sorting by index ensures consistency.

**SNMP Community String Has Explicit Precedence**: Three-tier lookup: monitor field → URL userinfo → default "public". Document precedence clearly. Never make precedence implicit or undefined. Users should understand which value wins when multiple specified. Or-chain evaluation order is the implementation contract.

**Sanitize Interface Names Deterministically**: re.sub(r'[^\w]', '_', if_name)[:15] ensures filesystem-safe, RRD-compatible DS names. Truncation before suffix addition prevents RRD 19-char limit violations. Alternative regex patterns (e.g., whitelist alphanumeric only) acceptable if consistently applied. Key: deterministic mapping from SNMP interface name to RRD DS name.

**RRD Path Uses Metric Type Parameter**: get_rrd_path(monitor_name, metric_type='availability') enables multiple RRD files per monitor. Availability monitors use 'availability', SNMP monitors use 'snmp'. Pattern supports future metric types (e.g., 'performance', 'security') without function signature changes. Alternative would encode metric type in monitor name—less flexible.

**Protocol-Specific Checkers Return Uniform Tuples (URL Resources Only)**: HTTP/QUIC/TCP/UDP must return (error_msg, status_code, headers, response_text) to enable unified expect checking. SNMP and ping exempt—different monitoring paradigms. Never make check_url_resource() protocol-aware—keep branching logic minimal by enforcing tuple contract.

**Thread-Local for Prefix Eliminates Parameter Threading**: Using threading.local() cleaner than passing prefix parameter through every function. Set once, retrieve many. Main thread never sets prefix (clean banner/startup output). All thread output functions use getattr(thread_local, 'prefix', '') pattern. Applies to all protocol types including SNMP.

**Flush at Synchronization Points**: Flush after banner, after config messages, inside update_state(), and in finally block. The update_state() flush key—called by every thread after work completion, natural synchronization point for output visibility. Applies uniformly regardless of protocol type.

**Prefix ALL Thread Output**: Every print/stderr from worker threads must include prefix. Main thread output (banner, startup, execution time) stays unprefixed for clarity. Enables clean log filtering and audit trails without log line ambiguity. SNMP verbose output includes prefix for all SNMP GET operations, vendor detection, fallback attempts.

**Validate Before Execute**: Config validation must be comprehensive and fail-fast. Better to exit with clear error than silently ignore invalid config. Validate types, formats, constraints, cross-field dependencies, protocol-specific rules. SNMP validation includes scheme check, hostname validation, forbidden field checks (expect, ssl_fingerprint, send, content_type).

**Configuration Change Detection Triggers Immediate Check**: SHA-256 mismatch on config load must bypass timing intervals and force immediate check. Detects any field change including protocol parameters, SNMP community string, vendor-specific settings, aggregate metric selection. Never cache old config—always load fresh, compute checksum, compare, act.

**Response Time Tracks All Protocols**: Calculate and store last_response_time_ms for ping, HTTP, QUIC, TCP, UDP, SNMP. Useful for performance trending and heartbeat timing adjustments. SNMP response time includes total poll duration (sysObjectID + interface walk + all metric polls + aggregation). Don't break down per-metric—total duration sufficient for monitoring purposes.

**Boolean Handling Uniformity**: Use to_natural_language_boolean() everywhere. Never scatter boolean checks. Provides consistent behavior and clear error messages. Handles None gracefully (returns False). Applies to all monitor types.

**PID Lock Cleanup in Finally**: Lockfile removal must be in outermost finally block to handle all exit paths. Never use multiple cleanup locations. Hash-based naming prevents collisions while enabling duplicate detection. Applies regardless of monitor types in config.

Would you like to see the code?