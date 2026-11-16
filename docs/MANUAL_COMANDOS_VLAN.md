# Manual básico de comandos para segmentar una red doméstica mediante VLAN

## 1. Objetivo

Esta guía muestra una configuración de referencia para dividir una red doméstica en tres segmentos:

| Entorno | VLAN | Red IP | Gateway |
|---|---:|---|---|
| Trabajo | 10 | 192.168.10.0/24 | 192.168.10.1 |
| Personal | 20 | 192.168.20.0/24 | 192.168.20.1 |
| IoT | 30 | 192.168.30.0/24 | 192.168.30.1 |

Los valores son ilustrativos. Se pueden cambiar según las necesidades de cada instalación. Se supone una configuración Cisco IOS con un switch de capa 2 y un router que realiza el enrutamiento entre VLAN (router-on-a-stick).

## 2. Esquema de conexión de referencia

- El router se conecta al switch mediante un enlace troncal.
- Los equipos finales se conectan a puertos de acceso del switch.
- Cada puerto de acceso se asigna a la VLAN correspondiente.
- El router utiliza subinterfaces, una por VLAN, para actuar como gateway y permitir el enrutamiento controlado.

Ejemplo de puertos del switch:
- Fa0/1: enlace troncal hacia el router.
- Fa0/2-Fa0/4: equipos del entorno Trabajo.
- Fa0/5-Fa0/8: equipos del entorno Personal.
- Fa0/9-Fa0/12: equipos IoT.

Los puertos son ejemplos; se deben sustituir por los que realmente se utilicen.

## 3. Configuración del switch

### 3.1 Crear las VLAN

Entrar al modo de configuración:

```text
enable
configure terminal
```

Crear las VLAN y asignarles nombres:

```text
vlan 10
 name TRABAJO
exit

vlan 20
 name PERSONAL
exit

vlan 30
 name IOT
exit
```

### 3.2 Asignar puertos de acceso

Ejemplo: asignar Fa0/2 a Fa0/4 a la VLAN 10:

```text
interface range fastEthernet 0/2-4
 switchport mode access
 switchport access vlan 10
exit
```

Asignar Fa0/5 a Fa0/8 a la VLAN 20:

```text
interface range fastEthernet 0/5-8
 switchport mode access
 switchport access vlan 20
exit
```

Asignar Fa0/9 a Fa0/12 a la VLAN 30:

```text
interface range fastEthernet 0/9-12
 switchport mode access
 switchport access vlan 30
exit
```

### 3.3 Configurar el enlace troncal hacia el router

Ejemplo: Fa0/1 conecta el switch con el router:

```text
interface fastEthernet 0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
exit
```

En algunos modelos puede ser necesario indicar la encapsulación 802.1Q antes de activar el modo trunk; en otros, el comando no está disponible porque 802.1Q es la opción predeterminada.

### 3.4 Guardar y verificar

```text
end
copy running-config startup-config
show vlan brief
show interfaces trunk
```

`show vlan brief` permite revisar las VLAN y los puertos asignados. `show interfaces trunk` permite revisar los enlaces troncales y las VLAN que transportan.

## 4. Configuración del router: subinterfaces y gateways

En este ejemplo, la interfaz física `GigabitEthernet0/0` del router conecta con el puerto troncal del switch. Ajustar el nombre de la interfaz al modelo utilizado.

```text
enable
configure terminal

interface gigabitEthernet 0/0
 no shutdown
exit
```

Crear una subinterfaz para cada VLAN:

```text
interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface gigabitEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

interface gigabitEthernet 0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
exit
```

Cada subinterfaz funciona como gateway para los equipos de su VLAN. La interfaz física debe estar conectada al puerto troncal del switch.

## 5. Configurar los pools DHCP

Se pueden crear pools independientes para entregar automáticamente direcciones IP a cada segmento.

Primero, excluir direcciones que se reservarán para gateways, impresoras u otros equipos:

```text
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20
```

Crear el pool de Trabajo:

```text
ip dhcp pool VLAN_TRABAJO
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
exit
```

Crear el pool Personal:

```text
ip dhcp pool VLAN_PERSONAL
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit
```

Crear el pool IoT:

```text
ip dhcp pool VLAN_IOT
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
exit
```

La dirección DNS del ejemplo es pública. En una instalación real se debe utilizar un servidor DNS disponible y autorizado para esa red. Si el servicio DHCP se encuentra en otro equipo, será necesario configurar el mecanismo de retransmisión DHCP (`ip helper-address`) en las interfaces correspondientes, según el diseño.

Verificar:

```text
show ip dhcp pool
show ip dhcp binding
```

## 6. Políticas básicas entre segmentos mediante ACL

La política de referencia es restringir el acceso entre segmentos y permitir únicamente las comunicaciones necesarias. Las ACL siguientes son ejemplos simplificados para ilustrar el control entre redes.

### 6.1 Bloquear IoT hacia Trabajo y Personal

