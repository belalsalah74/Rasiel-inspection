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

## 64. Native xrefs to `.gtap`, `sys.log`, `/proc`, and `logd`

**New static-disassembly result — `libholomod.so` (ARM64).** The native library is stripped, but its `.rodata` strings can be mapped to instruction references in `.text`. The disassembly now gives more than string-presence evidence for several paths.

### `.gtap`: concrete filesystem syscall path

The two `.gtap` paths are referenced at native code addresses `0x24538`, `0x24558`, `0x56eb8`, `0x56ed4`, `0x56ef0`, and `0x56f0c`:

- `/sdcard/Android/data/com.nianticlabs.pokemongo/files/.gtap`
- `/data/data/com.nianticlabs.pokemongo/files/.gtap`

The surrounding code loads the path into `x1` and issues AArch64 Linux syscalls directly with `svc #0`. The syscall numbers visible in the code include `48` (`faccessat`), `56` (`openat`), `65` (`read`), and `57` (`close`). This is consistent with checking for and opening/reading the `.gtap` file, with fallback between external and internal game paths. The code also contains a read operation after opening the path. This is materially stronger than a string-only observation: **native code is configured to access and read `.gtap`**, although the exact data format and meaning remain unknown.

The relevant sequence includes:

- `0x24538`: `faccessat`-style check using the external `.gtap` path.
- `0x24558`: `openat`-style attempt using the internal `.gtap` path after the first path check/failure branch.
- `0x56eb8` onward: repeated path checks/open attempts, followed by a `read` and `close` path.

The precise control-flow conditions and the consumer of the bytes still need a complete function-level trace. Do not call `.gtap` a telemetry file or assume it contains logs; the evidence only establishes file access/read behavior.

### `sys.log`: paths are referenced during native initialization

The two Pokémon GO `sys.log` strings are referenced at `0x21310` and `0x2132c` in the `JNI_OnLoad` region:

- `/sdcard/Android/data/com.nianticlabs.pokemongo/files/sys.log`
- `/data/data/com.nianticlabs.pokemongo/files/sys.log`

The references are passed into a native helper path that performs direct syscalls. The nearby startup code also contains calls to a logging helper. This establishes that the native initialization path uses the game-log path constants, but the current trace does not yet conclusively identify whether it reads, tails, truncates, or writes the log, nor whether log contents are sent anywhere. Further tracing of the helper and syscall arguments is required before describing the operation as log collection or upload.

### `/proc/self/status` and `/proc/self/maps`

- `/proc/self/status` is referenced at `0x210cc`, in the `JNI_OnLoad` startup sequence.
- `/proc/self/maps` is referenced at `0x2b2f8` and `0x2b348` in a separate native routine.

These references are consistent with reading process status and memory-map information. The `/proc/self/maps` routine contains direct syscall-based file access logic and repeated reads. The current pass has not yet fully reconstructed the loop's buffer handling or the purpose of the parsed map entries. This is evidence of native process/memory-map inspection, not by itself proof of anti-cheat evasion, injection, or data exfiltration.

### `/dev/socket/logd`

The `/dev/socket/logd` string is loaded as a 16-byte literal at `0x2162c` and placed into a stack buffer before a direct syscall at `0x21644`. This is a concrete code reference to Android's log daemon socket path. The exact syscall's arguments and subsequent control flow need further reconstruction; the current result does not yet prove a successful socket connection or log reading.

### Updated confidence table

| Artifact | Strongest evidence now available | What remains unknown |
|---|---|---|
| `.gtap` | Native path references plus `faccessat`/`openat`/`read`/`close` syscall sequence | File format, byte interpretation, origin, destination/use |
| `sys.log` | Native startup references to both game paths, passed through helper/syscall path | Read vs write vs existence check; content; any transmission |
| `/proc/self/status` | Native startup code reference | Whether it is read and which fields are parsed |
| `/proc/self/maps` | Two native references in a routine with direct file-read pattern | Parsed fields and purpose |
| `/dev/socket/logd` | Literal loaded into stack buffer and used in syscall setup | Whether connect succeeds and what data is read/written |

**Privacy conclusion update:** the native library does more than merely contain diagnostic path strings. It has a concrete filesystem access/read path for `.gtap`, and native code references the game's `sys.log`, process status/maps, and Android log daemon socket. No evidence in this pass establishes that these data are uploaded. The previously confirmed Java/Dex telemetry POST to `mklister.com` remains a separate, established outbound path.

## 65. Next native-analysis targets

