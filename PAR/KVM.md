# KVM

Configurar tarjeta de red.

Entramos en la configuración de la trjeta de red y tendremos que cambiar nuestra eth0 por br0 dejanlo el fichero ``/etc/network/interfaces`` tal y como lo miestro ahora.

~~~
auto lo
iface lo inet loopback

allow-hotplug br0
iface br0 inet dhcp
  bridge_ports eth0

#allow-hotplug eth0
#iface eth0 inet dhcp
~~~

Ahora levantamos br0 con el comando(Como root).

~~~
# ifup br0
~~~

Tendremos que isntalar el KVM, para instalar tendremos que poner el siguiente paquete.

~~~
# sudo apt-get install qemu-kvm
~~~

Ahora añadiremos una nueva interfaz de red virtual(Como root).

~~~
# ip tuntap add mode tap user user
~~~

Podemos ver esa nueva interfaz con el comando.

~~~
# ip tuntap list
~~~

Ahora lo conectamos al dispositivo br0(Como root).

~~~
# brctl addif br0 tap0
~~~

Por último levantamos la tarjeta de red(como root).

~~~
# ip l set dev tap0 up
~~~

Ahora nos salimos y nos metemos en nuestro usuario y tendremos que generar una MAC.

~~~
$ MAC0=$(echo "02:"`openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/.$//'`)
~~~

Podremos ver que la hemos creado correctamente.

~~~
$ echo $MAC0
~~~

Por último podremos levantar la máquina desde el usuario normal con el comando.

~~~
$ kvm -m 512 -hda jessie-1.qcow2 \
-device virtio-net,netdev=n0,mac=$MAC0 \
-netdev tap,id=n0,ifname=tap0,script=no,downscript=no
~~~

## Crear bond

Para crear una interface bond tendremos que ejecutar el siguiente comando añadiendo el nombre que queramos a la interfaz.

~~~
#ip link add name "name" type bond
~~~

Ahora creamos los dos tap para ello tendremos que ejecutar dos veces el siguiente comando.

~~~
# ip tuntap add mode tap user user
~~~

A continuación tendremos que poner tap0 y tap1 como master de bond0 con estos dos comandos.

~~~
ip link set dev tap0 master bond0
~~~

~~~
ip link set dev tap1 master bond0
~~~

Creamos las MAC correspondientes con los siguientes comandos.

~~~
$ MAC0=$(echo "02:"`openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/.$//'`)
~~~

~~~
$ MAC1=$(echo "02:"`openssl rand -hex 5 | sed 's/\(..\)/\1:/g; s/.$//'`)
~~~

Ahora tendremos que iniciar la máquina virtual con las dos interfaces de red, para ello tendremos que ejecutar el siguiente comando.

~~~
$ kvm -m 512 -hda jessie-1.qcow2 -device virtio-net,netdev=n0,mac=$MAC0 -device virtio-net,netdev=n1,mac=$MAC1 -netdev tap,id=n0,ifname=tap0,script=no,downscript=no -netdev tap,id=n1,ifname=tap1,script=no,downscript=no
~~~

Para finalizar tendremos que hacer eth0 y eth1 master de bond0 con los mismo comandos anteriores.

~~~
ip link set dev tap0 master bond0
~~~

~~~
ip link set dev tap1 master bond0
~~~

Ahora subimos el bond0 con el comando.

~~~
ip l set dev bond0 up
~~~

Y le configuramos una ip en mi caso es la siguiente.

~~~
ip a add 192.168.0.1/24 dev bond0
~~~

Por último en el otro lado del bond0 tendremos que configurar una ip para que puedan hacer ping entre ellas para ello le ponemos una ip del mismo rango que la red anterior.

~~~
ip a add 192.168.0.2/24 dev bond0
~~~

Para ver las redes que tenemos en nuestro bond podremos verlas abriendo el siguiente fichero.

~~~
cat /proc/net/bonding/bond0
~~~

~~~
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: load balancing (round-robin)
MII Status: up
MII Polling Interval (ms): 0
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: tap0
MII Status: up
Speed: 10 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: c6:6d:a8:98:fb:cd
Slave queue ID: 0

Slave Interface: tap1
MII Status: up
Speed: 10 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: be:06:a1:85:66:33
Slave queue ID: 0
~~~