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

## Arbeidsflyt – leggje til nye stiar

1. **Teikn/importer stien** i QGIS (GPS-track, manuelt digitalisert osv.)
2. **Køyr Drape** (`Sett Z-verdi frå raster`) på den nye stien med DTM10 frå Kartverket
3. **Kopier** feature frå drape-laget
4. **Lim inn** direkte i Supabase `features`-laget i QGIS
5. **Fyll inn attributtar** (namn, type, merka osv.) i attributtskjemaet
6. **Lagre**

Triggeren `sync_feature_properties` ordnar resten automatisk:
- Reknar ut `lengde_km`
- Oppdaterer `properties` JSONB
- Set `created_at` / `updated_at`

## Viktig om geometri
- Geometri skal vere 3D (Z-koordinatar frå DTM10) for at høgdeprofilane skal fungere
- Triggeren brukar `ST_CurveToLine` (ikkje `ST_Force2D`) for å bevare Z
- Bruk alltid Drape før import til Supabase

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
