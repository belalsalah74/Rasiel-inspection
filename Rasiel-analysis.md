# Rasiel reverse-engineering notes — source archive update

**Updated:** 2026-11-13  
**New input:** `sources.tar.gz` supplied in this conversation  
**Scope:** Decompiled Java source for `com.holomod.MenuHost`, `com.holomod.GoogleWebAuth`, and 177 obfuscated `com.holomod.pl` classes, cross-checked against the previously supplied `menu.dex` and `libholomod.so` where applicable.

> Evidence labels used below: **Confirmed in Java** means the behavior is visible in a decompiled method body. **Confirmed in native interface** means a Java declaration/call and native registration exist, but the native implementation may still be opaque. **Inference** means the evidence suggests a purpose but does not prove it. JADX-decompiled methods marked “not decompiled” are treated as unknown, not reconstructed by guesswork.

## 32. Source archive inventory and scope

The archive contains 179 Java source files:

- `com/holomod/MenuHost.java`
- `com/holomod/GoogleWebAuth.java`
- 177 classes under `com/holomod/pl/`

The package is heavily obfuscated by class names (`pl####`) but not uniformly opaque: method bodies, string constants, Android APIs, preference keys, and native declarations remain useful. The archive does not contain the native implementation source for `libholomod.so`; native behavior still requires ARM64 disassembly and runtime validation.

## 33. Account profiles: encrypted snapshots of the game's secret-manager files

**Confirmed in Java — `pl2756`.** The account manager stores named profiles below an app-private directory whose name is decoded at runtime. Each profile has a `.meta` file with fields including:

- `name`
- `id`
- `group`
- `provider`
- `ppk`
- `ppv`
- `last`

The profile metadata is parsed into a `pl0517` object. The manager also keeps account labels, groups, collapsed-group UI state, and last-login timestamps.

The account payload itself is not just the metadata. Rasiel copies files from the game's `nia-client-secrets-manager` directory. The copy path uses `GoogleWebAuth` encryption/decryption helpers. A device-protected-storage copy of that directory is also handled by the account manager.

The relevant operations are:

- `R(profile)`: restore the selected profile's files into the game's `nia-client-secrets-manager` directory, then restore provider-related state.
- `S(profile)`: save the current secret-manager files into a named profile, while preserving a device-protected-storage copy under `_de`.
- `V(profile)`: restore saved provider/credential preferences and call `nativeSetSwitchProvider(provider)`.
- `U()`: read the account PID key from `nativeAuthPidKey()`, falling back to the saved `acctppkey` preference.

This confirms that account switching involves copying/restoring game authentication state, not merely changing an email label or selecting a saved OAuth token.

## 34. Account switching deliberately terminates sibling processes under the same UID

**Confirmed in Java — `pl2756.A()` and `pl2756.Y(String)`.** The process cleanup routine obtains `ActivityManager.getRunningAppProcesses()`, filters for processes with the current app UID, and calls `Process.killProcess(pid)` for every other matching PID.

`Y(profile)` invokes this cleanup during account switching, after an initial profile restore and a check of `MenuHost.b0(...)`. It then restores the profile again, writes the selected profile name to the `acctrelaunch` preference, and posts a delayed task to continue the relaunch/switch process.

This is a significant side effect: account switching may terminate every other process sharing the app UID, not just a specifically named Pokémon GO service. The method catches broad exceptions, so failure of the process enumeration/termination does not necessarily abort the rest of the switch sequence.

## 35. Feed browser: saved filters, mini-feeds, and category model

**Confirmed in Java — `pl0394`.** The feed browser owns:

- 13 category filter slots (`pl1744[13]`)
- 2 mini-feed slots (`pl1744[2]`)
- a saved-filter list
- a fixed two-thread executor for background work
- a main-thread `Handler`
- a `LinkedHashMap` result/cache structure

Saved filters are restored from the `feedBrowserSaved` JSON preference, with a maximum of 100 entries. Mini-feed settings are restored from `feedBrowserMini`, with two slots. The constructor migrates an older mini-feed label, `Raid eggs`, to `Raids`.

If the newer saved-filter preference does not exist, the class migrates the older semicolon/pipe-delimited `feeds` preference into the new filter representation.

The saved-filter screen includes a `Nearby Pokémon` filter and allows each saved filter to be edited or deleted. The feed browser is therefore a persistent filter manager, not only a transient result list.

## 36. Feed categories and result data model

