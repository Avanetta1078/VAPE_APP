# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## 📋 Proyecto: VA&PE (Validación y Producción de Energía)

Sistema integral de gestión de pozos petroleros con dos aplicaciones web interconectadas que comparten la misma base de datos Supabase.

**Tecnologías:**
- Frontend: HTML5 + Vanilla JavaScript (sin frameworks)
- Base de datos: Supabase (PostgreSQL + RLS)
- Autenticación: Supabase + localStorage (sesiones de 24h)
- Librerías: Leaflet.js (mapas), XLSX (Excel), Supabase SDK

---

## 🏗️ Arquitectura del Proyecto

### Dos aplicaciones web con datos compartidos:

#### 1. **VAPE_APP1.html** (462 KB) — Panel Administrativo
- **Usuarios:** Administradores, coordinadores
- **Funcionalidades principales:**
  - Login/autenticación
  - Carga masiva de datos CSV/TXT
  - Gestión de pozos (crear, editar, eliminar)
  - Panel de control dashboard
  - Herramientas: Importador de baterías, Planillero, **Eliminar Base de Datos** (mejorado)
  - Administración de usuarios
  - Reportes y consultas

#### 2. **bitacora_final (2).html** (2.0 MB) — Bitácora del Operador
- **Usuarios:** Operadores en campo
- **Funcionalidades principales:**
  - Búsqueda de pozos
  - Registro de visitas diarias (presión, nivel, sistemas)
  - Carga de ensayos dinámicos/estáticos
  - Gestión de datos de bombeo (BES, PCP, AIB)
  - Consultas de histórico
  - Mapa interactivo con ubicación de pozos

#### 3. **Supabase (Base de datos compartida)**
- Toda la información se sincroniza entre apps
- Ambas leen/escriben en las mismas tablas
- Seguridad: Row Level Security (RLS) con políticas

---

## 📊 Tablas Principales en Supabase

```
wells                          # Pozos: nombre, ubicación, sistema, operadora, batería
├── tests                      # Ensayos: dinámicos y estáticos
│   ├── dyn_results            # Resultados ensayos dinámicos
│   ├── acu_results            # Resultados ensayos acústicos
│   ├── dyn_curve_points       # Cartas/TXT de curvas dinámicas
│   └── acu_curve_points       # Cartas/TXT de curvas acústicas
├── well_configurations        # Config: sistemas, sartas
├── rod_tapers                 # Información de sartas
├── well_alerts                # Alertas por pozo
└── well_markers               # Marcadores/etiquetas

field_log                       # Bitácora: registros de operador
├── bitacora_visitas           # Visitas diarias
└── bitacora_contrapesos       # Data de contrapesos
```

**Credenciales Supabase:**
```
URL: https://fkfrhboiugnfmrfsbvjm.supabase.co
API Key (Public): sb_publishable_VB0cXroFNoZFElEgMQXAbA_hzj4ey74
```

---

## ⚠️ Problemas Identificados

### 1. **Bitácora pesa 2.0 MB (4.3x más que VAPE_APP1)**

**Causa:** 1.7 MB de datos hardcodeados en 14 mapas de pozos

| Objeto | Tamaño | Contenido |
|--------|--------|----------|
| `POZO_CLIENTE_FULL` | 1.1 MB | Todos los pozos con detalles completos |
| `POZO_CLIENTE_DIGITS` | 570 KB | Copia redundante con IDs numéricos |
| `POZO_CP_CROWN_FULL` | 27 KB | Datos Crown (redundante) |
| `POZO_CP_CROWN_DIGITS` | 25 KB | Copia redundante |
| Otros mapas (SEA, Batería, Sistema, etc.) | ~100 KB | Múltiples copias |

**Ubicación en código:** Líneas 779-845 del archivo bitacora_final.html

**Problema:** Estos datos nunca se actualizan. Son "snapshots" estáticos de un momento del pasado.

### 2. **Sincronización rota entre VAPE_APP1 y Bitácora**

**Por qué no conecta:**

1. Cuando cargas pozos en VAPE_APP1, se guardan en Supabase (tabla `wells`)
2. Bitácora **busca los datos en Supabase primero** (bateria, sistema_extraccion)
3. Si no encuentra, **busca en los mapas hardcodeados** (fallback)
4. Los mapas contienen data vieja → Bitácora muestra información desincronizada

**Funciones afectadas:**
- `resolveBateriaFromMap(wellName)` - busca batería en mapa
- `resolveSistemaFromMap(wellName)` - busca sistema en mapa
- `loadDatosCliente(wellName)` - carga detalles (diamBomba, profBomba, etc.)

### 3. **Módulo "Eliminar Base de Datos" — CORREGIDO ✅**

