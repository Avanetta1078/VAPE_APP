# CLAUDE.md — VA&PE App

## Regla #1 de trabajo
**Ejecutar lo que el usuario pide. No suponer, no contradecir, no desviarme.** Si el usuario indica dónde está un dato o cómo resolver algo, hacer exactamente eso primero. Verificar después.

## Proyecto
Aplicación monolítica de monitoreo de pozos petroleros. Un solo archivo HTML (`VAPE_APP1.html`, ~9000+ líneas) con todo el CSS, HTML y JS inline.

## Stack
- **Frontend**: HTML/CSS/JS vanilla (sin frameworks)
- **Backend**: Supabase (project ID: `fkfrhboiugnfmrfsbvjm`)
- **Repo**: `Avanetta1078/VAPE_APP` en GitHub

## Tablas principales en Supabase

| Tabla | Propósito |
|---|---|
| `wells` | Pozos (datos maestros + motor/contrapesos) |
| `well_configs` | Configuraciones de cada pozo |
| `tests` | Ensayos (DYN/ACU) con fecha, hora, tipo |
| `dyn_results` | Resultados del ensayo dinamométrico |
| `acu_results` | Resultados del ensayo acústico/nivel |
| `dyn_points` | Puntos de la carta dinamométrica |
| `acu_points` | Puntos del sonolog |
| `field_log` | Importaciones históricas del planillero |
| `bitacora_visitas` | Bitácora Online (app separada, no tocar) |
| `bitacora_contrapesos` | Bitácora Online contrapesos (no tocar) |

## Mapeo CSV del Echometer/TAM

### IMPORTANTE: Columnas con prefijo DYN vs ACU
El CSV tiene columnas duplicadas con prefijos distintos. Ejemplo:
- `ACU Tubing Pressure (kilo (g))` → presión de tubing del ensayo acústico
- `DYN Tubing Pressure (kilo (g))` → presión de tubing del ensayo dinámico (**esta es la correcta para dyn_results**)
- `ACU Casing Pressure (kilo (g))` vs `DYN Casing Pressure (kilo (g))` → mismo caso

**Nunca usar un fallback genérico que agarre la primera columna "tubing pressure" — siempre buscar con el prefijo DYN para campos de dyn_results.**

### DYN_FIELD_MAP (campos clave)
```
dyn_tubing_pressure_kg → 'DYN Tubing Pressure (kilo (g))'
dyn_casing_pressure_kg → 'DYN Casing Pressure (kilo (g))'
pip_dyn_kg → 'PIP DYN (kilo (g))'
pump_discharge_pressure_kg → 'Pump Discharge Pressure (kilo (g))'
adjusted_fillage_pct → 'Adjusted Fillage (%)'
```

### Parseo de headers CSV
- Los headers pueden venir con comillas (`"header"`), se limpian con `.replace(/^"|"$/g, '')`
- El delimitador puede ser `;` (Echometer) o `,`
- `findCsvColumnValue()` busca: exacto → case-insensitive → última palabra como substring

## Caché en memoria (WELLS_DB)
`WELLS_DB[wellName]` se puebla en `loadLiveDataFromSupabase()`. Campos relevantes:
- `tubpres` ← `dyn_tubing_pressure_kg` de `dyn_results`
- `pipdyn` ← `pip_dyn_kg`
- `fillage` ← `adjusted_fillage_pct`
- `motorMarca`, `motorRpm`, `motorHp`, `motorPolea` ← columnas de `wells`
- `contrapesosRaw` ← columna `contrapesos` de `wells`
- `hasAcu`, `hasDyn` ← si tiene ensayos acústicos/dinámicos

## Matching tolerante de pozos (planillero)
4 niveles: exacto → lowercase → strip dashes/spaces → solo dígitos.
Funciones: `normLower()`, `normStrip()`, `normDigits()`, `matchWell()`.

## Vistas principales
- **Inicio**: grilla de pozos con filtros
- **Menú DINA (Resumen Pozo)**: carta dinamométrica + chips de datos + contrapesos
- **Menú Nivel**: panel acústico con wellSvg + buildup + tramos
- **Consultas**: tabla exportable con columnas configurables
- **Herramientas**: importadores (CSV/TXT, Planillero, Baterías)
- **Informe Rápido (fullReportModal)**: PDF con `fillReport()`

## Paneles inactivos
- `.rp-panel-dina.dina-inactive` → cuando `!w.hasDyn`
- `.rp-panel-nivel.nivel-inactive` → cuando `!w.hasAcu`
- Los badges (PKR, Sistema, Estado, clasificación) NO se desactivan

## Hz/RPM
Se extraen del comentario del ensayo de nivel (`w.commentAcu`) con regex:
- BES: `/(\d+(?:[.,]\d+)?)\s*hz/i`
- PCP: `/rpm\s*[:\-]?\s*(\d+(?:[.,]\d+)?)|(\d+(?:[.,]\d+)?)\s*rpm/i`

## Convenciones de código
- El usuario pide: "No me muestres código, al finalizar entrégame el html finalizado"
- Entregar con `SendUserFile` al terminar
- Commit messages en español descriptivo
- Branch actual: `claude/dina-tipo-ensayo-column-63hc7g`
- PR: https://github.com/Avanetta1078/VAPE_APP/pull/1
