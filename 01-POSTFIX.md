# 01 - Postfix

## Objetivo

Documentar la configuración de Postfix utilizada como MTA del servidor de correo de `enoblega.com.ar`.

> **Repositorio público:** este documento no contiene contraseñas, claves privadas, tokens ni contenido de archivos sensibles.

## Rol de Postfix

Postfix se encarga de:
- recibir correo SMTP desde Internet;
- recibir mensajes mediante SMTP Submission;
- entregar correo local mediante Dovecot LMTP;
- entregar correo saliente a servidores SMTP externos;
- integrar OpenDKIM;
- aplicar las restricciones de relay.

## Dominio y mailbox

```text
enoblega.com.ar
contacto@enoblega.com.ar
```

## Puertos

| Puerto | Servicio | Uso |
|---:|---|---|
| 25/TCP | SMTP | Recepción servidor-a-servidor y entrega saliente |
| 587/TCP | SMTP Submission | Clientes autenticados |
| 993/TCP | IMAPS | Acceso de clientes |

## SMTP Submission

El servicio `submission` de `/etc/postfix/master.cf` utiliza:

```text
-o smtpd_tls_security_level=encrypt
-o smtpd_sasl_auth_enable=yes
-o smtpd_sasl_type=dovecot
-o smtpd_sasl_path=private/auth
-o smtpd_sasl_security_options=noanonymous
-o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

En 587:
1. STARTTLS es obligatorio.
2. El cliente se autentica contra Dovecot.
3. Solamente usuarios autenticados pueden enviar.

## TLS

Postfix utiliza el certificado de:

```text
/etc/letsencrypt/live/smtp.enoblega.com.ar/fullchain.pem
```

y la clave privada correspondiente:

```text
/etc/letsencrypt/live/smtp.enoblega.com.ar/privkey.pem
```

Las claves privadas nunca deben subirse al repositorio.

Para SMTP general, `smtpd_tls_security_level = may` permite STARTTLS cuando el servidor remoto lo soporta. Submission fuerza TLS mediante `encrypt`.

## SASL

```text
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
```

## Restricción de relay

Configuración validada:

```text
smtpd_relay_restrictions =
    permit_mynetworks
    permit_sasl_authenticated
    defer_unauth_destination
```

Un cliente no autenticado no puede utilizar el servidor como relay hacia dominios externos.

### Prueba de Open Relay

Se realizó una prueba sin autenticación intentando entregar hacia un dominio externo.

Resultado:

```text
454 4.7.1 ... Relay access denied
```

**Open Relay: NO.**

## Integración con OpenDKIM

```text
smtpd_milters = unix:/opendkim/opendkim.sock
non_smtpd_milters = unix:/opendkim/opendkim.sock
```

El socket se encuentra dentro del spool de Postfix.

## Entrega local

Los mensajes dirigidos al buzón local se entregan mediante Dovecot LMTP. Las pruebas mostraron:

```text
postfix/lmtp
status=sent
```

## Verificación operacional

```bash
postfix check
postconf -n
systemctl status postfix --no-pager
journalctl -u postfix --no-pager
mailq
```

Logs:

```bash
journalctl -u postfix -f
tail -f /var/log/mail.log
```

## Resultado

Postfix quedó validado para recepción SMTP, Submission con TLS obligatorio, SASL mediante Dovecot, entrega LMTP, entrega externa, OpenDKIM y protección contra Open Relay.