**Confirmed in Java — `pl2669`.** The category labels are defined centrally as:

1. Pokémon
2. Raids
3. Quests
4. Rocket
5. Lures
6. Nests
7. Showcases
8. Kecleon
9. Max Battles
10. Weather
11. Hotspots
12. Trending
13. Golden stops

Raid tiers are separately represented as `All tiers`, `T1` through `T6`, and `Gigantamax`.

The common feed object is `pl1562`. The quest parser reads a `quests` array and converts records into this shared object. It captures location, name, reward text, condition text, reward type/ID/amount tuples, and constructs a stable quest key from coordinates and reward data.

The source includes raid-activity labels based on reported player counts. It distinguishes “active raiding area; player count not supplied” from explicit reported counts.

## 37. Feed ordering and expiry semantics

**Confirmed in Java — `pl2669`.** The shared comparator `i(...)` supports multiple sort modes. It compares expiry timestamps or another time field depending on the filter's sort configuration, then uses geographic distance as a tie-breaker.

The expiry helper treats Pokémon and Max Battle-like categories specially: records with no positive expiry can be treated as non-expiring, while active/egg state is considered separately. The exact complete behavior of every feed filter remains uncertain because `pl1744.b(...)` and `pl2669.v(...)` are JADX-undecoded in the supplied source.

## 38. R8 asset package format and encrypted update pipeline

**Confirmed in Java — `pl2669`.** The built-in `r8/` asset directory is packed into a custom binary container:

- Magic: `R8P1`
- Entry count: little-endian integer at offset 4
- Per-entry table: 12 bytes
- Entry name: reduced to a 32-bit FNV-1a-style hash
- Entry data: PNG bytes concatenated after the table

Entries are sorted by their name hash before serialization. The packed byte array is passed to `nativeR8Pack(byte[])`.

A downloaded package is fetched with `X-T` and `X-S` headers. `X-T` is a timestamp and `X-S` is an HMAC-SHA256 signature over the `r8|<timestamp>` string. The downloaded bytes must match a hardcoded SHA-256 digest before they are written to `pc.dat` through a temporary file.

The package uses AES-GCM. The decryption key is assembled from a derived 16-byte segment and the first 16 bytes returned by `nativeR8K()`. The IV is read from the first 12 bytes of the downloaded payload. The decrypted bytes are then supplied to the native packer.

The code starts the operation on a background thread named `r8-pack` and guards against concurrent or repeated starts with static state flags.

**Unknown:** the full semantic content of the decrypted package and the native interpretation of the `R8P1` container. The Java source proves the packaging and handoff, not what the native side does with every resource.

## 39. A native crash/tombstone parser is present

**Confirmed in Java — `pl2669.E(...)` and its protobuf-wire parser helpers.** The class contains a low-level parser for a structured binary message. It walks protobuf-style wire fields and formats selected crash details, including:

- signal name/number
- fault address
- abort message
- thread name
- selected thread/backtrace-related strings

This is distinct from the R8 asset package. It indicates Rasiel includes crash/tombstone diagnostic parsing logic. The source inspected here does not establish whether the formatted report is uploaded, stored, or only displayed/logged.

## 40. GPS callback injection — exact guard and synchronization behavior

**Confirmed in Java — `MenuHost.z3()`.** Location injection is skipped when spoofing is disabled, when the re-entrancy flag is set, or when the game location-manager object is unavailable.

The method lazily resolves `nativeLocationUpdate(String, Location, int, Context)` by reflection, constructs/reuses a `Location`, wraps longitude into the conventional `[-180, 180)` range, writes latitude/longitude, sets accuracy to 5 m, altitude to 10 m, and refreshes both time values.

It searches the class hierarchy for `callbackLock`. If found, it synchronizes on that object before invoking the game's callback with provider `n5`, update type `3`, and the selected context. It tracks uptime and sets a re-entrancy guard around the invocation.

This confirms direct invocation of the game's location update callback. It does not, by itself, establish whether Niantic server-side location validation is bypassed.

## 41. Hunt catch verification and wrong-species recovery

**Confirmed in Java — `MenuHost.statsUpdate(...)`.** When the hunt is in its catch stage and a catch completes, Rasiel queries `nativeLastCaughtDex()`. It compares the returned dex number with the expected target dex (`H2`). If the result is absent or matches, it proceeds with the normal completion handler. If it differs, Rasiel reports that Auto Catch caught the wrong dex and continues waiting for the intended target.

