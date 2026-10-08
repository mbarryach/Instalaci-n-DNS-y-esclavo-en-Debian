# Servidor DNS BIND9 maestro y esclavo con Ubuntu Server

Guía completa para montar un **servidor DNS maestro** y un **servidor DNS esclavo (secundario)** con BIND9 sobre Ubuntu Server, en VirtualBox, para el dominio `haven.local`.

## Datos del escenario

| Máquina | Rol | Hostname | IP | Interfaz red interna |
|---|---|---|---|---|
| Servidor maestro | Master | `ns1.haven.local` | `192.168.6.100/24` | `enp0s8` |
| Servidor esclavo | Slave | `ns2.haven.local` | `192.168.6.101/24` | `enp0s8` |

- Dominio: `haven.local`
- Zona inversa: `6.168.192.in-addr.arpa`
- Red interna: `192.168.6.0/24`
- Cada máquina tiene dos adaptadores en VirtualBox: NAT (`enp0s3`, Internet) y red interna (`enp0s8`).

> **¿Cómo funciona?** El maestro tiene los ficheros de zona originales. El esclavo no los edita nunca: los pide al maestro mediante una *transferencia de zona* (AXFR) y se queda una copia. Cada vez que el maestro cambia una zona (y aumenta su `Serial`), avisa al esclavo con un mensaje `NOTIFY` y este vuelve a descargarla.

## Índice

