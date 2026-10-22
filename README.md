# Infraestructura de hgsportafolio.site

Cómo está montado `hgsportafolio.site` por fuera: dominio, DNS, hosting, HTTPS, correo, publicación y monitoreo. Está escrito para quien quiera saber cómo se hizo o montar algo parecido con las mismas piezas: un dominio en Hostinger, dos sitios en Netlify, dos sitios en GitHub Pages y Resend para el correo.

> **Documentación de referencia.** Cada dato se comprobó el **2026-10-22** contra la configuración del repositorio y contra las respuestas públicas (DNS consultado en `8.8.8.8` y en el servidor autoritativo, cabeceras con `curl -sI`, certificados con `openssl`). Los proveedores cambian sus paneles y sus IP: antes de copiar un valor, compruébalo con los comandos de cada capítulo. Lo que no se pudo comprobar desde fuera está marcado como «por confirmar en el panel».
>
> Esta guía no contiene secretos, tokens, claves ni valores de verificación. Donde haría falta uno, se explica qué es y dónde se pone.

## En un vistazo

| Dominio | Qué sirve | Dónde vive | Cómo llega ahí |
| :-- | :-- | :-- | :-- |
| `hgsportafolio.site` | El portafolio (sitio estático) | Netlify, sitio `portafolio-hgs` | Registro `A` hacia el balanceador de Netlify |
| `www.hgsportafolio.site` | Redirección 301 a la raíz | Netlify, mismo sitio | `CNAME` a `portafolio-hgs.netlify.app` |
| `admin.hgsportafolio.site` | Panel privado de edición | Netlify, sitio aparte `admin-hgs` | `CNAME` a `admin-hgs.netlify.app` |
| `docs.hgsportafolio.site` | Documentación técnica | GitHub Pages, sitio de usuario `DevCat-HGS.github.io` | `CNAME` a `DevCat-HGS.github.io` |
| `health.hgsportafolio.site` | Monitor público de disponibilidad, rendimiento y seguridad | GitHub Pages, sitio de proyecto `DevCat-HGS/health` | `CNAME` a `devcat-hgs.github.io` |
| `send.hgsportafolio.site` | Dominio de retorno del correo saliente (no es una web) | Resend | `CNAME` que delega MX y SPF al proveedor |

El sitio y el admin están en Netlify porque necesitan build, cabeceras propias y funciones. La documentación y el monitor están en GitHub Pages porque son HTML estático que ya se genera en GitHub Actions. El monitor va **fuera de Netlify** a propósito: si Netlify o el sitio caen, la página que lo cuenta sigue en pie.

## Capítulos

1. [Mapa de la infraestructura](docs/01-mapa.md): qué dominio sirve qué, en qué proveedor y por qué.
2. [Dominio y DNS en Hostinger](docs/02-dominio-y-dns.md): la zona completa, TTL, propagación, cómo comprobar cada registro y qué registros no tener.
3. [Netlify: sitio principal](docs/03-netlify-sitio-principal.md): repo, build, dominio propio, `www`, HTTPS, sitio de respaldo, previews, redirecciones por idioma y cabeceras.
4. [Netlify: segundo sitio](docs/04-netlify-segundo-sitio.md): un sitio aparte para el panel privado, desde la misma rama.
5. [GitHub Pages](docs/05-github-pages.md): sitio de usuario publicado desde una rama, sitio de proyecto publicado con Actions, dominio propio, certificado y el incidente del 404.
6. [Correo con Resend](docs/06-correo-resend.md): SPF, DKIM, DMARC, el subdominio de retorno y por qué la raíz no tiene MX.
7. [Search Console y analítica](docs/07-search-console-y-analitica.md): verificación, sitemap, indexación y Umami sin cookies.
8. [Publicación y automatización entre repos](docs/08-publicacion-y-automatizacion.md): llaves de despliegue, espejo con lista blanca y el monitor `health`.
9. [Seguridad del lado del hosting](docs/09-seguridad-del-hosting.md): HSTS, cabeceras, `security.txt`, CAA y certificados.
10. [Comprobación final y solución de problemas](docs/10-comprobacion-y-problemas.md): lista de verificación y los fallos típicos con su comando.
11. [Glosario](docs/11-glosario.md).

## Orden recomendado para replicarlo

1. Compra el dominio y deja el DNS en el registrador (capítulo 2).
2. Crea el sitio en Netlify, conéctalo al repositorio y ponle el dominio raíz y `www` (capítulo 3).
3. Si necesitas un panel o una app aparte, crea el segundo sitio de Netlify con otra carpeta base (capítulo 4).
4. Publica en GitHub Pages lo que sea estático y no dependa del sitio principal (capítulo 5).
5. Verifica el dominio en el proveedor de correo antes de enviar nada (capítulo 6).
6. Verifica el dominio en Search Console y envía el sitemap (capítulo 7).
7. Automatiza la publicación entre repositorios con llaves de un solo repo (capítulo 8).
8. Revisa cabeceras, CAA y `security.txt`, y pasa la lista del capítulo 10.

## Comprobación rápida

```bash
# DNS de cada nombre (en Windows, nslookup viene instalado)
nslookup -type=NS hgsportafolio.site 8.8.8.8
nslookup -type=A hgsportafolio.site 8.8.8.8
nslookup -type=CNAME docs.hgsportafolio.site 8.8.8.8

# Respuesta HTTP y cabeceras
curl -sI https://hgsportafolio.site/
curl -sI http://hgsportafolio.site/          # debe dar 301 hacia https
curl -sI https://www.hgsportafolio.site/     # debe dar 301 hacia la raíz
```

---

## Summary in English

Reference documentation (verified on 2026-10-22) of how `hgsportafolio.site` is hosted. The domain is registered at Hostinger, which also runs the DNS zone. The apex points with an `A` record to Netlify's load balancer and `www` is a `CNAME` that Netlify redirects to the apex; a second Netlify site serves the private admin panel on `admin.`. The documentation (`docs.`) is a GitHub Pages user site published from a branch by a workflow that writes the `CNAME` file on every publish, and the status monitor (`health.`) is a GitHub Pages project site deployed from Actions, kept off Netlify so it stays up if the main site goes down. Outgoing mail goes through Resend with SPF and a bounce MX on `send.`, DKIM on `resend._domainkey` and DMARC on `_dmarc`; the apex receives no mail. Certificates are issued by Let's Encrypt (a CAA record allows only that CA at the apex), the Netlify sites send HSTS and a strict set of security headers, and `security.txt` follows RFC 9116. Cross-repository publishing uses deploy keys limited to one repository each, and the public code mirror is built from an allowlist with Actions turned off. No secrets, tokens or verification values are included.