This is an explicit post-catch verification step. It is stronger evidence than a UI success message or a native method name alone.

## 42. Hunt stop routine resets native flags

**Confirmed in Java — `MenuHost.z5(boolean)`.** Stopping the hunt:

1. Calls `nativeSetHuntActive(false)`.
2. Clears Java hunt state (`V1`).
3. Removes the scheduled hunt callback.
4. Calls `nativeSetBlockNonShiny(false)`.
5. Runs Java cleanup and restores relevant UI controls.
6. Chooses one of two follow-up UI/state paths depending on the boolean argument.

This confirms Block Non-Shiny is reset when the hunt is stopped, rather than being left enabled independently.

## 43. Route persistence and route execution

**Confirmed in Java — `MenuHost.x4()` and `MenuHost.x5(...)`.** Routes are serialized into the `routes` preference. Each route stores its name and a sequence of coordinate pairs. Route execution cancels the previous route callback, stores the selected route and mode flags, resets route indices/state, enables spoofing, updates the GPS UI checkbox, and schedules the route handler.

The route utility `pl0358` includes GPX parsing/export, coordinate parsing, coordinate validation, distance and projection helpers, preset locations, and time-zone/geocoding-related parsing. The exact behavior of every route movement mode still needs full callback tracing.

## 44. Corrections and caution against overclaiming

- The Java archive is 179 files total, not proof that all 177 obfuscated classes implement distinct user-visible features.
- The native registration count is 81, but the source/native interface must be reconciled method-by-method; the short `z62*`, `z66a`, and `z70*` methods are real registered entries.
- `nativeSetChores`'s signature and caller are confirmed. Do not claim a meaning for `eggLong` beyond the Java variable/setting name without tracing its native consumer.
- `nativeSetEncRange` changes a native range setting. The name “Spawn Booster” does not prove increased server spawn generation.
- The encrypted R8 package is verified and decrypted by Java, but its semantic contents and native use remain unknown.
- The presence of native IL2CPP APIs proves runtime reflection/invocation capability; it does not automatically prove every feature uses hooks or detours.
- The tombstone parser does not prove crash data is exfiltrated.

## 45. Next reverse-engineering priorities

1. Reconcile all 81 `RegisterNatives` descriptors against `MenuHost` declarations, including the obfuscated methods.
2. Recover the IL2CPP resolver's string arguments and call sites, then identify actual Pokémon GO classes/methods used by each scheduler.
3. Trace `nativeNearbySnapshot`, `nativePoiSnapshot`, `nativeMapProbe`, and `nativeArmDirect` return/argument semantics.
4. Trace native encounter discovery, shiny verification, and encounter-ID lifecycle.
5. Trace battle automation and the two UI settings concerning enemy attacks/dodging.
6. Recover the complete feed predicate and `pl2669.v(...)` parser from DEX bytecode/smali because JADX omitted their bodies.
7. Inspect the R8 decrypted package format and native consumer.
8. Validate account profile file handling and relaunch sequence against a controlled test environment.


## 46. Privacy audit: periodic heartbeat telemetry

**Confirmed Java call path — `MenuHost.l0()` and `MenuHost.r5()`.** When the overlay is active (`N0 == true`) and the last heartbeat is at least 30 minutes old (or the clock moves backwards), Rasiel updates the `st_beat` timestamp in its `holomod_stats` SharedPreferences and starts a background `pl0911` task with filename `s.json` and a JSON payload.

The payload is constructed as:

```json
{"i":"<stable-device-hash>","v":"<payloadVersion>"}
```

`i` is calculated by reading `Settings.Secure.ANDROID_ID`, concatenating the hardcoded suffix `|rasiel-st-1`, hashing the result with SHA-256, and encoding it as lowercase hexadecimal. The raw Android ID is not included in this JSON. The hash is deterministic for the same Android ID and suffix, so it is a persistent pseudonymous installation identifier, not anonymous data.

`v` is extracted from `files/p/manifest.json` using the `payloadVersion` field. It is the payload version, not the Rasiel application version as previously described. The sender task is started on a background thread. The `pl0911.run()` body was not decompiled in the supplied Java source, so the exact destination URL, HTTP method, headers, response handling, and server-side retention are not established by this source alone. The task name/filename and payload construction are strong evidence of a heartbeat telemetry path, but the final network transmission needs bytecode or runtime confirmation.

## 47. Catch/hunt event telemetry