1. Fully reconstruct the `.gtap` reader's syscall arguments, buffer loop, and caller/callee chain; identify whether the file is a configuration, state, or diagnostic artifact.
2. Follow the `sys.log` path references through the helper and determine whether native code checks, opens, reads, appends, or truncates the file.
3. Reconstruct the `/dev/socket/logd` syscall sequence and determine whether it creates a local socket connection and performs reads/writes.
4. Trace the `/proc/self/maps` parser and identify whether it searches for `libil2cpp.so`, game modules, executable regions, or hook targets.
5. Decode the APK binary manifest and inventory all DEX files and bundled SDKs before making whole-APK privacy claims.

## 66. Native `.gtap` handling: existence check and marker-file creation

Further instruction-level tracing clarifies the `.gtap` behavior beyond the earlier path-string xrefs.

### Two path candidates

The native code contains both:

- `/sdcard/Android/data/com.nianticlabs.pokemongo/files/.gtap`
- `/data/data/com.nianticlabs.pokemongo/files/.gtap`

At `0x24534–0x2456c`, it calls AArch64 syscall 48 (`faccessat`) against the external path first, then the internal path if the first check fails. The `dirfd` is `AT_FDCWD` (`-100`) and the mode argument is zero. If either path is accessible, the code records a state flag and skips the later creation branch. If neither is accessible, it proceeds to the helper at `0x212f0` with associated string arguments; the precise semantics of that helper call are not fully established here.

A separate routine at `0x56eb4–0x56f34` repeats the same two-path existence checks, then issues syscall 56 (`openat`) on the external path, falling back to the internal path. The open flags are `0x41` (`O_WRONLY | O_CREAT` on Linux). On successful open it immediately issues syscall 57 (`close`). There is no write syscall in this sequence. This is therefore consistent with creating/touching an empty `.gtap` marker when neither candidate already exists; it is not evidence that the file is populated with configuration or telemetry.

**Confidence:** high for the existence checks and `openat` flags/close; medium for interpreting the result as a marker/touch operation; unknown purpose and downstream consumer.

## 67. `/proc/self/maps` is actively read and parsed

The routine beginning at `0x2b2c4` is not merely a string reference. It opens the path held at `0x3000 + 0x84c` (the `.rodata` string is `/proc/self/maps`) using syscall 56 (`openat`) with `AT_FDCWD` and read-only flags. It then loops on syscall 63 (`read`), accumulating data into a large buffer (maximum requested capacity `0x3ffff`, 262,143 bytes), and closes the descriptor with syscall 57.

The subsequent code walks the captured text line by line. It parses hexadecimal address/range fields and permission-like characters, handles separators and path text, and builds/updates arrays of mapping-related values. It also compares parsed mapping ranges against other in-memory ranges and maintains mapping counts and selected range/offset values. The parser has a bounded read loop (up to 600 iterations) and bounded mapping bookkeeping (up to 127 entries in the visible path).

This establishes active process-memory-map inspection. It is compatible with module enumeration, address-range attribution, and detection of unexpected mappings or injected libraries. The disassembly inspected so far does **not** identify the exact detection rule, a named forbidden library, or a user-facing consequence. Do not label it definitively as anti-cheat or anti-hook logic until the downstream comparisons and decision branches are fully traced.

**Confidence:** high that `/proc/self/maps` is opened, read, parsed, and used to build mapping/range metadata; medium for the likely module-inspection purpose; unresolved whether it flags or blocks any specific module.

## 68. Native Android logd connection and structured log writes

The startup path around `0x215e0–0x21680` performs a concrete Unix-domain logging operation:

1. Syscall 198 creates a socket with domain `1` (`AF_UNIX`), type `2` (`SOCK_DGRAM`), protocol `0`.
2. It constructs a `sockaddr_un` buffer containing `/dev/socket/logd` and calls syscall 203 (`connect`) with a length of 110 bytes.
3. On success, it retains the descriptor in native state and enters the logging path. On failure, it closes the descriptor and records an error state.
4. The logging path writes a binary Android logger header and then the supplied tag/message strings using syscall 64 (`write`), with one-byte separators/terminators around the strings.

This is consistent with sending native log records directly to Android's `logd` datagram socket, rather than writing those particular messages to a file. The code path takes two strings as inputs and emits them as structured log data. The visible code does not show those records being sent to a remote server. This is a local device logging channel, separate from the confirmed Java/Dex HTTP telemetry POSTs documented earlier.

**Confidence:** high for AF_UNIX datagram socket creation, connection to `/dev/socket/logd`, and subsequent writes; exact Android log priority/tag interpretation should be confirmed against the logger wire-format definition and caller arguments.

