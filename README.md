# IRONSIGHT — AI Intelligence Layer Implementation Plan
**Prepared for:** Al Jazeera Arabic Open Source Monitoring Department  
**Date:** 2026-06-08  
**Status:** Pre-implementation design document  
**Scope:** Additive AI layer on top of existing IRONSIGHT codebase — zero modification to existing rendering pipeline

---

## Section 1 — Codebase Audit

### 1.1 Data Sources Inventory

#### Source 1: RSS News Feed
**File:** `src/app/api/news/route.ts`  
**Function:** `GET()` → calls `fetchRSS()` for each feed, then `isRelevant()`, then Hebrew translation  
**26 feeds ingested:**

| Source | URL | Trust Tier |
|--------|-----|-----------|
| BBC | feeds.bbci.co.uk/news/world/middle_east/rss.xml | Tier 2 |
| NYT | rss.nytimes.com/services/xml/rss/nyt/MiddleEast.xml | Tier 2 |
| Al Jazeera | aljazeera.com/xml/rss/all.xml | Tier 2 |
| Reuters | feeds.reuters.com/Reuters/worldNews | Tier 1 |
| Times of Israel | timesofisrael.com/feed/ | Tier 3 |
| JPost | jpost.com/rss/rssfeedsfrontpage.aspx | Tier 3 |
| Ynet | ynetnews.com/Integration/StoryRss2.xml | Tier 3 |
| N12 | rcs.mako.co.il/rss/news-military.xml | Tier 3 |
| Walla | rss.walla.co.il/feed/22 | Tier 3 |
| The National | thenationalnews.com/arc/outboundfeeds/rss/ | Tier 3 |
| CNN | rss.cnn.com/rss/edition_meast.rss | Tier 2 |
| Fox News | moxie.foxnews.com/google-publisher/world.xml | Tier 3 |
| WSJ | feeds.content.dowjones.io/public/rss/RSSWorldNews | Tier 2 |
| Google News (3 queries) | news.google.com/rss/search | Tier 3 aggregator |
| Breaking Defense | breakingdefense.com/feed/ | Tier 3 |
| Long War Journal | longwarjournal.org/feed | Tier 3 |
| Military Times | militarytimes.com/arc/outboundfeeds/rss/ | Tier 3 |
| War on Rocks | warontherocks.com/feed/ | Tier 3 |
| CENTCOM | centcom.mil/DesktopModules/ArticleCS/RSS.ashx | Tier 1 |
| DoD | defense.gov/DesktopModules/ArticleCS/RSS.ashx | Tier 1 |
| Haaretz (2 feeds) | haaretz.com/srv/... | Tier 3 |
| Drop Site News | dropsitenews.com/feed | Tier 3 |
| PressTV (3 feeds) | presstv.ir/rss.xml | Tier 3 |

**Output shape:**
```typescript
interface NewsItem {
  title: string;
  link: string;
  source: string;   // human-readable name, e.g. "Reuters"
  pubDate: string;  // RSS date string
  category?: string;
}
```
**Existing processing:** `isRelevant()` filters by `RELEVANCE_KEYWORDS` regex; Hebrew titles translated via unofficial Google Translate endpoint; deduplication by first 60 chars of lowercased title; sorted by proximity to now(); capped at 100 items.  
**Poll interval:** 90,000ms — set in `NewsFeed.tsx` line 30.

---

#### Source 2: Telegram OSINT
**File:** `src/app/api/telegram/route.ts`  
**Function:** `GET()` → `findLatestPostId()` → `fetchPost()` × 3 per channel  
**31 channels monitored:**

| Channel | Label | Affiliation |
|---------|-------|-------------|
| IDFofficial | IDF Official | Israeli military |
| RocketAlert | Rocket Alert | Alert aggregator |
| PressTV | PressTV (Iran) | Iranian state media |
| OSINTdefender | OSINT Defender | OSINT neutral |
| middle_east_spectator | ME Spectator | OSINT neutral |
| iranintl_en | Iran Intl | Opposition/diaspora |
| Alertisrael | Alert Israel | Alert aggregator |
| QudsNen | Quds News | Pro-Palestinian |
| TimesofIsrael | Times of Israel | Israeli media |
| FarsNews_EN | Fars News | Iranian state media |
| FotrosResistancee | Fotros Resist. | Pro-resistance |
| Alsaa_plus_EN | Al-Saa EN | Regional |
| warfareanalysis | Warfare Analysis | OSINT neutral |
| rnintel | RN Intel | OSINT neutral |
| GeoPWatch | GeoPol Watch | OSINT neutral |
| thecradlemedia | The Cradle | Pro-resistance |
| HAMASW | Hamas-Israel War | Conflict tracker |
| TasnimNewsEN | Tasnim News | Iranian state media |
| AbuAliExpress | Abu Ali Express | Pro-resistance OSINT |
| dropsitenews | Drop Site News | Independent |
| france24_en | France 24 | Tier 2 |
| SaberinFa | Saberin (IRGC) | IRGC-affiliated |
| defapress_ir | DefaPress (Iran MOD) | Iranian MOD |
| sepah | IRGC Official | IRGC official |
| wamnews_en | WAM (UAE) | UAE state media |
| gulfnewsUAE | Gulf News | UAE media |
| Alibk3 | Ali Bk | OSINT |
| aljazeeraglobal | Al Jazeera | Tier 2 |
| bintjbeilnews | Bint Jbeil | Lebanon local |
| kianmeli1 | Kian Meli (Iran) | Iranian |

**Output shape:**
```typescript
interface TelegramPost {
  channel: string;        // channel name slug
  channelLabel: string;   // human-readable
  color: string;          // display color hex
  postId: number;
  text: string;           // cleaned, translated if non-Latin
  date: string;           // ISO 8601
  url: string;            // t.me link
}

interface TelegramResponse {
  posts: TelegramPost[];
  channels: string[];
  updated: string;
}
```
**Server-side in-memory state:** `latestKnownIds: Record<string, number>` and `postCache: Record<string, {text, date}>` — both persist across requests within the same Next.js server process. This is important: the intelligence layer **cannot rely** on these being populated on cold start.  
**Poll interval:** 60,000ms — set in `TelegramPanel.tsx` line 22.

---

#### Source 3: Pikud HaOref (Israel Alert Status)
**File:** `src/app/api/alerts/route.ts`  
**Function:** `GET()` — fetches `api.tzevaadom.co.il/notifications`, translates Hebrew, manages sticky cache  

**Output shape:**
```typescript
interface AlertEvent {
  id: string;
  time: string;
  type: string;             // 'MISSILE' | 'ROCKET' | 'DRONE' | 'MORTAR' | 'INFILTRATION' | 'EARTHQUAKE' | 'TSUNAMI' | 'HAZMAT' | 'ALERT'
  threat: string;           // translated English threat description
  threatOriginal: string;   // Hebrew original
  locations: string[];      // translated English city names
  locationsOriginal: string[]; // Hebrew originals
  source: string;           // 'Pikud HaOref'
  active: boolean;
}

interface AlertResponse {
  status: 'ACTIVE' | 'CLEAR';
  activeCount: number;
  alerts: AlertEvent[];
  lastChecked: string;
  source: string;
}
```
**Server-side state:** `stickyAlerts` array with 90s persistence; alerts remain visible for 90 seconds after clearing from the API.  
**Cache-Control:** `public, s-maxage=5` — data is effectively real-time.  
**Poll interval:** 15,000ms — `AlertsPanel.tsx` line 32. Also polled by `ConflictMap.tsx` line 336 independently.

---

#### Source 4: Military ADS-B Flights
**File:** `src/app/api/flights/route.ts`  
**Function:** `GET()` — fetches `adsb.lol/v2/mil` (global military) + `adsb.lol/v2/lat/30/lon/48/dist/2500` (regional), merges, deduplicates by hex

**Output shape:**
```typescript
interface MilFlight {
  icao24: string;
  callsign: string;
  origin: string;          // derived from ICAO hex range via getOriginFromHex()
  lat: number;
  lon: number;
  altitude: number;        // feet
  heading: number;         // degrees
  speed: number;           // knots
  type: string;            // classifyAircraft() output: 'Fighter (F-35)', 'AWACS', etc.
  aircraftType: string;    // raw ICAO type code e.g. 'F35', 'C17'
  registration: string;
  description: string;
  squawk: string;
  isMilitary: boolean;     // dbFlags & 1
  isInteresting: boolean;  // dbFlags & 2
}
```
**Existing classification:** `classifyAircraft()` produces 30+ type strings; `isMilitaryCallsign()` checks against 60+ prefix list; `isMilitaryType()` checks 50+ ICAO type codes; `getOriginFromHex()` maps ICAO hex range to country.  
**Cache-Control:** `public, s-maxage=15`.  
**Poll interval:** 180,000ms — `FlightsPanel.tsx` line 65. Also consumed by `ConflictMap.tsx` line 334 independently (same interval).

---

#### Source 5: Naval Tracker
**File:** `src/app/api/ships/route.ts`  
**Function:** `GET()` → `getOSINTNavalPositions()` — returns hardcoded deployment data

**Critical note:** This is **static OSINT data** — positions are approximate, from public DoD/Navy press releases. There is no live AIS feed. The Finnish AIS endpoint referenced in comments only covers Baltic waters and is explicitly discarded (`void url`).

**Output shape:**
```typescript
interface NavalVessel {
  name: string;
  hull: string;
  type: string;   // 'Aircraft Carrier', 'Destroyer', 'Submarine', etc.
  class: string;
  navy: string;   // 'US Navy', 'Royal Navy', 'French Navy', 'Israeli Navy', 'Iran Navy', 'IRGC Navy', 'Saudi Navy'
  lat: number;
  lon: number;
  status: string;
  region: string; // 'Persian Gulf', 'Red Sea', 'Eastern Med', 'Strait of Hormuz'
  lastReported: string;
  group?: string;
}
```
**Cache-Control:** `public, s-maxage=300`.  
**Poll interval:** 300,000ms — `NavalPanel.tsx` line 51.

---

#### Source 6: NASA FIRMS Thermal Anomalies
**File:** `src/app/api/fires/route.ts`  
**Function:** `GET()` — downloads global SUOMI VIIRS 24h CSV from NASA FIRMS, filters to bbox (lat 20–42, lon 25–65)

**Output shape:**
```typescript
interface FireEvent {
  lat: number;
  lon: number;
  brightness: number;     // bright_ti4 Kelvin
  frp: number;            // Fire Radiative Power in MW — higher = more intense
  confidence: string;     // 'n', 'l', 'h' (nominal, low, high)
  intensity: 'low' | 'medium' | 'high' | 'extreme';
  datetime: string;       // ISO 8601 constructed from acq_date + acq_time
  daynight: string;       // 'D' or 'N'
  possibleExplosion: boolean; // frp > 80 && brightness > 380
}

interface FIRMSResponse {
  total: number;
  highIntensity: number;
  possibleExplosions: number;
  events: FireEvent[];    // top 100 by FRP, sorted desc
  source: string;
  updated: string;
}
```
**Cache-Control:** `public, s-maxage=600`.  
**Poll interval:** 600,000ms — `SatellitePanel.tsx` line 52.

---

#### Source 7: Conflict Monitor
**File:** `src/app/api/conflicts/route.ts`  
**Function:** `GET()` — 2 Google News RSS queries, keyword-based type + location classification

**Critical finding:** `lat` and `lon` in ConflictEvent are **always 0** as returned from this API. The coordinates field exists in `src/types/index.ts` (ConflictEvent interface, lines 36–42) but the API always sets them to 0. Geocoding happens client-side in `ConflictMap.tsx` via `geocodeStrike()` and the `STRIKE_LOCATIONS` lookup table. The intelligence layer must perform its own geo-resolution.

**Output shape:** `ConflictEvent[]` from `src/types/index.ts`:
```typescript
interface ConflictEvent {
  id: string;
  date: string;
  type: string;       // 'STRIKE' | 'DEFENSE' | 'MILITARY' | 'DIPLOMATIC' | 'NUCLEAR' | 'DRONE' | 'REPORT'
  location: string;   // text string only, e.g. "Tel Aviv, Israel"
  lat: number;        // always 0 from API
  lon: number;        // always 0 from API
  description: string;
  source: string;
  fatalities?: number;
}
```
**Poll interval:** 180,000ms — `ConflictFeed.tsx` line 16.

---

#### Source 8: Strike Tracker
**File:** `src/app/api/strikes/route.ts`  
**Function:** `GET()` — 2 Google News RSS queries, keyword-based category/severity/country classification

**Output shape:**
```typescript
interface StrikeEvent {
  id: string;
  date: string;
  category: string;   // 'MISSILE' | 'DRONE' | 'AIRSTRIKE' | 'ROCKET' | 'STRIKE' | 'INTERCEPTION' | 'REPORT'
  severity: 'low' | 'medium' | 'high' | 'critical';
  title: string;
  source: string;
  url: string;
  country: string;    // 'Iran' | 'Israel' | 'Lebanon' | 'Syria' | 'Yemen' | 'Middle East'
}
```
**Poll interval:** 120,000ms — `StrikesPanel.tsx` line 34.

