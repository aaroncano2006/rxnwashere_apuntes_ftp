# 📘 Apuntes de FTP y VSFTPD

## 📑 Índice

1. [Introducción a FTP](#1-introducción-a-ftp)
2. [Modos de Transferencia: Activo y Pasivo](#2-modos-de-transferencia-activo-y-pasivo)
3. [Instalación y Funcionamiento de VSFTPD](#3-instalación-y-funcionamiento-de-vsftpd)
4. [Configuración General de VSFTPD](#4-configuración-general-de-vsftpd)
5. [Forzar Modo Activo o Pasivo](#5-forzar-modo-activo-o-pasivo)
6. [Usuarios Anónimos](#6-usuarios-anónimos)
7. [Enjaular (Chroot) Usuarios FTP](#7-enjaular-chroot-usuarios-ftp)
8. [Excepciones a la Cárcel (chroot_list)](#8-excepciones-a-la-cárcel-chroot_list)
9. [Userdir + Apache + FTP](#9-userdir--apache--ftp)
10. [VirtualHosts con Apache y FTP](#10-virtualhosts-con-apache-y-ftp)
11. [Conexión Gráfica con FileZilla](#11-conexión-gráfica-con-filezilla)
12. [Conexión Segura con SFTP](#12-conexión-segura-con-sftp)
13. [Enjaular Usuarios SFTP con SSH](#13-enjaular-usuarios-sftp-con-ssh)
14. [Directivas Importantes de VSFTPD](#14-directivas-importantes-de-vsftpd)
15. [Enlaces de Interés](#15-enlaces-de-interés)


# 1. Introducción a FTP

FTP (**File Transfer Protocol**) es uno de los protocolos más antiguos para transferir ficheros entre un cliente y un servidor.

Características principales:

* No es seguro: el usuario y la contraseña viajan en texto plano.
* Usa **dos canales**:

  * Canal **de control** (comandos)
  * Canal **de datos** (transferencias)
* Funciona en **modo activo** o **modo pasivo**

Se usa habitualmente para:
✔ Servidores web
✔ Servidores internos
✔ Acceso rápido a carpetas de usuario


# 2. Modos de Transferencia: Activo y Pasivo

### 🔵 Modo Activo (PORT)

* Cliente se conecta al servidor por **21 (control)**
* El servidor abre una conexión de datos desde **el puerto 20** hacia un puerto aleatorio del cliente
* Si el cliente está detrás de un firewall → **suele fallar**

Esto te pasó en tu práctica: el firewall del cliente bloqueaba la conexión y tuviste que desactivarlo temporalmente.

### 🟢 Modo Pasivo (PASV)

* Cliente se conecta al servidor por **21**
* El servidor le responde con un puerto aleatorio (>1024)
* El cliente se conecta **al servidor**, no al revés
* Mucho más compatible con firewalls y NAT

Es el modo recomendado.


# 3. Instalación y Funcionamiento de VSFTPD

```bash
sudo apt install vsftpd
sudo systemctl enable --now vsftpd
sudo systemctl status vsftpd
```

Puertos típicos:

* **21** → Control
* **20** → Datos en modo activo
* **>1024** → Datos en modo pasivo


# 4. Configuración General de VSFTPD

Archivo principal:

```
/etc/vsftpd.conf
```

Parámetros básicos:

```conf
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
```

Después de modificar:

```bash
sudo systemctl reload vsftpd
```

# 5. Forzar Modo Activo o Pasivo

## 🔵 Forzar modo activo

```conf
pasv_enable=NO
port_enable=YES
```

## 🟢 Forzar modo pasivo (recomendado)

```conf
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10050
```

Al configurar el modo pasivo podemos limitar los puertos que se utilizan, la única condición es que todos estén por encima del puerto 1024.

# 6. Usuarios Anónimos

## Habilitar acceso anónimo

```conf
anonymous_enable=YES
write_enable=YES
anon_upload_enable=YES
anon_mkdir_write_enable=NO
```

La raíz por defecto del usuario anonymous es **<code>/srv/ftp</code>**.

Para permitir subir archivos sin comprometer la seguridad:

```bash
sudo mkdir /srv/ftp/upload
sudo chmod 777 /srv/ftp/upload
```

Puedes cambiar la home del usuario anónimo:

```conf
anon_root=/srv/ftp/public
```

# 7. Enjaular (Chroot) Usuarios FTP

“Enjaular” significa que un usuario **no puede salir de su /home** y la ve como si fuera `/`.

```conf
chroot_local_user=YES
allow_writeable_chroot=YES
```

Problema habitual:
El directorio home no puede ser escribible por seguridad, así que se debe crear una subcarpeta para subir archivos.


# 8. Excepciones a la Cárcel (chroot_list)

Crear archivo (o editar si ya existe o ya se ha creado previamente):

```
/etc/vsftpd.chroot_list
```

Configurar:

```conf
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list
```

Los usuarios listados en este archivo **NO estarán enjaulados**.

Ejemplo:

```
aaron
```

Solo deben añadirse los nombres de los usuarios que no se quieren enjaular, no hayq que especificar nada más.

# 9. Userdir + Apache + FTP

Permite que cada usuario tenga su propia web:

```
/home/usuario/public_html
```

Activar el módulo:

```bash
sudo a2enmod userdir
sudo systemctl reload apache2
```

URL de acceso:

```
http://host/~usuario
```

# 10. VirtualHosts con Apache y FTP

Crear un usuario para administrar un sitio:

```bash
sudo adduser web
sudo mkdir -p /var/www/web
sudo chown web:web /var/www/web
```

Crear VirtualHost:

```conf
<VirtualHost *:80>
    ServerName www.web.daw
    DocumentRoot /var/www/web
</VirtualHost>
```

Añadir al `/etc/hosts`:

```
127.0.0.1   www.web.daw
```


# 11. Conexión Gráfica con FileZilla

FileZilla permite probar:

✔ Usuario normal
✔ Usuario enjaulado
✔ Usuario anónimo
✔ Conexión segura (SFTP)

Si el usuario está enjaulado, FileZilla mostrará `/` aunque realmente esté en `/home/usuario`.


# 12. Conexión Segura con SFTP

```bash
sftp usuario@servidor
```

Notas importantes:

* **No usa FTP**, sino SSH.
* Todo el tráfico va cifrado.
* Wireshark lo identifica como SSH, no como FTP.


# 13. Enjaular Usuarios SFTP con SSH

Editar `/etc/ssh/sshd_config`:

```conf
Match User nombreusuario
    ChrootDirectory /home/nombreusuario
    ForceCommand internal-sftp
    X11Forwarding no
    AllowTcpForwarding no
```

Requisitos:

* El directorio raíz debe ser propiedad de root:

```bash
sudo chown root:root /home/nombreusuario
```

* Crear carpeta interior para subir archivos:

```bash
mkdir /home/nombreusuario/upload
chmod 777 /home/nombreusuario/upload
```

Reiniciar SSH:

```bash
sudo systemctl reload sshd
```


# 14. Directivas Importantes de VSFTPD

### 🔸 Autenticación

```conf
anonymous_enable
local_enable
pam_service_name
```

### 🔸 Escritura

```conf
write_enable
local_umask
anon_upload_enable
```

### 🔸 Modos de transferencia

```conf
pasv_enable
pasv_min_port
pasv_max_port
```

### 🔸 Seguridad y jaulas

```conf
chroot_local_user
chroot_list_enable
allow_writeable_chroot
```

### 🔸 Logs

```conf
xferlog_enable
xferlog_file
```

# 15. Enlaces de Interés

* Directivas VSFTPD
  [http://vsftpd.beasts.org/vsftpd_conf.html](http://vsftpd.beasts.org/vsftpd_conf.html)

* Administración de usuarios y grupos
  [http://www.ite.educacion.es/formacion/materiales/85/cd/linux/m1/administracin_de_usuarios_y_grupos.html](http://www.ite.educacion.es/formacion/materiales/85/cd/linux/m1/administracin_de_usuarios_y_grupos.html)

* Userdir en Apache
  [https://www.evaristogz.com/configurar-userdir-automatico-apache2/](https://www.evaristogz.com/configurar-userdir-automatico-apache2/)

---

<code>Hecho por Aarón Cano ([rxnwashere](https://github.com/rxnwashere))</code>