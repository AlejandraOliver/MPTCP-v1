# MPTCP v1

En este repositorio se muestran los pasos a seguir para la elaboración de un escenario simple con dos máquinas virtuales conectadas entre sí entre las que se ejecuta MPTCP v1.
### Instalación de las máquinas virtuales
Para crear el escenario se ha utilizado la herramienta *Oracle VM VirtualBox*, un software de virtualización para arquitecturas x86/amd64. Es recomendable utilizar la última versión que haya en el momento y que no muestre problemas ya que ofrece mayor compatibilidad con máquinas con kernels nuevos. Los pasos a seguir son:
- Ir a la página oficial de Oracle y descargar la versión correspondiente del [software](https://www.virtualbox.org/wiki/Downloads).
- Ir a la página oficial de Ubuntu y descargar la [imagen](https://releases.ubuntu.com/jammy/) de la versión de Ubuntu que se quiera para las máquinas. La última versión disponible en este momento es la 22.10 pero al no ser LTS (Long Term Support) se escoge la 22.04. (Que una versión sea LTS implica que tendrá soporte y será actualizada durante más tiempo que una versión normal y además suele ser más estable y ha sido probada por más usuarios).
- Crear dos máquinas virtuales en VirtualBox con dicha imagen (Ue1 y Ue2) con las siguientes características (a modo de guía):
	- Memoría de 2048 MB y 2 núcleos.
	- Disco virtual de 20 GB mínimo.
	- 4 interfaces de red (Adaptador NAT y 3 interfaces conectadas a una misma red interna con nombre *ue1-ue2*).

Una vez se han instalado ambas máquinas  en VirtualBox, se inician y se ponen en marcha siguiendo los pasos de instalación de Ubuntu que van apareciendo en pantalla.

### Instalación del kernel de Linux 6.1.18
Si se han seguido los pasos del apartado anterior, ya se tienen las dos máquinas virtuales con Ubuntu 22.04 funcionando. Ahora es momento de elegir el kernel sobre el que se va a trabajar. Echando un vistazo a la wiki del proyecto [*Upstream MPTCP*](https://github.com/multipath-tcp/mptcp_net-next/wiki)  (comunidad que se encarga de desarrollar, mantener y mejorar el protocolo Multipath TCP (MPTCP) (v1/RFC 8684) en el kernel de Linux ascendente). Los *upstream Linux kernels* son aquellos que empiezan en la versión 5.6 y posteriores. 
En el apartado ChangeLog se pueden ver los diferentes kernels a partir de los cuales la versión 1 MPTCP ya viene soportada, y las funcionalidades que añaden cada uno. Lo ideal sería escoger la versión más reciente del kernel pero por los mismos motivos por lo que se escogió Ubuntu 22.04, se va a escoger el kernel versión 6.1, en concreto, la versión 6.1.18 (casi diariamente surgen versiones nuevas por lo que es posible que cuando el usuario esté leyendo este guión, esta versión ya no sea la última del kernel 6.1).
Para llevar esto a cabo, basta con descargar los paquetes necesarios e instalarlos. Hay muchos sitios web donde se pueden encontrar pero uno de los más sonados es [este](https://kernel.ubuntu.com/~kernel-ppa/mainline/).
Los comandos a seguir son los siguientes:
~~~
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1.18/amd64/linux-headers-6.1.18-060118-generic_6.1.18-060118.202303111330_amd64.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1.18/amd64/linux-headers-6.1.18-060118_6.1.18-060118.202303111330_all.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1.18/amd64/linux-image-unsigned-6.1.18-060118-generic_6.1.18-060118.202303111330_amd64.deb
wget -c https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.1.18/amd64/linux-modules-6.1.18-060118-generic_6.1.18-060118.202303111330_amd64.deb
~~~
Una vez descargados los paquetes, se deben instalar y reiniciar la máquina para que los cambios se guarden:
~~~
sudo dpkg -i *.deb
reboot
~~~
Al reiniciarla se puede comprobar como ahora se tiene instalado y funcionando dicho kernel con el comando `uname -r` y como MPTCP viene soportado por defecto sin tener que instalar ningún parche para ello, mediante `sysctl net.mptcp.enabled`.

![Captura-de-pantalla-2023-03-13-192858.png](https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-03-13%20192858.png)

### Instalación del Demonio mptcpd y sus Dependencias

#### Dependencias necesarias
Aunque en los upstream kernels MPTCP esté soportado por defecto, es necesario instalar el demonio mptcpd para su correcto funcionamiento. Este demonio realiza operaciones relacionadas con la gestión de rutas MPTCP en el espacio del usuario; además, interactúa con el kernel de Linux a través de una conexión netlink genérica para rastrear información por conexión (direcciones remotas disponibles, interfaces de red disponibles, solicitud de nuevos subflujos MPTCP, manejo de solicitudes de subflujos, etc).
El paquete se puede descargar del propio [repositorio de Ubuntu oficial](https://packages.ubuntu.com/jammy/mptcpd)  en su versión 0.9 o clonando el [repositorio de github](https://github.com/multipath-tcp/mptcpd) de *ossama-othman* cuya versión es la 0.12 (al instalarlo aparece el demonio como versión 0.9 y la herramienta mptcpize como 0.12). En ambos casos se pueden descargar dos librerías adicionales si se quiere estar seguro de que todas las funcionalidades de mptcpd están instaladas, estas dos librerías son [*libmptcpwrap0*](https://packages.ubuntu.com/jammy/amd64/libmptcpwrap0) (Multipath TCP Converter Library) y[ *libmptcpd3*](https://packages.ubuntu.com/jammy/libmptcpd3) (Multipath TCP Daemon Library). Para este caso, se ha descargado el demonio desde github y ambas librerías de los repositorios de Ubuntu.

Antes de poder instalarlo es necesario instalar en nuestra/s máquinas una serie de dependencias de compilación:
- [C compiler (compatible con C99)](https://0and6.wordpress.com/2017/06/04/instalar-compilador-de-c-en-ubuntu/)
- [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/)
- [GNU Autoconf](https://www.gnu.org/software/autoconf/)
- [GNU Automake](https://www.gnu.org/software/automake/)
- [GNU Libtool](https://www.gnu.org/software/libtool/)
- [GNU Autoconf-archive](https://www.gnu.org/software/autoconf-archive/)
- [Pandoc >= 2.2.1](https://pandoc.org/installing.html)
- [Doxygen](https://www.doxygen.nl/download.html)

Todas las dependencias mencionadas se pueden instalar siguiendo los enlaces mostrados a través de archivos comprimidos o simplemente utilizando el comando `$ sudo apt install <nombre de la dependencia`. Este comando instalará la versión más reciente de dicho paquete que hay disponible para su sistema.

- Biblioteca Argp. Para instalar esta biblioteca se han seguido los pasos que ofrece [*Alexander Reguiero*](https://github.com/alexreg/libargp) en su plataforma de github pero reemplazando algunos archivos por los que ofrece [*Érico Nogueira Rolim*](https://github.com/ericonr/argp-standalone) en su repositorio. Esto se ha hecho así ya que éste segundo ofrece soluciones a los problemas de compilación de los archivos del primer repositorio.

- [Biblioteca de Linux integrada >= v0.30](https://git.kernel.org/pub/scm/libs/ell/ell.git/). Este paquete se puede instalar con los siguiente comandos:
~~~
git clone https://git.kernel.org/pub/scm/libs/ell/ell.git
cd ell
autoreconf -i
mkdir build
cd build
../configure
make
make check
sudo make install
~~~
` $ autoreconf -i` se emplea para generar los archivos de configuración necesarios como el archivo *configure*. ` $ ../configure` se emplea para configurar la librería y `$ make` para construirla. Finalmente. se puede ejecutar `$ make check` para ejecutar la carpeta de tests y comprobar que la libería funciona correctamente y `$ sudo make install` para terminar de instalarla junto con todos los archivos de cabecera que faltan.

- Encabezados de API de usuario de MPTCP del kernel   de Linux.

#### Instalación de mptcpd
Lo primero que se debe hacer es navegar hasta la carpeta donde se ha descargado  el demonio y observar que hay un fichero *bootstrap*, como su nombre indica es un fichero de arranque y es necesario ejecutarlo para que se creen los archivos necesarios para continuar con la instalación del demonio. Así se ejecuta `$ ./bootstrap`.

mptcpd sigue un procedimiento de compilación similar al que siguen los paquetes de software habilitados para Autotool, por lo que lo siguiente que hay que hacer es ejecutar el script *configure* en el directorio deseado y ejecutar `$ make` después. Antes de esto, es importante observar las diferentes opciones de ejecución que tiene el fichero *configure* mediante `$ configure --help`. Se observa como una de las opciones es `$ configure --with-path-manager`.

Con todo esto, se ejecuta:
~~~
./configure
sudo make
sudo make check
sudo make install
~~~
Para comenzar mptcpd inmediatamente después de la instalación, basta con ejecutar los siguientes comandos:
~~~
systemctl daemon-reload
systemctl start mptcp.service
~~~
![Captura-de-pantalla-2023-03-15-20122503.png](https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-03-15%20122503.png)

### Configuración de Routing
Lo siguiente que se debe hacer es configurar el routing entre ambas máquinas. Lo primero que hay que hacer es darle dirección IP a cada una de las interfaces de ambas máquinas (para esta prueba se van a establecer todas en la misma red con IP 10.1.1.0/24). Así, se accede al archivo ***/etc/netplan/01-network.manager-all.yaml* **y se configura la IP de cada interfaz:
- Para la máquina Ue1: enp0s8 (10.1.1.1), enp0s9 (10.1.1.2) y enp0s10 (10.1.1.3).
- Para la máquina Ue2: enp0s8 (10.1.1.4), enp0s9 (10.1.1.5) y enp0s10 (10.1.1.6).

Después de debe ejecutar `$ sudo netplan apply` para que los cambios se guarden.

#####Ue1
A continuación, se deben crear 3 tablas de enrutamiento; para ello se accede al fichero ****/etc/iproute2/rt_tables*** y se añaden 3 tablas con ID 100, 200 y 300 y nombres table1,table2 y table3 sucesivamente. Una vez creadas, se añaden las IP de origen correspondientes:
~~~
ip rule add from 10.1.1.1 table table1
ip rule add from 10.1.1.2 table table2
ip rule add from 10.1.1.3 table table3
~~~
Lo siguiente que se debe hacer es configurar dichas tablas. Para ello, se ejecutan:
~~~
ip route add 10.1.1.0/24 dev enp0s8 scope link table table1
ip route add default via 10.1.1.4 dev enp0s8 table table1
ip route add 10.1.1.0/24 dev enp0s9 scope link table table2
ip route add default via 10.1.1.4 dev enp0s9 table table2
ip route add 10.1.1.0/24 dev enp0s10 scope link table table3
ip route add default via 10.1.1.4 dev enp0s10 table table3
  ~~~
  
  #####Ue2
Se crean 3 tablas igual que en Ue1 y se añaden las IP de origen correspondientes:
~~~
ip rule add from 10.1.1.4 table table1
ip rule add from 10.1.1.5 table table2
ip rule add from 10.1.1.6 table table3
~~~
Lo siguiente que se debe hacer es configurar dichas tablas. Para ello, se ejecutan:
~~~
ip route add 10.1.1.0/24 dev enp0s8 scope link table table1
ip route add default via 10.1.1.1 dev enp0s8 table table1
ip route add 10.1.1.0/24 dev enp0s9 scope link table table2
ip route add default via 10.1.1.1 dev enp0s9 table table2
ip route add 10.1.1.0/24 dev enp0s10 scope link table table3
ip route add default via 10.1.1.1 dev enp0s10 table table3
