# U1 - Introducción
El correo electronico es un sistema distribuido no depende de un proveedor central. Los servidores de correo se descubren mediantes registros MX en el DNS.  
El protocolo __SNMP__ transporta mensajes entre clientes y servidores, como así tambien entre servidores. Mientras que el protocolo __IMAP__ permite acceder y sincronizar los busones de correo.  
El MTA (Mail Transfer Agent) es el "cartero", o sea, es quien se encarga de mover los correos, algunos ejemplos de MTA son: Posfix, Exim,Sendmail,etc.  
MDA (Mail Delivery Agent) es quien tiene como trabajo guardar el correo. Algunos ejemplos son Dovecot LDA, Procmail, Maildrop, etc. Estos agentes se encargan de exponer los busones de los clientes mediante  __IMAP__ y de la autentiación.  
Un mensaje de correo termina almacenado como uno o varios archivos en disco, normalmente en formato __maildir__.


# U2 - Diseño de la arquitectura
Aqui definiremos lo que quermos construir.
La idea de este proyecto es crear un servidor mail.enoblega.com.ar con una cantidad máxima de cuentas en 5. Como se busca entender el proceso e implementación de sistemas de correos, no se tendrá en cuenta alta disponibilidad, ni que sea para miles de usuarios ni multiempresa. La idea es que sea algo simple, mantenible y seguro.  
Para entenderlo mejor vamos a dividir el servidor en módulos:  
~~~
                   Internet
                       │
        ┌──────────────┴──────────────┐
        │                             │
     SMTP (25)                  HTTPS (443) 
        │                             │
        ▼                             ▼
    Postfix                         Nginx
        │                             │
        ▼                             ▼
     Dovecot                     Roundcube
        │
        ▼
     Maildir
        │
        ▼
      Storage
~~~

* Usaremos _Postfix_ como MTA, su trabajo va a consisir en recibir y enviar correos, mantener comunicación con otros servidores SMTP, mantener la cola de correos y decidir a qué servidor conectarse. ACLARACION: __NO MUESTRA LOS CORREOS AL USUARIO, SOLO ES QUIEN LOS ENTREGA.__
* _Dovecot_ realizará dos tareas, sincronizar los correos usando __IMAP__ y autenticar usuarios.
* _Maildir_ es quien se encargará de guardar los correos, estos se guardan como archivos, en general de tipos .eml. __Cada correo es un archivo independiente__.
* Se usarán usuarios virtuales para desacoplarlo del SO. Y para gestionar las credenciales, alias, dominios y demas se utilizará un archivo de configuración. En un futuro se debería migrar a MariaDB o alguna DB para escalar el proyecto.
* Se puede optar por una instalación nativa o correrlo sobre docker, en este caso la instalación será nativa.
* La administración del servidor se harán mediante scripts, sin embargo, si el proyecto crece a más cyuentas conviene buscar otra opción.
Teniendo todos estos puntos en cuenta el diseño quedaría de la siguiente manera:
~~~
                    Internet
                         │
          ┌──────────────┴──────────────┐
          │                             │
       SMTP 25                     HTTPS 443
          │                             │
          ▼                             ▼
      Postfix                       Nginx
          │                             │
          ▼                             ▼
      Dovecot                      Roundcube
          │
          ▼
     Autenticación
          │
          ▼
       MariaDB
          │
          ▼
       /var/vmail
          │
          ▼
        Maildir
~~~


# U3 - Protocolos del correo electrónico
### SMTP
Simple Mail Transfer Protocol, es un procolo diseñado únicamente para mover mensajes, no sirve para leerlos, organizarlos, sincronizar carpetas o mostrar archivos adjuntos.  
SMTP necesita una conexión confiable por lo que usa el protocolo TCP, por lo que cuando el servidor quiere enviar un correo primero hace el tree-way handsake para establecer una conexión confiable.  
SMTP es texto plano, es literalmente una conversación de texto.  
En la comunicación entre servidores cada uno define sus capacidades, es decir, que es lo que sabe hacer.

