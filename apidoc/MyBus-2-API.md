# myBus 2.x ("myBus online") — Public API Reference

**Unofficial, reverse-engineered documentation** of the undocumented HTTP API used by the Android
app *myBus online* (`com.taran.mybus`, version **2.8.13**, build 161) published by
**R&G PLUS Sp. z o.o.** (Mielec, Poland; app brand "myBus", legacy vendor domain `taran.com.pl`).

All information below was obtained by static analysis of the publicly distributed APK and by
**live verification against production servers** (read-only `GET` requests, public timetable data
only). Verified on **2026-11-17**.

> **Why this exists:** the app consumes public transit timetable and live-vehicle data that is
> served without any authentication to any client that mimics two trivial HTTP headers. This
> document describes that interface for interoperability purposes. See [Legal & ethical notes](#13-legal--ethical-notes).

---

## Table of contents

1. [Architecture overview](#1-architecture-overview)
2. [The three API layers](#2-the-three-api-layers)
3. [Access protocol: `User-Agent` + `Age` handshake](#3-access-protocol-user-agent--age-handshake)
4. [Per-city endpoints (SchedulesService)](#4-per-city-endpoints-schedulesservice)
5. [Central config API (shared.mybus.rg.com.pl)](#5-central-config-api-sharedmybusrgcompl)
6. [Legacy global config service (taran.com.pl)](#6-legacy-global-config-service-tarancompl)
7. [Response data models](#7-response-data-models)
8. [Schedule file format (gzipped SQLite)](#8-schedule-file-format-gzipped-sqlite)
9. [City registry (54 networks)](#9-city-registry-54-networks)
10. [Code examples](#10-code-examples)
11. [Live verification log](#11-live-verification-log)
12. [Undocumented-but-observed fields](#12-undocumented-but-observed-fields)
13. [Legal & ethical notes](#13-legal--ethical-notes)

---

## 1. Architecture overview

```
┌──────────────────────┐
│  myBus online 2.8.13 │  (com.taran.mybus)
└──────────┬───────────┘
           │
           │ ① bootstrap / "new app" promo
           ▼
   https://shared.mybus.rg.com.pl/api/v1/config/cities        (JSON, central, no auth)
           │
           │ ② maintenance / info messages (legacy)
           ▼
   http://www.taran.com.pl/myBusOnlineCfg/ConfigurationService.svc/GetMessage   (XML, legacy)
           │
           │ ③ everything else — per city/region
           ▼
   {CITY_BASE_URL}/PingService | GetScheduleFile | GetTimeTableReal | GetVehicles | ...
                            (WCF REST-style, XML / gzip-SQLite, custom Age header)
```

The app is a **multi-network client**: it embeds a registry of 54 cities/regions (see
[section 9](#9-city-registry-54-networks)), each served by an independent deployment of the same
**WCF (`*.svc`) service** exposing an identical set of GET endpoints. After choosing a city, the
app stores that city's base URL and 5-letter `clientCode` and talks only to that server.

Transport details observed in the decompiled client (`g4.c`, "NewRestClient"):

- plain `HttpURLConnection`, HTTP/1.1, `GET` only (the client contains a `POST` branch that is never used)
- default connect/read timeout: 5000 ms (widgets/services use it too)
- every request carries `User-Agent: myBusOnline` and the `Age` header (see section 3)
- query parameters are `application/x-www-form-urlencoded`-encoded (`URLEncoder`, UTF-8)
- responses are XML (`application/xml; charset=utf-8`), except `GetScheduleFile`
  (`application/octet-stream`, a gzip stream) and the central config API (JSON)
- most city endpoints are plain **HTTP** (the app sets `usesCleartextTraffic="true"`); some cities use HTTPS

---

## 2. The three API layers

| # | Layer | Base URL | Format | Auth | Status (2026-11-17) |
|---|-------|----------|--------|------|---------------------|
| 1 | Central config | `https://shared.mybus.rg.com.pl/api/v1/config` | JSON | none | **working** |
| 2 | Legacy global config | `http://www.taran.com.pl/myBusOnlineCfg/ConfigurationService.svc` | XML | none | unreachable from our test vantage point (connection refused); endpoint extracted from client code |
| 3 | Per-city schedule service | see [section 9](#9-city-registry-54-networks) | XML + gzip SQLite | `User-Agent` + `Age` | **working** (verified on Mielec, Lublin, Kielce, Łódź; Gdańsk WAF-blocked our test IP; several HTTPS endpoints timed out from the test network) |

---

## 3. Access protocol: `User-Agent` + `Age` handshake

Every request to a per-city `SchedulesService` **must** carry:

```
User-Agent: myBusOnline
Age: <statSecToken> + <sum of Unicode code points of the city's clientCode>
```

`statSecToken` is an integer the client persists in its settings (`statSecToken`, default **60**).
The handshake, verified live:

1. **Bootstrap:** call `PingService` with `Age = 60 + Σord(clientCode)`.
   (Missing/wrong `User-Agent`, missing `Age`, or any other `Age` value → `400 Unauthorized access`.)
2. `PingService` responds with `<int>T</int>` — the **current server-issued token**
   (`T = 2054` on 2026-11-17; the value was identical across all city servers tested, so it is a
   server-side/global value, not per-city). The client stores it as `statSecToken`.
3. **All subsequent data calls** must use `Age = T + Σord(clientCode)`.

> **Quirk (tested):** `PingService` accepts *only* the bootstrap value `60` in this arithmetic —
> re-pinging with a previously issued token (`2054`) is rejected with `400`, exactly like any
> other wrong value. In other words: bootstrap the session with `60`, and use the issued `T`
> only for the data endpoints.

Error response when the handshake is violated:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/xml; charset=utf-8

<string xmlns="http://schemas.microsoft.com/2003/10/Serialization/">Unauthorized access</string>
```

Example — Lublin (`clientCode = "LUBLI"`, Σord = 376):

| Step | Request | `Age` | Result |
|------|---------|-------|--------|
| 1 | `GET .../PingService` | `436` (= 60+376) | `200 <int>2054</int>` |
| 2 | `GET .../GetTimeTableReal?nBusStopId=1&nBusStopGroupId=0` | `2430` (= 2054+376) | `200 <Departures …>` |
| —  | same call with `Age: 436` | stale token | `400 Unauthorized access` |

> There is **no per-user registration, no API key, no account** — the scheme is a lightweight
> shared "secret" (a fixed user-agent string plus an integer token derived from a public city
> code) intended to filter naive scrapers. All inputs are shipped inside the public APK.

---

## 4. Per-city endpoints (`SchedulesService`)

All endpoints are `GET {CITY_BASE_URL}/{Endpoint}?{params}`. All responses are XML with root
elements as shown, except where noted. "Verified" = exercised against a live server on 2026-11-17.

### 4.1 `PingService`

Health check + token issuance. **This is the entry point of the handshake.**

- **Parameters:** none
- **Request:** `Age: 60 + Σord(clientCode)`
- **Response:** `<int xmlns="…">2054</int>` — current server token
- **Client behavior:** stores the value as `statSecToken` (used in the `Age` header of all
  subsequent calls) and records "today" as last-check date; re-pings at most once per day
  **Verified**: Lublin, Kielce, Mielec, Łódź → `2054`

### 4.2 `CompareScheduleFile`

Asks whether the client's locally cached schedule database is current.

- **Parameters:**
  - `nIdWersja` (int) — version id from the local DB's `WERSJE.id_wersja`
  - `nGeneracja` (int) — generation from the local DB's `WERSJE.generacja`
- **Response:** `<int>1</int>` if the client's copy is up to date, `<int>0</int>` otherwise
- **Client behavior:** when `0`, downloads `GetScheduleFile`
  **Verified**: Lublin — `(4597, 1)` → `1`; `(4597, 2)`, `(4596, 1)`, `(0, 0)` → `0`

### 4.3 `GetScheduleFile`

Downloads the complete static timetable database for the selected city.

- **Parameters:** none
- **Response:** `application/octet-stream` — a **gzip-compressed SQLite 3 database**
  (see [section 8](#8-schedule-file-format-gzipped-sqlite))
- The client sends `Accept-Encoding: identity` for this call and streams the body to a
  `GZIPInputStream`
  **Verified**: Lublin — 954,726 bytes gzip → 5,419,008 bytes SQLite (1,486 stops)

### 4.4 `GetTimeTableReal`

Real-time departure board for a stop (or stop group).

- **Parameters:**
  - `nBusStopId` (int) — stop id (from the schedule DB `PRZYSTANKI.id`)
  - `nBusStopGroupId` (int) — stop-group id, `0` for a single stop
- **Response:** `<Departures time="14:23" i="1">` containing an optional `<N>` notice element and
  `<D …/>` elements — see [model 7.2](#72-departure-d--gettimetablereal--getdepartureinfo)
- **Polling:** the app refreshes every `timetableRefreshTime` seconds (default 15–30 s)
  **Verified**: Lublin, Kielce, Mielec, Łódź

### 4.5 `GetDepartureInfo`

Single-departure detail (used by the "departure alarm" service while tracking one departure).

- **Parameters:**
  - `cDate` (string, `yyyy-MM-dd`) — service date of the trip leg (the `dg` attribute of a
    `GetConnections` leg, or today for board departures)
  - `nBusStopId` (int) — stop id
  - `nUqTripId` (long) — unique trip id (`uti` from a connections leg, or the `n` attribute of a
    board departure)
- **Response:** a single `<D …/>` element with the same attribute set as a board departure, or an
  empty `<D/>` when no data
  **Verified**: Kielce — `cDate=2026-11-17&nBusStopId=426&nUqTripId=6373224` → `<D i="2846" … v="15:34"/>`

### 4.6 `GetVehicles`

Live vehicle positions for the whole network (or a filtered subset).

- **Parameters** (all string lists; `;`-separated when multiple values; empty string = no filter):
  - `cNbLst` — filter by course numbers (`nb`)
  - `cIdLst` — filter by vehicle ids (`id`)
  - `cRouteLst` — filter by route numbers (`nr`)
  - `cTrackLst` — filter by track ids
  - `cDirLst` — filter by direction codes (`dd`/`kr`, e.g. `T`, `P`)
  - `cKrsLst` — filter by course ids (`ik`)
- **Response:** `<VL>` containing `<V …/>` elements — see [model 7.3](#73-vehicle-v--getvehicles)
- **Polling:** the app polls every ~1 s while animating markers, otherwise every ~5 × 200 ms loop
  cycle on the map screen
  **Verified**: Lublin — 32 live vehicles returned

### 4.7 `GetVehicleTimeTable`

Remaining stops and countdowns for one running course (vehicle).

- **Parameters:**
  - `nIdKursu` (int) — course id; the app passes `0`
  - `nNb` (int) — the vehicle's course number (`nb` from `GetVehicles`)
- **Response:** `<Schedules id="20261117" nr="23" type="J" o="KUPIECKA IKEA">` with `<Stop …/>`
  children — see [model 7.4](#74-vehicle-schedule-stop--getvehicletimetable)
  **Verified**: Lublin — `nIdKursu=0&nNb=1858` → 4 upcoming stops with countdowns

### 4.8 `GetRouteVariantWithTransitPoints`

Geometry (track polyline) of a route variant, split into segments between stops.

- **Parameters:**
  - `nIdKursu` (int) — course id (e.g. the `nb` of a live vehicle)
  - `cRoute` (string) — route number filter (app passes `""`)
  - `cRouteVariant` (string) — route variant filter (app passes `""`)
  - `nIdPodm` (int) — substitution id; the app passes `-1`
- **Response:** `<R r="18   " t="J">` with `<T i1 i2>` segments containing `<Pkt l x y>` points —
  see [model 7.5](#75-route-variant--getroutevariantwithtransitpoints)
  **Verified**: Lublin — `nIdKursu=1858` → 12,134 bytes of track geometry

### 4.9 `GetConnections`

Door-to-door trip planner (journey search).

- **Parameters:**
  - `nLat1`, `nLng1` (double as string) — origin coordinates (WGS-84, decimal degrees)
  - `nLat2`, `nLng2` (double as string) — destination coordinates
  - `nSearchMode` (int) — `0` = depart at given time, `1` = arrive by given time
  - `cStartHour` (string, `HH:mm`) — search time
  - `cDate` (string, `yyyy-MM-dd`) — search date
  - `nMaxTrans` (int) — maximum number of transfers
  - `nMaxMinutesChangeInterval` (int) — minimum transfer time in minutes (app sends `0`)
  - `cSkipRoutes` (string) — routes to skip (`;`-separated; app sends `""`)
  - `cPrefVehicleFlags` (string) — preferred vehicle flags (`;`-separated; app sends `"N"` when
    the user requires low-floor vehicles, else `""`)
- **Response:** `<CC>` with `<C …>` journey alternatives, each containing `<G …>` leg points with
  nested `<PP><P x y>` walking/transfer geometry — see [model 7.6](#76-connection-c--leg-g--getconnections)
- **Availability:** only on servers whose schedule DB has `CFG.lTripPlaner = 'Y'`
  (Lublin has `N` → `400`; Kielce has it enabled)
  **Verified**: Kielce — 60,805 bytes, 104 transit legs returned

### 4.10 `GetDispatcherMessages`

Traffic-control / dispatch messages for the city.

- **Parameters:** none
- **Response:** `<R>` with `<M …/>`-style children — see [model 7.7](#77-dispatcher-message--getdispatchermessages)
- The app polls this via a WorkManager background job and raises Android notifications for new
  messages
  **Verified**: Lublin, Kielce, Mielec, Łódź — `200` (no active messages on test date: `<R/>`)

---

## 5. Central config API (`shared.mybus.rg.com.pl`)

Plain JSON, no auth, no custom headers required. The 2.8.13 client uses it for exactly one
thing: the list of `clientCode`s that have the successor app (`pl.rg.mybus`, "myBus 3") available,
to show a promotional dialog.

### `GET /api/v1/config/cities`

**Verified 200** on 2026-11-17. Response (abridged; full sample in `samples/central_config_cities.json`):

```json
{
  "configVersion": 2,
  "cities": [
    {
      "id": "chelm",
      "dataId": "chelm",
      "clientCode": "CHELM",
      "name": "Chełm",
      "environment": "",
      "operator": "Chełmskie Linie Autobusowe Sp. z o.o.",
      "operatorShort": "CLA",
      "region": "woj. lubelskie",
      "regionI18n": { "cs": "…", "de": "…", "en": "Lublin Voivodeship", "…": "…" },
      "apiBaseUrl": "https://rozklad.cla.net.pl/api-mybus-3",
      "lat": 51.1431,
      "lon": 23.4716,
      "mapZoom": 12,
      "certificatePins": [],
      "modules": { "planner": true, "vehiclesLive": true, "messages": true, "qr": true },
      "cityFunctions": [
        { "id": "chelm-carrier-www", "type": "web-link", "url": "https://www.cla.net.pl" }
      ]
    }
  ]
}
```

Response fields (from live response, cross-referenced with the successor app's config schema):

| Field | Type | Description |
|-------|------|-------------|
| `configVersion` | int | Schema version (currently `2`) |
| `cities[]` | array | One object per network |
| `cities[].id` | string | Network slug |
| `cities[].dataId` | string | Data id (usually equals `id`) |
| `cities[].clientCode` | string | 5-letter code — matches the codes used by the 2.x per-city services and the `Age` computation |
| `cities[].name` | string | Display name (city) |
| `cities[].environment` | string | Environment tag (observed `""`) |
| `cities[].operator` | string | Full operator name |
| `cities[].operatorShort` | string | Operator abbreviation |
| `cities[].region` | string | Polish region name |
| `cities[].regionI18n` | object | Localized region names (cs/de/en/es/fr/it/sk/uk) |
| `cities[].apiBaseUrl` | string | **Base URL of the successor "myBus 3" JSON API** (e.g. `…/api-mybus-3`) — out of scope of this document, but discovered here |
| `cities[].lat`, `lon` | number | Map center |
| `cities[].mapZoom` | int | Default zoom |
| `cities[].certificatePins` | array | TLS certificate pins for the new app |
| `cities[].modules` | object | Feature flags: `planner`, `vehiclesLive`, `messages`, `qr` |
| `cities[].cityFunctions[]` | array | Extra functions (`web-link` entries with `id`/`type`/`url`) |

27 networks were listed on the verification date.

---

## 6. Legacy global config service (`taran.com.pl`)

Extracted from `g4.b` ("ConfigurationService"):

```
GET http://www.taran.com.pl/myBusOnlineCfg/ConfigurationService.svc/GetMessage?cityCode={clientCode}
```

- **Parameters:** `cityCode` (string) — the 5-letter city code
- **Response:** `<msg c="…" s="…" m="…"/>` where the client reads `c` (string), `s` (int) and
  `m` (string). The client treats `s == 1` as "show the message `m`" — used to display
  server-side notices (e.g. maintenance) when the main city service is unreachable.
- No `Age`/`User-Agent` validation was implemented in the client for this service
  (`secToken = null`).
- **Status:** the host resolves (185.39.160.188) but refused connections from our test vantage
  point on 2026-11-17; the endpoint is documented from static analysis only.

---

## 7. Response data models

Wire format is XML with **attributes** (no elements/text for the data fields). Attribute names
are short abbreviations. Below, every attribute is documented with: type, meaning, and its
consumer in the decompiled client. "n/a (legacy)" = parsed by the client into model fields that
no UI code reads anymore.

### 7.1 Root elements

| Endpoint | Root | Attributes |
|----------|------|------------|
| `PingService` | `int` | — (text content = token) |
| `CompareScheduleFile` | `int` | — (text content = 0/1) |
| `GetTimeTableReal` | `Departures` | `time` (server time `HH:mm`), `i` (flag, observed `1`; not consumed by 2.8.13) |
| `GetDepartureInfo` | `D` | see 7.2 |
| `GetVehicles` | `VL` | — |
| `GetVehicleTimeTable` | `Schedules` | `id` (course instance date `YYYYMMDD`), `nr` (route number), `type` (day-type code), `o` (direction/destination) |
| `GetRouteVariantWithTransitPoints` | `R` | `r` (route number), `t` (day-type code) |
| `GetConnections` | `CC` | — |
| `GetDispatcherMessages` | `R` | — |
| `GetMessage` | `msg` | `c`, `s`, `m` (see section 6) |

### 7.2 Departure `<D>` (`GetTimeTableReal`, `GetDepartureInfo`)

Observed live (Lublin, Kielce, Łódź):

```xml
<D i="2180" di="453793_0" n="24007" t="52025" r="74" dd="P" d="DWORZEC LUBLIN"
   p="A" kn="" vr="201" m="2" v="3 min" vn="CKNE" iks="20261117" ip="1"/>
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `i` | int | Departure/course-instance id (the client uses it as `departure_id`) |
| `di` | string | Departure-instance id (consumed by the successor app; e.g. `453793_0`) |
| `n` | int | Unique trip id (`nUqTripId` for `GetDepartureInfo`); `0` when the vehicle is not yet live-tracked. Non-zero ⇒ the client shows the "by vehicle location" (GPS) icon |
| `t` | int | Scheduled departure time, **seconds since midnight** (e.g. `52025` = 14:27) |
| `r` | string | Route/line number |
| `d` | string | Direction (destination name) |
| `dd` | string | Direction-variant code of the course (observed `T`/`P`); used as the `cDirLst` filter value for `GetVehicles` |
| `p` | string | Vehicle type: `A` = bus, `R` = trolleybus, `T` = tram |
| `kn` | string | Course/block label (usually empty; observed `a`) |
| `vr` | int | **Real-time** seconds until departure (negative = already departed; only meaningful with live data) |
| `m` | int | Status: `1` = imminent/just departed (client renders red), `2` = live countdown (green), `3`/other = scheduled only (black) |
| `v` | string | Display time ready for UI: `"<1min"`, `"3 min"`, or `"15:22"` |
| `vn` | string | Vehicle amenity flags, one char each: `N` low floor, `K` air conditioning, `B` ticket machine, `Z` contactless card payment, `R` bicycle transport, `E` electric vehicle |
| `iks` | string | Course-instance key — service date `YYYYMMDD` in some cities, numeric id in others |
| `ip` | int | Flag (observed `1`; consumed by the successor app) |

`<N>` child of `Departures`: free-text notice shown when no departures are available.

### 7.3 Vehicle `<V>` (`GetVehicles`)

Observed live (Lublin):

```xml
<V id="890765674" nb="1858" nr="23" wt="J" kr="P" ik="1353" lp="25" dp="469" dw="473"
   x="22.5714" y="51.27551" px="22.5714" py="51.27551" o="31" s="1" p="13:44"
   op="KUPIECKA IKEA" c="GNZ" nk="1354" nnr="23" nwt="B" nkr="T" nop="FELIN UNIWERSYTET"
   is="0" vt="R" kwi=""/>
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `id` | long | Vehicle id |
| `nb` | int | Course number — pass as `nNb` to `GetVehicleTimeTable` |
| `nr` | string | Current route number |
| `wt` | string | Day-type/course code (observed `J`, `C`, `A`) — n/a (legacy) |
| `kr` | string | Direction-variant code (`P`/`T`) |
| `ik` | int | Course id (internal; accepted in `cKrsLst` filter) |
| `lp` | int | Progress counter along the course (stop ordinal) — n/a (legacy) |
| `dp`, `dw` | int | Progress metrics (observed small ints) — n/a (legacy) |
| `x`, `y` | decimal string | **Current position** (x = longitude, y = latitude; decimal comma possible) |
| `px`, `py` | decimal string | Auxiliary/predicted position used by the client to compute heading and marker animation |
| `o` | int | Deviation from schedule in seconds: negative = **delayed** (client draws a down-arrow "Delayed"), positive = **running early** (up-arrow "Speeded"), `0` = on time. Only shown when `nk` ≠ `0` |
| `s` | int | Status flag (observed `1`, `6`) — n/a (legacy) |
| `p` | string | Time `HH:mm` (last position update / schedule time) — n/a (legacy) |
| `op` | string | Direction/destination name shown as the vehicle's label |
| `c` | string | Vehicle amenity flags (same encoding as `vn` in 7.2) |
| `nk` | int | Next course id (`0` = none) |
| `nnr` | string | Next route number |
| `nwt` | string | Next day-type code |
| `nkr` | string | Next direction-variant code |
| `nop` | string | Next direction/destination |
| `is` | int | Flag (observed `0`, `134`) — n/a (legacy) |
| `vt` | string | Vehicle type: `A` bus, `R` trolleybus, `T` tram (drives the marker icon) |
| `kwi` | string | Service note/label (usually empty; shown when CFG `lVehicleService = Y`) |
| `nap` | int | Vehicle occupancy level, 0–100 (client renders a progress bar: green < 50, orange < 80, red ≥ 80; absent when unknown) |

### 7.4 Vehicle-schedule `<Stop>` (`GetVehicleTimeTable`)

Observed live (Lublin, course 1858):

```xml
<Schedules id="20261117" nr="23" type="J" o="KUPIECKA IKEA">
  <Stop lp="27" id="110" name="DOŻYNKOWA 01" th="" tm="&lt;1min" s="11" m="1"/>
  <Stop lp="28" id="1092" name="IKEA 03" th="" tm="2 min" s="131" m="2"/>
</Schedules>
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `lp` | int | Stop ordinal on the course |
| `id` | int | Stop id (`PRZYSTANKI.id`) |
| `name` | string | Stop name |
| `th` | string | Theoretical (scheduled) time — empty when live data is shown |
| `tm` | string | Time-to-arrival display text (`"<1min"`, `"2 min"`) |
| `s` | int | Seconds until arrival at this stop |
| `m` | int | Status (same encoding as 7.2 `m`) |

### 7.5 Route variant (`GetRouteVariantWithTransitPoints`)

```xml
<R r="18   " t="J">
  <T i1="1049" i2="1046">
    <Pkt l="1" x="22.57718" y="51.30494"/>
    <Pkt l="2" x="22.57718" y="51.30488"/>
  </T>
  …
</R>
```

| Element | Attribute | Type | Description |
|---------|-----------|------|-------------|
| `R` | `r` | string | Route number |
| `R` | `t` | string | Day-type code |
| `T` | `i1`, `i2` | int | Stop ids bounding this track segment |
| `Pkt` | `l` | int | Point ordinal |
| `Pkt` | `x`, `y` | decimal | Longitude / latitude of the polyline point |

### 7.6 Connection `<C>` + leg `<G>` (`GetConnections`)

Observed live (Kielce):

```xml
<CC>
  <C gs="55920" ge="58500" js="56040" je="58380" jdr="2340" tdr="1560" wdr="240"
     tln="7911" wln="244" ss="3683">
    <G i="0"   n=""              g="142200" dg="2026-11-17" o="0" x="20.628"   y="50.866"
       w="1" cw="120" t="0" r=""  v="A" uti="0"      vn="" dem="0">
      <PP><P x="20.628" y="50.866"/> … </PP>
    </G>
    <G i="426" n="Jana Pawła II" g="142440" dg="2026-11-17" o="0" x="20.62842" y="50.86506"
       w="0" cw="0"   t="1" r="54" v="A" uti="6373224" vn="" dem="0">
      <PP><P x="20.62837" y="50.86507"/> … </PP>
    </G>
    …
  </C>
</CC>
```

`<C>` — one journey alternative:

| Attribute | Type | Description |
|-----------|------|-------------|
| `gs`, `ge` | int | Journey start / end, seconds since midnight |
| `js`, `je` | int | First/last transit leg time (observed; consumed by successor app) |
| `jdr` | int | Total journey duration, seconds |
| `tdr` | int | In-vehicle duration, seconds |
| `wdr` | int | Walking duration, seconds |
| `tln` | int | Transit distance (meters, inferred from magnitude) |
| `wln` | int | Walking distance (meters, inferred from magnitude) |
| `ss` | int | Slack/spare time, seconds (inferred) |

`<G>` — leg point (one per boarding/alighting/intermediate point; consecutive points with the
same `uti` form one transit leg):

| Attribute | Type | Description |
|-----------|------|-------------|
| `i` | int | Stop id (`0` for a free/non-stop point) |
| `n` | string | Stop name |
| `g` | int | Time at this point, seconds since midnight **plus n·86400** for after-midnight legs (the client displays `g % 86400`) |
| `dg` | string | Service date `yyyy-MM-dd` — pass as `cDate` to `GetDepartureInfo` |
| `o` | int | Ordinal/flag (observed `0`) |
| `x`, `y` | decimal | Longitude / latitude |
| `w` | int | `1` = walking leg |
| `cw` | int | Wait/walk duration at this point, seconds |
| `t` | int | `1` = transit (ride) point, `0` = walk point |
| `r` | string | Route number (empty on walking legs) |
| `v` | string | Vehicle type (`A`/`R`/`T`) |
| `uti` | long | Unique trip id — pass as `nUqTripId` to `GetDepartureInfo` (`0` on walking legs) |
| `vn` | string | Vehicle label/number |
| `dem` | int | `1` = request/flag stop (na żądanie) |

`<PP>` — point-to-point geometry between this leg point and the next (`<P x y>` = lon/lat).

### 7.7 Dispatcher message (`GetDispatcherMessages`)

```xml
<R>
  <M i="123" o="1" t="Title of the message" m="Body of the message"/>
</R>
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `i` | int | Message id (client deduplicates notifications by id) |
| `o` | int | Ordinal/sort key |
| `t` | string | Title (when absent, the client generates "Dispatcher news #N") |
| `m` | string | Message body |

(Element name observed as the first child of `<R>`; the client reads all children generically.
No live sample with content was available on the verification date.)

---

## 8. Schedule file format (gzipped SQLite)

`GetScheduleFile` returns a **gzip stream** (magic `1f 8b`) whose payload is a standard
**SQLite 3** database. Verified tables (Lublin, 2026-11-17):

| Table | Content (inferred from client SQL) |
|-------|-----------------------------------|
| `WERSJE` | Schedule versioning: `id_wersja`, (validity date), `generacja` — feed these into `CompareScheduleFile` |
| `CFG` | Feature flags: `lTripPlaner` (`Y`/`N` — enables `GetConnections`), `lStreets`, `lVehicleNotes`, `lVehicleSideNumber`, `lVehicleVariance`, `nTripPlanerVersion` |
| `DNI` | Day-type calendar: code (`PN` workday, `WN` Saturday, `SN` Sunday, holiday codes…), name, sort |
| `KALENDARZ` | Calendar of day types by date |
| `KIERUNKI` | Route directions: `numer`, `transport`, `podstawowy`, `kod` |
| `PRZYSTANKI` | Stops: `id`, `id_ul`, `nazwa`, `numer`, `slupek`, `odjazdy`, `lon`, `lat`, `transport`, `linieA`/`linieT`/`linieR` (routes by mode), `sort`, `kierunek`, `typ_tab` |
| `ODJAZDY` | Departure times per stop/route/course |
| `PUNKTY` | Track geometry points |
| `ULICE` | Street names (`id`, `nazwa`) |
| `UWAGI` | Route footnotes (`ozn_uwagi`, `tresc_uwag`, `numer_lini`) |

Sample row (`PRZYSTANKI`, Lublin):

```
id=1, nazwa='OGRÓD SASKI 01', numer='1001', lon=22.55166, lat=51.24715, transport='A',
linieA='4,8,13,15,20,32,40,44,74,200,201,302,950,N2,N3', kierunek='ŻEGLARSKA, ABRAMOWICE'
```

---

## 9. City registry (54 networks)

Extracted verbatim from the APK's built-in registry (`com.taran.mybus.g`). Each row is an
independent deployment of the same API.

| # | City (operator) | clientCode | Service base URL | Lang |
|---|---|---|---|---|
| 1 | BIAŁA PODLASKA (MZK) | `BIALA` | `http://rozklad.mzkbp.pl/AndroidService/SchedulesService.svc` | pl |
| 2 | BOLESŁAWIEC (MZK) | `BOLES` | `http://83.13.186.174:8081/AndroidService/SchedulesService.svc` | pl |
| 3 | CHEŁM (CLA) | `CHELM` | `http://rozklad.cla.net.pl/myBusServices/SchedulesService.svc` | pl |
| 4 | CIESZYN (ZGK) | `CIESZ` | `http://rozklad.zgk.cieszyn.pl/myBusServices/SchedulesService.svc` | pl |
| 5 | DĘBICA (MKS) | `DEBIC` | `http://rj.mks.debica.pl/AndroidService/SchedulesService.svc` | pl |
| 6 | ELBLĄG (ZKM) | `ELBLA` | `http://www.rozklad.zkm.elblag.com.pl/AndroidService/SchedulesService.svc` | pl |
| 7 | GDAŃSK (GAiT) | `GDANS` | `http://info.gait.pl/SchedulesService/SchedulesService.svc` | pl |
| 8 | GŁOGÓW (KM) | `GLOGO` | `http://89.151.3.5/myBusServices/SchedulesService.svc` | pl |
| 9 | INOWROCŁAW (MPK) | `INOWR` | `http://bus.inowroclaw.pl/myBusServices/SchedulesService.svc` | pl |
| 10 | JASTRZĘBIE-ZDRÓJ (MZK) | `JASTR` | `https://bilet.mzkjastrzebie.com:8088/myBusServices/SchedulesService.svc` | pl |
| 11 | JELENIA GÓRA (MZK) | `JELGO` | `http://www.seba.mzk.jgora.pl:8080/myBusServices/SchedulesService.svc` | pl |
| 12 | KIELCE (ZTM) | `KIELC` | `http://sip.ztm.kielce.pl/AndroidService/SchedulesService.svc` | pl |
| 13 | KOLEJE MAŁOPOLSKIE (KMŁ) | `KOMAL` | `https://mybus.kolejemalopolskie.com.pl/SchedulesService.svc` | pl |
| 14 | KOŁOBRZEG (KM) | `KOLOB` | `http://89.239.105.31/myBusServices/SchedulesService.svc` | pl |
| 15 | KRAŚNIK (MPK) | `KRASN` | `http://176.104.119.244/myBusServices/SchedulesService.svc` | pl |
| 16 | KROSNO (MKS) | `KROSN` | `http://rozklad.mks-krosno.pl/myBusServices/SchedulesService.svc` | pl |
| 17 | KUTNO (MZK) | `KUTNO` | `http://rozklad.km.kutno.pl/myBusServices/SchedulesService.svc` | pl |
| 18 | KĘDZIERZYN KOŹLE (MZK) | `KKOZL` | `http://79.188.1.197/SchedulesService.svc` | pl |
| 19 | LEGNICA (MPK) | `LEGNI` | `http://mybus.autobusy.legnica.eu/SchedulesService.svc` | pl |
| 20 | LESZNO (MZK) | `LESZN` | `http://rozklad.mzk.leszno.pl/myBusServices/SchedulesService.svc` | pl |
| 21 | LUBLIN (ZTM) | `LUBLI` | `http://sip.ztm.lublin.eu/AndroidService/SchedulesService.svc` | pl |
| 22 | MIELEC (MKS) | `MIELE` | `http://mapa.mks-mielec.pl/AndroidService/SchedulesService.svc` | pl |
| 23 | OLSZTYN (ZDZiT) | `OLSZT` | `http://sip.zdzit.olsztyn.eu/myBusServices/SchedulesService.svc` | pl |
| 24 | OSTROWIEC ŚWIĘTOKRZYSKI (MZK) | `OSTSW` | `http://rozklad.mzkostrowiec.pl/myBusServices/SchedulesService.svc` | pl |
| 25 | OSTROŁĘKA (MZK) | `OLEKA` | `http://rozklad.mzk.ostroleka.pl/myBusServices/SchedulesService.svc` | pl |
| 26 | OSTRÓW WIELKOPOLSKI (MZK) | `OSTRO` | `http://185.22.8.43/myBusServices/SchedulesService.svc` | pl |
| 27 | PABIANICE (MZK) | `PABIA` | `http://mybus.komunikacjapabianice.pl/SchedulesService.svc` | pl |
| 28 | POLKOWICE (ZKM) | `POLKO` | `http://zkmp.com.pl/myBusServices/SchedulesService.svc` | pl |
| 29 | PRZEMYŚL (MZK) | `PRMZK` | `http://rozklad.mzk.przemysl.pl/myBusServices/SchedulesService.svc` | pl |
| 30 | PUŁAWY (MZK) | `PULAW` | `http://mapa.mzk.pulawy.pl/AndroidService/SchedulesService.svc` | pl |
| 31 | PŁOCK (KM) | `PLOCK` | `https://rozkladjazdy.kmplock.eu/myBusServices/SchedulesService.svc` | pl |
| 32 | RADOM (MZDiK) | `RADOM` | `http://31.11.251.95/myBusServices/SchedulesService.svc` | pl |
| 33 | RADOMSKO (MPK) | `RDMSK` | `http://mpk.mpk-radomsko.pl:8081/SchedulesService.svc` | pl |
| 34 | RYBNIK (KM) | `RYBNI` | `http://rozklad.km.rybnik.pl/myBusServices/SchedulesService.svc` | pl |
| 35 | RZESZÓW (ZTM) | `RZZTM` | `https://mybus.erzeszow.pl/myBusServices/SchedulesService.svc` | pl |
| 36 | SANOK (SPGK) | `SANOK` | `http://rozkladjazdy.spgk.com.pl/myBusServices/SchedulesService.svc` | pl |
| 37 | SIEDLCE (MPK) | `SIEDL` | `https://rozklad.mpk.siedlce.pl/myBusServices/SchedulesService.svc` | pl |
| 38 | STALOWA WOLA (MZK) | `STALO` | `https://mybus.mzk.stalowa-wola.pl/SchedulesService.svc` | pl |
| 39 | STARACHOWICE (ZEC) | `STRCH` | `http://rozklad-jazdy.zecstar.pl/myBusServices/SchedulesService.svc` | pl |
| 40 | SUWAŁKI (PGK) | `SUWAL` | `http://89.22.39.6/AndroidService/SchedulesService.svc` | pl |
| 41 | SŁUPSK (ZIM) | `ZIMSL` | `http://176.97.80.62/myBusServices/SchedulesService.svc` | pl |
| 42 | TABOR (COMETT PLUS) | `TABOR` | `http://www.mhd-tabor.comettplus.cz/myBusServices/SchedulesService.svc` | cs |
| 43 | TARNOWSKIE GÓRY (MZKP) | `NOWAK` | `http://157.25.157.76/myBusServices/SchedulesService.svc` | pl |
| 44 | TARNÓW (ZDiK) | `TARNO` | `http://80.85.231.251/myBusServices/SchedulesService.svc` | pl |
| 45 | TCZEW (Gryf) | `TCZEW` | `http://www.rozklady.tczew.pl/myBusServices/SchedulesService.svc` | pl |
| 46 | WAŁBRZYCH (Gmina Wałbrzych) | `WALBR` | `http://rozklad.walbrzych.eu/myBusServices/SchedulesService.svc` | pl |
| 47 | ZAMOŚĆ (MZK) | `ZAMOS` | `http://37.109.29.250/myBusServices/SchedulesService.svc` | pl |
| 48 | ŁOWICZ (MZK) | `LOWIC` | `http://rozklad.mzklowicz.pl/myBusServices/SchedulesService.svc` | pl |
| 49 | ŁÓDŹ (ZDiT) | `LODZ ␠` | `http://rozklady.lodz.pl/myBusServices/SchedulesService.svc` | pl |
| 50 | ŚWIDNICA (MPK) | `SWIDN` | `http://www.grj.mpk.swidnica.pl/AndroidService/SchedulesService.svc` | pl |
| 51 | ŚWIEBODZICE (ZGK) | `SWIEB` | `http://213.108.80.2:8085/SchedulesService.svc` | pl |
| 52 | ŠIAULIAI (Busturas) | `SIAUL` | `http://kis.busturas.lt/myBusServices/SchedulesService.svc` | pl |
| 53 | ŻORY (BKM) | `ZORY ␠` | `https://master-pk-busy-cpz.autobusy.zory.pl/SchedulesService.svc` | pl |
| 54 | ŽILINA (DPMZ) | `ZILIN` | `https://www.mybus.dpmz.sk/myBusServices/SchedulesService.svc` | sk |

Notes:

- **Important:** the clientCode must be used **exactly as registered** — `LODZ ␠` (Łódź) and
  `ZORY ␠` (Żory) end with a real trailing space, which feeds the `Age` checksum:
  `"LODZ "` sums to 345 while `"LODZ"` sums to 313; only the former is accepted.
  (`␠` marks the space; `cities.json` carries the raw values.)
- The registry also stores each city's map-center coordinates (`lat`/`lng`) and default language —
  see `cities.json` in this repository for the machine-readable version.
- Availability differs: some hosts are HTTP-only, some HTTPS; some (e.g. Gdańsk/GAiT) sit behind a
  WAF that may reject non-mobile IPs; several HTTPS hosts timed out from our test network.
- The central config API (section 5) additionally lists which of these networks are already served
  by the successor "myBus 3" app (27 networks on 2026-11-17).

---

## 10. Code examples

### 10.1 curl

```bash
#!/usr/bin/env bash
# myBus 2.x API demo — live departures for a Lublin stop
BASE="http://sip.ztm.lublin.eu/AndroidService/SchedulesService.svc"
CODE="LUBLI"   # clientCode; the Age checksum adds the Unicode code points of this string

# 1) handshake: get the server token (bootstrap value 60 in the Age header)
TOKEN=$(curl -s -H "User-Agent: myBusOnline" \
     -H "Age: $((60 + $(python3 -c "print(sum(map(ord,'$CODE')))")) )" \
     "$BASE/PingService" | sed -n 's/.*<int>\([0-9]\+\)<\/int>.*/\1/p')
echo "server token: $TOKEN"

# 2) fetch the real-time departure board for stop 1
AGE=$((TOKEN + $(python3 -c "print(sum(map(ord,'$CODE')))")))
curl -s -H "User-Agent: myBusOnline" -H "Age: $AGE" \
     "$BASE/GetTimeTableReal?nBusStopId=1&nBusStopGroupId=0"

# 3) live vehicle positions (whole network)
curl -s -H "User-Agent: myBusOnline" -H "Age: $AGE" \
     "$BASE/GetVehicles?cNbLst=&cIdLst=&cRouteLst=&cTrackLst=&cDirLst=&cKrsLst="

# 4) download + unpack the full static timetable database (gzip -> SQLite)
curl -s -H "User-Agent: myBusOnline" -H "Age: $AGE" \
     -H "Accept-Encoding: identity" "$BASE/GetScheduleFile" -o schedule.gz
gunzip -c schedule.gz > schedule.db && sqlite3 schedule.db 'select * from PRZYSTANKI limit 5'
```

### 10.2 Python

```python
"""Minimal myBus 2.x client (verified against production on 2026-11-17)."""
import re
import gzip
import sqlite3
import urllib.request
import urllib.parse

CITY_URL, CITY_CODE = "http://sip.ztm.lublin.eu/AndroidService/SchedulesService.svc", "LUBLI"
CHARSUM = sum(map(ord, CITY_CODE))          # "Age" checksum component
UA = "myBusOnline"                          # mandatory user agent


def call(endpoint, params=None, token=60):
    url = f"{CITY_URL}/{endpoint}"
    if params:
        url += "?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={
        "User-Agent": UA,
        "Age": str(token + CHARSUM),        # handshake token + checksum
    })
    with urllib.request.urlopen(req, timeout=20) as r:
        return r.read()


# 1) handshake -> server token
token = int(re.search(rb"<int>(\d+)</int>", call("PingService")).group(1))

# 2) real-time departures for stop 1
xml = call("GetTimeTableReal",
           {"nBusStopId": 1, "nBusStopGroupId": 0}, token).decode()
for d in re.finditer(r'<D ([^>]+)/>', xml):
    attrs = dict(re.findall(r'(\w+)="([^"]*)"', d.group(1)))
    print(f"{attrs['v']:>8}  line {attrs['r']:>4} -> {attrs['d']}"
          f"  ({attrs['vn'] or 'no flags'})")

# 3) full static timetable database (gzip -> SQLite)
blob = call("GetScheduleFile", token=token)
open("schedule.db", "wb").write(gzip.decompress(blob))
con = sqlite3.connect("schedule.db")
print(con.execute("select count(*) from PRZYSTANKI").fetchone())   # -> 1486 stops
```

### 10.3 JavaScript (Node 18+ / Deno / browser via CORS proxy)

```js
// myBus 2.x client (verified 2026-11-17) — Node 18+
const CITY_URL = "http://sip.ztm.lublin.eu/AndroidService/SchedulesService.svc";
const CITY_CODE = "LUBLI";
const CHARSUM = [...CITY_CODE].reduce((s, c) => s + c.codePointAt(0), 0);

async function call(endpoint, params = {}, token = 60) {
  const qs = new URLSearchParams(params).toString();
  const res = await fetch(`${CITY_URL}/${endpoint}${qs ? "?" + qs : ""}`, {
    headers: {
      "User-Agent": "myBusOnline",          // required
      "Age": String(token + CHARSUM),       // handshake token + checksum
    },
  });
  if (!res.ok) throw new Error(`${res.status} ${await res.text()}`);
  return res;
}

// 1) handshake
const ping = await (await call("PingService")).text();
const token = Number(ping.match(/<int>(\d+)<\/int>/)[1]);

// 2) departures for stop 1
const board = await (await call("GetTimeTableReal",
  { nBusStopId: 1, nBusStopGroupId: 0 }, token)).text();

// 3) parse <D .../> attributes
const departures = [...board.matchAll(/<D ([^>]+)\/>/g)].map(m =>
  Object.fromEntries([...m[1].matchAll(/(\w+)="([^"]*)"/g)].map(a => [a[1], a[2]])));
console.table(departures.map(d => ({
  time: d.v, line: d.r, to: d.d, flags: d.vn, live: d.n !== "0",
})));
```

> **Browser caveat:** the city endpoints are plain HTTP and send no CORS headers, so direct calls
> from a web page will be blocked; use them from Node/Deno or a server-side proxy.

Runnable copies of these examples are included in `examples/`.

---

## 11. Live verification log

All requests were read-only `GET`s against public timetable endpoints, executed 2026-11-17
(UTC+1, daytime). Full raw samples are in `samples/`.

| # | Endpoint | City | Result |
|---|----------|------|--------|
| 1 | `GET /api/v1/config/cities` (central) | — | `200` JSON, 27 networks, `configVersion 2` |
| 2 | `PingService` | Mielec, Lublin, Kielce, Łódź | `200 <int>2054</int>` (all four cities returned the same token) |
| 3 | `PingService` with wrong/missing `Age` or missing `User-Agent` | Lublin | `400 Unauthorized access` (10/10 malformed variants rejected) |
| 4 | `GetDispatcherMessages` | Mielec, Lublin, Kielce, Łódź | `200 <R/>` (no active messages) |
| 5 | `GetTimeTableReal` (stop 1) | Lublin, Kielce, Mielec, Łódź | `200` with live departures (countdowns, vehicle flags, live trip ids) |
| 6 | `GetVehicles` (no filters) | Lublin | `200`, 32 vehicles with positions, delays, directions, next-trip info |
| 7 | `GetVehicleTimeTable` (`nIdKursu=0`, `nNb=<nb>`) | Lublin | `200` — 4 upcoming stops with `tm`/`s`/`m` countdowns |
| 8 | `GetRouteVariantWithTransitPoints` (`nIdKursu=1858`) | Lublin | `200` — 12,134 bytes of `<T>`/`<Pkt>` geometry |
| 9 | `GetDepartureInfo` (`cDate`/`nBusStopId`/`nUqTripId` from a connections leg) | Kielce | `200 <D i="2846" … v="15:34"/>` |
| 10 | `GetConnections` (city center → Kadzielnia, 15:30) | Kielce | `200` — 60,805 bytes, 104 transit legs |
| 11 | `GetConnections` | Lublin | `400` WCF request-error page — explained by the city's `CFG.lTripPlaner='N'` |
| 12 | `CompareScheduleFile(4597,1)` / `(4597,2)` / `(4596,1)` / `(0,0)` | Lublin | `1` / `0` / `0` / `0` — round-trip against `WERSJE` of the downloaded DB |
| 13 | `GetScheduleFile` | Lublin | `200` — 954,726 B gzip → 5,419,008 B SQLite, 10 tables, 1,486 stops |
| 14 | `PingService` | Gdańsk (GAiT) | `403 Forbidden` HTML (WAF; server is up but filters our test IP) |
| 15 | `PingService` | Płock, Rzeszów, Koleje Małopolskie | connection timeout from the test network |
| 16 | `GetMessage` (legacy `taran.com.pl`) | — | connection refused (host resolves to 185.39.160.188) |
| 17 | `PingService` with `clientCode "LODZ "` vs `"LODZ"` | Łódź | `200` with the trailing space (checksum 345) · `400` without it (313) — proves exact-code requirement |

---

## 12. Undocumented-but-observed fields

These attributes appear in live responses but are **not parsed by the 2.8.13 client** — they were
added for the successor app (`pl.rg.mybus`, "myBus 3"), whose JSON API base URLs are advertised by
the central config endpoint (section 5):

- `Departures@i`, `D@di`, `D@iks`, `D@ip`, `D@kn`
- `C@js`, `C@je` (journey start/end) in `GetConnections`
- `V@wt`, `V@kr`, `V@ik`, `V@lp`, `V@dp`, `V@dw`, `V@s`, `V@p`, `V@is` are parsed by 2.8.13 but
  not displayed ("legacy" in section 7.3)

If you need authoritative semantics for these, decompile the successor APK (`pl.rg.mybus`).

---

## 13. Legal & ethical notes

- **Provenance:** everything in this document comes from (a) static analysis of the publicly
  distributed APK `myBus (2).apk` (`com.taran.mybus` 2.8.13) and (b) read-only `GET` requests to
  production endpoints, identical to the traffic the app itself generates. No credentials,
  accounts, or access controls were bypassed — the only "protection" is a fixed user-agent string
  and an integer checksum, both shipped in the public binary.
- **Public-data context:** the payloads are public-transit timetables, stop inventories, and
  aggregate vehicle positions of municipal/regional operators. In Poland, timetables of public
  transit organized by public authorities are widely regarded as **public information** under the
  Act on Access to Public Information (ustawa o dostępie do informacji publicznej, 6 Sep 2001),
  and many operators additionally publish them as open data. This documentation is intended to
  support interoperability and civic/open-data use; it is not affiliated with, endorsed by, or
  derived from any confidential source of R&G PLUS Sp. z o.o.
- **Use responsibly:** the servers belong to public institutions. Cache aggressively
  (the schedule file changes at most a few times per day), keep polling rates comparable to the
  official app (15–30 s for departure boards; ~1 s only while actively tracking a vehicle),
  set a descriptive `User-Agent` of your own on top of the required one if your platform allows
  it, and prefer the official open-data channels where a city provides them.
- **No warranty:** this is an unofficial description of an undocumented interface that may change
  or break at any time. Field semantics marked "inferred" are best-effort interpretations, not
  guarantees.

---

*Generated 2026-11-17 from `myBus (2).apk` (`com.taran.mybus` 2.8.13, versionCode 161).*
