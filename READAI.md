# APMONITOR.PY v1.2.0 - TCP/UDP MONITORING WITH UNIFIED URL ARCHITECTURE

Hey! Welcome back to APMonitor.py. We just completed implementing TCP/UDP monitoring support with a unified architecture where all connection-oriented protocols (HTTP/QUIC/TCP/UDP) flow through a common `check_url_resource()` function. Let me get you oriented on the current state.

## Purpose

APMonitor monitors network resources (ping/HTTP/HTTPS/QUIC/TCP/UDP) on-premises with guaranteed alert delivery even if the monitoring host fails. Integrates with external heartbeat services for high-availability second-opinion monitoring. Designed for repeated cron/systemd invocation rather than daemon mode.

## What APMonitor Does

- Loads YAML/JSON configuration defining site, monitors (ping/http/quic/tcp/udp), email/webhook notifications, timing parameters
- Validates configuration structure including TCP/UDP-specific rules (scheme validation, port requirements, send/content_type/expect constraints)
- Checks resource availability via ICMP ping, HTTP/HTTPS GET/POST, QUIC/HTTP3 GET/POST, TCP connection/banner, UDP send/receive
- Supports HTTP/QUIC POST requests with configurable MIME types (application/json, application/x-www-form-urlencoded, text/plain)
- Supports TCP/UDP protocol checks with text/hex/base64 encoding for send data and optional expect validation
- TCP monitors automatically receive data after connecting (for banner protocols like SSH/SMTP/FTP)
- UDP monitors require send parameter (connectionless protocol), support fire-and-forget or expect-based validation
- Enforces per-monitor check intervals with site-level defaults and configuration change detection via SHA-256 checksums
- Tracks persistent state in JSON statefile with atomic rotation
- Sends email notifications via SMTP with per-recipient control flags (outages/recoveries/reminders)
- Sends webhook notifications (GET/POST with URL/HTML/JSON/CSVQUOTED encoding)
- Enforces notification throttling with escalating delays via quadratic Bezier curve
- Pings heartbeat URLs when resources are up with configurable intervals
- Validates SSL certificates via SHA-256 fingerprints and expiration checks (HTTP/QUIC only)
- Uses PID lockfiles to prevent duplicate instances per config file
- Runs multi-threaded with explicit stdout flushing for proper log ordering
- Prefixes all thread output with thread ID and site/resource context

## Key Architecture

**Unified URL Resource Architecture**: All connection-oriented protocols (HTTP/QUIC/TCP/UDP) share common entry point `check_url_resource()`. This function handles expect checking universally by extracting `(error_msg, status_code, headers, response_text)` tuples from protocol-specific checkers. HTTP/QUIC return actual status codes (200, 404, etc.), TCP/UDP return 200 for success by convention. Expect logic lives once in `check_url_resource()`, not duplicated per protocol.

**Content-Type Semantic Split**: HTTP/QUIC treat `content_type` as raw MIME type header (e.g., "application/json"), TCP/UDP treat it as wire-level encoding format (text/hex/base64). Both protocols accept `send` parameter but interpret it differently: HTTP/QUIC always UTF-8 encode for POST body, TCP/UDP decode according to content_type before transmission. This semantic split allows natural expression of both use cases.

**TCP Banner Protocol Support**: TCP monitors without `send` parameter perform connection-only checks but always attempt to receive data (via `sock.recv(4096)` wrapped in try/except). This enables monitoring banner protocols (SSH, SMTP, FTP) that send greetings on connect without requiring explicit send. Timeout on receive is not fatal unless expect specified—allows monitoring both interactive and passive TCP services.

**UDP Connectionless Behavior**: UDP requires `send` parameter (validation enforced). Success semantics differ from TCP: without expect, success means `sendto()` didn't raise socket error (packet may be dropped silently—UDP provides no delivery confirmation). With expect, success requires matching response within timeout. This matches UDP's connectionless nature while providing practical monitoring capability.

**Thread-Aware Prefixing**: `prefix_logline(site_name, resource_name)` generates `[T#XXXX Site/Resource]` prefix using `threading.get_native_id()` for unique identification. Prefix created once at start of `check_and_heartbeat()`, stored in thread_local storage, passed to all functions via `thread_local.prefix`. Enables clean log filtering and audit trails without parameter threading bloat.

**Output Buffering Solution**: Python stdout is line-buffered for terminals but fully buffered for pipes. Systemd captures via pipes, causing interleaved output. Solution: explicit `sys.stdout.flush()` after welcome banner, startup messages, and critically inside `update_state()` after every state write. The `update_state()` flush ensures thread output visible atomically with state mutations.

