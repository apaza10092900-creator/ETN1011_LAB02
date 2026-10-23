# Informe de Laboratorio 2: Implementación y Diagnóstico de FRRouting en WSL/Ubuntu

**Estudiante:** Apaza Queso Gaston Jhonny

**Fecha:** 23 de octubre de 2026

**Materia:** ETN-1011 sistemas de comunicacion II Laboratorio

**Entorno:** Windows Subsystem for Linux (WSL2) - Ubuntu

**Software:** FRRouting (FRR), iproute2, Cisco IOS (referencia)

---

## 1. Objetivo
El presente laboratorio 2 tiene como finalidad instalar y configurar un router basado en software (FRR) dentro de un entorno WSL2/Ubuntu. Se busca practicar la configuración de interfaces virtuales, rutas conectadas, rutas estáticas de descarte (blackhole), la persistencia de configuración y el diagnóstico de red, estableciendo paralelismos conceptuales con la sintaxis de Cisco IOS.

## 2. Topología
El laboratorio se realiza con un único router lógico (R1) sobre Linux:
*   **Interfaz `eth0`:** Interfaz de WSL (protegida, no se modificó su ruta por defecto).
*   **Interfaz `dummy0`:** Interfaz virtual de laboratorio (IP: `10.10.10.1/24`).

---

## 3. Desarrollo del Laboratorio

### 3.1. Preparación e Instalación
1. Se verificó el entorno WSL2 y la distribución Ubuntu, comprobando que mi usuario contaba con privilegios `sudo`.
2. Se identificó la interfaz `eth0` y la ruta `default` mediante `ip -br addr` e `ip -4 route`, garantizando no interferir con la conectividad de WSL.
3. Se instaló FRR y sus dependencias mediante:
   ```bash
   `sudo apt update`
   `sudo apt install -y frr frr-pythontools iproute2 iputils-ping`

### 3.2. Configuración de Daemons (Observación Técnica)
Al editar /etc/frr/daemons, se observó una diferencia respecto a la guía proporcionada. En las versiones modernas de FRR, los daemons zebra y staticd ya no aparecen explícitamente en el archivo porque se inician automáticamente por defecto (como indica el propio archivo en sus comentarios).

Se mantuvieron desactivados (no) los protocolos dinámicos (ospfd, bgpd).

Se habilitó el servicio con systemd:
   
   `sudo systemctl enable --now frr`

   `sudo systemctl restart frr`

### 3.3. Entorno CLI y Configuración de Interfaz

Se accedió a la consola unificada con sudo vtysh, explorando los comandos show version, show interface y show ip route.

Se creó y configuró la interfaz virtual desde Bash (plano de control de Linux):

`sudo ip link add dummy0 type dummy`  
`sudo ip addr add 10.10.10.1/24 dev dummy0`  
`sudo ip link set dummy0 up`

Resultado: Linux generó automáticamente la ruta conectada 10.10.10.0/24.

### 3.4. Enrutamiento Estático y Blackhole
Desde el modo de configuración de FRR (configure terminal), se añadió una ruta estática de descarte:
ip route 198.51.100.0/24 blackhole

Se verificó la configuración activa con `show running-config` y se procedió a guardarla en el disco mediante:

`write memory`

Se comprobó la creación del archivo /etc/frr/frr.conf exitosamente.

### 3.5. Diagnóstico y Pruebas de Descarte
1. Se utilizó `ip route get 198.51.100.10`, confirmando que el kernel resolvía el destino utilizando la ruta blackhole.

2. Se ejecutó un `ping -c 2 198.51.100.10`, observando que los paquetes ICMP eran descartados sin respuesta (comportamiento esperado).

3. Se revisaron los logs del servicio con `journalctl -u frr -n 50` para verificar el estado del demonio sin errores.

### 3.6. Preparación de OSPF
Se habilitó temporalmente el daemon ospfd modificando /etc/frr/daemons y reiniciando el servicio. En vtysh se configuró:

`router ospf`
`network 10.10.10.0/24` area 0

Al consultar show ip ospf neighbor, el resultado fue cero vecinos, lo que es correcto al tratarse de un entorno de un solo router sin otra instancia OSPF conectada.

### 4. Conclusiones

1. Separación de Planos: Se comprendió la diferencia fundamental entre Linux y FRR. Linux (mediante iproute2) gestiona el plano de datos (interfaces, asignación de IPs, kernel forwarding), mientras que FRR actúa en el plano de control (cálculo de rutas, protocolos) y le indica a Linux qué rutas instalar.

2. Integración: El daemon zebra es el puente vital que sincroniza las rutas calculadas por FRR con la tabla de enrutamiento del kernel Linux.

3. Persistencia: Los comandos de Linux no se guardan con write memory de FRR; la configuración de las interfaces creadas con ip link es volátil en WSL. Solo la configuración de protocolos de FRR se persiste en /etc/frr/frr.conf.

4. Equivalencias: Se logró mapear exitosamente la sintaxis de FRR con Cisco IOS (ej. blackhole vs Null0, write memory vs copy running-config startup-config).

### 5. Respuestas a las Preguntas de Evaluación

¿Qué diferencia hay entre FRR y el kernel Linux?

FRR calcula y administra las rutas (plano de control). El kernel de Linux realiza el reenvío real de los paquetes (plano de datos) y mantiene su propia tabla de rutas.

¿Qué función cumple zebra?

Es el daemon central de FRR que coordina las interfaces y las rutas, sincronizando la información entre los distintos protocolos de FRR y el kernel del sistema operativo.

¿Qué diferencia hay entre show ip route e ip -4 route?

show ip route (en vtysh) muestra la tabla de rutas desde la perspectiva de FRR. ip -4 route (en Bash) muestra la tabla de rutas real que el kernel de Linux está utilizando para reenviar tráfico.

¿Qué conserva write memory?

Conserva únicamente la configuración de los protocolos y rutas de FRR, escribiéndola en el archivo /etc/frr/frr.conf. No conserva interfaces temporales creadas con comandos de Linux (ip link).

¿Por qué no aparecen vecinos OSPF?

Porque OSPF es un protocolo de estado de enlace que requiere al menos otro router configurado con OSPF en la misma red para intercambiar mensajes Hello y formar una adyacencia.

¿Qué hace una ruta blackhole?

Indica al router que descarte silenciosamente (sin enviar mensajes ICMP de error) cualquier paquete cuyo destino coincida con ese prefijo.

¿Cuál es la equivalencia de write memory en Cisco IOS?

copy running-config startup-config.

¿Por qué la configuración de una IP se hace fuera de vtysh en este laboratorio?

Porque en Linux la administración de las interfaces de red y sus direcciones IP es competencia directa del sistema operativo (a través de iproute2), y FRR simplemente "lee" esas interfaces a través de zebra para operar sobre ellas.