```text
ip access-list extended ACL_IOT
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any
exit

interface gigabitEthernet 0/0.30
 ip access-group ACL_IOT in
exit
```

Esta regla impide que los dispositivos IoT inicien tráfico IP hacia las redes Trabajo y Personal. Permite el resto del tráfico originado en IoT, por lo que en una instalación real se recomienda restringir también los destinos externos a los estrictamente necesarios.

### 6.2 Restringir Personal hacia Trabajo

```text
ip access-list extended ACL_PERSONAL
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any
exit

interface gigabitEthernet 0/0.20
 ip access-group ACL_PERSONAL in
exit
```

Esta política bloquea el inicio de tráfico desde Personal hacia Trabajo, manteniendo permitido el resto del tráfico. Las excepciones necesarias deben definirse antes de la regla de denegación y limitarse por origen, destino y servicio.

### 6.3 Permitir una excepción específica de impresión

Supongamos que un computador de Trabajo (`192.168.10.10`) necesita imprimir en una impresora ubicada en Personal (`192.168.20.10`). La regla debe permitir solo el servicio requerido por el modelo y método de impresión.

Ejemplo conceptual con ICMP y TCP, que debe ajustarse a los puertos reales del servicio de impresión:

```text
ip access-list extended ACL_TRABAJO
 permit icmp host 192.168.10.10 host 192.168.20.10
 permit tcp host 192.168.10.10 host 192.168.20.10
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
exit

interface gigabitEthernet 0/0.10
 ip access-group ACL_TRABAJO in
exit
```

**Advertencia:** el ejemplo de TCP permite cualquier puerto TCP hacia esa impresora y no constituye una política final de mínimo privilegio. Para una instalación real, sustituirlo por los puertos específicos que requiera la impresora. Además, las ACL IPv4 clásicas son sin estado: se debe revisar el tráfico de respuesta y probar el funcionamiento del servicio. Una ACL aplicada de esta forma puede bloquear respuestas iniciadas desde otra VLAN si no se contemplan explícitamente.

### 6.4 Revisar el orden de las ACL

Las ACL se procesan de arriba hacia abajo y se aplica la primera regla coincidente. Por eso:
- Las excepciones específicas deben ubicarse antes de las reglas generales de denegación.
- Se debe considerar la denegación implícita al final de una ACL.
- La ubicación y dirección de aplicación (`in` o `out`) deben corresponder al flujo que se desea controlar.
- Después de aplicar una ACL, se deben probar tanto los accesos permitidos como los bloqueados.

## 7. Verificación general

En el switch:

```text
show vlan brief
show interfaces trunk
show running-config
```

En el router:

```text
show ip interface brief
show ip route
show ip dhcp pool
show ip dhcp binding
show access-lists
show running-config
```

En los equipos finales:
- Revisar que la dirección IP corresponda a la subred prevista.
- Verificar el gateway y DNS recibidos.
- Probar el gateway del propio segmento.
- Probar servicios que deban estar permitidos.
- Probar comunicaciones que deban estar bloqueadas.
- Registrar los resultados y las evidencias.

Una prueba fallida no demuestra por sí sola que la ACL esté bloqueando correctamente: se debe comprobar que las interfaces, rutas, VLAN, direccionamiento y servicios estén operativos.

## 8. Lista breve de pruebas

| Prueba | Resultado esperado |
|---|---|
| Equipo de Trabajo recibe IP de VLAN 10 | Correcto |
| Equipo Personal recibe IP de VLAN 20 | Correcto |
| Dispositivo IoT recibe IP de VLAN 30 | Correcto |
| Equipo IoT inicia conexión hacia Trabajo | Bloqueada por la política |
| Equipo Personal inicia conexión hacia Trabajo | Bloqueada, salvo excepción |
| Equipo autorizado de Trabajo imprime en la impresora | Permitido según la excepción |
| Equipos acceden a servicios externos requeridos | Permitido según las reglas y rutas |

## 9. Recomendaciones finales

1. Confirmar que router, switch y punto de acceso soporten VLAN y las funciones de filtrado requeridas.
2. Cambiar los números de VLAN, subredes, interfaces y puertos de acuerdo con la instalación.
3. Respaldar la configuración antes de realizar cambios.
4. No aplicar ACL sin comprender su dirección, orden y efecto sobre el tráfico de respuesta.
5. Empezar con reglas restrictivas y agregar excepciones específicas justificadas.
6. Documentar y probar cada cambio.
7. Recordar que la segmentación reduce la exposición entre redes, pero no reemplaza las actualizaciones, contraseñas seguras ni las medidas de protección de cada dispositivo.

## 10. Alcance del ejemplo

Los comandos se presentan como una base didáctica para equipos Cisco IOS con router-on-a-stick. No son una configuración universal ni deben copiarse sin adaptación. La sintaxis, las capacidades y los nombres de interfaz pueden variar por modelo y versión. Para otras marcas, se debe aplicar el mismo diseño lógico utilizando los procedimientos equivalentes del fabricante.