**Confirmed payload construction — `MenuHost.l5(int, String)`.** The code creates a second background `pl0911` task, using filename `n.json` and this JSON structure:

```json
{"i":"<stable-device-hash>","s":"<pokemon-species-name>","t":"<local-time>"}
```

The fields are:

- `i`: the same SHA-256-derived Android ID pseudonym described above.
- `s`: Pokémon species name, obtained from the dex number.
- `t`: local device time formatted as `h:mm a`.

The JSON helper strips characters outside letters, digits, `#`, colon, dot and spaces from values. The event is triggered by both the hunt completion path (`l5(..., "hunt")`) and the catch callback (`onCaught(...)` / `l5(..., "catch")`). The supplied `str` argument (`hunt` or `catch`) is not included in the JSON built by `l5`; the overloaded `pl0911` constructor stores only the filename and payload in its visible fields.

The sender is rate-limited: a repeated dex number is suppressed for five minutes, while a different dex can pass immediately. The exact remote endpoint and whether this task always transmits successfully remain unverified because `pl0911.run()` is missing from the decompiled source. The constructor call's diagnostic text calls this a `SHUNDO: webhook`, but the event triggers shown here include ordinary hunt/catch paths; it should not be described as a Shundo-only event.

## 48. Location data sent to third-party services

The source contains explicit location-bearing requests:

1. **iFlowGo nearby endpoint** (`pl2412.f(...)`):
   `https://pokecoords.iflowgo.com/iflowgopokecoords/api/v1/nearby?lat=<latitude>&lon=<longitude>&radius_km=2&layers=spawns&limit=100`
   This sends the supplied latitude and longitude in the URL query, with a 2 km radius and spawn layer. The source does not itself prove whether the coordinates at every call site are real GPS or spoofed coordinates; that depends on the array passed to this method.
2. **Open-Meteo forecast** (`MenuHost.s(double,double)`):
   `https://api.open-meteo.com/v1/forecast?latitude=<lat to 4 decimals>&longitude=<lon to 4 decimals>&timezone=auto&forecast_days=1`
   This sends coordinates to Open-Meteo to obtain forecast/time-zone information. The method caches the returned timezone by a coordinate-derived key.
3. **OpenStreetMap tiles** (`pl0937.run()`): requests use `https://tile.openstreetmap.org/{z}/{x}/{y}.png`. Tile z/x/y values identify map tiles and therefore disclose an approximate viewed map area to the tile server. The code sets a `MapBoard/1.0 (Android overlay map; contact: mapboard-support)` User-Agent.
4. **Nominatim geocoding**: the source contains `https://nominatim.openstreetmap.org/search?format=json&limit=8&q=`. This sends the user's typed place/search query to Nominatim. It is a text query, not proof that the user's exact GPS position is sent in that request.

## 49. Feed providers and information in requests

The source also makes requests to:

- `pokemongocoordinates.com/api/{feed|raids|quests|grunts|lures|nests}`. The Pokémon feed can include `min_iv` and species-name filters; raid requests can include raid level and active status; quest requests can include quest type/reward; grunt requests can include character; lure requests can include lure ID. The inspected `pl0634` code does not append the player's latitude/longitude to these URLs.
- `pokecoords.iflowgo.com/.../pokemon-search` for IV-sorted Pokémon search. The full call construction is in `pl2412.c(...)`, whose decompiled body is missing, so its complete query fields require bytecode recovery.
- `moonani.com/PokeList/` for a Pokémon table/search dataset. The visible request path is a supplied table/page string; the shown `pl1070` implementation does not append coordinates.
- Google OAuth endpoints for interactive authorization and token exchange. Those requests necessarily carry OAuth protocol material (authorization code, PKCE verifier, refresh token when refreshing) to Google. This is authentication traffic, not evidence that Rasiel sends those credentials to its own telemetry endpoint.

## 50. Local log access and export

**Confirmed Java behavior — `MenuHost.P2(...)`, `MenuHost.b(boolean)`, `g3(...)`, and `h3(...)`.** The Logs tab reads Pokémon GO's own `sys.log`, preferring `/sdcard/Android/data/com.nianticlabs.pokemongo/files/sys.log` when accessible and falling back to the game's internal files directory. Rasiel maintains up to three session generations by rotating the current file to `.1.log` and `.2.log` when its startup logic detects a new session.

