# CLAUDE.md — VA&PE App

## Regla #1 de trabajo
**Ejecutar lo que el usuario pide. No suponer, no contradecir, no desviarme.** Si el usuario indica dónde está un dato o cómo resolver algo, hacer exactamente eso primero. Verificar después. Nunca experimentar ni probar soluciones propias antes de hacer lo que el usuario pidió.

## Regla #2 — Entrega
- El usuario pide: "No me muestres código, al finalizar entrégame el html finalizado"
- Entregar con `SendUserFile` al terminar cada tarea
- Commit messages en español descriptivo

## Proyecto
Aplicación monolítica de monitoreo de pozos petroleros. Un solo archivo HTML (`VAPE_APP1.html`, ~9000+ líneas) con todo el CSS, HTML y JS inline.

## Stack
- **Frontend**: HTML/CSS/JS vanilla (sin frameworks)
- **Backend**: Supabase (project ID: `fkfrhboiugnfmrfsbvjm`)
- **Repo**: `Avanetta1078/VAPE_APP` en GitHub

---

## Tablas principales en Supabase

| Tabla | Propósito |
|---|---|
| `wells` | Pozos (datos maestros + motor/contrapesos). Columnas de motor: `motor_marca`, `motor_rpm`, `motor_hp`, `motor_polea`, `contrapesos` |
| `well_configs` | Configuraciones de cada pozo |
| `tests` | Ensayos (DYN/ACU) con fecha, hora, tipo. Columna `test_type` = 'DYN' o 'ACU' |
| `dyn_results` | Resultados del ensayo dinamométrico. Campo clave: `dyn_tubing_pressure_kg` |
| `acu_results` | Resultados del ensayo acústico/nivel |
| `dyn_points` | Puntos de la carta dinamométrica (surface/pump card) |
| `acu_points` | Puntos del sonolog |
| `field_log` | Importaciones históricas del planillero |
| `bitacora_visitas` | Bitácora Online (app separada, **NO TOCAR**) |
| `bitacora_contrapesos` | Bitácora Online contrapesos (**NO TOCAR**) |

---

## Mapeo CSV del Echometer/TAM

### IMPORTANTE: Columnas con prefijo DYN vs ACU
El CSV tiene columnas **duplicadas** con prefijos distintos. La columna ACU aparece PRIMERO en el CSV:
- `ACU Tubing Pressure (kilo (g))` → presión acústica (NO usar para dyn_results)
- `DYN Tubing Pressure (kilo (g))` → presión dinámica (**CORRECTA para dyn_results**)
- `ACU Casing Pressure (kilo (g))` vs `DYN Casing Pressure (kilo (g))` → mismo caso

**NUNCA usar un fallback genérico que agarre la primera columna "tubing pressure" — siempre buscar con el prefijo DYN para campos de dyn_results. Si se usa un fallback genérico, agarra la ACU (que vale 0) en vez de la DYN (que tiene el dato real).**

### DYN_FIELD_MAP (campos clave)
```
dyn_tubing_pressure_kg → 'DYN Tubing Pressure (kilo (g))'
dyn_casing_pressure_kg → 'DYN Casing Pressure (kilo (g))'
pip_dyn_kg             → 'PIP DYN (kilo (g))'
pump_discharge_pressure_kg → 'Pump Discharge Pressure (kilo (g))'
adjusted_fillage_pct   → 'Adjusted Fillage (%)'
```

### Parseo de headers CSV
- Los headers pueden venir con comillas (`"header"`), se limpian con `.replace(/^"|"$/g, '')`
- El delimitador puede ser `;` (Echometer) o `,` (TAM)
- `findCsvColumnValue()` busca: exacto → case-insensitive → última palabra como substring
- `parseCsvRows()` limpia comillas de headers y valores

### Importación CSV — Comportamiento del upsert
- Ruta con TXT: siempre hace upsert de `dyn_results` (test nuevo o existente)
- Ruta CSV-only (sin TXT): también hace upsert siempre (corregido — antes solo guardaba en tests nuevos)
- `ensureTest()` devuelve `{ testId, isNew }` — `isNew` solo se usa para el contador de resumen

---

## Caché en memoria (WELLS_DB)
`WELLS_DB[wellName]` se puebla en `loadLiveDataFromSupabase()`. Campos relevantes:
- `tubpres` ← `dyn_tubing_pressure_kg` de `dyn_results`
- `pipdyn` ← `pip_dyn_kg`
- `fillage` ← `adjusted_fillage_pct`
- `motorMarca`, `motorRpm`, `motorHp`, `motorPolea` ← columnas de `wells`
- `contrapesosRaw` ← columna `contrapesos` de `wells`
- `hasAcu`, `hasDyn` ← si tiene ensayos acústicos/dinámicos
- `sistema` ← tipo de extracción (AIB/BES/PCP/SSE)
- `commentAcu`, `commentDyn` ← comentarios de los ensayos
- `wellState` ← estado del pozo (Producing/Static)

