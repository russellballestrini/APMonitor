# APMONITOR.PY v1.2.5 - NETWORK MONITORING WITH SNMP BANDWIDTH METRICS

Hey! Welcome back to APMonitor.py. We've just completed SNMP monitoring support for network device interface bandwidth and TCP retransmit metrics, with full RRD integration for historical graphing. All connection-oriented protocols (HTTP/QUIC/TCP/UDP/SNMP) now flow through unified architecture patterns. Let me bring you up to speed.

## Purpose

APMonitor monitors network resources (ping/HTTP/HTTPS/QUIC/TCP/UDP/SNMP) on-premises with guaranteed alert delivery via external heartbeat integration. SNMP monitors collect interface bandwidth utilization and TCP retransmit metrics, storing data in RRD files for MRTG graphing. Designed for repeated cron/systemd invocation, not daemon mode.

## What APMonitor Does

- Loads YAML/JSON configuration defining site, monitors (ping/http/quic/tcp/udp/snmp), email/webhook notifications, timing parameters
- Validates configuration including SNMP-specific rules (scheme validation, community string, forbidden fields)
- Checks resource availability via ICMP ping, HTTP/HTTPS GET/POST, QUIC/HTTP3 GET/POST, TCP connection/banner, UDP send/receive, SNMP polling
- SNMP monitors automatically discover all device interfaces via SNMP walk of IF-MIB::ifDescr table
- SNMP monitors poll interface byte counters (ifInOctets, ifOutOctets) and global TCP retransmit counter (tcpRetransSegs)
- Supports three-way SNMP community string specification: monitor-level `community` field, URL userinfo (`snmp://community@host`), or default `public`
- Uses easysnmp library for synchronous SNMP operations with configurable timeout and retries
- Stores SNMP metrics in RRD files with dynamic data sources per discovered interface plus TCP retransmits
- RRD COUNTER type automatically calculates rates (bytes/second) and handles 32/64-bit wraparound
- All interfaces for a device stored in single RRD file (`{monitor-name}-snmp.rrd`) for atomic updates
- Supports HTTP/QUIC POST requests with configurable MIME types (application/json, application/x-www-form-urlencoded, text/plain)
- Supports TCP/UDP protocol checks with text/hex/base64 encoding for send data and optional expect validation
- TCP monitors automatically receive data after connecting for banner protocol support (SSH/SMTP/FTP)
- UDP monitors require send parameter (connectionless protocol), support fire-and-forget or expect-based validation
- Enforces per-monitor check intervals with site-level defaults and configuration change detection via SHA-256 checksums
- Tracks persistent state in JSON statefile with atomic rotation
- Sends email notifications via SMTP with per-recipient control flags (outages/recoveries/reminders)
- Sends webhook notifications (GET/POST with URL/HTML/JSON/CSVQUOTED encoding)
- Enforces notification throttling with escalating delays via quadratic Bezier curve
- Pings heartbeat URLs when resources are up with configurable intervals
- Validates SSL certificates via SHA-256 fingerprints and expiration checks (HTTP/QUIC only)
- Generates MRTG config files with dynamic targets for SNMP monitors when `--generate-mrtg-config` specified
- Uses PID lockfiles to prevent duplicate instances per config file
- Runs multi-threaded with explicit stdout flushing for proper log ordering
- Prefixes all thread output with thread ID and site/resource context

## Key Architecture

**SNMP Monitor Integration with Check Flow**: SNMP monitors integrate into existing `check_resource()` dispatch but don't use unified `check_url_resource()` function. SNMP is fundamentally different—polls multiple metrics per check (N interfaces + TCP retrans) rather than single up/down status. Function `check_snmp_resource()` returns `Optional[str]` (error message or None) like `check_ping_resource()`, not the `(error_msg, status_code, headers, response_text)` tuple used by URL resources. This keeps SNMP logic isolated from HTTP/QUIC/TCP/UDP expect-checking machinery.

**SNMP RRD Storage Architecture**: Each SNMP monitor creates single RRD file with dynamic data sources. Interface names sanitized to alphanumeric+underscore, truncated to 15 chars (leaves room for `_in`/`_out` suffix within 19-char DS limit). Data sources: `{interface}_in` (COUNTER), `{interface}_out` (COUNTER), `tcp_retrans` (COUNTER). All interfaces stored in one file for atomic updates—avoids race conditions from separate file-per-interface approach. Interface list discovered on first poll, RRD created with all discovered DS, then updated on subsequent polls.