The UI lets the user choose the current session, previous session, or two sessions ago and explicitly export the selected log. It can save to the Downloads collection or invoke Android's document-creation picker. This is a user-initiated local export path in the inspected Java code; it is not evidence that the log is automatically uploaded.

The log may contain game-generated diagnostic data. Its exact contents were not available in this source archive, so do not assume it contains (or does not contain) account identifiers, location, network details, or other personal data without inspecting an actual `sys.log` sample.

## 51. Crash handling versus crash exfiltration

The default uncaught-exception handler (`pl0007`) obtains `thread.getName()` and calls `Log.getStackTraceString(th)`, then forwards the exception to the previous handler. It does not write the stack trace to a Rasiel file or initiate a network request in the inspected handler.

The source also contains a tombstone/diagnostic parser and the native library includes strings referencing `/dev/socket/logd`, `/proc/self/maps`, `/proc/self/status`, and Pokémon GO's `sys.log`. These are indicators of diagnostic/process inspection capabilities. They do not establish that Rasiel uploads full Android logcat, tombstones, or the Pokémon GO log. The native references need call-site/xref analysis before making that claim.

## 52. Local storage of account-related data

The account manager uses two distinct local stores:

- `nia-client-secrets-manager` under the app's device-protected files directory for the game's account secret files. The Google login implementation encrypts its refresh-token data using Android Keystore-backed AES-GCM.
- `rasiellog` under the app's private files directory for per-account `.meta` files. Despite the name, this is an account metadata directory, not the game log directory. `pl0517.b()` serializes fields `name`, `id`, `group`, `provider`, `ppk`, `ppv`, and `last` as plaintext UTF-8 metadata. `ppk` is populated from `nativeAuthPidKey()` and `ppv` is the associated preference value; their exact semantic contents depend on the game's auth provider and should be treated as potentially sensitive account-state data. The metadata file is stored in the app-private sandbox, but it is not encrypted by `j0()` in the inspected code.

The app also stores a large amount of user configuration in the `holomod_stats` SharedPreferences file, including spoof coordinates (`lat`, `lng`), route definitions, hunt settings, profile settings, and automation preferences. These are local preferences; their presence is not evidence of remote transmission.

## 53. What the source does not show

In the supplied Java source, I found no use of APIs for reading contacts, SMS, call history, IMEI, phone number, advertising ID, or installed-app inventories. I also found no Firebase Crashlytics, Sentry, Mixpanel, Amplitude, or similar named analytics SDK in this overlay source.

This is a bounded negative finding: the source archive covers the overlay Java code, not every library inside the APK or every native operation. It does not rule out collection by embedded dependencies, the native library, the modified Pokémon GO client, or remote services. The app's manifest and all packaged dependency code still need a separate permissions/SDK audit.

## 54. Privacy risk assessment

- **High confidence, persistent identifier:** SHA-256 hash of Android ID with fixed salt is constructed for heartbeat and catch/hunt event payloads.
- **High confidence, location disclosure to external services:** coordinates are explicitly placed in iFlowGo nearby and Open-Meteo requests; map tile requests disclose approximate viewed areas.
- **High confidence, game log access:** Rasiel reads Pokémon GO `sys.log` and allows user-directed export. Automatic upload is not established.
- **High confidence, local account metadata:** account names/IDs/provider and auth-preference key/value are stored in private app files; `.meta` serialization is plaintext within the app sandbox.
- **Unresolved, remote telemetry destination:** `pl0911.run()` is missing from the JADX output. Need DEX bytecode/smali or runtime network capture to identify the endpoint and verify actual transmission.
- **Unresolved, native diagnostics:** `/dev/socket/logd` and `sys.log` strings in the native library need xref/call-site analysis to determine whether they are opened/read/written and whether anything leaves the device.

## 55. Corrections to earlier wording

- The heartbeat's `v` field is `payloadVersion` parsed from `p/manifest.json`, not the Rasiel app version.
- The directory named `rasiellog` stores account `.meta` files; it is not the Logs tab's `sys.log` storage.
- The `SHUNDO: webhook` label is diagnostic text supplied to the sender constructor. The event-building call sites include both hunt and catch events, so it is not established as Shundo-only.
- The Java uncaught-exception handler obtains a stack-trace string but does not itself persist or upload it.
- The presence of telemetry payload construction and background sender tasks strongly indicates intended telemetry, but the sender's missing method body prevents a definitive statement about the final endpoint and successful network transmission.

## 56. Next privacy-analysis steps

