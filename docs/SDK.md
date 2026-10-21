# ID TECH UHF SDK for Linux - developer guide

Version 3.1.0 · Ubuntu 22.04 (arm64) on the ZTL RK3576 board · Java 11+

There are two ways for your software to use the UHF reader:

| | Reader SDK | Client SDK |
|---|---|---|
| Jar | `idtech-uhf-sdk.jar` | `idtech-connect-client.jar` |
| Talks to | the reader, directly over its serial port | an ID TECH app's web service (Setting > Web in the app), over HTTP |
| Runs on | the board | any machine on the network (or the board) |
| Reader shared with the ID TECH app? | no - one process at a time | yes - the app owns the reader, any number of clients |
| Same as on Android | the platform SDK (`android.hardware.uhf`, `com.reader.uhf.sdk`) | the `connect-client` SDK |

Choose the **client SDK** when the ID TECH app is in use on the board (operators, MQTT, other
clients); choose the **reader SDK** for a dedicated application that owns the reader itself.
For other languages, use the [REST API](#rest-api) or [MQTT](#mqtt) directly.

Installed by `idtech-uhf_3.1.0_arm64.deb`:
`/usr/share/java/idtech-uhf-sdk.jar`, `/usr/share/java/idtech-connect-client.jar`, examples in
`/usr/share/doc/idtech-uhf/examples/`. The zip kits in `dist/` hold the same jars with sources.

---

## 1. Reader SDK

### 1.1 Quick start

```java
import android.hardware.uhf.*;
import com.reader.uhf.sdk.*;

try (UhfSdkSession s = UhfSdk.open()) {
    UhfReaderInfo info = s.getReaderInfo();
    System.out.println(info.getFirmwareVersion() + " " + info.getBandName());
    for (UhfTag t : s.inventory()) {
        System.out.println(t.getEpcHex() + "  RSSI " + t.getRssi());
    }
}
```

```
javac -cp /usr/share/java/idtech-uhf-sdk.jar App.java
java  -cp /usr/share/java/idtech-uhf-sdk.jar:. App
```

The API is the Android platform SDK's: code written for the Android board compiles unchanged,
except `UhfSdk.open(context)` becomes `UhfSdk.open()` (Linux has no Context).

### 1.2 Requirements

- Your user in group `dialout` (the reader's serial port). The board's `ubuntu` user already is.
- Only one process can hold the reader. If the ID TECH app has it connected, `open()` or the first
  call fails with `ERROR_BUSY` ("... is held by another process"). Disconnect in the app, or use
  the client SDK instead.
- The reader is found by `/etc/idtech/uhf.conf` (see [Configuration](#4-configuration)).

### 1.3 Entry points

| Call | Meaning |
|---|---|
| `UhfSdk.isSupported()` | a candidate reader port exists (not that a reader answers) |
| `UhfSdk.open()` | opens the exclusive reader session; `close()` it (try-with-resources) |
| `UhfSdk.openFake()` | an in-memory demo reader, for UI work without hardware |
| `UhfManager.getInstance()` | lower level: `isReaderAvailable()`, `getReaders()`, `openSession()`, `getActiveDevice()` |

### 1.4 `UhfSdkSession`

All calls are synchronous and serialized (the reader takes one command at a time). Methods that
talk to the reader throw `UhfException`.

**Reader configuration**

| Method | Notes |
|---|---|
| `UhfReaderInfo getReaderInfo()` | firmware, model, band, frequency range, max power, antenna mask |
| `void setRegion(int bandCode)` | regulatory band, persisted and re-applied at every start (codes below) |
| `void setRfPower(int dBm)` / `setRfPower(int dBm, boolean allowAboveSafeCeiling)` | 0-30; above the safe ceiling (default 20) needs the override |
| `int getRfPower()`, `getMaxRfPower()`, `getSafeRfPowerCeiling()` | |
| `void setEnabledAntennas(int mask)` / `int getEnabledAntennas()` | bit 0 = antenna 1 ... bit 3 = antenna 4 |
| `void setAntennaCheckEnabled(boolean)` / `boolean isAntennaCheckEnabled()` | refuse to transmit into a missing antenna |

**Inventory**

| Method | Notes |
|---|---|
| `List<UhfTag> inventory()` / `inventory(InventoryParams)` | one polled round; an empty field is an empty list, not an error |
| `void setStreamParams(StreamParams)` | applies at the next `startStreaming` |
| `void startStreaming(Executor, TagCallback)` / `void stopStreaming()` | real-time mode; batches on your executor |
| `boolean isStreaming()` | |
| `StreamStats getStreamStats()` / `void resetStreamStats()` | unique tags, reads, reads/s (from the first read) |
| `startBufferedInventory(InventoryParams)`, `getBufferedTagCount()`, `readBuffer()`, `clearBuffer()`, `stopBufferedInventory()` | reader-side buffered inventory |

**Tag memory and security** - every call takes a `TagFilter` selecting the tag:
`TagFilter.byEpc(epc)`, `TagFilter.byEpc(epc, accessPassword)` or `TagFilter.any()` (the only tag
in the field). Select by EPC whenever more than one tag may be in the field: a write and its
verification read are separate operations and would otherwise reach different tags.

| Method | Notes |
|---|---|
| `byte[] readTagMemory(filter, bank, wordAddress, wordCount)` | banks: 0 reserved, 1 EPC, 2 TID, 3 user (`UhfSession.BANK_*`) |
| `void writeTagMemory(filter, bank, wordAddress, byte[] data)` | even length; verified by read-back unless `setVerifyWrites(false)` |
| `void blockWrite(...)`, `void blockErase(filter, bank, wordAddress, wordCount)` | |
| `void writeEpc(filter, byte[] newEpc)` | rewrites the PC length bits when the length changes |
| `void setAccessPassword(filter, old, new)`, `void setKillPassword(filter, accessPw, newKillPw)` | 4-byte passwords |
| `void lockTag(filter, region, lockState, accessPw, confirmIrreversible)` | `LOCK_REGION_*`, `LOCK_OPEN/LOCKED/PERM_OPEN/PERM_LOCKED`; permanent states need `confirmIrreversible=true` |
| `void killTag(filter, byte[] killPassword)` | irreversible; an all-zero password is refused |
| `void setVerifyWrites(boolean)` | default on |

### 1.5 Types

| Type | Key members |
|---|---|
| `UhfTag` | `getEpc()`, `getEpcHex()`, `getTid()`, `getRssi()`, `getAntenna()`, `getTimestampNanos()` |
| `UhfReaderInfo` | `getFirmwareVersion()`, `getModelType()`, `getBandCode()`, `getBandName()`, `getMin/MaxFrequencyMhz()`, `getMaxRfPower()`, `getAntennaMask()`, `isAntennaCheckEnabled()` |
| `InventoryParams.Builder` | `setQValue(0..15)` (default 4), `setSession(SESSION_S0..S3)`, `setAntennaTargetScanTime(port, TARGET_A/B, ms)` |
| `StreamParams.Builder` | `setQValue`, `setSession`, `setReadPauseMillis`, `setFilterTimeMillis`, `setBatchWindowMillis`, `setMaxBatchSize`, `setUniqueOnly` |
| `StreamStats` | `getUniqueTagCount()`, `getTotalReadCount()`, `getReadsPerSecond()`, `getDroppedBatchCount()`, `getElapsedMillis()` |
| `TagCallback` | `onTagsRead(List<UhfTag>)`, `onStreamError(code, message)`, `onBatchesDropped(count)` |

Q value: 2^Q should roughly match the tag population (measured best on this reader: Q6 at ~210
tags, Q7 at ~310).

### 1.6 Errors - `UhfException.getErrorCode()`

| Code | Name | Typical cause |
|---|---|---|
| 1 | `ERROR_BUFFER_FULL` | reader buffer full (buffered inventory) |
| 2 | `ERROR_ANTENNA_FAULT` | antenna check failed |
| 3 | `ERROR_COMMAND_REJECTED` | no tag answered, wrong password, memory locked - the message says which |
| 4 | `ERROR_PROTOCOL` | corrupt reply, or write verification failed |
| 5 | `ERROR_INVALID_PARAMETER` | out-of-range argument, power above the safe ceiling |
| 6 | `ERROR_TIMEOUT` | the reader did not answer |
| 7 | `ERROR_DEVICE_LOST` | no reader on any candidate port, or it was unplugged |
| 8 | `ERROR_BUSY` | another session or process holds the reader |

A reader that is unplugged and plugged back in needs no new session: the session's next reader call
finds it again (the port is re-detected, the reader reset and its region re-applied) and a stream the
unplug ended can be started again. While the reader is away, calls fail with `ERROR_DEVICE_LOST`.

### 1.7 Region band codes

| Code | Band | | Code | Band |
|---|---|---|---|---|
| 1 (0001) | China 2 | | 7 (0111) | Peru |
| 2 (0010) | US | | 8 (1000) | China 1 |
| 3 (0011) | Korea | | 9 (1001) | EU3 |
| 4 (0100) | EU (default) | | 10 (1010) | Taiwan |
| 6 (0110) | Ukraine | | 12 (1100) | US 3 |

### 1.8 Command line: `uhfctl`

```
uhfctl ports | info | inventory [-n N] | stream [SECONDS] | power [DBM] [--force]
       region [CODE] | antenna [MASK] [--check on|off]
       read BANK PTR COUNT | write BANK PTR HEX | write-epc NEWEPC | kill KILLPW
       (tag commands: --epc HEX selects the tag, --pw HEX the access password)
```

---

## 2. Client SDK

### 2.1 Quick start

```java
import com.idtech.connectclient.core.*;

ApiClient api = new ApiClient("idtech.local:8080");
api.login("admin", password);
JsonValue info = api.connect();                 // the app opens the reader for you
EventStream events = new EventStream(api, new EventStream.Listener() {
    public void onEvent(String event, String data) {
        if ("tags".equals(event)) System.out.println(data);   // JSON array of tags
    }
    public void onClosed(String reason) { }
});
events.start();
api.startStream();
// ...
api.stopStream();
events.close();
api.logout();
```

```
javac -cp /usr/share/java/idtech-connect-client.jar ClientExample.java
java  -cp /usr/share/java/idtech-connect-client.jar:. ClientExample idtech.local:8080 admin <password>
```

Prerequisite: in the ID TECH app, **Setting > Web** switched on (the panel shows its addresses).
Accounts are the app's: `admin` must have changed the seeded password (`admin@123`) first.

### 2.2 `ApiClient`

| Method | REST call |
|---|---|
| `ApiClient(String hostPort)`, `setTimeoutMs(int)` | |
| `JsonValue login(user, password)` / `void logout()` / `JsonValue me()` | `/api/login`, `/api/logout`, `/api/me` |
| `void changePassword(current, next)` | `/api/password` |
| `JsonValue status()` | `/api/status` |
| `JsonValue connect()` / `void disconnect()` | `/api/session/connect`, `/api/session/disconnect` |
| `JsonValue readerInfo()` | `/api/reader` |
| `JsonValue params()` / `setParams(q, session, target, scanTimeMs)` | `/api/params` |
| `void startStream()` / `void stopStream()` | `/api/stream/start`, `/api/stream/stop` |
| `void setPower(int dBm)`, `void setRegion(int bandCode)` | `/api/power`, `/api/region` |
| `String readMemory(epc, bank, wordPtr, wordCount, accessPw)` | `GET /api/tag/memory` |
| `void writeMemory(epc, bank, wordPtr, dataHex, accessPw)` | `POST /api/tag/memory` |

Failures throw `ApiError` (an `IOException`) with `status` = the HTTP status: 401 not signed in,
403 password change required, 409 reader busy / tag operation refused, 503 no reader.

`EventStream` delivers the server-sent events of `/api/stream` (`tags`, `log`, `error`);
`TagList` aggregates `tags` batches into unique rows (`addBatch`, `snapshot`, `size`, `reads`);
`JsonValue` is a small JSON reader (`get`, `str`, `num`, `bool`, `list`).

---

## 3. REST API

Base `http://<board>:8080` (`idtech.local` on the local network). Sign in with
`POST /api/login {"username":..., "password":...}`; send the returned token as
`Authorization: Bearer <token>` (or use the session cookie). JSON in, JSON out.

| Area | Routes |
|---|---|
| Accounts | `POST /api/login`, `POST /api/logout`, `GET /api/me`, `POST /api/password`, `GET/POST /api/users`, `POST /api/users/delete`, `POST /api/users/reset` |
| Reader session | `GET /api/status`, `POST /api/session/connect`, `POST /api/session/disconnect`, `POST /api/session/reset`, `GET /api/reader` |
| Scanning | `GET/POST /api/params`, `POST /api/inventory`, `POST /api/stream/start`, `POST /api/stream/stop`, `GET /api/stream/stats`, `GET /api/stream` (SSE) |
| Reader settings | `POST /api/power`, `GET/POST /api/antenna`, `POST /api/region` |
| Tags | `POST /api/tag/epc`, `GET/POST /api/tag/memory`, `POST /api/tag/password`, `POST /api/tag/kill`, `POST /api/tag/lock`, `POST /api/tag/blockerase` |
| MQTT | `GET/POST /api/mqtt`, `POST /api/mqtt/test` |
| Diagnostics | `GET /api/network`, `GET /api/debuglog`, `POST /api/debuglog/clear` |

Tag JSON (inventory replies and `tags` events): `{"epc":"E200...","rssi":62,"antenna":1,"count":0}`.

## MQTT

Configured in the app (Settings > MQTT) or `POST /api/mqtt`. While enabled, the app keeps the
reader streaming and publishes tag batches to `<prefix>/tags` and its state to `<prefix>/status`
(QoS 0 or 1, optional TLS and credentials).

---

## 4. Configuration

`/etc/idtech/uhf.conf` (read by the app, `uhfctl` and the reader SDK):

| Key | Default | |
|---|---|---|
| `device` | `auto` | a port path, or `auto` to probe `scan` and use the first port where a reader answers |
| `scan` | `/dev/ttyUSB*,/dev/ttyACM*,/dev/ttyDC3` | USB adapters, then the board's native UART |
| `safe.power.ceiling` | `20` | dBm allowed without the explicit override |
| `http.port` | `8080` | the app's web service |
| `mdns.enabled`, `mdns.name` | `true`, `idtech.local` | |
| `state.dir` | `~/.local/share/idtech` | accounts, scan/MQTT/web settings, persisted region |

Each key can be overridden per process with `-Duhf.<key>=...` or the environment variable
`UHF_<KEY>` (dots become underscores), e.g. `UHF_DEVICE=/dev/ttyUSB0 uhfctl info`.
