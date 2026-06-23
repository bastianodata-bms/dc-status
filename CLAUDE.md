# CLAUDE.md — Dashboard de Infraestructura ODATA (ST02)

Este archivo es contexto de proyecto para Claude Code. Complementa al código —no lo reemplaza—: los archivos reales (`dashboard_online.html`, `mock_server.py`) son la fuente de verdad. Este MD explica el porqué de las decisiones, lo que ya está confirmado, y lo que falta.

**Diferencia importante vs. sesiones anteriores en claude.ai:** ahí Claude solo podía dar instrucciones de texto (diffs) porque no tenía acceso al sistema de archivos del usuario. En Claude Code sí tienes acceso directo — puedes leer y editar estos archivos sin pedirle al usuario que pegue/aplique nada manualmente.

---

## 1. Qué es esto

Dashboard HTML de una sola página, tipo semáforo, para monitorear la infraestructura del data center **ST02** (Santiago, Chile, operado por ODATA). Pensado para:
- Reemplazar/complementar Power BI con algo más visual
- Eventualmente compartirse con Google (cliente)

**8 sistemas:** Chiller, Cooling, UPS, Boards & Power, Genset, MV switchgear, Fire Extinguisher, Network
**9 ubicaciones del grid (placeholder, no confirmadas con datos reales aún):** SH1–SH6, MNR-A, MNR-B, SITE
**Semáforo:** 3=Óptimo, 2=Observación, 1=Crítico

---

## 2. Archivos del proyecto

| Archivo | Rol |
|---|---|
| `dashboard_online.html` | Frontend completo (HTML+CSS+JS en un solo archivo), hosteado en GitHub Pages |
| `mock_server.py` | Backend de prueba en Python stdlib puro — simula lo que hará la futura Azure Function |
| `Requisitos_Query_Produccion_IT.md` | Especificación de la query SQL real, para entregar a IT |
| `Estado_Proyecto_Dashboard_ODATA.md` | Resumen de estado general del proyecto (este archivo es más técnico/operativo para Claude Code) |

Repo: `https://github.com/bastianodata-bms/dc-status` → publicado en `https://bastianodata-bms.github.io/dc-status/`

---

## 3. Cómo correr el entorno local

```bash
python3 mock_server.py
# Sirve en http://localhost:8000
#   /api/equipos       -> CSV
#   /api/equipos.json  -> JSON
```

El HTML lee de `SHEETS_CSV_URL` (nombre legado de cuando la fuente era Google Sheets — ver sección 8). Hoy apunta a `http://localhost:8000/api/equipos`.

**Antes de entregar cualquier cambio al HTML, validar sintaxis JS:**
```bash
python3 -c "
import re
html = open('dashboard_online.html').read()
m = re.search(r'<script>(.*)</script>', html, re.S)
open('/tmp/extracted.js','w').write(m.group(1))
"
node --check /tmp/extracted.js
```
Esto ya salvó de al menos dos bugs reales (ver sección 9).

---

## 4. Arquitectura de datos

```
Data Lake SQL (ODATA) → Azure Function (API) → dashboard_online.html → navegador
```

El navegador no puede hablar SQL directo — siempre se necesita un traductor HTTP intermedio. Hoy ese traductor es `mock_server.py` (local); en producción será una Azure Function que IT debe construir. **El HTML no necesita cambios al pasar a producción** — solo se reemplaza la URL en `SHEETS_CSV_URL`. El parser ya lee columnas por nombre de header, no por posición, así que es agnóstico a la fuente.

---

## 5. Mapeo SQL confirmado (con datos reales de ST02)

### SISTEMA — de `alm_asset.asset_tag` (prefijo), NO de `alm_asset.model` (eso da el fabricante)

| Sistema | Prefijos reales confirmados |
|---|---|
| Chiller | `CH`, `VFD` |
| Cooling | `CRAH`, `AHU`, `PCWP`, `FANWALL`, `UC`, `UE` |
| UPS | `UPS A/B`, `Battery Bank`, `BDC A/B`, `UBD`, `RTF A` |
| Genset | `Gen A/B`, `ATS`, `MTS A/B` |
| MV switchgear | `TX A/B`, `RMU A/B`, `MV` |
| Boards & Power | `TDF`, `TDA`, `TDAF`, `T.D.A.F`, `PMDC A/B`, `DB`, `DP`, `PP A/B`, `Tablero N°`, `QC`, `ACP X.X` |
| Fire Extinguisher | `ASP`, `Fire Pump INC`, `LOOP 1-8` |
| Network | `Sistema Red ODATA`, `SSFR IDF`, `QA` |

Excluidos del semáforo (no son equipo): `ML` (Manlift, móvil), tags de ubicación (`ST02`, `MNR`, `Server Hall...`, `CAG...`), herramientas/EPP/rondas.

### SITIO — de `cmn_location.full_name`, NO de `sys_user_group`/`assignment_group`

Formato de ruta: `Región/Ciudad/Ciudad/SITIO/Piso/Sala` (ej. `LatAm/Santiago/Santiago/ST02/1st Floor/...`). Filtro: `full_name LIKE '%/ST02/%'`.

`assignment_group` (formato `ODC-ST02-OPS-Team`) **no sirve como fuente principal** — solo existe en tablas de tickets, así que un equipo sano sin incidente nunca lo tiene. `cmn_location.full_name` cubre el 100% de los equipos.

### Pendiente — UBICACION (nombre de sala)

Las 9 celdas del grid (`SH1-SH6, MNR-A, MNR-B, SITE`) son placeholder. Los nombres reales en `cmn_location` son distintos (ej. "Material Trap 1", "DCA3-210-01"). Falta el mismo ejercicio de mapeo que se hizo para SISTEMA, pero para sala → ubicación del grid. **Este es el próximo trabajo pendiente más importante.**

