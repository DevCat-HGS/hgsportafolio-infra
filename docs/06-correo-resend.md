# 6. Correo con Resend

[Volver al índice](../README.md)

El sitio **envía** correos transaccionales (por ejemplo, el aviso de un mensaje del formulario de contacto) con Resend, con remitente en `hgsportafolio.site`. **No recibe** correo en el dominio. Este capítulo cubre solo el DNS que hace falta para que esos correos lleguen a la bandeja de entrada y no a spam.

## Verificar el dominio en Resend

1. En Resend: Domains → Add domain → `hgsportafolio.site`. Elige la región que te convenga; los registros dependen de ella.
2. Resend muestra una lista de registros DNS. Créalos en Hostinger **tal cual** los muestra, con el nombre relativo (`send`, `resend._domainkey`), no el completo.
3. Pulsa Verify. Cuando todos estén en verde, el dominio puede enviar.

La clave de la API de Resend es un secreto: se guarda como variable de entorno del sitio que envía (en el panel de Netlify) y nunca en el repositorio ni en el navegador.

## Qué hace cada registro

| Registro | Nombre | Qué hace |
| :-- | :-- | :-- |
| `MX` (de retorno) | `send` | Recibe los rebotes y las quejas del correo saliente. Es el dominio del «Return-Path» (el remitente del sobre), no el de la dirección que ve el destinatario. |
| `TXT` SPF | `send` | Lista los servidores autorizados a enviar correo con `send.hgsportafolio.site` como remitente del sobre. Termina en `~all`: lo que no esté en la lista se marca como sospechoso. |
| `TXT` DKIM | `resend._domainkey` | Llave pública. Resend firma cada correo con la llave privada; el destinatario comprueba la firma con esta. Demuestra que el correo salió de tu cuenta y no se alteró. |
| `TXT` DMARC | `_dmarc` | Política del dominio: qué hacer con un correo que dice venir de `hgsportafolio.site` y no pasa SPF ni DKIM alineados, y a dónde mandar informes. |

### Cómo está montado aquí

En esta zona, `send` no tiene un `MX` y un `TXT` propios sino un **`CNAME`** hacia un host del proveedor, que es el que publica el `MX` (prioridad 10) y el `TXT` SPF. El resultado para quien consulta es el mismo, y el proveedor puede cambiar sus servidores sin que haya que tocar la zona. Consecuencia: `send` no puede tener ningún otro registro, porque un `CNAME` no convive con nada.

Que esos sean exactamente los registros que pide Resend hoy para esta cuenta está por confirmar en el panel de Resend: la documentación de Resend suele indicar un `MX` y un `TXT` directos en `send`. Si tu panel muestra esos, usa esos.

```bash
nslookup -type=CNAME send.hgsportafolio.site               8.8.8.8
nslookup -type=MX    send.hgsportafolio.site               8.8.8.8
nslookup -type=TXT   send.hgsportafolio.site               8.8.8.8   # debe empezar por v=spf1
nslookup -type=TXT   resend._domainkey.hgsportafolio.site  8.8.8.8   # debe empezar por p=
nslookup -type=TXT   _dmarc.hgsportafolio.site             8.8.8.8   # debe empezar por v=DMARC1
```

### Alineación: por qué SPF en `send` sirve para un remitente en la raíz

DMARC exige que al menos uno de los dos controles pase **y** esté alineado con el dominio del `From:`:

- **DKIM:** la firma lleva `d=hgsportafolio.site` (la llave está en `resend._domainkey.hgsportafolio.site`), así que coincide con el `From:`.
- **SPF:** se comprueba sobre `send.hgsportafolio.site`. Con alineación relajada (la de defecto), un subdominio del mismo dominio cuenta como alineado.

### DMARC

La política actual es `p=none` con informes agregados (`rua`) hacia un buzón propio. `p=none` no bloquea nada: solo pide informes. Es el punto de partida correcto. Cuando los informes muestren durante unas semanas que todo el correo legítimo pasa, el siguiente paso es `p=quarantine` y después `p=reject`, que impiden que otros envíen correo falso con tu dominio.

Ejemplo genérico (cambia el buzón por uno tuyo):

```text
v=DMARC1; p=none; rua=mailto:informes-dmarc@tu-dominio.com
```

## Por qué la raíz no tiene MX

El dominio no tiene buzones: nadie escribe a `algo@hgsportafolio.site` esperando respuesta, y los avisos del sitio llegan a un buzón externo. Un `MX` en la raíz solo tendría sentido con un servicio de correo entrante contratado.

Dos detalles si replicas esto:

- **Sin `MX`, algunos servidores intentan entregar en la IP del `A`** de la raíz (el «MX implícito» del estándar SMTP). Aquí esa IP es Netlify, que no acepta correo, y el envío falla tras reintentos. Para decirlo de forma explícita existe el «null MX» (RFC 7505): un `MX` de prioridad 0 con destino `.`. Hoy no está en la zona; es una mejora opcional.
- **El `MX` de `send` no es para recibir correo de personas.** Solo recibe rebotes del proveedor. No lo uses como buzón.

Si un día quieres recibir correo en el dominio (por ejemplo, con el buzón de Hostinger o con Google Workspace), agrega los `MX` que te dé ese servicio en la raíz y revisa el SPF de la raíz con él: cada servicio que envíe como `@hgsportafolio.site` debe estar incluido.

## Comprobar un envío real

Envía un correo de prueba a una cuenta de Gmail, ábrelo y usa «Mostrar original». Debe decir `SPF: PASS`, `DKIM: PASS` y `DMARC: PASS`.
