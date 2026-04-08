# Phase 0 Notes — USGS API Exploration
*RiverRecords Project — Completed April 2026*

---

## What Phase 0 Was

Before writing any application code, the goal was to study the USGS Water Services API directly — hitting real endpoints, reading real responses, and answering every question that affects the database schema and the `lib/usgs.ts` module design. The findings here are the reference document kept open during Phase 3 when those modules get built.

---

## Background: What "Hitting an API" Means

An API (Application Programming Interface) is a server that accepts HTTP requests and returns data instead of a webpage. Instead of returning HTML that a browser renders, it returns raw structured data — in USGS's case, JSON.

**JSON (JavaScript Object Notation)** is a text format for representing structured data using nested objects `{}` and arrays `[]`. Example:

```json
{
  "siteName": "MISSISSIPPI RIVER AT PRESCOTT, WI",
  "readings": [
    { "value": "30600", "dateTime": "2026-04-08T15:00:00.000-05:00" }
  ]
}
```

**Hitting an API** means sending an HTTP GET request to a URL and reading the JSON response back. The URL contains parameters (after the `?`) that tell the server what data you want. You can do this in a browser, with `curl` in the terminal, or in code using `fetch()`.

The USGS API is public and requires no API key — anyone can call it. The base endpoint for instantaneous (real-time) values is:

```
https://waterservices.usgs.gov/nwis/iv/
```

---

## Step 1 — Selected River Sections

Four Wisconsin river sections were selected as the initial seed data. These will be the rivers pre-populated in the database for the Phase 2 seed file.

| Site Code | River / Section | Location | Parameters Available |
|---|---|---|---|
| `05344500` | Mississippi River at Prescott | Prescott, WI | Gauge height, Discharge |
| `05369500` | Chippewa River at Durand | Durand, WI | Gauge height, Discharge |
| `05342000` | Kinnickinnic River Near River Falls | River Falls, WI | Gauge height, Discharge |
| `05340500` | St. Croix River at St. Croix Falls | St. Croix Falls, WI | Gauge height, Discharge, Temperature |

**Note:** Only the St. Croix has water temperature instrumentation. The other three have gauge height and discharge only. This is the normal distribution in the real world — temperature sensors are less common. The schema handles this with nullable fields.

---

## Step 2 — The Instantaneous Values (IV) Endpoint

### What It Is

The IV endpoint returns the most recent real-time reading(s) for a station. USGS stations record data at 5-minute intervals. This is the endpoint used for:
- Current conditions display on the river section page
- The hourly cron job that writes `StreamReading` records to the database

### Endpoint Format

```
https://waterservices.usgs.gov/nwis/iv/
  ?format=json
  &sites={site_code}
  &parameterCd=00060,00065,00010
  &siteStatus=active
```

**Parameter codes:**
- `00060` = Discharge (cubic feet per second, CFS)
- `00065` = Gauge height (feet)
- `00010` = Water temperature (degrees Celsius)

For the 7-day historical trend, add `&period=P7D` to the same endpoint. `P7D` is ISO 8601 duration notation for "7 days." For a 2-hour window use `PT2H`.

### Example URLs (live, tested)

```
# Current reading — Mississippi at Prescott
https://waterservices.usgs.gov/nwis/iv/?format=json&sites=05344500&parameterCd=00060,00065,00010&siteStatus=active

# 7-day trend — St. Croix at St. Croix Falls
https://waterservices.usgs.gov/nwis/iv/?format=json&sites=05340500&parameterCd=00060,00065,00010&period=P7D&siteStatus=active
```

---

## Step 3 — JSON Response Structure

### Top-Level Shape

The response is deeply nested. The useful data is buried under `response.value.timeSeries`:

```
response
└── value
    ├── queryInfo         ← metadata about the request (not needed)
    └── timeSeries[]      ← array of data series, one per parameter requested
```

### The `timeSeries` Array

This is the most important thing to understand about the USGS response. When you request three parameters (`00060,00065,00010`), the API returns an array with **one element per parameter** — so three elements total. Each element contains data for a single measurement type.

Confirmed from live responses (2-hour window on St. Croix `05340500`):

| Index | Variable Code | Variable Name |
|---|---|---|
| 0 | `00010` | Temperature, water, °C |
| 1 | `00060` | Streamflow, ft³/s |
| 2 | `00065` | Gage height, ft |

**Critical:** The order of elements in the array is not guaranteed to be the same as the order of `parameterCd` in the request URL. You cannot safely access `timeSeries[0]` and assume it's discharge. You must find the right element by checking `variable.variableCode[0].value` against the code you want.

### Structure of One `timeSeries` Element

```
timeSeries[n]
├── sourceInfo
│   ├── siteName              → "MISSISSIPPI RIVER AT PRESCOTT, WI"
│   ├── siteCode[0].value     → "05344500"
│   └── geoLocation
│       └── geogLocation
│           ├── latitude      → 44.747085
│           └── longitude     → -92.802152
│
├── variable
│   ├── variableCode[0].value → "00060"   ← use this to identify which parameter
│   ├── variableName          → "Streamflow, ft³/s"
│   └── noDataValue           → -999999.0  ← sentinel for missing data
│
└── values[0]
    └── value[]               ← array of individual readings
        └── value[n]
            ├── value         → "30600"    ← STRING, not a number
            ├── dateTime      → "2026-04-08T15:00:00.000-05:00"
            └── qualifiers    → ["P"]      ← "P" = provisional data
```