**RRD File Naming Convention**: Availability monitors use `{monitor-name}-availability.rrd`, SNMP monitors use `{monitor-name}-snmp.rrd`. Pattern implemented in `get_rrd_path(monitor_name, metric_type='availability')`. This enables both availability tracking (for ping/http/quic/tcp/udp) and performance tracking (for SNMP) for same logical resource if needed. All RRD files live in `{statefile_dir}/{statefile_stem}.rrd/` directory.

**SNMP Community String Precedence**: Three-way lookup with explicit precedence order: (1) monitor-level `community` field, (2) URL userinfo from parsed address (`snmp://community@host`), (3) default `public`. Code: `community = resource.get('community') or parsed.username or 'public'`. This allows per-monitor overrides while providing sensible defaults. Precedence order documented clearly for users.

**Interface Name Sanitization**: SNMP interface names (e.g., "GigabitEthernet0/1") contain characters invalid for RRD DS names. Sanitization: `re.sub(r'[^\w]', '_', if_name)[:15]`. Truncation to 15 chars leaves 4 chars for `_in`/`_out` suffix within 19-char RRD DS limit. Pattern ensures deterministic DS names while preventing RRD creation failures.

**SNMP Stable Sorting for Deterministic DS Order**: RRD update requires template string with DS names in exact order matching RRD creation. Code uses `sorted(interfaces.keys())` to ensure consistent ordering across create and update operations. Without stable sort, DS order nondeterministic, causing RRD update failures. Critical for multi-interface devices.

**Unified URL Resource Architecture**: HTTP/QUIC/TCP/UDP share common entry point `check_url_resource()`. This function handles expect checking universally by extracting `(error_msg, status_code, headers, response_text)` tuples from protocol-specific checkers. HTTP/QUIC return actual status codes (200, 404, etc.), TCP/UDP return 200 for success by convention. Expect logic lives once in `check_url_resource()`, not duplicated per protocol.

**Content-Type Semantic Split**: HTTP/QUIC treat `content_type` as raw MIME type header (e.g., "application/json"), TCP/UDP treat it as wire-level encoding format (text/hex/base64). Both protocols accept `send` parameter but interpret it differently: HTTP/QUIC always UTF-8 encode for POST body, TCP/UDP decode according to content_type before transmission. This semantic split allows natural expression of both use cases.

**TCP Banner Protocol Support**: TCP monitors without `send` parameter perform connection-only checks but always attempt to receive data (via `sock.recv(4096)` wrapped in try/except). This enables monitoring banner protocols (SSH, SMTP, FTP) that send greetings on connect without requiring explicit send. Timeout on receive not fatal unless expect specified—allows monitoring both interactive and passive TCP services.

**UDP Connectionless Behavior**: UDP requires `send` parameter (validation enforced). Success semantics differ from TCP: without expect, success means `sendto()` didn't raise socket error (packet may be dropped silently—UDP provides no delivery confirmation). With expect, success requires matching response within timeout. This matches UDP's connectionless nature while providing practical monitoring capability.

**Thread-Aware Prefixing**: `prefix_logline(site_name, resource_name)` generates `[T#XXXX Site/Resource]` prefix using `threading.get_native_id()` for unique identification. Prefix created once at start of `check_and_heartbeat()`, stored in thread_local storage, passed to all functions via `thread_local.prefix`. Enables clean log filtering and audit trails without parameter threading bloat.

**Output Buffering Solution**: Python stdout line-buffered for terminals but fully buffered for pipes. Systemd captures via pipes, causing interleaved output. Solution: explicit `sys.stdout.flush()` after welcome banner, startup messages, and critically inside `update_state()` after every state write. The `update_state()` flush ensures thread output visible atomically with state mutations.

**Stateless Execution with Flush Discipline**: Each invocation loads state, performs checks, flushes output at synchronization points, saves state atomically, exits. Output ordering guaranteed by strategic flush placement.

**Per-Config PID Locking**: Creates `/tmp/apmonitor-{hash}.lock` where hash = SHA256(config_path)[:16]. Prevents duplicate processes per config, allows concurrent monitoring of different sites. Stale lockfile detection via `os.kill(pid, 0)`. Unix-only.

