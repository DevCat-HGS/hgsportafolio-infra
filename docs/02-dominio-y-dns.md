# 2. Dominio y DNS en Hostinger

[Volver al índice](../README.md)

## Dónde se edita

El dominio está registrado en Hostinger y la zona DNS también vive ahí. Los nameservers son los de Hostinger:

```text
orbit.dns-parking.com
horizon.dns-parking.com
```

Los registros se editan en el panel de Hostinger (hPanel), en la sección de dominios: se elige `hgsportafolio.site` y se abre el editor de la zona DNS. El nombre exacto de cada menú puede cambiar con las versiones del panel.

Si algún día se mueven los nameservers a otro proveedor (por ejemplo, Netlify DNS), la zona de Hostinger deja de contar: hay que recrear **todos** los registros de esta página en el proveedor nuevo antes del cambio, o el correo y los subdominios se caen.

Compruébalo:

```bash
nslookup -type=NS hgsportafolio.site 8.8.8.8
nslookup -type=SOA hgsportafolio.site 8.8.8.8   # el SOA dice qué servidor es el primario
```

## La zona completa

Valores comprobados el 2026-10-22. El TTL es el que devuelve el servidor autoritativo (`orbit.dns-parking.com`), en segundos.

| Tipo | Nombre | Destino | TTL | Para qué sirve |
| :-- | :-- | :-- | :-- | :-- |
| `NS` | `@` | `orbit.dns-parking.com`, `horizon.dns-parking.com` | | Delegan la zona en Hostinger. Los fija el registrador. |
| `A` | `@` | `75.2.60.5` | 60 | La raíz apunta al balanceador de carga de Netlify. Es la IP que Netlify indica para dominios raíz con DNS externo. |
| `CNAME` | `www` | `portafolio-hgs.netlify.app` | 300 | Netlify sirve `www` y lo redirige con 301 a la raíz. |
| `CNAME` | `admin` | `admin-hgs.netlify.app` | 350 | Segundo sitio de Netlify (panel privado). |
| `CNAME` | `docs` | `DevCat-HGS.github.io` | 14400 | Sitio de usuario de GitHub Pages (documentación). |
| `CNAME` | `health` | `devcat-hgs.github.io` | 14400 | Sitio de proyecto de GitHub Pages (monitor). Mayúsculas y minúsculas dan igual en DNS. |
| `CNAME` | `send` | Un host del proveedor de correo | 3600 | Subdominio de retorno de Resend. El destino publica el `MX` de rebotes y el `TXT` SPF. Ver [capítulo 6](06-correo-resend.md). |
| `TXT` | `resend._domainkey` | Llave pública DKIM | 355 | Firma de los correos salientes. |
| `TXT` | `_dmarc` | Política DMARC | 14400 | Qué hacer con correos que fallen SPF y DKIM, y a dónde mandar informes. |
| `TXT` | `@` | `google-site-verification=…` | 400 | Prueba ante Google Search Console de que el dominio es tuyo. El valor no se publica aquí. |
| `CAA` | `@` | `0 issue "letsencrypt.org"` | 14400 | Solo Let's Encrypt puede emitir certificados para el dominio. Ver [capítulo 9](09-seguridad-del-hosting.md). |

No hay:

- `AAAA` en la raíz (no se publica IPv6 para la raíz);
- `MX` en la raíz (el dominio no recibe correo; ver [capítulo 6](06-correo-resend.md));
- comodín (`*`): un nombre inventado devuelve `NXDOMAIN`.

### Sobre el registro A de la raíz

Un `CNAME` no puede ir en la raíz (`@`) porque la raíz ya tiene `NS`, `SOA` y otros registros, y un `CNAME` no puede convivir con nada. Por eso la raíz usa un `A` hacia la IP del balanceador de Netlify. Algunos proveedores ofrecen `ALIAS` o `ANAME` (un `CNAME` aplanado que sí puede ir en la raíz); con esos se apuntaría a `portafolio-hgs.netlify.app`. Aquí no hace falta.

Antes de copiar `75.2.60.5`, confírmalo en la documentación de Netlify o en el panel del sitio (Domain management): es el valor que Netlify indica hoy, no algo que tú controles.

## Pasos para montarlo desde cero

1. **Primero el hosting, luego el DNS.** Agrega el dominio en Netlify o en GitHub Pages antes de crear el `CNAME`. Si el `CNAME` existe y el servicio no tiene un sitio con ese dominio, el subdominio queda colgante (ver más abajo).
2. **Crea los registros** en el editor de zona de Hostinger con los valores de la tabla. En el campo «nombre» va solo la parte izquierda (`www`, `docs`), no el dominio completo; `@` es la raíz.
3. **Borra registros por defecto que choquen.** Al comprar un dominio, el registrador suele crear un `A` o un `CNAME` de `www` hacia su página de estacionamiento. Un nombre no puede tener a la vez `CNAME` y otro registro.
4. **Comprueba** cada registro con los comandos de abajo antes de pedir el certificado.

