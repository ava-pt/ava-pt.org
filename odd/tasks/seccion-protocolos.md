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
- [x] T6: Entidad contraparte ("con quién firmamos") — vía vista pública
      `protocolos_partes_public` (solo `protocolo_id, entidade_id, papel, ordem`, filtrada por
      `visivel_landing`) + join con `entidades` (ya pública). Migración
      `ava-sys/apps/mail/migrations/20261024111004_protocolos-partes-public.sql` APLICADA.

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

## Verificación T6 (observada)
- Migración creada con `npx -y @insforge/cli db migrations new protocolos-partes-public` →
  `20261024111004_protocolos-partes-public.sql`, aplicada con `db migrations up <file>` → `✓ Applied`.
- Vista con `anon`: 4 filas (2 protocolos × papel `ava`/`parceiro`) y **exactamente 4 columnas**
  (`protocolo_id, entidade_id, papel, ordem`): sin `representante_email`, `representante_nome`,
  `representante_cargo` ni `coordinadores`.
- Regresión: `protocolo_partes` (tabla base) sigue devolviendo **401 permission denied** para `anon`.
- Harness con la vista viva → resolución automática de contraparte `true` en ambos protocolos:
  "Movimento 55+ Associação" y "Abogada Paula Calabria da Silva". `node --check` → OK.
- Commits: `6e5beb5` (modal) y `eb95e32` (contraparte) en `feat/seccion-protocolos`.

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
1. [RESUELTO 2026-10-24] La RLS/privilegios de `protocolos` permitían al `anon` leer `pdf_url`,
   `pdf_key`, `pdf_nome_original`, `pdf_tamanho`, `notas_internas`, `image_key` e `image_url`.
   Restringido con la migración
   `ava-sys/apps/mail/migrations/20261024134748_restrict-protocolos-public-columns.sql`
   (commit `5726f76` en `main`): `REVOKE SELECT ... FROM anon` + `GRANT SELECT (26 columnas públicas)`.
   `authenticated` conserva las 33 columnas (plataforma intacta).
   Verificado como `anon`: landing 200 con sus columnas exactas; `select=*`, `pdf_url`,
   `pdf_key`, `notas_internas`, `image_key` → 401 `permission denied for table protocolos`.
2. El esquema real está en `ava-sys/apps/mail/migrations/20261008170000_create-protocolos.sql`:
   `entidades` es **público** (datos institucionales, pensado "para la landing") y `protocolo_partes`
   es **solo `authenticated`** de forma deliberada por RGPD (contiene email y NIF personal de los
   representantes). La tabla `protocolos` de la landing NO tiene campo de entidad contraparte: el
   vínculo es `protocolo_partes(protocolo_id, entidade_id, papel)`.
   → Resuelto con la migración `20261024111004_protocolos-partes-public.sql`: vista pública
   `protocolos_partes_public` (solo ids/papel/ordem, filtrada por `visivel_landing`) que la landing
   cruza con `entidades`. `protocolo_partes` sigue siendo solo `authenticated`.
3. [PENDIENTE] Barrido empírico con la clave `anon` sobre tablas con columnas de nombres sensibles
   (`?limit=1`): con filas visibles y datos personales quedan
   `profiles` (8 filas con `email`, `telefone`), `favoritos` (1 fila con `email`) y
   `email_send_log` (54 filas con `recipient_email`). Menor: `events.image_key` y
   `entidades.logo_key` (bucket privado: la clave no da acceso) y `entidades.nif/notas`
   (hoy en `null`; el NIF de AVA es dato institucional público). Sin exposición:
   `ava_tasks`, `directiva`, `email_unsubscribe_tokens`, `meeting_participants`,
   `suppressed_emails`, `task_assignees` (RLS devuelve 0 filas a `anon`).