**Thread-Safe State Management**: Global `STATE` dict protected by `STATE_LOCK`. Updates written immediately to `.new` file inside lock, flushed immediately after lock release.

**Configuration Change Detection**: SHA-256 checksum of JSON-serialized monitor config triggers immediate checks when configuration changes, bypassing timing intervals.

**Interval-Based Scheduling**: Each monitor tracks last_checked, last_notified, last_successful_heartbeat timestamps. Decisions made by comparing elapsed time against configured intervals.

**Bezier Curve Notification Escalation**: Notification delays follow quadratic Bezier curve over first N notifications, then plateau. Formula: `t = (1/N) * index`, `delay = (1-t)² * 0 + 2(1-t)t * base + t² * base`.

## Important Modules & Communication

**SNMP Resource Checker** (`check_snmp_resource`):
- Validates `snmp://` scheme, extracts hostname and port (default 161)
- Resolves community string via precedence: monitor field → URL userinfo → default "public"
- Creates easysnmp Session with SNMPv2c, community, timeout=MAX_TRY_SECS, retries=MAX_RETRIES-1
- Walks IF-MIB::ifDescr table (OID 1.3.6.1.2.1.2.2.1.2) to discover all interfaces
- For each interface: polls ifInOctets (OID 1.3.6.1.2.1.2.2.1.10) and ifOutOctets (OID 1.3.6.1.2.1.2.2.1.16)
- Polls TCP-MIB::tcpRetransSegs (OID 1.3.6.1.2.1.6.12.0) for global TCP retransmit counter
- Individual interface poll failures set value to None (partial data better than no data)
- If RRD_ENABLED: creates SNMP RRD if missing via `create_snmp_rrd()`, updates via `update_snmp_rrd()`
- Returns None on success, error message string on failure
- Verbose output shows per-metric SNMP GET operations with OIDs and values

**SNMP RRD Creation** (`create_snmp_rrd`):
- Accepts RRD path, check interval, interfaces dict (keys=indices, values={name, ...})
- Sanitizes interface names: alphanumeric+underscore, max 15 chars
- Creates data sources: `{safe_name}_in` (COUNTER), `{safe_name}_out` (COUNTER), `tcp_retrans` (COUNTER)
- COUNTER type automatically calculates rates from cumulative values, handles wraparound
- Heartbeat = 2 × check interval (allows one missed update)
- Uses existing `create_rrd_rras()` for MRTG-compatible retention policy (1d/2d/12d/50d/2y aggregations)
- Stable sort on interface indices ensures deterministic DS order matching update operations

**SNMP RRD Update** (`update_snmp_rrd`):
- Accepts RRD path, timestamp, interfaces dict, tcp_retrans value
- Stable sort on interface indices (`sorted(interfaces.keys())`) ensures DS order matches creation
- Builds template string with DS names in consistent order
- Collects values in same order as template
- Missing values represented as 'U' (unknown) in RRD update
- Uses `rrdtool.update()` with `--template` parameter for explicit DS ordering
- Critical: DS order must be identical between create and update, hence stable sort

**RRD Path Generation** (`get_rrd_path`):
- Signature: `get_rrd_path(monitor_name: str, metric_type: str = 'availability') -> str`
- Sanitizes monitor name to filesystem-safe characters via `re.sub(r'[^\w\-.]', '_', monitor_name)`
- Constructs path: `{statefile_dir}/{statefile_stem}.rrd/{safe_name}-{metric_type}.rrd`
- metric_type values: 'availability' (ping/http/quic/tcp/udp) or 'snmp' (SNMP monitors)
- Enables both availability and performance tracking for same logical resource if needed

**Protocol Function Signatures** (unified tuple return for URL resources):
All URL resource checkers return: `(error_msg: Optional[str], status_code: Optional[int], headers: Any, response_text: Optional[str])`
- `check_http_url_resource()`: Returns actual HTTP status code, headers dict, decoded response text
- `check_quic_url_resource()`: Returns HTTP/3 status code, headers dict, decoded response text
- `check_tcp_url_resource()`: Returns 200 for success (HTTP-like convention), empty headers dict {}, received text
- `check_udp_url_resource()`: Returns 200 for success, empty headers dict {}, received text if any
This uniform interface enables `check_url_resource()` to handle expect checking identically across all protocols.