### CONTRAPESOS_CACHE y CONTRAPESOS_DETALLE_CACHE
- Se pueblan desde `wells` (columnas `motor_marca`, `motor_rpm`, `motor_hp`, `motor_polea`, `contrapesos`)
- `loadContrapesosForWell()` lee de WELLS_DB, no de field_log
- Motor y contrapesos se actualizan en `wells` durante la importación del planillero

---

## Matching tolerante de pozos (planillero)
4 niveles: exacto → lowercase → strip dashes/spaces → solo dígitos.
- `normLower(s)` → trim + lowercase
- `normStrip(s)` → normLower + quitar `[-\s().]`
- `normDigits(s)` → solo dígitos
- `matchWell(rawName)` → prueba los 4 niveles en orden
- `byDigits[d] = null` si es ambiguo (2+ pozos con mismo número)

Ejemplo: S-1000, s1000, s-1000, 1000 → todos matchean al mismo pozo.

---

## Vistas principales

### Inicio
Grilla de pozos con filtros (operadora, batería, zona, búsqueda)

### Menú DINA (Resumen Pozo) — 2 paneles lado a lado
- **Panel Dina (izquierda)**: carta dinamométrica + chips (Carga Máx/Mín, GPM, Llenado, P.Tbg, Fabricante, Modelo, Carrera, PIP Dyn)
- **Panel Nivel (derecha)**: wellSvg + buildup + tramos + chips (Nivel, Sumerg, Niv.corr, Sum.corr, PIP Acu, Prof.bba)
- Badges: PKR (posición absoluta, animación pulse), Sistema (AIB/BES/PCP/SSE), Estado (Dinámico/Estático/Packer), Clasificación (Combinada/Dina/Nivel)

### Menú Nivel (Sonolog)
Panel acústico dedicado con gráfico interactivo de sonolog

### Consultas
Tabla exportable con columnas configurables (CN_EXTRA_DEFS). Incluye contrapesos (6 columnas: cpTipoPpal, cpCantPpal, cpDistPpal, cpTipoSecu, cpCantSecu, cpDistSecu)

### Herramientas
- **Cargar Datos (CSV/TXT)**: importador Echometer/TAM con botones Ver vista previa, Guardar, Limpiar
- **Planillero**: importador de Excel del planillero diario con matching tolerante
- **Baterías**: importador de asignación de baterías

### Informe Rápido (fullReportModal)
PDF via print con `fillReport()`. Dos páginas:
- **Pág. 1 (Dina)**: dynChips, carta, comentario, detalle técnico, bomba, producción, unidad de bombeo, reductor, sarta, contrapesos
- **Pág. 2 (Nivel)**: acuChips (incluye Hz/RPM si BES/PCP + Niv.corr.), comentario, wellSvg, buildup (380x230), tramos
- Header: nombre pozo + sistema (BES/PCP/AIB), batería, operadora, fecha, clasificación

---

## Paneles inactivos (estado visual)
- `.rp-panel-dina.dina-inactive` → cuando `!w.hasDyn` (opacity .35, grayscale, fondo gris)
- `.rp-panel-nivel.nivel-inactive` → cuando `!w.hasAcu` (mismo estilo)
- Los badges (PKR, Sistema, Estado, clasificación) **NO se desactivan** — mantienen su color

---

## Hz/RPM
Se extraen del comentario del ensayo de nivel (`w.commentAcu`) con regex:
- BES: `/(\d+(?:[.,]\d+)?)\s*hz/i`
- PCP: `/rpm\s*[:\-]?\s*(\d+(?:[.,]\d+)?)|(\d+(?:[.,]\d+)?)\s*rpm/i`
- Se muestran en Resumen Pozo (chip rp-hzrpm) y en Informe Rápido (acuChips)

---

## Nivel Corregido
- Cálculo: `pumpIntakeDepth - sumergc`
- Se muestra en Resumen Pozo (chip rp-nivcorr) con alerta si difiere de nivel > 1m
- Se muestra en Informe Rápido entre Sumerg y Sum.corr

---

## Contrapesos
- Datos en `wells.contrapesos` (JSON con tipo/cant/secu para cada manivela)
- Motor en `wells.motor_marca/rpm/hp/polea`
- Se importan desde el planillero y se guardan en `wells` (UPDATE)
- UI: cards unificadas (Manivela 1 y 2 en un solo contenedor), color unificado
- 6 columnas en Consultas (CN_EXTRA_DEFS)

---

## Errores frecuentes y lecciones aprendidas
1. **CSV con columnas duplicadas DYN/ACU**: siempre usar el prefijo DYN para dyn_results. Un fallback genérico agarra la ACU (que aparece primero y vale 0).
2. **Tests existentes**: el upsert debe ejecutarse siempre, no solo para tests nuevos (`isNew`). Si se condiciona a `isNew`, re-importar un CSV no actualiza los campos que faltaban.
3. **Headers con comillas**: parseCsvRows debe limpiar `"` de headers y valores.
4. **well_number NULL**: el matching de planillero usa `well_name` con tolerancia de 4 niveles, no `well_number`.
5. **Motor/contrapesos**: se leen de la tabla `wells` directamente, no de `field_log`. Se eliminó la consulta separada a `field_log` en `loadLiveDataFromSupabase()`.
