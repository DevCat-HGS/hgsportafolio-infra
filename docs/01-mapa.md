# 1. Mapa de la infraestructura

[Volver al índice](../README.md)

## El dibujo

```text
                         Hostinger (registrador + DNS)
                 nameservers: orbit.dns-parking.com, horizon.dns-parking.com
                                        |
     +------------------+---------------+---------------+------------------+----------------+
     |                  |               |               |                  |                |
 hgsportafolio.site   www.            admin.          docs.             health.          send.
   A 75.2.60.5     CNAME           CNAME            CNAME             CNAME            CNAME
     |             portafolio-hgs  admin-hgs        DevCat-HGS        devcat-hgs       (Resend)
     |             .netlify.app    .netlify.app     .github.io        .github.io          |
     v                  |               |               |                  |              MX + SPF
 +---------------------------+   +-------------+   +--------------------------------+   de retorno
 | Netlify: portafolio-hgs   |   | Netlify:    |   | GitHub Pages                    |
 | build desde portfolio/    |   | admin-hgs   |   |  - sitio de usuario (docs.)     |
 | www -> 301 -> raíz        |   | base: admin |   |    rama main de                 |
 | respaldo:                 |   |             |   |    DevCat-HGS.github.io         |
 | portafolio-hgs.netlify.app|   |             |   |  - sitio de proyecto (health.)  |
 +---------------------------+   +-------------+   |    repo DevCat-HGS/health,      |
               ^                        ^          |    desplegado con Actions       |
               |                        |          +--------------------------------+
               +----- un mismo repo ----+                     ^
                 (rama main, privado)                         |
                        |                                     |
                        +--- workflows publican en repos públicos (llaves de despliegue)
```

## Quién hace qué

| Pieza | Proveedor | Por qué ahí |
| :-- | :-- | :-- |
| Registro del dominio y zona DNS | Hostinger | Es donde se compró el dominio. La zona se edita en su panel; no hace falta mover los nameservers a otro proveedor. |
| Sitio principal | Netlify (`portafolio-hgs`) | Construye desde Git en cada push, sirve con CDN, da HTTPS gratis, permite cabeceras y redirecciones declaradas en el repo y publica un preview por pull request. |
| Panel privado | Netlify (`admin-hgs`) | Sitio aparte desde el mismo repositorio, con su propia política de cabeceras y sus propios despliegues. Ver [capítulo 4](04-netlify-segundo-sitio.md). |
| Documentación | GitHub Pages, sitio de usuario | Es HTML estático que ya se genera en GitHub Actions; no necesita build en el hosting. |
| Monitor `health` | GitHub Pages, sitio de proyecto | Tiene que seguir arriba cuando el sitio cae. Si viviera en Netlify, una caída de Netlify se llevaría también la página que la reporta. |
| Correo saliente | Resend | API de envío con verificación de dominio (SPF, DKIM) y un subdominio propio para rebotes. |
| Certificados | Let's Encrypt, emitidos por Netlify y por GitHub Pages | Ambos proveedores los piden y los renuevan solos. |
| Analítica | Umami Cloud | Sin cookies, así que no hace falta banner de consentimiento. |
| Indexación | Google Search Console | Verificación del dominio, sitemap y solicitudes de indexación. |

## Repositorios

| Repositorio | Visibilidad | Papel |
| :-- | :-- | :-- |
| Repositorio del sitio | Privado | Fuente única: portafolio, panel, documentación y workflows. Netlify construye los dos sitios desde su rama `main`. |
| `DevCat-HGS/DevCat-HGS.github.io` | Público | Destino de la documentación. Un workflow del repo privado reemplaza su contenido en cada publicación. |
| `DevCat-HGS/hgsportafolio` | Público | Espejo de solo lectura del código del sitio. Tiene GitHub Actions apagadas. |
| `DevCat-HGS/health` | Público | Monitor: mide, guarda los datos y publica su propia página en GitHub Pages. |

## Cómo fluye un cambio

1. Se hace push a `main` del repositorio privado.
2. Netlify recibe el aviso de GitHub y construye `portafolio-hgs`. El sitio del panel solo construye si cambió su carpeta ([capítulo 4](04-netlify-segundo-sitio.md)).
3. En paralelo, workflows de GitHub Actions publican la documentación en `DevCat-HGS.github.io` y el código público en el espejo, cada uno con su propia llave de despliegue ([capítulo 8](08-publicacion-y-automatizacion.md)).
4. Cada 30 minutos, el monitor `health` comprueba que las páginas respondan y abre un issue si alguna cae.

## Por qué el monitor va separado

El monitor mide la portada, los casos de estudio, la documentación y su propia página. Corre en GitHub Actions y se publica en GitHub Pages, que no comparte infraestructura con Netlify. Así:

- una caída de Netlify o un build roto del sitio no apagan el monitor;
- los datos en bruto y los registros de cada medición quedan en un repositorio público que cualquiera puede revisar;
- el monitor se vigila a sí mismo: un 404 en su propio dominio cuenta como incidente.

El límite es que GitHub Pages y GitHub Actions son el mismo proveedor: una caída de GitHub apaga el monitor y la documentación a la vez.