**Check Resource Dispatch** (`check_resource`):
- Top-level dispatcher based on resource['type']
- Ping: calls `check_ping_resource()` directly (returns Optional[str])
- HTTP/QUIC/TCP/UDP: calls `check_url_resource()` (returns Optional[str] after tuple processing)
- SNMP: calls `check_snmp_resource()` directly (returns Optional[str])
- Updates state atomically after each check with thread-safe locking
- Returns error message or None consistently across all protocol types

**Configuration Validation** (`print_and_exit_on_bad_config`):
SNMP-specific validation rules:
- SNMP monitors must use `snmp://` scheme
- Address must include hostname/IP (validated via urlparse)
- Hostname must be valid IPv4, IPv6, or DNS name (regex patterns)
- `community` field optional, must be non-empty string if specified
- `expect`, `ssl_fingerprint`, `ignore_ssl_expiry` not allowed for SNMP
- `send`, `content_type` not allowed for SNMP
- `heartbeat_url`, `heartbeat_every_n_secs` allowed (same as other types)
All validation happens before any monitoring begins—fail-fast on config errors.

**State Management** (`load_state`, `save_state`, `update_state`):
- `load_state`: Reads JSON at startup
- `update_state`: Thread-safe in-memory update + immediate write to `.new` + immediate flush
- `save_state`: Atomic rotation (current → `.old`, `.new` → current)
- Per-monitor state includes: is_up, last_checked, last_response_time_ms, down_count, last_alarm_started, last_notified, last_successful_heartbeat, notified_count, error_reason, last_config_checksum
- SNMP monitors track response time as total poll duration (includes all interface queries)

**Main Orchestration** (`check_and_heartbeat`, `main`):
- `main`: Acquires PID lock, loads config/state, spawns thread pool, waits for completion with explicit result retrieval, flushes output, records execution time, saves state, releases lock
- `check_and_heartbeat`: Creates prefix once at start, stores in thread_local, performs checks, handles state transitions with proper notification types
- Thread pool uses `executor.submit()` to launch, then `future.result()` in sequential loop to wait for ALL threads and propagate exceptions
- All protocols (ping/http/quic/tcp/udp/snmp) flow through same orchestration

## Technical Tactics

**SNMP Library Choice**: Using easysnmp (wrapper around Net-SNMP C library) rather than pysnmp (pure Python). easysnmp provides simpler synchronous API with session.walk() and session.get() methods. pysnmp v6+ uses async-only API requiring asyncio integration, adds complexity without performance benefit for APMonitor's use case (sequential interface polling). easysnmp session-based design fits naturally with thread-per-monitor pattern.

**SNMP Walk with Sorted Results**: `session.walk(OID_IF_DESCR)` returns list of SNMPVariable objects. Extract interface index from each OID via `item.oid.split('.')[-1]`. Store in dict keyed by index for stable ordering. Critical: Interface indices may not be sequential (gaps, high values), sorting by index ensures deterministic DS order for RRD operations.

**Partial SNMP Poll Tolerance**: Individual interface poll failures set value to None rather than failing entire check. Reasoning: Better to collect 9/10 interfaces successfully than fail completely because one interface unreachable. Missing values represented as 'U' in RRD update. This matches RRD's design philosophy—gaps acceptable, complete data loss unacceptable.

**RRD Template Parameter Usage**: `rrdtool.update()` without template uses DS order from RRD file creation (stored in RRD metadata). With template, explicitly specifies DS order for current update. Code always uses template parameter with sorted DS list to ensure deterministic updates regardless of RRD internal ordering. Prevents "found extra data" and "expected N data values but got M" errors.

**Community String Precedence Chain**: `resource.get('community') or parsed.username or 'public'` provides three-tier lookup with explicit fallthrough. Short-circuit evaluation ensures first non-None/non-empty value wins. Alternative would require nested if/else blocks. Or-chain more Pythonic and clearer intent.

**SNMP Verbose Output Per Metric**: When VERBOSE enabled, prints each SNMP GET operation with full OID and returned value. Format: `[T#XXXX Site/Switch] SNMP GET 1.3.6.1.2.1.2.2.1.10.1 (ifInOctets) = 12345678`. Enables debugging of SNMP communication issues, verifying device responses, understanding interface discovery. High-signal telemetry without excessive noise.

