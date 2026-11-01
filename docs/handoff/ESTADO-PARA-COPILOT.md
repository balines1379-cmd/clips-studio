# Traspaso: estado de la máquina de reels (1 nov 2026)

Escrito por Claude (Cowork) para que GitHub Copilot continúe. El usuario es poco técnico, escribe en español y quiere frases cortas. **No publicar nada, no gastar dinero ni cambiar cuentas sin su "sí" explícito.** Los secretos (claves de Tailscale, API keys) no están aquí: pídeselos al usuario.

## Arquitectura (todo en el PC Windows del usuario)
- **ClipsKitty 2.1.0** (instalado) — API local `http://127.0.0.1:8765`, sin auth. Vigila canales de YouTube (`/automation/watches`), descarga cada vídeo nuevo, saca clips verticales 9:16 con subtítulos y puntuación 0-100. IA y transcripción ya van por OpenRouter (nube). El PC **no tiene GPU**: el render (seguimiento de caras + codificación) es el cuello de botella (~1 h/vídeo).
- **n8n** local `http://localhost:5678`, flujo `ljrV8MaL4z7WzPLO` "ClipsKitty - Publicar reels (YouTube + TikTok)", publicado. Turnos 12, 14, 16, 18, 20 h (Europe/Amsterdam). Por cada canal de salida publica 1 clip por turno: de cada vídeo nuevo (7 días, nunca `local_…`) los 2 mejores con 80+, vídeo más reciente primero. Publica llamando a ClipsKitty: `POST /clips/{id}/publish` (YouTube directo, `channel_id`) y `POST /woopsocial/clips/{id}/publish` (TikTok). Registro anti-duplicados: `GET /clips/{id}/posts`. Config de canales en el nodo Code "Videos de los canales" (`CANALES`).
- Copia del flujo: `clipskitty-publicar-reels.n8n.json` (puede estar algo desfasada respecto al publicado: el publicado tiene 6 canales y la bandera `tiktok`).
- Skill de operación con todos los trucos y trampas: `clipskitty-ops-SKILL.md`.

## Canales de salida (YouTube, todos conectados en ClipsKitty)
| Canal | id | TikTok | Fuentes vigiladas |
|---|---|---|---|
| Clippendo @clippendoista | UCEQOoBRLPLdM61n6BgZ4KAA | sí (@clippendiosa, WoopSocial) | 30 streamers hispanos |
| Mindset Clips | UC68Z0G6BZa1FDyKXVMOXrww | no | DOAC, Hormozi, MFM, Bilyeu, Ali Abdaal, Codie Sanchez |
| Brain & Body Clips | UChjVXK0j58pQSld1ORg7Hrw | no | Huberman, Attia, Mel Robbins, Rich Roll, Jay Shetty, Chatterjee |
| Riendo Clips | UCqiShokkz1NA-o_gNda0d1Q | no | Wild Project, Pija y Quinqui, Estirando el chicle, Poco se habla, Worldcast, La Revuelta |
| Fútbol Hablado | UC4nb-PvxyvpgTE3w1hHDAKQ | no | Chiringuito Inside, Partidazo COPE, Carrusel, Post United, Relevo |
| AI Unfiltered | UCFzMIoghqEauDGS2wERmb5w | no | Lex Fridman, Dwarkesh, No Priors, All-In, 20VC, Y Combinator |

59 canales vigilados, todos con `options: {max_clips: 2, captions: true, caption_style: {position: 'bottom'}}`, `min_minutes: 3`, `publish.mode: 'off'` (publica n8n). **Requisito innegociable del usuario: subtítulos quemados, abajo, vídeo vertical.**

## Pendiente
1. **Verificar que el flujo v3 (6 canales) está publicado en n8n.** El último "Publish" lo tuvo que pulsar el usuario; si `activeVersionId` ≠ `versionId`, pedirle que pulse Publish.
2. **RunPod render remoto** (presupuesto: 20 € cada 2 semanas). Pod "clipskitty-render" RTX 2000 Ada (0,24 $/h) en "Deploy when available" hasta el 2 nov 13:26. Cuando arranque: instalar Tailscale en el pod (`tailscaled --tun=userspace-networking`, clave de auth del usuario), clonar este repo, `python main.py render-worker --pair <IP tailscale del PC>:8766 <código>` (el código lo genera el usuario en ClipsKitty → Ajustes → Advanced → Remote rendering → Add a render PC; vale 10 min y una vez). En RunPod NVENC no funciona: usa x264. Luego "Render clips on: Automatic". Encenderlo solo con cola cargada para no pasarse de presupuesto.
3. **Foto y descripción de los 5 canales nuevos**: imágenes y textos preparados por Claude (pedírselos al usuario o regenerar); hay que subirlos en YouTube Studio con la cuenta dueña de los canales.
4. **TikTok para los canales nuevos**: requiere 5 cuentas TikTok + WoopSocial de pago + instalar la rama `feature/woopsocial-account-choice` de este fork (añade `account_ids` a la publicación por WoopSocial; sin ella un clip sale en todas las cuentas TikTok). Luego rellenar `tiktok: true` y `tiktok_account_id` en `CANALES`.
5. **Capacidad**: 59 fuentes generan más vídeos de los que caben (~24/día). Si la cola crece sin parar, quitar fuentes flojas o activar RunPod. ClipsKitty no tiene filtro de duración máxima; añadirlo al código sería útil (saltarse vídeos > 90 min).

## Trampas conocidas
- La API de ClipsKitty rechaza Host distinto de 127.0.0.1/localhost y bloquea CORS: desde un navegador, hacer fetch desde una pestaña abierta en `http://127.0.0.1:8765`.
- n8n `/rest/*` necesita la cabecera `browser-id` (localStorage `n8n-browserId`); editar crea borrador, hay que pulsar Publish.
- WoopSocial gratis: 2 cuentas y cupo diario compartido de YouTube (por eso YouTube va directo).
- YouTube directo: 100 subidas/día por proyecto; si `actual_privacy` = `locked_private`, el proyecto de Google no está auditado.
- La descripción fija de YouTube en ClipsKitty (texto sobre pipas) es intencionada.
- Los bots antiguos de Copilot ("Descarga de VODs nuevos", "VODs a Shorts") deben estar apagados: los sustituye la vigilancia de ClipsKitty.
