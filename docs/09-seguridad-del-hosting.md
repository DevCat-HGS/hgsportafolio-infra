# 9. Seguridad del lado del hosting

[Volver al índice](../README.md)

Lo que se puede configurar en el hosting y en el DNS, sin tocar la aplicación.

## HTTPS en todo

| Nombre | HTTP → HTTPS | Quién redirige |
| :-- | :-- | :-- |
| `hgsportafolio.site`, `www` | 301 | Netlify, al tener el certificado activo |
| `admin.` | 301 | Netlify |
| `docs.`, `health.` | 301 | GitHub Pages, con Enforce HTTPS marcado |

```bash
for h in hgsportafolio.site www.hgsportafolio.site admin.hgsportafolio.site docs.hgsportafolio.site health.hgsportafolio.site; do
  printf '%-28s ' "$h"; curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' "http://$h/"
done
```

## HSTS

`Strict-Transport-Security` le dice al navegador: «durante este tiempo, entra a este dominio solo por HTTPS, aunque te den un enlace `http://`». Evita que alguien en la misma red intercepte la primera petición sin cifrar.

Lo que envían hoy:

| Sitio | Valor |
| :-- | :-- |
| `hgsportafolio.site` | `max-age=31536000; includeSubDomains` |
| `admin.hgsportafolio.site` | `max-age=31536000; includeSubDomains` |
| `docs.`, `health.` (GitHub Pages) | No envían HSTS. GitHub Pages no lo permite en dominios propios. |
| `portafolio-hgs.netlify.app` | Netlify añade `preload` por su cuenta en su propio dominio. |

- `max-age=31536000` es un año.
- `includeSubDomains` extiende la regla a todos los subdominios. Como la raíz la envía, un navegador que visitó `hgsportafolio.site` también exigirá HTTPS en `docs.` y `health.`. Antes de activarlo, asegúrate de que **todos** los subdominios sirven HTTPS; si uno no lo hace, deja de abrir para esos visitantes durante un año.
- **No hay `preload`.** Agregarlo y enviar el dominio a la lista de precarga de los navegadores protege incluso la primera visita, pero es difícil de deshacer (salir de la lista tarda meses). Es una decisión, no un olvido: tómala solo si el dominio va a servir HTTPS en todos sus nombres para siempre.

## Cabeceras de seguridad

El sitio principal las declara en el `_headers` que genera el build ([capítulo 3](03-netlify-sitio-principal.md)); el segundo sitio, en su `netlify.toml` ([capítulo 4](04-netlify-segundo-sitio.md)).

| Cabecera | Para qué |
| :-- | :-- |
| `Content-Security-Policy` | Lista de orígenes permitidos para scripts, estilos, imágenes y conexiones. Con `default-src 'self'` y sin `'unsafe-inline'` en `script-src`, un script inyectado no corre. |
| `X-Frame-Options: DENY` y `frame-ancestors 'none'` | Nadie puede meter el sitio en un `iframe` (clickjacking). La primera es para navegadores viejos. |
| `X-Content-Type-Options: nosniff` | El navegador no adivina el tipo de un archivo: un `.txt` no se ejecuta como script. |
| `Referrer-Policy` | Cuánto de la dirección de origen se envía al salir del sitio. `strict-origin-when-cross-origin` manda solo el dominio a otros sitios. |
| `Permissions-Policy` | Apaga funciones del navegador que el sitio no usa (cámara, micrófono, ubicación, pagos, USB). |
| `Cross-Origin-Opener-Policy: same-origin` | Aísla la pestaña de las ventanas que abra o que la abran. |
| `X-Robots-Tag: noindex, nofollow` | Solo en el panel privado: que no aparezca en buscadores. |

```bash
curl -sI https://hgsportafolio.site/ | grep -iE "content-security|strict-transport|x-frame|x-content|referrer|permissions|cross-origin"
```

Para una revisión externa sirven `https://securityheaders.com` y el Observatory de Mozilla. El monitor `health` revisa a diario un subconjunto de estas cabeceras y publica si están o no, sin mostrar sus valores completos.

## security.txt (RFC 9116)

`https://hgsportafolio.site/.well-known/security.txt` dice a quién avisar si alguien encuentra un fallo de seguridad. Campos que usa:

| Campo | Para qué |
| :-- | :-- |
| `Contact` | Un `mailto:` de contacto. Obligatorio. |
| `Expires` | Fecha de vencimiento. Obligatorio: un archivo vencido indica que nadie lo mantiene. |
| `Preferred-Languages` | `es, en`. |
| `Canonical` | La dirección oficial del archivo. |

Va en `public/.well-known/` del sitio, así Netlify lo sirve tal cual con `Content-Type: text/plain`. Una prueba del repositorio falla un mes antes de que venza `Expires`, como recordatorio para renovarlo. El archivo hoy vence en octubre de 2027.

```bash
curl -s https://hgsportafolio.site/.well-known/security.txt
```

El archivo de la raíz dice que cubre también el subdominio del panel. Pero ese subdominio no lo sirve: por la regla de una sola página (`/*` → `/index.html`), `https://admin.hgsportafolio.site/.well-known/security.txt` responde 200 con el HTML de la aplicación. `docs.` y `health.` responden 404. Mejora sugerida: en cada subdominio, un `security.txt` propio o una redirección 301 a la canónica, puesta **antes** de la regla `/*` en el caso del panel.

```bash
for h in admin docs health; do
  printf '%-8s ' "$h"; curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "https://$h.hgsportafolio.site/.well-known/security.txt"
done
```

## CAA

Un registro `CAA` dice qué autoridades pueden emitir certificados para el dominio. Si un atacante engaña a otra autoridad, esta debe negarse.

Comprobado el 2026-10-22: **existe**.

```text
hgsportafolio.site.  CAA  0 issue "letsencrypt.org"
```

Cómo se aplica a cada nombre (RFC 8659): la autoridad busca un `CAA` en el nombre exacto; si el nombre es un `CNAME`, lo sigue; si no encuentra nada, sube al dominio padre.

| Nombre | `CAA` que cuenta | Resultado |
| :-- | :-- | :-- |
| `hgsportafolio.site` | El de la raíz | Solo Let's Encrypt |
| `www.`, `admin.` | El destino `*.netlify.app` no tiene `CAA`, así que cuenta el de la raíz | Solo Let's Encrypt |
| `docs.`, `health.` | El del destino `devcat-hgs.github.io`, que publica GitHub | Let's Encrypt, Sectigo y DigiCert |

Netlify y GitHub Pages emiten hoy con Let's Encrypt, así que todo encaja. Si un día Netlify cambia de autoridad, la renovación fallaría: agrega la nueva autoridad al `CAA` antes. No hay `issuewild` (no se usan certificados comodín, y sin él rige `issue`) ni `iodef` (un correo para avisos de solicitudes rechazadas, opcional).

```bash
curl -s "https://dns.google/resolve?name=hgsportafolio.site&type=CAA"
dig +short hgsportafolio.site CAA
```

## Certificados y su renovación

| Nombre | Emisor | Quién lo renueva |
| :-- | :-- | :-- |
| `hgsportafolio.site` y `www.` (un certificado) | Let's Encrypt | Netlify |
| `admin.` | Let's Encrypt | Netlify |
| `docs.` | Let's Encrypt | GitHub Pages |
| `health.` | Let's Encrypt | GitHub Pages |

Duran unos 90 días y se renuevan solos, siempre que:

- el DNS siga apuntando al proveedor (si cambias el `CNAME`, la renovación falla);
- el `CAA` permita a la autoridad;
- el dominio siga configurado en el sitio.

El monitor `health` revisa a diario los certificados de la raíz, `docs.` y `health.`. Para verlo a mano:

```bash
h=hgsportafolio.site
echo | openssl s_client -connect $h:443 -servername $h 2>/dev/null | openssl x509 -noout -issuer -enddate
```

## Qué nunca va en el repositorio ni en esta guía

- Claves de API, tokens, llaves privadas, contraseñas y PIN: van como variables del panel de Netlify o como secretos de GitHub Actions.
- Valores de verificación de dominio (Google, GitHub): están en el DNS, pero no hace falta copiarlos a la documentación.
- Archivos `.env`: están en `.gitignore`, y el espejo público solo copia archivos versionados.

El repositorio privado corre gitleaks en cada push y pull request sobre todo el historial, porque el escaneo de secretos de GitHub no está disponible en repositorios privados del plan gratuito.
