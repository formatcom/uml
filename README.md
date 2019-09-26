# User-mode linux kernel

## 1.- Descargar ultima version del linux kernel
~~~
host $ curl -LO https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.1.tar.xz
host $ tar xvf linux-5.3.1.tar.xz
host $ cd linux-5.3.1
~~~

## 2.- Configuracion manual ( Si quieres utilizar ipv6 debes darle soporte aqui )
~~~
host $ make menuconfig ARCH=um SUBARCH=x86_64
~~~

## 3.- Compilar el kernel modo User-mode linux [uml]
~~~
host $ make -j2 linux ARCH=um SUBARCH=x86_64    # -j numero de cores para compilar
host # ln -s $PWD/linux-5.3.1/linux   /bin/linux
host # ln -s $PWD/linux-5.3.1/vmlinux /bin/vmlinux
host $ cd ..
~~~

## 4.- Crear un rootfs (alpine)
~~~
host $ export REPO=http://dl-cdn.alpinelinux.org/alpine/v3.10/main
host $ dd if=/dev/zero of=rootfs.ext4 bs=1M seek=50 count=0
host $ mkfs.ext4 rootfs.ext4
host $ mkdir rootfs
host # losetup /dev/loop0 rootfs.ext4
host # mount /dev/loop0 rootfs
host # chown -R formatcom:formatcom rootfs
host $ curl -LO $REPO/main/x86_64/apk-tools-static-2.10.4-r2.apk 
host $ tar -xvf apk-tools-static-2.10.4-r2.apk -C rootfs
host # ./rootfs/sbin/apk.static --repository $REPO \
	--update-cache --allow-untrusted --root $PWD/rootfs \
	--initdb add alpine-base
host # chown -R formatcom:formatcom rootfs
~~~

## 5.- Compilar y agregar modulos del kernel al rootfs
~~~
host $ cd linux-5.3.1
host # mount /dev/loop0 mods
host $ make modules INSTALL_MOD_PATH=mods ARCH=um SUBARCH=x86_64
host $ make modules_install INSTALL_MOD_PATH=mods ARCH=um SUBARCH=x86_64
host # umount mods
host # losetup -d /dev/loop0
host $ cd ..
~~~

## 6.- Habilitar iniciar session desde tty0 
### (terminal donde se ejecuta el boot) modificar rootfs/etc/securetty
~~~
console
tty0
tty1
tty2
tty3
tty4
tty5
tty6
tty7
tty8
tty9
tty10
tty11
~~~

## 7.- Redirigir tty1 -> tty0 modificar rootfs/etc/inittab
~~~
# /etc/inittab

::sysinit:/sbin/openrc sysinit
::sysinit:/sbin/openrc boot
::wait:/sbin/openrc default

# Set up a couple of getty's
tty1::respawn:/sbin/getty 38400 tty0

# Put a getty on the serial port
#ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100

# Stuff to do for the 3-finger salute
::ctrlaltdel:/sbin/reboot

# Stuff to do before rebooting
::shutdown:/sbin/openrc shutdown
~~~

## 8.- Agregar al arranque el script de hostname y otros con openrc /etc/runlevels
~~~
host $ ln -s /etc/init.d/hostname rootfs/etc/runlevels/sysinit/hostname
host $ ln -s /etc/init.d/networking rootfs/etc/runlevels/sysinit/networking
host $ ln -s /etc/init.d/modules rootfs/etc/runlevels/sysinit/modules
~~~

## 9.- Agregar soporte para asignar el hostname desde los args del kernel 
### modificar rootfs/etc/init.d/hostname
~~~
#!/sbin/openrc-run

description="Sets the hostname of the machine."

depend() {
	keyword -prefix -lxc
}

start() {

	KERNEL_ARGS=$(dmesg | grep "Kernel command line")
	HOSTNAME=$(echo $KERNEL_ARGS | grep -o "hostname=.*" | cut -d'=' -f2-)
	HOSTNAME=${HOSTNAME%% *}

	if [ -n "$HOSTNAME" ] ; then
		opts=$HOSTNAME
	elif [ -s /etc/hostname ] ; then
		opts="-F /etc/hostname"
	else
		opts="${hostname:-localhost}"
	fi
	ebegin "Setting hostname"
	hostname $opts
	eend $?
}
~~~

## 10.- Agregar rootfs/etc/network/interfaces
~~~
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback
~~~

## 11.- Ejecutar Kernel uml

### Forma mas basica
~~~
host $ linux umid=uml1 ubda=rootfs.ext4 rw

umid ->  identificador de la maquina virtual
ubda ->  virtual block device
rw   ->  lectura/escritura
~~~

### Usando nuestro hostname
~~~
host $ linux umid=uml1 hostname=uml1 ubda=rootfs.ext4 rw
~~~

## 12.- Networking tuntap
~~~
host # ip tuntap add tap0 mode tap 
host # ip link set tap0 up
host # ip address add 192.168.100.100/24 dev tap0
host $ linux umid=uml1 hostname=uml1 eth0=tuntap,tap0 ubda=rootfs.ext4 rw

uml1 # ip link set eth0 up
uml1 # ip address add 192.168.100.101/24 dev eth0
uml1 # ping 192.168.100.100
~~~

## NAT UML con el HOST
### HOST
~~~
host $ lsmod | grep iptable_nat
host $ modprobe iptable_nat
host $ sysctl net.ipv4.ip_forward
host # sysctl -w net.ipv4.ip_forward=1

host # iptables -t nat -A POSTROUTING -o wlp2s0 -j MASQUERADE
host # iptables -A FORWARD -i eth1 -j ACCEPT
~~~

### UML
~~~
uml1 # route add default gw 192.168.100.100
~~~

## PREGUNTAR ESTAS DOS REGLAS
### Solucion del bloqueo
~~~
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
~~~
	

## NOTAS (DOCUMENTAR ESTO EN MI REPOSITORIO DE NETWORKING)
~~~
host # tcpdump -i tap0 -XX
~~~


## TODO: /etc/resolv.conf
## TODO: COW: Copy-On-Write
