# Bitácora del Operador — Especificación de Lógica

## Ubicación
Módulo dentro de **App_Operador**

## Flujo principal

### 1. Búsqueda del pozo
- Campo de búsqueda con autocompletado (por nombre de pozo)
- Al seleccionar → trae TODO de Supabase: último registro, configuración, alertas activas
- Si tiene alerta activa → semáforo visible antes de cargar nada (rojo/amarillo/verde)

### 2. Vista del pozo seleccionado
Secciones colapsables:

**Encabezado fijo:**
- Nombre pozo | Yacimiento | Batería | Operadora
- Estado actual: Activo / Cancelado (motivo) / Parado
- Tipo de sistema de extracción: BES / Bombeo Mecánico / otro
- Tipo de ensayo: Dina / Nivel / Combinada
- Estado ensayo: Realizado / Pendiente / No se realizó (motivo)

**Tarjeta Instalación (solo lectura):**
- Datos del ingeniero (viene de planilla externa Excel)
- Bomba, varillas, cañería, ancla
- El operador VE pero NO edita esta sección

**Tarjeta Contrapesos:**
- Tipo | Cantidad | CM | Secuencia
- Segundo juego (cant2, cm2)
- Editable con validación

**Tarjeta Motor:**
- Marca | RPM | Potencia HP | Polea
- Editable

**Tarjeta Observaciones:**
- Campo libre de texto

### 3. Nuevo Registro vs Edición

**Nuevo Registro ("Pum, traer todo"):**
1. Operador toca "Nuevo Registro"
2. Se pre-cargan TODOS los campos del último registro guardado (cualquier fecha)
3. Los campos vienen grisados (heredados, no editados aún)
4. Al tocar/editar un campo → cambia de gris a color activo (borde teal)
5. El operador ve de un vistazo qué tocó y qué quedó del registro anterior
6. Fecha y hora se completan automáticamente

**Edición de registro existente:**
- Selecciona fecha del registro a editar
- Se carga todo, campos en color activo (ya fueron confirmados)
- Al modificar algo → se marca como "modificado" (borde distinto, ej. amarillo)

**Borrar registro:**
- Confirmación doble ("¿Seguro? Esta acción no se puede deshacer")
- Solo borra el registro del día, no el histórico

### 4. Validación antes de guardar

Al tocar "Guardar", el sistema verifica campos pendientes:

- Contrapesos ¿cargados?
- Motor ¿cargado?
- Tipo de ensayo ¿definido?
- Estado del ensayo ¿definido?
- Observaciones ¿tiene algo? (advertencia, no bloquea)

Checklist visual:
- Verde: completo
- Amarillo: advertencia (puede guardar igual)
- Rojo: obligatorio, no deja guardar sin completar

El operador decide: Guardar completo o Guardar parcial (para volver después)

### 5. Alertas y novedades

- Lista de alertas predefinidas (del catálogo alertas_catalogo en Supabase)
- Selección rápida tipo chip/tag: "Fuga", "Varilla cortada", "Motor recalentado", etc.
- Agregar alerta personalizada (texto libre)
- Cada alerta: puede editarse, eliminarse
- Se vincula al pozo + fecha
- Visible desde VAPE_APP como semáforo

### 6. Cancelación de ensayo

Si el ensayo no se realizó:
- Operador marca "No realizado"
- Selecciona motivo: Pozo parado / Sin acceso / Equipo dañado / Otro (texto)
- Queda registrado en Supabase con estado + motivo
- VAPE_APP lo muestra pero no lo cuenta como ensayo válido

### 7. Estados visuales de campos

| Estado | Color | Significado |
|--------|-------|-------------|
| Heredado | Gris | Viene del registro anterior, no editado |
| Editado | Teal (borde) | El operador lo tocó en este registro |
| Modificado | Amarillo | Era un registro guardado, se cambió ahora |
| Obligatorio vacío | Rojo | No puede guardar sin completar |

---

## Módulo "Finalizar Día"

Dentro de App_Operador. El operador presiona al terminar la jornada.

### Flujo:
1. Detecta fecha del CSV → arma carpeta correlativa (Yacimiento/Mes/DD-MM), la crea si no existe
2. Estructura: Escritorio/Crown Point/Septiembre/24-09/
3. Pre-validación con semáforo (DYN, ACU, Planillero)
4. Tabla resumen: pozos nuevos vs existentes, inconsistencias
5. Importa a Supabase con lógica existente (parseo CSV, planillero, upsert)
6. Genera parte con correlativo automático
7. Marca día cerrado
8. VAPE_APP levanta todo automáticamente desde Supabase

### Estructura de carpetas:
```
Escritorio/
  Crown Point/
    Septiembre/
      24-09/   ← día correlativo (se crea si no existe)
        DYN*.csv
        ACU*.csv
        Planillero*.xlsx
      25-09/
      26-09/
```

---

## Paleta de colores (consistente con VAPE_APP)
- Headers/acentos: teal #2D5F6F
- Steel glow: #7ec8d9
- Supabase project: fkfrhboiugnfmrfsbvjm
- Supabase client variable: supabaseClient (NO supabase)
