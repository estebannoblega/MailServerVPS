# 03 - Configuración de Dovecot

**Proyecto:** Servidor de Correo `enoblega.com.ar`  
**Sistema Operativo:** Debian 13 (Trixie)  
**Software:** Dovecot 2.4.x

---

# Objetivo

Configurar Dovecot como servidor IMAP utilizando:

- Usuarios virtuales.
- Maildir.
- Un único usuario del sistema (`vmail`).
- Autenticación mediante archivo.
- Sin depender de usuarios Linux.
- Sin base de datos.

Esta configuración será posteriormente integrada con Postfix.

---

# Arquitectura

```
                    Thunderbird
                          │
                     IMAPS (993)
                          │
                          ▼
                      Dovecot
                          │
               ┌──────────┴──────────┐
               │                     │
           Autenticación        Lectura Maildir
               │                     │
               ▼                     ▼
      /etc/dovecot/users      /var/vmail
```

Dovecot **NO recibe correos**.

Su responsabilidad es:

- autenticar usuarios;
- localizar buzones;
- servir el contenido mediante IMAP.

Los mensajes serán entregados posteriormente por Postfix.

---

# Decisiones de diseño

Durante el desarrollo del proyecto se tomaron las siguientes decisiones.

## Formato de almacenamiento

Se eligió **Maildir**.

No se utilizará mbox.

### Motivos

- evita corrupción de buzones grandes;
- permite acceso concurrente;
- cada correo es un archivo independiente;
- es el formato recomendado actualmente.

---

## Usuarios virtuales

No se utilizarán usuarios Linux.

No existirán cuentas como:

```
useradd contacto
```

Todos los buzones serán usuarios virtuales.

---

## Usuario propietario

Se creó un único usuario del sistema:

```
vmail
```

UID:

```
5000
```

GID:

```
5000
```

Este usuario será propietario de todos los buzones.

---

## Ubicación de los buzones

```
/var/vmail
```

Estructura:

```
/var/vmail

└── enoblega.com.ar
    └── contacto
        └── Maildir
            ├── cur
            ├── new
            └── tmp
```

Esta estructura permite agregar nuevos dominios sin modificar la arquitectura.

Ejemplo:

```
/var/vmail

├── enoblega.com.ar
└── empresa2.com.ar
```

---

# Instalación

```
apt update

apt install dovecot-core dovecot-imapd
```

---

# Backup previo

Antes de cualquier modificación se realizó:

```bash
cp -a /etc/dovecot /etc/dovecot.bak-$(date +%F)
```

Regla del proyecto:

Siempre realizar:

1. Backup.
2. Modificación.
3. Validación.
4. Reinicio del servicio.

---

# Configuración de Maildir

Archivo:

```
/etc/dovecot/conf.d/10-mail.conf
```

Configuración aplicada:

```ini
mail_driver = maildir

mail_home = /var/vmail/%{user | domain}/%{user | username}

mail_path = %{home}/Maildir
```

---

## Explicación

### mail_driver

Indica el formato utilizado por los buzones.

Anteriormente:

```
mbox
```

Ahora:

```
maildir
```

---

### mail_home

Define el directorio raíz del usuario.

Ejemplo.

Usuario:

```
contacto@enoblega.com.ar
```

Resultado:

```
/var/vmail/enoblega.com.ar/contacto
```

---

### mail_path

Define dónde se encuentra el Maildir.

Resultado final:

```
/var/vmail/enoblega.com.ar/contacto/Maildir
```

---

# Configuración de autenticación

Archivo:

```
/etc/dovecot/conf.d/10-auth.conf
```

Configuración original:

```ini
!include auth-system.conf.ext
```

Configuración nueva:

```ini
#!include auth-system.conf.ext

!include auth-passwdfile.conf.ext
```

---

## Motivo

Eliminar completamente la dependencia de:

- PAM
- /etc/passwd

La autenticación pasará a realizarse mediante un archivo.

---

# Configuración passwd-file

Archivo:

```
/etc/dovecot/conf.d/auth-passwdfile.conf.ext
```

Contenido:

```ini
passdb passwd-file {

    default_password_scheme = ARGON2ID

    passwd_file_path = /etc/dovecot/users

}

userdb passwd-file {

    passwd_file_path = /etc/dovecot/users

}
```

---

## Explicación

### passdb

Valida la contraseña.

Pregunta:

> ¿La contraseña ingresada es correcta?

---

### userdb

Localiza el buzón.

Pregunta:

> ¿Dónde vive este usuario?

---

# Archivo de usuarios

```
/etc/dovecot/users
```

Ejemplo.

```
contacto@enoblega.com.ar:{ARGON2ID}HASH:5000:5000::/var/vmail/enoblega.com.ar/contacto
```

---

## Importante

Este archivo **NO será administrado manualmente**.

Será generado automáticamente desde:

```
/opt/mailserver/config/users
```

mediante los scripts del proyecto.

---

# Contraseñas

Algoritmo utilizado:

```
ARGON2ID
```

Generación:

```bash
doveadm pw -s ARGON2ID
```

Motivos:

- recomendado actualmente;
- resistente a ataques GPU;
- soportado nativamente por Dovecot.

---

# Creación del Maildir

Comando utilizado.

```bash
sudo -u vmail maildirmake.dovecot /var/vmail/enoblega.com.ar/contacto/Maildir
```

Resultado.

```
Maildir

├── cur

├── new

└── tmp
```

---

# Validaciones realizadas

## Validar configuración

```bash
doveconf -n
```

---

## Verificar usuario

```bash
doveadm user contacto@enoblega.com.ar
```

Resultado esperado.

```
uid=5000

gid=5000

home=/var/vmail/enoblega.com.ar/contacto

mail_path=/var/vmail/enoblega.com.ar/contacto/Maildir
```

---

## Verificar autenticación

```bash
doveadm auth test contacto@enoblega.com.ar
```

Resultado esperado.

```
auth succeeded
```

---

# Estado actual del proyecto

✔ Dovecot instalado.

✔ Maildir configurado.

✔ Usuarios virtuales.

✔ ARGON2ID.

✔ Maildir creado.

✔ Autenticación funcional.

✔ Independencia total de usuarios Linux.
