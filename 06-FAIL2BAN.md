# 06 - Fail2ban

## Objetivo

Documentar la incorporación de Fail2ban como segunda capa de protección para los servicios SSH, Dovecot y SMTP AUTH.

El firewall perimetral de DonWeb continúa siendo la primera capa.

## Instalación

Versión:

```text
Fail2ban 1.1.0
```

Servicio:

```bash
systemctl status fail2ban --no-pager
```

Backend disponible:

```text
nftables
```

`iptables` está utilizando el backend `nf_tables`.

## Jails activos

```text
sshd
dovecot
postfix-sasl
```

El jail `sshd` ya existía y no fue reemplazado.

## Filtro Postfix específico

Archivo:

```text
/etc/fail2ban/filter.d/postfix-sasl.conf
```

Regex utilizada:

```ini
[Definition]

failregex = ^.*postfix/smtpd\[[0-9]+\]: warning: .*?\[<HOST>\]: SASL PLAIN authentication failed:

ignoreregex =
```

El filtro fue validado contra el log real y encontró el intento de autenticación fallida generado deliberadamente durante las pruebas.

## Jail Postfix

Archivo:

```text
/etc/fail2ban/jail.d/mailserver.conf
```

Configuración:

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

Esto significa:

```text
5 fallos
dentro de 10 minutos
        ↓
ban durante 1 hora
```

El filtro se limita a fallos de autenticación SASL PLAIN.

## Jail Dovecot

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

Se utiliza el filtro oficial incluido con Fail2ban.

## Validación

Antes de reiniciar el servicio:

```bash
fail2ban-client -t
```

Resultado:

```text
OK: configuration test is successful
```

Después:

```bash
systemctl restart fail2ban
```

Y:

```bash
fail2ban-client status
```

Resultado validado:

```text
Number of jail: 3
Jail list: dovecot, postfix-sasl, sshd
```

## Estado inicial de los jails

Al finalizar la configuración:

```text
postfix-sasl
Currently failed: 0
Total failed: 0
Currently banned: 0
Total banned: 0

dovecot
Currently failed: 0
Total failed: 0
Currently banned: 0
Total banned: 0
```

Esto es normal: el servicio está activo, pero no había suficientes fallos de autenticación para generar bans.

## Decisión de diseño

No se utilizó el filtro Postfix genérico como jail de bloqueo.

Durante las pruebas se observaron eventos como:

```text
Relay access denied
improper command pipelining
```

Estos eventos pueden provenir de servidores externos, scanners o pruebas legítimas y no necesariamente representan ataques de credenciales.

Por ese motivo, el filtro de SMTP se restringió a:

```text
SASL PLAIN authentication failed
```

Esto reduce el riesgo de falsos positivos.

## Firewall DonWeb

Fail2ban no reemplaza el firewall perimetral.

La arquitectura queda:

```text
Internet
   |
   v
Firewall DonWeb
   |
   v
VPS
   |
   +-- Fail2ban
   |     +-- sshd
   |     +-- postfix-sasl
   |     +-- dovecot
   |
   +-- Postfix
   +-- Dovecot
   +-- OpenDKIM
```

No se requieren puertos adicionales para Fail2ban.
