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

Se aplica una configuración segura para proteger el servidor web, siguiendo las recomendaciones del benchmark CIS.

### 2.1. Generar Certificado SSL/TLS Autofirmado
- Se crea un certificado y una clave privada para habilitar las conexiones seguras HTTPS.
    ```bash
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/nginx-selfsigned.key \
    -out /etc/ssl/certs/nginx-selfsigned.crt
    ```

### 2.2. Aplicar Configuración Segura del Sitio
- Se edita el fichero `/etc/nginx/sites-available/default` y se reemplaza su contenido con la configuración final, que incluye un bloque `default_server` para capturar hosts desconocidos y directivas de seguridad Nivel 1 y 2 de CIS.
    ```nginx
    # (CIS 2.4.2) Bloque para capturar peticiones con cabecera Host inválida o desconocida
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;

        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
        
        server_name _;
        return 404;
    }

    # BLOQUE 1: Redirección de HTTP (80) a HTTPS (443) para el sitio válido
    server {
        listen 80;
        listen [::]:80;
        
        # Se especifica el nombre del servidor para que responda solo a peticiones válidas
        server_name 127.0.0.1 localhost;
        
        return 301 https://$host:8443$request_uri;
    }

    # BLOQUE 2: Servidor principal seguro (HTTPS) para el sitio válido
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        
        # Se especifica el nombre del servidor para que responda solo a peticiones válidas
        server_name 127.0.0.1 localhost;

        # --- Ficheros del Certificado SSL ---
        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

        # --- Hardening SSL/TLS Avanzado (CIS 4.1.4, 4.1.5, 4.1.6, 4.1.12) ---
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_dhparam /etc/nginx/dhparam.pem;
        ssl_session_tickets off;

        # --- Hardening: Cabeceras de Seguridad Avanzadas (CIS 5.3.x, 4.1.8) ---
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;
        add_header Content-Security-Policy "default-src 'self'" always;
        add_header Referrer-Policy "no-referrer" always;

        # --- Configuración del Servidor ---
        root /var/www/html;
        index index.html index.htm;

        # --- Hardening Avanzado: Timeouts y Límites (CIS 2.4.3, 5.2.x) ---
        keepalive_timeout 10;
        client_body_timeout 10;
        client_header_timeout 10;
        client_max_body_size 100k;
        large_client_header_buffers 2 1k;
        limit_conn limitperip 10;
        limit_req zone=ratelimit burst=10 nodelay;

        # (CIS 2.5.3) Denegar acceso a ficheros ocultos
        location ~ /\. {
            deny all;
        }

        location / {
            try_files $uri $uri/ =404;
        }

        # --- Hardening: Limitar Métodos HTTP (CIS 5.1.2) ---
        if ($request_method !~ ^(GET|HEAD|POST)$) {
            return 405;
        }
    }
    ```

### 2.3. Ocultar Versión de Nginx y Configurar Límites Globales
- Para evitar revelar información y definir zonas de limitación, se modifica el fichero `/etc/nginx/nginx.conf`.
    ```bash
    # Para editar el fichero:
    sudo nano /etc/nginx/nginx.conf
    ```
- Se añade o descomenta `server_tokens off;` y se añaden las directivas `limit_conn_zone` y `limit_req_zone` dentro del bloque `http { ... }`.
    ```nginx
    http {
        # ... otras directivas http ...

        # Hardening Básico: Ocultar Versión (CIS 2.5.1)
        server_tokens off;

        # Hardening Avanzado: Límites de Conexión y Peticiones (CIS 5.2.4, 5.2.5)
        limit_conn_zone $binary_remote_addr zone=limitperip:10m;
        limit_req_zone $binary_remote_addr zone=ratelimit:10m rate=5r/s;

        # ... resto de la configuración http ...
    }
    ```

### 2.4. Aplicar todos los cambios de Nginx
- Se comprueba la sintaxis y se reinicia el servicio para aplicar toda la configuración.
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
    Para resolver un error `403 Forbidden`, se asigna la propiedad de los ficheros web al usuario del servidor (`www-data`) y se aplican los permisos mínimos necesarios.
    ```bash
    sudo chown -R www-data:www-data /var/www/html
    sudo find /var/www/html -type d -exec chmod 755 {} \;
    sudo find /var/www/html -type f -exec chmod 644 {} \;
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

### 3.4. Generar Parámetros Diffie-Hellman (CIS 4.1.6)
- Se generan parámetros DH de 2048 bits para reforzar el intercambio de claves.
    ```bash
    sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
    ```

### 3.5. Asegurar Permisos de Ficheros Críticos (CIS 4.1.3, 2.3.3)
- Se aplican permisos explícitos a la clave privada SSL y al fichero de proceso (PID) de Nginx.
    ```bash
    sudo chmod 400 /etc/ssl/private/nginx-selfsigned.key
    sudo chown root:root /var/run/nginx.pid
    sudo chmod 644 /var/run/nginx.pid
    ```

---

## 4. Anexo: Configuración de Red para VM (VirtualBox)

Durante la puesta en marcha, se detectó un problema de conectividad entre el PC anfitrión y la máquina virtual (VM). El problema se solucionó configurando el **Reenvío de Puertos (Port Forwarding)**.

- **Modo de Red de la VM:** NAT
- **Reglas de Reenvío:**
    - **HTTP:** Puerto anfitrión `8080` -> Puerto invitado `80`
    - **HTTPS:** Puerto anfitrión `8443` -> Puerto invitado `443`
