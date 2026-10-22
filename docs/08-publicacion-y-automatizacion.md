# 8. Publicación y automatización entre repositorios

[Volver al índice](../README.md)

El código fuente vive en un repositorio privado. Dos cosas salen de ahí hacia repositorios públicos: la documentación y un espejo del código del sitio. Un tercer repositorio público, `health`, mide el sitio desde fuera.

## Llaves de despliegue: una por repositorio

Un workflow necesita permiso para hacer push a **otro** repositorio. El `GITHUB_TOKEN` de Actions solo sirve para el repositorio donde corre. Las opciones son un token personal o una llave de despliegue (deploy key). Aquí se usan llaves de despliegue porque:

- cada llave da acceso a **un solo repositorio**; si se filtra, el daño se limita a ese destino;
- no depende de una persona ni de sus permisos sobre toda la cuenta;
- no caduca sola, a diferencia de los tokens de grano fino.

Se usa una llave distinta para cada destino.

### Cómo crearla

```bash
# Un par de llaves nuevo, sin frase de paso, solo para este uso
ssh-keygen -t ed25519 -N "" -C "publicar-docs" -f ./publicar-docs
```

1. **Llave pública** (`publicar-docs.pub`) en el repositorio **destino**: Settings → Deploy keys → Add deploy key, y marca **Allow write access**.
2. **Llave privada** (`publicar-docs`) en el repositorio **origen**: Settings → Secrets and variables → Actions → New repository secret.
3. Borra los dos archivos de tu equipo. Si hace falta otra vez, se genera un par nuevo.

### Cómo la usa el workflow

El patrón de los dos workflows de publicación:

```yaml
permissions:
  contents: read            # el token del origen solo lee

concurrency:
  group: publicar-docs      # una publicación a la vez
  cancel-in-progress: false # no cortar una a la mitad

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Clonar el destino con su llave
        env:
          DEPLOY_KEY: ${{ secrets.NOMBRE_DEL_SECRETO }}
        run: |
          mkdir -p ~/.ssh
          printf '%s\n' "$DEPLOY_KEY" > ~/.ssh/key
          chmod 600 ~/.ssh/key
          ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts
          echo "GIT_SSH_COMMAND=ssh -i ~/.ssh/key -o IdentitiesOnly=yes" >> "$GITHUB_ENV"
          GIT_SSH_COMMAND="ssh -i ~/.ssh/key -o IdentitiesOnly=yes" \
            git clone --depth 1 "git@github.com:USUARIO/DESTINO.git" destino

      - name: Reemplazar el contenido
        run: |
          find destino -mindepth 1 -maxdepth 1 ! -name .git -exec rm -rf {} +
          # ... generar o copiar los archivos nuevos en destino/ ...

      - name: Commit y push solo si cambió algo
        working-directory: destino
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add -A
          git diff --cached --quiet && exit 0
          git commit -m "Edición desde ${GITHUB_SHA::7}"
          git push origin HEAD:main
```

Puntos que importan:

- **`find ... ! -name .git`** vacía todo menos el historial. Así se borran también los archivos que ya no existen en el origen. El costo: borra el `CNAME` de un sitio de Pages, así que hay que reescribirlo (ver [capítulo 5](05-github-pages.md)).
- **`IdentitiesOnly=yes`** obliga a ssh a usar solo esa llave.
- **Sin cambios, sin commit.** El destino no se llena de commits vacíos.
- **`ssh-keyscan`** confía en la llave de host que reciba en ese momento. Una mejora es fijar en el workflow las llaves de host que GitHub publica en su documentación, en vez de pedirlas en cada corrida.

Los nombres de los secretos de este repositorio no se publican; en tu caso, usa uno por destino.

## Documentación: publicar en un sitio de GitHub Pages

El workflow corre en cada push a `main` que toque la carpeta de la documentación (o a mano). Genera una **edición pública** de los documentos, que deja fuera las partes marcadas como privadas, y la escribe en la raíz de `DevCat-HGS/DevCat-HGS.github.io`, más `.nojekyll` y el `CNAME`. GitHub Pages publica esa rama sola. El historial del repositorio destino conserva las ediciones anteriores.

## Espejo público del código

`DevCat-HGS/hgsportafolio` es una copia de solo lectura del código del sitio. Sigue el mismo patrón, con tres protecciones más:

1. **Lista blanca, no lista negra.** Un script nombra lo que se copia (la carpeta del sitio, `.gitattributes`, el README del espejo y seis workflows del sitio). Todo lo demás queda privado, incluido lo que se agregue al repositorio en el futuro. Con una lista negra, olvidar una línea publicaría algo.
2. **Solo archivos versionados.** La copia sale de `git ls-files`, así nada ignorado (`node_modules`, `dist`, `.env`) puede colarse.
3. **Última barrera.** Antes de publicar, el script busca rutas privadas en la copia; si encuentra una, falla y no se publica nada.

