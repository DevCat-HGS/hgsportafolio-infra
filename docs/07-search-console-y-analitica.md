# 7. Search Console y analítica

[Volver al índice](../README.md)

## Verificar el dominio en Google Search Console

Search Console ofrece dos tipos de propiedad:

| Propiedad | Cómo se verifica | Qué cubre |
| :-- | :-- | :-- |
| **Dominio** (`hgsportafolio.site`) | Un registro `TXT` `google-site-verification=…` en la raíz del DNS | Todo: raíz, `www`, subdominios, `http` y `https`. |
| **Prefijo de URL** (`https://hgsportafolio.site/`) | Varias formas; la más simple es una meta etiqueta `<meta name="google-site-verification" content="…">` en el `<head>` | Solo esa dirección exacta. |

Aquí están las dos: hay un `TXT` de verificación en la raíz y el `<head>` de cada página lleva dos meta etiquetas de verificación. Los valores no son secretos (cualquiera los ve en el DNS o en el HTML), pero esta guía no los copia porque son propios de la cuenta.

Pasos para el `TXT`:

1. Search Console → Agregar propiedad → Dominio → `hgsportafolio.site`.
2. Copia el `TXT` que te da Google y créalo en Hostinger con nombre `@`.
3. Espera a que se propague y pulsa Verificar. Deja el registro puesto: si lo borras, la propiedad pierde la verificación.

```bash
nslookup -type=TXT hgsportafolio.site 8.8.8.8
curl -s https://hgsportafolio.site/ | grep -c 'google-site-verification'
```

## Sitemap y robots

El build genera los dos archivos. `robots.txt` permite todo y declara el sitemap:

```bash
curl -s https://hgsportafolio.site/robots.txt
# User-agent: *
# Allow: /
#
# Sitemap: https://hgsportafolio.site/sitemap.xml
```

El sitemap lista cada página en cada idioma, con sus alternativas `hreflang` y su `lastmod`. La fecha sale del último commit que tocó contenido visible, no de la fecha del build: una fecha que cambia en cada despliegue enseña al buscador a ignorarla.

Para enviarlo: Search Console → Sitemaps → escribe `sitemap.xml` → Enviar. Search Console vuelve a leerlo solo; no hace falta enviarlo en cada despliegue.

## Pedir indexación

Search Console → Inspección de URLs → pega la dirección → Solicitar indexación. Sirve para una página nueva o muy cambiada. No acelera todo el sitio, solo esa URL, y tiene un límite diario.

Antes de pedirla, comprueba que la página:

- responde 200 (`curl -sI <url>`);
- declara como canonical su propia dirección en el dominio propio;
- no lleva `noindex` (ni en `<meta name="robots">` ni en `X-Robots-Tag`).

```bash
curl -s https://hgsportafolio.site/en/ | grep -oE '<link rel="(canonical|alternate)"[^>]*>'
curl -sI https://hgsportafolio.site/en/ | grep -i x-robots-tag   # no debe salir nada
```

## Analítica sin cookies con Umami

El sitio usa Umami Cloud. No guarda cookies ni datos personales, así que no hace falta banner de consentimiento.

Cómo se conecta:

1. En Umami Cloud: Add website → `hgsportafolio.site`. Umami da un **Website ID**, un UUID.
2. En Netlify, sitio principal: Site configuration → Environment variables → `VITE_UMAMI_WEBSITE_ID` con **solo** el UUID.
3. Vuelve a desplegar. La variable se lee al construir; cambiarla sin un deploy nuevo no tiene efecto.

El Website ID no es un secreto: termina en el HTML público de cada página. Aun así, solo identifica al sitio y no da acceso al panel de Umami.

Detalles del código (en el [espejo público](https://github.com/DevCat-HGS/hgsportafolio), `portfolio/seo/headers.ts`):

- Si la variable está vacía, no se carga el script y la CSP no abre los orígenes de Umami.
- Si alguien pega en la variable el `<script>` completo que da Umami en vez del UUID, el build extrae el UUID y descarta lo demás, para no romper el `<head>`.
- La CSP permite cargar el script desde `cloud.umami.is` y enviar eventos a la pasarela de Umami.

```bash
curl -sI https://hgsportafolio.site/ | grep -i content-security-policy | grep -o 'umami[^ ;]*'
```
