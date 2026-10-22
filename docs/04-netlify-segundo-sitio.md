# 4. Netlify: segundo sitio desde el mismo repositorio

[Volver al índice](../README.md)

El panel privado de edición se publica en `admin.hgsportafolio.site` desde el **mismo repositorio** y la misma rama `main`, pero como un sitio de Netlify aparte (`admin-hgs`). Este capítulo cubre solo el hosting; cómo funciona el panel por dentro queda fuera de esta guía.

## Por qué un sitio aparte

- **Su propia política de cabeceras.** El panel no se indexa y tiene una CSP distinta a la del portafolio. En un solo sitio habría que mezclar las dos.
- **Sus propios despliegues.** Un cambio de contenido del portafolio no debe reconstruir ni publicar el panel, y al revés.
- **Su propia configuración.** Sus variables se configuran en el panel de Netlify de ese sitio y no se publican. El sitio principal no las ve.

## Pasos

1. Netlify → Add new site → Import an existing project → el mismo repositorio.
2. **Branch to deploy:** `main`.
3. **Base directory:** `admin`. Netlify lee `admin/netlify.toml`.
4. Domain management → Add a domain → `admin.hgsportafolio.site`.
5. En Hostinger, `CNAME` de `admin` hacia `admin-hgs.netlify.app` ([capítulo 2](02-dominio-y-dns.md)).
6. HTTPS → Provision certificate. El certificado cubre solo `admin.hgsportafolio.site`.

## Cuándo construye

`admin/netlify.toml` define el build y una regla para saltarlo:

```toml
[build]
  command = "npm run build"
  publish = "dist"
  ignore = "git diff --quiet $CACHED_COMMIT_REF $COMMIT_REF -- ."
```

`ignore` es un comando que Netlify corre antes de construir. Si termina con código 0, Netlify cancela el build. `git diff --quiet ... -- .` termina en 0 cuando no hay cambios dentro de la carpeta base (`.` es `admin/`, porque el comando corre ahí) entre el último commit desplegado (`$CACHED_COMMIT_REF`) y el actual (`$COMMIT_REF`). Así el segundo sitio solo se reconstruye cuando cambia su propia carpeta.

Este patrón sirve para cualquier monorepo con varios sitios de Netlify. Si un sitio depende también de código fuera de su carpeta, agrega esas rutas al final del `git diff`.

## Una sola página

El panel es una aplicación de una sola página. Toda ruta sirve el mismo `index.html`:

```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

El estado 200 hace que sea una reescritura y no una redirección: la dirección del navegador no cambia.

## Fuera de los buscadores

El sitio envía `X-Robots-Tag: noindex, nofollow` en todas sus respuestas y publica un `robots.txt` propio:

```bash
curl -sI https://admin.hgsportafolio.site/ | grep -i x-robots-tag
```

`noindex` en la cabecera funciona aunque el buscador llegue por un enlace externo; `robots.txt` solo pide no rastrear.

## Previews

Igual que el sitio principal, Netlify publica un preview por pull request y lo reporta como estado `netlify/admin-hgs/deploy-preview` (ver [capítulo 3](03-netlify-sitio-principal.md)). La regla `ignore` también se evalúa en los previews, así que un PR que no toca `admin/` normalmente se salta el build de este sitio. Qué commit toma Netlify como `$CACHED_COMMIT_REF` en el primer preview de una rama conviene comprobarlo en el registro del deploy.