**Stateless Execution with Flush Discipline**: Each invocation loads state, performs checks, flushes output at synchronization points, saves state atomically, exits. Output ordering guaranteed by strategic flush placement.

**Per-Config PID Locking**: Creates `/tmp/apmonitor-{hash}.lock` where hash = SHA256(config_path)[:16]. Prevents duplicate processes per config, allows concurrent monitoring of different sites. Stale lockfile detection via `os.kill(pid, 0)`. Unix-only.

**Thread-Safe State Management**: Global `STATE` dict protected by `STATE_LOCK`. Updates written immediately to `.new` file inside lock, flushed immediately after lock release.

**Configuration Change Detection**: SHA-256 checksum of JSON-serialized monitor config triggers immediate checks when configuration changes, bypassing timing intervals.

**Interval-Based Scheduling**: Each monitor tracks last_checked, last_notified, last_successful_heartbeat timestamps. Decisions made by comparing elapsed time against configured intervals.

**Bezier Curve Notification Escalation**: Notification delays follow quadratic Bezier curve over first N notifications, then plateau. Formula: `t = (1/N) * index`, `delay = (1-t)² * 0 + 2(1-t)t * base + t² * base`.

## Important Modules & Communication

**Protocol Function Signatures** (unified tuple return):
All URL resource checkers return: `(error_msg: Optional[str], status_code: Optional[int], headers: Any, response_text: Optional[str])`
- `check_http_url_resource()`: Returns actual HTTP status code, headers dict, decoded response text
- `check_quic_url_resource()`: Returns HTTP/3 status code, headers dict, decoded response text
- `check_tcp_url_resource()`: Returns 200 for success (HTTP-like convention), empty headers dict {}, received text
- `check_udp_url_resource()`: Returns 200 for success, empty headers dict {}, received text if any
This uniform interface enables `check_url_resource()` to handle expect checking identically across all protocols.

**Check URL Resource Flow** (`check_url_resource`):
1. Extracts url, name, expect, ssl_fingerprint, ignore_ssl_expiry, send_data, content_type from resource dict
2. Converts ignore_ssl_expiry to boolean
3. Dispatches to protocol-specific checker via chained ternary conditional (http → quic → tcp → udp → unknown)
4. Receives `(error_msg, status_code, headers, response_text)` tuple
5. Returns connection error immediately if error_msg not None
6. If expect specified: checks substring match in response_text, returns success or "expected content not found"
7. If no expect: checks status_code == 200, returns success or "error response code"
This centralizes expect logic rather than duplicating across protocols.

**TCP Resource Checker** (`check_tcp_url_resource`):
- Validates `tcp://` scheme and extracts hostname:port
- Creates SOCK_STREAM socket with MAX_TRY_SECS timeout
- Connects via three-way handshake
- If send_data: encodes per content_type (hex/base64/text), sends via sendall()
- Always attempts recv(4096) wrapped in try/except for timeout handling
- Returns (None, 200, {}, response_text) on success
- Returns (error_msg, None, None, None) on timeout or socket error
Key insight: Always attempt receive even without send_data for banner protocol support.

**UDP Resource Checker** (`check_udp_url_resource`):
- Validates `udp://` scheme and extracts hostname:port
- Requires send_data (validation enforced in config validation)
- Creates SOCK_DGRAM socket with MAX_TRY_SECS timeout
- Encodes send_data per content_type (hex/base64/text)
- Sends via sendto() (connectionless—no handshake)
- Always attempts recvfrom(4096) wrapped in try/except
- Receive timeout not fatal if no expect specified (fire-and-forget success)
- Returns (None, 200, {}, response_text) on success
- Returns (error_msg, None, None, None) on socket error
Key insight: UDP success without expect means "packet sent" not "packet received by service."

**HTTP/QUIC POST Support** (existing functions enhanced):
- `check_http_url_resource()`: If send_data, uses requests.post() with UTF-8 encoded body and Content-Type header from content_type parameter or "text/plain; charset=utf-8" default
- `check_quic_url_resource()`: If send_data, sends HTTP/3 POST with `:method` = POST, headers with Content-Type and Content-Length, data via send_data() with end_stream=True
- Both continue to support GET when send_data is None (original behavior preserved)
- content_type is raw MIME type string, not encoding format (semantic difference from TCP/UDP)

