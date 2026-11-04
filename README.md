# Monitoreo de Red con Zabbix + SNMP

### Arlene Fernández Herrera · Matrícula: 2025-0730

**Seguridad de Redes · ITLA**

---

## 🎥 Video Demostrativo

**[▶ Ver video de demostración](https://youtu.be/REEMPLAZAR-CON-TU-ID)**

---

## 📋 Tabla de Contenido

1. [Propósito del Laboratorio](#1-propósito-del-laboratorio)
2. [Topología y Direccionamiento](#2-topología-y-direccionamiento)
   - [Diagrama de Topología](#21-diagrama-de-topología)
   - [Tabla de Dispositivos](#22-tabla-de-dispositivos)
   - [Parámetros DHCP](#23-parámetros-dhcp)
   - [Parámetros SNMP](#24-parámetros-snmp)
3. [Parte 1 — Router R1](#3-parte-1--router-r1)
   - [3.1 Configuración de Interfaces](#31-configuración-de-interfaces)
   - [3.2 Servidor DHCP](#32-servidor-dhcp)
   - [3.3 Comunidad SNMP de solo lectura](#33-comunidad-snmp-de-solo-lectura)
4. [Parte 2 — Switch SW1](#4-parte-2--switch-sw1)
   - [4.1 VLAN e IP de Gestión](#41-vlan-e-ip-de-gestión)
   - [4.2 Comunidad SNMP de solo lectura](#42-comunidad-snmp-de-solo-lectura)
5. [Parte 3 — PC Cliente](#5-parte-3--pc-cliente)
6. [Parte 4 — Servidor Zabbix (Ubuntu)](#6-parte-4--servidor-zabbix-ubuntu)
   - [6.1 Máquina virtual y conexión a la red](#61-máquina-virtual-y-conexión-a-la-red)
   - [6.2 Cliente DHCP del servidor Zabbix](#62-cliente-dhcp-del-servidor-zabbix)
   - [6.3 Instalación de Zabbix](#63-instalación-de-zabbix)
   - [6.4 Acceso a la GUI de Zabbix](#64-acceso-a-la-gui-de-zabbix)
   - [6.5 Agregar el Router como Equipo](#65-agregar-el-router-como-equipo)
   - [6.6 Agregar el Switch como Equipo](#66-agregar-el-switch-como-equipo)
   - [6.7 Verificar los datos SNMP](#67-verificar-los-datos-snmp)
7. [Cumplimiento de requisitos](#7-cumplimiento-de-requisitos)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)
9. [Estructura del Repositorio](#9-estructura-del-repositorio)
10. [Referencias](#10-referencias)

---

## 1. Propósito del Laboratorio

Esta topología implementa un sistema de monitoreo centralizado con **Zabbix** sobre una red LAN. El laboratorio muestra cómo un servidor de monitoreo obtiene su dirección IP por **DHCP** y recolecta, mediante **SNMP**, los datos de un Router y un Switch Cisco. Los objetivos específicos son:

* Direccionar el Router y el Switch dentro de la red LAN `10.7.30.0/24`.
* Configurar el Router como **servidor DHCP** de la LAN, para que los equipos de la red obtengan su configuración de forma automática.
* Configurar una **comunidad SNMP de solo lectura** en el Router y en el Switch, limitada al servidor Zabbix.
* Desplegar un servidor **Zabbix 7.0 LTS en Ubuntu** que obtiene su IP como **cliente DHCP**.
* **Verificar en Zabbix** los datos SNMP del Router y del Switch: disponibilidad, información del sistema, interfaces y eventos.

El direccionamiento se deriva de la matrícula `2025-0730`: los últimos cuatro dígitos (`0730`) dan la red privada `10.7.30.0/24`. No se usa la primera IP utilizable de la red: el Router, que es el gateway, usa la segunda (`10.7.30.2`).

---

## 2. Topología y Direccionamiento

### 2.1 Diagrama de Topología

```
                         10.7.30.0/24
                               │
                        e0/0: 10.7.30.2
                       ┌───────┴───────┐
                       │    Router R1  │
                       │  Cisco IOS    │
                       └───────┬───────┘
                          e0/0 │
                       ┌───────┴───────┐
                       │    Switch     │
                       │  Cisco IOS    │
                       └──┬────────┬───┘
                        e1│        │e2
               ┌──────────┘        └──────────┐
               │ e0                        e0 │
       ┌───────┴───────┐           ┌──────────┴───────┐
       │    W10        │           │     ZABBIX       │
       │  (Host Win.)  │           │   10.7.30.253    │
       │  DHCP:        │           │   (Estática)     │
       │  10.7.30.X    │           └──────────────────┘
       └───────────────┘

  Monitoreo SNMP:
  Zabbix (10.7.30.253) ──SNMP v2c──► R1 (10.7.30.2)
  Zabbix (10.7.30.253) ──SNMP v2c──► Switch (10.7.30.3)

  Acceso GUI Zabbix desde W10:
  W10 ──HTTP──► http://10.7.37.253/zabbix
```

Monitoreo SNMP (UDP 161), iniciado siempre por el servidor Zabbix:

```
Zabbix (10.7.30.253) ──SNMP v2c, solo lectura──► R1  (10.7.30.2)
Zabbix (10.7.30.253) ──SNMP v2c, solo lectura──► SW1 (10.7.30.3)
```

### 2.2 Tabla de Dispositivos

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Método | Rol |
|---|---|---|---|---|---|---|
| **R1** | Et0/0 | 10.7.30.2 | /24 | — | Estática | Gateway de la LAN, servidor DHCP y agente SNMP |
| **SW1** | Vlan10 | 10.7.30.3 | /24 | 10.7.30.2 | Estática | Conmutación y agente SNMP |
| **Servidor Zabbix** (Ubuntu) | ens33 | 10.7.30.253 | /24 | 10.7.30.2 | **DHCP** (reserva) | Servidor de monitoreo |
| **PC Cliente** (Windows) | Ethernet | 10.7.30.10 – .200 (rango) | /24 | 10.7.30.2 | **DHCP** | Cliente de prueba y acceso a la GUI de Zabbix |

> El servidor Zabbix tiene un segundo adaptador (`ens33`, red NAT de VMware) que solo se usa para descargar los paquetes durante la instalación. La monitorización se hace por `ens33`, dentro de la LAN.

### 2.3 Parámetros DHCP

| Parámetro | Valor |
|---|---|
| Red | `10.7.30.0/24` |
| Gateway (opción 3) | `10.7.30.2` |
| DNS (opción 6) | `8.8.8.8`, `8.8.4.4` |
| Rango dinámico | `10.7.30.10 – 10.7.30.200` |
| Excluidos | `10.7.30.1 – 10.7.30.9` (infraestructura) y `10.7.30.201 – 10.7.30.254` (reservas) |
| Reserva | `10.7.30.253` para el servidor Zabbix, por identificador de cliente (MAC) |
| Duración de la concesión | 1 día |

### 2.4 Parámetros SNMP

| Parámetro | Valor |
|---|---|
| Versión | SNMPv2c |
| Comunidad | `zbx-ro-0730` |
| Tipo de acceso | Solo lectura (`RO`) |
| Origen permitido | Solo `10.7.30.253` (lista de acceso `SNMP-ZABBIX`) |
| Puerto | UDP 161 |
| Ubicación | `Laboratorio-ITLA-2025-0730` |

> No se configura ninguna comunidad de lectura y escritura. La comunidad se limita al servidor Zabbix con una lista de acceso, de modo que ningún otro equipo de la red puede consultar el Router ni el Switch.

---

## 3. Parte 1 — Router R1

### 3.1 Configuración de Interfaces

```bash
enable
configure terminal

hostname R1
no ip domain-lookup

interface Ethernet0/0
 description RED-LAN
 ip address 10.7.30.2 255.255.255.0
 no shutdown
exit

end
write memory
```

Verificar:

```bash
show ip interface brief
```

> Ver evidencia: [01_router_interfaces.png](screenshots/01_router_interfaces.png)

---

### 3.2 Servidor DHCP

```bash
configure terminal

! Direcciones fuera del rango dinámico
ip dhcp excluded-address 10.7.30.1 10.7.30.9
ip dhcp excluded-address 10.7.30.201 10.7.30.254

! Pool DHCP para la LAN
ip dhcp pool LAN-POOL
 network 10.7.30.0 255.255.255.0
 default-router 10.7.30.2
 dns-server 8.8.8.8 8.8.4.4
 lease 1
exit

end
write memory
```

Verificar:

```bash
show ip dhcp pool
```

> La reserva para el servidor Zabbix se agrega en el apartado 6.2, porque necesita la dirección MAC de su adaptador.

> Ver evidencia: [02_router_dhcp_pool.png](screenshots/02_router_dhcp_pool.png)

---

### 3.3 Comunidad SNMP de solo lectura

```bash
configure terminal

! Solo el servidor Zabbix puede consultar el equipo
ip access-list standard SNMP-ZABBIX
 permit host 10.7.30.253
 deny any log
exit

! Comunidad de solo lectura
snmp-server community zbx-ro-0730 RO SNMP-ZABBIX

! Información del sistema (visible en Zabbix)
snmp-server location Laboratorio-ITLA-2025-0730
snmp-server contact Arlene Fernandez Herrera
snmp-server ifindex persist

end
write memory
```

Verificar:

```bash
show snmp community
show access-lists SNMP-ZABBIX
```

> Ver evidencia: [03_router_snmp.png](screenshots/03_router_snmp.png)
>
> Nota: puede aparecer también una comunidad `ILMI` de solo lectura creada automáticamente por el subsistema ATM del IOS. Es normal en la imagen de IOS usada en el simulador y no es una comunidad configurada; Zabbix no la utiliza.

---

## 4. Parte 2 — Switch SW1

### 4.1 VLAN e IP de Gestión

```bash
enable
configure terminal

hostname SW1
no ip domain-lookup

vlan 10
 name LAN
exit

! Puertos usados: Router, PC Cliente y servidor Zabbix
interface range Ethernet0/0 - 2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit

! IP de gestión
interface Vlan10
 description Gestion
 ip address 10.7.30.3 255.255.255.0
 no shutdown
exit

! Gateway para responder fuera del segmento
ip default-gateway 10.7.30.2

! Puertos sin uso apagados
interface range Ethernet0/3 , Ethernet1/0 - 3
 shutdown
exit

end
write memory
```

Verificar:

```bash
show vlan brief
show ip interface brief
ping 10.7.30.2
```

> Ver evidencia: [04_switch_ip.png](screenshots/04_switch_ip.png)

---

### 4.2 Comunidad SNMP de solo lectura

```bash
configure terminal

ip access-list standard SNMP-ZABBIX
 permit host 10.7.30.253
 deny any log
exit

snmp-server community zbx-ro-0730 RO SNMP-ZABBIX
snmp-server location Laboratorio-ITLA-2025-0730
snmp-server contact Arlene Fernandez Herrera
snmp-server ifindex persist

end
write memory
```

Verificar:

```bash
show snmp community
show access-lists SNMP-ZABBIX
```

> Ver evidencia: [05_switch_snmp.png](screenshots/05_switch_snmp.png)

---

## 5. Parte 3 — PC Cliente

El PC Cliente es un equipo Windows conectado a `Ethernet0/1` del switch. Su adaptador de red usa **DHCP** y desde él se accede a la GUI de Zabbix.

Configurar el adaptador en modo automático (en PowerShell como administrador; ajustar el nombre del adaptador con `Get-NetAdapter`):

```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
ipconfig /renew
ipconfig /all
```

Verificar que el adaptador muestra:

| Campo | Valor esperado |
|---|---|
| Dirección IPv4 | `10.7.30.X` (dentro del rango `.10 – .200`) |
| Máscara | `255.255.255.0` |
| Puerta de enlace | `10.7.30.2` |
| Servidor DHCP | `10.7.30.2` |
| DNS | `8.8.8.8`, `8.8.4.4` |

En el Router, el cliente aparece como una concesión dinámica:

```bash
show ip dhcp binding
```

> Ver evidencia: [08_pc_cliente_dhcp.png](screenshots/08_pc_cliente_dhcp.png)

---

## 6. Parte 4 — Servidor Zabbix (Ubuntu)

### 6.1 Máquina virtual y conexión a la red

El servidor corre sobre **Ubuntu Server 24.04 LTS** en una máquina virtual de VMware.

| Recurso | Valor |
|---|---|
| Sistema | Ubuntu Server 24.04 LTS |
| CPU / RAM | 2 vCPU / 4 GB |
| Adaptador 1 (`ens33`) | NAT (`VMnet8`): salida a Internet para descargar paquetes |
| Adaptador 2 (`ens33`) | Red virtual Host-only sin DHCP de VMware, enlazada a PNETLab con un nodo Cloud hacia `Ethernet0/2` de `SW1` |

Cada máquina virtual entra a la topología por **un solo cable**.

---

### 6.2 Cliente DHCP del servidor Zabbix

El servidor Zabbix obtiene su IP por DHCP. Como Zabbix y las listas de acceso SNMP necesitan una dirección fija, el Router le entrega siempre la misma mediante una **reserva**.

**1. Obtener la dirección MAC de `ens33` (en Ubuntu):**

```bash
ip -br link show ens33
```

Ejemplo de salida: `ens33  UP  00:0c:29:ab:cd:ef`. El identificador de cliente que usa el Router es `01` seguido de la MAC. Para esa MAC es `0100.0c29.abcd.ef`.

**2. Crear la reserva en el Router (con la MAC real):**

```bash
configure terminal

ip dhcp pool ZABBIX-RESERVA
 host 10.7.30.253 255.255.255.0
 client-identifier 0100.0c29.abcd.ef
 default-router 10.7.30.2
 dns-server 8.8.8.8 8.8.4.4
exit

end
write memory
```

**3. Configurar `ens33` como cliente DHCP (en Ubuntu):**

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      dhcp-identifier: mac
      dhcp4-overrides:
        use-routes: false
        use-dns: false
    ens34:
      dhcp4: true
```

* `ens33` pide su dirección al Router. `dhcp-identifier: mac` envía la MAC como identificador, que es lo que busca la reserva. `use-routes: false` evita una segunda ruta por defecto por la LAN.
* `ens34` conserva el acceso a Internet (ruta por defecto y DNS). 

```bash
sudo netplan apply
```

**4. Verificar la concesión:**

En Ubuntu:

```bash
ip -4 addr show ens33
ip route
```

`ens33` debe mostrar `10.7.30.253/24` con la etiqueta `dynamic`, y la ruta por defecto debe seguir saliendo por `ens33`.

En el Router:

```bash
show ip dhcp binding
```

Debe aparecer `10.7.30.253` con el identificador de cliente del servidor y el tipo `Manual`.

> Ver evidencia: [06_zabbix_mac_reserva.png](screenshots/06_zabbix_mac_reserva.png), [07_zabbix_dhcp_cliente.png](screenshots/07_zabbix_dhcp_cliente.png)

---

### 6.3 Instalación de Zabbix

Se instala **Zabbix 7.0 LTS sobre Ubuntu 24.04 LTS**, con MariaDB y Apache (script: [`scripts/zabbix-instalacion.sh`](scripts/zabbix-instalacion.sh)).

```bash
# Repositorio oficial de Zabbix 7.0 LTS para Ubuntu 24.04
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_7.0-2+ubuntu24.04_all.deb
sudo apt update

# Zabbix server, frontend, apache-conf, sql-scripts, agente y herramientas SNMP
sudo apt install zabbix-server-mysql zabbix-frontend-php \
    zabbix-apache-conf zabbix-sql-scripts zabbix-agent snmp -y

# MariaDB
sudo apt install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Crear la base de datos y el usuario:

```bash
sudo mysql -uroot -e "
  CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
  CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZabbixPass123!';
  GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
  SET GLOBAL log_bin_trust_function_creators = 1;
  FLUSH PRIVILEGES;"
```

Importar el esquema y desactivar el indicador temporal:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
    mysql --default-character-set=utf8mb4 -uzabbix -pZabbixPass123! zabbix

sudo mysql -uroot -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

Configurar la contraseña de la base de datos e iniciar los servicios:

```bash
sudo sed -i 's/# DBPassword=/DBPassword=ZabbixPass123!/' /etc/zabbix/zabbix_server.conf

sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

Verificar que el servicio esté corriendo:

```bash
sudo systemctl status zabbix-server
```

> Ver evidencia: [09_zabbix_instalado.png](screenshots/09_zabbix_instalado.png)

---

### 6.4 Acceso a la GUI de Zabbix

Desde el **PC Cliente**, abrir el navegador y entrar a:

```
http://10.7.30.253/zabbix
```

Seguir el asistente de instalación web (`setup.php`):

1. **Bienvenido**
2. **Comprobación de requisitos previos:** todo debe quedar en verde (versión de PHP, `memory_limit`, extensiones, etc.)
3. **Configurar la conexión de BD:** tipo `MySQL`, servidor `localhost`, puerto `0` (por defecto), nombre de BD `zabbix`, usuario `zabbix`, contraseña `ZabbixPass123!`
4. **Ajustes:** nombre del servidor Zabbix, zona horaria `America/Santo_Domingo`, tema
5. **Resumen de preinstalación**
6. **Instalar** y **Finalizar**

**Credenciales por defecto (después de finalizar el asistente):**

| Campo | Valor |
|---|---|
| Usuario | `Admin` (con A mayúscula) |
| Contraseña | `zabbix` |

> Cambiar la contraseña al primer ingreso desde `Configuración de usuario → Contraseña`.

> Ver evidencia: [10_zabbix_setup_requisitos.png](screenshots/10_zabbix_setup_requisitos.png), [11_zabbix_login.png](screenshots/11_zabbix_login.png)

---

### 6.5 Agregar el Router como Equipo

**Ruta GUI:** `Recopilación de datos → Equipos → Crear equipo` (botón arriba a la derecha)

**Pestaña Equipo:**

| Campo | Valor |
|---|---|
| Nombre de host | `R1` |
| Nombre visible | `Router R1 — 10.7.30.2` |
| Grupos | `Network devices` |
| Interfaces → Agregar → SNMP | |
| Dirección IP | `10.7.30.2` |
| Puerto | `161` |
| Versión de SNMP | `SNMPv2` |
| Comunidad SNMP | `{$SNMP_COMMUNITY}` |

**Pestaña Plantillas:**

| Campo | Valor |
|---|---|
| Plantillas | `Cisco IOS by SNMP` (buscar y seleccionar) |

**Pestaña Macros:**

| Macro | Valor |
|---|---|
| `{$SNMP_COMMUNITY}` | `zbx-ro-0730` |

Clic en **Agregar** para guardar.

> Después de unos minutos, el indicador de disponibilidad SNMP del equipo pasa de gris a verde: Zabbix recibió datos SNMP correctamente.

> Ver evidencia: [12_zabbix_host_router.png](screenshots/12_zabbix_host_router.png)

---

### 6.6 Agregar el Switch como Equipo

**Ruta GUI:** `Recopilación de datos → Equipos → Crear equipo`

**Pestaña Equipo:**

| Campo | Valor |
|---|---|
| Nombre de host | `SW1` |
| Nombre visible | `Switch SW1 — 10.7.30.3` |
| Grupos | `Network devices` |
| Interfaces → Agregar → SNMP | |
| Dirección IP | `10.7.30.3` |
| Puerto | `161` |
| Versión de SNMP | `SNMPv2` |
| Comunidad SNMP | `{$SNMP_COMMUNITY}` |

**Pestaña Plantillas:**

| Campo | Valor |
|---|---|
| Plantillas | `Cisco IOS by SNMP` |

**Pestaña Macros:**

| Macro | Valor |
|---|---|
| `{$SNMP_COMMUNITY}` | `zbx-ro-0730` |

Clic en **Agregar** para guardar.

> Ver evidencia: [13_zabbix_host_switch.png](screenshots/13_zabbix_host_switch.png)

---

### 6.7 Verificar los datos SNMP

**Disponibilidad de los equipos:**

**Ruta:** `Recopilación de datos → Equipos`

`R1` y `SW1` deben mostrar el indicador **SNMP** en verde.

> Ver evidencia: [14_zabbix_hosts_disponibles.png](screenshots/14_zabbix_hosts_disponibles.png)

**Datos del Router y del Switch:**

**Ruta:** `Monitorización → Últimos datos` → filtrar por el equipo (`R1`, y después `SW1`)

Deben aparecer métricas SNMP como:

| Métrica | Qué comprueba |
|---|---|
| System description / System name | Información del sistema consultada por SNMP |
| System location | Debe mostrar `Laboratorio-ITLA-2025-0730`, el valor configurado en el equipo |
| Uptime | Tiempo en línea del equipo |
| Interface Et0/0 (Router) o Vlan10 y Et0/x (Switch): Bits received / Bits sent | Tráfico por interfaz |
| Interface: Operational status | Estado de cada interfaz |
| CPU utilization | Carga de CPU, si la imagen la soporta |
| Hardware model name / Hardware serial number | Datos de hardware, si la imagen los expone |

> Algunos elementos de la plantilla pueden quedar sin datos (`Not supported`) en la imagen de IOS del simulador. Los elementos de información del sistema y de interfaces son los que se verifican.

> Ver evidencia: [15_zabbix_datos_router.png](screenshots/15_zabbix_datos_router.png), [16_zabbix_datos_switch.png](screenshots/16_zabbix_datos_switch.png)

**Eventos:**

**Ruta:** `Monitorización → Problemas`

Aquí aparecen las alertas generadas por la plantilla; por ejemplo, si una interfaz monitoreada cambia de estado, Zabbix genera un evento.

> Ver evidencia: [17_zabbix_eventos.png](screenshots/17_zabbix_eventos.png)

**Verificación SNMP desde la línea de comandos del servidor:**

```bash
snmpwalk -v2c -c zbx-ro-0730 10.7.30.2 system
snmpwalk -v2c -c zbx-ro-0730 10.7.30.3 system
snmpwalk -v2c -c zbx-ro-0730 10.7.30.2 ifDescr
```

La salida muestra `sysDescr`, `sysUpTime`, `sysContact` (`Arlene Fernandez Herrera`) y `sysLocation` (`Laboratorio-ITLA-2025-0730`) de cada equipo.

**Comprobación de que la comunidad es de solo lectura:**

```bash
snmpset -v2c -c zbx-ro-0730 10.7.30.2 sysContact.0 s "prueba"
```

El comando debe **fallar** con un error de acceso: la comunidad solo permite lectura.

> Ver evidencia: [18_snmpwalk_router_switch.png](screenshots/18_snmpwalk_router_switch.png), [19_snmpset_solo_lectura.png](screenshots/19_snmpset_solo_lectura.png)

---

## 7. Cumplimiento de requisitos

| Requisito | Cómo se cumple | Sección |
|---|---|---|
| Repositorio de GitHub | Este repositorio | — |
| Documentación profesional en el repositorio | Este `README.md` y la carpeta `screenshots/` | — |
| Video demostrativo al principio del repositorio | Enlace al inicio de este documento | Inicio |
| Documentación con imágenes | Capturas numeradas | 8 |
| Documentación con diagramas | Diagrama de topología | 2.1 |
| Propósito del laboratorio | Sección 1 | 1 |
| Todo script utilizado en el repositorio | Carpeta `scripts/` | 9 |
| Running-configs en el repositorio | Carpeta `running-configs/` | 9 |
| **Router y Switch:** direccionamiento IP | Interfaz `Et0/0` del Router y `Vlan10` del Switch, con la tabla de dispositivos | 2.2, 3.1, 4.1 |
| **Router y Switch:** servidor DHCP para la red LAN | `LAN-POOL` en el Router, con gateway, DNS y rango | 2.3, 3.2 |
| **Router y Switch:** comunidad SNMP de solo lectura | `snmp-server community zbx-ro-0730 RO` en ambos equipos | 2.4, 3.3, 4.2 |
| **Servidor Zabbix:** cliente DHCP | `ens33` con `dhcp4: true` y reserva `10.7.30.253` en el Router | 6.2 |
| **Servidor Zabbix:** verificar los datos SNMP del Router y del Switch | Equipos `R1` y `SW1` con plantilla `Cisco IOS by SNMP`, datos y comandos `snmpwalk` | 6.5, 6.6, 6.7 |

---

## 8. Capturas de Pantalla

Todas las capturas están en la carpeta [`screenshots/`](screenshots/).

| # | Archivo | Descripción |
|---|---|---|
| 01 | [`01_router_interfaces.png`](screenshots/01_router_interfaces.png) | `show ip interface brief` en R1 con `Ethernet0/0: 10.7.30.2` en estado `up/up`. |
| 02 | [`02_router_dhcp_pool.png`](screenshots/02_router_dhcp_pool.png) | `show ip dhcp pool` en R1 con el pool `LAN-POOL`. |
| 03 | [`03_router_snmp.png`](screenshots/03_router_snmp.png) | `show snmp community` y `show access-lists SNMP-ZABBIX` en R1 con la comunidad `zbx-ro-0730` de solo lectura. |
| 04 | [`04_switch_ip.png`](screenshots/04_switch_ip.png) | `show ip interface brief` y `show vlan brief` en SW1 con `Vlan10: 10.7.30.3` en estado `up/up`. |
| 05 | [`05_switch_snmp.png`](screenshots/05_switch_snmp.png) | `show snmp community` en SW1 con la comunidad `zbx-ro-0730` de solo lectura. |
| 06 | [`06_zabbix_mac_reserva.png`](screenshots/06_zabbix_mac_reserva.png) | MAC de `ens33` en Ubuntu y reserva `ZABBIX-RESERVA` en el Router. |
| 07 | [`07_zabbix_dhcp_cliente.png`](screenshots/07_zabbix_dhcp_cliente.png) | `ip -4 addr show ens33` con `10.7.30.253/24` dinámica y `show ip dhcp binding` en el Router. |
| 08 | [`08_pc_cliente_dhcp.png`](screenshots/08_pc_cliente_dhcp.png) | `ipconfig /all` en el PC Cliente con IP del rango `.10 – .200`, gateway y servidor DHCP `10.7.30.2`. |
| 09 | [`09_zabbix_instalado.png`](screenshots/09_zabbix_instalado.png) | `systemctl status zabbix-server` con estado `active (running)`. |
| 10 | [`10_zabbix_setup_requisitos.png`](screenshots/10_zabbix_setup_requisitos.png) | Asistente web de Zabbix, paso de comprobación de requisitos previos, todo en verde. |
| 11 | [`11_zabbix_login.png`](screenshots/11_zabbix_login.png) | Pantalla de inicio de sesión de Zabbix en `http://10.7.30.253/zabbix`. |
| 12 | [`12_zabbix_host_router.png`](screenshots/12_zabbix_host_router.png) | Formulario del equipo `R1` con interfaz SNMP `10.7.30.2:161`, plantilla `Cisco IOS by SNMP` y la macro `{$SNMP_COMMUNITY}`. |
| 13 | [`13_zabbix_host_switch.png`](screenshots/13_zabbix_host_switch.png) | Formulario del equipo `SW1` con interfaz SNMP `10.7.30.3:161`, plantilla `Cisco IOS by SNMP` y la macro `{$SNMP_COMMUNITY}`. |
| 14 | [`14_zabbix_hosts_disponibles.png`](screenshots/14_zabbix_hosts_disponibles.png) | Lista de equipos con `R1` y `SW1` con el indicador SNMP en verde. |
| 15 | [`15_zabbix_datos_router.png`](screenshots/15_zabbix_datos_router.png) | `Monitorización → Últimos datos` de `R1` con información del sistema, tráfico y estado de interfaces. |
| 16 | [`16_zabbix_datos_switch.png`](screenshots/16_zabbix_datos_switch.png) | `Monitorización → Últimos datos` de `SW1` con información del sistema, tráfico y estado de interfaces. |
| 17 | [`17_zabbix_eventos.png`](screenshots/17_zabbix_eventos.png) | `Monitorización → Problemas` con los eventos de los equipos monitoreados. |
| 18 | [`18_snmpwalk_router_switch.png`](screenshots/18_snmpwalk_router_switch.png) | `snmpwalk` desde el servidor hacia R1 y SW1. |
| 19 | [`19_snmpset_solo_lectura.png`](screenshots/19_snmpset_solo_lectura.png) | `snmpset` rechazado: la comunidad es de solo lectura. |

---

## 9. Estructura del Repositorio

```
/
├── README.md                  ← este documento
├── screenshots/               ← capturas numeradas de cada configuración
├── scripts/
│   ├── r1.txt                 ← router: interfaces, DHCP y SNMP
│   ├── sw1.txt                ← switch: VLAN, IP de gestión y SNMP
│   ├── zabbix-netplan.yaml    ← servidor Zabbix: cliente DHCP
│   └── zabbix-instalacion.sh  ← servidor Zabbix: instalación y base de datos
└── running-configs/
    ├── r1-running-config.txt
    └── sw1-running-config.txt
```

---

## 10. Referencias

* Zabbix LLC. *Zabbix 7.0 Documentation: Installation from packages* y *Monitoring with SNMP*. zabbix.com.
* Cisco Systems. *Cisco IOS Network Management Command Reference: SNMP*. cisco.com.
* Cisco Systems. *Cisco IOS IP Addressing Services Command Reference: DHCP*. cisco.com.
* Case, J. et al. (1990). *RFC 1157: A Simple Network Management Protocol (SNMP)*. IETF.
* Droms, R. (1997). *RFC 2131: Dynamic Host Configuration Protocol*. IETF.
