# Open APIs Without Credentials

## 1. Weather EU

| API | URL | Details |
|-----|-----|---------|
| **Open-Meteo** | `https://api.open-meteo.com/v1/forecast` | Fully free, no API key, no registration. 1–11 km resolution, forecasts up to 16 days, 80+ years historical data. Integrates DWD ICON, MeteoFrance, ECMWF IFS and more. JSON responses, CORS supported, <10 ms response times. Open-source (AGPLv3). |
| **Bright Sky** | `https://api.brightsky.dev` | Free JSON API wrapping DWD (German Weather Service) open data. No key required. Includes weather alerts endpoint. |

## 2. German Taxi Organizations

**No truly open/keyless APIs exist from German taxi organizations.**

Major German taxi services and their status:

- **taxi.eu / Taxi Deutschland** — No public API. Consumer-facing booking app only.
- **FREENOW (formerly MyTaxi)** — API access restricted to business partners (fleet operators, dispatchers). Not publicly available.
- **Taxi fare estimation** — Available via RapidAPI, but requires a (free) RapidAPI key.

Closest alternatives without credentials for German transport data:

| API | URL | Details |
|-----|-----|---------|
| **Deutschland API** | `https://deutschland-api.dev/` | Aggregates German public data (cities, districts, transport infrastructure). No key. Not taxi-specific. |
| **DB Transport REST** | `https://v6.db.transport.rest/` | Deutsche Bahn transport data, no key, 100 req/min. |

## 3. Weather Warnings

| API | URL | Details |
|-----|-----|---------|
| **MeteoAlarm API** | `https://api.meteoalarm.org/` | Free EDR API for weather warnings across 33 European countries. OGC-compliant, GeoJSON format. Covers wind, rain, flooding, snow, thunderstorms, fog, extreme temps, coastal events, forest fires, avalanches, droughts, marine hazards. Also offers real-time MQTT feed at `mqtts://api.meteoalarm.org`. |
| **Bright Sky (Alerts)** | `https://api.brightsky.dev/alerts` | DWD weather alerts for Germany. No key required. JSON format. |
| **DWD Open Data** | `https://opendata.dwd.de` | Raw open data from Deutscher Wetterdienst including warning data. Free access. |
| **DWD GeoServer** | WMS/WFS services | OGC-compliant map/feature services including warning layers. No key. |

## 4. Bicycle Leasing / Bike-Sharing Services

| API | URL | Details |
|-----|-----|---------|
| **CityBikes API** | `https://api.citybik.es/v2/networks` | Free, no auth required. Real-time bike-sharing data across Europe. 300 req/hour without key. Supports GBFS v2.3 and v3.0. |
| **GBFS Feeds (via CityBikes)** | `https://api.citybik.es/gbfs/3/manifest.json` | GBFS standard feeds — vehicle/station locations, availability, pricing. Designed to be open with zero authentication by spec. |

APIs that require minimal (free) registration:

- **Call-a-Bike / Flinkster (Deutsche Bahn)** — Free unlimited API key after registration at `developer.deutschebahn.com`.
- **nextbike** — Requires API key.
- **JobRad Connect** — Corporate bike leasing, requires auth.

## 5. Vehicle Rentals / Carsharing / Scooter Sharing

### No key, no registration required

| API | Type | URL / Access | Details |
|-----|------|-------------|---------|
| **Share Now (MQTT)** | Carsharing | `mqtts://driver.eu.share-now.com:443` | Anonymous MQTT connection (clientId: random UUID prefixed with `a:`). No username/password needed. Real-time locations and availability of Share Now vehicles across Europe. Community project: `github.com/T3rm1/share-now-api`. |
| **GBFS Public Feeds** | Multi-vehicle (bikes, scooters, mopeds, cars) | Catalog: `https://raw.githubusercontent.com/MobilityData/gbfs/master/systems.csv` | 800+ shared mobility systems worldwide. GBFS spec mandates no authentication. Includes operators like Dott (scooters), nextbike (bikes), and others. Each system publishes a `gbfs.json` auto-discovery URL. |
| **CityBikes API** | Bike/Scooter sharing | `https://api.citybik.es/v2/networks` | No auth, 300 req/hour. Real-time vehicle/station data across Europe. |
| **Hamburg Carsharing Open Data** | Carsharing | Via EU open data portal | Free access to carsharing station data (Cambio provider). |

### Free registration required (key is free and unlimited)

| API | Type | URL | Details |
|-----|------|-----|---------|
| **Flinkster / Call-a-Bike (DB)** | Carsharing + Bikes | `developer.deutschebahn.com` | Free unlimited API key after registration. Covers carsharing (Flinkster) and bike rental (Call-a-Bike) in Germany, Netherlands, Austria. |
| **Sixt Developer Portal** | Car Rental | `developers.sixt.com` | RENT API + SHARE API. Requires developer registration. |

### Partner access only (not publicly available)

- **TIER** — E-scooters, e-bikes, e-mopeds. Requires `X-Api-Key`.
- **Voi** — E-scooters. Requires `X-Auth-Token` and partner approval.
- **FREENOW** — Taxi + multi-mobility. Business partners only.
- **Bolt** — No public API.
- **Lime** — No public API.
