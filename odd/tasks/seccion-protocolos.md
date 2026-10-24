# Feature: Sección "Protocolos Existentes" en la landing

## Objetivo
Replicar en la landing pública (`ava-pt.org`) la misma mecánica que eventos/actividades: una sección nueva
"Protocolos Existentes" que carga sus datos desde la tabla `protocolos` de InsForge, con tarjetas resumidas.

## Problema / Por qué
La AVA publica protocolos de colaboración y acuerdos de servicios. Hoy solo existen en la plataforma
administrativa; la landing no los muestra.

## Restricciones (decisión del usuario, 2026-10-24)
- Tarjeta resumida, **sin publicar el PDF**: el PDF es información interna.
- Por lo tanto el front NO debe pedir ni exponer `pdf_key`, `pdf_url`, `pdf_nome_original`, `pdf_tamanho`,
  `notas_internas`, `image_key`, `image_url`: se usa `select=` explícito con solo campos públicos.
- Mismo look & feel que la sección de Actividades Periódicas; sección ubicada justo después de
  Actividades Periódicas y antes de "¿Quiénes Somos?".
- i18n ES/PT igual que el resto (los campos de contenido son de una sola variante, en portugués, y se
  muestran tal cual).

## Fuente de datos
`GET {INSFORGE_URL}/api/database/records/protocolos?select=<campos públicos>&order=ordem&limit=200`
Campos públicos: `id,titulo,tipo,areas,resumo,descricao,data_firma,local_firma,tipo_vigencia,vigencia_desde,vigencia_hasta,estado,projeto_vinculado,financiadores,visivel_landing,destaque,ordem`
Filtro de visibilidad: solo `visivel_landing === true`. Sin fallback embebido (si falla → estado vacío).

Datos actuales (2 registros, ambos `visivel_landing=true`, estado `ativo`):
- `tipo`: `colaboracao` (1), `servicos` (1) → filtros por tipo.
- `tipo_vigencia`: `ligada-projeto` (1), `indeterminada` (1).
- Ambos con `destaque` desigual (true/false) → badge de destaque opcional.

## Checklist
- [x] T1: Sección HTML + link de nav en `index.html` (grid, filtros por tipo, estado vacío)
- [x] T2: Loader + render + filtros + i18n ES/PT en `script.js`
- [x] T3: Estilos `.protocols` / `.protocol-card` en `styles.css`, consistente con `.activities`
- [x] T4: Verificación + commit `c95e9e5` en rama `feat/seccion-protocolos`

## Criterios de aceptación
- La sección aparece entre Actividades Periódicas y "¿Quiénes Somos?" y se vea en ES y PT.
- Se renderizan los 2 protocolos visibles, filtrables por tipo.
- Ninguna petición ni el DOM contienen campos de PDF ni notas internas.

## Progreso
- 2026-10-24: feature creada; ruta elegida: delegated direct (writer único, 3 archivos no triviales).
  El intento de delegación devolvió `Free usage limit reached` en la entrega del resultado, pero el writer
  sí ejecutó y dejó los 3 archivos escritos; el padre hizo la verificación y el commit (degradación inline).
- 2026-10-24: T1–T4 completados. Commit `c95e9e5` (417 líneas: index.html 25, script.js 213, styles.css 179).

## Verificación (observada)
- `node --check script.js` → OK.
- `python3 -m http.server 8123` + `curl` → sección presente (5 coincidencias `protocolos` en HTML),
  `renderProtocolos` presente en el JS servido.
- Petición real con las columnas exactas del código → HTTP 200, 2 filas, `visivel_landing=true` en ambas,
  exactamente los 17 campos públicos, `grep /pdf_|notas_internas/` sobre la respuesta → sin coincidencias.
- Referencias a `pdf_*`/`notas_internas` en `script.js` → solo dentro del comentario de advertencia (líneas 890-891).
- RDD: `off` (global) → sin revisión nativa; checks ordinarios nominales.
- Pendiente: revisión visual humana en ES/PT y decisión sobre push/PR.

## Hallazgo de seguridad (no abordado por esta feature)
La RLS de la tabla `protocolos` permite al `anon` leer `pdf_url`, `pdf_key`, `pdf_nome_original`,
`pdf_tamanho` y `notas_internas` (confirmado con un GET sin `select`). El front no los pide, pero
cualquier persona puede. Recomendado: restringir esos campos en RLS/admin.
