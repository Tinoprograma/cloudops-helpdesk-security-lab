# Día 4 — Qué hice y por qué: explicación técnica completa

**Para quién es este documento**: Para vos, que ejecutaste los comandos pero querés entender qué estaba pasando en cada paso. Esto es lo que tenés que poder explicar en una entrevista.

---

## El concepto general antes de arrancar

Instalaste un **servidor web** en la nube. Un servidor web es una computadora (en este caso virtual, corriendo en los datacenters de Amazon) que está permanentemente conectada a internet, esperando que alguien le haga una solicitud HTTP y respondiendo con una página web o una aplicación.

Lo que construiste hoy se llama **stack LAMP**:

| Letra | Componente     | Rol                                                                  |
| ----- | -------------- | -------------------------------------------------------------------- |
| L     | Linux (Ubuntu) | Sistema operativo del servidor                                       |
| A     | Apache         | Servidor web — recibe las solicitudes HTTP                           |
| M     | MariaDB        | Base de datos — guarda los tickets, usuarios, configuración          |
| P     | PHP            | Lenguaje de programación — el código de osTicket está escrito en PHP |

osTicket es una aplicación web escrita en PHP que necesita estos tres servicios corriendo para funcionar. Vos instalaste cada uno manualmente y los conectaste entre sí.

---

## Paso a paso — qué pasó realmente

### "sudo apt update" y "sudo apt upgrade -y"

**¿Qué es apt?**
`apt` es el gestor de paquetes de Ubuntu. Es como una "tienda de software" para Linux que descarga e instala programas desde repositorios oficiales. Cuando instalás algo en Ubuntu, casi siempre usás `apt`.

**¿Qué hizo update?**
Descargó la lista actualizada de paquetes disponibles. No instaló nada — solo actualizó el "catálogo". Es como ir al supermercado y leer la lista de precios antes de comprar.

**¿Qué hizo upgrade?**
Instaló las versiones más nuevas de todo lo que ya tenía Ubuntu instalado. Importante hacerlo primero para tener parches de seguridad al día — en un servidor bancario,esto sería obligatorio antes de cualquier otra cosa.

**¿Qué es sudo?**
`sudo` significa "Super User DO". Es como "ejecutar como administrador" en Windows. En Linux, el usuario normal no puede instalar software ni modificar archivos del sistema. `sudo` te da permisos temporales de root (administrador) para ese comando específico.

---

### Instalar Apache, PHP y extensiones

```bash
sudo apt install -y apache2 php php-mysql php-intl php-apcu php-gd php-xml php-mbstring php-zip libapache2-mod-php
```

**¿Qué hace este comando?**
Instala múltiples paquetes de una sola vez. El `-y` significa "sí a todo" — sin él, apt te preguntaría "¿estás seguro?" para cada instalación.

**¿Qué es cada componente?**

- `apache2`: el servidor web. Una vez instalado, empieza a escuchar en el puerto 80 (HTTP) y responde a cualquier solicitud con una página web.
- `php`: el intérprete de PHP. Cuando Apache recibe una solicitud y el archivo pedido es `.php`, le pasa el archivo a PHP para que lo ejecute y devuelva el resultado al navegador.
- `php-mysql`: extensión que permite a PHP conectarse a bases de datos MySQL/MariaDB. Sin esto, osTicket no podría guardar ni leer datos.
- `php-gd`: permite a PHP manipular imágenes. osTicket lo usa para redimensionar logos y adjuntos.
- `php-xml`, `php-mbstring`, `php-zip`: extensiones para manejar XML, texto unicode y archivos ZIP respectivamente. osTicket las necesita para importar/exportar datos.
- `php-intl`: internacionalización — permite a PHP trabajar con múltiples idiomas y formatos de fecha/número.
- `php-apcu`: sistema de caché en memoria. Hace que PHP sea más rápido guardando resultados de operaciones repetidas.
- `libapache2-mod-php`: el "puente" entre Apache y PHP. Sin este módulo, Apache no sabría pasarle archivos PHP al intérprete.

**¿Por qué falló php-imap?**
`php-imap` es la extensión para leer emails vía protocolo IMAP (lo que usás cuando revisás email en un cliente). En Ubuntu 24.04 este paquete fue removido del repositorio principal. osTicket lo usa para "recibir" tickets por email, pero para el lab no lo necesitamos.

---

### Instalar MariaDB

**¿Qué es MariaDB?**
MariaDB es un sistema de gestión de bases de datos relacionales (RDBMS). Es un fork (bifurcación) de MySQL — técnicamente casi idéntico pero de código abierto. Cuando osTicket necesita guardar un ticket, un usuario, o una configuración, lo escribe en la base de datos MariaDB. Cuando necesita leerlo, hace una consulta SQL.

