## [1.72] – 2026-05-09

### Critical
- fix: proxy deadlock every ~5min — addon_sig_lock held during 15s HTTP POST; replaced with threading.Event pattern
- fix: VavooProxy() instantiated at module level → background thread on every import. Deferred to start_proxy()
- fix: _restart_proxy() started new server while old serve_forever() still running → "Address already in use"
- fix: log writer thread started at import time → Enigma2 crash. Now lazy-started on first log() call
- fix: log() called ensure_str() defined 350 lines later → NameError at startup
- fix: duplicate getAuthSignature() — second definition silently overwrote first at import

### Security
- fix: SSL CERT_NONE applied globally to all HTTPS. Now disabled only for vavoo.to, kool.to, lokke.app

### vUtils.py
- fix: getUrl() returned magic string "__HTTP451__" on 451 — callers silently treated it as valid data. Now returns b"" with optional raise_on_451=True
- fix: get_country_code() had internal 39-entry map ignoring country_codes (100+ entries). Now uses country_codes directly
- fix: download_flag_online() re-opened file to verify PNG header. Now validates flag_data[:8] in memory
- fix: get_proxy_channels() passed raw bytes to json.loads() — fails on Py2 and Py≤3.5
- fix: is_proxy_ready() / get_proxy_status() swallowed all errors silently with bare except
- fix: redundant from time import inside fix_cache_format() loop

### vavoo_proxy.py
- fix: active_streams counter had no lock → race condition with concurrent streams
- fix: resolve_cache eviction non-deterministic in Py2 (dict unordered). Changed to OrderedDict
- fix: token_monitor_loop slept 60s before first check. Now checks first, then sleeps
- fix: token_monitor_loop slept 120s when streaming — token could expire mid-stream. Now 30s streaming / 60s idle
- fix: ProxyHealthMonitor threshold 550s vs TOKEN_REFRESH_AGE 480s → double-refresh race. Unified
- fix: resolve_cache_ttl=30s too aggressive during streaming. Increased to 300s
- fix: start_proxy() gave up after 3 failures permanently. Now retries indefinitely with exponential backoff
- fix: socket.setdefaulttimeout(30) at module level poisoned all sockets in process. Removed
- fix: Connection:close in session HEADERS disabled keep-alive globally. Removed
- fix: /stream wfile.write(chunk) had no per-chunk exception handling — broken pipe killed handler thread
- fix: do_GET accessed global proxy with no None guard → crash before start_proxy(). Added send_error(503)
- fix: duplicate token_monitor_loop class method (dead code). Removed
- fix: str(max_restarts) NameError after variable was removed from start_proxy()
- fix: _refresh_in_progress logic inverted — every refresh blocked 15s before HTTP. Replaced with Event
- fix: _restart_proxy() called old_proxy.server.shutdown() without None guard
- fix: is_proxy_already_running() had unreachable return False

### plugin.py
- fix: EPG fetched even when disabled — get_current_epg(), show_help_overlay(), showIMDB(), manage_epg_source() all unguarded. All 4 now check cfg.epg_enabled.value
- fix: epgOverlay shown when EPG disabled. Now hidden
- fix: checkInternet() set global socket timeout + leaked socket
- fix: is_port_in_use() used "with socket" context manager — not supported in Py2
- fix: vavoo.cat() compared bytes from getUrl() to str "null" → always True in Py3
- fix: _fallback_to_original_method() passed bytes to json.loads()
- fix: raises() streamed full response body to check reachability. Status code check only now
- fix: get_proxy_stream_url() duplicated from vUtils. Removed
- fix: keep_proxy_alive() infinite loop with no stop → thread leak on reload. Added threading.Event + stop_proxy_monitor()
- fix: global _SKIN_CACHE / _PNG_CACHE declared with unused global statement → F824 lint error. Removed
- perf: _update_proxy_status_display() blocked main thread with getUrl(timeout=5) every 30s. Moved to daemon thread + reactor.callLater()
- perf: show_countries/categories_view() rescanned all_data on every switch. Cached with _invalidate_view_cache()
- perf: flag preloading blocked main thread (8 flags synchronous). All flags now background thread
- perf: EPG XML fetched and ET.fromstring() parsed on every channel nav. Cached per URL, 5min TTL
- perf: _build_channel_list() used append loop. Replaced with list comprehension
- perf: proxy_monitor_timer every 10s → 30s (all 3 locations)
- perf: show_list() called loadPNG() O(n) per rebuild. Added _PNG_CACHE, each icon loaded once

### epg_manager.py (full rewrite)
- perf: 34 sources loaded sequentially. Now parallel with MAX_WORKERS=4 thread pool
- perf: get_current_program() O(n) linear scan. Now O(log n) via bisect_left on sorted list
- perf: normalize_name() recompiled 3 regexes every call. Pre-compiled + _NORM_CACHE dict
- perf: parse_xmltv_date() called strptime every element. Cached in _DATE_CACHE per unique string
- perf: full .gz buffered then decompressed (2× RAM peak). Stream-decompress on the fly
- perf: programs unsorted per channel. Sorted once after load — enables binary search
- perf: get_upcoming_programs() sorted on every call. Now bisect + slice, no sort
- perf: EPGCache.is_valid() read meta JSON from disk every call. Cached in _validity_cache
- fix: class UTC(datetime.tzinfo) → AttributeError in Py2. Fixed: from datetime import tzinfo as _tzinfo
- fix: normalize_name() encoded unicode→bytes then .upper() → mangled non-ASCII in Py2
- fix: gzip.decompress() Py3-only → AttributeError in Py2. Added GzipFile fallback

### bouquet_manager.py
- fix: PLUGIN_PATH assigned twice — first assignment dead code. Removed
- fix: get_local_ip() no timeout on socket.connect() → blocked indefinitely offline. Added settimeout(2)

### __init__.py
- fix: _init_log() bypassed vUtils log system. Now delegates to vUtils.log() with direct-write fallback
- feat: country_codes expanded 16 → 100+ entries (Europe, Americas, Asia, Africa, Oceania + aliases)

### Logging (vUtils.py rewrite)
- perf: file opened/closed on every log() call → async write queue, disk I/O in background thread
- perf: isfile()+getsize() on every write → checked every 50 writes via counter
- fix: only 1 log backup (.1) → 2 backups (.1, .2) with rolling rotation
- fix: no threading lock on writes → concurrent threads interleaved lines. Added _log_lock
- fix: DEBUG_ENABLED evaluated once at import → re-read per call, togglable at runtime
- fix: stdout.write() on every log call flooded Enigma2 kernel log. Now only WARNING+ or VAVOO_DEBUG=1
- fix: log_exception() opened file N times (one per traceback line). Now single batched write
- feat: VAVOO_LOG_LEVEL env var sets minimum log level at runtime (DEBUG/INFO/WARNING/ERROR)
