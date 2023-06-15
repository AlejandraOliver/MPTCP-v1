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

###  Instalación del *kernel* de Linux 6.4-rc5
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

<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-06-15%20200233.png" width="500" />
</p>

### Configuración del *routing* en las máquinas
Lo siguiente que se debe hacer es configurar el routing en ambas máquinas.La estructura que se pretende seguir es la siguiente:
<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-06-15%20202932.png" width="500" />
</p>

Como se observa, cada máquina tiene 3 interfaces conectadas a una misma red interna *ue_ue_v1_2* a través de las cuales se implementará MPTCP, además de una cuarta interfaz a NAT a la que se le asigna IP mediante DHCP, por lo que de ésta última no hay que preocuparse. Pues bien, lo primero que hay que hacer es darle dirección IP a cada una de las interfaces de la red interna, para ello se accede al archivo */etc/netplan/01-network.manager-all.yaml* y se configura:

- Para la máquina *Client_kernelOficial*: enp0s8 (10.1.1.1), enp0s9 (10.1.1.2) y enp0s10 (10.1.1.3).
- Para la máquina *Server_kernelOficial*: enp0s8 (10.1.1.4), enp0s9 (10.1.1.5) y enp0s10 (10.1.1.6).

Después de debe ejecutar `$ sudo netplan apply` para que los cambios se guarden.

#### *Client_kernelOficial*
A continuación, se deben crear 3 tablas de enrutamiento basadas en la IP de origen:
~~~
sudo ip rule add from 10.1.1.1 table 10
sudo ip rule add from 10.1.1.2 table 20
sudo ip rule add from 10.1.1.3 table 30
~~~
Lo siguiente que se debe hacer es configurar dichas tablas. Para ello, se ejecutan 2 comandos para cada una de ellas: el primero indica que cualquier tráfico destinado a la red con el prefijo 10.1.1.0/24, debe enviarse directamente a través del dispositivo de red enp0s8, enp0s9 o enp0s10 (dependiendo de la tabla que se esté configurando). El segundo indica que para el tráfico que salga de la interfaz enp0s8 (tabla 1), enp0s9 (tabla 2) o enp0s10 (tabla 3), y que no tenga destino definido, la IP por defecto será la IP de la interfaz enp0s8 del otro extremo (10.1.1.4).
~~~
sudo ip route add 10.1.1.0/24 dev enp0s8 scope link table 10
sudo ip route add default via 10.1.1.4 dev enp0s8 table 10

sudo ip route add 10.1.1.0/24 dev enp0s9 scope link table 20
sudo ip route add default via 10.1.1.4 dev enp0s9 table 20

sudo ip route add 10.1.1.0/24 dev enp0s10 scope link table 30
sudo ip route add default via 10.1.1.4 dev enp0s10 table 30
~~~
Además se deben emplear dos comandos más:
~~~
sudo ip route add default scope global nexthop via 10.1.1.4 dev enp0s8
sudo ip route del 169.254.0.0/16
~~~
El primero de ellos establece la ruta por defecto para el proceso de selección del tráfico normal de Internet, y el segundo elimina un enlace local que se crea por defecto y que no es necesario, lo único que hace es entorpecer el *routing* creado.

#### *Server_kernelOficial*
Siguiendo los mismo pasos que para el cliente, pero con sus IP correspondientes, se lleva a cabo lo siguiente:
Se crean las tablas:
~~~
sudo ip rule add from 10.1.1.4 table 10
sudo ip rule add from 10.1.1.5 table 20
sudo ip rule add from 10.1.1.6 table 30
~~~
Se configuran dichas tablas:
~~~
sudo ip route add 10.1.1.0/24 dev enp0s8 scope link table 10
sudo ip route add default via 10.1.1.1 dev enp0s8 table 10 

sudo ip route add 10.1.1.0/24 dev enp0s9 scope link table 20
sudo ip route add default via 10.1.1.1 dev enp0s9 table 20

sudo ip route add 10.1.1.0/24 dev enp0s10 scope link table 30
sudo ip route add default via 10.1.1.1 dev enp0s10 table 30
~~~
Se emplean los otros dos comandos:
~~~
sudo ip route add default scope global nexthop via 10.1.1.1 dev enp0s8
sudo ip route del 169.254.0.0/16
~~~

Con todo ello se puede comprobar que el *routing* se ha establecido de forma correcta ejecutan los comandos que se muestran en las dos imágenes siguientes:

*Routing* en el cliente            |  *Routing* en el servidor
:-------------------------:|:-------------------------:
![1](https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-06-15%20205158.png)  |  ![2](https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-06-15%20205332.png)


### Configuración del *scheduling* en las máquinas
Los *kernels* oficiales de Linux a partir de v5.6 ofrecen soporte al protocolo, pero no todas las funcionalidades de éste. A diferencia de MPTCP v0, el cual poseía varios tipos de *schedulers*, el protocolo en su versión 1 implementado en estos *kernels* solo posee un tipo de *scheduler*, el *scheduler* por defecto. Éste envía los datos por todos los subflujos disponibles, buscando el mejor rendimiento. Por todo ello, no hay nada que configurar en este ámbito.

### Configuración del *path management* en las máquinas
A la hora de implementar la gestión de rutas, se tienen dos tipos de *path manager*:
- *In-kernel path manager*: este gestor de rutas se implementa en el propio kernel. Para configurarlo se hace uso del comando `ip mptcp' (incluido en iproute2-ss200602), estableciendo así cómo se van a intentar crear los subflujos, qué direcciones van a ser anunciadas, etc.
- *User-space path manager*: este gestor de rutas se implementa en el espacio de usuario a través del demonio *mptcpd*. Éste interactua con el *kernel* de Linux haciendo uso de una conexión *netlink* genérica, siendo así capaz de solicitar nuevos subflujos, gestionar las solicitudes por parte de los otros extremos, etc.

La principal tarea que va a tener que llevar cualquiera de los *path managers* mencionados, es establecer de qué tipo van a ser las interfaces de cada máquina. Éstas se van a poder configurar como *endpoints* de 4 tipos:
- Subflow: las interfaces configuradas de esta forma enviarán mensajes MP_JOIN para iniciar nuevos subflujos.
- Signal: la interfaz configurada así permite que su IP sea anunciada mediante mensajes ADD_ADDR. Ella nunca inicia un subflujo (no manda mensajes MP_JOIN).
- Fullmesh: las interfaces configuradas de esta forma enviarán mensajes MP_JOIN para iniciar nuevos subflujos a cada una de las demás interfaces de la otra máquina, formando una toplogía de subflujos todo-a-todo. 
- Backup: esta opción indica que los subflujos creados utilizándola, serán con mensajes MP_JOIN con el flag backup a 1, por lo que no se utilizarán hasta que sea necesario. Esta opción no puede ir junto con signal.

A continuación se describen los dos tipos de gestor de rutas.

#### Configuración del *path management* a través de *mptcpd*

**1. Instalación de dependencias y descarga del paquete *mptcpd***

Lo primero es descargar el paquete que contiene el demonio dentro de cada una de las máquinas. Éste se puede descargar del propio [repositorio de Ubuntu oficial](https://packages.ubuntu.com/jammy/mptcpd)  o clonando el [repositorio de github](https://github.com/multipath-tcp/mptcpd) de *ossama-othman*. En ambos casos se pueden descargar dos librerías adicionales si se quiere estar seguro de que todas las funcionalidades de mptcpd están instaladas, estas dos librerías son [*libmptcpwrap0*](https://packages.ubuntu.com/jammy/amd64/libmptcpwrap0) (Multipath TCP Converter Library) y[ *libmptcpd3*](https://packages.ubuntu.com/jammy/libmptcpd3) (Multipath TCP Daemon Library). En este caso, se ha descargado el demonio desde GitHub y ambas librerías de los repositorios de Ubuntu.

Antes de poder instalarlo es necesario instalar en nuestra/s máquinas una serie de dependencias de compilación:
- [C compiler (compatible con C99)](https://0and6.wordpress.com/2017/06/04/instalar-compilador-de-c-en-ubuntu/)
- [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/)
- [GNU Autoconf](https://www.gnu.org/software/autoconf/)
- [GNU Automake](https://www.gnu.org/software/automake/)
- [GNU Libtool](https://www.gnu.org/software/libtool/)
- [GNU Autoconf-archive](https://www.gnu.org/software/autoconf-archive/)
- [Pandoc >= 2.2.1](https://pandoc.org/installing.html)
- [Doxygen](https://www.doxygen.nl/download.html)

Todas las dependencias mencionadas se pueden instalar siguiendo los enlaces mostrados a través de archivos comprimidos o simplemente utilizando el comando `sudo apt install <nombre de la dependencia>`. Este comando instalará la versión más reciente de dicho paquete que hay disponible para su sistema.

- Biblioteca Argp. Para instalar esta biblioteca se han seguido los pasos que ofrece [*Alexander Reguiero*](https://github.com/alexreg/libargp) en su plataforma de github pero reemplazando algunos archivos por los que ofrece [*Érico Nogueira Rolim*](https://github.com/ericonr/argp-standalone) en su repositorio. Esto se ha hecho así ya que éste segundo ofrece soluciones a los problemas de compilación de los archivos del primer repositorio.

- [Biblioteca de Linux integrada >= v0.30](https://git.kernel.org/pub/scm/libs/ell/ell.git/). Este paquete se puede instalar con los siguiente comandos:
~~~
sudo git clone https://git.kernel.org/pub/scm/libs/ell/ell.git
cd ell
sudo autoreconf -i
sudo mkdir build
cd build
sudo ../configure
sudo make
sudo make check
sudo make install
~~~
` $ autoreconf -i` se emplea para generar los archivos de configuración necesarios como el archivo *configure*. ` $ ../configure` se emplea para configurar la librería y `$ make` para construirla. Finalmente. se puede ejecutar `$ make check` para ejecutar la carpeta de tests y comprobar que la libería funciona correctamente y `$ sudo make install` para terminar de instalarla junto con todos los archivos de cabecera que faltan.

- Encabezados de API de usuario de MPTCP del kernel  de Linux: la biblioteca que contiene los encabezados de API de usuario de MPTCP es la "libmnl-dev" (Netlink Library). Es una biblioteca C que proporciona una interfaz de programación de aplicaciones (API) para la creación y el manejo de mensajes de netlink en el espacio de usuario de Linux, incluye las funciones necesarias para interactuar con el módulo de kernel MPTCP y crear y enviar mensajes de netlink para configurar y controlar la conexión MPTCP. Los encabezados de API de usuario de MPTCP se encuentran en el archivo "mnl/mptcp.h" dentro de la biblioteca libmnl.

**2. Instalación de *mptcpd***

Lo primero que se debe hacer es navegar hasta la carpeta donde se ha descargado el demonio y observar que hay un fichero *bootstrap*, como su nombre indica es un fichero de arranque y es necesario ejecutarlo para que se creen los archivos necesarios para continuar con la instalación del demonio. Así, se ejecuta `./bootstrap`.

*mptcpd* sigue un procedimiento de compilación similar al que siguen los paquetes de software habilitados para*Autotool*, por lo que lo siguiente que hay que hacer es ejecutar el script *configure* en el directorio deseado y hacer `make` después. Antes de esto, es importante observar las diferentes opciones de ejecución que tiene el fichero *configure* mediante `configure --help`. Una de las opciones es `configure --with-kernel=upstream`.

Con todo esto, se ejecuta:
~~~
sudo ./configure --with-kernel=upstream
sudo make
sudo make check
sudo make install
~~~
Para comenzar *mptcpd* inmediatamente después de la instalación, basta con ejecutar los siguientes comandos:
~~~
systemctl daemon-reload
systemctl start mptcp.service
~~~
Aparecerá algo similar a lo siguiente:

<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Captura%20de%20pantalla%202023-06-15%20223013.png" width="500" />
</p>

**3. Configuración de *mptcpd.conf***
Lo siguiente que se debe hacer es modificar el fichero de configuración *mptcpd.conf*. Este fichero se encuentra en */usr/local/etc/mptcpd* de cada máquina, y le dice al demonio cómo debe configurar las interfaces. Para ello, el demonio hace uso de *plugins* que pueden ser construidos para servir algunos propósitos específicos usando la API C. Por defecto, viene con *plugins* que le dirán al gestor de rutas del *kernel* que use todas las IP disponibles como puntos finales para crear subflujos (caso de uso del cliente), pero esto se puede modificar fácilmente para marcar todos los puntos finales como signal en su lugar (caso de uso del servidor). Así, el fichero en el lado del cliente debe quedar como se muestra a continuación:
~~~
# SPDX-License-Identifier: BSD-3-Clause
#
# Copyright (c) 2018-2019, 2021-2022, Intel Corporation

# ------------------------------------------------------------------
#                     mptcpd Configuration File
# ------------------------------------------------------------------

[core]
# -------------------------
# Default logging mechanism
# -------------------------
#   stderr  - standard unbuffered I/O stream
#   syslog  - POSIX based system logger
#   journal - systemd Journal
#   null    - disable logging
log=stderr

# ----------------
# Plugin directory
# ----------------
plugin-dir=/usr/local/lib/mptcpd

# -------------------
# Path manager plugin
# ---------------------
# The plugin found with the most favorable (lowest) priority will be
# used if one is not specified.
path-manager=addr_adv

# --------------------------
# Address announcement flags
# --------------------------
# A comma separated list containing one or more of the flags:
#
#   subflow
#   signal    (do not use with the "fullmesh" flag)
#   backup
#   fullmesh  (do not use with the "signal" flag)
#
# Plugins that deal with the in-kernel path manager may use these
# flags when advertising addresses.
#
# See the ip-mptcp(8) man page for details.
#
addr-flags=subflow,fullmesh

# --------------------------
# Address notification flags
# --------------------------
# A comma separated list containing one or more of the flags:
#
#   existing
#     Notify plugins of the addresses that exist at mptcpd start
#
#   skip_link_local
#     Ignore (do not notify) [ipv6] link local address updates
#
#   skip_loopback
#     Ignore (do not notify) host (loopback) address updates
#
# These flags determine whether mptpcd plugins will be notified when
# related addresses are updated.
#
notify-flags=existing

# --------------------------
# Plugins to load
# --------------------------
# A comma separated list containing one or more plugins to load.
#
load-plugins=addr_adv
~~~

Como se observa, en la sección de *Plugin directory* se ha indicado el directorio donde se encuentra el *plugin* escogido, éste corresponde con la política que va a llevar el *path manager*, en este caso se escoge el *addr_adv* (se van a advertir las IP disponibles y no hay limitación de número de subflujos por interfaz). Seguidamente, se asigna esta política al *path manager*, y en la sección *Address announcement flags*, se establecen los endpoints (las interfaces) como *subflow* *fullmesh*, lo que indica que cada una intentará crear subflujos con mensajes MP_JOIN con cada una de las interfaces del servidor.

En el apartado *Address notification flags* se establecen las *flags* con la opción *existing*, para sean notificadas, y por último, se carga el plugin *addr_adv*.

Para el lado del servidor, la configuración es la misma salvo por al apartado *Address announcement flags*, el cual se pone a *signal*. Esto hace que las interfaces del servidor no se utilicen para crear nuevos subflujos, sino que anuncien sus IP para que sea el cliente el que los anuncie. Esto se hace así para simular los escenarios del Internet actual, en el que el cliente posiblemente esté detrás de un NAT o *firewall* y sea el servidor el que recibe muchas solicitudes de clientes, pero no sea el que empiece la conexión con ellos.

Una vez configurado el archivo *mptcpd.conf*, se deben volver a ejecutar los siguientes comandos con el fin de que los cambios en la configuración se guarden:
~~~
systemctl daemon-reload
systemctl start mptcp.service
~~~


#### Configuración del *path management* a través de *ip mptcp*
Este *path manager* está implementado en el propio *kernel* de la máquina y la configuración es similar a la que se lleva a cabo con el demonio, pero utilizando línea de comandos, en concreto, el comando `ip mptcp`.

Lo primero que se debe hacer es configurar cada interfaz de cada máquina como un *endpoint*. Al igual que antes, las interfces del cliente serán *subflow fullmesh*, y las del servidor *signal*. Por lo tanto se configura:

##### *Client_kernelOficial*
~~~
sudo ip mptcp endpoint add 10.1.1.1 dev enp0s8 subflow fullmesh
sudo ip mptcp endpoint add 10.1.1.2 dev enp0s9 subflow fullmesh
sudo ip mptcp endpoint add 10.1.1.3 dev enp0s10 subflow fullmesh
~~~

##### *Server_kernelOficial*
~~~
sudo ip mptcp endpoint add 10.1.1.4 dev enp0s8 signal
sudo ip mptcp endpoint add 10.1.1.5 dev enp0s9 signal
sudo ip mptcp endpoint add 10.1.1.6 dev enp0s10 signal
~~~

Después hay que ejecutar un comando más en cada una:
~~~
sudo ip mptcp limits set subflow 8 add_addr_accepted 8
~~~

Con este último comando se establecen dos cosas:
- El número máximo de subflujos adicionales que se van a poder crear en la conexión MPTCP entre las dos máquinas (sin contar con el que se crea para hacer la conexión). Este valor se establece a 8, para que puede haber subflujos entre las 3 interfaces de una máquina y las 3 de la otra (además esté es el limite máximo).
- El número máximo de mensajes ADD_ADDR que se van a permitir. También se establece a 8 en ambas, aunque en el caso del servidor no haría falta establecer límite ya que él solo recibirá mensajes MP_JOIN.


### Pruebas de verificación
En este último apartado se lleva a cabo una prueba de rendimiento para verificar que el sistema funciona.
#### Prueba de rendimiento con *mptcpd*
Si se han seguido los pasos mostrados en esta guía, en cuanto a creación de escenario en VirtualBox, configuración de *routing*, y configuración del demonio *mptcpd*, sólo faltan un par de pasos para poder comprobar que el sistema funciona.

Uno de los pasos es obligar a las aplicaciones a que creen sockets MPTCP en lugar de TCP. Para ello, se puede usar IPPROTO_MPTCP como prototipo: (socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP);) o el comando `mptcpize` incluido con el demonio *mptcpd*. En este caso, se usa *mptcpize*. Lo siguiente es instalar las herramientas `iperf3` e `ifstat`. `iperf3` permite hacer pruebas de rendimiento en la red, mientras que `ifstat` muestra las estadísticas de red de cada una de las interfaces de la máquina en tiempo real. Ambas se pueden instalar ejecutando:
~~~
sudo apt install iperf3
sudo apt install ifstat
~~~
Una vez instaladas, para realizar las pruebas se ejecuta:
~~~
mptcpize run iperf3 -s & ifstat (en el servidor)
mptcpize run iperf3 -c 10.1.1.1 & ifstat (en el cliente)
~~~
La salida que se obtiene es la siguiente:

<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Imagen1.png" width="800" />
</p>

#### Prueba de rendimiento con *ip mptcp*
Si se quiere probar el escenario, pero usando el *in-kernel path manager*, se deben seguir los pasos comentados (creación de escenario en VirtualBox, configuración de *routing*, y gestión de rutas (*path management*) mediante *ip mptcp*).

***IMPORTANTE: si se va a configurar el *in-kernel path manager* en las mismas máquinas donde se configuró el demonio *mptcpd*, es necesario pararlo antes de realizar cualquier tipo de *routing* u otra configuración mediante el comando `sudo systemctl stop mptcp.service`; además de eliminar los *endpoints* que se hayan creado usando `sudo ip mptcp endpoint flush`.***

Una vez dicho esto, se siguen los pasos de *ip mptcp* y se instalan tanto `iperf3` e `ifstat` (en caso de que sea el mismo escenario ya están instalados). Además, si en un prueba anterior se utilizó *mptcpd*, la herramienta `mptcpize` ya está instalada con versión 0.12, en caso contrario, se puede instalar con `sudo apt install mptcpize` en versión 0.9 (no hay diferencia entre versiones).

Ejecutando `mptcpize run iperf3 -s & ifstat` en el servidor, y `mptcpize run iperf3 -c 10.1.1.1 & ifstat` en el cliente, se obtiene lo siguiente:

<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/Imagen2.png" width="800" />
</p>

Como se ha podido observar, se utilizan todas las interfaces para la creación de subflujos con un *throughput* de 88 Mb/s por interfaz. Esto quiere decir que.


Por último, para probar el funcionamiento de backup, se establece la interfaz enp0s9 del cliente como backup (ya sea con *mptcpd* o con *ip mptcp*). Esto se puede hacer al inicio de la configuración, pero si se desea hacer en mitad de la conexión (opción MP_PRIO en MPTCP), se debe utilizar el comando `sudo ip mptcp endpoint change <IP> backup`. Esta opción de *ip mptcp* está disponible en la última versión de *iproute2*, la cual se puede descargar de [aquí](https://mirrors.edge.kernel.org/pub/linux/utils/net/iproute2/) e instalar. Se destaca que esta versión de *iproute* instala una nueva versión de `ifstat`, si se quiere volver a la anterior, se puede encontrar [aquí](https://packages.debian.org/buster/ifstat).

Se observa el siguiente comportamiento:

<p align="center">
  <img src="https://github.com/AlejandraOliver/MPTCP-v1/blob/main/ImagenesRepositorio/imagen3.png" width="600" />
</p>

La interfaz enp0s9 no se utiliza hasta que enp0s8 y enp0s10 se caen, en dicho momento se reactiva.

