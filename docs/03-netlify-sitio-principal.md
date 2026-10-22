# 3. Netlify: sitio principal

[Volver al índice](../README.md)

El portafolio es un sitio estático (Vite y React) en el sitio de Netlify `portafolio-hgs`. Su configuración vive en el repositorio, en `portfolio/netlify.toml`, para que cualquier cambio de build, redirecciones o caché pase por un commit y por revisión.

## Conectar el repositorio

1. En Netlify: Add new site → Import an existing project → GitHub, y autoriza la app de Netlify solo sobre el repositorio que vas a publicar.
2. **Branch to deploy:** `main`.
3. **Base directory:** `portfolio`. El código del sitio está en esa carpeta del repositorio; Netlify entra ahí, instala dependencias y busca el `netlify.toml`.
4. **Build command** y **Publish directory:** déjalos vacíos en el panel. Netlify los toma de `portfolio/netlify.toml`, y lo que dice el archivo gana sobre el panel.

Lo que dice `portfolio/netlify.toml`:

```toml
[build]
  command = "npm run images && npm run build"
  publish = "dist"

[build.environment]
  VITE_SITE_URL = "https://hgsportafolio.site"
```

- `npm run images` convierte a WebP las imágenes nuevas antes del build.
- `npm run build` es `tsc --noEmit && vite build`: si el tipado falla, el build falla y no se publica nada.
- `publish = "dist"` es relativo a la carpeta base, o sea `portfolio/dist`.
- `VITE_SITE_URL` es el dominio de producción. De ahí salen el canonical, los `hreflang`, Open Graph, los datos estructurados y el sitemap. No es un secreto; va en el archivo para que el build de un preview también genere direcciones de producción.

La versión de Node que usa Netlify no está fijada en el repositorio (no hay `.nvmrc` ni `NODE_VERSION`): por confirmar en el panel (Site configuration → Build & deploy → Dependency management). Fijarla evita sorpresas cuando Netlify cambia su versión por defecto.

## Dominio propio y www

1. En el sitio: Domain management → Add a domain → `hgsportafolio.site`. Netlify ofrece agregar también `www`.
2. Como el DNS se queda en Hostinger, elige la opción de DNS externo. Netlify muestra qué registros crear: `A` de la raíz hacia su balanceador y `CNAME` de `www` hacia `<sitio>.netlify.app`. Créalos como en el [capítulo 2](02-dominio-y-dns.md).
3. Marca la raíz como **dominio principal** (primary domain). Netlify redirige con 301 el otro nombre hacia el principal.

Resultado comprobado:

```bash
curl -sI http://hgsportafolio.site/        # 301 -> https://hgsportafolio.site/
curl -sI https://www.hgsportafolio.site/   # 301 -> https://hgsportafolio.site/
curl -sI http://www.hgsportafolio.site/    # 301 -> https://www.hgsportafolio.site/ (y de ahí a la raíz)
```

## HTTPS con Let's Encrypt

Cuando el DNS ya apunta a Netlify, Domain management → HTTPS → Verify DNS configuration y luego Provision certificate. Netlify pide a Let's Encrypt un certificado que cubre `hgsportafolio.site` y `www.hgsportafolio.site`, y lo renueva solo antes de que venza (los de Let's Encrypt duran unos 90 días).

```bash
echo | openssl s_client -connect hgsportafolio.site:443 -servername hgsportafolio.site 2>/dev/null \
  | openssl x509 -noout -issuer -enddate -ext subjectAltName
```

El 2026-10-22 el emisor era Let's Encrypt y los nombres del certificado, la raíz y `www`.

Si hay un registro `CAA` en la zona, debe permitir `letsencrypt.org` o el certificado no se emite (ver [capítulo 9](09-seguridad-del-hosting.md)).

Netlify redirige HTTP a HTTPS por su cuenta cuando el certificado está activo. No hace falta una regla en `netlify.toml`.

## El sitio de respaldo `portafolio-hgs.netlify.app`

Todo sitio de Netlify tiene su dirección `<nombre>.netlify.app`. Aquí se dejó abierta a propósito, sin redirección a la raíz, como respaldo gratuito: si un día el dominio tiene un problema de DNS o de renovación, el sitio sigue accesible ahí.

No compite con el dominio propio en Google porque cada página declara su versión canónica en `hgsportafolio.site`:

```bash
curl -s https://portafolio-hgs.netlify.app/ | grep -o '<link rel="canonical"[^>]*>'
# <link rel="canonical" href="https://hgsportafolio.site/" />
```