---

#### Source 9: Defense & Markets
**File:** `src/app/api/markets/route.ts`  
**Function:** `GET()` via `fetchYahoo()` — unofficial Yahoo Finance v8 chart API  
**Symbols:** LMT, RTX, NOC, BA, GD, LHX, ^GSPC, ^DJI, ^VIX, GC=F, DX-Y.NYB  
**Cache-Control:** `public, s-maxage=300`.  
**Poll interval:** 600,000ms — `MarketsPanel.tsx` line 15.

---

#### Source 10: Oil / Energy Markets
**File:** `src/app/api/oil/route.ts`  
**Function:** `GET()` via unofficial Yahoo Finance  
**Symbols:** CL=F (WTI), BZ=F (Brent), NG=F (Natural Gas), HO=F (Heating Oil), RB=F (RBOB Gasoline)  
**Poll interval:** 600,000ms — `OilPanel.tsx` line 15, `MetricsBar.tsx` line 14.

---

#### Source 11: Crypto Markets
**File:** `src/app/api/crypto/route.ts`  
**Function:** `GET()` — CoinGecko free API (no key required)  
**Coins:** Bitcoin, Ethereum, Solana, BNB  
**Poll interval:** 600,000ms — `CryptoPanel.tsx` line 21.

---

#### Source 12: Polymarket Prediction Markets
**File:** `src/app/api/polymarket/route.ts`  
**Function:** `GET()` — Polymarket Gamma API, filtered by Middle East conflict keywords  
**Output:** Up to 20 active markets sorted by Yes probability  
**Poll interval:** 600,000ms — `PolymarketPanel.tsx` line 34.

---

#### Source 13: Regional Alerts
**File:** `src/app/api/regional-alerts/route.ts`  
**Function:** `GET()` — Google News RSS per country (10 countries), severity scoring  
**Countries:** Lebanon, Iran, Iraq, Syria, Yemen, Kuwait, Bahrain, UAE, Saudi Arabia, Jordan  
**Severity scoring:** `scoreSeverity()` function — CRITICAL_TERMS, HIGH_TERMS, MEDIUM_TERMS keyword arrays  
**Alert level:** `getAlertLevel()` — CLEAR | MONITORING | ALERT | CRITICAL  
**Poll interval:** 60,000ms — `RegionalAlertsPanel.tsx` line 57.

---

### 1.2 Data Flow — Full Path

```
External Sources
       │
       ▼
Next.js API Routes (/src/app/api/*)
  ├── Fetch + parse (fetchWithTimeout, parseXML)
  ├── Filter/classify (isRelevant, categorizeAlert, classifyAircraft, etc.)
  ├── Translate Hebrew (translateHebrew, translateFreeText via Google Translate)
  └── Return JSON with Cache-Control headers
       │
       ▼
useDataFeed hook (src/lib/hooks.ts)
  ├── useState<T> for data storage
  ├── setInterval() every [interval]ms
  ├── Cache-bust via _t=Date.now() query param
  └── Preserves previous data if new response empty
       │
       ▼
React Components (src/components/panels/*)
  └── Render data directly from hook state
       │
  ConflictMap.tsx also:
  ├── Independently polls 6 of the same APIs (flights, naval, alerts, conflicts, strikes, telegram)
  ├── Performs client-side geocoding via geocodeStrike() + STRIKE_LOCATIONS lookup
  ├── Stores flight trails in flightTrailsRef.current (Map<icao24, [lat,lon][]>)
  └── Dispatches/listens to 'map-focus' CustomEvent for cross-component coordination
```

**Important:** There is no global state store. Each component that calls `useDataFeed('/api/flights', 180000)` makes its own independent fetch cycle. `ConflictMap.tsx` and `FlightsPanel.tsx` both fetch `/api/flights` every 180 seconds — these are separate timers.

---

### 1.3 Existing State Storage Locations

| Data | Location | Type |
|------|----------|------|
| News items | `NewsFeed.tsx` component state | `NewsItem[]` via `useDataFeed` |
| Telegram posts | `TelegramPanel.tsx` component state | `TelegramResponse` via `useDataFeed` |
| Alert status | `AlertsPanel.tsx` component state | `AlertData` via `useDataFeed` |
| Flights | `FlightsPanel.tsx` + `ConflictMap.tsx` state | `FlightDataResponse` via `useDataFeed` (duplicate) |
| Ships | `NavalPanel.tsx` + `ConflictMap.tsx` state | `NavalData` via `useDataFeed` (duplicate) |
| Conflicts | `ConflictFeed.tsx` + `ConflictMap.tsx` state | `ConflictEvent[]` via `useDataFeed` (duplicate) |
| Strikes | `StrikesPanel.tsx` + `ConflictMap.tsx` state | `StrikeEvent[]` via `useDataFeed` (duplicate) |
| Telegram (map) | `ConflictMap.tsx` state | `TelegramData` via `useDataFeed` (duplicate) |
| Flight trails | `ConflictMap.tsx` `flightTrailsRef.current` | `Map<string, [number,number][]>` in-memory ref |
| Latest Telegram post IDs | `telegram/route.ts` module-level variable | `Record<string, number>` server-side |
| Telegram post cache | `telegram/route.ts` module-level variable | `Record<string, {text, date}>` server-side |
| Sticky alerts | `alerts/route.ts` module-level variable | `AlertEvent[]` server-side |

---

### 1.4 Existing Filtering and Categorization Logic

| Function | File | Purpose |
|----------|------|---------|
| `isRelevant()` | `api/news/route.ts:70` | Keyword regex filter for news relevance |
| `RELEVANCE_KEYWORDS` | `api/news/route.ts:68` | 70+ term regex for ME conflict |
| `categorizeAlert()` | `api/alerts/route.ts:127` | Maps Hebrew threat → MISSILE/ROCKET/DRONE/etc |
| `classifyAircraft()` | `api/flights/route.ts:190` | Maps type/callsign/desc → 30+ classification strings |
| `isMilitaryCallsign()` | `api/flights/route.ts:175` | 60+ callsign prefix check |
| `isMilitaryType()` | `api/flights/route.ts:183` | 50+ ICAO type code check |
| `getOriginFromHex()` | `api/flights/route.ts:247` | ICAO hex → country (US/UK/France/Germany/Italy/Spain/Israel/Iran/Turkey/Saudi/UAE) |
| `scoreSeverity()` | `api/regional-alerts/route.ts:87` | Keyword → critical/high/medium/low |
| `getAlertLevel()` | `api/regional-alerts/route.ts:95` | Event array → CLEAR/MONITORING/ALERT/CRITICAL |
| `geocodeStrike()` | `components/map/ConflictMap.tsx:260` | Description text → coordinate (client-side only) |
| `getRegion()` | `components/panels/SatellitePanel.tsx:34` | lat/lon bbox → country name string |
| `translateHebrew()` | `lib/hebrew.ts:57` | Hebrew → English dictionary lookup |
| `translateFreeText()` | `lib/hebrew.ts:101` | Any script → English via unofficial Google Translate |

---

### 1.5 Polling Intervals Summary

| Feed | Interval | Fastest Consumer |
|------|----------|-----------------|
| Pikud HaOref alerts | 15s | `AlertsPanel.tsx` |
| Telegram | 60s | `TelegramPanel.tsx` |
| Regional alerts | 60s | `RegionalAlertsPanel.tsx` |
| News | 90s | `NewsFeed.tsx` |
| Strikes | 120s | `StrikesPanel.tsx` |
| Conflicts | 180s | `ConflictFeed.tsx` |
| Flights | 180s | `FlightsPanel.tsx` |
| Naval | 300s | `NavalPanel.tsx` |
| Markets | 600s | `MarketsPanel.tsx` |
| Oil | 600s | `OilPanel.tsx` + `MetricsBar.tsx` |
| Crypto | 600s | `CryptoPanel.tsx` |
| Polymarket | 600s | `PolymarketPanel.tsx` |
| NASA FIRMS | 600s | `SatellitePanel.tsx` |

---

## Section 2 — Data Tap Points

The AI layer intercepts data at the API route layer, not the component layer. This keeps IRONSIGHT's existing React components entirely untouched. Each tap point is a new Next.js API route that calls the same upstream sources as the existing routes — or, for structured internal data, reads from Supabase snapshots.

### Tap Point 1: News + Conflicts + Strikes (Text Events)
**Files:** `src/app/api/news/route.ts`, `src/app/api/conflicts/route.ts`, `src/app/api/strikes/route.ts`  
**Available data at tap:** `NewsItem[]`, `ConflictEvent[]`, `StrikeEvent[]`  
**Transformation needed:**
- Merge all three into a unified `RawTextEvent` struct
- Extract source domain → trust tier via `sourceTrust.ts` lookup
- Resolve `ConflictEvent.location` string to coordinates via `geoResolver` agent (ConflictEvent.lat/lon are always 0)
- Strip Google News aggregator suffix (" - Source") already done in existing routes
- Pass to compression worker for clustering

**Requires modifying existing code:** No. The compression worker calls these same API endpoints directly, or reads the same upstream URLs itself.

---

### Tap Point 2: Telegram (Unstructured OSINT)
**File:** `src/app/api/telegram/route.ts`  
**Available data at tap:** `TelegramPost[]` with pre-translated text  
**Transformation needed:**
- The existing route already translates non-Latin text
- Assign channel-level trust tier (see Section 3 source tiers — IRGC-affiliated channels score 0.20, neutral OSINT 0.40)
- Pass full `text` field to `eventExtractor` agent for event classification
- Extract any geographic references from text for geo-resolution

**Channel affiliation map for trust scoring:**
```typescript
const CHANNEL_TIERS: Record<string, { tier: 4; score: number }> = {
  'IDFofficial': { tier: 4, score: 0.35 },       // Official but unverified
  'PressTV': { tier: 3, score: 0.30 },            // Iranian state media — use Tier 3 score
  'SaberinFa': { tier: 4, score: 0.20 },          // IRGC-affiliated
  'defapress_ir': { tier: 4, score: 0.20 },        // Iranian MOD
  'sepah': { tier: 4, score: 0.20 },               // IRGC official
  'OSINTdefender': { tier: 4, score: 0.40 },       // Neutral OSINT
  'warfareanalysis': { tier: 4, score: 0.40 },     // Neutral OSINT
  'rnintel': { tier: 4, score: 0.40 },             // Neutral OSINT
  // ... all 31 channels assigned
};
```
**Requires modifying existing code:** No.

---

