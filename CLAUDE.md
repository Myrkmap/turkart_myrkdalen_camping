# Turkart Myrkdalen Camping

## Prosjektstruktur
- `index.html` — hovudfil for webkartet (MapLibre GL JS)
- `vilkar.js` — brukarvilkår (rediger her, ikkje i index.html)
- `personvern.js` — personvernerklæring (rediger her, ikkje i index.html)
- `ikoner/` — PNG-ikoner for interessepunkt
- `.github/workflows/keep-alive.yml` — GitHub Actions som held Supabase aktiv kvar 3. dag

## Supabase
- **Prosjekt-ID:** `tebynwryfadpnekxybpq`
- **URL:** `https://tebynwryfadpnekxybpq.supabase.co`
- **Tilkopling for QGIS (Session Pooler):**
  - Host: `aws-1-eu-north-1.pooler.supabase.com`
  - Port: `5432`
  - Database: `postgres`
  - Brukarnamn: `postgres.tebynwryfadpnekxybpq`

## Arbeidsflyt – komplett frå GPS-track til kart

### ⚠️ ALDRI bruk Kopier → Lim inn mellom lag i QGIS — det drep Z-koordinatane!
### Bruk alltid DB Manager for import til Supabase.

---

### Steg 1 – Samle inn GPS-track i felt
- Bruk Garmin, telefon (t.d. Komoot, Strava, OsmAnd) eller liknande
- Eksporter som `.gpx` eller `.geojson`

### Steg 2 – Importer track til QGIS
- `Lag → Legg til lag → Vektorlag` → vel `.gpx`-fila
- Vel laget `tracks` (ikkje `waypoints`)
- Kontroller at stien ser riktig ut i kartet

### Steg 3 – Rediger og tilpass til eksisterande stinett
- Legg til `features`-laget frå Supabase i QGIS som referanse (skriveverna)
- Rediger track-laget:
  - Klyp/forleng endepunkt så dei snapper til eksisterande stiar
  - Fjern duplikat-segment der den nye stien overlapper eksisterande
  - Del opp stien viss ho skal registrerast som fleire separate features
  - Glatt ut GPS-støy ved behov (`Forenkle geometri` med låg toleranse)
- Sjekk topologi: endepunkt skal treffe kvarandre, ikkje henge i lause lufta
- Lagre det redigerte laget som eige lag (t.d. `sti_redigert`) før Drape

### Steg 4 – Køyr Drape (legg til Z frå DTM10)
- `Behandling → Verktøykasse → Sett Z-verdi frå raster` (søk på «drape»)
- **Input:** `sti_redigert`-laget
- **Raster:** DTM10 frå Kartverket
- **Output:** lagre som mellombordplate (t.d. `sti_drape`)
- Køyr og sjekk at resultatet har Z-verdiar (høgreklikk → Eigenschaften → Info)

### Steg 5 – Importer til Supabase via DB Manager (IKKJE lim inn!)
1. `Database → DB Manager → DB Manager`
2. Utvid `PostGIS → Supabase → public`
3. Klikk **Import Layer/File** (pil-inn-ikon øvst)
4. **Input layer:** vel `sti_drape`-laget
5. **Table:** `features_import` (ny mellombordplate — ikkje direkte til `features`)
6. **Schema:** `public`
7. **Geometry column:** `geom`
8. **Source CRS:** EPSG:4326
9. ✅ Create spatial index
10. Klikk **OK**

### Steg 6 – Flytt frå mellombordplate til `features` i Supabase SQL-editor
```sql
-- Sjekk at Z er med (skal vise 3)
SELECT ST_CoordDim(geom), ST_AsText(ST_StartPoint(geom))
FROM features_import LIMIT 3;

-- Insert til features (fyll inn attributtar her)
INSERT INTO features (geom, navn, type, beskrivelse, merka, status)
SELECT
  geom,
  'Namnet på stien'   AS navn,
  'sti'               AS type,
  'Beskriving her'    AS beskrivelse,
  true                AS merka,
  'publisert'         AS status
FROM features_import;

-- Rydd opp
DROP TABLE features_import;
```

Triggeren `sync_feature_properties` ordnar resten automatisk:
- Reknar ut `lengde_km`
- Oppdaterer `properties` JSONB
- Set `created_at` / `updated_at`

### Steg 7 – Verifiser i databasen
```sql
SELECT id, navn, lengde_km, ST_CoordDim(geom) AS dim
FROM features
ORDER BY created_at DESC
LIMIT 5;
```
`dim` skal vere `3`. `lengde_km` skal vere utrekna automatisk.

### Steg 8 – Sjekk i kartet
- Opne [turkart.no](https://turkart.no) (eller lokal preview)
- Klikk på den nye stien — panel skal opne seg med namn, lengde og høgdeprofil

---

## Viktig om geometri
- Geometri **må** vere 3D (Z-koordinatar frå DTM10) for at høgdeprofilane skal fungere
- QGIS copy-paste drep Z-koordinatane — alltid DB Manager!
- Triggeren brukar `ST_CurveToLine` (ikkje `ST_Force2D`) for å bevare Z

## Feilsøking – features har dim=2 (mista Z)
```sql
-- Finn alle 2D-features
SELECT id, navn, ST_CoordDim(geom) AS dim
FROM features
WHERE ST_CoordDim(geom) = 2;
```
Fix: køyr Drape på nytt i QGIS, importer til `features_z_fix` via DB Manager, køyr:
```sql
UPDATE features f
SET geom = z.geom
FROM features_z_fix z
WHERE f.navn = z.navn;
DROP TABLE features_z_fix;
```

## Ikoner
Legg nye PNG-ikoner i `ikoner/`-mappa og registrer dei i `IKON_MAP` i `index.html`:
```javascript
const IKON_MAP = {
  parkering:         { fil: 'parking',           farge: null },
  hytte:             { fil: 'hytte',             farge: null },
  myrkdalen_camping: { fil: 'myrkdalen_camping', farge: null },
  butikk:            { fil: 'butikk',            farge: null }
};
```
Hugs også å oppdatere filtera i `interessepunkt-sirklar` og `interessepunkt-ikon`-laga.
