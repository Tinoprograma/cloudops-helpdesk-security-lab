# Day 4 — LAMP Stack Deployment: What I Did and Why

**Intended audience**: Written for self-reference — to be able to explain every command and decision in an interview without notes.

---

## The big picture before starting

What was installed on Day 4 is called a **LAMP stack**:

| Letter | Component      | Role                                                              |
| ------ | -------------- | ----------------------------------------------------------------- |
| L      | Linux (Ubuntu) | Server operating system                                           |
| A      | Apache         | Web server — receives HTTP requests                               |
| M      | MariaDB        | Database — stores tickets, users, configuration                   |
| P      | PHP            | Programming language — osTicket's code is written in PHP          |

osTicket is a web application written in PHP that requires all three services running. Each was installed manually and connected together.

---

## Step by step — what actually happened

### `sudo apt update` and `sudo apt upgrade -y`

**What is apt?**
`apt` is Ubuntu's package manager — like a software store that downloads and installs programs from official repositories.

**What did update do?**
Downloaded the updated list of available packages. Nothing was installed — just the catalog was refreshed.

**What did upgrade do?**
Installed newer versions of everything already on the system. Critical first step to have security patches current — mandatory in a banking server context.

**What is sudo?**
"Super User DO" — like "Run as administrator" on Windows. Gives temporary root (admin) privileges for that specific command.

---

### Installing Apache, PHP and extensions

```bash
sudo apt install -y apache2 php php-mysql php-intl php-apcu php-gd php-xml php-mbstring php-zip libapache2-mod-php
```

The `-y` flag means "yes to all" — without it, apt would ask for confirmation on each package.

**What each component does:**

- `apache2`: the web server. Once installed, it listens on port 80 (HTTP) and responds to any request with a web page.
- `php`: the PHP interpreter. When Apache receives a request for a `.php` file, it passes it to PHP to execute and return the result to the browser.
- `php-mysql`: extension allowing PHP to connect to MySQL/MariaDB databases. Without this, osTicket can't save or read data.
- `php-gd`: allows PHP to manipulate images. osTicket uses it to resize logos and attachments.
- `php-xml`, `php-mbstring`, `php-zip`: extensions for XML, unicode text, and ZIP files. osTicket needs them for import/export.
- `php-intl`: internationalization — allows PHP to handle multiple languages and date/number formats.
- `php-apcu`: in-memory cache system. Makes PHP faster by storing results of repeated operations.
- `libapache2-mod-php`: the "bridge" between Apache and PHP. Without this module, Apache wouldn't know to pass PHP files to the interpreter.

**Why did php-imap fail?**
`php-imap` was removed from the Ubuntu 24.04 main repository. osTicket uses it to receive tickets via email (IMAP), but it's not needed for this lab's use case.

---

### Installing MariaDB

**What is MariaDB?**
A relational database management system (RDBMS). It's a fork of MySQL — technically almost identical but open source. When osTicket needs to save a ticket, user, or configuration, it writes it to MariaDB. When it needs to read it, it makes an SQL query.

**What did mariadb-secure-installation do?**
Applied basic security settings:
- **Remove anonymous users**: by default MariaDB allows connections without credentials for testing. This removes that.
- **Disallow root login remotely**: the DB root user can only connect from the server itself (localhost), not from the internet.
- **Remove test database**: deletes a default test database that anyone could access.
- **Set root password**: protects the DB admin with a password.

---

### Creating the database for osTicket

```sql
CREATE DATABASE osticket;
```
Creates an empty database called "osticket" — like creating a new folder where all application data tables will live.

```sql
CREATE USER 'osticket_user'@'localhost' IDENTIFIED BY 'OsTicket2026!';
```
Creates a database-specific user for osTicket. The `@'localhost'` means this user can only connect from the server itself, not from the internet.

**Why not use root?**
Principle of least privilege — the same principle applied at BIND with Active Directory. If osTicket has a security bug and someone exploits it, they'll only have access to the `osticket` database, not everything on the server.

```sql
GRANT ALL PRIVILEGES ON osticket.* TO 'osticket_user'@'localhost';
```
Gives full permissions to `osticket_user` but **only on the osticket database**. The `*` means "all tables within osticket".

```sql
FLUSH PRIVILEGES;
```
Tells MariaDB to reload permissions immediately. Permission changes aren't instant — this command applies them right away.

---

### Downloading and installing osTicket

```bash
cd /tmp
```
Navigates to the `/tmp` (temporary) folder. In Linux, `/tmp` is used for temporary files that don't need to be kept. The right place for installers.

```bash
wget https://github.com/osTicket/osTicket/releases/download/v1.18.1/osTicket-v1.18.1.zip
```
`wget` is a command-line tool for downloading files from the internet. It's the equivalent of right-click → "Save as" in a browser, but from the terminal.

```bash
sudo unzip osTicket-v1.18.1.zip -d /var/www/html/osticket
```
Unzips into `/var/www/html/osticket`.

**Why /var/www/html?**
This is Apache's root folder. When someone visits `http://your-ip/`, Apache looks for files in `/var/www/html/`. Everything placed there is accessible from a browser.

