# 10. Comprobación final y solución de problemas

[Volver al índice](../README.md)

## Lista de comprobación

Marca cada punto con su comando. Si usas Windows sin `dig`, cambia `dig +short NOMBRE TIPO` por `nslookup -type=TIPO NOMBRE 8.8.8.8`.

### DNS

- [ ] Los nameservers son los del registrador: `dig +short hgsportafolio.site NS`
- [ ] La raíz apunta a Netlify: `dig +short hgsportafolio.site A` → la IP que indica Netlify
- [ ] Cada subdominio apunta a su servicio: `dig +short www.hgsportafolio.site CNAME` (y `admin`, `docs`, `health`)
- [ ] No hay `CNAME` colgantes: cada subdominio responde con tu contenido (ver abajo)
- [ ] Existe el `CAA` y permite a la autoridad de tus proveedores: `dig +short hgsportafolio.site CAA`
- [ ] El dominio está verificado en GitHub (Settings → Pages → Verified domains) y su `TXT` sigue en la zona

### Sitios

- [ ] Cada nombre responde 200 por HTTPS:

  ```bash
  for u in https://hgsportafolio.site/ https://hgsportafolio.site/en/ https://admin.hgsportafolio.site/ https://docs.hgsportafolio.site/ https://health.hgsportafolio.site/; do
    printf '%-40s ' "$u"; curl -s -o /dev/null -w '%{http_code}\n' "$u"
  done
  ```

- [ ] HTTP redirige a HTTPS y `www` a la raíz: `curl -sI http://hgsportafolio.site/` y `curl -sI https://www.hgsportafolio.site/`
- [ ] El canonical de la copia `netlify.app` apunta al dominio propio: `curl -s https://portafolio-hgs.netlify.app/ | grep -o '<link rel="canonical"[^>]*>'`
- [ ] Una ruta inexistente da 404 (no 200): `curl -s -o /dev/null -w '%{http_code}\n' https://hgsportafolio.site/no-existe`
- [ ] El inglés redirige según el navegador: `curl -sI -H "Accept-Language: en" https://hgsportafolio.site/`

### Seguridad

- [ ] Cabeceras presentes: `curl -sI https://hgsportafolio.site/ | grep -iE "content-security|strict-transport|x-frame|x-content"`
- [ ] El panel no se indexa: `curl -sI https://admin.hgsportafolio.site/ | grep -i x-robots-tag`
- [ ] `security.txt` vigente: `curl -s https://hgsportafolio.site/.well-known/security.txt | grep Expires`
- [ ] Certificados con más de 30 días por delante: comando de `openssl` del [capítulo 9](09-seguridad-del-hosting.md)
- [ ] Enforce HTTPS activo en los dos sitios de Pages: `gh api repos/USUARIO/REPO/pages --jq .https_enforced`

### Correo

- [ ] SPF, DKIM y DMARC publicados: comandos del [capítulo 6](06-correo-resend.md)
- [ ] El dominio sale como verificado en el panel de Resend
- [ ] Un correo de prueba a Gmail muestra `SPF: PASS`, `DKIM: PASS`, `DMARC: PASS`

### Buscadores y monitoreo

- [ ] Propiedad verificada en Search Console y sitemap enviado
- [ ] `robots.txt` apunta al sitemap: `curl -s https://hgsportafolio.site/robots.txt`
- [ ] El monitor corrió en la última hora: pestaña Actions de `DevCat-HGS/health`
- [ ] No hay issues de incidente abiertos sin explicar

## Solución de problemas

### 404 de GitHub Pages en un dominio propio

Síntoma: «There isn't a GitHub Pages site here» o un 404 con `Server: GitHub.com`.

Causa más común: el DNS apunta a GitHub, pero ningún repositorio tiene ese dominio configurado. Pasa cuando un push borró el archivo `CNAME` de un sitio publicado desde una rama, o cuando se cambió la fuente de publicación y se perdió el ajuste.

```bash
curl -sI https://docs.hgsportafolio.site/ | grep -iE "^HTTP|^server"
gh api repos/DevCat-HGS/DevCat-HGS.github.io/pages --jq .cname     # debe dar el dominio, no null
gh api repos/DevCat-HGS/DevCat-HGS.github.io/contents/CNAME --jq .content | base64 -d
```

