# NGINX Configuration Guide

This document provides instructions for setting up and configuring NGINX with a focus on initial setup, security hardening, and system-level protection.

---

## 1. Instalación

- **Instalar NGINX**
    Se actualizan los repositorios y se instala el paquete de Nginx.
    ```bash
    sudo apt update
    sudo apt install nginx -y
    ```

- **Verificar el estado del servicio**
    Se comprueba que el servicio Nginx se haya instalado y se esté ejecutando correctamente.
    ```bash
    sudo systemctl status nginx
    ```
---

## 2. Configuración de Seguridad de Nginx

Se aplica una configuración segura para proteger el servidor web.

### 2.1. Generar Certificado SSL/TLS Autofirmado
- Se crea un certificado y una clave privada para habilitar las conexiones seguras HTTPS.
    ```bash
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/nginx-selfsigned.key \
    -out /etc/ssl/certs/nginx-selfsigned.crt
    ```

### 2.2. Aplicar Configuración Segura del Sitio
- Se edita el fichero `/etc/nginx/sites-available/default` y se reemplaza su contenido con el siguiente bloque, que fuerza HTTPS, añade cabeceras de seguridad y usa protocolos de cifrado robustos.
    ```nginx
    # BLOQUE 1: Redirección de HTTP (80) a HTTPS (443)
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        return 301 https://$host:8443$request_uri;
    }

    # BLOQUE 2: Servidor principal seguro (HTTPS)
    server {
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;

        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';

        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;

        root /var/www/html;
        index index.html index.htm;
        server_name _;

        location / {
            try_files $uri $uri/ =404;
        }

        if ($request_method !~ ^(GET|HEAD|POST)$) {
            return 405;
        }
    }
    ```

### 2.3. Ocultar Versión de Nginx
- Para evitar revelar información a posibles atacantes, se modifica el fichero `/etc/nginx/nginx.conf` para añadir o descomentar la siguiente directiva.
    ```nginx
    server_tokens off;
    ```
- **Aplicar todos los cambios de Nginx**
    Se comprueba la sintaxis y se reinicia el servicio para aplicar la configuración.
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

---

## 3. Hardening del Sistema Operativo

Se aplican medidas de seguridad adicionales a nivel de sistema para proteger el entorno del servidor.

### 3.1. Asegurar Permisos de Ficheros
- **Proteger configuración de Nginx:**
    Se asegura que solo el usuario `root` pueda modificar los ficheros de configuración.
    ```bash
    sudo chown -R root:root /etc/nginx
    sudo find /etc/nginx -type d -exec chmod 755 {} \;
    sudo find /etc/nginx -type f -exec chmod 644 {} \;
    ```
- **Proteger ficheros de la web:**
    Se cambia la propiedad de los archivos web al usuario administrador para que el proceso de Nginx (`www-data`) solo pueda leerlos.
    ```bash
    sudo chown -R $USER:$USER /var/www/html
    sudo chmod -R 755 /var/www/html
    ```

### 3.2. Instalar Defensa Proactiva con Fail2ban
- Se instala `Fail2ban` para monitorizar los logs y bloquear automáticamente IPs con comportamiento malicioso.
    ```bash
    sudo apt install fail2ban -y
    sudo systemctl enable fail2ban
    sudo systemctl start fail2ban
    ```

### 3.3. Aplicar Hardening del Kernel
- Se edita el fichero `/etc/sysctl.conf` para añadir reglas que refuercen la pila de red contra ataques comunes.
    ```bash
    # Para editar el fichero:
    sudo nano /etc/sysctl.conf
    ```
- Se añaden las siguientes líneas al final del fichero:
    ```ini
    # --- Hardening de Red ---
    net.ipv4.tcp_syncookies = 1
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.default.rp_filter = 1
    net.ipv4.conf.all.accept_redirects = 0
    net.ipv6.conf.all.accept_redirects = 0
    net.ipv4.conf.all.secure_redirects = 0
    ```
- Se aplican los cambios al kernel sin necesidad de reiniciar.
    ```bash
    sudo sysctl -p
    ```
---

## 4. Anexo: Configuración de Red para VM (VirtualBox)

Durante la puesta en marcha, se detectó un problema de conectividad entre el PC anfitrión y la máquina virtual (VM) que impedía el acceso al servidor web. El problema se solucionó configurando el **Reenvío de Puertos (Port Forwarding)** en el software de virtualización.

- **Modo de Red de la VM:** NAT
- **Reglas de Reenvío:**
    - **HTTP:** Puerto anfitrión `8080` -> Puerto invitado `80`
    - **HTTPS:** Puerto anfitrión `8443` -> Puerto invitado `443`
