# 05 --- Integración

## Objetivo

Documentar la integración del servidor de correo de `enoblega.com.ar`
con los servicios y componentes externos necesarios para su operación.

El esquema implementado es:

``` text
                         INTERNET
                            │
             ┌──────────────┼──────────────┐
             │              │              │
           DNS            Gmail          Clientes
             │              │          Outlook / móvil
             │              │              │
             ▼              ▼              ▼
      enoblega.com.ar   SMTP :25      IMAP :993
             │          smtp.enoblega   imap.enoblega
             │              │              │
             └──────────────┼──────────────┘
                            │
                    66.97.38.104
                            │
                     ┌──────▼──────┐
                     │    VPS      │
                     │   Debian    │
                     └──────┬──────┘
                            │
             ┌──────────────┼──────────────┐
             │              │              │
          Postfix        Dovecot       OpenDKIM
             │              │              │
             └──────────────┼──────────────┘
                            │
                         Maildir
```

------------------------------------------------------------------------

# 1. Integración DNS

El dominio utilizado es:

``` text
enoblega.com.ar
```

Los DNS autoritativos son:

``` text
ns1.donweb.com
ns2.donweb.com
```

### Registros principales

``` text
smtp.enoblega.com.ar.  A    66.97.38.104
imap.enoblega.com.ar.  A    66.97.38.104
```

Registro MX:

``` text
enoblega.com.ar.  MX  10 smtp.enoblega.com.ar.
```

El MX determina que el servidor responsable de recibir correo para
`enoblega.com.ar` es `smtp.enoblega.com.ar`.

### SPF

``` text
enoblega.com.ar. TXT "v=spf1 ip4:66.97.38.104 -all"
```

Esto autoriza únicamente a `66.97.38.104` a enviar correo en nombre del
dominio.

### DKIM

Selector:

``` text
mail
```

Registro:

``` text
mail._domainkey.enoblega.com.ar
```

La clave pública publicada corresponde a la clave privada utilizada por
OpenDKIM.

### DMARC

Registro:

``` text
_dmarc.enoblega.com.ar
```

Política inicial:

``` text
v=DMARC1; p=none
```

Se dejó `p=none` como etapa inicial de monitoreo.

------------------------------------------------------------------------

# 2. Integración Postfix ↔ Dovecot

Postfix utiliza Dovecot para autenticar usuarios SMTP mediante SASL.

En Postfix:

``` text
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
```

Dovecot publica el socket de autenticación:

``` text
/var/spool/postfix/private/auth
```

El usuario:

``` text
contacto@enoblega.com.ar
```

fue probado exitosamente mediante SMTP Submission.

Resultado observado:

``` text
235 2.7.0 Authentication successful
```

Por lo tanto:

``` text
Cliente
  ↓
SMTP 587
  ↓
Postfix
  ↓
Dovecot SASL
  ↓
autenticación exitosa
```

------------------------------------------------------------------------

# 3. Integración Postfix ↔ Dovecot LMTP

La entrega de correo local utiliza LMTP.

Socket:

``` text
/var/spool/postfix/private/dovecot-lmtp
```

El flujo es:

``` text
Postfix
   ↓
LMTP
   ↓
Dovecot
   ↓
/var/vmail/enoblega.com.ar/contacto/Maildir
```

Durante las pruebas se obtuvo:

``` text
status=sent
250 2.0.0 ... Saved
```

Esto confirma que Dovecot acepta correctamente la entrega local.

------------------------------------------------------------------------

# 4. Integración Postfix ↔ OpenDKIM

Postfix utiliza OpenDKIM como Milter.

Configuración:

``` text
milter_default_action = accept
milter_protocol = 6
smtpd_milters = unix:/opendkim/opendkim.sock
non_smtpd_milters = unix:/opendkim/opendkim.sock
```

El socket real está ubicado dentro del entorno chroot de Postfix:

``` text
/var/spool/postfix/opendkim/opendkim.sock
```

OpenDKIM utiliza:

``` text
Domain:  enoblega.com.ar
Selector: mail
```

KeyTable:

``` text
mail._domainkey.enoblega.com.ar enoblega.com.ar:mail:/etc/opendkim/keys/enoblega.com.ar/mail.private
```

SigningTable:

``` text
*@enoblega.com.ar mail._domainkey.enoblega.com.ar
```

La clave fue validada mediante:

``` bash
opendkim-testkey \
  -d enoblega.com.ar \
  -s mail \
  -k /etc/opendkim/keys/enoblega.com.ar/mail.private \
  -vvv
```

Resultado:

``` text
key not secure
key OK
```

`key OK` confirma que la clave privada local coincide con la clave
pública publicada.

`key not secure` se debe a que la validación no está respaldada por
DNSSEC y no indica un problema de DKIM.

------------------------------------------------------------------------

# 5. Integración TLS / Let's Encrypt

Los certificados son administrados mediante Certbot.

## SMTP

Hostname:

``` text
smtp.enoblega.com.ar
```

Certificado:

``` text
/etc/letsencrypt/live/smtp.enoblega.com.ar/fullchain.pem
```

Clave:

``` text
/etc/letsencrypt/live/smtp.enoblega.com.ar/privkey.pem
```

## IMAP

Hostname:

``` text
imap.enoblega.com.ar
```

Certificado:

``` text
/etc/letsencrypt/live/imap.enoblega.com.ar/fullchain.pem
```

Clave:

``` text
/etc/letsencrypt/live/imap.enoblega.com.ar/privkey.pem
```

## Renovación

Certbot informó que configuró la renovación automática.

Debe comprobarse periódicamente:

``` bash
certbot renew --dry-run
```

Después de una renovación real se debe verificar que Postfix y Dovecot
hayan recargado los nuevos certificados.

------------------------------------------------------------------------

# 6. Integración con Nginx / Docker para ACME

Nginx funciona en el contenedor:

``` text
reverse-proxy
```

El webroot utilizado por Certbot es:

``` text
/root/PROXY/certbot
```

El challenge ACME se publica mediante:

``` text
/.well-known/acme-challenge/
```

La validación se probó creando:

``` text
/root/PROXY/certbot/.well-known/acme-challenge/test-smtp.txt
```

y comprobando:

``` bash
curl http://smtp.enoblega.com.ar/.well-known/acme-challenge/test-smtp.txt
```

Resultado:

``` text
certbot-test-smtp
```

Esto confirma que Let's Encrypt puede utilizar el webroot para validar
los hostnames.

------------------------------------------------------------------------

# 7. Integración con Gmail

Gmail fue utilizado como destino de las pruebas de envío.

La prueba exitosa utilizó:

``` text
From: contacto@enoblega.com.ar
To: noblegaesteban@gmail.com
```

El envío se realizó mediante:

``` text
smtp.enoblega.com.ar:587
STARTTLS
SASL PLAIN
```

La autenticación fue exitosa:

``` text
235 2.7.0 Authentication successful
```

Postfix entregó el mensaje a Gmail con:

``` text
dsn=2.0.0
status=sent
```

Gmail reportó autenticación positiva para SPF y DKIM.

## Error inicial

Una prueba anterior utilizó como remitente:

``` text
root@vps-5798471-x.dattaweb.com
```

Gmail rechazó ese mensaje porque la identidad del remitente no estaba
autenticada por SPF/DKIM para `enoblega.com.ar`.

La prueba posterior utilizando:

``` text
contacto@enoblega.com.ar
```

funcionó correctamente.

------------------------------------------------------------------------

# 8. Integración para recepción

El flujo esperado es:

``` text
noblegaesteban@gmail.com
          │
          ▼
       Gmail
          │
       DNS MX
          │
          ▼
smtp.enoblega.com.ar
          │
       TCP/25
          │
          ▼
       Postfix
          │
        LMTP
          │
          ▼
       Dovecot
          │
          ▼
/var/vmail/enoblega.com.ar/contacto/Maildir
```

Postfix fue probado localmente mediante:

``` bash
swaks \
  --server smtp.enoblega.com.ar \
  --port 25 \
  --quit-after RCPT \
  --from noblegaesteban@gmail.com \
  --to contacto@enoblega.com.ar
```