## 69. Native inspection status and next unresolved questions

The native evidence now supports three concrete conclusions: (1) `.gtap` is checked and may be created as an empty marker; (2) `/proc/self/maps` is read and parsed into mapping/range metadata; and (3) native messages are sent to the local Android logging daemon. None of these findings, by themselves, proves that `.gtap`, process maps, or local log records are uploaded.

Remaining high-value work:

- Trace the `/proc/self/maps` parser's final consumers and identify the exact conditions that set warning/error flags or alter execution.
- Recover the call sites and string arguments for the `logd` writer to determine what events it records (including whether it logs mapping anomalies, `.gtap` state, or ordinary diagnostics).
- Trace the helper called after the `.gtap` existence checks at `0x24570` and its caller chain to establish what the marker gates.
- Continue mapping native JNI entry points to their Java callers and game-side IL2CPP methods, keeping confirmed behavior separate from inferred feature labels.


## 70. `/proc/self/maps` parser: what it actually extracts

A deeper pass through `0x2b2c4–0x2bc1c` establishes more than simple line parsing.

### Input and parsing

- The function opens the `/proc/self/maps` path at `0x2b388` with syscall 56 (`openat`), `AT_FDCWD`, and read-only flags (zero).
- It reads into a dedicated global buffer in a loop using syscall 63 (`read`), requesting up to `0x3ffff` bytes per iteration and stopping on EOF/error or after 600 loop iterations.
- It NUL-terminates the collected text and closes the descriptor.
- It iterates each newline-delimited maps record. It parses the hexadecimal start and end addresses around the dash, advances over the permission/offset/device/inode columns, and extracts the trailing pathname (or other mapping label).
- The parser converts hexadecimal text manually; it does not simply retain the raw line strings.

### Range bookkeeping and cross-checks

The routine stores parsed address-derived values in global arrays, tracks the number of discovered mappings (bounded at 127 entries in the relevant arrays), and associates entries with additional per-mapping counters/metadata. It also compares parsed ranges with pre-existing address/size records in other global tables. For matching address intervals it calculates relative offsets and performs further comparisons against values held in other native tables.

This is consistent with reconciling the process's live virtual-memory layout against a separately known set of ranges or metadata. It could support IL2CPP/module address resolution, integrity checking, or hook/injection inspection. The disassembly alone does not prove which of those is the primary purpose: the compared global tables are not yet symbolically identified, and the final decision logic has not been tied to a named JNI feature.

### Caller context

The only direct call site found in the current disassembly is at `0x23b7c`, where a preceding runtime-object operation sequence completes and then invokes the maps parser before returning from that native routine. This places the inspection in a native initialization/setup path, rather than demonstrating that it runs continuously on every gameplay action. No periodic scheduler or repeated Java callback has been established for this parser.

**Assessment:** high confidence that the routine reads and parses the live process map and cross-references address ranges; medium confidence that this supports module/IL2CPP address resolution or integrity inspection; low confidence in any specific anti-cheat/anti-hook interpretation until the global tables and result consumers are identified.

## 71. Limits of the current anti-tamper conclusion

The new trace narrows the claim but does not justify calling this a confirmed anti-cheat detector:

- Confirmed: process maps are read, hexadecimal mapping boundaries are parsed, mapping records are stored, and address ranges are cross-referenced with other native metadata.
- Not yet confirmed: a blacklist of module names, detection of Frida/Xposed/LSPosed, a decision to terminate or disable the overlay, or reporting of a detection to a server.
- The parser's call site is currently seen during native setup. Continuous monitoring has not been demonstrated.
- The native library also has confirmed local `logd` writes, but the inspected parser has not yet been connected to a specific log message.

Next, identify the native globals used as the reference range tables, map the post-parse comparison result into its caller, and recover the log writer's callers/arguments. Those are needed to distinguish ordinary IL2CPP address discovery from anti-instrumentation behavior.


## 72. Key correction: the maps parser explicitly locates `libil2cpp.so`

The parser's pathname comparison now resolves an important ambiguity from sections 70–71.

- The parser's comparison string is at native address `0x4af3`, and resolves exactly to `libil2cpp.so`.
- While walking each `/proc/self/maps` line, it compares the extracted pathname against this string. It therefore identifies mappings belonging to the Pokémon GO IL2CPP runtime, rather than treating every mapped library as an equivalent candidate.
- For matching records, it retains the mapping start/end information and associates the mapping with separately maintained native metadata tables.
- The routine then cross-references mapping-relative addresses/offsets against range-and-offset tables. This is consistent with resolving native IL2CPP addresses and validating that the expected address lies inside an appropriate mapped segment.
- Later code formats/logs diagnostic strings using the selected range and mapping information. The exact human-readable message text remains obfuscated in the string pool, so no user-facing wording is asserted.

