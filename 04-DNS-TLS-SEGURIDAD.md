# 04 - DNS, TLS y seguridad

## DNS

Dominio:

```text
enoblega.com.ar
```

### MX

Registro validado:

```text
enoblega.com.ar. MX 10 smtp.enoblega.com.ar.
```

Comprobación:

```bash
dig @ns1.donweb.com MX enoblega.com.ar +short
dig @8.8.8.8 MX enoblega.com.ar +short
dig @1.1.1.1 MX enoblega.com.ar +short
```

Resultado:

```text
10 smtp.enoblega.com.ar.
```

### SPF

El dominio publica un SPF que autoriza la IP pública del servidor y utiliza `-all`.

Para documentación pública se recomienda representar la IP como:

```text
<IP_PUBLICA_DEL_SERVIDOR>
```

### DKIM

Selector:

```text
mail
```

Registro:

```text
mail._domainkey.enoblega.com.ar TXT
```

### DMARC

Registro:

```text
_dmarc.enoblega.com.ar TXT
```

Política inicial:

```text
v=DMARC1; p=none
```

La política fue validada mediante correo real. Gmail mostró:

```text
SPF: PASS
DKIM: PASS
DMARC: PASS
```

## TLS

### SMTP - 25

STARTTLS está disponible para transporte SMTP servidor-a-servidor.

### Submission - 587

STARTTLS es obligatorio mediante:

```text
-o smtpd_tls_security_level=encrypt
```

### IMAPS - 993

El acceso de clientes se realiza mediante IMAP sobre TLS.

## Firewall perimetral

El firewall de DonWeb permite los puertos necesarios:

```text
25/TCP   SMTP
587/TCP  SMTP Submission
993/TCP  IMAPS
```

SSH se mantiene restringido según las reglas administrativas existentes.

Fail2ban no requiere abrir puertos adicionales.

## Fail2ban

Versión utilizada:

```text
1.1.0
```

Jails activos:

```text
sshd
dovecot
postfix-sasl
```

Backend:

```text
nftables
```

### SMTP AUTH

Filtro:

```text
/etc/fail2ban/filter.d/postfix-sasl.conf
```

Detecta exclusivamente:

```text
SASL PLAIN authentication failed
```

Jail:

```ini
[postfix-sasl]
enabled = true
filter = postfix-sasl
logpath = /var/log/mail.log
backend = auto
port = submission
maxretry = 5
findtime = 10m
bantime = 1h
```

### Dovecot

Se utiliza el filtro oficial:

```text
/etc/fail2ban/filter.d/dovecot.conf
```

Configuración:

```ini
[dovecot]
enabled = true
filter = dovecot
logpath = /var/log/mail.log
backend = auto
port = imaps
maxretry = 5
findtime = 10m
bantime = 1h
```

### Por qué no se usa el filtro Postfix genérico

El filtro general `postfix.conf` detecta múltiples eventos, como `Relay access denied`, comandos SMTP incorrectos y pipelining incorrecto.

Durante las pruebas se observaron rechazos de relay legítimos y conexiones automatizadas. Para evitar falsos positivos se creó un filtro específico para fallos de `SASL PLAIN`.

## Validación

```bash
fail2ban-client -t
```

Resultado:

```text
OK: configuration test is successful
```

Estado:

```bash
fail2ban-client status
```

Resultado:

```text
Number of jail: 3
Jail list: dovecot, postfix-sasl, sshd
```

## Seguridad del repositorio

Nunca subir:

```text
/etc/dovecot/users
/etc/opendkim/keys/*/mail.private
/etc/letsencrypt/live/*/privkey.pem
archivos .env
contraseñas
tokens
backups de Maildir
logs con información sensible
```

Usar:

```text
<PASSWORD>
<PRIVATE_KEY>
<SECRET>
<IP_PUBLICA>
```
