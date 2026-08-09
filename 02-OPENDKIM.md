# 02 - OpenDKIM

## Objetivo

Documentar la configuración de OpenDKIM utilizada para firmar correo saliente de `enoblega.com.ar`.

## Rol

OpenDKIM funciona como milter de Postfix y agrega la firma DKIM a los mensajes salientes.

Selector:

```text
mail
```

Registro DNS:

```text
mail._domainkey.enoblega.com.ar
```

## Clave privada

La clave privada se encuentra en el servidor en:

```text
/etc/opendkim/keys/enoblega.com.ar/mail.private
```

**No debe formar parte de un repositorio público.**

## Socket

El socket utilizado por Postfix es:

```text
/var/spool/postfix/opendkim/opendkim.sock
```

Directorio:

```text
/var/spool/postfix/opendkim
```

El proceso `postfix` fue agregado al grupo `opendkim` para permitir el acceso al socket.

## Integración con Postfix

```text
smtpd_milters = unix:/opendkim/opendkim.sock
non_smtpd_milters = unix:/opendkim/opendkim.sock
```

## DNS

El registro DKIM publicado tiene la forma:

```text
mail._domainkey.enoblega.com.ar TXT
```

con:

```text
v=DKIM1; h=sha256; k=rsa; p=<CLAVE_PUBLICA>
```

La clave pública puede publicarse en DNS. La clave privada nunca debe publicarse.

## Verificación

```bash
opendkim-testkey   -d enoblega.com.ar   -s mail   -k /etc/opendkim/keys/enoblega.com.ar/mail.private   -vvv
```

Resultado validado:

```text
key not secure
key OK
```

`key not secure` indica que la respuesta DNS no está protegida por DNSSEC; no invalida la clave DKIM.

El resultado relevante es:

```text
key OK
```

## Servicio

```bash
systemctl status opendkim --no-pager
systemctl restart opendkim
journalctl -u opendkim --no-pager
```

## Verificación externa

Un correo enviado hacia Gmail fue validado con:

```text
DKIM: PASS
```

Esto confirma que Postfix entregó el mensaje al milter, OpenDKIM lo firmó y el receptor pudo validar la firma mediante DNS.

## Seguridad

Nunca versionar:

```text
mail.private
*.pem
*.key
credenciales
/etc/dovecot/users
```

Usar placeholders como:

```text
<PRIVATE_KEY>
<PASSWORD>
<SECRET>
```