This is strong evidence that a primary purpose of the maps parser is **IL2CPP module/segment discovery and address translation**. It may also provide integrity checks as a side effect, but the evidence currently available does not establish that the parser is principally an anti-Frida/anti-Xposed/anti-LSPosed detector.

### Revised assessment

| Claim | Assessment |
|---|---|
| Reads and parses `/proc/self/maps` | Confirmed |
| Explicitly matches `libil2cpp.so` | Confirmed |
| Uses map ranges with native address/offset metadata | Confirmed |
| Supports IL2CPP address resolution | Strongly supported |
| Detects a named hooking framework | Not established |
| Terminates or blocks execution on a hook detection | Not established |
| Uploads map contents | No evidence found |

This supersedes any earlier implication that the maps parser itself is likely an anti-hook scanner. Its demonstrated behavior is much more directly explained by the overlay's extensive IL2CPP reflection/address-resolution requirements.

## 73. Startup call-chain context for IL2CPP map discovery

The direct call to the maps parser at `0x23b7c` occurs after a sequence of JNI/IL2CPP runtime operations in the same native setup routine:

- It obtains runtime-associated objects through indirect function-table calls.
- It invokes a method through a runtime interface and checks the returned value.
- It then calls the maps parser and returns from the setup routine.

The parser's output is subsequently used by code that constructs diagnostic/address descriptions and resolves addresses against metadata tables. This connects the maps scan to runtime initialization and native address setup, not to a demonstrated periodic background surveillance loop.

**Practical implication:** the maps scan is relevant to understanding how Rasiel finds the correct loaded IL2CPP segment and converts metadata-relative offsets into live process addresses. It should not be presented as proof of anti-cheat behavior without a separate, explicit detection branch.

## 74. Remaining native questions after identifying the IL2CPP mapping

The highest-value next steps are now:

1. Identify the structure and provenance of the native range/offset tables used with the `libil2cpp.so` mapping.
2. Trace the address-description helper at `0x29808` and its callers to determine whether it formats diagnostics, resolves symbols, or both.
3. Map the IL2CPP segment-relative offsets to the JNI registrations and the Java `MenuHost` methods already catalogued.
4. Inspect the later initialization branches for any separate explicit module-name checks. Keep those separate from the now-explained `libil2cpp.so` discovery routine.


## 75. Correction: native helper at `0x29808` is a diagnostic formatter, not an address resolver

The previous section 74 listed `0x29808` as an address-description helper to investigate. Direct disassembly shows its role more precisely, and corrects that label.

### Function behavior

The routine at `0x29808`:

1. Measures the NUL-terminated lengths of two input strings (`x0` and `x1`).
2. Copies both strings into a stack buffer with a separating space.
3. Appends the literal marker `x=`.
4. Converts the third argument (`x2`) to hexadecimal using the character table `0123456789abcdef`.
5. Sends the assembled message to the native logging helper at `0x212f0`.

The call convention visible in its callers is therefore approximately `log_hex(prefix, description, value)`. The helper formats a diagnostic string; it does not itself look up an IL2CPP method or translate an RVA into a live pointer.

### Evidence from callers

- The routine has hundreds of call sites (746 direct branch-and-link references in the disassembly), indicating it is a shared diagnostic formatter used throughout the native library.
- At `0x2bac0`, the maps/ELF inspection path passes a selected base/address value into this formatter.
- At `0x2bb44` and `0x2bc14`, the same formatter is called while iterating address/range-related tables.
- Other callers pass values obtained from JNI/runtime operations and native initialization structures.

This connects the formatter to diagnostic reporting of values discovered by the runtime inspection code, but it does not make the formatter itself the resolver. The actual range/offset resolution logic remains in the callers and the map/ELF parsing routines.

## 76. The maps routine proceeds into ELF metadata validation

The code following the maps-file read does not stop at pathname matching. The parser:

- Selects the mapping associated with the target runtime library.
- Uses mapping boundaries as bounds for subsequent memory accesses.
- Reads ELF-style structures and validates that derived pointers/offsets remain within the selected mapped range.
- Walks indexed metadata tables and checks candidate offsets against recorded segment/range boundaries before accepting a derived address.
- Emits diagnostic values through the formatter described in section 75 when it reaches selected success/error paths.

The visible code includes checks of the form `candidate >= range_start` and `candidate + required_size <= range_end`, as well as offset calculations relative to a selected base. This is consistent with parsing/validating ELF load segments and translating file/segment-relative offsets into addresses in the currently loaded `libil2cpp.so`.