Otros detalles:

- **Historial limpio.** Cada corrida añade un solo commit al espejo, con el asunto del commit de origen y su hash corto en el cuerpo. El historial privado no se arrastra.
- **Actions apagadas en el destino.** Los workflows del espejo están para leerse; corren en el origen. Se apagan en Settings → Actions → General → Disable actions. Así un workflow copiado no corre dos veces ni pide secretos que el espejo no tiene.

```bash
gh api repos/DevCat-HGS/hgsportafolio/actions/permissions   # {"enabled":false,...}
```

## El monitor `health`

`DevCat-HGS/health` es público y mide el sitio desde fuera. No usa llaves ni secretos propios: escribe en su propio repositorio con el `GITHUB_TOKEN` del workflow y publica con el despliegue de Pages desde Actions.

| Workflow | Cuándo | Qué hace |
| :-- | :-- | :-- |
| `uptime.yml` | `cron: "7,37 * * * *"` (cada 30 minutos, a los minutos 7 y 37) | Pide cada página de una lista fija en el código. Una página cuenta como arriba solo si responde 200 **y** su HTML trae un texto esperado, así un error servido con 200 también es una caída. Un fallo se reintenta a los 10 segundos antes de contar. Guarda la muestra en `data/`, abre o cierra issues de incidente y vuelve a publicar la página. |
| `daily.yml` | `cron: "23 7 * * *"` (07:23 UTC, una vez al día) | Lighthouse en móvil y escritorio contra producción, y controles de seguridad que cualquiera puede hacer desde fuera (cabeceras, redirección a HTTPS, `security.txt`, certificados). Guarda los informes HTML como artefacto 90 días. |
| `deploy.yml` | Push que toque la página, a mano, o llamado por los otros dos (`workflow_call`) | Arma la página con sus datos y la publica en GitHub Pages. |

Por qué los minutos 7, 37 y 23: GitHub recomienda evitar el inicio de la hora, cuando más tareas programadas arrancan. Aun así, GitHub puede empezar una tarea programada con minutos de retraso; la página lo advierte.

### Incidentes como issues

Cuando una página cae, el monitor abre un issue con la etiqueta `incidente`; cuando vuelve a responder, lo cierra. El historial de incidentes queda público:

```text
https://github.com/DevCat-HGS/health/issues?q=label%3Aincidente
```

Para eso el workflow declara `issues: write` en sus permisos. Sin token (en local), el script mide pero no abre issues.

### Commits de datos sin choques

Las dos mediciones escriben en `data/`, en archivos distintos. Antes de empujar hacen `git pull --rebase` y reintentan hasta tres veces, porque la otra pudo empujar mientras tanto:

```bash
for i in 1 2 3; do git pull -q --rebase origin main && git push -q origin HEAD:main && exit 0; sleep 5; done
exit 1
```

### Concurrencia por job, y el bloqueo que la motivó

GitHub permite, por cada grupo de concurrencia, **un** trabajo corriendo y **uno** en espera; si llega otro, el que esperaba se cancela.

Al principio el grupo estaba a nivel de **workflow**: una medición tomaba el turno y lo retenía durante toda la corrida, despliegue incluido. El despliegue quedaba en cola mientras la siguiente medición esperaba el mismo turno, y la corrida terminaba esperándose a sí misma.

La corrección (commit `fix(ci): take the data turn per job so a run never waits on itself`, 2026-10-22) movió la concurrencia a los **jobs**:

```yaml
jobs:
  check:
    concurrency:          # solo el job que escribe data/ toma este turno
      group: data
      cancel-in-progress: false
    # ...

  deploy:
    needs: check
    uses: ./.github/workflows/deploy.yml   # deploy.yml tiene su propio grupo "pages", también en el job
```

La regla que queda: **el turno lo toma solo el job que necesita exclusividad**, y un workflow llamado con `workflow_call` declara su concurrencia en el job, porque así se aplica también cuando otro workflow lo llama.

### Qué no publica

La lista de páginas medidas está fija en el código (nadie puede pedirle que mida otra dirección), y la página no muestra versiones de software ni valores completos de cabeceras. El panel privado no se monitorea.

GitHub desactiva los workflows programados de un repositorio público tras 60 días sin actividad. Aquí los commits de datos mantienen el repositorio activo; si un día los workflows aparecen desactivados, se reactivan en la pestaña Actions.