**Thread-Local Prefix Storage** (`thread_local`):
- Global `thread_local = threading.local()` with `.prefix` attribute
- Set once at start of `check_and_heartbeat()`: `thread_local.prefix = prefix_logline(site_config['name'], resource['name'])`
- Retrieved via `prefix = getattr(thread_local, 'prefix', '')` in all output functions
- Eliminates prefix parameter threading through all function calls
- Main thread never sets prefix (getattr returns '' default)

**Configuration Validation** (`print_and_exit_on_bad_config`):
TCP/UDP-specific validation rules:
- TCP monitors must use `tcp://` scheme, UDP must use `udp://` scheme
- Address must include hostname/IP and port (validated via urlparse)
- UDP monitors require `send` parameter (enforced with clear error)
- `content_type` can only be specified if `send` present
- `content_type` must be one of: text, hex, base64 (for TCP/UDP)
- `ssl_fingerprint` and `ignore_ssl_expiry` not allowed for TCP/UDP
- `expect` is optional for both TCP and UDP
All validation happens before any monitoring begins—fail-fast on config errors.

**State Management** (`load_state`, `save_state`, `update_state`):
- `load_state`: Reads JSON at startup
- `update_state`: Thread-safe in-memory update + immediate write to `.new` + immediate flush
- `save_state`: Atomic rotation (current → `.old`, `.new` → current)
- Per-monitor state includes: is_up, last_checked, last_response_time_ms, down_count, last_alarm_started, last_notified, last_successful_heartbeat, notified_count, error_reason, last_config_checksum
- Response time tracked for all protocol types (TCP/UDP connection time included)

**Main Orchestration** (`check_and_heartbeat`, `main`):
- `main`: Acquires PID lock, loads config/state, spawns thread pool, waits for completion with explicit result retrieval, flushes output, records execution time, saves state, releases lock
- `check_and_heartbeat`: Creates prefix once at start, stores in thread_local, performs checks, handles state transitions with proper notification types
- Thread pool uses `executor.submit()` to launch, then `future.result()` in sequential loop to wait for ALL threads and propagate exceptions
- All protocols (ping/http/quic/tcp/udp) flow through same orchestration

## Technical Tactics

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
This enables unified expect checking after dispatch while handling protocol-specific connection logic separately.

**Status Code Convention for Non-HTTP Protocols**: TCP/UDP return 200 for success (HTTP-like convention) to enable unified expect checking logic. Alternative would duplicate expect checking across protocols or require protocol-aware branches in `check_url_resource()`. Convention approach cleaner.

**Hex Encoding with Whitespace Stripping**: For hex content_type, clean input via `send_data.replace(' ', '').replace(':', '')` before `bytes.fromhex()`. Allows user-friendly hex specification with spaces/colons (e.g., "01 02 03 04" or "01:02:03:04") while ensuring valid hex parsing.

**TCP Banner Always-Receive Pattern**: Wrap `sock.recv(4096)` in try/except even when send_data not specified. Timeout on receive is logged but not fatal. Enables monitoring banner protocols (SSH, SMTP, FTP) without explicit send configuration. If expect specified but timeout, that IS fatal—expect requires successful receive.

**UDP Fire-and-Forget vs Expect-Based**: Without expect, UDP success means `sendto()` succeeded (packet transmitted). With expect, success requires matching response within timeout. This matches UDP's delivery guarantees—sendto() success doesn't mean packet arrived or service processed it. Practical monitoring requires expect for real validation.

**Thread-Local Storage for Prefix**: Using `threading.local()` eliminates prefix parameter threading through all function calls. Set once in `check_and_heartbeat()`, retrieved via getattr with default in output functions. Main thread never sets prefix (getattr returns empty string), keeping banner/startup output clean. Balances clean logs with implementation simplicity.

**Explicit Flush for Pipe-Captured Output**: Systemd captures stdout/stderr via pipes (fully buffered, not line-buffered). Without explicit flush, thread output accumulates and appears out-of-order. Strategic flush after state updates ensures output visibility aligns with state mutations. The `update_state()` flush is critical—called by every thread, making it natural synchronization point.

**Future Result Retrieval Pattern**: After submitting all jobs to thread pool, loop through futures calling `future.result()`. This blocks until each future completes AND re-raises exceptions from worker threads. Without explicit result retrieval, thread exceptions silently swallowed. Sequential result retrieval ensures proper exception propagation while maintaining execution concurrency.

**Socket Timeout Coordination**: Socket timeout set to MAX_TRY_SECS to ensure operation completes within retry interval. For TCP, connection + send + receive must complete within timeout. For UDP, send + receive must complete. Timeout expiration raises socket.timeout exception caught in try/except block.