**¿Qué hizo mariadb-secure-installation?**
Aplicó configuraciones de seguridad básicas:
- **Remove anonymous users**: por defecto MariaDB permite conectarse sin usuario/contraseña para testing. Esto lo elimina.
- **Disallow root login remotely**: el usuario root de la DB solo puede conectarse desde el propio servidor (localhost), no desde internet.
- **Remove test database**: elimina una base de datos de prueba que viene por defecto y que cualquiera podría acceder.
- **Set root password**: protege el acceso al administrador de la base de datos con contraseña.

---

### Crear la base de datos para osTicket

Entraste a MariaDB con `sudo mariadb -u root -p` y ejecutaste:

```sql
CREATE DATABASE osticket;
```
Crea una base de datos vacía llamada "osticket". Es como crear una carpeta nueva donde van a vivir todas las tablas con los datos de la aplicación.

```sql
CREATE USER 'osticket_user'@'localhost' IDENTIFIED BY 'OsTicket2026!';
```
Crea un usuario de base de datos específico para osTicket. El `@'localhost'` significa que este usuario solo puede conectarse desde el propio servidor, no desde internet.

**¿Por qué no usar root?**
Principio de menor privilegio — el mismo que aplicabas en BIND con Active Directory. Si osTicket tiene un bug de seguridad y alguien lo explota, solo va a tener acceso a la base de datos `osticket`, no a todas las bases de datos del servidor.

```sql
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket_user'@'localhost';
```
Le da permisos completos al usuario `osticket_user` pero **solo sobre la base de datos osticket**. El `*` significa "todas las tablas dentro de osticket".

```sql
FLUSH PRIVILEGES;
```
Le dice a MariaDB "recargá los permisos ahora". Los cambios de permisos en MariaDB no son instantáneos — este comando los aplica inmediatamente.

---

### Descargar e instalar osTicket

```bash
cd /tmp
```
Navegás a la carpeta `/tmp` (temporal). En Linux, `/tmp` se usa para archivos temporales que no necesitás conservar. Es el lugar correcto para descargar instaladores.

```bash
wget https://github.com/osTicket/osTicket/releases/download/v1.18.1/osTicket-v1.18.1.zip
```
`wget` es una herramienta de línea de comandos para descargar archivos de internet. Le pasás la URL y descarga el archivo en la carpeta actual. Es el equivalente a hacer click derecho → "Guardar como" en el navegador, pero desde la terminal.

```bash
sudo unzip osTicket-v1.18.1.zip -d /var/www/html/osticket
```
Descomprime el ZIP en `/var/www/html/osticket`. 

**¿Por qué /var/www/html?**
Es la carpeta raíz de Apache. Cuando alguien entra a `http://tu-ip/`, Apache busca los archivos en `/var/www/html/`. Todo lo que pongas ahí es accesible desde el navegador.

```bash
sudo cp /var/www/html/osticket/upload/include/ost-sampleconfig.php \
        /var/www/html/osticket/upload/include/ost-config.php
```
Copia el archivo de configuración de ejemplo como configuración real. osTicket viene con un `ost-sampleconfig.php` de plantilla — necesitás copiarlo con el nombre correcto para que la aplicación lo encuentre.

```bash
sudo chmod 0666 /var/www/html/osticket/upload/include/ost-config.php
```
`chmod` cambia los permisos de un archivo. `0666` significa que cualquier usuario puede leer y escribir ese archivo. Temporalmente necesario para que el installer web pueda escribir la configuración.

**¿Por qué es temporal?**
Dar permisos de escritura a cualquier usuario en un archivo de configuración es un riesgo de seguridad. Por eso al final del día ejecutaste `chmod 0644` para reducir los permisos — solo el dueño puede escribir, los demás solo leer.

```bash
sudo chown -R www-data:www-data /var/www/html/osticket
```
`chown` cambia el dueño de archivos. `www-data` es el usuario del sistema con el que corre Apache. El `-R` significa recursivo — aplica el cambio a todos los archivos y subcarpetas.

**¿Por qué es necesario?**
Apache corre como el usuario `www-data`, no como `ubuntu` (tu usuario). Para que Apache pueda leer los archivos de osTicket, esos archivos tienen que pertenecer a `www-data`. Sin esto, Apache diría "no tengo permiso para leer este archivo" y mostraría un error 403.

---

### Configurar Apache con VirtualHost

```bash
sudo nano /etc/apache2/sites-available/osticket.conf
```
Abrís `nano`, un editor de texto de terminal. Creás un archivo de configuración nuevo en `/etc/apache2/sites-available/` — la carpeta donde Apache guarda las configuraciones de sus "sitios".

**¿Qué es un VirtualHost?**
Un VirtualHost es una configuración que le dice a Apache "cuando alguien entre a esta dirección, servile estos archivos". En un servidor real podés tener múltiples VirtualHosts para múltiples sitios web en la misma máquina.

**¿Qué configuró el VirtualHost que creaste?**