Google trata las dos direcciones como copias y solo indexa la canónica. La alternativa (un 301 de `netlify.app` hacia la raíz) también es válida; aquí se probó y se quitó para conservar el respaldo.

## Deploy previews

Netlify construye un preview por cada pull request contra `main`, en una dirección propia (`deploy-preview-<número>--portafolio-hgs.netlify.app`), y lo reporta a GitHub como un estado del commit llamado `netlify/<sitio>/deploy-preview`. Un workflow del repositorio escucha ese evento `status` y deja en el PR un comentario con el enlace, editado en cada despliegue.

El preview usa el mismo `netlify.toml`, así que su canonical apunta a producción y no se indexa como sitio aparte.

Si un check de GitHub Actions detiene o no el despliegue de producción depende de la configuración del panel: por confirmar. Lo que sí lo detiene siempre es que falle el build de Netlify.

## Redirecciones por idioma

La raíz `/` está en español. Un visitante con el navegador en inglés llega a `/en/`. Lo hacen reglas como esta en `netlify.toml`, una por cada página con versión en inglés:

```toml
[[redirects]]
  from = "/"
  to = "/en/"
  status = 302
  force = true
  conditions = { Language = ["en"] }
```

Por qué así:

- **302 y no 301:** la redirección depende del visitante, no es un cambio permanente de dirección.
- **`Language`:** Netlify mira primero la cookie `nf_lang` y después la cabecera `Accept-Language`. El botón de idioma del sitio fija esa cookie, así una elección manual se respeta.
- **Los rastreadores no mandan idioma** y reciben `/`, la versión en español; la inglesa la descubren por `hreflang` y por el sitemap.
- **Cada página nueva necesita su regla.** La condición no se hereda.

```bash
curl -sI -H "Accept-Language: en" https://hgsportafolio.site/   # 302, Location: /en/
curl -sI https://hgsportafolio.site/                            # 200
```

## Cabeceras: caché y seguridad

Hay dos sitios donde se declaran cabeceras en Netlify, y aquí se usan los dos:

1. **`[[headers]]` en `netlify.toml`** para la caché.
2. **Un archivo `_headers`** dentro de la carpeta publicada, que el build escribe, para las cabeceras de seguridad.

### Caché

```toml
# Archivos con hash en el nombre: nunca cambian.
[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

# Archivos que conservan el nombre aunque cambien: revalidar siempre.
[[headers]]
  for = "/cv/*"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"
```

Con `must-revalidate` el navegador pregunta en cada visita; si no hubo cambio, Netlify contesta 304 (por la `ETag`) y no se descarga nada.

### Seguridad

Las cabeceras de seguridad se definen una sola vez en código (`portfolio/seo/headers.ts`, en el [espejo público](https://github.com/DevCat-HGS/hgsportafolio)). El build de Vite las escribe en `dist/_headers` para Netlify, y el servidor de `vite preview` envía las mismas, así las pruebas del navegador ejercitan la política real.

El patrón de un `_headers`:

```text
/*
  Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
  Cross-Origin-Opener-Policy: same-origin
  Strict-Transport-Security: max-age=31536000; includeSubDomains
```

Ese es un ejemplo genérico, no la política exacta del sitio. Ideas que sí aplica la política real:

- `default-src 'self'` y se abren solo los orígenes que hacen falta. El script de analítica solo se permite si la analítica está activa en el build.
- Sin scripts en línea: lo que antes iba en un `<script>` dentro del HTML vive en un archivo propio, y `script-src` no necesita `'unsafe-inline'`.
- `frame-ancestors 'none'` y `X-Frame-Options: DENY` impiden que otro sitio meta el portafolio en un `iframe`.

Comprobación:

```bash
curl -sI https://hgsportafolio.site/ | grep -iE "content-security|strict-transport|x-frame|x-content|referrer|permissions|cross-origin"
```

Más detalle en el [capítulo 9](09-seguridad-del-hosting.md).

## Otros archivos que sirve Netlify

- `404.html` en la raíz de lo publicado: Netlify lo usa para cualquier ruta que no exista (con estado 404).
- `/.well-known/security.txt`: contacto de seguridad ([capítulo 9](09-seguridad-del-hosting.md)).
- `/robots.txt` y `/sitemap.xml`: los genera el build ([capítulo 7](07-search-console-y-analitica.md)).