```bash
sudo cp /var/www/html/osticket/upload/include/ost-sampleconfig.php \
        /var/www/html/osticket/upload/include/ost-config.php
```
Copies the sample config file as the real config. osTicket ships with `ost-sampleconfig.php` as a template — it needs to be copied with the correct name for the application to find it.

```bash
sudo chmod 0666 /var/www/html/osticket/upload/include/ost-config.php
```
`chmod` changes file permissions. `0666` means any user can read and write the file. Temporarily necessary so the web installer can write configuration.

**Why is this temporary?**
Giving write permissions to any user on a config file is a security risk. That's why `chmod 0644` was run at the end of the day — only the owner can write, others can only read.

```bash
sudo chown -R www-data:www-data /var/www/html/osticket
```
`chown` changes file ownership. `www-data` is the system user Apache runs as. The `-R` means recursive — applies the change to all files and subdirectories.

**Why is this necessary?**
Apache runs as `www-data`, not as `ubuntu` (your user). For Apache to read osTicket's files, those files must belong to `www-data`. Without this, Apache returns a 403 error.

---

### Configuring Apache with VirtualHost

```bash
sudo nano /etc/apache2/sites-available/osticket.conf
```
Opens `nano`, a terminal text editor. Creates a new config file in `/etc/apache2/sites-available/` — where Apache stores its "site" configurations.

**What is a VirtualHost?**
A VirtualHost configuration tells Apache "when someone visits this address, serve these files". In a real server you can have multiple VirtualHosts for multiple websites on the same machine.

```apache
DocumentRoot /var/www/html/osticket/upload
```
Tells Apache where osTicket's files are. When someone visits `http://your-ip/`, Apache looks for files in this folder.

```apache
AllowOverride All
```
Allows osTicket to use `.htaccess` files for its own URL rules. osTicket needs these for clean URLs (like `/tickets/123`) to work correctly.

```bash
sudo a2ensite osticket.conf
```
"Apache 2 Enable Site" — activates the configuration just created. Files in `sites-available` are available but not active. They need to be explicitly enabled.

```bash
sudo a2enmod rewrite
```
Activates Apache's `rewrite` module. osTicket needs it to transform friendly URLs into internal system paths.

```bash
sudo a2dissite 000-default.conf
```
"Apache 2 Disable Site" — deactivates Apache's default site (the "It works!" page). Without this, Apache doesn't know which of the two sites to send requests to.

```bash
sudo systemctl restart apache2
```
`systemctl` manages system services. `restart` stops and restarts Apache so all configuration changes take effect. Like restarting a service in Windows.

---

### The osTicket web installer

When visiting `http://[IP]/setup`, this is what happened:

1. The browser sent an HTTP GET request to the server's IP
2. Apache received the request and looked for files in `/var/www/html/osticket/upload`
3. Found `setup/index.php` and passed it to PHP
4. PHP executed the installer code
5. The installer verified all prerequisites were installed (PHP, extensions, permissions)
6. The form was completed with DB and admin credentials
7. The installer connected to MariaDB, created all necessary tables, and saved config to `ost-config.php`
8. osTicket was ready to use

**Why was the /setup folder deleted at the end?**
Once installed, `/setup` has no further use. If left in place, anyone knowing the URL could attempt to reinstall osTicket on the existing installation, wiping all data. Removing it is standard post-installation security practice — same logic as disabling debug endpoints in production.

---

## The complete flow — mental model

```
[Your browser]
      |
      | HTTP request (port 80)
      v
[Apache — listens on port 80]
      |
      | "this file is .php, process with PHP"
      v
[PHP — executes osTicket code]
      |
      | "I need to save/read data"
      v
[MariaDB — database]
      |
      | returns data
      v
[PHP — generates HTML with the data]
      |
      v
[Apache — sends HTML to the browser]
      |
      v
[Your browser — displays the page]
```

Every time the osTicket panel is opened, this cycle repeats in milliseconds.

---

## Three concepts for interviews

**1. Least privilege applied to the DB**
`osticket_user` was created with access only to the `osticket` database. If asked "how do you apply security to a database?", this is a concrete answer with a real example.

**2. System user separation**
Apache runs as `www-data`, not as root. If Apache had a security bug and someone exploited it, they'd only have access to the `www-data` user, not the entire system. This is defense in depth.

**3. The request-response flow of a web application**
Apache → PHP → MariaDB is the basic flow of any traditional web application. WordPress, Laravel, Symfony, Joomla — they all work the same way. When in tech support someone says "the site won't load", you already know which layer to check.

---

## Interview questions this document prepares you for

- *"What is a LAMP stack?"* → Linux + Apache + MySQL/MariaDB + PHP
- *"What does Apache do?"* → Web server that receives HTTP requests and serves static files or passes them to PHP
- *"Why would you use a dedicated DB user per application?"* → Least privilege — if the app is compromised, impact is isolated
- *"What is sudo?"* → Execute a command with temporary admin privileges
- *"What does chmod do?"* → Changes read/write/execute permissions on a file
- *"Why did you delete the /setup folder?"* → Post-installation security practice to prevent unauthorized reinstallation