The string pool is encoded/decoded at runtime, so several diagnostic labels appear as short opaque fragments in the on-disk `.rodata` and should not be guessed from those raw bytes. The behavior above is based on control flow and bounds checks, not assumed meanings for the encrypted labels.

**Confidence:** high that the routine combines process-map discovery with bounded ELF-style metadata/offset validation; high that `0x29808` is a hex diagnostic formatter; medium-high that the overall purpose is runtime module/segment address resolution. Exact ELF header fields and each table's schema remain to be mapped.

## 77. Updated native-analysis interpretation

The best-supported model is now:

1. Native setup reads `/proc/self/maps` to locate the live `libil2cpp.so` mapping.
2. It uses the mapping's boundaries to safely inspect/validate runtime module metadata.
3. It cross-checks derived offsets and pointers against mapped ranges.
4. It reports selected values through a shared local logging formatter that writes to Android `logd`.

This is a coherent implementation pattern for locating and validating IL2CPP runtime addresses used by a game overlay. It does not establish that the code scans for Frida/Xposed/LSPosed, nor that map contents are transmitted off-device.

Remaining work: identify the precise ELF fields and global table layouts, then tie individual resolved addresses to the JNI registration table and Java `MenuHost` methods. That will make it possible to distinguish generic runtime support from game-specific hooks.


## 78. Recovered the complete JNI registration table for `com.holomod.MenuHost`

The JNI_OnLoad disassembly contains a direct, statically recoverable registration sequence:

- It resolves the Java class descriptor `com/holomod/MenuHost`.
- It passes a pointer to `0xa5ad0` as the `JNINativeMethod` array.
- It passes `0x51` (81) as the method count.
- It calls the JNI function-table entry at offset `0x6b8`, which is `RegisterNatives` for the JNI 1.6 interface.
- The table is in `.data.rel.ro`; its 81 entries are three-pointer records: method name, JNI signature, native function address. The dynamic relocation addends recover the exact strings and function RVAs.

This gives a direct method-name/signature/function mapping, rather than relying on symbol names or Java string searches.

### All 81 registered methods