1. **Completed:** recovered `pl0911.run()` from `menu.dex`; the telemetry host, POST method, headers, timeouts, and response-code call are documented in sections 57–58. Runtime success and server-side handling remain to be verified.
2. Trace native references to `/dev/socket/logd` and `/sdcard/Android/data/com.nianticlabs.pokemongo/files/sys.log` to determine whether native code reads or writes logs.
3. Inspect the Rasiel manifest and packaged dependencies for permissions, SDKs, services, receivers, and network-security configuration.
4. Recover `pl2412.c(...)` to enumerate all iFlowGo search parameters and whether it sends additional identifiers or coordinates.
5. Inspect actual runtime traffic with a controlled test account and synthetic coordinates; compare heartbeat, catch/hunt, feed, weather, geocoding, and tile requests separately.

## 57. Recovered telemetry transport: `mklister.com`

**Confirmed from the `pl0911.run()` DEX bytecode in `menu.dex` (the method JADX could not decompile).** The telemetry sender builds its URL by concatenating the decoded `MenuHost.cc` constant with the filename stored in `pl0911.b`. `MenuHost.cc` decodes to:

`https://mklister.com/assets/st/`

The two observed caller filenames therefore resolve to:

- `s.json` — heartbeat payload: `https://mklister.com/assets/st/s.json`
- `n.json` — catch/hunt event payload: `https://mklister.com/assets/st/n.json`

The bytecode performs the following transport operations:

1. Constructs a `java.net.URL` from the base plus filename and opens an `HttpURLConnection`.
2. Calls `setRequestMethod("POST")` and `setDoOutput(true)`.
3. Sets `Content-Type: application/json`.
4. Sets `User-Agent: Mozilla/5.0 (Linux; Android 13) AppleWebKit/537.36`.
5. Sets both connect and read timeouts to 12,000 ms.
6. Gets the output stream, converts the payload string to UTF-8 bytes, writes it, and closes the stream.
7. Calls `getResponseCode()` and then `disconnect()`.

The recovered method does not show an Authorization header, API key, cookie, or additional identifier being added by this sender. It also does not read or parse the response body in this branch. The response code is obtained, but the inspected bytecode alone does not establish what status is accepted or whether the server stores the payload.

**Privacy significance:** the earlier heartbeat and catch/hunt payload findings are no longer merely queued-task evidence. The DEX contains a concrete outbound POST path to a fixed third-party host. The heartbeat contains the stable Android-ID-derived hash and `payloadVersion`; the event payload contains the same hash, species name, and local time. This makes `mklister.com` the confirmed destination configured for those payloads. A live capture or server-side evidence would still be needed to prove that a particular run completed successfully and that the server retained or processed the data.

**Implementation detail:** the four-argument `pl0911(String, String, boolean, String)` constructor used by the telemetry callers stores only the filename and JSON body in fields `b` and `c`; the boolean and diagnostic label arguments are not retained by that constructor. The network branch therefore uses the same transport for both JSON filenames rather than embedding the caller's diagnostic label in the request.

## 58. Updated telemetry assessment

- **Confirmed configured endpoint:** `https://mklister.com/assets/st/s.json` and `https://mklister.com/assets/st/n.json`.
- **Confirmed request type and headers:** HTTP POST, JSON content type, Android browser-like User-Agent.
- **Confirmed transport behavior:** UTF-8 request body, 12-second connect/read timeouts, response-code retrieval, connection disconnect.
- **Still unverified:** whether the server returns success for actual user sessions, server-side retention/processing, TLS certificate behavior at runtime, and any downstream sharing by the endpoint operator.
- **Still separate from this sender:** the iFlowGo, Open-Meteo, OpenStreetMap/Nominatim, and Pokémon Go Coordinates feed requests described above.

This materially upgrades the earlier conclusion: Rasiel's Java/Dex layer has an explicit outbound telemetry implementation and a hard-coded telemetry host. It does not, by itself, prove the operator's identity, the host's ownership, or what happens to data after receipt.

## 59. Native diagnostic strings: what can and cannot be concluded

**Confirmed string presence in `libholomod.so`.** The stripped ARM64 library contains these exact path strings:

- `/dev/socket/logd`
- `/proc/self/status`
- `/proc/self/maps`
- `/sdcard/Android/data/com.nianticlabs.pokemongo/files/sys.log`
- `/data/data/com.nianticlabs.pokemongo/files/sys.log`
- `/sdcard/Android/data/com.nianticlabs.pokemongo/files/.gtap`
- `/data/data/com.nianticlabs.pokemongo/files/.gtap`