### Capacidades
* PIPELINING: Permite enviar varios comandos sin esperar respuesta, lo que hace al protocolo más rápido.
* SIZE: Define el tamaño de los mensajes
* STARTTLS: Indica que la comunicación debe ser cifrada
* AUTH: Indica que acepta autentiación.
### ¿Como se envia un correo?
Suponiendo que quiero enviar un correo de: conctacto@enoblega.com.ar a personal@gmail.com.
La conversación sería algo así:
* Servidor: 220
* Cliente: EHLO mail.enoblega.com.ar
* Servidor: 250 OK
* Cliente: MAIL FROM:<contacto@enoblega.com.ar>
* Servidor: 250 OK
* Cliente: RCPT TO: <personal@gmail.com>
* Servidor: 250 OK
* Cliente: DATA
* Servidor: 354 End data with <CRLF>.<CRLF>
* Recién aqui comienza el correo que vemos desde thunderbird:
~~~
From: contacto@enoblega.com.ar

To: persona@gmail.com

Subject: Hola

Hola Esteban

¿Cómo estás?

.
~~~
* Servidor: 250 Message accepted
* Cliente: QUIT
* Servidor: 221 Bye


### ¿Qué es el puerto 25?
Este puerto es el que permite la comunicación entre servidores, por ejemplo de _Postfix_ a _Gmail_. Siempre de servidor a servidor.
El puerto __587__ es el que usa el cliente para enviar el correo ya que el servidor espera usuario, contraseña STARTTLS.  
Por lo que el flujo de los correos en realidad es:
~~~
Thunderbird

↓

587

↓

Postfix

↓

25

↓

Google
~~~
### STARTTLS
Es el mismo concepto que cuando se usa https. Convierte la conexión SMTP (texto plano) en una conexión cifrada mediante TLS.
Una vez establecido el canal cifrado, el cliente vuelve a enviar EHLO y recién entonces continúa la autenticación.
### SASL
Simple Authentication and Security Layer. No es un protocolo de correos sino un __mecanismo de autenticación__.   
Cuando thunderbird envía un usuario y contraseña, normalmente viaja mediante SASL, Postfix no tiene por qué validar la contraseña por sí mismo, generalmente delega esta tarea a Dovecot. Por lo que la secuencia queda así:
~~~
Thunderbird

↓

AUTH

↓

Postfix

↓

Dovecot

↓

OK
~~~
### IMAP
Suponiendo que el correo llegó al servidor, el protocolo SMTP terminó yel mensaje quedó almacenado en _Maildir_. Aquí entra en juego IMAP.
~~~
Thunderbird

↓

993

↓

Dovecot

↓

Maildir
~~~
Cuando se abre la bandeja de entrada Thunderbird no descarga todo el buzón, primero solicita la información como qué carpetas existen, qué mensajes hay, cuáles están leídos, etc. Solo después descarga el contenido que necesitaba.  

### Códigos SMTP
Listado con los códigos más comunes:
~~~
| Código | Significado                                    |
| ------ | ---------------------------------------------- |
| 220    | El servidor está listo.                        |
| 221    | La conexión se cierra correctamente.           |
| 235    | Autenticación exitosa.                         |
| 250    | La operación fue aceptada.                     |
| 354    | Comenzá a enviar el contenido del mensaje.     |
| 421    | Servicio no disponible temporalmente.          |
| 450    | Buzón temporalmente no disponible.             |
| 550    | Buzón inexistente o acceso denegado.           |
| 554    | Transacción rechazada (spam, políticas, etc.). |

~~~

Hasta el momento la estructura va de la siguiente manera:
~~~
                   INTERNET
                       │
              SMTP (25 / 587)
                       │
                       ▼
                  +-----------+
                  |  Postfix  |
                  +-----------+
                       │
          Entrega el mensaje al buzón
                       │
                       ▼
                +---------------+
                |    Maildir    |
                +---------------+
                       ▲
                       │
                 Lee y autentica
                       │
                  +-----------+
                  | Dovecot   |
                  +-----------+
                       ▲
                  IMAP (993)
                       │
                  Thunderbird
~~~
Donde Postfix transporte, Dovecot expone los buzones y autentica los usuarios

# U4 - Diseño de la infraestructura
Tiene como objetivo definir software a instalar, definir directorios de configuración, definir puertos a utilizar, qué proceso atiende a cada conexión y cómo se relacionan todos los componentes.

### El sistema operativo
En este caso al tratarde de un proyecto pequeño el servidor de correo será un servicio "más" dentro de mi VPS. Ya que el correo en general consume pocos recursos.
### Software necesario
El stack que se utilizará:
~~~
| Software                 | Función                   |
| ------------------------ | ------------------------- |
| Postfix                  | SMTP                      |
| Dovecot                  | IMAP + autenticación      |
| Let's Encrypt            | TLS                       |
| rsyslog + journald       | Logs                      |
| logrotate                | Rotación de logs          |
| Fail2ban                 | Protección contra ataques |
| Roundcube (más adelante) | Webmail                   |
~~~