### Key Path to Extract a Reading

```
response.value.timeSeries[n].values[0].value[0].value     → the measurement (string)
response.value.timeSeries[n].values[0].value[0].dateTime  → ISO 8601 timestamp with offset
response.value.timeSeries[n].variable.variableCode[0].value → parameter code ("00060" etc.)
response.value.timeSeries[n].variable.noDataValue          → -999999.0
```

---

## Step 4 — Critical Findings for `lib/usgs.ts`

### Finding 1: Values are strings, not numbers

The `value` field in every reading is a **string**: `"30600"`, not `30600`. The parser must call `parseFloat()` on every value before storing or using it.

### Finding 2: Missing data is a sentinel value, not null

When a parameter has no data, USGS does not return `null` — it returns the sentinel value `-999999.0` (stored in `variable.noDataValue`). The parser must check every reading against `noDataValue` and convert it to `null` before storing. If this is not handled, a gauge height of `-999999` will make it into the database and break the fishability score and chart rendering.

### Finding 3: Missing parameters are absent from the array

For sites that don't have temperature instrumentation (Mississippi, Chippewa, Kinnickinnic), the `timeSeries` array will contain **2 elements** (discharge and gauge height) with no `00010` entry at all. The parser must handle the case where `find("00010")` returns `undefined` — not an error, just null temperature.

### Finding 4: Timestamps include timezone offset

Timestamps look like `"2026-04-08T15:00:00.000-05:00"` (CST). JavaScript's `new Date()` parses this correctly. All timestamps should be stored as UTC in PostgreSQL — Prisma handles this automatically when you pass a `Date` object.

### Finding 5: Readings are at 5-minute intervals

USGS stations record at **5-minute intervals**, not 15 as originally assumed. A 7-day fetch returns approximately **2,016 data points per parameter** (7 days × 24 hours × 12 readings/hour). That is too many to send to the browser or pass to Claude.

**Downsampling strategy:** When fetching the 7-day trend, filter the response array to keep only readings where the minutes value is `:00` — one reading per hour. This reduces 2,016 points to ~168, which is a reasonable chart resolution without losing meaningful trend shape.

---

## Step 5 — Parser Pattern for `lib/usgs.ts`

Based on all of the above, the parsing logic for a raw USGS response should follow this pattern:

```typescript
// Find a timeSeries element by parameter code
const find = (code: string) =>
  response.value.timeSeries.find(
    (s: any) => s.variable.variableCode[0].value === code
  );

// Extract the most recent value, returning null if missing or no-data
const parseLatestValue = (series: any): number | null => {
  if (!series) return null;
  const raw = series.values[0]?.value[0]?.value;
  const noData = series.variable?.noDataValue;
  if (raw === undefined || raw === null) return null;
  const n = parseFloat(raw);
  return n === noData ? null : n;
};

// Parse timestamp from most recent reading
const parseTimestamp = (series: any): Date | null => {
  const dt = series?.values[0]?.value[0]?.dateTime;
  return dt ? new Date(dt) : null;
};

// Usage
const dischargeSeries   = find("00060");
const gaugeSeries       = find("00065");
const temperatureSeries = find("00010"); // undefined for most sites — that is fine

const reading = {
  discharge_cfs:    parseLatestValue(dischargeSeries),
  gauge_height_ft:  parseLatestValue(gaugeSeries),
  temperature_c:    parseLatestValue(temperatureSeries), // null if site has no temp sensor
  reading_timestamp: parseTimestamp(dischargeSeries ?? gaugeSeries),
};
```

---

## Step 6 — Schema Implications Confirmed

These findings confirm that the `StreamReading` Prisma model in the SRS is correct as written:

```prisma
model StreamReading {
  id                Int          @id @default(autoincrement())
  section_id        Int
  section           RiverSection @relation(fields: [section_id], references: [id])
  gauge_height_ft   Float        // never null in practice — all 4 sites have this
  discharge_cfs     Float        // never null in practice — all 4 sites have this
  temperature_c     Float?       // nullable — only St. Croix has this
  reading_timestamp DateTime
  ingested_at       DateTime     @default(now())
}
```

`temperature_c` being `Float?` is the correct call. All downstream features (fishability score, conditions panel, catch snapshot, AI context) must handle a null temperature gracefully.

---

## Step 7 — Cron and Caching Strategy Confirmed

- USGS is a public government API with no API key required
- No documented hard rate limit, but the responsible use guidance recommends not polling more than once per hour per site for automated clients
- The 1-hour cron job cadence and 1-hour Redis TTL confirmed as appropriate
- Do not poll USGS from the browser — all calls go through the server-side cron or on-demand server route

---

## What Comes Next (Phase 1)

With Phase 0 complete:
- All 4 seed site codes are confirmed with their available parameters
- The JSON response structure is fully mapped
- The parser pattern is designed
- The schema is validated against real data

Phase 1 is project setup: Next.js, TailwindCSS, Clerk, Prisma, Railway databases, environment variables, GitHub repository, and initial Vercel deploy. No USGS code yet — that is Phase 3.