The dynamic symbol table exposes only `JNI_OnLoad`; the library is stripped, so ordinary symbol-name inspection does not reveal the functions that reference these strings. The strings are consistent with native process inspection and access to game diagnostic/log files, but string presence alone does not prove that any file is opened, read, modified, or transmitted.

**Current status:** unresolved. A defensible next step is to locate the string references in the ELF `.rodata` section, identify the ARM64 instructions that materialize their addresses, and follow the surrounding control flow to `open/openat/read/write`, Android logging, or networking functions. Where static control flow is ambiguous, confirm with a controlled runtime trace. Do not classify `.gtap` as a log or upload artifact until its read/write call sites and format are understood.

## 60. APK manifest and dependency audit: inventory result and limitation

The supplied `Rasiel.apk` is a large APK containing three DEX files (`classes.dex`, `classes2.dex`, `classes3.dex`), a binary Android manifest, resources, and native libraries. The manifest is Android binary XML, not plain text. The current analysis environment has no `aapt`, `aapt2`, `apkanalyzer`, `apktool`, or JADX executable available to decode it directly. Therefore, this pass does **not** claim a verified permission list, component list, exported-component status, network security configuration, or complete SDK inventory.

The earlier statement that the overlay Java source did not show contact/SMS/call-history/IMEI/phone-number/advertising-ID/installed-app collection remains bounded to the inspected Java source. It must not be generalized to the complete APK until all three DEX files, manifest, bundled libraries, and native code have been audited.

## 61. Repository publication status

The requested GitHub repository `belalsalah74/Rasiel-inspection` exists but was empty when inspected. The repository metadata was accessible, while the initial Contents API request returned 404 with GitHub's explicit message that the repository is empty. The report can therefore be published as a new file rather than updating an existing tracked report. This section records the state before publication; it is not itself proof that a commit was successfully created.

## 62. Current evidence ledger and remaining work

### Confirmed from supplied Java/Dex

- Overlay source archive inventory and account-profile behavior (sections 32–34).
- Feed categories, saved filters, feed model, sorting/expiry caveats (sections 35–37).
- R8 resource packing and encrypted update pipeline (section 38).
- Crash/tombstone parsing logic (section 39).
- GPS callback injection details (section 40).
- Hunt catch verification and stop/reset behavior (sections 41–42).
- Route persistence and execution setup (section 43).
- Heartbeat and catch/hunt telemetry payloads, including a stable Android-ID-derived pseudonym (sections 46–47).
- Explicit location-bearing requests to iFlowGo and Open-Meteo, map-tile requests, and Nominatim text search (section 48).
- Feed-provider request paths and locally read/exported game logs (sections 49–50).
- The `pl0911.run()` POST transport to `mklister.com`, including endpoint paths, headers, and timeouts (sections 57–58).

### Confirmed only as native interface/string evidence

- The JNI/native interface exposes extensive IL2CPP introspection and invocation capability; exact behavior must be established per native call site.
- The native library contains diagnostic/process/log path strings (section 59), but their actual use is not yet proven.

### Still unresolved

1. Reconcile all 81 JNI registrations with Java declarations and native implementations.
2. Disassemble native xrefs to the `logd`, `sys.log`, `.gtap`, `/proc/self/maps`, and `/proc/self/status` strings.
3. Decode the APK manifest and audit all permissions, activities, services, receivers, providers, exported components, and network security settings.
4. Inventory bundled SDKs and all three DEX files for additional analytics, crash reporting, and network clients.
5. Recover the complete iFlowGo `pokemon-search` request builder and determine all transmitted parameters.
6. Trace native encounter discovery, shiny verification, encounter IDs, battle automation, and the enemy attack/dodge controls.
7. Validate the telemetry requests in a controlled runtime capture and determine whether the endpoint returns success; server retention and downstream use remain unknown.
8. Inspect actual `sys.log` and `.gtap` samples before making claims about their contents.

## 63. Historical section numbering note

The working Markdown artifact available for this continuation begins at section 32. The original text of sections 1–31 was not present in the supplied working file or the searchable Library copy; those sections cannot be reproduced verbatim from the evidence currently available. They are intentionally not fabricated or silently replaced with guessed prose. Sections 32 onward are preserved here, followed by the new sections 59–63.