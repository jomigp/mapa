# Plan del Proyecto — Dashboard Clínico MAPA

Documento de referencia para la evolución del proyecto. Estado a fecha de creación.

---

## 1. Visión

Convertir el informe MAPA actual (un archivo `index.html` estático con datos
embebidos) en una **aplicación web** donde el médico pueda **cargar los datos
crudos de cualquier monitor** de presión arterial ambulatoria (MAPA / ABPM) y
obtener el mismo dashboard clínico de forma automática, listo para imprimir o
exportar a PDF.

Objetivo clínico: análisis según **ESH/ESC 2023** (promedios día/noche, carga
hipertensiva, patrón dipping, distribución de severidad y perfil hemodinámico).

---

## 2. Estado actual (hecho ✅)

- Dashboard en **un solo archivo** `index.html` (Tailwind + Chart.js vía CDN).
- Procesamiento en el navegador con **filtro blindado** (ignora cabeceras,
  filas mal formadas y lecturas fallidas `---/---`).
- Cálculos: promedios global/día/noche (PAS, PAD, FC), carga hipertensiva
  (día >135/85, noche >120/70), **patrón dipping sistólico** (Extreme / Normal /
  Non-Dipper / Riser) y distribución sistólica (Normal / Pre-HTA / Grado 1 / 2).
- Gráficos: evolución 24h, hemodinámica (PAM y PP), dona de severidad y
  dispersión PAS vs PAD.
- **Corrección de reloj (+12 h)** por AM/PM invertido del equipo; se muestran
  las horas reales y la clasificación día/noche usa la hora real.
- Cabecera con datos del paciente editables y bloque de conclusiones + firma.
- Modo impresión A4 (`@media print`) y botón "Exportar a PDF".
- Diseño **mobile-first**.

**Limitación:** los datos están embebidos en el HTML. Para cada paciente hay
que editar el archivo a mano.

---

## 3. Meta: carga de archivos

Permitir subir un archivo del monitor y procesarlo en tiempo real. Formatos
objetivo:

| Formato | Origen típico | Dificultad | Notas |
|---|---|---|---|
| `.csv` | Exportación estándar de muchos monitores | Baja | Parser configurable (auto-detección de separador y columnas). |
| `.txt` | Volcados de texto | Baja–Media | Suele ser CSV con otro separador o ancho fijo. |
| `.awb` | **Contec ABPM50** | Alta | Formato **binario propietario**; requiere ingeniería inversa de su estructura. Se necesita un `.awb` de ejemplo para mapear los campos. |

---

## 4. Arquitectura propuesta

```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌───────────────┐
│  Subir      │ →  │  Parser por      │ →  │  Modelo común   │ →  │  Dashboard    │
│  archivo    │    │  formato         │    │  de lecturas    │    │  (render)     │
│ (txt/csv/awb)│   │ (csv|txt|awb)    │    │  Reading[]      │    │  = actual     │
└─────────────┘    └──────────────────┘    └─────────────────┘    └───────────────┘
```

- **Modelo común `Reading`**: `{ fecha, hora, sys, dia, hr, pam, pp, period }`.
  Todo el motor de cálculo y los gráficos ya trabajan sobre este modelo, así que
  solo cambia la *fuente* de datos.
- **Capa de parsers**: una función por formato que devuelve `Reading[]`. Reutiliza
  el filtro blindado actual.
- **Ajustes por estudio** (panel de opciones antes de calcular):
  - Ventana día/noche configurable (por defecto 07:00–21:59 / 22:00–06:59).
  - **Corrección de reloj** (offset en horas) — hoy fijo en +12 h; hacerlo un
    campo editable con detección/aviso automático.
  - Umbrales de carga hipertensiva editables.

---

## 5. Fases

### Fase 1 — Ingesta CSV/TXT (MVP)
- Botón "Cargar archivo" + arrastrar y soltar.
- Parser CSV/TXT con auto-detección de separador y mapeo de columnas
  (con vista previa para confirmar qué columna es PA, hora, fecha, FC).
- Sustituir los datos embebidos por los del archivo cargado.
- Datos del paciente editables (ya existe) + persistencia opcional en el
  navegador (localStorage) mientras dura la sesión.

### Fase 2 — Ajustes clínicos configurables
- Panel para ventana día/noche, offset de reloj y umbrales.
- Detección automática del desfase de reloj (avisar si el inicio "no cuadra"
  con la hora esperada) en lugar del +12 h fijo.

### Fase 3 — Formato `.awb` (Contec ABPM50)
- Conseguir archivos `.awb` de ejemplo (idealmente con su PDF/CSV equivalente
  para validar).
- Ingeniería inversa de la estructura binaria (offsets de fecha/hora, PA, FC).
- Parser `.awb` → `Reading[]` + pruebas contra los ejemplos.

### Fase 4 — Producto
- Multi-estudio / histórico por paciente.
- Exportación PDF server-side de mayor fidelidad (opcional).
- Base para desplegar en Vercel (hoy ya es estático y desplegable tal cual).

---

## 6. Consideraciones

- **Privacidad (PHI):** el informe contiene datos identificables del paciente.
  Todo el procesamiento es **en el navegador** (no se suben datos a servidores),
  lo cual es una ventaja. Si se despliega en una URL pública, esa URL es
  accesible por cualquiera con el enlace: compartir con criterio.
- **Validación clínica:** cualquier cambio en fórmulas (dipping, cargas,
  clasificaciones) debe verificarse contra un caso conocido antes de usarse.
- **Despliegue:** al ser estático, Vercel lo sirve sin configuración (framework
  "Other", sin build). Cada push puede redeployar automáticamente.

---

## 7. Próximo paso inmediato

Arrancar **Fase 1** (ingesta CSV/TXT) y, para la Fase 3, **conseguir un `.awb`
de ejemplo del ABPM50** para analizar su formato.
