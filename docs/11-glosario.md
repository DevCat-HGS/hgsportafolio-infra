# 11. Glosario

[Volver al índice](../README.md)

| Término | Qué es |
| :-- | :-- |
| **A** | Registro DNS que asocia un nombre a una dirección IPv4. |
| **AAAA** | Igual que `A`, para IPv6. |
| **ALIAS / ANAME** | Registro de algunos proveedores de DNS que se comporta como un `CNAME` pero puede ir en la raíz: el proveedor resuelve el destino y responde con sus IP. No es un tipo estándar. |
| **Apex (raíz)** | El dominio sin subdominio: `hgsportafolio.site`. En los paneles se escribe `@`. No puede llevar `CNAME`. |
| **CAA** | Registro que dice qué autoridades pueden emitir certificados para el dominio. |
| **Canonical** | Etiqueta `<link rel="canonical">` que dice al buscador cuál es la dirección oficial de una página cuando hay copias. |
| **CDN** | Red de servidores repartidos por el mundo que sirven copias del sitio desde el más cercano. |
| **CNAME** | Registro que dice «este nombre es un alias de este otro». No convive con otros registros en el mismo nombre. |
| **CNAME colgante** | Un `CNAME` hacia un servicio donde ya no hay un sitio que reclame ese nombre. Permite la toma de subdominio. |
| **CSP** | Content Security Policy: cabecera que limita de dónde puede cargar la página scripts, estilos, imágenes y conexiones. |
| **Deploy key** | Llave SSH asociada a un solo repositorio de GitHub, con permiso de solo lectura o de escritura. |
| **Deploy preview** | Copia del sitio que Netlify construye para un pull request, en una dirección propia. |
| **DKIM** | Firma criptográfica de cada correo. La llave pública va en un `TXT` de `<selector>._domainkey`. |
| **DMARC** | Política publicada en `_dmarc` que dice qué hacer con correos que fallan SPF y DKIM alineados, y a dónde mandar informes. |
| **HSTS** | Strict-Transport-Security: cabecera que obliga al navegador a usar solo HTTPS en el dominio durante un tiempo. |
| **hreflang** | Etiqueta que indica las versiones de una página en otros idiomas. |
| **Let's Encrypt** | Autoridad de certificación gratuita y automática. Sus certificados duran unos 90 días. |
| **MX** | Registro que dice qué servidor recibe el correo de un nombre, con una prioridad. |
| **Nameserver (NS)** | Servidor que tiene la zona DNS del dominio y responde por ella. |
| **Null MX** | `MX` de prioridad 0 con destino `.` (RFC 7505): declara que el dominio no recibe correo. |
| **Propagación** | El tiempo que tardan los resolvedores en olvidar el valor viejo de un registro. Depende del TTL. |
| **Resolvedor** | Servidor que consulta el DNS en nombre de tu equipo y guarda las respuestas (por ejemplo, `8.8.8.8`). |
| **Return-Path** | Remitente del «sobre» de un correo, a donde vuelven los rebotes. Aquí, `send.hgsportafolio.site`. |
| **security.txt** | Archivo en `/.well-known/security.txt` (RFC 9116) con el contacto para reportar fallos de seguridad. |
| **Sitio de proyecto** | Sitio de GitHub Pages de cualquier repositorio; sin dominio propio vive en `<usuario>.github.io/<repo>/`. |
| **Sitio de usuario** | Sitio de GitHub Pages del repositorio `<usuario>.github.io`; uno por cuenta. |
| **SOA** | Registro de inicio de autoridad de la zona: servidor primario, número de serie y tiempos, incluido el TTL de las respuestas negativas. |
| **SPF** | `TXT` que lista los servidores autorizados a enviar correo con un dominio como remitente del sobre. |
| **Toma de subdominio** | Cuando un tercero sirve contenido en tu subdominio aprovechando un `CNAME` colgante. |
| **TTL** | Segundos que un resolvedor puede guardar una respuesta antes de volver a preguntar. |
| **TXT** | Registro de texto libre. Se usa para SPF, DKIM, DMARC y verificaciones de propiedad. |
| **Verified domain (GitHub)** | Dominio que tu cuenta demostró poseer con un `TXT`. Solo tu cuenta puede usarlo en GitHub Pages. |