### Tap Point 3: ADS-B Flights
**File:** `src/app/api/flights/route.ts`  
**Available data at tap:** `MilFlight[]` with type classification already done  
**Transformation needed:**
- Pass directly as `aircraft_observations` — already has icao24, callsign, lat, lon, altitude, speed, heading, type, aircraftType, origin
- **No mission inference** — type classification from `classifyAircraft()` is observation-only (aircraft type, not purpose)
- Enrich with area name via reverse geocoding (closest named city from ConflictMap's `CITIES` array)

**Requires modifying existing code:** No.

---

### Tap Point 4: NASA FIRMS Thermal
**File:** `src/app/api/fires/route.ts`  
**Available data at tap:** `FireEvent[]` with lat, lon, frp, brightness, datetime, possibleExplosion  
**Transformation needed:**
- Pass directly as `thermal_anomalies` — coordinates and timestamp only
- Enrich with reverse-geocoded country/region name via `getRegion()` function (already exists in `SatellitePanel.tsx:34` — copy logic to server-side)
- **No cause inference**

**Requires modifying existing code:** No.

---

### Tap Point 5: Pikud HaOref Alerts
**File:** `src/app/api/alerts/route.ts`  
**Available data at tap:** `AlertEvent[]` with type, locations, time  
**Transformation needed:**
- Count alert events in last 30 minutes
- Extract distinct locations
- Map type strings to `active_alerts` schema

**Requires modifying existing code:** No.

---

### Tap Point 6: Market Signals (Oil + Markets)
**Files:** `src/app/api/oil/route.ts`, `src/app/api/markets/route.ts`  
**Available data at tap:** `OilPrice[]`, `MarketItem[]`  
**Transformation needed:**
- Compute 30-minute change for WTI (CL=F), Brent (BZ=F), VIX (^VIX)
- Flag if absolute change > 1.5% in last 30 minutes
- Requires storing previous snapshot value — compare current vs stored `prev_oil_price` in Supabase `intelligence_config`

**Requires modifying existing code:** No.

---

### Tap Point 7: Naval Tracker
**File:** `src/app/api/ships/route.ts`  
**Available data at tap:** `NavalVessel[]` — static OSINT positions  
**Transformation needed:**
- Pass region and vessel type for context only
- Do NOT use coordinates for geospatial correlation — data is approximate OSINT, not live AIS
- Label as "last reported" not "current position"

**Requires modifying existing code:** No.

---

## Section 3 — The Compression Worker

### 3.1 Architecture

**Location:** `src/app/api/intelligence/compress/route.ts`  
**Trigger:** HTTP POST from Vercel Cron every 5 minutes  
**Vercel Cron config** (in `vercel.json`):
```json
{
  "crons": [{
    "path": "/api/intelligence/compress",
    "schedule": "*/5 * * * *"
  }]
}
```
**Input:** Raw events from all tap points collected in the last 30 minutes, fetched in parallel at compression time  
**Output:** `IntelligenceSnapshot` JSON stored in Supabase `intelligence_snapshots` table

---

### 3.2 IntelligenceSnapshot Schema

```typescript
interface IntelligenceSnapshot {
  id: string;                          // UUID
  created_at: string;                  // ISO 8601
  window_start: string;                // 30 minutes ago
  window_end: string;                  // now

  top_clusters: EventCluster[];
  signal_correlations: SignalCorrelation[];
  aircraft_observations: AircraftObservation[];
  thermal_anomalies: ThermalAnomaly[];
  active_alerts: AlertSummary;
  market_signals: MarketSignal[];

  snapshot_token_count: number;        // must never exceed 3000
  cluster_count: number;
  lead_cluster_id: string | null;      // id of highest priority_score cluster
}

interface EventCluster {
  id: string;                          // UUID
  area_name: string;                   // e.g. "Tel Aviv, Israel"
  area_name_ar: string;                // Arabic name e.g. "تل أبيب، إسرائيل"
  centroid_lat: number;
  centroid_lon: number;
  event_count: number;
  source_count: number;

  tier1_sources: SourceRef[];          // Reuters, AP, AFP, CENTCOM, DoD
  tier2_sources: SourceRef[];          // Al Jazeera, BBC, France24, CNN
  tier3_sources: SourceRef[];          // ToI, JPost, PressTV, regional state
  tier4_sources: SourceRef[];          // Telegram channels

  priority_score: number;              // 0–10, computed (see Section 4)
  verification_status: 'confirmed' | 'probable' | 'unverified' | 'contested';

  top_headline_ar: string;             // Arabic headline, translated if needed
  top_headline_en: string;             // English for processing
  event_type: string;                  // 'STRIKE' | 'MISSILE' | 'DRONE' | 'AIRSTRIKE' | 'DEFENSE' | 'MILITARY' | 'NUCLEAR' | 'DIPLOMATIC'
  target_category: string;             // from Section 4 asset table

  casualties_mentioned: boolean;
  casualties_confirmed: boolean;
  first_seen: string;                  // ISO 8601 of earliest source
  corroboration_count: number;         // distinct sources reporting same core claim
}

interface SourceRef {
  source_name: string;
  headline_en: string;
  url: string;
  published_at: string;
  trust_score: number;
}

interface SignalCorrelation {
  id: string;
  area_name: string;
  centroid_lat: number;
  centroid_lon: number;
  signal_types: ('rss' | 'telegram' | 'firms' | 'adsb' | 'alert')[];
  cluster_id: string;                  // references EventCluster
  description_en: string;             // factual only: "RSS + FIRMS + ADS-B activity detected"
  window_minutes: number;             // time span these signals span
  significance: 'HIGH' | 'MEDIUM';   // HIGH if 3+ signal types; MEDIUM if 2
}

interface AircraftObservation {
  icao24: string;
  callsign: string;
  aircraft_type_raw: string;          // e.g. "F35", "RC135"
  aircraft_type_classified: string;   // classifyAircraft() output
  origin_country: string;
  area_name: string;                   // nearest city from CITIES array
  lat: number;
  lon: number;
  altitude_ft: number;
  speed_kts: number;
  heading_deg: number;
  first_seen: string;
  last_seen: string;
  // NO mission_inference field. NO purpose field.
  // tier1_context: only if Reuters/AP/AFP/CENTCOM/DoD explicitly described this specific flight
  tier1_context: string | null;
}

interface ThermalAnomaly {
  lat: number;
  lon: number;
  area_name: string;
  frp_mw: number;
  brightness_k: number;
  confidence: string;
  intensity: 'low' | 'medium' | 'high' | 'extreme';
  acquisition_time: string;           // ISO 8601
  day_night: 'D' | 'N';
  flagged_anomalous: boolean;         // frp > 80 && brightness > 380
  // NO cause_inference field. NO "possible explosion" in output text — only in flagged_anomalous boolean.
  // tier1_context: only if Tier 1 source confirmed cause for these coordinates
  tier1_context: string | null;
}

interface AlertSummary {
  count_30min: number;
  locations: string[];
  threat_types: string[];
  most_recent: string;                // ISO 8601
}

interface MarketSignal {
  instrument: string;                 // 'WTI', 'BRENT', 'VIX', 'GOLD'
  change_percent: number;
  direction: 'up' | 'down';
  flagged: boolean;                   // |change_percent| > 1.5
  observation_window: string;         // "30 minutes"
}
```

---

### 3.3 Clustering Algorithm

The clustering algorithm runs inside `src/intelligence/workers/compressionWorker.ts`, function `clusterEvents()`.

**Step 1 — Collect raw events**

Fetch all tap points in parallel with a 10-second timeout each:
```typescript
const [news, telegram, conflicts, strikes, flights, fires, alerts, oil, markets] =
  await Promise.allSettled([
    fetch('/api/news'), fetch('/api/telegram'), fetch('/api/conflicts'),
    fetch('/api/strikes'), fetch('/api/flights'), fetch('/api/fires'),
    fetch('/api/alerts'), fetch('/api/oil'), fetch('/api/markets'),
  ]);
```

Collect only events with `published_at >= now - 30 minutes`. This is the compression window.

**Step 2 — Geo-resolve text events**

For each `ConflictEvent` and `NewsItem` and `TelegramPost` that does not have coordinates:
1. Run the `geoResolver` agent (see Section 10) against the text
2. If no resolution: skip event for clustering purposes but include in narrative context
3. Resolved coordinate goes to `centroid_lat/lon` for that event

**Step 3 — Haversine grid clustering**

Group events into clusters where:
- Geographic distance between any two events ≤ 0.1° in both lat and lon (≈ 11km grid cell)
- Time difference ≤ 30 minutes

Implementation in `src/intelligence/lib/haversine.ts`:
```typescript
function gridKey(lat: number, lon: number): string {
  return `${Math.floor(lat / 0.1)},${Math.floor(lon / 0.1)}`;
}
```
All events mapping to the same grid key are placed in the same candidate cluster. Adjacent grid cells are merged if they share more than 2 events (prevents artificial splits at cell boundaries).

**Step 4 — Deduplication within clusters**

Within each cluster:
- Group events where the first 50 chars of `title.toLowerCase()` are identical → one entry, `corroboration_count++`
- Group events with 80%+ word overlap in title → one entry, keep highest trust source as `top_headline`
- Keep count of how many distinct sources report the core claim

**Step 5 — Area name resolution**

For each cluster centroid, resolve to a human-readable area name:
1. Check against the 23 cities in `ConflictMap.tsx` CITIES array (lines 94–118) — use nearest if within 50km
2. Fall back to `getRegion()` logic from `SatellitePanel.tsx:34` for country-level naming
3. Translate to Arabic via the `geoResolver` agent

**Step 6 — Sort by priority_score DESC**

Apply the scoring engine (Section 4). Sort clusters. Assign `lead_cluster_id` to the top-scoring cluster.

**Step 7 — Signal correlation detection**

For each cluster, check if multiple signal type categories have events within the same 0.1° grid cell and 30-minute window:
- RSS sources → signal type `rss`
- Telegram channels → signal type `telegram`
- NASA FIRMS events → signal type `firms`
- ADS-B flights in area → signal type `adsb`
- Pikud HaOref alerts at that location → signal type `alert`

If ≥2 signal types present: create `SignalCorrelation` entry. If ≥3 signal types: mark `significance: 'HIGH'`.

**Step 8 — Token budget enforcement**

Count estimated tokens before writing to Supabase. Target: ≤3000 tokens.

Token estimation in `src/intelligence/lib/tokenCounter.ts`:
```typescript
function estimateTokens(snapshot: IntelligenceSnapshot): number {
  const json = JSON.stringify(snapshot);
  return Math.ceil(json.length / 4); // GPT-4 approximation: ~4 chars/token
}
```

If over budget: truncate clusters from the bottom of the priority-sorted list until under 3000 tokens. Aircraft observations capped at 15 entries. Thermal anomalies capped at 20 entries.

---

### 3.4 Source Trust Tiers

Implemented in `src/intelligence/lib/sourceTrust.ts` as a deterministic lookup table. The LLM never determines trust — this is always a lookup.

```typescript
interface SourceTrustEntry {
  tier: 1 | 2 | 3 | 4;
  score: number;
  name: string;
}

const SOURCE_TRUST_MAP: Record<string, SourceTrustEntry> = {
  // Tier 1 — score 0.95
  'Reuters': { tier: 1, score: 0.95, name: 'Reuters' },
  'AP': { tier: 1, score: 0.95, name: 'Associated Press' },
  'AFP': { tier: 1, score: 0.95, name: 'Agence France-Presse' },
  'CENTCOM': { tier: 1, score: 0.95, name: 'US CENTCOM' },
  'DoD': { tier: 1, score: 0.95, name: 'US Department of Defense' },
  'Pentagon': { tier: 1, score: 0.95, name: 'Pentagon' },

  // Tier 2 — score 0.80
  'Al Jazeera': { tier: 2, score: 0.80, name: 'Al Jazeera' },
  'BBC': { tier: 2, score: 0.80, name: 'BBC' },
  'France24': { tier: 2, score: 0.80, name: 'France 24' },
  'CNN': { tier: 2, score: 0.75, name: 'CNN' },
  'NYT': { tier: 2, score: 0.80, name: 'New York Times' },
  'WSJ': { tier: 2, score: 0.78, name: 'Wall Street Journal' },

  // Tier 3 — score 0.55
  'Times of Israel': { tier: 3, score: 0.55, name: 'Times of Israel' },
  'JPost': { tier: 3, score: 0.55, name: 'Jerusalem Post' },
  'Haaretz': { tier: 3, score: 0.58, name: 'Haaretz' },
  'Ynet': { tier: 3, score: 0.50, name: 'Ynet' },
  'PressTV': { tier: 3, score: 0.40, name: 'PressTV' },
  'The National': { tier: 3, score: 0.55, name: 'The National' },
  'Breaking Def': { tier: 3, score: 0.60, name: 'Breaking Defense' },
  'Long War Jrnl': { tier: 3, score: 0.58, name: 'Long War Journal' },
  'Mil Times': { tier: 3, score: 0.55, name: 'Military Times' },
  'TasnimNewsEN': { tier: 3, score: 0.35, name: 'Tasnim News' },
  'FarsNews_EN': { tier: 3, score: 0.35, name: 'Fars News' },

  // Tier 4 — Telegram channels (variable)
  'IDFofficial': { tier: 4, score: 0.35, name: 'IDF Official Telegram' },
  'OSINTdefender': { tier: 4, score: 0.40, name: 'OSINT Defender' },
  'warfareanalysis': { tier: 4, score: 0.40, name: 'Warfare Analysis' },
  'rnintel': { tier: 4, score: 0.40, name: 'RN Intel' },
  'GeoPWatch': { tier: 4, score: 0.38, name: 'GeoPol Watch' },
  'AbuAliExpress': { tier: 4, score: 0.35, name: 'Abu Ali Express' },
  'SaberinFa': { tier: 4, score: 0.20, name: 'Saberin (IRGC)' },
  'defapress_ir': { tier: 4, score: 0.20, name: 'DefaPress (Iran MOD)' },
  'sepah': { tier: 4, score: 0.20, name: 'IRGC Official' },
  'FotrosResistancee': { tier: 4, score: 0.25, name: 'Fotros Resistance' },
  'QudsNen': { tier: 4, score: 0.30, name: 'Quds News' },
};

export function getTrustEntry(sourceName: string): SourceTrustEntry {
  const direct = SOURCE_TRUST_MAP[sourceName];
  if (direct) return direct;
  // Fallback: Google News aggregated items get Tier 3 default
  if (sourceName === 'Google News') return { tier: 3, score: 0.45, name: 'Google News' };
  return { tier: 4, score: 0.25, name: sourceName };
}
```

---

### 3.5 Verification Logic

Implemented in `src/intelligence/agents/verificationAssessor.ts`:

```typescript
function assessVerification(cluster: EventCluster): VerificationStatus {
  const t1t2Sources = [...cluster.tier1_sources, ...cluster.tier2_sources];
  const t3t4Sources = [...cluster.tier3_sources, ...cluster.tier4_sources];

  // Check for contradiction
  if (sourcesContradict(cluster)) return 'contested';

  // Confirmed: ≥2 independent Tier 1/2 sources on core claim
  if (t1t2Sources.length >= 2) return 'confirmed';

  // Probable: 1 T1/T2 + ≥2 T3/T4
  if (t1t2Sources.length >= 1 && t3t4Sources.length >= 2) return 'probable';

  // Probable: ≥3 independent T3/T4
  if (t3t4Sources.length >= 3) return 'probable';

  return 'unverified';
}

function sourcesContradict(cluster: EventCluster): boolean {
  // Check if any source headlines directly negate another
  // Simple heuristic: look for negation terms in same-cluster headlines
  const headlines = [
    ...cluster.tier1_sources,
    ...cluster.tier2_sources,
    ...cluster.tier3_sources,
  ].map(s => s.headline_en.toLowerCase());

  const affirmTerms = ['confirmed', 'struck', 'hit', 'killed', 'destroyed'];
  const denyTerms = ['denied', 'false', 'fake', 'no strike', 'not confirmed', 'disputed'];

  const hasAffirm = headlines.some(h => affirmTerms.some(t => h.includes(t)));
  const hasDeny = headlines.some(h => denyTerms.some(t => h.includes(t)));
  return hasAffirm && hasDeny;
}
```

---

## Section 4 — Priority Scoring Engine

Implemented in `src/intelligence/agents/priorityScorer.ts`.

The scoring engine is **entirely deterministic lookup + arithmetic**. The LLM is never called for scoring. This prevents hallucinated priority inflation.

### 4.1 Signal 1 — Strategic Asset Value (weight 30%)

Determined by keyword matching against the cluster's `event_type` + `top_headline_en` + `target_category`. The `target_category` field is set by the `eventExtractor` agent using this table:

```typescript
const ASSET_VALUES: Record<string, number> = {
  // 1.00 — existential/strategic
  'advanced_fighter': 1.00,      // F-35, F-22, B-2, F-15
  'nuclear_facility': 1.00,      // Dimona, Natanz, Bushehr, Isfahan
  'wmd_site': 1.00,

  // 0.95 — command-level personnel
  'senior_commander': 0.95,      // General rank or above

  // 0.85 — major air defense
  'air_defense_system': 0.85,    // S-300, Patriot, Iron Dome, Bavar-373, Arrow

  // 0.80 — civilian mass casualty potential
  'civilian_infrastructure': 0.80, // hospital, water, power grid
  'population_center': 0.78,

  // 0.75 — critical infrastructure
  'strategic_bridge': 0.75,
  'strategic_dam': 0.75,
  'port': 0.72,

  // 0.70 — command and control
  'command_control': 0.70,

  // 0.65 — military bases
  'military_airbase': 0.65,
  'missile_base': 0.68,

  // 0.60 — naval
  'naval_vessel': 0.60,
  'naval_base': 0.62,

  // 0.55 — surveillance / ISR
  'isr_asset': 0.55,

  // 0.40 — ground forces
  'armored_convoy': 0.40,
  'military_convoy': 0.38,

  // 0.30 — artillery
  'artillery_position': 0.30,
  'rocket_battery': 0.32,

  // 0.20 — generic
  'generic_military': 0.20,
  'unknown': 0.15,
};
```

The `eventExtractor` agent (`src/intelligence/agents/eventExtractor.ts`) classifies the target from the headline text and maps it to one of these keys. It receives the headline and source tier — not the full snapshot.

### 4.2 Signal 2 — Escalation Delta (weight 25%)

Requires querying `intelligence_snapshots` table for the last 48 hours:

```typescript
async function computeEscalationDelta(
  event_type: string,
  target_category: string,
  centroid_lat: number,
  centroid_lon: number,
  supabase: SupabaseClient
): Promise<number> {
  // Grid key for 0.5° search radius (looser than clustering, to catch same-area events)
  const latMin = centroid_lat - 0.5;
  const latMax = centroid_lat + 0.5;
  const lonMin = centroid_lon - 0.5;
  const lonMax = centroid_lon + 0.5;

  const { data } = await supabase
    .from('intelligence_snapshots')
    .select('snapshot_data, created_at')
    .gte('created_at', new Date(Date.now() - 48 * 3600 * 1000).toISOString())
    .order('created_at', { ascending: false });

  // Count prior occurrences of same event_type + target_category in same area
  let priorCount = 0;
  let mostRecentHoursAgo = 999;

  for (const snap of (data || [])) {
    const clusters = snap.snapshot_data.top_clusters as EventCluster[];
    const match = clusters.find(c =>
      c.event_type === event_type &&
      c.target_category === target_category &&
      Math.abs(c.centroid_lat - centroid_lat) < 0.5 &&
      Math.abs(c.centroid_lon - centroid_lon) < 0.5
    );
    if (match) {
      priorCount++;
      const hoursAgo = (Date.now() - new Date(snap.created_at).getTime()) / 3600000;
      if (hoursAgo < mostRecentHoursAgo) mostRecentHoursAgo = hoursAgo;
    }
  }

  // Score lookup
  if (priorCount === 0) return 1.0;                          // Never seen in 48h
  if (priorCount === 1 && mostRecentHoursAgo > 24) return 0.65; // Once, >24h ago
  if (priorCount >= 2 && priorCount <= 4) return 0.35;       // 2-4 times in 24h
  return 0.10;                                               // 5+ times (routine)
}
```

**First-48h fallback:** Until 48h of snapshot history has accumulated, return 0.5 for all events (neutral — neither escalation nor routine).

### 4.3 Signal 3 — Velocity (weight 20%)

```typescript
function computeVelocity(
  corroboration_count: number,
  first_seen: string
): number {
  const t = (Date.now() - new Date(first_seen).getTime()) / 3600000; // hours
  const raw = Math.min(corroboration_count / 5, 1.0);
  return raw * Math.exp(-0.3 * t);
  // 4 sources in 30 minutes: min(4/5,1) * e^(-0.3*0.5) ≈ 0.80 * 0.86 ≈ 0.69
  // 5 sources in 30 minutes: 1.0 * e^(-0.15) ≈ 0.86
  // 4 sources in 6 hours: 0.80 * e^(-1.8) ≈ 0.80 * 0.165 ≈ 0.13
}
```

### 4.4 Signal 4 — Breadth of Impact (weight 15%)

```typescript
function computeBreadth(cluster: EventCluster): number {
  let score = 0;
  if (cluster.casualties_confirmed) score += 0.30;
  else if (cluster.casualties_mentioned) score += 0.15;
  if (cluster.event_count > 3) score += 0.20;    // multiple locations or reports
  const isCapitalOrMajorCity = MAJOR_CITIES.some(c =>
    Math.abs(c.lat - cluster.centroid_lat) < 0.3 &&
    Math.abs(c.lon - cluster.centroid_lon) < 0.3
  );
  if (isCapitalOrMajorCity) score += 0.20;
  return Math.min(score, 1.0);
}
```

`MAJOR_CITIES` is the CITIES array from `ConflictMap.tsx` (23 cities) exported to the intelligence module.

### 4.5 Signal 5 — Audience Relevance for Al Jazeera Arabic (weight 10%)

```typescript
const AUDIENCE_RELEVANCE: Record<string, number> = {
  'Gaza': 1.00, 'West Bank': 1.00,
  'Lebanon': 0.85, 'Yemen': 0.85, 'Syria': 0.85,
  'Iran': 0.75, 'Iraq': 0.75,
  'Saudi Arabia': 0.65, 'Jordan': 0.65, 'Egypt': 0.65,
  'Kuwait': 0.60, 'Bahrain': 0.60, 'UAE': 0.60, 'Qatar': 0.60,
  'Israel': 0.80,    // High relevance for AJ Arabic audience
  'Turkey': 0.55,
};

function computeAudienceRelevance(area_name: string): number {
  for (const [region, score] of Object.entries(AUDIENCE_RELEVANCE)) {
    if (area_name.includes(region)) return score;
  }
  return 0.30;
}
```

### 4.6 Final Score Formula

```typescript
function computePriorityScore(
  s1: number, s2: number, s3: number, s4: number, s5: number
): number {
  return (s1 * 0.30 + s2 * 0.25 + s3 * 0.20 + s4 * 0.15 + s5 * 0.10) * 10;
}
```

**Lead candidate threshold:** priority_score ≥ 7.5

### 4.7 Tunable Weights

Weights stored in Supabase `intelligence_config` table (single row). The compression worker reads weights from this table at start of each run — never from hardcoded values. The defaults above are the database seed values.

```sql
SELECT weight_asset_value, weight_escalation_delta, weight_velocity,
       weight_breadth, weight_audience
FROM intelligence_config
LIMIT 1;
```

Editor-facing weight sliders (`WeightSliders.tsx`) write to this table via a protected API route. Only authenticated sessions with `role = 'editor'` can write.

---

## Section 5 — The Analysis Workers

Two LLM calls per compression cycle. The cycle runs every 5 minutes; LLM calls run once per cycle regardless of how many users are online. This means 12 LLM calls per hour total for shared reports — a fixed cost, not per-user.

### 5.1 Worker A — Situation Report (GPT-4o)

**Trigger:** Called by `analysisWorker.ts` immediately after compression worker stores the snapshot  
**Input:** Current `IntelligenceSnapshot` serialized to JSON (≤3000 tokens) + system prompt  
**Output:** Stored as `base_report` in `intelligence_reports` table, `report_type = 'base'`

**Full system prompt** (stored in `src/intelligence/prompts/situationReport.ts`):

```
أنت محرر أول في قناة الجزيرة العربية، متخصص في تغطية شؤون الشرق الأوسط والأمن الإقليمي.
مهمتك: إعداد نشرة موجزة عملياتية بالعربية الفصحى المعاصرة، بناءً على البيانات المُدخلة.

قواعد العزو — يجب الالتزام بها تمامًا:
1. الحقائق المرصودة تُذكر مباشرةً دون قيد: بيانات ADS-B (المسار والموقع والارتفاع)، إحداثيات NASA FIRMS، عدد تنبيهات تسيفا أدوم ومناطقها.
2. أي ادعاء تفسيري يستلزم: "وفقاً لـ [اسم المصدر]".
3. مهمة الطائرات / الغرض منها: لا تُذكر إطلاقاً ما لم تُصرح بها وكالة رويترز أو AP أو AFP أو وزارة الدفاع الأمريكية أو CENTCOM صراحةً في الخبر المُرفق.
4. سبب الشذوذ الحراري: لا تُذكر إطلاقاً ما لم يُؤكده مصدر من الدرجة الأولى.
5. إذا لم يكن ثمة مصدر من الدرجة الأولى لادعاء ما، احذفه كليًا — لا تلطّفه، لا تقل "يُحتمل"، احذفه.

تصنيف الادعاءات:
- مؤكد (confirmed): مصدران مستقلان من الدرجة الأولى/الثانية
- مرجّح (probable): مصدر من الدرجة الأولى/الثانية + مصدران من الدرجة الثالثة/الرابعة، أو ثلاثة مصادر من الدرجة الثالثة/الرابعة
- غير مؤكد (unverified): مصدر واحد فقط

صيغة الإخراج (JSON):
{
  "lead_paragraph_ar": "فقرة افتتاحية — 3 جمل كحد أقصى — أهم حدث موثق",
  "secondary_bullets_ar": ["نقطة 1", "نقطة 2", ...],  // 3-5 نقاط
  "verification_notes_ar": { "event_id": "حالة التحقق والمصدر" },
  "word_count": 0
}

الحد الأقصى للكلمات: 250 كلمة.
لا تستخدم لغة عاطفية أو إنشائية. اكتب بأسلوب تقريري محايد.
```

**Model:** `claude-opus-4-8` (most capable for Arabic) — or `gpt-4o` if using OpenAI  
**Token budget:** ≤3000 tokens input + ≤400 tokens output

---

### 5.2 Worker B — Lead Story Recommendation (GPT-4o-mini / claude-haiku-4-5)

**Input:** Top 5 clusters from snapshot by priority_score  
**Output:** 3 story recommendations, stored as `report_type = 'lead_recommendations'` in `intelligence_reports`

**Full system prompt** (stored in `src/intelligence/prompts/leadStory.ts`):

```
أنت منتج أخبار في قناة الجزيرة العربية.
ستحصل على قائمة بأبرز مجموعات الأحداث مع نقاطها الإجمالية.
مهمتك: اختيار 3 قصص تصلح لأن تكون تقارير رئيسية، مع تفسير موجز لكل منها.

أخرج JSON بالتنسيق التالي — بالعربية الفصحى:
{
  "recommendations": [
    {
      "rank": 1,
      "headline_ar": "عنوان القصة بالعربية",
      "editorial_reason_ar": "جملة واحدة: لماذا تهم هذه القصة جمهور الجزيرة العربية الآن",
      "driving_signal": "asset_value | escalation | velocity | breadth | audience",
      "suggested_focus_query_ar": "استفسار صياغي مقترح للصحفي — جملة واحدة"
    }
  ]
}
```

**Model:** `claude-haiku-4-5-20251001` (fast, low cost for this simple ranking task)

---

### 5.3 Storage

Both outputs stored in `intelligence_reports` table with:
- `snapshot_id` (FK to `intelligence_snapshots.id`)
- `generated_at` (immutable — never updated, new row each cycle)
- `report_data` (jsonb — the full JSON output)
- `model_used` (string — for audit trail)
- `tokens_in`, `tokens_out` (for cost tracking)

Previous reports retained for 7 days via Supabase row-level TTL or a scheduled cleanup function.

---

## Section 6 — Personal Report System

### 6.1 Story Focus Selector UI

**Placement:** A new collapsible panel added as a 13th column-span-12 row below the existing 3-row main grid in `src/app/page.tsx`. It slides up from the bottom of the viewport and can be minimized. This is the only modification to `page.tsx` — a single import and one JSX element.

```tsx
// In page.tsx, after the closing </main> tag, before </footer>:
import IntelligencePanel from '@/components/intelligence/IntelligencePanel';
// ...
<IntelligencePanel />
```

The panel occupies a fixed-position overlay at the bottom of the screen, 280px tall when open, toggled by a keyboard shortcut (Alt+I) or a button in the footer bar. This keeps all existing panels completely unchanged.

**UI Elements in `StoryFocusSelector.tsx`:**
1. Three auto-suggested focus buttons populated from Worker B's `recommended[].suggested_focus_query_ar`
2. Free-text Arabic query input with placeholder "اكتب موضوع تقريرك..."
3. Active focus display with timestamp of last-generated report
4. "تحديث التقرير" button (manual trigger)
5. "تصفير" (clear/reset) button

---

### 6.2 Personal Report Generation

**API endpoint:** `src/app/api/intelligence/report/personal/route.ts`

**Trigger conditions:**
1. Journalist sets a new story focus (immediate generation)
2. Journalist manually clicks "تحديث التقرير"
3. New snapshot arrives AND contains new clusters matching journalist's focus keywords AND journalist has not generated a report in last 10 minutes

NOT triggered on every snapshot — journalist controls timing.

**Rate limit:** Hard limit of 1 personal report generation per `user_id` per 10 minutes. Implemented as a token bucket: check `journalist_sessions.last_report_at` before calling LLM. Reject with 429 if within 10-minute window.

**Input to LLM:**
```typescript
const input = {
  snapshot: filterSnapshotToFocus(currentSnapshot, storyFocusQuery),
  journalist_query: storyFocusQuery,
  system_prompt: PERSONAL_REPORT_PROMPT, // from src/intelligence/prompts/personalReport.ts
};
```

**Focus filtering** (`filterSnapshotToFocus()`): Keep only clusters where `area_name_ar` or `top_headline_ar` or `top_headline_en` contains at least one keyword from the journalist's query. Also keep all `signal_correlations` and `aircraft_observations` that overlap with matching cluster coordinates.

---

### 6.3 Personal Report Schema

```typescript
interface PersonalReport {
  generated_at: string;
  snapshot_id: string;
  story_focus: string;
  lead_paragraph_ar: string;

  confirmed_claims: Array<{
    text_ar: string;
    source: string;
    source_tier: 1 | 2 | 3 | 4;
    timestamp: string;
  }>;

  unconfirmed_reports: Array<{
    text_ar: string;
    source: string;
    source_tier: 3 | 4;
    caveat_ar: string;  // e.g. "لم يتم التأكيد من مصادر مستقلة"
  }>;

  observed_signals: Array<{
    signal_type: 'aircraft' | 'thermal' | 'alert' | 'naval';
    description_ar: string;   // observed facts ONLY
    coordinates?: { lat: number; lon: number };
    timestamp: string;
    tier1_context: string | null;
  }>;

  watch_next_ar: string;
  export_ready: boolean;  // true if verification_status is confirmed or probable with ≥1 Tier 1 source
}
```

**Full personal report system prompt** (stored in `src/intelligence/prompts/personalReport.ts`):

```
أنت محرر متخصص في الجزيرة العربية تساعد صحفيًا يعمل على قصة محددة.
ستحصل على:
1. بيانات استخباراتية مصفاة تتعلق بتركيز القصة
2. استفسار الصحفي بالعربية

مهمتك: إعداد تقرير شخصي منظم يتضمن:
- الادعاءات المؤكدة فقط مع عزوها لمصادرها
- التقارير غير المؤكدة مع تحذيرات صريحة
- الإشارات الرصدية (ADS-B، NASA FIRMS، التنبيهات) كوقائع فحسب

قواعد الصرامة في العزو:
[نفس القواعد الواردة في Worker A بالكامل]

أخرج JSON صارمًا وفق مخطط PersonalReport.
لا تضف حقولاً إضافية. لا تتجاوز حقل tier1_context إذا لم يكن ثمة مصدر من الدرجة الأولى.
```

---

### 6.4 Caching

Personal reports cached in `personal_report_cache` table:
- Cache key: `SHA-256(user_id + story_focus_normalized)`
- Cache TTL: 30 minutes (`expires_at = generated_at + 30min`)
- Invalidation: new snapshot with matching clusters AND >10 minutes since last report
- Read before generating: if valid cache entry exists, return it immediately without LLM call

### 6.5 Cost Model at 300 Concurrent Users

| Component | Frequency | Input tokens | Output tokens | Cost/call | Daily cost |
|-----------|-----------|-------------|---------------|-----------|-----------|
| Compression worker | 12×/hour | 0 (no LLM) | — | $0 | $0 |
| Worker A (Situation Report) | 12×/hour | ~3000 | ~400 | $0.045 | $12.96 |
| Worker B (Lead Story) | 12×/hour | ~800 | ~200 | $0.004 | $1.15 |
| Personal reports | 3×/user/hour avg × 300 users | ~1500 | ~600 | $0.012 | $259.20 |
| **Total** | | | | | **~$273/day** |

Worst case (all 300 users hitting rate limit every 10 minutes, 6 calls/hour): ~$518/day.  
Realistic (3 calls/user/hour): ~$273/day.  
Off-peak (100 active users): ~$95/day.

---

## Section 7 — Export System

All exports are generated server-side from the `PersonalReport` object. No client-side PDF generation.

### 7.1 Export A — Vizrt Broadcast JSON

**Endpoint:** `GET /api/intelligence/export/vizrt?report_id={id}`  
**File:** `src/app/api/intelligence/export/vizrt/route.ts`

```json
{
  "story_title_ar": "",
  "generated_at": "",
  "snapshot_id": "",
  "journalist_id": "",
  "lead_event": {
    "label_ar": "",
    "lat": 0.0,
    "lng": 0.0,
    "event_type_ar": "",
    "importance": "high|medium|low",
    "display_size": "large|medium|small",
    "verification": "confirmed|probable|unverified",
    "source_ar": "",
    "coordinate_confidence_km": 5
  },
  "secondary_events": [
    {
      "label_ar": "",
      "lat": 0.0,
      "lng": 0.0,
      "event_type_ar": "",
      "importance": "medium|low",
      "display_size": "medium|small",
      "verification": "confirmed|probable|unverified",
      "source_ar": ""
    }
  ],
  "broadcast_safe": true,
  "broadcast_safe_reason": "",
  "coordinate_precision_km": 5
}
```

`broadcast_safe = true` only when:
- `verification` is `confirmed` or `probable`
- At least one Tier 1 source in the cluster
- `geo_confidence` of coordinates ≥ 0.75 (set by `geoResolver` agent)

Only events with `geo_confidence ≥ 0.75` are included in this export.

---

### 7.2 Export B — GeoJSON

**Endpoint:** `GET /api/intelligence/export/geojson?report_id={id}`  
**File:** `src/app/api/intelligence/export/geojson/route.ts`

Standard `FeatureCollection`. Each confirmed or probable cluster becomes a `Point` feature. All `PersonalReport` fields included in `properties`. Compatible with ArcGIS, QGIS, Mapbox Studio drag-and-drop.

Only confirmed and probable events included. Unverified events excluded from GeoJSON to prevent geographic misinformation.

---

### 7.3 Export C — Arabic PDF Brief

**Endpoint:** `GET /api/intelligence/export/pdf?report_id={id}`  
**File:** `src/app/api/intelligence/export/pdf/route.ts`  
**Implementation:** `@react-pdf/renderer` for Next.js-compatible PDF generation (pure React, no browser APIs). Falls back to `jsPDF` with Arabic RTL support if `@react-pdf/renderer` has issues.

**PDF Structure (RTL, Arabic):**
- **Header:** Story focus title, generated_at, journalist name (from session)
- **Section 1 — الفقرة الرئيسية:** lead_paragraph_ar
- **Section 2 — الادعاءات المؤكدة:** confirmed_claims with source footnotes
- **Section 3 — تقارير غير مؤكدة:** unconfirmed_reports, clearly labelled "غير مؤكد"
- **Section 4 — الإشارات الرصدية:** observed_signals (aircraft, thermal, alerts) as observed facts
- **Section 5 — ما يجب مراقبته:** watch_next_ar
- **Footer:** "تم إنشاء هذا التقرير بمساعدة الذكاء الاصطناعي — يخضع للمراجعة التحريرية"

---

## Section 8 — Satellite Imagery Integration

**Endpoint:** `GET /api/intelligence/satellite?lat={}&lon={}&mode={breaking|investigative}`  
**File:** `src/app/api/intelligence/satellite/route.ts`

### 8.1 Mode A — Breaking News (Single Recent Image)

Query Sentinel Hub API (ESA, free tier available):
- 5km × 5km bounding box centered on event coordinates
- `SentinelHub.Process.SENTINEL2_L2A` — optical, 10m resolution, 5-day cadence
- Fallback: `SentinelHub.Process.SENTINEL1_GRD` — SAR, all-weather, 6-12 day cadence

**Critical:** Display acquisition date prominently — journalist must know how old the image is relative to the event. A 5-day-old Sentinel-2 image for a today's event is important context.

Response schema:
```typescript
interface SatelliteImageResponse {
  mode: 'breaking';
  coordinates: { lat: number; lon: number };
  bbox_km: 5;
  image_url: string;        // signed URL to image tile
  acquisition_date: string; // ISO 8601 — acquisition time, not download time
  sensor: 'SENTINEL2' | 'SENTINEL1_SAR';
  resolution_m: 10 | 20;   // varies by Sentinel product
  cloud_cover_percent?: number;
  // NO interpretation fields
  // NO cause_inference field
  caption_ar: string;       // "صورة الأقمار الاصطناعية المتاحة الأخيرة لهذه الإحداثيات، التقطت في [date]"
}
```

### 8.2 Mode B — Investigative (Before/After Timeline)

- Query images from 30 days before `cluster.first_seen` → present
- Return array of dated image tiles sorted by acquisition date
- Presented as side-by-side comparison panel in `SatelliteViewer.tsx`
- This is the documentary evidence workflow for investigative journalism

### 8.3 What the System Never Does With Imagery

The caption field is pre-populated with the factual statement in Arabic. The LLM is not called for satellite image interpretation. The journalist interprets. The system delivers the image and the date.

---

## Section 9 — Database Schema

All new tables in Supabase. None of these tables conflict with any existing IRONSIGHT schema — IRONSIGHT has no database at present.

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- =====================================================
-- intelligence_snapshots
-- =====================================================
CREATE TABLE intelligence_snapshots (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  window_start TIMESTAMPTZ NOT NULL,
  window_end TIMESTAMPTZ NOT NULL,
  snapshot_data JSONB NOT NULL,   -- full IntelligenceSnapshot
  token_count INTEGER NOT NULL,
  cluster_count INTEGER NOT NULL,
  lead_cluster_id TEXT,           -- UUID of top cluster, nullable
  compression_duration_ms INTEGER -- for performance monitoring
);

CREATE INDEX idx_snapshots_created_at ON intelligence_snapshots(created_at DESC);

-- TTL: delete snapshots older than 7 days
CREATE OR REPLACE FUNCTION cleanup_old_snapshots()
RETURNS void AS $$
  DELETE FROM intelligence_snapshots WHERE created_at < NOW() - INTERVAL '7 days';
$$ LANGUAGE SQL;

-- =====================================================
-- intelligence_reports
-- =====================================================
CREATE TABLE intelligence_reports (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  snapshot_id UUID NOT NULL REFERENCES intelligence_snapshots(id) ON DELETE CASCADE,
  report_type TEXT NOT NULL CHECK (report_type IN ('base', 'personal', 'lead_recommendations')),
  story_focus TEXT,               -- NULL for base reports
  story_focus_hash TEXT,          -- SHA-256 hash of normalized focus query
  report_data JSONB NOT NULL,     -- PersonalReport or base report JSON
  model_used TEXT NOT NULL,
  tokens_in INTEGER NOT NULL,
  tokens_out INTEGER NOT NULL,
  generation_duration_ms INTEGER
);

CREATE INDEX idx_reports_snapshot_id ON intelligence_reports(snapshot_id);
CREATE INDEX idx_reports_story_focus_hash ON intelligence_reports(story_focus_hash);
CREATE INDEX idx_reports_created_at ON intelligence_reports(created_at DESC);

-- =====================================================
-- intelligence_config
-- =====================================================
-- Single row — the editor weight configuration
CREATE TABLE intelligence_config (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  weight_asset_value DECIMAL(4,2) NOT NULL DEFAULT 0.30,
  weight_escalation_delta DECIMAL(4,2) NOT NULL DEFAULT 0.25,
  weight_velocity DECIMAL(4,2) NOT NULL DEFAULT 0.20,
  weight_breadth DECIMAL(4,2) NOT NULL DEFAULT 0.15,
  weight_audience DECIMAL(4,2) NOT NULL DEFAULT 0.10,
  -- Weights must sum to 1.0 — enforced by check constraint
  CONSTRAINT weights_sum_to_one CHECK (
    ABS((weight_asset_value + weight_escalation_delta + weight_velocity +
         weight_breadth + weight_audience) - 1.00) < 0.01
  ),
  prev_oil_price_wti DECIMAL(8,2),  -- stored for 30-minute change detection
  prev_oil_price_brent DECIMAL(8,2),
  prev_vix DECIMAL(8,2),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_by TEXT
);

-- Seed with defaults
INSERT INTO intelligence_config DEFAULT VALUES;

-- =====================================================
-- personal_report_cache
-- =====================================================
CREATE TABLE personal_report_cache (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id TEXT NOT NULL,
  story_focus_hash TEXT NOT NULL,  -- SHA-256(user_id + normalized_focus)
  story_focus TEXT NOT NULL,
  report_data JSONB NOT NULL,      -- PersonalReport object
  snapshot_id UUID NOT NULL REFERENCES intelligence_snapshots(id) ON DELETE CASCADE,
  generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL, -- generated_at + 30 minutes
  UNIQUE (user_id, story_focus_hash)
);

CREATE INDEX idx_cache_user_id ON personal_report_cache(user_id);
CREATE INDEX idx_cache_expires_at ON personal_report_cache(expires_at);

-- =====================================================
-- journalist_sessions
-- =====================================================
CREATE TABLE journalist_sessions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id TEXT NOT NULL UNIQUE,
  story_focus TEXT,
  focus_set_at TIMESTAMPTZ,
  last_report_at TIMESTAMPTZ,     -- for rate limiting: max 1 report per 10 minutes
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user_id ON journalist_sessions(user_id);

-- =====================================================
-- Row-Level Security Policies
-- =====================================================

ALTER TABLE intelligence_snapshots ENABLE ROW LEVEL SECURITY;
CREATE POLICY "snapshots_public_read" ON intelligence_snapshots
  FOR SELECT USING (true);  -- snapshots are not sensitive

ALTER TABLE intelligence_reports ENABLE ROW LEVEL SECURITY;
CREATE POLICY "reports_authenticated_read" ON intelligence_reports
  FOR SELECT USING (auth.role() = 'authenticated');

ALTER TABLE personal_report_cache ENABLE ROW LEVEL SECURITY;
CREATE POLICY "cache_own_rows_only" ON personal_report_cache
  FOR ALL USING (auth.uid()::text = user_id);

ALTER TABLE intelligence_config ENABLE ROW LEVEL SECURITY;
CREATE POLICY "config_authenticated_read" ON intelligence_config
  FOR SELECT USING (auth.role() = 'authenticated');
CREATE POLICY "config_service_role_write" ON intelligence_config
  FOR ALL USING (auth.role() = 'service_role');
```

---

## Section 10 — New File Structure

```
IRONSIGHT/
├── src/                              ← existing, untouched except page.tsx (1 import + 1 JSX line)
│   ├── app/
│   │   ├── api/
│   │   │   ├── alerts/route.ts       ← existing, NO changes
│   │   │   ├── conflicts/route.ts    ← existing, NO changes
│   │   │   ├── crypto/route.ts       ← existing, NO changes
│   │   │   ├── fires/route.ts        ← existing, NO changes
│   │   │   ├── flights/route.ts      ← existing, NO changes
│   │   │   ├── markets/route.ts      ← existing, NO changes
│   │   │   ├── news/route.ts         ← existing, NO changes
│   │   │   ├── oil/route.ts          ← existing, NO changes
│   │   │   ├── polymarket/route.ts   ← existing, NO changes
│   │   │   ├── regional-alerts/route.ts ← existing, NO changes
│   │   │   ├── ships/route.ts        ← existing, NO changes
│   │   │   ├── strikes/route.ts      ← existing, NO changes
│   │   │   ├── telegram/route.ts     ← existing, NO changes
│   │   │   └── intelligence/         ← NEW
│   │   │       ├── compress/
│   │   │       │   └── route.ts      ← POST: called by Vercel Cron every 5min
│   │   │       ├── report/
│   │   │       │   └── personal/
│   │   │       │       └── route.ts  ← POST: per-user report generation
│   │   │       ├── export/
│   │   │       │   ├── vizrt/
│   │   │       │   │   └── route.ts  ← GET: Vizrt JSON download
│   │   │       │   ├── geojson/
│   │   │       │   │   └── route.ts  ← GET: GeoJSON download
│   │   │       │   └── pdf/
│   │   │       │       └── route.ts  ← GET: Arabic PDF download
│   │   │       └── satellite/
│   │   │           └── route.ts      ← GET: Sentinel Hub image fetch
│   │   ├── globals.css               ← existing, NO changes
│   │   ├── layout.tsx                ← existing, NO changes
│   │   └── page.tsx                  ← MINIMAL change: +1 import, +1 JSX element
│   ├── components/
│   │   ├── map/ConflictMap.tsx        ← existing, NO changes
│   │   ├── panels/                    ← all existing, NO changes
│   │   └── intelligence/              ← NEW
│   │       ├── IntelligencePanel.tsx  ← main container, overlay at bottom of screen
│   │       ├── StoryFocusSelector.tsx ← focus input + 3 auto-suggestions
│   │       ├── PersonalReportView.tsx ← renders PersonalReport with attribution
│   │       ├── SituationReportView.tsx← renders base situation report
│   │       ├── ExportButtons.tsx      ← Vizrt | GeoJSON | PDF export buttons
│   │       ├── SatelliteViewer.tsx    ← before/after image comparison
│   │       ├── PriorityScoreBars.tsx  ← visual breakdown of 5 scoring signals
│   │       └── WeightSliders.tsx      ← editor-facing weight tuning (auth-gated)
│   ├── intelligence/                  ← NEW — entirely additive module
│   │   ├── workers/
│   │   │   ├── compressionWorker.ts  ← clustering, dedup, snapshot creation
│   │   │   ├── analysisWorker.ts     ← two LLM calls per cycle
│   │   │   └── personalReportWorker.ts ← per-user report generation
│   │   ├── agents/
│   │   │   ├── eventExtractor.ts     ← classify event type + target_category from text
│   │   │   ├── geoResolver.ts        ← resolve place names to coordinates + Arabic names
│   │   │   ├── priorityScorer.ts     ← composite score computation (no LLM)
│   │   │   └── verificationAssessor.ts ← source trust + verification status (no LLM)
│   │   ├── prompts/
│   │   │   ├── situationReport.ts    ← full system prompt for Worker A
│   │   │   ├── leadStory.ts          ← full system prompt for Worker B
│   │   │   └── personalReport.ts     ← full system prompt with attribution rules
│   │   ├── schemas/
│   │   │   ├── snapshot.ts           ← IntelligenceSnapshot TypeScript interface
│   │   │   ├── report.ts             ← PersonalReport TypeScript interface
│   │   │   └── export.ts             ← Vizrt, GeoJSON, PDF export interfaces
│   │   └── lib/
│   │       ├── supabase.ts           ← Supabase client for intelligence tables
│   │       ├── haversine.ts          ← clustering distance calculation
│   │       ├── tokenCounter.ts       ← ensure snapshots never exceed 3000 tokens
│   │       └── sourceTrust.ts        ← source domain → tier + trust score lookup
│   ├── data/city-data.json           ← existing, NO changes
│   ├── lib/                          ← all existing files, NO changes
│   └── types/index.ts                ← existing, NO changes
├── vercel.json                        ← NEW (or modified): Cron config
└── INTELLIGENCE_LAYER_PLAN.md        ← this document
```

**Total files requiring modification of existing code:** 1 (`src/app/page.tsx` — 2 lines added)

---

## Section 11 — Integration Points with Existing IRONSIGHT Code

### NewsFeed / /api/news
**What it does:** Fetches 26 RSS feeds, filters, translates, deduplicates, returns up to 100 `NewsItem[]`.  
**What the intelligence layer needs:** `title`, `source`, `pubDate` for text event clustering; `source` for trust tier lookup.  
**How it taps:** The compression worker calls `/api/news` directly (same endpoint). No modification needed.  
**Modification required:** None.

---

### TelegramPanel / /api/telegram
**What it does:** Scrapes 31 Telegram channels, translates non-Latin text, returns sorted `TelegramPost[]`.  
**What the intelligence layer needs:** `text`, `channelLabel`, `date` for event extraction; channel label for trust scoring.  
**Important:** The `postCache` and `latestKnownIds` in-memory variables in `telegram/route.ts` mean the compression worker will benefit from the panel's polling having already populated the cache. However, the compression worker must handle cold-cache gracefully (binary search will run on first call per channel per server restart).  
**How it taps:** The compression worker calls `/api/telegram` directly.  
**Modification required:** None.

---

### AlertsPanel / /api/alerts
**What it does:** Polls Pikud HaOref every 15 seconds, maintains 90-second sticky alert cache.  
**What the intelligence layer needs:** Alert count, locations, types, timestamps within the last 30 minutes.  
**How it taps:** The compression worker calls `/api/alerts` directly.  
**Modification required:** None.

---

### FlightsPanel / /api/flights
**What it does:** Fetches adsb.lol military + regional ADS-B, classifies aircraft, returns `MilFlight[]`.  
**What the intelligence layer needs:** All fields — the classification is already done by `classifyAircraft()`. Only `icao24`, `callsign`, `lat`, `lon`, `altitude`, `speed`, `type`, `aircraftType`, `origin` used.  
**How it taps:** The compression worker calls `/api/flights` directly.  
**Modification required:** None.

---

### SatellitePanel / /api/fires
**What it does:** Downloads NASA FIRMS CSV, filters to Middle East bbox, returns `FireEvent[]`.  
**What the intelligence layer needs:** `lat`, `lon`, `frp`, `brightness`, `datetime`, `intensity`, `possibleExplosion`.  
**How it taps:** The compression worker calls `/api/fires` directly.  
**Modification required:** None.

---

### ConflictFeed / /api/conflicts
**What it does:** Google News RSS → `ConflictEvent[]` with keyword-based type/location classification. **lat/lon always 0.**  
**What the intelligence layer needs:** `title`, `type`, `location` (text), `date`, `source`. Must geo-resolve `location` string.  
**How it taps:** The compression worker calls `/api/conflicts`. The `geoResolver` agent handles coordinate extraction from the `location` text field using the same `STRIKE_LOCATIONS` dictionary that `ConflictMap.tsx` uses (this dictionary is extracted to `src/intelligence/lib/geoDict.ts`).  
**Modification required:** None to existing files. The geo dictionary is copied (not moved).

---

### StrikesPanel / /api/strikes
**What it does:** Google News RSS → `StrikeEvent[]` with category/severity/country classification.  
**What the intelligence layer needs:** `title`, `category`, `severity`, `country`, `date`, `source`.  
**How it taps:** Compression worker calls `/api/strikes` directly.  
**Modification required:** None.

---

### ConflictMap
**What it does:** Independently polls 6 APIs, geocodes strikes client-side via `geocodeStrike()` + `STRIKE_LOCATIONS` lookup, renders Leaflet map with aircraft/naval/alert/strike markers.  
**What the intelligence layer needs:** Nothing directly from ConflictMap — the map's data is already sourced from the same APIs the compression worker calls.  
**Integration opportunity:** `IntelligencePanel.tsx` can dispatch a `'map-focus'` CustomEvent (same mechanism FlightsPanel uses at `ConflictMap.tsx:451`) to fly the map to a cluster centroid when a journalist clicks a cluster in their personal report. This requires no modification to ConflictMap.  
**Modification required:** None.

---

### NavalPanel / /api/ships
**What it does:** Returns hardcoded OSINT naval positions — static data, not live AIS.  
**What the intelligence layer needs:** Region and vessel type for narrative context only. Coordinates are not reliable enough for geospatial clustering.  
**How it taps:** Compression worker calls `/api/ships` directly. Naval data is placed in the snapshot's narrative context but NOT used for signal correlation (due to approximate nature).  
**Modification required:** None.

---

### MarketsPanel + OilPanel / /api/markets + /api/oil
**What it does:** Yahoo Finance unofficial API for defense stocks, indices, oil prices, VIX.  
**What the intelligence layer needs:** WTI, Brent, VIX values for 30-minute change detection.  
**How it taps:** Compression worker fetches `/api/oil` and `/api/markets`. Compares to `prev_oil_price_wti`, `prev_vix` stored in `intelligence_config`. Updates stored values after comparison.  
**Modification required:** None.

---

### page.tsx (Dashboard)
**Current state:** Imports 14 panel components, renders a 12-column CSS grid with 3 rows.  
**Required modification:** Add `IntelligencePanel` import and one JSX element. The panel renders as a fixed-position overlay — it does not disrupt the existing grid layout.  

```tsx
// Addition at line 19 (after existing imports):
import IntelligencePanel from '@/components/intelligence/IntelligencePanel';

// Addition at line 134 (after </main>, before <footer>):
<IntelligencePanel />
```

This is the only modification to any existing file.

---

## Section 12 — Multi-User Architecture

### How 200-300 Concurrent Journalists Are Served

The key insight: **LLM cost does not scale with user count for shared reports**. Worker A and Worker B run once per 5-minute cycle, regardless of how many users are connected. The per-user cost is only for personal reports, and that cost is gated by a 10-minute rate limit.

### Supabase Realtime Setup

Each journalist's browser subscribes to a Supabase Realtime channel:
```typescript
// In IntelligencePanel.tsx
const channel = supabase
  .channel('intelligence-snapshots')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'intelligence_snapshots',
  }, (payload) => {
    handleNewSnapshot(payload.new as IntelligenceSnapshot);
  })
  .subscribe();
