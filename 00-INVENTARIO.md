# 00 - Inventario

## Objetivo

Este documento proporciona una visión general y sanitizada de la infraestructura del servidor de correo.

---

## Servidor

| Parámetro | Valor |
|---|---|
| Sistema operativo | Debian 13 |
| Hostname | `<MAIL_SERVER_HOSTNAME>` |
| IP pública | `<IP_PUBLICA>` |
| Dominio principal | `enoblega.com.ar` |
| Rol | Servidor de correo |

---

## Servicios

| Componente | Función |
|---|---|
| Postfix | MTA / SMTP |
| Dovecot | IMAP, LMTP y autenticación |
| OpenDKIM | Firma DKIM |
| Fail2ban | Protección contra abuso |
| Let's Encrypt | Certificados TLS |

---

## Servicios de correo

### SMTP

```text
25/TCP
```

Uso:

- recepción de correo desde otros servidores;
- transporte SMTP servidor-a-servidor;
- entrega de correo saliente.

### SMTP Submission

```text
587/TCP
```

Uso:

- envío de correo desde clientes;
- autenticación mediante Dovecot;
- STARTTLS obligatorio.

### IMAPS

```text
993/TCP
```

Uso:

- acceso seguro de clientes al buzón;
- IMAP sobre TLS.

No se utiliza IMAP sin cifrado como servicio público.

---

## Dominio

Dominio principal:

```text
enoblega.com.ar
```

Cuenta documentada:

```text
contacto@enoblega.com.ar
```

La cantidad de cuentas puede ampliarse sin modificar la arquitectura general.

---

## DNS

El dominio utiliza los siguientes mecanismos de autenticación y direccionamiento:

```text
MX
SPF
DKIM
DMARC
```

### MX

```text
enoblega.com.ar
    |
    +--> smtp.enoblega.com.ar
```

### SPF

El dominio publica una política SPF que autoriza al servidor de correo.

La IP real se omite deliberadamente de este inventario público.

### DKIM

Selector:

```text
mail
```

Registro:

```text
mail._domainkey.enoblega.com.ar
```

### DMARC

Registro:

```text
_dmarc.enoblega.com.ar
```

Política actual:

```text
v=DMARC1; p=none
```

La autenticación DMARC fue validada mediante correo real.

---

## Almacenamiento

Los buzones virtuales se almacenan bajo:

```text
/var/vmail/
```

Estructura conceptual:

```text
/var/vmail/
└── enoblega.com.ar/
    └── contacto/
        └── Maildir/
```

El contenido de los buzones **no forma parte del repositorio**.

---

## Archivos y directorios de configuración

### Postfix

```text
/etc/postfix/
```

### Dovecot

```text
/etc/dovecot/
```

### OpenDKIM

```text
/etc/opendkim/
```

### Fail2ban

```text
/etc/fail2ban/
```

### Certificados

```text
/etc/letsencrypt/
```

Las claves privadas y certificados sensibles no deben versionarse.

---

## Seguridad

### Firewall perimetral

Existe un firewall perimetral proporcionado por el proveedor de infraestructura.

Los servicios públicos necesarios son:

```text
25/TCP   SMTP
587/TCP  SMTP Submission
993/TCP  IMAPS
```

SSH permanece restringido mediante las reglas administrativas correspondientes.

No se documentan aquí las IPs permitidas para administración.

### Fail2ban

Jails activos:

```text
sshd
dovecot
postfix-sasl
```

El filtro específico de Postfix protege los intentos fallidos de:

```text
SASL PLAIN authentication
```

La configuración utiliza nftables como mecanismo de bloqueo.

---

## TLS

Los servicios de correo utilizan certificados TLS gestionados mediante Let's Encrypt.

Servicios protegidos:

```text
SMTP / STARTTLS
SMTP Submission / STARTTLS obligatorio
IMAPS
```

Los archivos de claves privadas permanecen exclusivamente en el servidor.

---

## Integraciones

```text
                    Internet
                       |
              Firewall perimetral
                       |
        +--------------+--------------+
        |              |              |
      SMTP 25       SMTP 587       IMAPS 993
        |              |              |
        +--------------+--------------+
                       |
                    Postfix
                       |
          +------------+------------+
          |                         |
       OpenDKIM                  Dovecot
          |                         |
          |                    IMAP / LMTP
          |                         |
          +-------------------- Maildir
```

Fail2ban funciona como capa adicional de protección sobre los servicios expuestos.

---

## Validaciones realizadas

El servicio fue validado operacionalmente mediante:

- recepción desde Gmail;
- envío hacia Gmail;
- envío y recepción mediante cliente Outlook;
- SMTP Submission autenticado;
- IMAPS;
- entrega local mediante LMTP;
- validación de DKIM;
- validación de SPF;
- validación de DMARC;
- prueba de Open Relay;
- validación de Fail2ban.

### Resultado

```text
SPF       PASS
DKIM      PASS
DMARC     PASS
OpenRelay CLOSED
SMTP      OK
Submission OK
IMAPS     OK
LMTP      OK
Fail2ban  ACTIVE
```

---

## Información que NO debe publicarse

Nunca incluir en este repositorio:

```text
Contraseñas
Claves privadas DKIM
Claves privadas TLS
/etc/dovecot/users
Tokens
API keys
Archivos .env
Backups
Maildir
Contenido de correos
Logs completos
IPs de administración
IPs de origen permitidas para SSH
Reglas internas del firewall
Credenciales de servicios externos
```

Utilizar placeholders:

```text
<IP_PUBLICA>
<MAIL_SERVER_HOSTNAME>
<PASSWORD>
<PRIVATE_KEY>
<SECRET>
```

---

## Mantenimiento

Este inventario debe actualizarse cuando cambien:

- sistema operativo;
- componentes principales;
- puertos públicos;
- dominio;
- cuentas de correo;
- arquitectura de almacenamiento;
- mecanismos de autenticación;
- políticas DNS;
- mecanismos de seguridad.

El objetivo de este archivo es proporcionar una **vista rápida y segura del entorno**, no reemplazar la documentación técnica específica de cada componente.
