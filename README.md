# Laboratorio: Implementación de Infraestructura AAA con RADIUS y Active Directory

> **📺 Video Demostrativo**
> [Haz clic aquí para ver el video demostrativo de la práctica] https://youtu.be/Ha1q7vksv3I

## 📌 Propósito del Laboratorio

<img width="601" height="561" alt="imagen" src="https://github.com/user-attachments/assets/1e41bfea-30ed-4eaa-afeb-a905451f0061" />


El propósito principal de este laboratorio es diseñar, configurar y validar una infraestructura de red segura implementando el modelo **AAA (Autenticación, Autorización y Contabilización)**. Se utiliza un entorno de red emulado en GNS3 integrado con un servidor Windows Server real. El objetivo es centralizar la administración de accesos de los dispositivos de red (Routers y Switches) mediante un servidor **RADIUS (Network Policy Server - NPS)** apoyado por **Active Directory**, asignando niveles de privilegios específicos (0, 5 y 10) basados en la pertenencia a grupos de seguridad, y garantizando la trazabilidad mediante logs.

## 🏗️ Topología de Red y Arquitectura

La infraestructura está compuesta por un entorno híbrido (GNS3 y Virtualización Host):

* **1 Router (R1):** Puerta de enlace de la topología y cliente RADIUS.

* **1 Switch (SW1):** Dispositivo de acceso de capa 2 y cliente RADIUS.

* **1 Windows Server:** Actúa como Controlador de Dominio (Active Directory) y Servidor RADIUS (NPS). Integrado a GNS3 mediante un nodo Cloud.

* <img width="601" height="561" alt="imagen" src="https://github.com/user-attachments/assets/d7ddcc44-e725-4822-8ea5-7f2898633b4f" />


## 🖧 Tabla de Direccionamiento IP

El direccionamiento fue diseñado utilizando la base **10.20.24.0/24**, correspondiente a la matrícula `20240886`.

| Dispositivo | Interfaz / Rol | Dirección IP | Máscara | Gateway | 
| ----- | ----- | ----- | ----- | ----- | 
| **Router (R1)** | FastEthernet0/0 (LAN) | `10.20.24.1` | `255.255.255.0` | N/A | 
| **Switch (SW1)** | VLAN 1 (Management) | `10.20.24.2` | `255.255.255.0` | `10.20.24.1` | 
| **Windows Server** | AD / NPS (Ethernet) | `10.20.24.88` | `255.255.255.0` | `10.20.24.1` | 

## 🛠️ Diario de Configuración y Troubleshooting Operativo

Durante la implementación, se presentaron y resolvieron los siguientes retos técnicos:

1. **Troubleshooting de Nombres de Roles en PowerShell:**

   * **Incidente:** Al intentar instalar el rol de NPS vía PowerShell usando el nombre heredado (`NPAS`), el sistema arrojó un error de nombre inválido.

   * **Solución:** Se utilizó el Server Manager gráfico para sortear la diferencia de versiones en las características internas de Windows Server, logrando la instalación e integración exitosa con AD.

2. **Troubleshooting de Cifrado SSH (Cipher Mismatch):**

   * **Incidente:** Al probar la conexión SSH desde el Switch (SW1) hacia el Router (R1), se presentó el error `%SSH-3-NO_MATCH: No matching cipher found`.

   * **Solución:** Se identificó que la imagen de IOS del cliente solicitaba cifrados en bloque `ctr`, mientras el servidor soportaba `cbc`. Se resolvió forzando el algoritmo de encriptación en el cliente ejecutando `ssh -c aes256-cbc -l user10 10.20.24.1`.

3. **Asignación de IP Estática en Windows Server:**

   * **Incidente:** El adaptador de red tomaba configuración DHCP por defecto del hipervisor (VMware/VirtualBox).

   * **Solución:** Se desactivó el DHCP por PowerShell y se forzó la IP estática `10.20.24.88/24` apuntando el DNS al `localhost` (127.0.0.1) para el correcto funcionamiento de Active Directory.

## ⚙️ Configuraciones Implementadas Paso a Paso

1. **Preparación de la Red Base:** Configuración de interfaces en R1 (FastEthernet0/0) y SW1 (VLAN 1) garantizando la conectividad ICMP (Ping) hacia la Nube (Windows Server).

2. **Despliegue de Active Directory:** Promoción del servidor a Controlador de Dominio (`lab.local`) y creación de 3 grupos de seguridad: `Nivel_0`, `Nivel_5`, y `Nivel_10`.

3. **Implementación de NPS:** Instalación del rol, registro en el AD y configuración de R1 y SW1 como clientes RADIUS con la clave secreta precompartida.

4. **Configuración de Políticas de Red (NPS):** Creación de reglas vinculando los grupos de Windows con los Vendor-Specific Attributes (VSA) de Cisco (`shell:priv-lvl=X`).

5. **Aseguramiento AAA en Cisco:** Habilitación de `aaa new-model`, definición de grupos RADIUS con fallback a base local, y aplicación estricta a los puertos Consola, Auxiliar y VTY.

## 🔒 Políticas de Seguridad

Aunque no se desplegaron firewalls perimetrales, las políticas de seguridad de control de acceso implementadas son:

* **Autenticación Estricta Unicast:** Solo los puertos de administración (Console, Auxiliar y VTY/SSH) solicitan credenciales.

* **Fallback Local:** Si el servidor RADIUS cae, el acceso recae en el usuario local `admin` cifrado con un hash `secret` de nivel 15.