```

When the compression worker inserts a new snapshot, all 300 subscribers receive the update simultaneously via Supabase's WebSocket broadcast. This replaces 300 individual polling loops with a single server-side event.

**Connection count estimates:**
- 300 journalists × 1 WebSocket per journalist = 300 concurrent WebSocket connections
- Supabase Free tier: 500 concurrent connections — sufficient
- Supabase Pro tier: 10,000 concurrent connections — more than adequate

### Personal Report Cache Prevents Redundant LLM Calls

If 50 journalists are all covering the same story focus (e.g., "غارة اسرائيلية على تل أبيب"), only the **first** journalist to request a report after the 10-minute cache window triggers an LLM call. The remaining 49 read from `personal_report_cache` instantly.

Cache lookup in `personalReportWorker.ts`:
```typescript
async function getOrGeneratePersonalReport(
  userId: string,
  storyFocus: string,
  snapshotId: string
): Promise<PersonalReport> {
  const hash = sha256(userId + normalizeArabic(storyFocus));

  // 1. Check cache
  const { data: cached } = await supabase
    .from('personal_report_cache')
    .select('report_data, expires_at')
    .eq('user_id', userId)
    .eq('story_focus_hash', hash)
    .gt('expires_at', new Date().toISOString())
    .single();

  if (cached) return cached.report_data as PersonalReport;

  // 2. Check rate limit
  const { data: session } = await supabase
    .from('journalist_sessions')
    .select('last_report_at')
    .eq('user_id', userId)
    .single();

  const lastReport = session?.last_report_at ? new Date(session.last_report_at) : null;
  const minutesSinceLast = lastReport
    ? (Date.now() - lastReport.getTime()) / 60000
    : Infinity;

  if (minutesSinceLast < 10) {
    throw new RateLimitError('Report generation rate limit: 1 per 10 minutes');
  }

  // 3. Generate and cache
  const report = await generatePersonalReport(userId, storyFocus, snapshotId);
  await cacheReport(userId, hash, storyFocus, report, snapshotId);
  return report;
}
```

### Rate Limiting Implementation

Token bucket per `user_id`, enforced at database level via `journalist_sessions.last_report_at`:
- Read `last_report_at` before any LLM call
- If `now - last_report_at < 10 minutes`: return 429 with `Retry-After` header
- After successful generation: update `last_report_at = now`
- This is atomic via Supabase row-level locking (the UPDATE is the lock)

### Fallback: 30-Second Polling

If a journalist's WebSocket connection drops (network interruption, browser tab backgrounded):
```typescript
// IntelligencePanel.tsx — fallback polling
useEffect(() => {
  const pollInterval = setInterval(async () => {
    const { data } = await supabase
      .from('intelligence_snapshots')
      .select('id, created_at')
      .order('created_at', { ascending: false })
      .limit(1)
      .single();

    if (data && data.id !== lastSnapshotId) {
      fetchFullSnapshot(data.id);
    }
  }, 30000);

  return () => clearInterval(pollInterval);
}, [lastSnapshotId]);
```

The Realtime subscription is primary; the 30-second poll is a degraded-mode fallback.

---

## Section 13 — What This Does Not Do

*Presented to Al Jazeera senior editorial staff for awareness before deployment.*

**Satellite imagery is archive-only.** Sentinel-2 optical imagery has a 5-day revisit cadence. Sentinel-1 SAR: 6-12 days. For a strike that occurred today, the most recent satellite image may be from 3-5 days before the event — it shows the pre-strike state. The acquisition date is displayed prominently to prevent misuse, but journalists must understand the temporal gap.

**Aircraft mission and intent are never inferred.** ADS-B data shows that a specific aircraft was at specific coordinates at a specific altitude moving at a specific speed. The system records these observed facts and nothing more. What the aircraft was doing, whether it was conducting a strike, reconnaissance, or a routine patrol — this is never stated. The existing IRONSIGHT `classifyAircraft()` function names aircraft types (F-35, AWACS, tanker) — these are platform identifications, not mission statements. This distinction must be understood by every journalist who reads the reports.

**NASA FIRMS thermal anomalies are not interpreted.** The system reports coordinates, timestamp, fire radiative power, and brightness temperature. The `possibleExplosion` flag in the existing code (frp > 80 && brightness > 380) is an internal operational flag — it is not surfaced in AI-generated text. The word "explosion" never appears in an AI-generated Arabic report unless a Tier 1 source has confirmed the cause. A thermal anomaly in a known industrial area may be a refinery flare. One in a known conflict zone may be an airstrike. The journalist decides.

**LLM outputs require editorial review before broadcast.** The system is an aid to the journalist, not a publisher. The footer of every PDF export states this explicitly: "تم إنشاء هذا التقرير بمساعدة الذكاء الاصطناعي — يخضع للمراجعة التحريرية". The `export_ready` flag in PersonalReport is a technical flag (verification status ≥ probable + ≥1 Tier 1 source), not an editorial approval.

**Arabic dialect vs. فصحى.** The system outputs Modern Standard Arabic (العربية الفصحى). Telegram source material may contain colloquial dialect, informal abbreviations, or regional expressions in Arabic, Farsi, and Hebrew. The quality of extraction from dialect-heavy Telegram posts will be lower than from formal wire service text. The `caveat_ar` field in `unconfirmed_reports` accounts for this, but editors should be aware that Telegram-sourced claims carry inherently higher uncertainty in the extraction layer, independent of the source trust scoring.

**GDELT / Google News lag.** The `/api/conflicts` and `/api/strikes` routes use Google News RSS, which typically has a 15-45 minute delay for breaking events. Telegram channels in the feed frequently report events 15-30 minutes before Google News aggregates them. The system assigns lower trust to Telegram (Tier 4), but in fast-breaking situations, the Telegram signal may be the earliest available. Journalists should monitor both the AI report and the raw Telegram feed for genuinely breaking events.

**Geo-resolution is approximate.** The `geoResolver` agent maps place names extracted from text to coordinates. For specific city names (Tel Aviv, Beirut, Tehran), accuracy is within 1km. For general descriptions ("southern Lebanon", "western Iran"), accuracy may be 20-50km. The `geo_confidence` field in each cluster reflects this. Coordinates with `geo_confidence < 0.75` are excluded from Vizrt broadcast output. They are included in GeoJSON with `geo_confidence` clearly marked.

**First 48 hours have no escalation history.** The escalation delta signal (Signal 2) requires 48 hours of stored snapshots to function. During the initial deployment period, this signal defaults to 0.5 (neutral) for all events, causing priority scores to depend more heavily on the other four signals. Editors should be aware that escalation detection is degraded until historical baseline is established.

**Naval data is static OSINT, not live AIS.** The `/api/ships` route returns hardcoded positions based on public DoD/Navy press releases (visible in `ships/route.ts` lines 139-314). These positions are approximate and may be days or weeks old. The system labels all naval data "last reported" — never "current position." Naval data is used for narrative context only and is excluded from signal correlation clustering.

**Unofficial endpoints are a production risk.** IRONSIGHT currently uses the unofficial Google Translate API (`translate.googleapis.com/translate_a/single?client=gtx`) in `lib/hebrew.ts:101` and the unofficial Yahoo Finance v8 API in `api/markets/route.ts` and `api/oil/route.ts`. These endpoints have no SLA, no authentication, and can be blocked or rate-limited at any time. For production deployment at a news organization, these must be replaced with:
- Google Cloud Translation API (official, $20/1M characters)
- Refinitiv / Bloomberg API for financial data
- Or acceptable alternative official sources

---

## Section 14 — Implementation Roadmap

### Phase 1 — Foundation (Weeks 1-2)

**Goal:** Get the compression worker running and producing valid snapshots with no LLM involved.

1. **Supabase setup:** Create all four tables with RLS policies as specified in Section 9. Seed `intelligence_config` with default weights.
2. **`src/intelligence/lib/` module:** Implement `haversine.ts`, `tokenCounter.ts`, `sourceTrust.ts`, `supabase.ts`.
3. **`src/intelligence/agents/geoResolver.ts`:** Port `STRIKE_LOCATIONS` and `CITIES` from ConflictMap into a server-side lookup. Add coordinate extraction from text for unresolved locations.
4. **`src/intelligence/agents/eventExtractor.ts`:** Keyword-based event type + target_category classification using the asset table in Section 4.1. No LLM in Phase 1 — rule-based only.
5. **`src/intelligence/agents/priorityScorer.ts`:** Implement all five scoring signals. Escalation delta returns 0.5 (bootstrap default) until history exists.
6. **`src/intelligence/agents/verificationAssessor.ts`:** Implement trust tier lookup and verification logic from Section 3.5.
7. **`src/intelligence/workers/compressionWorker.ts`:** Full clustering pipeline — collect, geo-resolve, cluster, verify, score, truncate to token budget, store snapshot.
8. **`src/app/api/intelligence/compress/route.ts`:** POST endpoint. Auth: verify `Authorization: Bearer [CRON_SECRET]` header (Vercel Cron includes this).
9. **`vercel.json`:** Add cron configuration.
10. **Validation:** Run the compression worker manually against live data. Inspect snapshots. Verify token counts stay under 3000. Verify cluster quality. Verify no lat:0,lon:0 events reach the clustering stage.

**Success criteria:** Snapshots appear in Supabase every 5 minutes with realistic cluster counts (3-15 clusters typically), all under 3000 tokens, with valid priority scores and verification statuses.

---

### Phase 2 — Analysis Workers (Weeks 3-4)

**Goal:** Produce Arabic situation reports and lead story recommendations from snapshots.

1. **`src/intelligence/prompts/situationReport.ts`:** Finalize Worker A prompt (Arabic, full attribution rules).
2. **`src/intelligence/prompts/leadStory.ts`:** Finalize Worker B prompt.
3. **`src/intelligence/workers/analysisWorker.ts`:** Two LLM calls after each compression cycle. Store results in `intelligence_reports`.
4. **Supabase Realtime:** Configure broadcast channel for snapshot inserts.
5. **`src/components/intelligence/IntelligencePanel.tsx`:** Basic read-only panel. Subscribes to Realtime. Renders `SituationReportView.tsx` with latest base report.
6. **`src/components/intelligence/SituationReportView.tsx`:** Renders Worker A output with source attribution visible.
7. **`src/app/page.tsx` modification:** Add single import and JSX element.
8. **Editorial review session:** Al Jazeera Arabic editors review first 48 hours of automated situation reports. Validate attribution rules. Adjust system prompt if any violations found.

**Success criteria:** Situation reports in Modern Standard Arabic appear every 5 minutes, each ≤250 words, with correct source attribution. Zero instances of aircraft mission inference or thermal anomaly cause stated without Tier 1 confirmation.

---

### Phase 3 — Personal Reports (Weeks 5-6)

**Goal:** Each journalist can get a tailored Arabic report for their specific story focus.

1. **`src/intelligence/prompts/personalReport.ts`:** Full personal report system prompt.
2. **`src/intelligence/workers/personalReportWorker.ts`:** Cache check → rate limit check → LLM call → cache store.
3. **`src/app/api/intelligence/report/personal/route.ts`:** POST endpoint with auth.
4. **`src/components/intelligence/StoryFocusSelector.tsx`:** Arabic text input + 3 auto-suggested focuses from Worker B.
5. **`src/components/intelligence/PersonalReportView.tsx`:** Renders `PersonalReport` with confirmed/unconfirmed split, attribution footnotes, `export_ready` badge.
6. **`journalist_sessions` table:** Wire up rate limiting.
7. **`personal_report_cache` table:** Wire up cache reads and writes.
8. **Load testing:** Simulate 300 concurrent focus queries. Verify rate limiting works. Verify cache prevents LLM storm.

**Success criteria:** Personal reports generate within 8 seconds of request. Cache returns in <200ms. Rate limiting correctly rejects requests within 10-minute window. Reports are Arabic, structured, and attribution is correct.

---

### Phase 4 — Exports + Satellite (Weeks 7-8)

1. **`src/app/api/intelligence/export/vizrt/route.ts`:** Vizrt JSON generation from PersonalReport.
2. **`src/app/api/intelligence/export/geojson/route.ts`:** GeoJSON FeatureCollection export.
3. **`src/app/api/intelligence/export/pdf/route.ts`:** Arabic PDF with RTL layout via `@react-pdf/renderer`.
4. **`src/components/intelligence/ExportButtons.tsx`:** Three download buttons with file naming convention: `{focus_hash}_{timestamp}.json/geojson/pdf`.
5. **`src/app/api/intelligence/satellite/route.ts`:** Sentinel Hub API integration. Mode A (single image) first, Mode B (before/after) second.
6. **`src/components/intelligence/SatelliteViewer.tsx`:** Image display with acquisition date prominently labeled.
7. **Vizrt integration test:** Provide sample Vizrt JSON to Al Jazeera broadcast team. Validate it loads in Viz Trio without manual data entry.

**Success criteria:** All three export formats work. Vizrt JSON accepted by Al Jazeera broadcast graphics workflow. Satellite images display with correct acquisition dates. PDF renders correctly in Arabic RTL.

---

### Phase 5 — Polish + Hardening (Weeks 9-10)

1. **`src/components/intelligence/WeightSliders.tsx`:** Editor-facing weight tuning UI, auth-gated. Writes to `intelligence_config` via protected API route.
2. **`src/components/intelligence/PriorityScoreBars.tsx`:** Visual breakdown of the 5 scoring signals per cluster so editors can understand why a story was surfaced.
3. **Performance testing:** 300 concurrent Supabase Realtime connections. Verify no connection saturation. Test fallback polling.
4. **Replace unofficial endpoints:**
   - `lib/hebrew.ts:101` → Google Cloud Translation API
   - `api/markets/route.ts` + `api/oil/route.ts` → official financial data API
5. **Full Arabic RTL UI:** All intelligence panel text properly right-aligned. Input fields accept Arabic input without cursor issues. PDF renders with proper Arabic OpenType font (Noto Naskh Arabic recommended).
6. **Editorial review of attribution rules:** Final review session with Al Jazeera Arabic senior editors. Focus on: are there any cases where the system states uncertain things as fact? Document all adjustments to prompts.
7. **`INTELLIGENCE_LAYER_PLAN.md` archival:** Update this document with any decisions that changed during implementation.

---

## Section 15 — Key Design Decisions and Rationale

### Decision 1: Pre-Computation vs. Per-User LLM Calls for the Situation Report

**Decided:** Worker A (situation report) runs once per 5-minute compression cycle. All users read the same base report.

**Why:** At 300 concurrent journalists, per-request LLM calls for the situation report would cost $0.045 × 300 × 12 = $162/hour — $3,888/day for what is substantively the same content. Pre-computation produces one call at $0.045/cycle regardless of user count.

**Alternative rejected:** Generating a fresh situation report for each journalist's page load. Rejected because the snapshot input doesn't change between requests in the same 5-minute window — calling the LLM 300 times on identical input is pure waste with identical output.

**Trade-off:** The situation report is up to 5 minutes old when a user reads it. This is acceptable because the snapshot itself is 5 minutes old — the situation report cannot be more current than its input. For truly breaking events (missile alerts), the alert data updates every 15 seconds in the existing Pikud HaOref panel, which is separate from the AI report.

---

### Decision 2: Structured JSON Output Schema vs. Raw Prose from LLM

**Decided:** The LLM outputs a structured JSON schema (`PersonalReport`) on every call. All fields are explicitly typed.

**Why:** Raw prose from an LLM cannot be programmatically parsed for export to Vizrt, GeoJSON, or PDF without a second parsing pass. Structured output enables direct machine consumption. It also enforces the attribution schema — the LLM cannot omit the `source_tier` field on a confirmed claim because the schema requires it.

**Alternative rejected:** Have the LLM write free-form Arabic prose, then parse it to extract structure. Rejected because: (1) Arabic NLP parsing is fragile, (2) requires a second LLM call to parse, doubling cost, (3) free-form prose makes the attribution rules harder to enforce and audit.

**Trade-off:** Structured output is slightly less naturally flowing than pure prose. Mitigation: the `lead_paragraph_ar` field is free prose; structure is applied to the secondary sections where it adds value.

---

### Decision 3: Separate Priority Score from Verification Status

**Decided:** `priority_score` (0-10) and `verification_status` (confirmed|probable|unverified|contested) are separate fields, never merged.

**Why:** A high-priority unverified event (e.g., single Telegram channel reports strike on nuclear facility) must be surfaced to the journalist precisely because it is high-priority — but the journalist must see clearly that it is unverified. If priority score and verification were merged into a single field, the unverified high-priority event might be suppressed (low final score) and missed entirely, which is worse editorially than surfacing it with clear caveats.

**Alternative rejected:** Multiply priority_score by verification confidence (confirmed=1.0, probable=0.7, unverified=0.3). Rejected because this suppresses potentially critical but fast-breaking events that haven't yet been corroborated.

---

### Decision 4: Aircraft Data — Observation Only, Never Interpretation

**Decided:** The system records ADS-B data as observed facts and never infers mission, purpose, or intent from flight path or aircraft type.

**Why:** This is both an editorial integrity requirement and a legal one. An erroneous claim that a specific aircraft conducted a specific strike, published under Al Jazeera's brand, with only ADS-B data as the basis, would be a severe credibility and legal risk. The existing IRONSIGHT `classifyAircraft()` function uses the word "ISR Drone" or "Fighter" — these are platform type identifications (what the aircraft is), not mission descriptions (what it was doing). This distinction is maintained throughout the intelligence layer.

**Alternative rejected:** Inferring likely mission from aircraft type, area, altitude, and flight path combined. Rejected categorically. ADS-B alone cannot distinguish a training flight from a strike package, a reconnaissance mission from a transit, or a show of force from an attack run.

---

### Decision 5: Discrete Timestamped Snapshots vs. Continuously Updating Report

**Decided:** Each compression cycle produces an immutable `IntelligenceSnapshot` with a fixed `created_at`. Reports reference their `snapshot_id`. Previous snapshots are  retained for 7 days.

**Why:** Immutable snapshots create an audit trail. A journalist can look at a report they generated at 14:35 and compare it to the 14:30 snapshot — and see exactly what data was available when the report was written. This is critical for editorial accountability. If the AI system makes a mistake, the cause can be traced to the snapshot that drove it.

**Alternative rejected:** Continuously updating a single "current state" object. Rejected because: (1) no audit trail, (2) race conditions if multiple workers update simultaneously, (3) makes it impossible to compare the state at two points in time.

---

### Decision 6: Haversine Clustering at 0.1° Grid vs. Larger/Smaller Grid

**Decided:** 0.1° grid cell (≈ 11km) for event clustering.

**Why:** At 0.1°, events in Tel Aviv and events in Dimona (35km apart) correctly fall in separate clusters. Events in the same neighborhood of Beirut correctly cluster together. A 0.5° grid would merge distinct incidents across an entire city into one misleading cluster. A 0.05° grid would split a single incident reported by multiple sources into separate clusters, defeating deduplication.

**Calibration example:**
- Tel Aviv (32.09°N, 34.78°E) → grid key `(320, 347)`
- Dimona (31.07°N, 35.03°E) → grid key `(310, 350)` — separate cluster ✓
- Two reports from central Beirut (33.89°N, 35.50°E) and (33.90°N, 35.50°E) → both grid key `(338, 355)` — same cluster ✓

**Alternative rejected:** DBSCAN dynamic clustering. Rejected because it requires a tunable ε parameter, is computationally heavier, and produces variable-size clusters that are harder to reason about for the token budget calculation.

---

### Decision 7: GPT-4o / claude-opus-4-8 for Extraction vs. Smaller Models

**Decided:** Use the most capable available model (claude-opus-4-8 or GPT-4o) for Worker A (situation report) and personal report generation. Use the smallest capable model (claude-haiku-4-5 or GPT-4o-mini) for Worker B (lead story recommendation).

**Why Worker A needs the best model:** Arabic fصحى output with strict attribution rules is a high-complexity task. Smaller models hallucinate Arabic constructions, drop attribution requirements, or produce grammatically poor Arabic under the attribution constraints. The situation report is published to 300 journalists — quality matters.

**Why Worker B can use a smaller model:** Ranking 5 clusters and explaining the editorial rationale is a structured, lower-complexity task. The input is already scored and ranked; the model only needs to translate the mechanical ranking into editorial Arabic. Haiku/GPT-4o-mini handles this reliably at 1/10th the cost.

**Alternative rejected:** Using the same model for all calls. Rejected on cost grounds — using Opus for Worker B would add $0.004 × 12/hour = $1.15/day unnecessarily.

---

### Decision 8: Supabase Realtime vs. Polling for 300-User Serving

**Decided:** Supabase Realtime WebSocket subscriptions as primary delivery mechanism for snapshot updates.

**Why:** Polling at 30-second intervals from 300 clients = 600 HTTP requests/minute to Supabase — significant load and unnecessary cost. Realtime push means one server-side event triggers 300 simultaneous deliveries at negligible marginal cost. With Supabase Pro plan (10,000 concurrent connections), 300 connections uses 3% of capacity.

**Alternative rejected:** Each journalist polls the `/api/intelligence/compress` endpoint every 30 seconds. Rejected because: (1) creates query fan-out that scales with user count, (2) introduces up to 30-second latency after snapshot publication, (3) drains Supabase read quota unnecessarily.

**Fallback retained:** The 30-second polling fallback is still implemented because WebSocket connections can drop (mobile networks, browser tab backgrounding, corporate firewalls). The fallback ensures degraded-but-functional behavior without relying on WebSocket reliability.

---

## Post-Audit Summary

### The 3 Most Important Findings from the Codebase

**1. ConflictEvent lat/lon are always 0.** The `/api/conflicts/route.ts` endpoint explicitly sets `lat: 0` and `lon: 0` for every event (lines 81-90). Geocoding happens entirely client-side in `ConflictMap.tsx` via the `geocodeStrike()` function and `STRIKE_LOCATIONS` dictionary. This means the intelligence layer must implement its own geo-resolution for conflict events — it cannot rely on the existing API. The `geoResolver.ts` agent was designed specifically to address this gap. The `STRIKE_LOCATIONS` dictionary at `ConflictMap.tsx:141-163` and `geocodeStrike()` at line 260 are the reference implementation to port.

**2. Duplicate API calls throughout the UI.** `ConflictMap.tsx` independently polls `/api/flights`, `/api/ships`, `/api/alerts`, `/api/conflicts`, `/api/strikes`, and `/api/telegram` — the same six APIs already polled by their respective panels. Every one of these APIs is therefore called twice per poll interval in normal operation. The compression worker adds a third caller for most of these APIs. The APIs are stateless and idempotent, so this causes no correctness issue — but at high load, the Telegram scraping endpoint (which does outbound fetches to t.me for each of 31 channels) could see 3× its current request volume. The `postCache` and `latestKnownIds` in-memory variables in `telegram/route.ts` (lines 58-60) mitigate this for Telegram specifically.

**3. Naval data is static, not live.** The `ships/route.ts` route (lines 136-314) returns hardcoded approximate positions from public DoD press releases — `lastReported: new Date().toISOString()` (set on every request, not when data was actually reported). This means the `lastReported` field is misleading — it always shows "now" regardless of when the position was actually known. For the intelligence layer, this means: do NOT use naval position data for signal correlation. Use it for narrative context only. The plan documents this clearly in Section 11 (NavalPanel integration point) and Section 13 (limitations).

### Assumptions Made Due to Codebase Ambiguity

**1. AP feed not in the codebase.** The plan designates Reuters, AP, AFP as Tier 1. However, the existing `news/route.ts` has no direct AP RSS feed — only Reuters. AP content appears via Google News aggregation (`source: 'Google News'`). The trust tier lookup in `sourceTrust.ts` handles this: if `source === 'AP'` it gets Tier 1 treatment, but in practice, AP content arriving via Google News gets the `'Google News'` → Tier 3/0.45 default. **Assumption:** An official AP RSS feed should be added to `RSS_FEEDS` in `news/route.ts` for proper Tier 1 coverage. This is flagged as a recommendation, not a blocking issue for Phase 1.

**2. User authentication mechanism.** The plan requires `user_id` for personal report caching and rate limiting, and `auth.uid()` for Supabase RLS. IRONSIGHT currently has no authentication — it is an open dashboard. **Assumption:** Supabase Auth will be added (email/password or SSO via Al Jazeera's existing IdP) as part of Phase 3 deployment. The personal report system cannot function without it. The base situation report (Worker A output) can be served unauthenticated.

**3. Vercel deployment.** The plan uses Vercel Cron for the compression worker trigger. **Assumption:** IRONSIGHT will be deployed on Vercel (consistent with its Next.js architecture and `force-dynamic` API routes). If deployed elsewhere, the cron trigger must be replaced with an equivalent scheduled task (Railway cron, AWS EventBridge, etc.).

### Files That Require Modification

**Only one existing file requires modification:**

`src/app/page.tsx` — two lines added (one import, one JSX element).

All other existing files — all 13 API routes, all 14 panel components, the map component, the layout, the hooks, the lib utilities, and the type definitions — remain entirely unchanged. The intelligence layer is imported as a module, not woven through the existing codebase.

---

*Document prepared: 2026-06-08. Review before implementation. All file paths verified against current codebase state.*
