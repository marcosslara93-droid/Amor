# StreamFlix PRO — Web

Clon estilo Netflix (películas, series y canales de TV en vivo) desplegado
íntegramente en **Cloudflare Pages**: el sitio estático y la capa de API viven
en el mismo proyecto y se despliegan con un solo comando.

## Arquitectura

```
[Navegador]
    │  fetch("/api/movies") — mismo dominio, sin CORS
    ▼
[Cloudflare Pages]
    ├── public/        → sitio estático (HTML/CSS/JS, se sirve tal cual)
    └── functions/api/ → Pages Functions (edge, se despliegan junto al sitio)
            │  agrega el bearer token desde los secrets, nunca visible al cliente
            ▼
    [streamflix-api]   → tu backend FastAPI ya existente
            │
            ▼
    Xtream Codes (dplatino, jumangis, p2premium, micaplayapp) + TMDB
```

Esto cumple lo mismo que ya tenías planeado (un proxy delante de FastAPI para
que apps de interceptación de tráfico como Reqable nunca vean host/usuario/
contraseña reales), pero usando **Pages Functions** en vez de un Worker
aparte: un solo proyecto, un solo deploy.

## Estructura

```
streamflix-web/
├── public/                  # sitio estático
│   ├── index.html
│   ├── css/styles.css
│   └── js/
│       ├── config.js        # categorías mostradas, base de la API
│       ├── api.js           # fetch wrappers
│       ├── carousel.js      # filas estilo Netflix + grid de canales
│       ├── player.js        # HLS.js — playVod() y playLive() separados
│       └── app.js           # orquestador (hero, filas, nav, modal, búsqueda)
├── functions/api/           # backend edge (Cloudflare Pages Functions)
│   ├── movies.js
│   ├── series.js
│   ├── live.js
│   ├── details/[id].js
│   └── stream/[id].js
├── wrangler.toml
├── package.json
└── README.md
```

## ⚠️ Antes de desplegar: ajustar el contrato de datos

Los `functions/api/*.js` reenvían tal cual a tu backend
(`GET {BACKEND_URL}/movies`, `/series`, `/live`, `/details/:id`,
`/stream/:id`). Revisa que:

1. Esos endpoints existan en tu `streamflix-api` con esos nombres (o edita
   las rutas en `functions/api/*.js` para que coincidan).
2. Los campos que el frontend espera coincidan con lo que devuelve tu API:
   `id`, `title`, `posterPath`, `backdropPath`, `year`, `rating`, `overview`,
   `genres`, `duration` — ajusta `app.js` / `carousel.js` si tu backend usa
   otros nombres (ej. `poster_path` en vez de `posterPath`).
3. `/stream/:id` devuelva `{ "url": "https://...m3u8?sig=..." }` (la URL
   firmada HMAC con TTL que ya generas).

## Desplegar en Cloudflare Pages

No tengo acceso directo a tu cuenta de Cloudflare desde este entorno (sin
navegador para el login OAuth ni salida de red hacia la API de Cloudflare),
así que estos comandos los corres tú, localmente o en tu propia terminal:

```bash
cd streamflix-web
npm install          # instala wrangler como devDependency
npx wrangler login   # abre el navegador para autenticar tu cuenta

# Configura los secrets (nunca van en el código ni en git)
npx wrangler pages secret put BACKEND_URL --project-name=streamflix-web
# → pega la URL pública de tu streamflix-api, ej. https://api.tudominio.com

npx wrangler pages secret put BACKEND_API_KEY --project-name=streamflix-web
# → pega el bearer token que ya usa tu backend

# Primer deploy (crea el proyecto si no existe)
npx wrangler pages deploy public --project-name=streamflix-web
```

Deploys siguientes: `npm run deploy`.

Desarrollo local con Functions incluidas:

```bash
npm run dev   # wrangler pages dev public — sirve todo en localhost, Functions incluidas
```

### Dominio propio

Desde el dashboard de Cloudflare → tu proyecto Pages → **Custom domains** →
agrega tu dominio (debe estar en la misma cuenta de Cloudflare). El
certificado TLS se emite automáticamente.

### Opcional: caché con KV

`functions/_shared/proxy.js` ya cachea las respuestas de catálogo 2 minutos
en el edge vía `cf.cacheTtl`. Si más adelante quieres cachear también entre
despliegues (no solo por request), se puede añadir un KV namespace — avísame
y lo integro.

## Alternativa: conectar Cloudflare vía MCP

Si conectas el conector de **Cloudflare Developer Platform** en este chat,
puedo crear el proyecto Pages, configurar KV namespaces y gestionar Workers
directamente desde la conversación en vez de darte comandos para tu
terminal.