| # | Java native method | JNI signature | Native RVA |
|---:|---|---|---:|
| 1 | `nativeAcctEvents` | `()I` | `0x216ec` |
| 2 | `nativeArmDirect` | `(DDIJ)I` | `0x21704` |
| 3 | `nativeArmFeed` | `(JLjava/lang/String;)Z` | `0x21990` |
| 4 | `nativeArmFind` | `()Z` | `0x21c0c` |
| 5 | `nativeAuthPidKey` | `()Ljava/lang/String;` | `0x21cf0` |
| 6 | `nativeAuthProvider` | `()Ljava/lang/String;` | `0x21d04` |
| 7 | `nativeCachedLogin` | `()Z` | `0x21d18` |
| 8 | `nativeCamArm` | `(FFFF)V` | `0x21dd8` |
| 9 | `nativeCamGeo` | `(Z)Z` | `0x21e0c` |
| 10 | `nativeCatchStats` | `()I` | `0x21ecc` |
| 11 | `nativeCooldownActions` | `()I` | `0x21ee4` |
| 12 | `nativeCooldownLast` | `()I` | `0x21ef0` |
| 13 | `nativeCooldownSoft` | `()I` | `0x21efc` |
| 14 | `nativeEncBad` | `()I` | `0x21f08` |
| 15 | `nativeEncOpen` | `()I` | `0x21f14` |
| 16 | `nativeEncStallMs` | `()I` | `0x21f28` |
| 17 | `nativeEncStallReset` | `()V` | `0x21fb0` |
| 18 | `nativeEncStatus` | `()I` | `0x21fc4` |
| 19 | `nativeFindCatch` | `(I)I` | `0x21fd0` |
| 20 | `nativeFindEncId` | `()J` | `0x22194` |
| 21 | `nativeFrames` | `()I` | `0x221a0` |
| 22 | `nativeGameLogout` | `()Z` | `0x221ac` |
| 23 | `nativeGameSoftReset` | `()Z` | `0x2238c` |
| 24 | `nativeGruntHop` | `(Z)Z` | `0x22474` |
| 25 | `nativeGruntSupplies` | `(I)I` | `0x22524` |
| 26 | `nativeGruntTags` | `()Ljava/lang/String;` | `0x225a4` |
| 27 | `nativeHasLoginArgs` | `()Z` | `0x225d8` |
| 28 | `nativeHuntReset` | `()V` | `0x225f8` |
| 29 | `nativeHuntTap` | `(DDI)Z` | `0x227f4` |
| 30 | `nativeInit` | `()V` | `0x22a20` |
| 31 | `nativeLastCaughtDex` | `()I` | `0x2430c` |
| 32 | `nativeLoginAs` | `()Z` | `0x24318` |
| 33 | `nativeLogoutDone` | `()Z` | `0x243e4` |
| 34 | `nativeMapHasSpawns` | `()Z` | `0x24404` |
| 35 | `nativeMapProbe` | `(FFZ)Z` | `0x244a8` |
| 36 | `nativeMapRefreshNow` | `()V` | `0x24808` |
| 37 | `nativeNearbySnapshot` | `()Ljava/lang/String;` | `0x24828` |
| 38 | `nativeOpenFind` | `()I` | `0x26794` |
| 39 | `nativePlayerIdent` | `()Ljava/lang/String;` | `0x269a0` |
| 40 | `nativePoiCount` | `()I` | `0x26c50` |
| 41 | `nativePoiSnapshot` | `()Ljava/lang/String;` | `0x26c5c` |
| 42 | `nativeQuestRemove` | `(Ljava/lang/String;)Z` | `0x27d70` |
| 43 | `nativeQuestSeen` | `()Ljava/lang/String;` | `0x27e68` |
| 44 | `nativeQuestsHeld` | `()Ljava/lang/String;` | `0x281b4` |
| 45 | `nativeR8K` | `()[B` | `0x28408` |
| 46 | `nativeR8Pack` | `([B)Z` | `0x285d8` |
| 47 | `nativeRadarCount` | `(I)I` | `0x286e4` |
| 48 | `nativeRangeExtra` | `()I` | `0x2871c` |
| 49 | `nativeReady` | `()I` | `0x287bc` |
| 50 | `nativeSetBagClean` | `(Z)V` | `0x287c8` |
| 51 | `nativeSetBlockNonShiny` | `(Z)V` | `0x28808` |
| 52 | `nativeSetBuddy` | `(ZZZZI)V` | `0x28838` |
| 53 | `nativeSetChores` | `(IZZZZZZII)V` | `0x28924` |
| 54 | `nativeSetEncRange` | `(I)V` | `0x28af4` |
| 55 | `nativeSetFarScan` | `(III)V` | `0x28b18` |
| 56 | `nativeSetGyms` | `(ZZI)V` | `0x28bac` |
| 57 | `nativeSetHuntActive` | `(Z)V` | `0x28cb4` |
| 58 | `nativeSetHuntKeepIv` | `(I)V` | `0x28cf8` |
| 59 | `nativeSetItemLimit` | `(II)V` | `0x28d1c` |
| 60 | `nativeSetSwitchProvider` | `(Ljava/lang/String;)Z` | `0x28d3c` |
| 61 | `nativeStampGo` | `(Ljava/lang/String;Ljava/lang/String;DD)Z` | `0x28e00` |
| 62 | `nativeStampList` | `()Ljava/lang/String;` | `0x290b4` |
| 63 | `nativeStampLoad` | `()V` | `0x290e0` |
| 64 | `nativeStampOn` | `(Z)V` | `0x29108` |
| 65 | `nativeStampPoll` | `()I` | `0x2914c` |
| 66 | `nativeSweepMoved` | `()V` | `0x29190` |
| 67 | `nativeTapDropped` | `()I` | `0x291d0` |
| 68 | `nativeTrainerName` | `()Ljava/lang/String;` | `0x291dc` |
| 69 | `nativeTryEncounterId` | `(J)V` | `0x29260` |
| 70 | `nativeUnsetBuddy` | `()V` | `0x2928c` |
| 71 | `nativeWildCount` | `()I` | `0x292ac` |
| 72 | `nativeWildLeft` | `(I)I` | `0x292b8` |
| 73 | `z62a` | `(JZ)V` | `0x29368` |
| 74 | `z62b` | `(J)J` | `0x295e0` |
| 75 | `z62c` | `()V` | `0x29660` |
| 76 | `z62d` | `(JDDLjava/lang/String;)V` | `0x296e8` |
| 77 | `z62e` | `()Z` | `0x296f8` |
| 78 | `z62f` | `()Z` | `0x29744` |
| 79 | `z66a` | `(II)V` | `0x2975c` |
| 80 | `z70a` | `(JDDLjava/lang/String;)V` | `0x29790` |
| 81 | `z70b` | `(J)V` | `0x297a0` |

**Note:** `nativeR8K` returns a byte array; its exact JNI descriptor is `()[B`.