```apache
DocumentRoot /var/www/html/osticket/upload
```
Le dice a Apache dónde están los archivos de osTicket. Cuando alguien entra a `http://tu-ip/`, Apache busca los archivos en esta carpeta.

```apache
AllowOverride All
```
Permite que osTicket use archivos `.htaccess` para sus propias reglas de URL. osTicket los necesita para que las URLs limpias (como `/tickets/123`) funcionen correctamente.

```bash
sudo a2ensite osticket.conf
```
`a2ensite` significa "Apache 2 Enable Site". Activa la configuración que creaste. En Apache, los archivos en `sites-available` son configuraciones disponibles pero no activas. Para activarlas hay que "habilitarlas".

```bash
sudo a2enmod rewrite
```
Activa el módulo `rewrite` de Apache. osTicket lo necesita para transformar URLs amigables en rutas internas del sistema.

```bash
sudo a2dissite 000-default.conf
```
`a2dissite` significa "Apache 2 Disable Site". Desactiva el sitio por defecto de Apache (la página "It works!" que viene preinstalada). Si no lo desactivás, Apache no sabe a cuál de los dos sitios mandar las solicitudes.

```bash
sudo systemctl restart apache2
```
`systemctl` gestiona los servicios del sistema. `restart` detiene y vuelve a iniciar Apache para que aplique todos los cambios de configuración. Como reiniciar un servicio en Windows.

---

### El installer web de osTicket

Cuando entraste a `http://184.73.106.130/setup`, lo que pasó fue:

1. Tu navegador mandó una solicitud HTTP GET a la IP de tu servidor
2. Apache recibió la solicitud y buscó los archivos en `/var/www/html/osticket/upload`
3. Encontró el archivo `setup/index.php` y se lo pasó a PHP
4. PHP ejecutó el código del installer
5. El installer verificó que todos los prerequisitos estuvieran instalados (PHP, extensiones, permisos)
6. Vos completaste el formulario con los datos de la DB y el admin
7. El installer se conectó a MariaDB, creó todas las tablas necesarias, y guardó la configuración en `ost-config.php`
8. osTicket quedó listo para usar

**¿Por qué eliminaste la carpeta /setup al final?**
Una vez instalado, la carpeta `/setup` ya no tiene utilidad. Si la dejás, cualquiera que sepa la URL podría intentar reinstalar osTicket sobre tu instalación existente, borrando todos los datos. Eliminarla es una práctica de seguridad estándar — la misma lógica que deshabilitar endpoints de debug en producción.

---

## El flujo completo en una imagen mental

```
[Tu navegador]
      |
      | HTTP request (puerto 80)
      v
[Apache - escucha en puerto 80]
      |
      | "este archivo es .php, lo proceso con PHP"
      v
[PHP - ejecuta el código de osTicket]
      |
      | "necesito guardar/leer datos"
      v
[MariaDB - base de datos]
      |
      | devuelve datos
      v
[PHP - genera HTML con los datos]
      |
      v
[Apache - envía el HTML al navegador]
      |
      v
[Tu navegador - muestra la página]
```

Cada vez que abrís el panel de osTicket, este ciclo se repite en milisegundos.

---

## Tres conceptos que vas a usar en entrevistas

**1. Principio de menor privilegio aplicado en la DB**
Creaste `osticket_user` con acceso solo a la base de datos `osticket`. Si en una entrevista te preguntan "¿cómo aplicás seguridad en una base de datos?", esta es una respuesta concreta con ejemplo real.

**2. Separación de usuarios del sistema**
Apache corre como `www-data`, no como root. Si Apache tuviera un bug de seguridad y alguien lo explotara, solo tendría acceso al usuario `www-data`, no a todo el sistema. Esto es defensa en profundidad.

**3. El flujo request-response de una aplicación web**
Entender que Apache → PHP → MariaDB es el flujo básico de cualquier aplicación web tradicional. WordPress, Laravel, Symfony, Joomla — todos funcionan igual. Cuando en soporte técnico alguien dice "la web no carga", ya sabés en qué capa puede estar fallando.

---

## Preguntas que podrían hacerte en una entrevista sobre esto

- *"¿Qué es un stack LAMP?"* → Linux + Apache + MySQL/MariaDB + PHP
- *"¿Para qué sirve Apache?"* → Servidor web que recibe solicitudes HTTP y sirve archivos estáticos o los pasa a PHP
- *"¿Por qué usarías un usuario de DB específico para cada aplicación?"* → Principio de menor privilegio — si la app es comprometida, el impacto queda aislado
- *"¿Qué es sudo?"* → Ejecutar un comando con privilegios de administrador temporalmente
- *"¿Qué hace chmod?"* → Cambia los permisos de lectura/escritura/ejecución de un archivo
- *"¿Por qué eliminaste la carpeta /setup?"* → Práctica de seguridad post-instalación para evitar reinstalaciones no autorizadas