- [Parte 1 · Servidor DNS maestro](#parte-1--servidor-dns-maestro-ns1--1921686100)
- [Parte 2 · Servidor DNS esclavo](#parte-2--servidor-dns-esclavo-ns2--1921686101)

Haz primero la Parte 1 y comprueba que el maestro responde; después sigue con la Parte 2.

---

## Parte 1 · Servidor DNS maestro (`ns1` – 192.168.6.100)

### 1. Instalar Ubuntu Server

Instalamos **Ubuntu Server** en VirtualBox.

---

### 2. Configurar la red en VirtualBox

Configuramos dos adaptadores de red:

- **Adaptador 1:** NAT
- **Adaptador 2:** Red interna

El primer adaptador nos proporciona acceso a Internet y el segundo se utilizará para nuestra red interna.

---

### 3. Configurar la red en Ubuntu Server

Editamos el fichero de Netplan:

```bash
sudo nano /etc/netplan/*.yaml
```

En nuestro caso utilizamos:

- `enp0s3` → DHCP, conectado al adaptador NAT.
- `enp0s8` → IP estática `192.168.6.100/24`, conectado a la red interna.

La configuración queda:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true

    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.6.100/24
      nameservers:
        addresses:
          - 192.168.6.100
          - 9.9.9.9
          - 8.8.8.8
        search:
          - haven.local
```

Aplicamos los cambios:

```bash
sudo netplan try
```

Si todo funciona correctamente, confirmamos la configuración. Después:

```bash
sudo netplan apply
```

Podemos comprobar la configuración con:

```bash
ip a
ip route
```

La interfaz `enp0s8` debe tener la dirección:

```
192.168.6.100/24
```

---

### 4. Instalar BIND9

Actualizamos los paquetes:

```bash
sudo apt update
```

Opcionalmente podemos actualizar el sistema:

```bash
sudo apt upgrade
```

Instalamos BIND9 y las herramientas necesarias:

```bash
sudo apt install bind9
```

Comprobamos que el servicio está funcionando:

```bash
sudo systemctl status bind9
```

Debe aparecer:

```
Active: active (running)
```

---

### 5. Configurar las zonas en `named.conf.local`

Primero hacemos una copia de seguridad:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.BKP
```

Editamos el fichero:

```bash
sudo nano /etc/bind/named.conf.local
```

Definimos la zona directa y la zona inversa:

```
zone "haven.local" {
        type master;
        file "/etc/bind/zones/db.haven.local";
};

zone "6.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.6.168.192";
};
```

Comprobamos la sintaxis:

```bash
sudo named-checkconf
```

Si no devuelve nada, significa que no se han encontrado errores de sintaxis.

---

### 6. Crear los ficheros de zona

Creamos el directorio:

```bash
sudo mkdir /etc/bind/zones
```

#### 6.1. Zona directa

Copiamos la plantilla de BIND:

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.haven.local
```

Editamos el fichero:

```bash
sudo nano /etc/bind/zones/db.haven.local
```

Contenido:

```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.haven.local. admin.haven.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.haven.local.
@       IN      A       192.168.6.100
ns1     IN      A       192.168.6.100
```

Cada vez que modifiquemos una zona debemos incrementar el valor de `Serial`.

Comprobamos la zona:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
```

Si todo está correcto:

```
zone haven.local/IN: loaded serial 2
OK
```

---

### 7. Crear la zona inversa

La red utilizada es `192.168.6.0/24`, por lo tanto la zona inversa es:

```
6.168.192.in-addr.arpa
```

Copiamos la plantilla `db.127`:

```bash
sudo cp /etc/bind/db.127 /etc/bind/zones/db.6.168.192
```

Editamos:

```bash
sudo nano /etc/bind/zones/db.6.168.192
```

Contenido:

```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.haven.local. admin.haven.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.haven.local.
100     IN      PTR     ns1.haven.local.
```

Esto indica que la dirección `192.168.6.100` se corresponde con `ns1.haven.local`. El `100` corresponde al último octeto de la dirección IP.

Comprobamos la zona inversa:

```bash
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
```

Si todo está correcto:

```
zone 6.168.192.in-addr.arpa/IN: loaded serial 1
OK
```

También podemos volver a comprobar la zona directa:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
```

---

### 8. Configurar `named.conf.options`

Hacemos una copia de seguridad:

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.BKP
```

Editamos:

```bash
sudo nano /etc/bind/named.conf.options
```

Configuramos una ACL para permitir las consultas de nuestra red:

```
acl "safeclients" {
        localhost;
        192.168.6.0/24;
        localnets;
};

options {
        directory "/var/cache/bind";

        recursion yes;
        allow-recursion { safeclients; };

        listen-on { 192.168.6.100; };
        allow-transfer { none; };

        allow-query { safeclients; };
        allow-query-cache { safeclients; };

        forwarders {
                9.9.9.9;
                8.8.8.8;
        };
};
```

Los `forwarders` son servidores DNS externos que BIND puede consultar cuando no puede resolver directamente una consulta.

Comprobamos la configuración:

```bash
sudo named-checkconf
```

Si no devuelve nada, la sintaxis es correcta.

---

### 9. Forzar el uso de IPv4

Editamos:

```bash
sudo nano /etc/default/named
```

Buscamos y dejamos:

```
# run resolvconf?
RESOLVCONF=no

# startup options for the server
OPTIONS="-u bind -4"
```

La opción `-4` fuerza a BIND a utilizar IPv4.

---

### 10. Comprobación final antes de reiniciar

Comprobamos la configuración general:

```bash
sudo named-checkconf
```

Comprobamos la zona directa:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
```

Comprobamos la zona inversa:

```bash
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
```

Las tres comprobaciones deben ser correctas.

---

### 11. Reiniciar BIND9

Reiniciamos el servicio:

```bash
sudo systemctl restart bind9
```

Comprobamos su estado:

```bash
sudo systemctl status bind9
```

Debe aparecer:

```
Active: active (running)
```

Además, el proceso debería aparecer utilizando la opción:

```
-4
```

---

### 12. Configurar `/etc/resolv.conf`

Comprobamos el enlace:

```bash
ls -l /etc/resolv.conf
```

En nuestro caso debe apuntar a:

```
/run/systemd/resolve/resolv.conf
```

Si apunta al `stub-resolv.conf`, podemos cambiarlo mediante:

```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Comprobamos el contenido:

```bash
cat /etc/resolv.conf
```

Debe aparecer nuestro servidor DNS:

```
nameserver 192.168.6.100
```

También pueden aparecer otros servidores DNS configurados por el sistema.

---

### 13. Pruebas con `nslookup`

#### Resolución directa

Comprobamos que el dominio se resuelve correctamente:

```bash
nslookup haven.local
```

Debe devolver:

```
Server:         192.168.6.100
Address:        192.168.6.100#53

Name:   haven.local
Address: 192.168.6.100
```

También comprobamos el nombre del servidor DNS:

```bash
nslookup ns1.haven.local
```

Debe devolver:

```
Server:         192.168.6.100
Address:        192.168.6.100#53

Name:   ns1.haven.local
Address: 192.168.6.100
```

#### Resolución inversa

Comprobamos que la dirección IP se puede convertir en un nombre:

```bash
nslookup 192.168.6.100
```

Debe devolver:

```
100.6.168.192.in-addr.arpa      name = ns1.haven.local.
```

También podemos forzar la consulta directamente contra nuestro servidor DNS:

```bash
nslookup haven.local 192.168.6.100
```

```
Server:         192.168.6.100
Address:        192.168.6.100#53

Name:   haven.local
Address: 192.168.6.100
```

De esta forma revisamos directamente que BIND9 está respondiendo las consultas.

---

## Parte 2 · Servidor DNS esclavo (`ns2` – 192.168.6.101)

Esta parte parte del maestro ya funcionando (Parte 1). Los pasos de la sección 8 se hacen en el **maestro**; el resto, en el **esclavo**.

### 1. Crear la máquina virtual en VirtualBox

Instalamos otra máquina con **Ubuntu Server** (igual que el maestro) y configuramos dos adaptadores de red:

- **Adaptador 1:** NAT (acceso a Internet).
- **Adaptador 2:** Red interna, **con el mismo nombre de red interna que el maestro**. Si no, las dos máquinas no se verán.

---

### 2. Configurar la red en Ubuntu Server

Editamos el fichero de Netplan:

```bash
sudo nano /etc/netplan/*.yaml
```

En nuestro caso utilizamos:

- `enp0s3` → DHCP, conectado al adaptador NAT.
- `enp0s8` → IP estática `192.168.6.101/24`, conectado a la red interna.

La configuración queda:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true

    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.6.101/24
      nameservers:
        addresses:
          - 192.168.6.101
          - 192.168.6.100
          - 9.9.9.9
        search:
          - haven.local
```

Aplicamos los cambios:

```bash
sudo netplan try
```

Si todo funciona correctamente, confirmamos la configuración. Después:

```bash
sudo netplan apply
```

Comprobamos la configuración con:

```bash
ip a
ip route
```

La interfaz `enp0s8` debe tener la dirección:

```
192.168.6.101/24
```

Comprobamos también que vemos al maestro:

```bash
ping -c 3 192.168.6.100
```

---

### 3. Configurar el nombre de la máquina

Asignamos el nombre `ns2`:

```bash
sudo hostnamectl set-hostname ns2
```

Editamos `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Y dejamos (o añadimos) la línea:

```
127.0.1.1   ns2.haven.local ns2
```

Comprobamos que el FQDN se resuelve correctamente:

```bash
hostname -f
```

Debe devolver:

```
ns2.haven.local
```

---

### 4. Instalar BIND9

Actualizamos los paquetes:

```bash
sudo apt update
```

Instalamos BIND9 y las herramientas necesarias:

```bash
sudo apt install bind9 bind9utils dnsutils
```

Comprobamos que el servicio está funcionando:

```bash
sudo systemctl status bind9
```

Debe aparecer:

```
Active: active (running)
```

---

### 5. Configurar `named.conf.options`

Hacemos una copia de seguridad:

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.BKP
```

Editamos:

```bash
sudo nano /etc/bind/named.conf.options
```

La configuración es la misma que la del maestro, cambiando únicamente la IP de `listen-on`:

```
acl "safeclients" {
        localhost;
        192.168.6.0/24;
        localnets;
};

options {
        directory "/var/cache/bind";

        recursion yes;
        allow-recursion { safeclients; };

        listen-on { 192.168.6.101; };
        allow-transfer { none; };

        allow-query { safeclients; };
        allow-query-cache { safeclients; };

        forwarders {
                9.9.9.9;
                8.8.8.8;
        };
};
```

> Mantenemos `allow-transfer { none; }` también en el esclavo: nadie debe poder descargarse las zonas desde esta máquina.

Comprobamos la configuración:

```bash
sudo named-checkconf
```

Si no devuelve nada, la sintaxis es correcta.

---

### 6. Forzar el uso de IPv4

Editamos:

```bash
sudo nano /etc/default/named
```

Dejamos la línea `OPTIONS` así:

```
OPTIONS="-u bind -4"
```

La opción `-4` fuerza a BIND a utilizar IPv4.

---

### 7. Configurar las zonas esclavas en `named.conf.local`

Primero hacemos una copia de seguridad:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.BKP
```

Editamos el fichero:

```bash
sudo nano /etc/bind/named.conf.local
```

Definimos las dos zonas como `type slave`, indicando quién es el maestro:

```
zone "haven.local" {
        type slave;
        masters { 192.168.6.100; };
        file "/var/cache/bind/slaves/db.haven.local";
};

zone "6.168.192.in-addr.arpa" {
        type slave;
        masters { 192.168.6.100; };
        file "/var/cache/bind/slaves/db.6.168.192";
};
```

> **Importante:** en el esclavo los ficheros de zona se guardan en `/var/cache/bind/slaves/` y **no** en `/etc/bind/zones/`. En Ubuntu, AppArmor solo permite a BIND escribir en `/var/cache/bind`, y el esclavo necesita escribir los ficheros que descarga.

Creamos el directorio y le damos los permisos adecuados:

```bash
sudo mkdir -p /var/cache/bind/slaves
sudo chown bind:bind /var/cache/bind/slaves
```

Comprobamos la sintaxis:

```bash
sudo named-checkconf
```

Aún **no reiniciamos** el esclavo: primero hay que autorizarlo en el maestro.

---

### 8. Actualizar el servidor maestro (`ns1` – 192.168.6.100)

Todos los pasos de esta sección se hacen en el **maestro**.

#### 8.1. Autorizar la transferencia de zona

Hacemos una copia de seguridad y editamos:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.BKP2
sudo nano /etc/bind/named.conf.local
```

Añadimos `allow-transfer`, `also-notify` y `notify` en cada zona con la IP del esclavo:

```
zone "haven.local" {
        type master;
        file "/etc/bind/zones/db.haven.local";
        allow-transfer { 192.168.6.101; };
        also-notify { 192.168.6.101; };
        notify yes;
};

zone "6.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.6.168.192";
        allow-transfer { 192.168.6.101; };
        also-notify { 192.168.6.101; };
        notify yes;
};
```

> Aunque en `named.conf.options` tenemos `allow-transfer { none; }`, la directiva definida **dentro de la zona tiene prioridad**, así que solo el esclavo podrá descargar estas zonas.

#### 8.2. Actualizar la zona directa

```bash
sudo nano /etc/bind/zones/db.haven.local
```

Añadimos el registro `NS` y el registro `A` del nuevo servidor, y **aumentamos el `Serial`** (estaba en `2`, lo subimos a `3`):

```
;
; BIND data file for haven.local
;
$TTL    604800
@       IN      SOA     ns1.haven.local. admin.haven.local. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.haven.local.
@       IN      NS      ns2.haven.local.
@       IN      A       192.168.6.100
ns1     IN      A       192.168.6.100
ns2     IN      A       192.168.6.101
```

#### 8.3. Actualizar la zona inversa

```bash
sudo nano /etc/bind/zones/db.6.168.192
```

Añadimos el registro `NS` y el `PTR` del esclavo, y aumentamos el `Serial` (de `1` a `2`):

```
;
; BIND reverse data file for haven.local
;
$TTL    604800
@       IN      SOA     ns1.haven.local. admin.haven.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.haven.local.
@       IN      NS      ns2.haven.local.
100     IN      PTR     ns1.haven.local.
101     IN      PTR     ns2.haven.local.
```

> Recuerda: **cada vez que modifiquemos una zona hay que incrementar el `Serial`**. Si no se aumenta, el esclavo piensa que ya tiene la última versión y no se descarga los cambios.

#### 8.4. Comprobar y recargar

```bash
sudo named-checkconf
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
```

Las tres comprobaciones deben ser correctas. Para las zonas debe aparecer:

```
zone haven.local/IN: loaded serial 3
OK
zone 6.168.192.in-addr.arpa/IN: loaded serial 2
OK
```

Recargamos la configuración:

```bash
sudo rndc reload
```

O reiniciamos el servicio:

```bash
sudo systemctl restart bind9
```

> Si el maestro tiene el cortafuegos activo (`ufw`), hay que permitir el puerto 53 **TCP y UDP**. Las transferencias de zona usan TCP: `sudo ufw allow 53`.

---

### 9. Reiniciar BIND9 en el esclavo (`ns2` – 192.168.6.101)

Volvemos al esclavo y hacemos la comprobación final:

```bash
sudo named-checkconf
```

Reiniciamos el servicio:

```bash
sudo systemctl restart bind9
```

Comprobamos su estado:

```bash
sudo systemctl status bind9
```

Debe aparecer:

```
Active: active (running)
```

Además, el proceso debería aparecer utilizando la opción:

```
-4
```

Comprobamos que las zonas se han descargado del maestro:

```bash
ls -l /var/cache/bind/slaves
```

Deben aparecer los dos ficheros:

```
db.haven.local
db.6.168.192
```

Los ficheros de zona esclavos se guardan en formato binario, así que no se pueden leer con `cat`. Para ver los registros de la transferencia consultamos los logs:

```bash
sudo journalctl -u bind9 --no-pager | tail -n 20
```

Deben aparecer líneas indicando que la transferencia de zona se ha completado (`Transfer completed`) para `haven.local` y `6.168.192.in-addr.arpa`.

---

### 10. Configurar `/etc/resolv.conf`

Comprobamos el enlace:

```bash
ls -l /etc/resolv.conf
```

Debe apuntar a:

```
/run/systemd/resolve/resolv.conf
```

Si apunta al `stub-resolv.conf`, podemos cambiarlo mediante:

```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Comprobamos el contenido:

```bash
cat /etc/resolv.conf
```

Debe aparecer nuestro servidor DNS:

```
nameserver 192.168.6.101
```

También pueden aparecer otros servidores DNS configurados por el sistema (el maestro `192.168.6.100`, `9.9.9.9`...).

---

### 11. Pruebas de funcionamiento

#### 11.1. Resolución directa contra el esclavo

Forzamos la consulta directamente contra el esclavo:

```bash
nslookup haven.local 192.168.6.101
```

Debe devolver:

```
Server:         192.168.6.101
Address:        192.168.6.101#53

Name:   haven.local
Address: 192.168.6.100
```

Comprobamos el nuevo registro del esclavo:

```bash
nslookup ns2.haven.local 192.168.6.101
```

Debe devolver:

```
Name:   ns2.haven.local
Address: 192.168.6.101
```

#### 11.2. Resolución inversa contra el esclavo

```bash
nslookup 192.168.6.101 192.168.6.101
```

Debe devolver:

```
101.6.168.192.in-addr.arpa      name = ns2.haven.local.
```

#### 11.3. Comparar el `Serial` en ambos servidores

Si la transferencia funciona, los dos servidores deben devolver **el mismo número de serie**:

```bash
dig @192.168.6.100 haven.local SOA +short
dig @192.168.6.101 haven.local SOA +short
```

Ambos deben devolver:

```
ns1.haven.local. admin.haven.local. 3 604800 86400 2419200 604800
```

#### 11.4. Forzar una transferencia de zona manualmente

Desde el esclavo podemos pedir la zona al maestro para comprobar que está autorizado:

```bash
dig @192.168.6.100 haven.local AXFR
```

Debe listar todos los registros de la zona. Si devuelve `Transfer failed`, revisa el `allow-transfer` del maestro.

---

### 12. Prueba de sincronización

Vamos a comprobar que un cambio en el maestro llega solo al esclavo.

**1.** En el **maestro**, añadimos un registro de prueba al final de `/etc/bind/zones/db.haven.local` y aumentamos el `Serial` de `3` a `4`:

```
prueba  IN      A       192.168.6.50
```

**2.** Comprobamos y recargamos:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo rndc reload
```

**3.** En el **esclavo**, vemos los logs en tiempo real:

```bash
sudo journalctl -u bind9 -f
```

Debe aparecer la notificación del maestro y la transferencia de la zona.

**4.** Consultamos el registro nuevo en ambos servidores:

```bash
dig @192.168.6.100 prueba.haven.local +short
dig @192.168.6.101 prueba.haven.local +short
```

Los dos deben devolver:

```
192.168.6.50
```

Si el esclavo no devuelve nada, comprueba que has incrementado el `Serial`. Como último recurso, puedes forzar la actualización desde el esclavo:

```bash
sudo rndc retransfer haven.local
```

**5.** Una vez comprobado, borramos el registro `prueba` del maestro, incrementamos de nuevo el `Serial` y recargamos.

#### Prueba de tolerancia a fallos

Paramos el DNS en el **maestro**:

```bash
sudo systemctl stop bind9
```

Desde un cliente (o desde el propio esclavo) comprobamos que el esclavo sigue respondiendo:

```bash
nslookup ns1.haven.local 192.168.6.101
```

Volvemos a arrancar el maestro:

```bash
sudo systemctl start bind9
```

> Para que los clientes aprovechen esta redundancia, deben tener configurados **los dos servidores DNS** (`192.168.6.100` y `192.168.6.101`).

---

### 13. Problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| El directorio `/var/cache/bind/slaves` está vacío | El maestro rechaza la transferencia | Revisar `allow-transfer { 192.168.6.101; };` en el maestro y el cortafuegos (puerto 53 TCP/UDP) |
| El esclavo responde con un `Serial` antiguo | No se incrementó el `Serial` en el maestro | Aumentar el `Serial`, `sudo rndc reload` y, si hace falta, `sudo rndc retransfer haven.local` en el esclavo |
| `permission denied` al escribir la zona en los logs | Ruta o permisos incorrectos | Usar `/var/cache/bind/slaves/` y `sudo chown bind:bind /var/cache/bind/slaves` |
| `dig` desde el esclavo hacia el maestro no responde | Las VM no están en la misma red interna | Revisar el nombre de la red interna en el Adaptador 2 de VirtualBox |
| Las consultas son rechazadas (`REFUSED`) | El cliente no está en la ACL | Revisar `allow-query` y la ACL `safeclients` en `named.conf.options` |