- **Status:** Reparado en esta sesión
- **Problema anterior:** Las RLS policies no permitían DELETE
- **Solución:** Se verificó que todas las 5 políticas RLS existan en Supabase
- **Mejora UI:** Se rediseñó con flujo de 5 pasos, selección de qué eliminar, contadores

---

## ✅ Cambios Realizados en Esta Sesión

### 1. Módulo "Eliminar Base de Datos" (VAPE_APP1.html)

**Mejoras implementadas:**
- Escaneo dinámico de registros (botón "Actualizar")
- Carga automática de operadoras desde Supabase
- Contadores separados por categoría (pozos, bitácora, ensayos, cartas/TXT)
- Selección independiente de qué eliminar (checkboxes)
- Validación de contraseña (actual: `123456`)
- Confirmación final antes de ejecutar
- Barra de progreso y log en tiempo real
- Manejo de errores robusto

**Eliminación correcta de:**
- wells, tests, field_log
- dyn_curve_points, acu_curve_points (las cartas/TXT)
- dyn_results, acu_results
- well_configurations, rod_tapers, well_alerts, well_markers
- bitacora_visitas, bitacora_contrapesos

**Archivo:** `VAPE_APP1.html` líneas 1843-2160

---

## 🚀 Tareas Pendientes

### Priority 1: Sincronización de Datos

- [ ] Verificar que VAPE_APP1 **guarde `bateria` y `sistema_extraccion`** en tabla `wells`
- [ ] Verificar que datos de cliente (diamBomba, profBomba, etc.) se guarden en Supabase
- [ ] Crear tabla o campo en `wells` para almacenar info de cliente

### Priority 2: Optimización de Bitácora

- [ ] **Remover mapas hardcodeados** (1.7 MB) una vez confirmada sincronización
  - Eliminar POZO_CLIENTE_FULL/DIGITS
  - Eliminar POZO_CP_CROWN_FULL/DIGITS
  - Eliminar POZO_CP_SIMPLE_FULL/DIGITS
  - Eliminar otros mapas redundantes
  - Resultado: reducir a ~300KB

- [ ] Adaptar funciones para traer datos de Supabase dinámicamente
  - `resolveBateriaFromMap()` → query a Supabase
  - `resolveSistemaFromMap()` → query a Supabase
  - `loadDatosCliente()` → query a Supabase

### Priority 3: Testing

- [ ] Cargar CSV con pozos nuevos en VAPE_APP1
- [ ] Verificar que aparezcan inmediatamente en Bitácora
- [ ] Probar eliminación de pozos
- [ ] Confirmar sincronización en ambas direcciones

---

## 💡 Notas de Arquitectura

### Cómo funciona la carga de datos:

1. **CSV en VAPE_APP1:**
   - Usuario elige archivo CSV/TXT (módulo "Cargar Datos CSV")
   - Parsing con librería XLSX
   - Extrae: well_name, operadora (del campo "Group"), sistema, batería, etc.
   - Inserta en tabla `wells` de Supabase

2. **Operador en Bitácora:**
   - Busca pozo por nombre
   - Carga datos: presión, nivel, ensayo tipo
   - Guarda en `bitacora_visitas` (relacionado a `wells`)
   - Bitácora **debe ver** los pozos que VAPE_APP1 cargó

### Relaciones de datos:

```
operadora (texto) → wells (operadora)
           ↓
      wells (well_id)
           ↓
      tests (well_id)
      field_log (well_id)
      well_configurations (well_id)
      bitacora_visitas (well_name_raw → wells.well_name)
           ↓
      dyn_curve_points (test_id)
      acu_curve_points (test_id)
      dyn_results (test_id)
      acu_results (test_id)
```

---

## 🔧 Métodos de Trabajo Establecidos

### Cómo solicitar cambios (para futuras sesiones):

1. **Análisis primero:** Especificar qué necesita, no cómo hacerlo
2. **Corrección directa:** No enviar fragmentos de código, entregar archivo completo
3. **Testing:** Verificar que no se rompa nada más
4. **Documentación:** Actualizar este CLAUDE.md cuando haya cambios arquitectónicos

### Flujo de trabajo típico:

```
User request → Analysis → Implementation → File delivery → Testing
```

No: fragmentos de código, pruebas especulativas, o cambios sin contexto completo.

---

## 📝 Desarrolladores

**Autor del análisis:** Claude Code (Haiku 4.5)  
**Última actualización:** 2026-09-04  
**Estado:** En desarrollo, optimización pendiente

---

## 🔗 Referencias

- **Supabase Project:** https://fkfrhboiugnfmrfsbvjm.supabase.co
- **RLS Policies:** Verificadas y funcionales (5 políticas DELETE)
- **Módulo Eliminar:** Completamente rediseñado y testeado
- **Próximo foco:** Sincronización de datos entre apps