* **Control de Privilegios Basado en Roles (RBAC):**

  * Usuarios Nivel 10 y 5: Tienen acceso administrativo restringido según las capacidades del IOS de Cisco.

  * Usuarios Nivel 0: Restricción casi total, solo pueden ejecutar comandos base (`enable`, `exit`).

* **Protocolo Unencrypted PAP:** Configurado exclusivamente entre NAS (Cisco) y RADIUS para soportar las consultas nativas del IOS.

## 🧪 Validación y Pruebas Analíticas

Para garantizar la correcta ejecución de las políticas, se realizaron las siguientes pruebas:

1. **Prueba de Conectividad (Ping):** Exitosa entre las interfaces `10.20.24.1`, `.2` y `.88`.

2. **Validación de Logs (Accounting):** Revisión de la ruta `C:\Windows\System32\LogFiles` en Windows Server comprobando el registro de los inicios de sesión (Start-Stop).

3. **Prueba de Autenticación y Autorización (SSH):**

   * Inicio de sesión desde SW1 hacia R1 con usuario `user10`.

   * Verificación de permisos ejecutando `show privilege`.

   * **Resultado analítico:** El servidor RADIUS interceptó correctamente la solicitud, consultó el Active Directory, validó la contraseña, verificó la membresía del grupo e inyectó el atributo `priv-lvl=10`.

## 📸 Diagramas y Evidencias Visuales


*Figura 1: Pruebas de conectividad ICMP entre equipos.*

<img width="909" height="598" alt="imagen" src="https://github.com/user-attachments/assets/12b85ccc-2386-4b63-8f3f-8bbb7a7fa3a6" />


*Figura 2: Políticas de red y Atributos de Cisco (VSA) en NPS.*

<img width="1224" height="844" alt="imagen" src="https://github.com/user-attachments/assets/ab46a575-e954-47af-9bcb-e70fde094d6b" />


*Figura 3: Conexión SSH exitosa y validación del nivel de privilegios con `show privilege`.*

<img width="907" height="595" alt="imagen" src="https://github.com/user-attachments/assets/fbffd951-8e62-42c1-8108-6a6b95986a28" />


*Figura 4: Registros de Accounting generados por el Servidor RADIUS.*

<img width="1159" height="790" alt="imagen" src="https://github.com/user-attachments/assets/7809b269-8264-4bfb-8612-85397a11351b" />


## 📜 Scripts y Archivos Utilizados

Se utilizaron los siguientes comandos de PowerShell automatizados en el Servidor Windows:

```
# 1. Configuración de IP Estática
Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.20.24.88 -PrefixLength 24 -DefaultGateway 10.20.24.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1

# 2. Despliegue de Active Directory
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local" -InstallDns -Force

# 3. Creación de Grupos de Seguridad
New-ADGroup -Name "Nivel_0" -GroupCategory Security -GroupScope Global
New-ADGroup -Name "Nivel_5" -GroupCategory Security -GroupScope Global
New-ADGroup -Name "Nivel_10" -GroupCategory Security -GroupScope Global

```

## 📄 Running Configurations Completas

*(Puedes guardar las configuraciones de R1 y SW1 en archivos de texto dentro de tu repo y enlazarlas, o pegarlas directamente aquí. A continuación el código principal implementado).*

### R1 (Router) - Fragmento AAA

```
hostname R1
ip domain-name lab.local
username admin privilege 15 secret 5 $1$mERr$hx5rVt7rPNoS4wqbXKX7m0
!
aaa new-model
aaa authentication login LISTA_ACCESO group radius local
aaa authorization exec LISTA_ACCESO group radius local
aaa accounting auth-proxy default start-stop group radius
aaa accounting exec default start-stop group radius
aaa accounting commands 15 default start-stop group radius
!
radius server NPS-SERVER
 address ipv4 10.20.24.88 auth-port 1812 acct-port 1813
 key RadiusSecret2024
!
line con 0
 login authentication LISTA_ACCESO
 authorization exec LISTA_ACCESO
line aux 0
 login authentication LISTA_ACCESO
 authorization exec LISTA_ACCESO
line vty 0 4
 login authentication LISTA_ACCESO
 authorization exec LISTA_ACCESO
 transport input ssh

```

### SW1 (Switch) - Fragmento AAA

```
hostname SW1
ip domain-name lab.local
!
aaa new-model
aaa authentication login LISTA_ACCESO group radius local
aaa authorization exec LISTA_ACCESO group radius local
aaa accounting exec default start-stop group radius
aaa accounting commands 15 default start-stop group radius
!
radius server NPS-SERVER
 address ipv4 10.20.24.88 auth-port 1812 acct-port 1813
 key RadiusSecret2024
!
line con 0
 login authentication LISTA_ACCESO
 authorization exec LISTA_ACCESO
line vty 0 4
 login authentication LISTA_ACCESO
 authorization exec LISTA_ACCESO
 transport input ssh

```

## 🎯 Conclusión

La realización de este laboratorio demostró de manera práctica la importancia de descentralizar la autenticación en las redes empresariales. La implementación de Active Directory junto con NPS (RADIUS) permitió gestionar el acceso de forma granular a los equipos de infraestructura, aplicando el principio de mínimo privilegio. Adicionalmente, el troubleshooting aplicado en escenarios de incompatibilidad de algoritmos de cifrado (SSH) y despliegue de roles de servidor, fortaleció la capacidad de análisis y resolución de problemas operativos.
