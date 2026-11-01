---
name: clipskitty-ops
description: Operar la máquina de reels (ClipsKitty + n8n + YouTube/TikTok): diagnosticar por qué no se publica o no se procesa, auditar los datos de ClipsKitty (vídeos, clips, puntuaciones, publicaciones), informar del rendimiento por canal y por streamer, y verificar o añadir canales fuente. Úsala siempre que el usuario hable de sus canales de reels/Shorts, ClipsKitty o Clips Kitty, el flujo de n8n que publica, Clippendo, WoopSocial, "no se ha publicado nada", "qué streamer funciona mejor", "añade estos canales", aunque no nombre la herramienta. Para construir el flujo de n8n desde cero, usa clipskitty-n8n-publisher.
---

# Operar la máquina de reels

La máquina tiene tres piezas, todas en el PC del usuario:

| Pieza | Dónde | Qué hace |
|---|---|---|
| **ClipsKitty** (app propia del usuario, repo `ColinGPT9/clips-studio`) | API local `http://127.0.0.1:8765`, sin autenticación | Vigila canales de YouTube, descarga cada vídeo nuevo una vez, saca clips verticales con subtítulos, título, descripción, hashtags y puntuación 0–100. Guarda el registro de publicaciones. |
| **n8n** | `http://localhost:5678`, flujo «ClipsKitty - Publicar reels (YouTube + TikTok)» | A horas fijas elige los 2 mejores clips (80+) de cada vídeo nuevo y los publica llamando a la API de ClipsKitty. |
| **Destinos** | YouTube directo desde ClipsKitty; TikTok vía WoopSocial desde ClipsKitty | Canal 1: Clippendo (`@clippendoista`, id `UCEQOoBRLPLdM61n6BgZ4KAA`) y TikTok `@clippendiosa`. El plan es llegar a 5 canales con contenido distinto. |

Cada tarea cae en una de cuatro **ramas**. Identifica la rama, lee su sección y cierra con su criterio de terminado:

1. **Diagnóstico**: algo no publica, no procesa, no arranca o da error → [Rama 1](#rama-1-diagnóstico).
2. **Auditoría de datos**: estado de la cola, calidad de clips, si hay material para 5 reels/día → [Rama 2](#rama-2-auditoría-de-datos).
3. **Informe de rendimiento**: qué canal, streamer o tipo de clip funciona → [Rama 3](#rama-3-informe-de-rendimiento).
4. **Fuentes y canales**: añadir/quitar canales vigilados, dar de alta un canal de salida → [Rama 4](#rama-4-fuentes-y-canales).

## Reglas que valen en todas las ramas

- **Lectura primero.** Mira el estado real (API, ejecuciones, logs) antes de cambiar nada. Un `GET` nunca rompe; un `POST`/`PATCH` a ClipsKitty o a n8n cambia el sistema del usuario.
- **Publicar es irreversible.** Llamar a `/clips/{id}/publish`, `/woopsocial/clips/{id}/publish`, ejecutar el flujo completo de n8n o pulsar «Publish» en n8n necesita un «sí» explícito del usuario para esa acción. Para probar, ejecuta solo los nodos de lectura («Execute previous nodes» en el nodo «Kitty - Publicaciones») o publica en privado (`privacy: 'private'`, TikTok `privacyLevel: 'SELF_ONLY'`) y devuelve los nodos a público al acabar.
- **Credenciales las pone el usuario.** Contraseñas, claves de API, OAuth de YouTube/TikTok/WoopSocial: indícale la pantalla y espera. Si aparece un secreto en una salida, no lo repitas.
- **Datos, nunca inventados.** Cada número que des sale de una respuesta de la API, de un export o de lo que el usuario pegó; si falta, dilo como hueco.
- **Explica en su idioma.** El usuario escribe en español y es poco técnico: frases cortas, sin jerga, un resultado por línea.

## Cómo llegar a cada pieza desde esta sesión

Elige la primera vía que funcione y mantenla:

1. **Shell en el PC del usuario** (herramienta de shell remota o Desktop Commander `start_process`): `curl -s http://127.0.0.1:8765/health`. Es la vía más simple para lectura.
2. **Pestaña de Chrome en el propio origen.** Abre una pestaña en `http://127.0.0.1:8765/health` y usa `fetch('/ruta')` desde esa pestaña con la herramienta de JavaScript. Desde otra pestaña (p. ej. la de n8n) el navegador bloquea la petición por CORS y parece que la app está apagada, aunque no lo esté. Cierra la pestaña al terminar.
3. **n8n por su API interna**, desde la pestaña de n8n: `fetch('/rest/workflows/<id>', {headers: {'browser-id': localStorage.getItem('n8n-browserId')}})`. Si responde 401, la sesión de n8n caducó: pide al usuario que inicie sesión (recargar la página puede cerrarla).

Comprobación de vida: `GET /health` → `{"ok":true,"app_version":...}`. Si no responde por ninguna vía, ClipsKitty está cerrado: arrancarlo es decisión del usuario.

## Mapa de la API de ClipsKitty (lo que se usa)

| Para | Llamada | Trampa conocida |
|---|---|---|
| Vídeos procesados | `GET /videos` | `status` puede ser `failed` y aun así tener clips. Los VODs locales tienen `video_id` `local_…` y `channel_id` vacío. |
| Clips de un vídeo | `GET /videos/{video_id}/clips` | Un id inexistente devuelve `200 []`, igual que un vídeo sin clips: confirma el id en `/videos`. |
| Dónde se publicó un clip | `GET /clips/{id}/posts` | Es **el registro** que usa n8n para no duplicar. Incluye filas de WoopSocial y la subida directa a YouTube (solo si ya tiene `youtube_id`). |
| Estado de subida a YouTube | `GET /clips/{id}/publish` | Compara `privacy` con `actual_privacy`: `locked_private` = proyecto de Google sin auditar; los vídeos quedan privados para siempre. |
| Cola de procesado | `GET /queue` → `processing`, `queued`, `paused`, `estimate.per_video_seconds` | Una cola en pausa parece una app rota: mira `paused`. |
| Canales vigilados | `GET /automation/watches`, `GET /automation/items?watch_id=` | `counts.baseline` = vídeos que ya existían al añadir el canal; no se procesan. |
| Automatización global | `GET /automation` | `enabled` debe ser `true`; `watching` = canales activos. |
| WoopSocial | `GET /woopsocial/status`, `GET /woopsocial/connections`, `GET /woopsocial/schedule`, `POST /woopsocial/refresh` | Sin `account_ids`, una publicación va a **todas** las cuentas de esa red. El plan gratis admite 2 cuentas y su cupo compartido de YouTube se agota («Free plans have used up today's shared YouTube publishing capacity»). |
| YouTube directo | `GET /youtube/status` | Límite de 100 subidas/día por proyecto de Google Cloud (desde junio de 2026), contado en `quota`. |
| Documentación viva | `http://127.0.0.1:8765/docs` | Generada del código; manda sobre esta tabla si discrepan. |

## Rama 1: Diagnóstico

Recorre la cadena en orden y para en el primer eslabón roto. Cada paso es una lectura:

1. **¿ClipsKitty vivo?** `GET /health`. No → «ClipsKitty está cerrado o no responde».
2. **¿Vigilando?** `GET /automation` (`enabled`, `watching`) y `GET /automation/watches` (`last_error` por canal, `last_ok_poll_at` reciente). Un canal con error de resolución se arregla re-añadiéndolo con `https://www.youtube.com/channel/UC…` en vez del `@handle`.
3. **¿Procesando?** `GET /queue` (pausada, atascada, cuántos en espera) y `GET /automation/items?watch_id=` (`status`: `waiting_for_queue`, `failed` con `reason`).
4. **¿Hay material?** Vídeos de los últimos 7 días con clips de 80+ y `path` (Rama 2 da el detalle).
5. **¿n8n ejecuta?** Lista de ejecuciones del flujo: `GET /rest/executions?filter={"workflowId":"<id>"}&limit=10` desde la pestaña de n8n, o la pestaña «Executions». Mira si el flujo está publicado (`active: true` y `activeVersionId` = la versión que esperas: editar crea un borrador que no corre hasta pulsar «Publish»).
6. **¿Falla al publicar?** El error del nodo «YouTube - Publicar» o «TikTok - Publicar (WoopSocial)» y `GET /clips/{id}/posts` del clip afectado (`state`, `error`).

Traduce cada error a una frase y una acción. Errores ya vistos:

| Síntoma | Causa | Qué hacer |
|---|---|---|
| `fetch` falla desde otra pestaña | CORS | Vía 2: pestaña en `127.0.0.1:8765`. |
| HTTP 400 «Invalid host header» desde n8n | n8n en Docker llamando a `host.docker.internal` | Añadir cabecera `Host: 127.0.0.1:8765` en los nodos HTTP, o correr n8n sin Docker. |
| n8n «Workflow executed» pero no publica nada | Ningún clip pasa los filtros (sin vídeos nuevos, puntuación < 80, ya publicados) | Normal; confirmarlo con la Rama 2. |
| Clip publicado en varias cuentas de TikTok | ClipsKitty sin el cambio `account_ids` | Aplicar la rama `feature/woopsocial-account-choice` (parche ya preparado) y recompilar. |
| WoopSocial `failed` por capacidad | Plan gratis | Esperar al día siguiente o plan de pago; YouTube mejor por la vía directa. |
| YouTube `locked_private` | Proyecto de Google sin auditoría | El usuario pide la auditoría a YouTube; no es arreglable desde aquí. |
| n8n 401 en `/rest` | Sesión caducada | Que el usuario inicie sesión. |

Para comandos en el PC del usuario: detecta primero el sistema y la shell (Windows/PowerShell en este usuario), usa rutas absolutas, enseña la línea del proceso antes de matar uno y pide confirmación para todo lo destructivo (borrar, sobrescribir, `git reset --hard`, matar procesos ajenos). Puertos ocupados en Windows: `Get-NetTCPConnection -LocalPort 8765`.

Para conectar ClipsKitty como herramienta MCP en Claude Desktop: `python main.py mcp` (checkout) o `api.exe mcp` (instalado, en `resources\backend`). Copia de seguridad del JSON de configuración antes de editarlo, valida el JSON después y confirma que las herramientas aparecen.

**Terminado cuando:** puedes nombrar el eslabón roto con la evidencia que lo demuestra (la respuesta o el error real) y la acción concreta, o has comprobado los 6 pasos sanos y dices qué esperar en la próxima ejecución.

## Rama 2: Auditoría de datos

Perfil de lo que hay en ClipsKitty, para saber si la máquina tiene material y si está sana. Script de lectura para la pestaña en `127.0.0.1:8765` (o Node 18+ cambiando `BASE`):

```js
const BASE = ''; // en Node: 'http://127.0.0.1:8765'
const j = p => fetch(BASE + p).then(r => r.json());
const dias = 7, minimo = 80, limite = Date.now() - dias * 864e5;
const [vids, watches, queue] = await Promise.all([j('/videos'), j('/automation/watches'), j('/queue')]);
const fuente = {}; for (const w of watches) if (w.creator) fuente[w.creator.id] = w.name;
const filas = [];
for (const v of vids) {
  // n8n ignora los VODs locales (video_id local_…): no son material aunque tengan clips buenos.
  if (String(v.video_id).startsWith('local_')) continue;
  const clips = await j('/videos/' + encodeURIComponent(v.video_id) + '/clips');
  const buenos = clips.filter(c => c.score >= minimo && c.path).sort((a, b) => b.score - a.score);
  let publicados = 0;
  for (const c of buenos.slice(0, 2)) {
    const p = await j('/clips/' + c.id + '/posts');
    if ((p.posts || []).length) publicados++;
  }
  filas.push({ video: v.video_id, fuente: fuente[v.creator_id] || v.channel_name || '?', status: v.status,
    reciente: Date.parse(v.updated_at || v.created_at) >= limite, clips: clips.length,
    buenos: buenos.length, top2_publicados: publicados, mejor: clips.reduce((m, c) => Math.max(m, c.score), 0) });
}
JSON.stringify({ watches: watches.length, con_error: watches.filter(w => w.last_error).map(w => w.name + ': ' + w.last_error),
  cola: { procesando: (queue.processing || []).length, en_espera: (queue.queued || []).length, pausada: queue.paused,
          seg_por_video: queue.estimate && queue.estimate.per_video_seconds },
  filas });
```

Con el resultado, comprueba y reporta:

- **Material suficiente**: cuenta como n8n: solo vídeos de canales vigilados (nunca `local_…`), de los últimos 7 días, top 2 de 80+ con `path`, sin ninguna fila en el registro para esa red. Compáralo con lo que consume el plan: n8n publica **1 clip por canal de salida en cada turno**, 5 turnos al día. Menos de un día de margen = aviso.
- **Calidad por fuente**: qué canales vigilados dan vídeos sin ningún clip de 80+, o vídeos `failed`. Son candidatos a quitar.
- **Formato**: todos los canales vigilados con `options.captions: true` y `options.caption_style.position: 'bottom'`; un fotograma de un clip reciente en vertical.
- **Atascos**: vídeos `failed`, cola parada, canales con `last_error`, `waiting_for_queue` largo.
- **Capacidad**: `seg_por_video` de la cola da cuántos vídeos caben al día (86 400 / seg_por_video). Si los canales vigilados suben más vídeos de los que caben, la cola crece sin fin: propone quitar fuentes o bajar `max_clips`.
- **Coherencia del registro**: clips del top 2 publicados solo en una red (falta YouTube o TikTok) y filas con `state` `failed`, que n8n no reintenta solo.
- **Disco**: muchos vídeos descargados sin borrar; `delete_sources` solo borra lo que publica la propia vigilancia, no lo que publica n8n.

Formato de salida:

```
## Estado de la máquina (fecha)
- Material listo: N clips → X días al ritmo actual
- Problemas: lista corta con gravedad (alto/medio/bajo)
| Fuente | Vídeos 7d | Con clip 80+ | Mejor puntuación | Publicados |
- Siguientes pasos: 1–3 acciones
```

**Terminado cuando:** cada vídeo de la ventana está contado, cada problema lleva su evidencia (id de vídeo, clip o canal) y hay una conclusión de si hay material para mañana.

## Rama 3: Informe de rendimiento

ClipsKitty y WoopSocial **no miden** visualizaciones. Los números salen de **YouTube Studio** y de la **analítica de TikTok** (exportación CSV o lo que el usuario pegue); `GET /clips/{id}/posts` aporta el enlace de cada publicación y el título permite atribuir cada Short a su streamer («Streamer - título»).

Método (METER, adaptado a Shorts):

1. **Mapa**: objetivo del canal (crecer suscriptores, visualizaciones, monetizar) → 1 métrica principal + 2 de apoyo. Ventana fija (7 o 28 días).
2. **Extrae** del panel nativo: por Short, visualizaciones, % que se quedan a verlo frente a deslizar, duración media vista / % visto, compartidos y suscriptores ganados. En TikTok: tiempo medio, % completado, compartidos.
3. **Recorta** lo vanidoso (likes sueltos, impresiones) y quédate con retención, compartidos y suscriptores ganados.
4. **Evalúa** con honestidad: compara con la media del propio canal, no con otros; 1–2 Shorts por fuente es muestra pequeña, dilo; correlación no es causa (un streamer puede rendir por el tema del vídeo, no por él).
5. **Reporta y cierra el ciclo**: primero el resultado, luego la tabla, al final exactamente 3 recomendaciones que se puedan ejecutar en la máquina.

Tabla por fuente (la pieza que más decide):

```
| Streamer | Shorts | Visualizaciones medianas | % se queda | Compartidos/1000 | Subs ganados | Veredicto |
```

Las recomendaciones apuntan a palancas reales del sistema: quitar o añadir canales vigilados (Rama 4), subir o bajar el umbral de puntuación (`MIN_PUNTOS` en el nodo «Top 2 por video (80+)»), cambiar horas (nodo de disparo), o `max_clips` por vídeo en los canales vigilados.

**Terminado cuando:** cada número cita su fuente (panel y ventana), los huecos están marcados, la tabla cubre todas las fuentes con datos y hay 3 recomendaciones atadas a una palanca concreta.

## Rama 4: Fuentes y canales

### Verificar un canal fuente

Fuente de verdad: la propia página de YouTube, nunca un listado de terceros. Desde una pestaña en `https://www.youtube.com/robots.txt` (el `fetch` necesita el mismo origen):

```js
const nombres = ['IbaiLlanos', 'TheGrefg']; // handles o nombres a comprobar
const out = [];
for (const q of nombres) {
  const t = await (await fetch('/results?sp=EgIQAg%253D%253D&search_query=' + encodeURIComponent(q))).text();
  const re = /"channelRenderer":\{"channelId":"(UC[\w-]{22})","title":\{"simpleText":"([^"]+)"\}[\s\S]{0,4000}?"canonicalBaseUrl":"\/@([^"]+)"/g;
  let m, got = [];
  while ((m = re.exec(t)) && got.length < 3) {
    const seg = t.slice(m.index, m.index + 6000);
    const subs = (seg.match(/"simpleText":"([^"]*suscriptores[^"]*)"/) || [])[1] || '';
    got.push(`${m[2]} @${m[3]} ${m[1]} ${subs}${/CHECK_CIRCLE_THICK|BADGE_STYLE_TYPE_VERIFIED/.test(seg) ? ' ✓' : ''}`);
  }
  out.push(q + ' => ' + got.join(' || '));
}
out.join('\n');
```

Elige el canal **principal** (verificado, más suscriptores, nombre que coincide), no el de VODs ni el de clips de fans. Si el primer resultado no es la persona (pasó con «elded» → otra persona), busca por el nombre artístico y anota la duda para el usuario.

### Añadir canales vigilados

Con el usuario de acuerdo con la lista, desde la pestaña en `127.0.0.1:8765`, de uno en uno o en lotes de 4 (cada alta resuelve el canal con yt-dlp y tarda):

```js
await fetch('/automation/watches', { method: 'POST', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ platform: 'youtube', channel: '@Handle', preset: 'standard',
                         options: { max_clips: 3, captions: true, caption_style: { position: 'bottom' } },
                         publish: { mode: 'off' } }) });
// después, con el id devuelto:
await fetch('/automation/watches/ID', { method: 'PATCH', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ min_minutes: 3, backlog: 'newest' }) });
```

Las opciones fijan el **formato innegociable** del usuario: subtítulos quemados, abajo, y vídeo vertical 9:16. Lo vertical viene de `clips.vertical: true` en la configuración global de ClipsKitty (no hay opción por canal); compruébalo midiendo un fotograma: `new Image()` con `src = '/clips/ID/frame?t=2'` debe dar alto > ancho (p. ej. 1280×2276). `PATCH options` reemplaza el objeto entero: manda siempre las tres claves juntas. `publish.mode: 'off'` porque publica n8n; si se dejara en `auto`, ClipsKitty publicaría también y habría duplicados. Si un `@handle` falla al resolverse, usa `https://www.youtube.com/channel/UC…`. Comprueba al final `GET /automation/watches`: total esperado y ningún `last_error`.

### Dar de alta un canal de salida (canales 2 a 5)

1. El usuario conecta el canal de YouTube en ClipsKitty (Ajustes → YouTube, aparece en `GET /youtube/status` → `accounts`) y su TikTok en WoopSocial desde ClipsKitty (aparece en `GET /woopsocial/connections`). Más de 2 cuentas en WoopSocial exige plan de pago.
2. Añade sus canales fuente como arriba.
3. En n8n, nodo «Videos de los canales», añade un bloque a `CANALES` con `nombre`, `youtube_channel_id` (id `UC…` del canal de salida), `tiktok_account_id` (id de la cuenta en WoopSocial) y `fuentes` (`'UC…': 'Nombre en el título'`). Un canal fuente pertenece a un solo bloque.
4. Guarda el flujo, ejecuta solo los nodos de lectura para comprobar que aparecen sus vídeos y pide el «sí» para pulsar «Publish».

Sin el cambio `account_ids` en ClipsKitty, con varias cuentas de TikTok cada clip saldría en todas: confírmalo antes del paso 4.

**Terminado cuando:** cada canal nuevo está verificado en YouTube, vigilado sin error, asignado a un único canal de salida y la lectura de n8n lo ve; o tienes la lista de dudas que solo el usuario puede resolver.

## Datos que conviene tener a mano

- Flujo n8n: id `ljrV8MaL4z7WzPLO`. Horario 12, 14, 16, 18 y 20 h (Europe/Amsterdam). Nodos: «5 al dia (12-20h)» → «Kitty - Canales vigilados» → «Kitty - Videos» (Execute Once) → «Videos de los canales» (config `CANALES`) → «Kitty - Clips» → «Top 2 por video (80+)» → «Kitty - Publicaciones» → «Elegir siguiente clip» → «Falta YouTube?»/«Falta TikTok?» → nodos de publicar.
- Formato obligatorio de cada reel: vertical 9:16, subtítulos quemados y colocados abajo. Si un canal vigilado no tiene `captions: true` y `caption_style.position: 'bottom'` en `options`, corrígelo (es un hallazgo de prioridad alta en la Rama 2).
- El flujo no reintenta una publicación fallida: cualquier fila en el registro cuenta como hecha. Reintentar es decisión del usuario.
- La descripción fija de YouTube en ClipsKitty (texto sobre pipas de girasol) es intencionada: déjala.
- Los bots antiguos de Copilot («Descarga de VODs nuevos», «VODs a Shorts») quedaron sustituidos por la vigilancia de ClipsKitty.
