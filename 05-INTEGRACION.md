# 05 --- Integración

## Objetivo

Documentar la integración entre los componentes que forman el servidor
de correo de `enoblega.com.ar`.

La configuración documentada en este archivo corresponde al entorno real
utilizado durante la implementación, pero los valores específicos de
infraestructura se presentan como variables o placeholders cuando no son
necesarios para reproducir el procedimiento.

------------------------------------------------------------------------

## 1. Arquitectura de integración

Flujo general:

``` text
                         INTERNET
                            │
             ┌──────────────┼──────────────┐
             │              │              │
            DNS           Gmail          Clientes
             │              │          Outlook / móvil
             │              │              │
             ▼              ▼              ▼
      enoblega.com.ar   SMTP :25      IMAP :993
             │          smtp.<DOMAIN>  imap.<DOMAIN>
             │              │              │
             └──────────────┼──────────────┘
                            │
                       MAIL SERVER
                            │
             ┌──────────────┼──────────────┐
             │              │              │
          Postfix        Dovecot       OpenDKIM
             │              │              │
             └──────────────┼──────────────┘
                            │
                         Maildir
```

### Componentes

  Componente   Función
  ------------ -----------------------------------------------------
  DNS          Resolución de MX, A, SPF, DKIM y DMARC
  Postfix      SMTP, Submission y transporte de correo
  Dovecot      IMAP, autenticación SASL y entrega LMTP
  OpenDKIM     Firma DKIM
  Certbot      Obtención y renovación de certificados TLS
  Nginx        Publicación del webroot ACME
  Gmail        Plataforma utilizada para validar envío y recepción

------------------------------------------------------------------------

# 2. Integración DNS

El dominio utilizado durante la implementación es:

``` text
enoblega.com.ar
```

Los DNS autoritativos son:

``` text
ns1.donweb.com
ns2.donweb.com
```

## Registros principales

### A

``` text
smtp.<DOMAIN>  A  <MAIL_SERVER_IPV4>
imap.<DOMAIN>  A  <MAIL_SERVER_IPV4>
```

En el entorno implementado ambos hostnames apuntan a la IPv4 pública del
VPS.

### MX

``` text
<DOMAIN>.  MX  10 smtp.<DOMAIN>.
```

El registro MX determina el servidor responsable de recibir correo para
el dominio.

El MX debe apuntar a un hostname que disponga de un registro A/AAAA
válido.

### SPF

Ejemplo utilizado:

``` text
<DOMAIN>. TXT "v=spf1 ip4:<MAIL_SERVER_IPV4> -all"
```

Este registro autoriza a la IPv4 del servidor a enviar correo en nombre
del dominio.

### DKIM

Selector utilizado:

``` text
mail
```

Registro:

``` text
mail._domainkey.<DOMAIN>
```

La clave pública publicada corresponde a la clave privada utilizada por
OpenDKIM.

### DMARC

Registro:

``` text
_dmarc.<DOMAIN>
```

Política inicial:

``` text
v=DMARC1; p=none
```

Se utilizó `p=none` durante la etapa inicial de validación.

------------------------------------------------------------------------

# 3. Integración Postfix ↔ Dovecot

Postfix utiliza Dovecot para autenticar usuarios SMTP mediante SASL.

Configuración relevante de Postfix:

``` text
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
```

Dovecot publica el socket de autenticación dentro del entorno de
Postfix:

``` text
/var/spool/postfix/private/auth
```

El flujo es:

``` text
Cliente
   │
   │ SMTP Submission :587
   ▼
Postfix
   │
   │ SASL
   ▼
Dovecot
   │
   └── autenticación
```

Durante las pruebas se obtuvo:

``` text
235 2.7.0 Authentication successful
```

Esto confirmó que la autenticación SMTP mediante Dovecot estaba
funcionando correctamente.

------------------------------------------------------------------------

# 4. Integración Postfix ↔ Dovecot LMTP

La entrega de correo local utiliza LMTP.

Socket:

``` text
/var/spool/postfix/private/dovecot-lmtp
```

Flujo:

``` text
Postfix
   │
   │ LMTP
   ▼
Dovecot
   │
   ▼
Maildir
```

La cuenta utilizada durante las pruebas fue:

``` text
contacto@enoblega.com.ar
```

El Maildir se encuentra bajo:

``` text
/var/vmail/<DOMAIN>/contacto/Maildir
```

Durante las pruebas se obtuvo una respuesta LMTP equivalente a:

``` text
250 2.0.0 ... Saved
```

Esto confirmó que Dovecot aceptaba correctamente la entrega local.

------------------------------------------------------------------------

# 5. Integración Postfix ↔ OpenDKIM

Postfix utiliza OpenDKIM como Milter.

Configuración relevante:

``` text
milter_default_action = accept
milter_protocol = 6
smtpd_milters = unix:/opendkim/opendkim.sock
non_smtpd_milters = unix:/opendkim/opendkim.sock
```