**Configuration Change Detection**: JSON-serialize monitor config with `sort_keys=True`, compute SHA-256, store as `last_config_checksum`. Compare on load—mismatch forces immediate check bypassing timing intervals. Detects any field change including protocol parameters.

**Atomic State File Rotation**: Write to `.new`, atomically rename to current. Keep `.old` as backup. Ensures state consistency even with kill -9. State mutations always: update memory → write .new → flush → rotate on exit.

**SSL Certificate Pinning**: Fetch cert, convert PEM→DER, compute SHA-256, compare to configured fingerprint. Works for both HTTP and QUIC. Enables trust of self-signed certs. Not applicable to TCP/UDP (no TLS layer at monitoring level).

**QUIC/HTTP3 Implementation**: Async function using aioquic. Custom protocol class handles HTTP/3 events. Wraps in `asyncio.run()` for sync interface. Timeout via `asyncio.timeout()`. Peer certificate from TLS layer. POST support via send_headers() with end_stream=False then send_data() with end_stream=True.

## Engineering Principles for This Code

**Protocol-Specific Checkers Return Uniform Tuples**: All URL resource checkers must return `(error_msg, status_code, headers, response_text)` to enable unified expect checking. Never make `check_url_resource()` protocol-aware—keep branching logic minimal by enforcing tuple contract.

**Status Codes Are HTTP-Like Conventions**: TCP/UDP return 200 for success even though they're not HTTP. This enables unified status checking rather than protocol-specific branches. Convention beats complexity when interfaces are shared.

**Content-Type Semantics Are Protocol-Specific**: HTTP/QUIC treat content_type as MIME type header, TCP/UDP treat as encoding format. Document this clearly but don't try to unify—different protocols have different natural interpretations. Split semantics better than forced consistency.

**Always-Receive for Banner Protocols**: TCP should always attempt receive even without send_data. Many protocols (SSH, SMTP, FTP, POP3, IMAP) send banners on connect. Timeout on receive is logged but not fatal unless expect specified. This enables monitoring without protocol-specific knowledge.

**UDP Requires Send Parameter**: Enforce this in validation. UDP is connectionless—can't verify service listening without application-layer data exchange. Fire-and-forget (no expect) is allowed but documented as "packet sent" not "service responding."

**Thread-Local for Prefix Eliminates Parameter Threading**: Using `threading.local()` is cleaner than passing prefix parameter through every function. Set once, retrieve many. Main thread never sets prefix (clean banner/startup output). All thread output functions use `getattr(thread_local, 'prefix', '')` pattern.

**Flush at Synchronization Points**: Flush after banner, after config messages, inside `update_state()`, and in finally block. The `update_state()` flush is key—called by every thread after work completion, making it natural synchronization point for output visibility.

**Prefix ALL Thread Output**: Every print/stderr from worker threads must include prefix. Main thread output (banner, startup, execution time) stays unprefixed for clarity. Enables clean log filtering and audit trails without log line ambiguity.

**Thread Pool Exception Handling**: Always call `future.result()` on all futures to propagate exceptions. Without explicit result retrieval, thread exceptions silently lost. Wrap each result retrieval in try/except to handle gracefully rather than crashing main thread.

**State Transitions Require Prefix**: Outage and recovery messages include prefix for audit trails. Makes clear which thread detected which state transition. Critical for debugging timing issues or understanding concurrent execution.

**Validate Before Execute**: Config validation must be comprehensive and fail-fast. Better to exit with clear error than silently ignore invalid config. Validate types, formats, constraints, cross-field dependencies, protocol-specific rules.

**Hex/Base64 Encoding Is Convenience**: Allow user-friendly hex input with spaces/colons, strip before parsing. This is about human readability in config files—internal representation is always bytes. Don't enforce strict hex format in config.

**Socket Cleanup in Finally**: Always close sockets in finally block even if exception raised. Resource leaks bad. Pattern: create socket → try (connect/send/receive) → except (handle errors) → finally (close socket).

**Response Time Tracks All Protocols**: Calculate and store last_response_time_ms for ping, HTTP, QUIC, TCP, UDP. Useful for performance trending and heartbeat timing adjustments (adjust next heartbeat check time by last response time to account for check duration).

**Boolean Handling Uniformity**: Use `to_natural_language_boolean()` everywhere. Never scatter boolean checks. Provides consistent behavior and clear error messages. Handles None gracefully (returns False).

**PID Lock Cleanup in Finally**: Lockfile removal must be in outermost finally block to handle all exit paths. Never use multiple cleanup locations. Hash-based naming prevents collisions while enabling duplicate detection.

Would you like to see the code?