---

## 6. Query SQL de producción (referencia rápida)

Ver `Requisitos_Query_Produccion_IT.md` para el detalle completo. Lo crítico:

```sql
LEFT JOIN incident i
    ON i.cmdb_ci_sys_id = a.sys_id
    AND i.active = 1          -- ⚠ filtrar AQUÍ, no después en un CASE
```

**Por qué importa:** los incidentes cerrados no se borran, se acumulan para siempre. Filtrar tarde hace que el motor procese todo el histórico en cada refresh. Es el único punto de eficiencia que de verdad importa a escala — el resto (índice en `full_name`, separar endpoint resumen/detalle) es secundario.

---

## 7. Convenciones del código

- **Idioma:** todo el UI, comentarios y nombres de columna están en español. Mantener consistencia (`SISTEMA`, `UBICACION`, `ESTADO`, no traducir a inglés).
- **Colores (CSS vars en `:root`):** `--purple` (marca ODATA), `--s3`/`--s2`/`--s1` (verde/amarillo/rojo del semáforo). Nunca hardcodear hex de estado — usar las variables.
- **Columnas CSV/JSON esperadas:** `SISTEMA, UBICACION, EQUIPO, ESTADO, PRIORIDAD, SEVERIDAD, ESTADO_TICKET, MOTIVO_ESPERA, OBSERVACION, DESCRIPCION, FECHA_CREACION, FECHA_ACTUALIZACION_SN, SITIO, ULTIMA_ACTUALIZACION`. El parser detecta por nombre de header (`findCol`), tolera alias — ver constantes `iSys`, `iLoc`, etc. en el `<script>`.
- **Patrón de filtro/estado global:** `ALL_DATA` (todo lo cargado) vs `DATA` (filtrado visible). Cualquier función de render debe leer de `DATA`, nunca de `ALL_DATA` directamente.
- **Diseño:** animaciones discretas/profesionales (fade-up, pulso suave) — este es un dashboard operativo cliente-facing, evitar algo "juguetón".

---

## 8. Funcionalidades ya implementadas

- Login con contraseña (SHA-256, `PASS_HASH`)
- Fetch automático + auto-refresh configurable (`AUTO_REFRESH_MIN`) + botón manual
- **Botón de pausa/reanudar** — congela actualizaciones (útil al presentar en vivo)
- Filtro de sitios (chips, multi-selección), visible en Home y detalle — solo aparece con 2+ sitios en los datos
- Badges de Prioridad/Severidad, estado de Ticket (con motivo de espera), descripción larga desplegable
- Indicadores de "reciente" (ventana 15 min): punto pulsante + reordenamiento al tope de la tabla
- Signo de exclamación animado para Crítico + Severidad Alta
- Fecha de creación del incidente visible, con tooltip de última actualización en ServiceNow (`sys_updated_on`)
- Animaciones de entrada (fade-up escalonado)

`mock_server.py` (v7) simula todo esto: prefijos reales por sistema, 3 sitios de prueba (`ST02` real + `ST05`/`ST08` ficticios para el filtro), mutación de estado en vivo cada ~20s (resuelve/crea incidentes), campos de fecha realistas.

---

## 9. Errores ya cometidos — no repetir

Estos bugs reales rompieron el script completo (ediciones manuales por find-replace, antes de tener Claude Code con acceso directo al archivo):

1. **Variable de configuración corrompida:** una edición pegó el texto completo de la declaración dentro del string de comparación (`SHEETS_CSV_URL === 'const SHEETS_CSV_URL = ...'`). Causa: copiar/pegar impreciso. Mitigación: con Claude Code, usar reemplazo de texto exacto y validar con `node --check` después de cada edit.
2. **Declaración de función borrada accidentalmente:** un find-replace que debía insertar código *antes* de una función terminó borrando la línea `function locState(sys,loc) {` y dejando el cuerpo huérfano. Mismo mitigante: validar sintaxis después de cada cambio, no asumir que un edit fue limpio.

**Regla general:** después de cualquier edición al `<script>` del HTML, correr el chequeo de sintaxis de la sección 3 antes de considerar el cambio terminado.

---

## 10. Historial de decisiones descartadas (para no proponerlas de nuevo sin razón)

- **Google Sheets como fuente de datos:** se probó (fetch a CSV publicado), funcionó técnicamente, pero se descartó porque la fuente de verdad real es el Data Lake SQL de ODATA, no una planilla mantenida a mano. El nombre de variable `SHEETS_CSV_URL` quedó como residuo histórico — no implica que hoy se use Sheets.
- **Edición en vivo desde el HTML hacia Sheets (Google Apps Script):** se diseñó la arquitectura (`Editor HTML → Apps Script → Sheet`) pero no se implementó, por el cambio de rumbo hacia SQL/Azure.
- **Power Automate / Logic Apps como intermediario no-code:** evaluado como alternativa a Azure Function, descartado por costo de licencia del conector SQL Server premium.

---

## 11. Próximos pasos (en orden de prioridad)

1. **Mapear nombres reales de sala → las 9 ubicaciones del grid** (mismo método que SISTEMA: pedir `SELECT DISTINCT name FROM cmn_location WHERE full_name LIKE '%/ST02/%'`, agrupar)
2. Compartir `Requisitos_Query_Produccion_IT.md` con IT antes de que construyan la Azure Function
3. Cuando exista el endpoint real, cambiar solo `SHEETS_CSV_URL` — no se anticipan más cambios de HTML
4. Decidir si se retoma el plan de mockup compartido con contraseña (Netlify) para Google, o se espera a tener datos reales conectados