El socket se encuentra dentro del entorno de Postfix:

``` text
/var/spool/postfix/opendkim/opendkim.sock
```

Esta ubicación es importante porque Postfix utiliza chroot para
determinados servicios.

## OpenDKIM

Dominio:

``` text
<DOMAIN>
```

Selector:

``` text
mail
```

KeyTable:

``` text
mail._domainkey.<DOMAIN> <DOMAIN>:mail:/etc/opendkim/keys/<DOMAIN>/mail.private
```

SigningTable:

``` text
*@<DOMAIN> mail._domainkey.<DOMAIN>
```

La clave privada debe permanecer fuera del repositorio y con permisos
restrictivos.

### Validación

La clave se validó con:

``` bash
opendkim-testkey \
  -d <DOMAIN> \
  -s mail \
  -k /etc/opendkim/keys/<DOMAIN>/mail.private \
  -vvv
```

Resultado obtenido:

``` text
key not secure
key OK
```

`key OK` confirmó que la clave privada local correspondía con la clave
pública publicada.

El mensaje:

``` text
key not secure
```

no indicó un problema con la clave DKIM; corresponde a la ausencia de
validación DNSSEC en esa comprobación.

------------------------------------------------------------------------

# 6. Integración TLS / Let's Encrypt

Los certificados TLS son administrados mediante Certbot.

## SMTP

Hostname:

``` text
smtp.<DOMAIN>
```

Certificado:

``` text
/etc/letsencrypt/live/smtp.<DOMAIN>/fullchain.pem
```

Clave:

``` text
/etc/letsencrypt/live/smtp.<DOMAIN>/privkey.pem
```

## IMAP

Hostname:

``` text
imap.<DOMAIN>
```

Certificado:

``` text
/etc/letsencrypt/live/imap.<DOMAIN>/fullchain.pem
```

Clave:

``` text
/etc/letsencrypt/live/imap.<DOMAIN>/privkey.pem
```

## Validación SMTP

Se validó mediante:

``` bash
swaks \
  --server smtp.<DOMAIN> \
  --port 587 \
  --tls \
  --quit-after EHLO
```

La prueba confirmó:

-   TLS 1.3.
-   Certificado correspondiente al hostname.
-   Validación de CA correcta.
-   Validación del hostname correcta.
-   STARTTLS operativo.

## Renovación

Comprobar periódicamente:

``` bash
certbot renew --dry-run
```

Después de una renovación real debe verificarse que Postfix y Dovecot
hayan recargado los nuevos certificados.

------------------------------------------------------------------------

# 7. Integración Nginx / Docker / ACME

Nginx funciona como reverse proxy dentro de Docker.

El webroot utilizado para los desafíos ACME es:

``` text
/root/PROXY/certbot
```

El endpoint publicado es:

``` text
/.well-known/acme-challenge/
```

## Validación del webroot

Se creó temporalmente un archivo:

``` text
/root/PROXY/certbot/.well-known/acme-challenge/test-smtp.txt
```

y se verificó externamente mediante:

``` bash
curl http://smtp.<DOMAIN>/.well-known/acme-challenge/test-smtp.txt
```

El contenido esperado fue:

``` text
certbot-test-smtp
```

Esto confirmó que el webroot utilizado por Certbot era accesible desde
Internet.

------------------------------------------------------------------------

# 8. Integración con Gmail

Gmail fue utilizado como plataforma externa para validar el envío.

La prueba exitosa utilizó:

``` text
From: contacto@enoblega.com.ar
To: <TEST_GMAIL_ADDRESS>
```

El envío se realizó mediante:

``` text
smtp.<DOMAIN>:587
STARTTLS
SASL PLAIN
```

La autenticación devolvió:

``` text
235 2.7.0 Authentication successful
```

Postfix entregó posteriormente el mensaje a Gmail con:

``` text
dsn=2.0.0
status=sent
```

Gmail verificó correctamente SPF y DKIM.

## Error encontrado durante una prueba inicial

Una prueba anterior utilizó como remitente una identidad asociada al
hostname del VPS en lugar del dominio de correo.

Gmail rechazó ese mensaje porque la identidad utilizada no estaba
autenticada para el dominio de envío.

La prueba posterior utilizando:

``` text
contacto@enoblega.com.ar
```

funcionó correctamente.

### Recomendación

Las pruebas de autenticación deben utilizar siempre una dirección del
dominio que está siendo firmado y autorizado por SPF/DKIM.

------------------------------------------------------------------------

# 9. Integración para recepción

El flujo esperado es:

``` text
Remitente externo
       │
       ▼
      DNS
       │
       │ MX
       ▼
smtp.<DOMAIN>
       │
       │ TCP/25
       ▼
    Postfix
       │
       │ LMTP
       ▼
    Dovecot
       │
       ▼
    Maildir
```