### Estructura de directorios
Para entender este punto se debe pensar el servidor como un árbol. Donde los directorios más importantes serán:
* __/etc__: Solo contendrá configuración. Ej:
    ~~~
    /etc/postfix/
    /etc/dovecot/
    ~~~
* __/var__: Datos que cambian constantemente. Aquí vivirán los correos. Ej:
    ~~~
    /var/log/
    /var/spool/
    /var/vmail/
    ~~~
* __/usr__: Contiene programas.
    ~~~
    /usr/sbin/postfix
    /usr/sbin/dovecot
    ~~~

### Recorrido de un correo dentro del servidor
Suponiendo que gmail quiere entregar un correo a una casilla de mi dominio:
~~~
Internet
↓
Puerto 25
↓
Postfix
↓
¿Existe contacto@enoblega.com.ar?
↓
Sí
↓
Guardar Maildir
↓
Fin
~~~

Cuando el usuario de mi dominio abra thunderbird se realizará este flujo:
~~~
Thunderbird
↓
993
↓
Dovecot
↓
Leer Maildir
↓
Mostrar correo
~~~

Aqui hay que tener en cuenta que __Postfix__ nunca habla con __Thunderbird__ y __Thunderbird__ nunca habla con __Postfix__ para leer los correos.

### Puertos a utilizar
Cada puerto a utilizar termina en un proceso distinto por lo que no debería haber ningún conflicto.
~~~
80
↓

nginx

-------------------

443
↓

nginx

-------------------

25
↓

Postfix

-------------------

587
↓

Postfix

-------------------

465
↓

Postfix

-------------------

993
↓

Dovecot
~~~

### Certificados
Lo que se necesita para este proyecto es un certificado de dominio no para postfix. El certificado generado será usado por Postfix, Dovecot, Roundcube. Si, todos pueden compartir exactamente el mismo certificado.

### Usuarios
Como definimos antes, los usarios serán virtuales no serán usuarios de linux. Esto permite merjorar la seguridad del proyecto.

### Almacenamiento
En este punto se definirá como usar __Maildir__. En donde tendremos algo parecido a:
~~~
/var/vmail

└── enoblega.com.ar

    ├── contacto

    │      Maildir

    │

    ├── ventas

    │      Maildir

    │

    └── soporte

           Maildir
~~~
Es decir, cada cuenta tiene su propio __Maildir__ y cada __Maildir__ tiene: _cur/_, _new/_, _tmp/_.
Si el servidor se reincia no se perderá ningun archivo ya que todo quedará en el disco.

#### Logs
Los archivos de logs se deberán guardar en _/var/log/_.

### La cola de correo
Es un concepto de Postfix, si el servidor de correo destinatario no responde, postfix guarda el correo en una cola, espera y luego reintenta la conexión con el servidor destinatario.

### Servidor Completo
La idea es que al terminar el proyecto el servidor quede algo así:
~~~
                    Internet
                         │
        ┌────────────────┴────────────────┐
        │                                 │
      HTTP/HTTPS                      SMTP/IMAP
        │                                 │
        ▼                                 ▼
      nginx                           Postfix
        │                                 │
        ▼                                 │
   Sitios web                        Mail Queue
                                          │
                                          ▼
                                     Maildir
                                          ▲
                                          │
                                      Dovecot
                                          ▲
                                          │
                                     Thunderbird
~~~

# U5 - Preparación del servidor
En esta unidad se preparará el servidor definiendo un hostname correcto, fqdn configurado, resolución DNS correcta, puertos definidos. De manera que nos aseguremos que Postfix puede instalarse sin problemas.

### Revisar estado del servidor
Debian13 - Hostname.
Defino el FQDN: mail.enoblega.com.ar
Muchos filtros antispam miral el banner SMTP que es donde viaja el hostname, por lo que es recomendable que no diga debian,localhost o algo similar.
### FQDN
El servidor tiene tres nombres:
* Hostname: mail
* Dominio: enoblega.com.ar
* FQDN: mail.enoblega.com.ar

Los 3 son cosas distintas y _Postfix_ usa FQDN.

### GUARDADO DE USUARIOS
Usar el directorio: _/etc/mailsever/_ y dentro de éste un único archivo de configuración: __users.conf__. Este archivo contendrá algo parecido a:
~~~
contacto@enoblega.com.ar:hash
ventas@enoblega.com.ar:hash
info@enoblega.com.ar:hash
~~~