### Feature clusters visible directly from the registered API

- **Hunt/encounter:** `nativeArmFind`, `nativeFindCatch`, `nativeFindEncId`, `nativeHuntReset`, `nativeHuntTap`, `nativeSetHuntActive`, `nativeSetHuntKeepIv`, `nativeLastCaughtDex`, `nativeWildCount`, `nativeWildLeft`.
- **Map/feed/POI:** `nativeArmDirect`, `nativeArmFeed`, `nativeMapHasSpawns`, `nativeMapProbe`, `nativeMapRefreshNow`, `nativeNearbySnapshot`, `nativePoiCount`, `nativePoiSnapshot`, `nativeRadarCount`, `nativeSetFarScan`.
- **Automation/settings:** `nativeSetBagClean`, `nativeSetBuddy`, `nativeSetChores`, `nativeSetGyms`, `nativeSetItemLimit`, `nativeSetEncRange`.
- **Quest/stamp route:** `nativeQuestRemove`, `nativeQuestSeen`, `nativeQuestsHeld`, `nativeStampGo`, `nativeStampList`, `nativeStampLoad`, `nativeStampOn`, `nativeStampPoll`.
- **Account/auth:** `nativeAuthPidKey`, `nativeAuthProvider`, `nativeCachedLogin`, `nativeGameLogout`, `nativeLoginAs`, `nativeLogoutDone`, `nativeSetSwitchProvider`, `nativePlayerIdent`, `nativeTrainerName`.
- **Diagnostics/state:** `nativeAcctEvents`, `nativeCatchStats`, cooldown and encounter-status getters, `nativeFrames`, `nativeReady`, `nativeTapDropped`, `nativeSweepMoved`.

These clusters are name/signature groupings only. Each method's actual effect still requires call-site and implementation tracing; a name is not sufficient to infer the full behavior.

## 79. JNI_OnLoad performs an explicit debugger check before registering native methods

The startup sequence at `0x21094` does more than register methods. Before calling JNI `GetEnv`, `FindClass`, and `RegisterNatives`, it:

1. Opens `/proc/self/status` using syscall 56 (`openat`) with `AT_FDCWD` and read-only flags.
2. Reads up to `0x7ff` bytes using syscall 63 (`read`), then closes the descriptor.
3. Scans the text for the literal field name `TracerPid:`.
4. Skips spaces/tabs and parses the following decimal digits.
5. If the parsed tracer PID is greater than zero, branches to the JNI_OnLoad failure return (`-1`), so the library does not complete native registration.

If no positive tracer PID is found, it continues with JNI version `0x00010006` (JNI 1.6), resolves `com/holomod/MenuHost`, and registers the 81 methods listed in section 78.

**Assessment:** this is confirmed Linux `TracerPid`-based anti-debugging at library load time. It detects a process being traced by a debugger using the kernel's `/proc/self/status` field. It does not, by itself, prove detection of Frida gadget, Xposed, LSPosed, Zygisk, or every instrumentation framework; those may require separate checks.

## 80. Important scope note: Java native declarations exceed this one registration table

The extracted `menu.dex` string table also contains names such as `nativeLocationUpdate` and `z62g`, which do not appear among the 81 entries registered for `com.holomod.MenuHost` by this JNI_OnLoad table. The Java code previously inspected resolves `nativeLocationUpdate` reflectively. Therefore, do not assume every native-looking DEX string belongs to this specific `MenuHost` registration array. Other native resolution paths/classes or dynamically resolved methods may exist and need separate tracing.


## 81. `nativeArmDirect(double, double, int, long)`: nearest-entry selection over the native 80-byte record table

The registered implementation at `0x21704` is now traced through its main selection loop.

### Record layout observed by the routine

It walks a global array rooted at `0xb8000 + 0x120`, using an 80-byte stride (`0x50`). For each record it reads:

- `+0x00`: pointer, required non-null
- `+0x08`: pointer, required non-null
- `+0x10`: 64-bit identifier/key, conditionally compared with the JNI `long` argument
- `+0x18` and `+0x20`: two doubles used as coordinates
- `+0x28`: integer field conditionally compared with the JNI `int` argument

The routine also checks the first pointer against a separate exclusion-pointer array rooted at `0xb9000 + 0xf48`, with count at `0xb9000 + 0xf44`.

### Selection behavior

