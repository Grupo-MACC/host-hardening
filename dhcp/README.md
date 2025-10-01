# Host Hardening para Servidor DHCP (ISC DHCP Server en Linux)

Este documento describe los pasos recomendados para fortalecer la seguridad de un servidor DHCP en Linux (ejemplo: Debian/Ubuntu con `isc-dhcp-server`).  

---

## 1. Sistema Operativo

### 1.1. Actualizar sistema
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2. Instalar solo lo necesario
```bash
sudo apt install isc-dhcp-server -y
sudo apt remove telnet ftp rsh-server -y
```

### 1.3. Permisos y usuarios
```bash
ps aux | grep dhcpd
sudo chown root:root /etc/dhcp/dhcpd.conf
sudo chmod 640 /etc/dhcp/dhcpd.conf
```

### 1.4. Desactivar servicios innecesarios
```bash
systemctl list-unit-files --type=service --state=enabled
sudo systemctl disable telnet.socket
sudo systemctl disable rlogin.service
```

---

## 2. Red y Acceso

### 2.1. Firewall (ufw)
```bash
sudo ufw allow 67/udp
sudo ufw allow 68/udp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

### 2.2. DHCP Snooping (en switch, no en servidor)
```text
ip dhcp snooping
ip dhcp snooping vlan 10
interface GigabitEthernet0/1
  ip dhcp snooping trust
```

### 2.3. SSH seguro  
Editar `/etc/ssh/sshd_config`:
```conf
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
AllowUsers adminuser
```
Reiniciar:
```bash
sudo systemctl restart ssh
```

---

## 3. Configuración del servicio DHCP

## 3.1. Verificar interfaces disponibles
```bash
ip addr
```
Esto te mostrará las interfaces reales. En distribuciones recientes puede ser algo como enp0s3, ens33, enp1s0, etc.
```text
1: lo: <LOOPBACK,UP> ...
2: enp0s3: <BROADCAST,MULTICAST,UP> ...
```

## 3.2. Edita /etc/default/isc-dhcp-server
```bash
sudo nano /etc/default/isc-dhcp-server
```
Cambia la linea:
```conf
INTERFACESv4=""
```
Por tu interfaz real, por ejemplo:
```conf
INTERFACESv4="enp0s3"
```

## 3.3. Asegúrate de declarar la subred correcta en dhcpd.conf
Archivo principal: `/etc/dhcp/dhcpd.conf`
```conf
subnet 10.0.2.0 netmask 255.255.255.0 {
  range 10.0.2.100 10.0.2.200;
  option routers 10.0.2.15;
  option domain-name-servers 8.8.8.8;
  default-lease-time 600;
  max-lease-time 7200;

  host equipo1 {
    hardware ethernet 00:11:22:33:44:55;
    fixed-address 10.0.2.150;
  }
}
```

Validar configuración:
```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```
Ahora debería arrancar correctamente en la interfaz correcta.

Ver logs:
```bash
tail -f /var/log/syslog | grep dhcpd
```

---

## 4. Monitoreo y Respuesta

### 4.1. Logs centralizados
Editar `/etc/rsyslog.conf`:
```conf
*.* @192.168.10.5:514
```
Reiniciar:
```bash
sudo systemctl restart rsyslog
```

### 4.2. Detectar rogue DHCP
```bash
sudo apt install dhcping
dhcping -s 192.168.10.1
sudo nmap --script broadcast-dhcp-discover -e eth0
```

---

## 5. Copias de seguridad

Respaldar configuración y leases:
```bash
sudo cp /etc/dhcp/dhcpd.conf /backup/dhcpd.conf.$(date +%F)
sudo cp /var/lib/dhcp/dhcpd.leases /backup/dhcpd.leases.$(date +%F)
```

Automatizar con cron (`sudo crontab -e`):
```cron
0 2 * * * cp /etc/dhcp/dhcpd.conf /backup/dhcpd.conf.$(date +\%F)
```

---