La aceptación del destinatario se probó localmente con:

``` bash
swaks \
  --server smtp.<DOMAIN> \
  --port 25 \
  --quit-after RCPT \
  --from <EXTERNAL_TEST_ADDRESS> \
  --to contacto@<DOMAIN>
```

Postfix respondió:

``` text
250 2.1.0 Ok
250 2.1.5 Ok
```

Esto confirmó que el servidor acepta el dominio y el destinatario.

Como se utilizó:

``` text
--quit-after RCPT
```

la prueba terminó antes de enviar el contenido del mensaje. Por lo
tanto, que Postfix cancele la transacción posteriormente es
comportamiento esperado.

------------------------------------------------------------------------

# 10. Incidencia de propagación / caché DNS

Durante la validación de recepción se observó una diferencia temporal
entre resolvers públicos.

Los DNS autoritativos de DonWeb devolvían correctamente:

``` text
10 smtp.<DOMAIN>.
```

Cloudflare Public DNS (`1.1.1.1`) también llegó a devolver el MX
correctamente.

Google Public DNS (`8.8.8.8`) mantuvo temporalmente una respuesta
negativa para el MX.

Esto explicaba por qué Gmail no generaba conexiones entrantes visibles
en Postfix durante la prueba de recepción.

El servidor no requería modificaciones mientras los servidores
autoritativos continuaran devolviendo correctamente el MX.

## Verificación

``` bash
dig @ns1.donweb.com MX <DOMAIN> +short
dig @ns2.donweb.com MX <DOMAIN> +short

dig @8.8.8.8 MX <DOMAIN> +short
dig @1.1.1.1 MX <DOMAIN> +short
```

El estado esperado es:

``` text
10 smtp.<DOMAIN>.
```

en todos los resolvers consultados.

------------------------------------------------------------------------

# 11. Configuración de clientes

## Entrada --- IMAP

``` text
Servidor: imap.<DOMAIN>
Puerto: 993
Seguridad: SSL/TLS
Usuario: contacto@<DOMAIN>
```

## Salida --- SMTP

``` text
Servidor: smtp.<DOMAIN>
Puerto: 587
Seguridad: STARTTLS
Autenticación: contraseña
Usuario: contacto@<DOMAIN>
```

El puerto 25 no debe utilizarse como puerto de envío desde clientes
finales.

El puerto 25 queda destinado principalmente al transporte SMTP entre
servidores.

------------------------------------------------------------------------

# 12. Estado de integración

  Integración              Estado
  ------------------------ -------------------------------
  DNS autoritativo         OK
  Registro A SMTP          OK
  Registro A IMAP          OK
  MX                       Publicado
  SPF                      OK
  DKIM                     OK
  DMARC                    Publicado
  Postfix SMTP 25          OK
  Postfix Submission 587   OK
  SASL Dovecot             OK
  Dovecot LMTP             OK
  IMAPS 993                OK
  TLS SMTP                 OK
  TLS IMAP                 OK
  Envío hacia Gmail        Validado
  Recepción externa        Pendiente de validación final
  Cliente IMAP externo     Pendiente

------------------------------------------------------------------------

# 13. Validación final

Una vez que el MX sea visible de forma consistente en los resolvers
públicos:

### 1. Verificar MX

``` bash
dig @8.8.8.8 MX <DOMAIN> +short
dig @1.1.1.1 MX <DOMAIN> +short
```

### 2. Monitorear Postfix

``` bash
journalctl -u postfix -f
```

### 3. Enviar desde una cuenta externa

``` text
<EXTERNAL_TEST_ADDRESS>
        ↓
contacto@<DOMAIN>
```

### 4. Verificar Maildir

``` bash
find /var/vmail/<DOMAIN>/contacto/Maildir/new -type f -ls
```

### 5. Probar IMAP

Conectar mediante:

``` text
imap.<DOMAIN>:993
```

### 6. Responder desde el servidor

Enviar desde:

``` text
contacto@<DOMAIN>
```

hacia una cuenta externa.

### 7. Verificar autenticación

En Gmail u otro proveedor compatible:

``` text
SPF: PASS
DKIM: PASS
DMARC: PASS
```

### 8. Verificar renovación

``` bash
certbot renew --dry-run
```

------------------------------------------------------------------------

# 14. Seguridad y mantenimiento

## Credenciales

Las contraseñas utilizadas durante pruebas no deben almacenarse en este
repositorio.

Si una contraseña real fue expuesta durante una prueba, debe cambiarse
antes de considerar la cuenta operativa.

## Clave DKIM

No publicar:

``` text
/etc/opendkim/keys/<DOMAIN>/mail.private
```

## Backups

Definir un esquema de backup para:

``` text
/etc/postfix/
/etc/dovecot/
/etc/opendkim/
/etc/letsencrypt/
/var/vmail/
```

Las claves privadas y los Maildir deben almacenarse en un backup
protegido.