- The candidate count is the global integer at `0xb8000 + 0x11c`.
- Null records and records whose pointers appear in the exclusion list are skipped.
- When the integer argument is positive, records with a positive `+0x28` value must match it.
- When the `long` argument is nonzero, the record's `+0x10` value is checked against it. A fallback comparison uses high bits when an exact match is not established.
- Candidate coordinates at `+0x18/+0x20` are compared with the two incoming doubles. The code computes squared Euclidean distance in the coordinate plane and keeps the closest eligible candidate.
- A selected candidate is accepted only if it is within the routine's configured distance threshold. It then copies the record's two leading pointers and identifier into global selection state, resets a related status field, logs the selected identifier and integer field, and returns true. Otherwise it returns false and may log the candidate count or distance diagnostic.

This is not a generic JNI wrapper: it is a real nearest-candidate selector over a native record collection, with exclusion, type/category, identifier and distance filters.

**What is not yet established:** the semantic meaning of the two pointers, the integer at `+0x28`, or the 64-bit key. They could represent a game entity, POI, feed record, or another overlay object. The method name `ArmDirect` and its arguments do not prove the user-facing feature on their own.

## 82. `nativeHuntTap(double, double, int)`: guarded nearest-object selection

The registered implementation at `0x227f4` is a short guard followed by a larger selection routine beginning at `0x22820`.

### Preconditions

It immediately returns false if any of three global state values are nonzero:

- `[0xba000 + 0x968]`
- `[0xbc000 + 0x690]`
- `[0xb9000 + 0xf40]`

These appear to be mutually relevant operation/busy/status flags; their exact names are unavailable in the stripped binary.

### Candidate traversal

- It scans a pointer array rooted at `0xbc000 + 0x580`, at most 24 entries.
- Null pointers are skipped.
- A pointer exclusion list rooted at `0xbc000 + 0x6a8` is consulted using the count at `0xbc000 + 0x668`.
- If the JNI integer argument is positive, helper `0x2aaf4(candidate)` is called and its returned integer must match the requested value.
- Helper `0x2ac70(candidate, &coordA, &coordB)` extracts two double coordinates. Candidates for which it returns false are skipped.
- The routine computes squared distance from the supplied JNI double coordinates to each candidate and retains the closest match.
- It compares the winning distance against a configured threshold. If no eligible candidate is found, or the threshold test fails, the function returns false.

There is also a fallback path using a previously stored global pointer at `0xb9000 + 0xf28` when the integer argument is negative. The function can ask helper `0x2ad78` for a candidate in that fallback path.

**Assessment:** high confidence this method selects the nearest eligible object from a small live candidate set using coordinates and optional integer filtering. The method name suggests a hunt interaction, but the binary evidence alone does not prove that the selected object is a Pokémon encounter or that the routine performs a tap itself. No game-side tap/click call is visible in the selection loop shown here.

## 83. JNI initialization resolves and caches a large set of Java-side runtime references

The registered `nativeInit()` implementation at `0x22a20` begins by emitting a native diagnostic, then uses the supplied `JNIEnv` and Java object to resolve and cache many Java-side IDs and references in global storage.

The visible pattern repeatedly:

- Loads a string pair from the native string pool.
- Calls JNI function-table entries through the current `JNIEnv`.
- Stores returned IDs/references in global slots under `0xbc000` (for example offsets `0x950`, `0x958`, `0x960`, `0x968`, `0x980`, `0x988`, `0x990`, and `0x998`).
- Checks for pending Java exceptions after selected JNI calls and clears them through the JNI exception functions before continuing.

This is consistent with a bootstrap phase that caches Java methods/fields or class references needed by later native callbacks. The actual class/member names are not safely recoverable from the currently decoded string view, so this section does not invent names or claim specific Android APIs.

**Important:** `nativeInit` is not itself the game-side IL2CPP resolver. It is a Java/JNI bridge initialization routine. The separate `/proc/self/maps` and `libil2cpp.so` inspection path described earlier belongs to native runtime setup and should be analyzed independently.

## 84. What these three functions establish—and what remains open

| Function | Confirmed behavior | Still unresolved |
|---|---|---|
| `nativeArmDirect` | Filters an 80-byte record array and chooses the nearest coordinate match; stores selected record state | Record/object type and exact feature semantics |
| `nativeHuntTap` | Checks busy flags, scans up to 24 candidate pointers, filters and selects the nearest coordinate match | Whether candidates are Pokémon, and where actual game interaction occurs |
| `nativeInit` | Resolves/caches JNI-side IDs/references and handles pending exceptions | Exact Java classes/methods/fields represented by the encoded strings |

Next trace the producer of the 80-byte records and 24-entry candidate array. Their insertion/update JNI methods will reveal the actual object types and connect these selection functions to feed, encounter, map, or hunt behavior.