**RRD DS Name Truncation Strategy**: Truncate interface names to 15 chars before adding `_in`/`_out` suffix. Total = 15 + 4 = 19 chars (RRD limit). Alternative would truncate full DS name to 19 chars, risking collision if two interfaces differ only in trailing chars. Truncating base name preserves uniqueness better in practice. Example: "GigabitEthernet0/1" → "GigabitEtherne_in" (15 + 4 = 19).

**COUNTER vs GAUGE for SNMP Metrics**: Using COUNTER type for all SNMP metrics (interface bytes, TCP retransmits). COUNTER automatically calculates rate from cumulative values, stores rate in RRD (bytes/second not total bytes). Handles 32-bit and 64-bit wraparound. Alternative GAUGE would store absolute values, requiring application-level rate calculation. COUNTER matches MRTG/SNMP monitoring best practices.

**SNMP Response Time Calculation**: Response time for SNMP check includes: interface walk time + (N interfaces × 2 poll operations) + TCP retrans poll. This is total duration of all SNMP operations, not per-interface breakdown. Stored in last_response_time_ms for consistency with other monitor types. Useful for heartbeat timing adjustments and performance trending.

**Chained Ternary for Protocol Dispatch**: `check_url_resource()` uses chained ternary conditional to dispatch to protocol-specific checkers and handle unknown types:
```python
error_msg, status_code, headers, response_text = (
    check_http_url_resource(...) if resource_type == 'http'
    else check_quic_url_resource(...) if resource_type == 'quic'
    else check_tcp_url_resource(...) if resource_type == 'tcp'
    else check_udp_url_resource(...) if resource_type == 'udp'
    else (f"Unknown URL resource type: {resource_type}", None, None, None)
)
```
This enables unified expect checking after dispatch while handling protocol-specific connection logic separately. SNMP not included here—different return signature, no expect checking.

**Status Code Convention for Non-HTTP Protocols**: TCP/UDP return 200 for success (HTTP-like convention) to enable unified expect checking logic. Alternative would duplicate expect checking across protocols or require protocol-aware branches in `check_url_resource()`. Convention approach cleaner. SNMP doesn't use this—no status code concept, just error message or None.

**Hex Encoding with Whitespace Stripping**: For hex content_type, clean input via `send_data.replace(' ', '').replace(':', '')` before `bytes.fromhex()`. Allows user-friendly hex specification with spaces/colons (e.g., "01 02 03 04" or "01:02:03:04") while ensuring valid hex parsing. Used by TCP/UDP, not applicable to SNMP.

**TCP Banner Always-Receive Pattern**: Wrap `sock.recv(4096)` in try/except even when send_data not specified. Timeout on receive logged but not fatal. Enables monitoring banner protocols (SSH, SMTP, FTP) without explicit send configuration. If expect specified but timeout, that IS fatal—expect requires successful receive.

**UDP Fire-and-Forget vs Expect-Based**: Without expect, UDP success means `sendto()` succeeded (packet transmitted). With expect, success requires matching response within timeout. This matches UDP's delivery guarantees—sendto() success doesn't mean packet arrived or service processed it. Practical monitoring requires expect for real validation.

**Thread-Local Storage for Prefix**: Using `threading.local()` eliminates prefix parameter threading through all function calls. Set once in `check_and_heartbeat()`, retrieved via getattr with default in output functions. Main thread never sets prefix (getattr returns empty string), keeping banner/startup output clean. Balances clean logs with implementation simplicity.

**Explicit Flush for Pipe-Captured Output**: Systemd captures stdout/stderr via pipes (fully buffered, not line-buffered). Without explicit flush, thread output accumulates and appears out-of-order. Strategic flush after state updates ensures output visibility aligns with state mutations. The `update_state()` flush critical—called by every thread, natural synchronization point.

**Future Result Retrieval Pattern**: After submitting all jobs to thread pool, loop through futures calling `future.result()`. This blocks until each future completes AND re-raises exceptions from worker threads. Without explicit result retrieval, thread exceptions silently swallowed. Sequential result retrieval ensures proper exception propagation while maintaining execution concurrency.

**Socket Timeout Coordination**: Socket timeout set to MAX_TRY_SECS to ensure operation completes within retry interval. For TCP, connection + send + receive must complete within timeout. For UDP, send + receive must complete. For SNMP, session timeout set to MAX_TRY_SECS, retries set to MAX_RETRIES-1 (easysnmp API). Timeout expiration raises exceptions caught in try/except blocks.