### Documentación del proyecto 
La idea es documentar todo el servidor como si fuera un proyecto de software, algo asi:
~~~
/opt/mailserver

README.md

INSTALACION.md

usuarios.conf

scripts/

backup.sh

crear_usuario.sh

cambiar_password.sh

eliminar_usuario.sh
~~~

De esta manera ademas de tener un servidor funcional, tenga un repositorio autocontenido con documentación y scripts de administración.

## U5.5 - Funcionamiento de Postfix
Postfix no es un único programa, sino un conjunto de varios programas. Tiene su proceso master y a partir de ahi se subdividen en programas/procesos que se encargan de una tarea en específico. Algo similar a esto:
~~~
master
        │
        ├── smtpd
        ├── pickup
        ├── cleanup
        ├── qmgr
        ├── trivial-rewrite
        ├── smtp
        ├── local
        ├── virtual
        └── bounce
~~~

Este proceso _master_ no entrega correos, no habla SMTP, no escribe buzones. Su único trabajo es administrar procesos. Por ejemplo cuando ingresa un correo el proceso master dice "Necesito un proceso SMTP" y entonces crea uno: _smtpd_.  
El recorrido del correo quedaría algo así:
~~~
Internet
     │
     ▼
smtpd
     │
     ▼
cleanup
     │
     ▼
qmgr
     │
     ├────────► local
     │
     ├────────► virtual
     │
     └────────► smtp
~~~

* __smtpd__ Es el primer proceso, su trabajo es únicamente hablar SMTP, solo entiende cosas como: EHLO, MAIL FROM, DATA, QUIT. Nada más, solo conversa. Es quien recibe los correos.
* Luego, cuando termina la conversación le entrega el mensaje a otro proceso: __cleanup__. Este es que agrega cabecera, verifica el formato, normaliza el mensaje, y otras cosas. Es básicmanete un inspector que revisa si está todo bien. Si está ok lo manda a la cola.
* __qmgr__ Basicamente es el corazón de postfix, es el __queue manager__ todo correo debe terminar aqui. Es el que se encarga de distribuir los correos que llegan.
* __smtp__ Este proceso sí es el que entrega los correos.
* __local__ sirve para entregar correos a usuarios Linux.
* __virtual__ sirve para entregar correos a usuarios virtuales que solo existen en postfix.
* __bounce__ es el que genera los mensajes de error.
* __trivial-rewrite__ es el que se encarga de "autocompletar" lo que escribis.
* __pickup__ analiza si el correo entra por SNMP si no entra por SNMP lo pone en cola un correo segun corresponda.

__smtp__, __local__ y __virtual__ entregan correos.

### Recorrido Completo
Una vez definida (resumidamente) la arquitectura interna de Postfix, podemos graficarla de la siguiente manera:
~~~
                   Internet
                        │
                        ▼
                     smtpd
                        │
                        ▼
                    cleanup
                        │
                        ▼
                      qmgr
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
      local          virtual         smtp
         │              │              │
         ▼              ▼              ▼
   Usuario Linux     Maildir      Otro servidor
~~~

### Ayuda memoria 
Este ayuda memoria permite entender o recordar que hace cada proceso:
~~~
master
        Director general

smtpd
        Recepcionista

pickup
        Empleado que junta cartas del buzón interno

cleanup
        Inspector de calidad

qmgr
        Jefe de logística

smtp
        Camión que sale a otras ciudades

virtual
        Cartero del barrio "usuarios virtuales"

local
        Cartero del barrio "usuarios Linux"

bounce
        Oficina de devoluciones
~~~


# U6 - MANOS A LA OBRA
Para dejar documentado el proyecto se trabajará pensandolo como un proyecto de software, para el cual definiremos la siguiente estructura:
~~~
/opt/mailserver
│
├── README.md
├── INSTALACION.md
├── ARQUITECTURA.md
├── ADMINISTRACION.md
├── TROUBLESHOOTING.md
├── CHANGELOG.md
│
├── config/
│   ├── users
│   ├── aliases
│   └── domains
│
├── scripts/
│   ├── crear_usuario.sh
│   ├── eliminar_usuario.sh
│   ├── cambiar_password.sh
│   ├── backup.sh
│   ├── restaurar.sh
│   └── verificar_config.sh
│
└── backups/
~~~