Postfix respondió:

``` text
250 2.1.0 Ok
250 2.1.5 Ok
```

Esto confirma que el servidor acepta el dominio y el destinatario.

La prueba utilizó `--quit-after RCPT`, por lo que no se entregó un
mensaje real y Postfix posteriormente canceló la transacción. Esto es
comportamiento esperado.

------------------------------------------------------------------------

# 9. Estado de la integración externa

Al momento de documentar esta etapa:

  Integración                Estado
  -------------------------- ---------------------------
  DNS autoritativo           OK
  A `smtp.enoblega.com.ar`   OK
  A `imap.enoblega.com.ar`   OK
  MX                         Publicado
  SPF                        OK
  DKIM                       OK
  DMARC                      Publicado
  Postfix SMTP 25            OK
  Postfix Submission 587     OK
  SASL Dovecot               OK
  Dovecot LMTP               OK
  IMAPS 993                  OK
  TLS SMTP                   OK
  TLS IMAP                   OK
  Gmail → servidor           Pendiente de prueba final
  Cliente IMAP externo       Pendiente

------------------------------------------------------------------------

# 10. Incidencia de propagación DNS

Durante la validación se observó una diferencia entre resolvers
públicos.

Los DNS autoritativos de DonWeb devolvían:

``` text
10 smtp.enoblega.com.ar.
```

Cloudflare Public DNS (`1.1.1.1`) también llegó a devolver correctamente
el MX.

Google Public DNS (`8.8.8.8`), sin embargo, mantuvo temporalmente una
respuesta negativa para el MX.

Esto explica que el correo enviado desde Gmail no generara ninguna
conexión entrante visible en Postfix durante la prueba.

El servidor no debe modificarse por este motivo mientras los DNS
autoritativos continúen devolviendo correctamente el MX.

Verificación:

``` bash
dig @ns1.donweb.com MX enoblega.com.ar +short
dig @ns2.donweb.com MX enoblega.com.ar +short

dig @8.8.8.8 MX enoblega.com.ar +short
dig @1.1.1.1 MX enoblega.com.ar +short
```

El objetivo es que todos terminen devolviendo:

``` text
10 smtp.enoblega.com.ar.
```

Una vez que los resolvers converjan, repetir:

``` text
noblegaesteban@gmail.com
        ↓
contacto@enoblega.com.ar
```

y observar:

``` bash
journalctl -u postfix -f
```

------------------------------------------------------------------------

# 11. Configuración de clientes

## Entrada --- IMAP

``` text
Servidor: imap.enoblega.com.ar
Puerto: 993
Seguridad: SSL/TLS
Usuario: contacto@enoblega.com.ar
```

## Salida --- SMTP

``` text
Servidor: smtp.enoblega.com.ar
Puerto: 587
Seguridad: STARTTLS
Autenticación: contraseña
Usuario: contacto@enoblega.com.ar
```

No utilizar puerto 25 como puerto de envío desde clientes finales.

El puerto 25 queda destinado principalmente al transporte SMTP entre
servidores.

------------------------------------------------------------------------

# 12. Próximas validaciones

Una vez propagado el MX:

1.  Enviar desde `noblegaesteban@gmail.com` a
    `contacto@enoblega.com.ar`.
2.  Ejecutar en el VPS:

``` bash
journalctl -u postfix -f
```

3.  Confirmar conexión entrante desde infraestructura de Google.
4.  Confirmar entrega LMTP a Dovecot.
5.  Confirmar archivo nuevo en:

``` text
/var/vmail/enoblega.com.ar/contacto/Maildir/new/
```

6.  Configurar un cliente IMAP.
7.  Verificar recepción desde IMAP.
8.  Responder desde `contacto@enoblega.com.ar`.
9.  Verificar nuevamente SPF/DKIM/DMARC en Gmail.
10. Ejecutar:

``` bash
certbot renew --dry-run
```

11. Configurar backups y monitoreo.
12. Cambiar la contraseña utilizada durante las pruebas antes de
    considerar la cuenta lista para producción.