**Configuration Change Detection**: JSON-serialize monitor config with `sort_keys=True`, compute SHA-256, store as `last_config_checksum`. Compare on load—mismatch forces immediate check bypassing timing intervals. Detects any field change including protocol parameters and SNMP community string.

**Atomic State File Rotation**: Write to `.new`, atomically rename to current. Keep `.old` as backup. Ensures state consistency even with kill -9. State mutations always: update memory → write .new → flush → rotate on exit.

**SSL Certificate Pinning**: Fetch cert, convert PEM→DER, compute SHA-256, compare to configured fingerprint. Works for HTTP and QUIC. Enables trust of self-signed certs. Not applicable to TCP/UDP/SNMP (no TLS layer at monitoring level).

**QUIC/HTTP3 Implementation**: Async function using aioquic. Custom protocol class handles HTTP/3 events. Wraps in `asyncio.run()` for sync interface. Timeout via `asyncio.timeout()`. Peer certificate from TLS layer. POST support via send_headers() with end_stream=False then send_data() with end_stream=True.

## Engineering Principles for This Code

**SNMP Monitors Return Error String Not Tuple**: SNMP returns `Optional[str]` like ping, not `(error_msg, status_code, headers, response_text)` like URL resources. Reason: SNMP polls multiple metrics per check (N interfaces + TCP retrans), doesn't fit single response paradigm. Expect checking not applicable—SNMP validates by successful metric retrieval, not content matching. Keep SNMP isolated from URL resource machinery.

**All Interfaces In Single RRD File**: Never create separate RRD file per interface. Single file enables atomic updates of all interface counters, prevents timestamp skew between interfaces, simplifies state management. Interface list discovered on first poll, RRD created once with all DS, then updated on subsequent polls. If interface list changes, current implementation keeps stale DS (unused but harmless). Alternative would recreate RRD—complexity not justified.

**Stable Sort Interface Indices for RRD Operations**: Always use `sorted(interfaces.keys())` when building DS lists for RRD create/update. RRD template parameter requires exact DS order match. Without stable sort, DS order nondeterministic, causing RRD errors. Interface indices may not be sequential—sorting by index ensures consistency.

**Partial Poll Data Better Than No Data**: Individual interface poll failures set value to None, represented as 'U' in RRD update. Don't fail entire SNMP check because one interface unreachable. Reasoning: 9/10 interfaces successfully polled provides useful data. Complete failure provides nothing. RRD designed to handle gaps gracefully.

**SNMP Community String Has Explicit Precedence**: Three-tier lookup: monitor field → URL userinfo → default "public". Document precedence clearly. Never make precedence implicit or undefined. Users should understand which value wins when multiple specified. Or-chain evaluation order is the implementation contract.

**Sanitize Interface Names Deterministically**: `re.sub(r'[^\w]', '_', if_name)[:15]` ensures filesystem-safe, RRD-compatible DS names. Truncation before suffix addition prevents RRD 19-char limit violations. Alternative regex patterns (e.g., whitelist alphanumeric only) acceptable if consistently applied. Key: deterministic mapping from SNMP interface name to RRD DS name.

**COUNTER Type for Cumulative Metrics**: Use COUNTER not GAUGE for interface bytes and TCP retransmits. COUNTER calculates rate automatically, handles wraparound, matches SNMP/MRTG conventions. GAUGE would require application-level rate calculation and wraparound handling—complexity not justified.

**Verbose Output Shows All SNMP Operations**: When VERBOSE enabled, print each SNMP GET with full OID and value. Format: `[prefix] SNMP GET {OID} ({description}) = {value}`. Enables debugging SNMP communication, verifying device responses, understanding interface discovery. High-signal telemetry—every line provides actionable diagnostic information.

**RRD Path Uses Metric Type Parameter**: `get_rrd_path(monitor_name, metric_type='availability')` enables multiple RRD files per monitor. Availability monitors use 'availability', SNMP monitors use 'snmp'. Pattern supports future metric types (e.g., 'performance', 'security') without function signature changes. Alternative would encode metric type in monitor name—less flexible.

