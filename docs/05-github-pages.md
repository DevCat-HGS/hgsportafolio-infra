# 5. GitHub Pages

[Volver al índice](../README.md)

Hay dos sitios en GitHub Pages, publicados de dos maneras distintas. La diferencia importa por un motivo concreto: **dónde se guarda el dominio propio**.

| | Documentación | Monitor |
| :-- | :-- | :-- |
| Dominio | `docs.hgsportafolio.site` | `health.hgsportafolio.site` |
| Tipo de sitio | Sitio de usuario | Sitio de proyecto |
| Repositorio | `DevCat-HGS/DevCat-HGS.github.io` | `DevCat-HGS/health` |
| Cómo se publica | Desde una rama (`main`, carpeta raíz) | Con GitHub Actions (`build_type: workflow`) |
| Quién escribe los archivos | Un workflow de otro repositorio hace push a la rama | El workflow `deploy.yml` del propio repo sube un artefacto |
| Dónde vive el dominio | Archivo `CNAME` en la raíz de la rama | Ajuste del repositorio (Settings → Pages) |
| `CNAME` en DNS | `DevCat-HGS.github.io` | `devcat-hgs.github.io` |
| HTTPS forzado | Sí | Sí |

Ambos datos se pueden leer con la API pública:

```bash
gh api repos/DevCat-HGS/DevCat-HGS.github.io/pages --jq '{cname, build_type, source, https_enforced, cert: .https_certificate.state}'
gh api repos/DevCat-HGS/health/pages --jq '{cname, build_type, https_enforced, cert: .https_certificate.state}'
```

## Sitio de usuario vs. sitio de proyecto

- **Sitio de usuario:** el repositorio se llama `<usuario>.github.io` y se sirve en `https://<usuario>.github.io/`. Solo hay uno por cuenta.
- **Sitio de proyecto:** cualquier otro repositorio con Pages activado. Sin dominio propio se sirve en `https://<usuario>.github.io/<repo>/`.

Cuando el sitio de usuario tiene un dominio propio, GitHub redirige ahí su dirección `github.io`. Los sitios de proyecto con dominio propio se redirigen al suyo:

```bash
curl -sI https://devcat-hgs.github.io/          # 301 -> https://docs.hgsportafolio.site/
curl -sI https://devcat-hgs.github.io/health/   # 301 -> https://health.hgsportafolio.site/
```

Por eso los dos `CNAME` del DNS apuntan al mismo host (`<usuario>.github.io`): GitHub decide qué sitio servir según el nombre que pide el navegador y el dominio que cada repositorio reclama.

## (a) Sitio de usuario publicado desde una rama

La documentación se genera en el repositorio privado y un workflow la copia a la rama `main` de `DevCat-HGS.github.io`. En el repositorio destino, Settings → Pages → Build and deployment → Source: **Deploy from a branch**, rama `main`, carpeta `/ (root)`.

Lo que hace el workflow en cada publicación:

```bash
# Vacía todo menos .git
find site -mindepth 1 -maxdepth 1 ! -name .git -exec rm -rf {} +
# Genera la edición pública
node Doc/build.mjs --public site
# HTML estático: que GitHub no lo pase por Jekyll
touch site/.nojekyll
# El dominio propio, que el rm de arriba acaba de borrar
echo "docs.hgsportafolio.site" > site/CNAME
```

Y luego commit y push, solo si algo cambió. El detalle de las llaves está en el [capítulo 8](08-publicacion-y-automatizacion.md).

### Por qué el archivo CNAME

En un sitio publicado desde una rama, GitHub guarda el dominio propio **como un archivo `CNAME`** en la raíz de esa rama. Cuando lo escribes en Settings → Pages → Custom domain, GitHub hace un commit con ese archivo. Si un push posterior lo borra, el sitio pierde el dominio.

`.nojekyll` hace que GitHub sirva los archivos tal cual, sin procesarlos con Jekyll (que, por ejemplo, ignora carpetas que empiezan con `_`).

## (b) Sitio de proyecto publicado con Actions

En `DevCat-HGS/health`, Settings → Pages → Source: **GitHub Actions**. El workflow `deploy.yml` arma la carpeta y la publica con las acciones oficiales:

```yaml
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    concurrency:
      group: pages
      cancel-in-progress: false
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: |
          mkdir -p _site
          cp -r site/. _site/
          cp -r data _site/data
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site
      - id: deployment
        uses: actions/deploy-pages@v4
```