## TTL y propagación

El TTL dice cuántos segundos puede guardar un resolvedor una respuesta antes de volver a preguntar. Un cambio «se propaga» cuando caducan las copias guardadas con el valor viejo.

- Con TTL de 300, un cambio se ve en todo el mundo en unos 5 minutos. Con 14400, puede tardar hasta 4 horas.
- Antes de un cambio importante (mudar el sitio, cambiar de proveedor), baja el TTL a 300 uno o dos días antes. Cuando todo funcione, súbelo otra vez.
- Las respuestas negativas también se guardan: si consultaste un nombre antes de crearlo, el «no existe» queda en caché según el TTL mínimo del `SOA` (aquí, 600 segundos).
- Para ver el valor real sin cachés intermedias, pregunta al servidor autoritativo: `nslookup -type=A hgsportafolio.site orbit.dns-parking.com`.

## Cómo comprobar cada registro

Con `nslookup` (Windows, macOS, Linux):

```bash
nslookup -type=A     hgsportafolio.site                    8.8.8.8
nslookup -type=AAAA  hgsportafolio.site                    8.8.8.8
nslookup -type=CNAME www.hgsportafolio.site                8.8.8.8
nslookup -type=CNAME admin.hgsportafolio.site              8.8.8.8
nslookup -type=CNAME docs.hgsportafolio.site               8.8.8.8
nslookup -type=CNAME health.hgsportafolio.site             8.8.8.8
nslookup -type=MX    hgsportafolio.site                    8.8.8.8
nslookup -type=MX    send.hgsportafolio.site               8.8.8.8
nslookup -type=TXT   send.hgsportafolio.site               8.8.8.8
nslookup -type=TXT   resend._domainkey.hgsportafolio.site  8.8.8.8
nslookup -type=TXT   _dmarc.hgsportafolio.site             8.8.8.8
nslookup -type=TXT   hgsportafolio.site                    8.8.8.8
```

Con `dig` (Linux, macOS, WSL), que además muestra el TTL:

```bash
dig +noall +answer @8.8.8.8 hgsportafolio.site A
dig +noall +answer @8.8.8.8 www.hgsportafolio.site CNAME
dig +noall +answer @8.8.8.8 hgsportafolio.site CAA
dig +noall +answer @orbit.dns-parking.com hgsportafolio.site A   # servidor autoritativo
```

El `nslookup` de Windows no conoce el tipo `CAA`. Para ese y para ver el TTL sin `dig`, sirve la API DNS sobre HTTPS de Google:

```bash
curl -s "https://dns.google/resolve?name=hgsportafolio.site&type=CAA"
curl -s "https://dns.google/resolve?name=docs.hgsportafolio.site&type=CNAME"
```

## Registros que conviene NO tener

### CNAME colgantes (toma de subdominio)

Un `CNAME` que apunta a un servicio donde **no** hay un sitio que reclame ese nombre es una invitación: cualquiera que cree un sitio en ese servicio y le ponga tu subdominio puede servir contenido en `algo.tudominio.com`, con certificado válido.

Casos típicos:

- `docs.` apunta a `usuario.github.io` y ningún repositorio tiene `docs.tudominio.com` configurado. Aquí pasó (ver [capítulo 5](05-github-pages.md)): el sitio dio 404 hasta que se repuso el dominio.
- Se borra un sitio de Netlify y su `CNAME` queda en la zona.
- Se prueba un servicio (un blog, una tienda), se abandona y el registro se olvida.

Cómo evitarlo:

1. Cuando des de baja un sitio, borra primero el registro DNS y después el sitio.
2. Revisa la zona cada cierto tiempo: cada `CNAME` debe responder con tu contenido.
3. En GitHub, verifica el dominio (Settings → Pages → Verified domains). Con el dominio verificado, solo tu cuenta puede usarlo en GitHub Pages. Hoy no existe el registro `TXT` de verificación (`_github-pages-challenge-…`) en la zona, así que el dominio no está verificado: es la mejora pendiente más útil de esta lista.

Comprobación rápida de todos los subdominios:

```bash
for h in www admin docs health; do
  printf '%s  ' "$h"; curl -s -o /dev/null -w '%{http_code}\n' "https://$h.hgsportafolio.site/"
done
```

Un 404 de `Server: GitHub.com` o una página de «Site not found» de Netlify en un subdominio tuyo es una alarma.

### Otros

- **Comodines (`*`)** que apunten a un servicio externo: cualquier nombre inventado caería ahí.
- **Varios registros SPF** en el mismo nombre: SPF exige uno solo; dos lo invalidan.
- **`MX` hacia servidores que ya no usas.**
- **`A` de la raíz hacia la página de estacionamiento** del registrador junto al de Netlify: los resolvedores alternarían entre las dos IP.
