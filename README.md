# Implementación de MPTCP v1 en un escenario virtual con VirtualBox

En este repositorio se muestran los pasos a seguir para la elaboración de un escenario simple con dos máquinas virtuales conectadas entre sí entre las que se ejecuta MPTCP v1. Los pasos a seguir se describen a continuación.

### Instalación de las máquinas virtuales
Para crear el escenario se ha utilizado la herramienta *Oracle VM VirtualBox*, un software de virtualización para arquitecturas x86/amd64. Es recomendable utilizar la última versión que haya en el momento y que no muestre problemas, ya que ofrece mayor compatibilidad con máquinas con *kernels* nuevos. Se deben seguir las siguientes instrucciones:
- Ir a la página oficial de Oracle y descargar la versión correspondiente del [software](https://www.virtualbox.org/wiki/Downloads).
- Ir a la página oficial de Ubuntu y descargar la [imagen](https://releases.ubuntu.com/jammy/) de la versión de Ubuntu que se quiera para las máquinas. La última versión disponible en este momento es la 22.10, pero al no ser LTS (*Long Term Support*) se escoge la 22.04 (una versión LTS implica que tendrá soporte y será actualizada durante más tiempo que una versión normal, y además suele ser más estable y ha sido probada por más usuarios).
- Crear dos máquinas virtuales en VirtualBox con dicha imagen (en este caso se les ha dado el nombre *Client_kernelOficial* y *Server_kernelOficial*) con las siguientes características (a modo de guía):
	- Memoría de 2048 MB y 2 núcleos.
	- Disco virtual de 20 GB mínimo.
	- 4 interfaces de red (Adaptador NAT y 3 interfaces conectadas a una misma red interna, a ésta se le ha dado el nombre de *ue1-ue2_v1_2*).

Una vez se han instalado ambas máquinas  en VirtualBox, se inician y se ponen en marcha siguiendo los pasos de instalación de Ubuntu que van apareciendo en pantalla.

###  Instalación de las dependencias necesarias para poder compilar el *kernel*
Antes de intentar compilar e instalar el *kernel* que se quiera en cada una de las máquinas, es necesario instalar una serie de herramientas, las cuales permitirán la posterior compilación del *kernel* sin errores. Estas dependencias se pueden ver en la [*wiki* de *Ubuntu*](https://wiki.ubuntu.com/KernelTeam/GitKernelBuild).

### Instalación del *kernel* de Linux 6.4-rc5
Si se han seguido los pasos del apartado anterior, ya se tienen las dos máquinas virtuales con Ubuntu 22.04 LTS funcionando, y con las dependencias necesarias instaladas. Ahora es momento de elegir el kernel sobre el que se va a trabajar. Echando un vistazo a la  web [MultiPath TCP - Linux Kernel implementation](https://multipath-tcp.org/) (esta *wiki* se encarga de la implementación de MPTCP v0 en el *kernel* de Linux), se observa como los *kernels* que ofrecen soporte por defecto a la versión 1 de MPTCP son aquellos que empiezan en la versión v5.6.

El *kernel* más moderno que actualmente ofrece soporte a MPTCP v1 es el 6.4-rc5, por lo que el escenario se va a montar sobre máquinas que lo incluyen (cuando el usuario este leyendo esto, habrán salido *kernels* más nuevos, pero los pasos a seguir que se describen en este repositorio siguen siendo válidos).

Pues bien, en cada una de ellas:
Si se accede a los [repositorios oficiales de Linux](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/), se puede descargar la carpeta comprimida que contiene el código fuente de dicho *kernel*.  Dentro de ella, se deben seguir los siguientes pasos:
~~~
sudo make clean
sudo make menuconfig
sudo make -j`nproc`
sudo make -j`nproc` bindeb-pkg
~~~

Cabe destacar que al llevar a cabo el comando `sudo make menuconfig`, es importante comprobar que la sección *Networking support->Networking options->TCP/IP networking->MPTCP protocol (MPTCP)* está marcada con un *.

Una vez se han seguido los pasos, se han generado cuatro paquetes: `linux-headers-*.deb`, `linux-image-*.deb`,  `linux-libc-*.deb` y `linux-image-dbg*.deb`.   El último paquetes está relacionado con temas de *debug*, por lo que no es necesario instalarlo y se puede eliminar, los demás se deben instalar con el comando `sudo dpkg -i *.deb`.

A continuación se puede descargar el archivo *grub-menu.sh* del repositorio de [Jorge Navarro Ortiz](https://github.com/jorgenavarroortiz/multitechnology_testbed_v0/blob/main/vagrant/vagrant/MPTCP0.96_kernel5.4.144_WRR05/grub-menu.sh), y ejecutarlo mediante `./grub-menu-sh`. Este mostrará un menu en el que se visualiza una secuencia de números al lado de cada uno de los *kernels* disponibles en el sistema. La secuencia que pertenece al *kernel* 6.4-rc5, que será de la forma '1>0', se debe cambiar por el valor que tenga GRUB_DEFAULT en el archivo /etc/default/grub (de la forma GRUB_DEFAULT="1>0"). Seguidamente se ejecuta `sudo update-grub` y se reinicia la máquina con `sudo reboot`. Las configuraciones en grub garantizan que cada vez que la máquina virtual se encienda, se selecciona el *kernel* 6.4-rc5 por defecto.

Al reiniciar la máquina (en este caso se muestra el cliente), se puede comprobar como ahora se tiene instalado y funcionando dicho kernel con el comando `uname -r`, y cómo mediante el comando `sysctl net.mptcp.enabled` se puede comprobar que MPTCP viene soportado por defecto sin tener que instalar ningún parche para ello:

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

Todas las dependencias mencionadas se pueden instalar siguiendo los enlaces mostrados a través de archivos comprimidos o simplemente utilizando el comando `$ sudo apt install <nombre de la dependencia>`. Este comando instalará la versión más reciente de dicho paquete que hay disponible para su sistema.

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

- Encabezados de API de usuario de MPTCP del kernel   de Linux: la biblioteca que contiene los encabezados de API de usuario de MPTCP es la "libmnl-dev" (Netlink Library). Es una biblioteca C que proporciona una interfaz de programación de aplicaciones (API) para la creación y el manejo de mensajes de netlink en el espacio de usuario de Linux, incluye las funciones necesarias para interactuar con el módulo de kernel MPTCP y crear y enviar mensajes de netlink para configurar y controlar la conexión MPTCP. Los encabezados de API de usuario de MPTCP se encuentran en el archivo "mnl/mptcp.h" dentro de la biblioteca libmnl.


#### Instalación de mptcpd
Lo primero que se debe hacer es navegar hasta la carpeta donde se ha descargado  el demonio y observar que hay un fichero *bootstrap*, como su nombre indica es un fichero de arranque y es necesario ejecutarlo para que se creen los archivos necesarios para continuar con la instalación del demonio. Así, se ejecuta `$ ./bootstrap`.

mptcpd sigue un procedimiento de compilación similar al que siguen los paquetes de software habilitados para Autotool, por lo que lo siguiente que hay que hacer es ejecutar el script *configure* en el directorio deseado y ejecutar `$ make` después. Antes de esto, es importante observar las diferentes opciones de ejecución que tiene el fichero *configure* mediante `$ configure --help`. Una de las opciones es `$ configure --with-kernel=upstream`.

Con todo esto, se ejecuta:
~~~
./configure --with-kernel=upstream
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
Lo siguiente que se debe hacer es configurar el routing entre ambas máquinas. Lo primero que hay que hacer es darle dirección IP a cada una de las interfaces de ambas máquinas (para esta prueba se van a establecer todas en la misma red con IP 10.1.1.0/24). Así, se accede al archivo ***/etc/netplan/01-network.manager-all.yaml*** y se configura la IP de cada interfaz:
- Para la máquina Ue1: enp0s8 (10.1.1.1), enp0s9 (10.1.1.2) y enp0s10 (10.1.1.3).
- Para la máquina Ue2: enp0s8 (10.1.1.4), enp0s9 (10.1.1.5) y enp0s10 (10.1.1.6).

Después de debe ejecutar `$ sudo netplan apply` para que los cambios se guarden.

##### Ue1
A continuación, se deben crear 3 tablas de enrutamiento; para ello se accede al fichero ***/etc/iproute2/rt_tables*** y se añaden 3 tablas con ID 100, 200 y 300 y nombres table1,table2 y table3 sucesivamente. Una vez creadas, se añaden las IP de origen correspondientes:
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
  
  ##### Ue2
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
~~~

Con esto, el routing de ambas máquinas está configurado. Para comprobarlo se pueden ejecutar los comandos `$ip rule show`, `$ip route` o  `$ip route show table ID`.

### Configuración de Path-manager
El path-manager se configura usando la herramienta `ip mptcp`.En concreto, se emplean dos comandos:
- ip mptcp limits set subflow NR add_addr_accepted NR.
- ip mptcp endpoint add  'ip'  dev 'iface' 'subflow|signal'.

El primero se utiliza para establecer el número de direcciones IP que se van a admitir por subflujo y el segundo para establecer las direcciones IP de las interfaces que van a funcionar para subflujos.

La máquina Ue1 se va a utilizar como servidor y la Ue2 como cliente.

##### Ue1
- Se agrega un punto final MPTCP para cada dirección IP en las tres interfaces de red:
~~~
ip mptcp endpoint add 10.1.1.1 dev enp0s8 subflow
ip mptcp endpoint add 10.1.1.2 dev enp0s9 subflow
ip mptcp endpoint add 10.1.1.3 dev enp0s10 subflow
~~~

- Se establece un límite de 3 direcciones IP adicionales para cada subflujo MPTCP:
~~~
ip mptcp limits set subflow 1 add_addr_accepted 3
~~~

##### Ue2
- Se agrega un punto final MPTCP para cada dirección IP en las tres interfaces de red:
~~~
ip mptcp endpoint add 10.1.1.4 dev enp0s8 subflow
ip mptcp endpoint add 10.1.1.5 dev enp0s9 subflow
ip mptcp endpoint add 10.1.1.6 dev enp0s10 subflow
~~~

- Se establece un límite de 3 direcciones IP adicionales para cada subflujo MPTCP:
~~~
ip mptcp limits set subflow 1 add_addr_accepted 3
~~~

De esta forma, ambas máquinas estarían configuradas para transmitir y recibir datos por todas sus interfaces.

### Sockets
Lo último que se debe hacer es obligar a las aplicaciones a que creen sockets MPTCP en lugar de TCP. Para ello, se puede usar IPPROTO_MPTCP como prototipo: (socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP);) o el comando `mptcpize` incluido con el demonio mptcpd. En este caso, se usa mptcpize y las herramientas iperf3 e ifstat. Ambas se pueden instalar ejecutando:
~~~
sudo apt install iperf3
sudo apt install ifstat
~~~

### Pruebas
Para realizar las pruebas se instala en las máquinas (no importa si en Ue1 o Ue2) Wireshark y se pone a capturar. Seguidamente se ejecuta:
~~~
- mptcpize run iperf3 -s & ifstat 
- mptcpize run iperf3 -c 10.1.1.1 & ifstat 
~~~

### Resultados