Con esta forma de publicar, el dominio propio es **un ajuste del repositorio**, no un archivo. Ningún archivo del build puede borrarlo, y un `CNAME` dentro del artefacto se ignora. Se pone una vez en Settings → Pages → Custom domain.

`id-token: write` permite que `deploy-pages` se identifique ante GitHub Pages sin ningún secreto. `cancel-in-progress: false` evita cortar a la mitad un despliegue en curso.

## Poner el dominio propio

1. **Verifica el dominio en tu cuenta** (recomendado, antes de todo): en GitHub, Settings de tu cuenta → Pages → Verified domains → Add a domain. GitHub te da un `TXT` con nombre `_github-pages-challenge-<usuario>.<dominio>`; créalo en Hostinger y pulsa Verify. Deja el `TXT` puesto. Con el dominio verificado, ningún otro usuario de GitHub puede usar tus subdominios en Pages aunque tengas un `CNAME` colgante. Hoy este `TXT` no existe en la zona: por hacer.
2. **Configura el dominio en el repositorio:** Settings → Pages → Custom domain → `docs.hgsportafolio.site` → Save. En un sitio de rama, además, asegúrate de que el workflow escriba el archivo `CNAME` (si no, la próxima publicación lo borra).
3. **Crea el `CNAME` en el DNS** hacia `<usuario>.github.io` (sin el nombre del repositorio). Para la raíz de un dominio, GitHub pide registros `A` hacia sus IP en vez de un `CNAME`; aquí no aplica porque la raíz está en Netlify.
4. **Espera la comprobación de DNS.** GitHub muestra «DNS check successful» y pide el certificado a Let's Encrypt. Suele tardar de minutos a una hora.
5. **Activa Enforce HTTPS** cuando la casilla deje de estar gris. Solo se puede marcar cuando el certificado ya está emitido. Con ella activa, `http://` responde 301 hacia `https://`.

```bash
curl -sI http://docs.hgsportafolio.site/     # 301 -> https://docs.hgsportafolio.site/
gh api repos/DevCat-HGS/DevCat-HGS.github.io/pages --jq '.https_certificate'
```

El 2026-10-22 los dos certificados estaban en estado `approved`, emitidos por Let's Encrypt, cada uno solo para su subdominio. GitHub los renueva solo mientras el DNS siga apuntando a GitHub.

GitHub Pages no envía `Strict-Transport-Security` en los dominios propios. Cubre a estos subdominios el HSTS con `includeSubDomains` que envía la raíz desde Netlify, pero solo en navegadores que ya visitaron la raíz (ver [capítulo 9](09-seguridad-del-hosting.md)).

## El incidente del 404 en docs

**Qué pasó.** `docs.hgsportafolio.site` respondió con el 404 de GitHub («There isn't a GitHub Pages site here»). El DNS estaba bien: el `CNAME` apuntaba a `DevCat-HGS.github.io`. El problema era que **ningún repositorio reclamaba ese dominio**. El workflow de publicación vaciaba la raíz del sitio en cada edición, y con ella se iba el archivo `CNAME` que guardaba el dominio. GitHub recibía las peticiones para `docs.` y no sabía qué sitio servir.

**El riesgo de fondo.** Ese estado es justo el de un subdominio colgante: el DNS apunta a GitHub Pages y ningún repositorio tiene el dominio. Mientras dure, otra cuenta podría configurar `docs.hgsportafolio.site` en su propio repositorio y servir contenido ahí.

**Cómo se arregló.** El workflow escribe el `CNAME` en cada publicación, después de vaciar la raíz (commit `fix(docs): serve the documentation at docs.hgsportafolio.site`, 2026-10-22).

**Cómo se evita en general:**

- Si un workflow reemplaza toda la raíz de una rama de Pages, que escriba el `CNAME` (y `.nojekyll`) al final, siempre.
- Mejor aún, para sitios nuevos: publicar con Actions, como `health`. El dominio queda en el ajuste del repositorio y ningún build puede borrarlo. El comentario de `deploy.yml` lo dice así: «the docs site once lost its domain that way».
- Verificar el dominio en la cuenta (paso 1 de arriba), para que un descuido así no permita tomar el subdominio.
- Vigilarlo: el monitor `health` comprueba `docs.` cada 30 minutos y exige que el HTML traiga un texto esperado, así un 404 servido por GitHub cuenta como caída.