Arreglo: vuelve a poner el dominio en Settings → Pages → Custom domain y, si el sitio se publica desde una rama, haz que el workflow escriba el `CNAME` en cada publicación ([capítulo 5](05-github-pages.md)). Mientras dure, el subdominio es tomable: verifica el dominio en la cuenta.

### Certificado pendiente

Síntoma: el navegador avisa de certificado no válido, Netlify dice «Waiting on DNS propagation» o en GitHub la casilla Enforce HTTPS sigue gris.

Causas: el DNS todavía no apunta al proveedor, hay un `CAA` que no permite a Let's Encrypt, o hay registros que chocan (por ejemplo, un `AAAA` viejo en la raíz que apunta a otro servidor).

```bash
dig +short hgsportafolio.site A
dig +short hgsportafolio.site AAAA      # no debe apuntar a otro proveedor
dig +short hgsportafolio.site CAA       # debe incluir letsencrypt.org o no existir
echo | openssl s_client -connect hgsportafolio.site:443 -servername hgsportafolio.site 2>/dev/null | openssl x509 -noout -subject -issuer -enddate
```

Arreglo: corrige el DNS, espera el TTL y vuelve a pedir el certificado (Netlify: Domain management → HTTPS → Renew certificate o Verify DNS configuration; GitHub: quita y vuelve a poner el dominio si en una hora no avanza).

### «Domain already in use» o «already managed» en Netlify

Síntoma: Netlify no deja agregar el dominio porque otro sitio o cuenta ya lo tiene.

Causas: el dominio sigue asignado a un sitio viejo de tu cuenta (o de otro equipo), o quedó una zona de Netlify DNS creada para él.

Comprobación: en Netlify, busca el dominio en todos tus sitios (Domain management de cada uno) y en Domains de la cuenta o del equipo.

Arreglo: quítalo del sitio viejo o borra la zona de Netlify DNS que no uses. Si está en una cuenta ajena, Netlify permite reclamarlo demostrando que el dominio es tuyo con un `TXT` (sigue las instrucciones que da su panel o su soporte).

### Propagación lenta

Síntoma: tú ves el valor nuevo y otros el viejo, o al revés.

```bash
dig +noall +answer @8.8.8.8 docs.hgsportafolio.site                 # resolvedor de Google, con TTL restante
dig +noall +answer @1.1.1.1 docs.hgsportafolio.site                 # Cloudflare
dig +noall +answer @orbit.dns-parking.com docs.hgsportafolio.site   # la fuente
```

Si la fuente ya tiene el valor nuevo, solo queda esperar el TTL. Para acelerar el próximo cambio, baja el TTL a 300 un día antes. Vacía también la caché local:

```bash
ipconfig /flushdns          # Windows
sudo resolvectl flush-caches   # Linux con systemd-resolved
```

En Chrome, `chrome://net-internals/#dns` → Clear host cache.

### Contenido mixto (mixed content)

Síntoma: la página carga por HTTPS pero falta una imagen, un script o una fuente, y la consola del navegador dice «Mixed Content».

Causa: algún recurso se pide con `http://`. Los navegadores lo bloquean en páginas HTTPS. Con `default-src 'self'` en la CSP, un recurso de otro origen también se bloquea, aunque sea HTTPS, si no está permitido.

```bash
curl -s https://hgsportafolio.site/ | grep -o 'http://[^"'"'"' )]*' | sort -u
```

Arreglo: cambia los enlaces a `https://` o a rutas relativas. Si es un servicio externo legítimo, agrega su origen a la directiva correspondiente de la CSP. Como red de seguridad, la directiva `upgrade-insecure-requests` hace que el navegador pida por HTTPS lo que diga `http://`.

### El sitio de respaldo aparece en Google

Comprueba que el canonical de la copia `netlify.app` apunte al dominio propio (comando de la lista de arriba). Si apunta a sí misma, el build no tenía la dirección del sitio: revisa que `VITE_SITE_URL` esté en `netlify.toml`.

### El monitor dice caído pero el sitio abre

El monitor exige un 200 **y** un texto concreto en el HTML. Si cambiaste el texto de la página, actualiza el texto esperado en `scripts/targets.mjs` del repositorio `health`. Revisa también el registro de la corrida en la pestaña Actions.
