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
- [x] T5: Botón "Más información" por tarjeta + modal de detalle con descripción, servicios,
      beneficios y financiadores (commit de la rama feature, ver más abajo)
- [ ] T6: Mostrar la entidad contraparte ("con quién firmamos") — BLOQUEADO: la tabla `protocolos`
      no tiene ese campo y `protocolo_partes` es solo `authenticated` por RGPD. Pendiente de decisión
      del usuario entre: vista pública no personal de `protocolo_partes` + join con `entidades`
      (pública), columna denormalizada en `protocolos`, o no mostrarla.

## Criterios de aceptación
- La sección aparece entre Actividades Periódicas y "¿Quiénes Somos?" y se vea en ES y PT.
- Se renderizan los 2 protocolos visibles, filtrables por tipo.
- Ninguna petición ni el DOM contienen campos de PDF ni notas internas.

## Progreso
- 2026-10-24: feature creada; ruta elegida: delegated direct (writer único, 3 archivos no triviales).
  El intento de delegación devolvió `Free usage limit reached` en la entrega del resultado, pero el writer
  sí ejecutó y dejó los 3 archivos escritos; el padre hizo la verificación y el commit (degradación inline).
- 2026-10-24: T1–T4 completados. Commit `c95e9e5` (417 líneas: index.html 25, script.js 213, styles.css 179).
- 2026-10-24: merge fast-forward a `master` + push (cuenta `ava-pt`, activa restaurada a `jdapalma`),
  build de GitHub Pages `success` y sección visible en https://ava-pt.org/.
- 2026-10-24: T5 completado (modal de detalle). T6 bloqueado por el dato de la entidad contraparte.
  Ruta: inline (la delegación sigue inutilizable por el límite de OpenCode).

## Verificación T5 (observada)
- `node --check script.js` → OK.
- Servido localmente: ids `protocol-modal`, `protocol-modal-close`, `protocol-modal-body` en HTML;
  12 coincidencias de las piezas JS y 10 de las clases CSS nuevas.
- API con las columnas actualizadas (+`servicos`,`beneficios`) → 200, 19 campos, 4/5 servicios y
  3/3 beneficios por registro, sin `pdf_*` ni `notas_internas`.
- Harness funcional `/tmp/opencode/protocolos-harness.js` (script.js real + DOM stub + API real):
  2 protocolos cargados; detalle con título, todos los servicios, todos los beneficios, sin rastros
  de PDF/notas; etiquetas ES correctas → `TODO OK`.

## Verificación (observada)
- `node --check script.js` → OK.
- `python3 -m http.server 8123` + `curl` → sección presente (5 coincidencias `protocolos` en HTML),
  `renderProtocolos` presente en el JS servido.
- Petición real con las columnas exactas del código → HTTP 200, 2 filas, `visivel_landing=true` en ambas,
  exactamente los 17 campos públicos, `grep /pdf_|notas_internas/` sobre la respuesta → sin coincidencias.
- Referencias a `pdf_*`/`notas_internas` en `script.js` → solo dentro del comentario de advertencia (líneas 890-891).
- RDD: `off` (global) → sin revisión nativa; checks ordinarios nominales.
- Pendiente: revisión visual humana en ES/PT y decisión sobre push/PR.

## Hallazgo de seguridad / modelo de datos (no abiertos por esta feature)
1. La RLS de la tabla `protocolos` permite al `anon` leer `pdf_url`, `pdf_key`, `pdf_nome_original`,
   `pdf_tamanho` y `notas_internas` (confirmado con un GET sin `select`). El front no los pide, pero
   cualquier persona puede. Recomendado: restringir esos campos en RLS/admin.
2. El esquema real está en `ava-sys/apps/mail/migrations/20261008170000_create-protocolos.sql`:
   `entidades` es **público** (datos institucionales, pensado "para la landing") y `protocolo_partes`
   es **solo `authenticated`** de forma deliberada por RGPD (contiene email y NIF personal de los
   representantes). La tabla `protocolos` de la landing NO tiene campo de entidad contraparte: el
   vínculo es `protocolo_partes(protocolo_id, entidade_id, papel)`.
   → Para mostrar "con quién firmamos" hace falta una proyección pública no personal (vista de
   `protocolo_partes` con solo ids/papel/ordem) o denormalizar en `protocolos`.