**Protocol-Specific Checkers Return Uniform Tuples (URL Resources Only)**: HTTP/QUIC/TCP/UDP must return `(error_msg, status_code, headers, response_text)` to enable unified expect checking. SNMP and ping exempt—different monitoring paradigms. Never make `check_url_resource()` protocol-aware—keep branching logic minimal by enforcing tuple contract.

**Status Codes Are HTTP-Like Conventions (URL Resources Only)**: TCP/UDP return 200 for success even though they're not HTTP. This enables unified status checking rather than protocol-specific branches. Convention beats complexity when interfaces are shared. SNMP doesn't use this—no status code concept.

**Content-Type Semantics Are Protocol-Specific**: HTTP/QUIC treat content_type as MIME type header, TCP/UDP treat as encoding format. Document this clearly but don't try to unify—different protocols have different natural interpretations. Split semantics better than forced consistency. SNMP doesn't have content_type—not applicable.

**Always-Receive for Banner Protocols (TCP Only)**: TCP should always attempt receive even without send_data. Many protocols (SSH, SMTP, FTP, POP3, IMAP) send banners on connect. Timeout on receive logged but not fatal unless expect specified. This enables monitoring without protocol-specific knowledge. Not applicable to UDP (no connection) or SNMP (application-layer protocol).

**UDP Requires Send Parameter**: Enforce this in validation. UDP is connectionless—can't verify service listening without application-layer data exchange. Fire-and-forget (no expect) allowed but documented as "packet sent" not "service responding." SNMP also connectionless but validation different—SNMP session handles protocol details.

**Thread-Local for Prefix Eliminates Parameter Threading**: Using `threading.local()` cleaner than passing prefix parameter through every function. Set once, retrieve many. Main thread never sets prefix (clean banner/startup output). All thread output functions use `getattr(thread_local, 'prefix', '')` pattern. Applies to all protocol types including SNMP.

**Flush at Synchronization Points**: Flush after banner, after config messages, inside `update_state()`, and in finally block. The `update_state()` flush key—called by every thread after work completion, natural synchronization point for output visibility. Applies uniformly regardless of protocol type.

**Prefix ALL Thread Output**: Every print/stderr from worker threads must include prefix. Main thread output (banner, startup, execution time) stays unprefixed for clarity. Enables clean log filtering and audit trails without log line ambiguity. SNMP verbose output includes prefix for all SNMP GET operations.

**Thread Pool Exception Handling**: Always call `future.result()` on all futures to propagate exceptions. Without explicit result retrieval, thread exceptions silently lost. Wrap each result retrieval in try/except to handle gracefully rather than crashing main thread. Applies to all monitor types including SNMP.

**State Transitions Require Prefix**: Outage and recovery messages include prefix for audit trails. Makes clear which thread detected which state transition. Critical for debugging timing issues or understanding concurrent execution. SNMP monitors follow same pattern.

**Validate Before Execute**: Config validation must be comprehensive and fail-fast. Better to exit with clear error than silently ignore invalid config. Validate types, formats, constraints, cross-field dependencies, protocol-specific rules. SNMP validation includes scheme check, hostname validation, forbidden field checks.

**Hex/Base64 Encoding Is Convenience (TCP/UDP Only)**: Allow user-friendly hex input with spaces/colons, strip before parsing. This about human readability in config files—internal representation always bytes. Don't enforce strict hex format in config. Not applicable to SNMP—no send data concept.

**Socket Cleanup in Finally**: Always close sockets in finally block even if exception raised. Resource leaks bad. Pattern: create socket → try (connect/send/receive) → except (handle errors) → finally (close socket). SNMP uses easysnmp Session context—cleanup handled by library.

**Response Time Tracks All Protocols**: Calculate and store last_response_time_ms for ping, HTTP, QUIC, TCP, UDP, SNMP. Useful for performance trending and heartbeat timing adjustments. SNMP response time includes total poll duration (all interface queries + TCP retrans).

**Boolean Handling Uniformity**: Use `to_natural_language_boolean()` everywhere. Never scatter boolean checks. Provides consistent behavior and clear error messages. Handles None gracefully (returns False). Applies to all monitor types.

**PID Lock Cleanup in Finally**: Lockfile removal must be in outermost finally block to handle all exit paths. Never use multiple cleanup locations. Hash-based naming prevents collisions while enabling duplicate detection. Applies regardless of monitor types in config.

Would you like to see the code?