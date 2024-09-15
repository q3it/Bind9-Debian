# Instalación de un servidor DNS Bind9 en Debian 12

## Autor

- [q3it](https://www.blogger.com/profile/16118326082555553765)

## Resumen

El objetivo de esta guia es mostrar cómo configurar un servidor `DNS Bind9` que resuelva direcciones IP en nombres de dominio dentro de una red privada. En este proceso de configuración se requieren conocimientos mínimos de `REDES` y `DNS`. En las [referencias](#referencias), encontrará enlaces a sitios de `Internet` que pueden ayudar.

## Escenario

Es común encontrarnos entornos de red, donde se necesite que un mismo servidor de nombres `DNS` devuelva registros tanto de tipo canónico como direcciones `IP`, dependiendo de la red desde donde se originen las consultas. Por ejemplo:

Existencia de un servidor `DNS` dentro del direccionamiento `TCP/IP` de la subred de servicios, o cómo se le conoce comúnmente red `DMZ`, que da servicio a redes públicas (`Internet`, la red externa del provedor `ISP` o la `VPN` externa de una organización) y redes privadas (`Intranet` o red `LAN`, una `DMZ`, una `VPN` interna, o a todas ellas). Si se realiza la consulta desde el exterior, deben devolverse los registros públicos; sin embargo, si se hace la misma consulta desde cualquiera de las subredes internas, la resolución deberá ser a un registro privado. Esto es posible lograrlo, gracias a `Bind9`.

## Administración del servidor

El servidor `DNS` de ejemplo utilizá los siguientes parámetros de configuración de red:

* Dirección `IP` del servidor: `192.168.1.76`
* Dominio `DNS`: `clockwork.local`
* `FQDN` del servidor: `ns.servidor.clockwork.local`
* Red interna de la zona de servicios: `192.168.1.0/24`
* Red externa para registros públicos: `192.168.210.142/24`

### Ajustes de los parámetros de red

```bash
nano /etc/network/interfaces

# The loopback network interface
auto lo
iface lo inet loopback

#Primer adaptador ens160 DHCP
auto ens160
iface ens160 inet dhcp

#Segundo adaptador ens256 Interna
auto ens256
iface ens256 inet static
address 192.168.1.76
netmask 255.255.255.0
network 192.168.1.0
broadcast 192.168.1.255
```

```bash
nano /etc/resolv.conf

nameserver 192.168.1.76
options ens256 trust-ad
search clockwork.local
```

```bash
nano /etc/hosts

127.0.0.1     clockwork.local   localhost
192.168.1.76   servidor.clockwork.local          ns
```

### Instalación

```bash
apt install bind9 bind9-dnsutils
```

Para disponer de la documentación `off-line`, instalar además `bind9-doc`.

### Configuración

En las distribuciones `Debian GNU/Linux`, los ficheros de configuración del paquete `bind9`, se encuentran en `/etc/bind`. Ellos son: `named.conf` (fichero de configuración principal), `named.conf.default-zones` (contiene las zonas predefinidas de reenvío (`forward`), inversa (`reverse`) y difusión (`broadcast`) para el `localhost`), `named.conf.options` (contiene todos los parámetros para la operación del servicio), y `named.conf.local` (contiene las opciones de configuración y las declaraciones de zonas del servidor `DNS` local).

> **NOTA**: Es importante leer el archivo **`/usr/share/doc/bind9/README.Debian.gz`**, para obtener información sobre la estructura de los archivos de configuración del servicio `BIND` en `Debian`. De igual forma es una buena práctica realizar copias de seguridad de los ficheros mencionados en el párrafo anterior en su estado por defecto, **ANTES** de realizar modificaciones.

1. Editar fichero de configuración principal.

```bash
cp /etc/bind/named.conf{,.bak}
nano /etc/bind/named.conf
```
```bash
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

2. Permitir solo direcciones IPv4

```bash
nano /etc/default/named
```
```bash
# run resolvconf?
RESOLVCONF=no

# startup options for the server
OPTIONS="-u bind -4"
```

3. Editar parámetros para la operación del servicio.

```bash
cp /etc/bind/named.conf.options{,.org}
nano /etc/bind/named.conf.options
```
```bash
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        listen-on { any; };
        allow-query { localhost; 192.168.1.0/24; };
        forwarders {
                8.8.8.8;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation no;

        #listen-on-v6 { any; };
};
```

4. Definir opciones de configuración y declaraciones de zonas del servidor `DNS`.

```bash
cp /etc/bind/named.conf.local{,.bak}
nano /etc/bind/named.conf.local
```
```bash
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "clockwork.local" IN {
        type master;
        file "etc/bind/zones/db.clockwork.local";
};

zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.1.168.192";
};
```

5. Crear ficheros de zonas.

El sistema `DNS` está compuesto por varios registros, conocidos como Registros de Recursos (`Resource Records` o `RR`, en Inglés), que definen la información en el sistema de nombres de dominio, tanto para resolución de nombre -conocida también como directa o canónica (conversión de nombre a dirección `IP`)-, como para resolución de nombre inversa (conversión de dirección `IP` a nombre).

- Creamos el directorio de zonas.

```bash
mkdir /etc/bind/zonas
```

- Copiamos el directorio de Zona Directa.
   
```bash
cp /etc/bind/db.local /etc/bind/zonas/db.clockwork.local
```
- Configuramos.

```bash
nano /etc/bind/db.clockwork.local
```
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     servidor.clockwork.local. root.clockwork.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
                IN      NS      servidor.clockwork.local.
servidor        IN      A       192.168.1.76
equipo01        IN      A       192.168.1.54
server          IN      CNAME   servidor
```

- Copiamos el directorio de Zona Inversa.
   
```bash
cp /etc/bind/zonas/db.clockwork.local /etc/bind/zonas/db.1.168.192
```

- Configuramos.

```bash
nano /etc/bind/db.1.168.192
```
```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     servidor.clockwork.local. root.clockwork.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
                IN      NS      servidor.clockwork.local.
76              IN      PTR     servidor.clockwork.local.
```

6. Comprobar la existencia de errores tanto en la configuración como en los ficheros de zonas.

```bash
named-checkconf /etc/bind/named.conf.local
named-checkzone clockwork.local /etc/bind/zonas/db.clockwork.local
named-checkzone 1.168.192.in-addr.arpa /etc/bind/zonas/db.1.168.192
```

7. Reiniciar servicios y realizar comprobaciones.

```bash
systemctl restart bind9
```

- Comprobar estado del servidor

```bash
systemctl status bind9
```

- Conexión desde máquina cliente 

```bash
ping servidor.clockwork.local
ping equipo01
nslookup clockwork.local
host servidor
```

## Conclusiones

`Bind9` es un mecanismo muy útil en entornos de red que brindan servicios detrás de cortafuegos, es posible presentar una configuración del servidor `DNS` distinta a varios dispositivos. Algo particularmente provechoso si se ejecuta un servidor que recibe consultas desde redes privadas y públicas como es el caso de `Internet`.

## Demo

https://github.com/user-attachments/assets/103de973-110f-4d2f-a6d3-bba2cee04388

## Referencias

* [Bind9 - Debian Wiki](https://wiki.debian.org/Bind9)
* [Internet Systems Consortium](https://www.isc.org/bind/)
* [Cómo configurar BIND en Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-18-04-es)
* [DNS con Bind9](https://www.redeszone.net/tutoriales/servidores/configurar-servidor-dns-bind-linux/)
