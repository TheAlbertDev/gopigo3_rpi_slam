# Gmapping y SLAM en GoPiGo3

En la última sesión, generamos un mapa de la estancia en la que se encontraba GoPiGo3 (GPG3) utilizando su LiDAR y un algoritmo de seguimiento de pared que hacía que el robot navegara de manera autónoma por la estancia. El nodo *gmapping* recogía las distancias publicadas por el LiDAR junto con la odometría publicada por los motores del GPG3 para generar dicho mapa. Mientras que esto se hacía con un rendimiento medianamente aceptable, no era este el caso cuando le pedíamos a GPG3 que calculara las rutas posibles entre dos localizaciones del mapa para desplazarse. En ese instante, la Raspberry Pi (RPi) que controla el GPG3 está corriendo un sistema operativo (Linux), gestionando la conexión WiFi, transmitiendo en remoto su Escritorio mediante VNC, corriendo el master de ROS, ejecutando RViz con los pertinentes modelos 3D, calculando las rutas entre localizaciones, controlando los motores, haciendo *streaming* de la cámara, etc. Mucha carga computacional para una simple RPi. Por suerte, ROS es una red distribuida de nodos (¡su principal punto fuerte!), por lo que podemos repartir la carga computacional entre diferentes máquinas/ordenadores.

En esta sesión vamos a crear una red ROS distribuida en dos máquinas: la RPi y nuestro ordenador (PC). La primera se encargará de simplemente controlar los motores (que no la trayectoria a realizar) y publicar su odometría, encender el LiDAR y publicar las distancias medidas, y encender la cámara para publicar sus imágenes. Respecto la sesión anterior, la RPi **no** calculará el recorrido a realizar, **no** gestionará el master de ROS (y toda la gestión de paquetes que ello supone), **no** transmitirá su Escritorio mediante VNC, y **no** mostrará RViz ni sus modelos 3D. En cambio, ahora el PC será quien ejecute el master de ROS, quién ejecute RViz y sus modelos 3D, y quien calcule las trayectorias.

## Estructura del repositorio

Para esta sesión utilizaremos el repositorio disponible en: https://github.com/TheAlbertDev/gopigo3_rpi_slam

> Si visitas el repositorio, haces uso de él y te ha ayudado, dale amor con una ⭐️
> Su humilde autor te lo agradecerá 😉

El repositorio cuenta con dos carpetas: `pc` y `rpi`. Ambos son *workspaces* de ROS. El primero es el que ejecutaremos desde el ordenador, mientras que el segundo lo ejecutaremos en la RPi.

## Sincronización de fecha y hora

> Esta sección es importante que la leáis para profundizar en aspectos del funcionamiento de ROS y en su gestión en el día a día de un desarrollador de ROS, pero esto ya se os viene dado. **No hace falta que hagáis nada de esta sección.**

Antes de empezar con la sesión, cabe tener en cuenta el aspecto de sincronización entre máquinas. Para que la red funcione correctamente, es importante que los nodos funcionen de forma sincronizada (es decir, que todas las máquinas tengan la misma fecha y hora). Para asegurar que esto ocurre, se hace uso de una utilidad llamada [chrony](https://chrony.tuxfamily.org/). Esta utilidad se debe de instalar en cada una de las máquinas que se unan a la red ROS. Una de las máquinas hará de servidor (es decir, será quien indique la fecha/hora a utilizar) y las otras máquinas simplemente cogerán la fecha y hora indicada por el servidor.

Para instalar la utilidad, utilizamos el siguiente comando en cada una de las máquinas (en esta sesión , tanto en el PC como en la RPi):

```shell
sudo apt-get install chrony
```

### Configuración del servidor

Una vez instalado chrony, debemos configurarlo. Para el caso del servidor, ejecutamos el siguiente comando para abrir/editar la configuración:

```shell
sudo nano /etc/chrony/chrony.conf
```

Se nos abrirá [nano](https://www.nano-editor.org/) con la configuración. Al final del archivo, añadimos las siguientes líneas:

```
local stratum 8
allow 192.168.4.1
```

En este caso, la IP indicada es la de la RPi, pero si fuera otra IP o tuvierais que trabajar con más máquinas, simplemente indicadlas ahí. Guardamos y cerramos el editor (<kbd>CONTROL</kbd>+<kbd>O</kbd> y después <kbd>CONTROL</kbd>+<kbd>X</kbd>).

Ahora, debemos de reiniciar chrony para aplicar los cambios:

```shell
sudo /etc/init.d/chrony stop
sudo /etc/init.d/chrony start
```

### Configuración de los clientes

Los clientes son las máquinas que preguntan al servidor qué fecha/hora es. En este caso, la RPi. Vamos a abrir la configuración de chrony:

```shell
sudo nano /etc/chrony/chrony.conf
```

Y añadimos la siguiente línea al final de la configuración:

```
server 192.168.4.18 minpoll 0 maxpoll 5 maxdelay .05
```

**Importante:** la IP que aparece en la configuración es dinámica. Esto quiere decir que 192.168.4.18 es la IP que tiene mi ordenador en el momento de redactar este documento, pero en otro ordenador (o en este ordenador pero en otro momento), es muy probable que la IP sea otra. **Pon ahí la IP de tu ordenador.** Puedes obtener la IP de tu ordenador ejecutando en el terminal:

```shell
ip a
```

Guardamos y cerramos el editor (<kbd>CONTROL</kbd>+<kbd>O</kbd> y después <kbd>CONTROL</kbd>+<kbd>X</kbd>) y reincidamos chrony:

```shell
sudo /etc/init.d/chrony stop
sudo /etc/init.d/chrony start
```

Con todo esto, ¡ya debemos tener nuestras máquinas sincronizadas!

## Creación del mapa

Ya lo hicimos la sesión anterior, pero vamos a hacerlo otra vez con un mapa un poco más "complicado" y veremos como ahora podemos crearlo en un plis-plas.

### Conectarse a la RPi

En esta sesión prescindiremos de VNC para conectarnos a la RPi y lo haremos mediante SSH.

> **Consejo importante para futuros ingenieros:** si está la opción de conexión mediante Escritorio remoto o SSH con terminal, escoged la que más cómoda os sea, pero si los recursos (capacidad de computación, calidad de la conexión de red, etc.) son limitados, utilizad el terminal. Está allí para ayudaros. Aprendedla a utilizar y acostumbraros a ella y, incluso sin recursos limitados, la preferiréis antes que una conexión mediante Escritorio remoto. Os lo prometo.

Abrimos un terminal en nuestro PC y ejecutamos el siguiente comando:

```shell
ssh pi@192.168.4.1
```

Si es la primera vez que nos conectamos, nos saldrá un mensaje que nos pide responder *yes* o *no*. Escribimos *yes* y <kbd>ENTER</kbd>. Si no nos pregunta nada, pues no hay que hacer nada. Seguidamente nos pedirá contraseña. Escribimos `raspberry` y <kbd>ENTER</kbd> (al indicar contraseñas, estas no aparecen en el terminal por seguridad, así que aunque parece que no estáis escribiendo, sí que lo estáis haciendo).

Si todo ha ido bien, ahora tenemos un terminal donde estamos dentro de la RPi.

### Iniciar el master de ROS en el PC y lanzar RViz y gmapping

De vuelta al PC, vamos a iniciar RViz para iniciar la creación del mapa. Vamos al siguiente directorio:

```shell
cd ~/Desktop/gopigo3_rpi_slam/pc
```

> Los *workspaces* tanto del PC como de la RPi ya te se han sido descargados en su lugar para que los tengas disponibles. **No tienes que descargar nada.**

Una vez allí, compilamos el *workspaces*, hacemos que todo él sea ejecutable, y hacemos un `source` de las variables de entorno :

```shell
catkin_make
chmod -R +x ./*
source devel/setup.sh
```

Ahora sí, ejecutamos el siguiente *launch file* que, como seguro que nos acordamos, nos lanzará a la vez `roscore` si no está corriendo ya. **Importante:** debéis pasar al *launch file* el parámetro `lidar_model` en el que indicaréis el modelo de LiDAR que utilizáis (`yd` para el modelo azul, `rp` para el modelo negro).

```shell
roslaunch gopigo_slam gopigo3_slam.launch lidar_model:=yd
```

Se abrirá RViz y veréis un modelo blanco 3D de GoPiGo3, pero no veréis ni láser, ni mapa, ni nada,... ¿Razón? No hemos arrancado GPG3 y nadie está publicando distancias (LiDAR) ni odometría (motores) con los que hacer el mapa. Vamos a arrancar el GPG3.

### Iniciar el LiDAR, los motores y el seguimiento de pared

Nos vamos al terminal conectado a la RPi y nos dirigimos al directorio:

```shell
cd ~/Desktop/gopigo3_rpi_slam/rpi
```

Y, antes de arrancar nada, reflexionemos... "¿Quién es el master en mi red de ROS?" Exactamente. En nuestro caso, el master es el PC, no la RPi. Por ello, tenemos que decirle a ROS quién es el master y dónde puede encontrarlo. Para ello, se hace uso de las variables de entorno. Ejecutamos el siguiente comando:

```shell
export ROS_MASTER_URI=http://192.168.4.18:11311/
```

Como en el caso de la configuración de chrony, debemos de indicar la IP del PC. Modificar la IP del comando anterior y ejecutadlo. (Si ya lo habéis ejecutado antes de modificar la IP, simplemente volvedlo a ejecutar con la IP buena).

> En el directorio en el que os encontráis actualmente hay un fichero llamado `set_env_vars.sh`. Este contiene justo la instrucción anterior de tal modo que si hacéis:
>
> ```shell
> source set_env_vars.sh
> ```
>
> Estaréis ejecutando el comando. Es más cómodo y a la vez permite ir añadiendo comandos que necesitéis ejecutar con regularidad (como podría ser el hacer un `source devela/setup.sh`.
> Si ejecutáis este archivo, acordaros de editar la IP que contiene y poner la de vuestro PC.

Una vez indicado el master, aseguraros que el GPG3 está dentro del mapa/estancia, que tiene la pared a su derecha, compilamos el *workspace*, hacemos el `source` pertinente y lanzamos el *launch file* (indicando el modelo de LiDAR que utilizáis):

```shell
catkin_make
chmod -R +x ./*
source devel/setup.sh
roslaunch gopigo_control autonomous_navigation.launch lidar_model:=yd
```

Si hemos seguido las instrucciones, el LiDAR empezará a funcionar y el GPG3 empezará a moverse siguiendo la pared a su derecha.

### Guardar el mapa

Volvemos al PC y ahora debemos ver que se va generando un mapa en RViz de manera mucho más fluida que en la sesión anterior. Esperamos a que se complete el mapa (idealmente, esperaremos que el robot recorra todo el mapa) y una vez hecho el mapa lo guardamos. Para guardarlo, primero vamos (en un terminal nuevo) a la carpeta donde queremos guardar el mapa y allí lo guardamos. Lo hacemos con los siguientes comandos:

```shell
cd ~/Desktop/gopigo3_rpi_slam/rpi/src/gopigo_slam/maps
rosrun map_server map_saver -f mimapa
```

Una vez guardado, abrimos el archivo `.pgm` para ver que se ha guardado correctamente. Si ha sido el caso, vamos al terminal de la RPi y detenemos el robot (<kbd>CONTROL</kbd>+<kbd>C</kbd>) y luego volvemos al PC y cerramos RViz.

## Navegación autónoma punto a punto

Ya con el mapa creado, vamos a hacer que el robot vaya de una localización a otra de manera autónoma calculando la mejor ruta. Para ello, en el PC ejecutamos el siguiente comando:

```shell
roslaunch gopigo_slam gopigo3_navigation.launch lidar_model:=yd map_file:=mimapa
```

Indicad vuestro modelo de LiDAR y el nombre del mapa que habéis creado antes. Se os abrirá el RViz con el mapa cargado, pero una vez más, sin ver bien el modelo 3D del GPG3 ni las lecturas del LiDAR. Aún falta arrancarlo.

Nos vamos al terminal de RPi y ejecutamos:

```shell
roslaunch gopigo_control slave_navigation.launch lidar_model:=yd
```

El LiDAR empezará a funcionar y los motores estarán listos para recibir instrucciones del PC sobre a dónde debe de ir.

Volvemos al RViz del PC y ahora debemos de ver el modelo del GPG3 renderizado correctamente y las medidas del LiDAR junto a un mapa de costes. El mapa de costes es un mapa que muestra en azul las zonas del mapa que tienen asociado un menor coste, mientras que las zonas rojas muestran las zonas de mayor coste. El nodo navigation calcula la mejor ruta calculando la que supone un menor coste. Por eso, las paredes aparecen sombreadas en rojo indicando que pasar a través de ella supone un alto coste (lo cual evita que el robot intente traspasarlas).

Antes de empezar a navegar, hay que indicar al robot en qué localización del mapa se encuentra (seguramente os pasará que las lecturas del LiDAR no están alineadas con vuestro mapa). El robot se puede auto-localizar si lo fuéramos moviendo por el mapa (con el key_telop, por ejemplo), pero podemos indicarle directamente dónde se encuentra mediante la herramienta 2D Pose Estimation (flecha verde). Con la herramienta seleccionada, clicamos en la localización del mapa en el que se encuentra el robot y, **sin soltar el clic**, movemos el ratón hacia la dirección/orientación del robot. Una vez indicada la orientación, soltamos el clic. Si hemos acertado (y si no es el caso, repetimos esta acción hasta que lo logremos), se alineará el mapa con las lecturas del LiDAR (no hace falta una alineación milimétrica).

Ahora sí, todo listo para navegar. Seleccionamos la herramienta 2D Nav Goal (fecha roja) y, al igual que con el 2D Pose Estimation, clicamos la localización a la que queramos que vaya el GPG3 y sin soltar indicamos la orientación final.

Si lo hemos hecho todo correctamente, deberemos de ver cómo el robot va perfectamente al destino indicado. ¡Enhorabuena! Tómate tus 5 minutos para jugar un poco con el robot y su navegación. Te lo has ganado.

## Navegación autónoma en ruta

Por último, vamos a hacer que el robot vaya automáticamente a diferentes localizaciones. Es decir, que haga una ruta. Además, en cada punto de la ruta, haremos que el robot tome una fotografía.

Si no has detenido ningún terminal, estás de suerte. Tienes la mitad del trabajo hecho. Si no es el caso, vuelve a arrancar RViz con el mapa y también el LiDAR y los motores del GPG3.

Lo primero que haremos es arrancar la cámara. En un terminal de RPi nuevo (es decir, abre un nuevo terminal en tu PC y conéctate a ella mediante SSH) y ves al directorio del *workspace*:

```shell
cd ~/Desktop/gopigo3_rpi_slam/rpi
```

Indica el master (puedes utilizar el archivo `set_env_vars.sh` si lo has modificado anteriormente), haz un `source`del *workspace* y lanza el *launch file* de la cámara:

```shell
source set_env_vars.sh
source devel/setup.sh
roslaunch raspicam_node camerav2_1280x960_10fps.launch enable_raw:=true camera_frame_id:="base_scan"
```

A continuación, vamos a asegurarnos que la ruta existente es válida para nuestro mapa. Vamos al directorio (en el PC)

```shell
cd ~/Desktop/gopigo3_rpi_slam/pc/src/gopigo3_route/src
```

y abrimos el fichero `route.yml`. Veremos una serie de puntos/localizaciones a los que queremos que GPG3 vaya y tome una fotografía. El nombre de la fotografía también se indica para cada punto. Todas las fotografías se guardaran en la carpeta `photos` existente en el directorio inmediatamente superior al que nos encontramos.

Si los puntos son correctos, ya podemos ejecutar el comando:

```shell
rosrun gopigo3_route follow_the_route.py
```

Si todo funciona correctamente, veremos cómo el robot empieza a dirgirse a la primera localocalización y hace una fotografía al llegar a ella. Así con los diferentes puntos hasta finalizar la ruta indicada.

Una vez finalizada, comprueba que las fotografías se han almacenado correctamente en la carpeta `~/Desktop/gopigo3_rpi_slam/pc/src/gopigo3_route/photos`.

Con esto ya hemos logrado nuestro objetivo. ¡Juega con diferentes rutas!

## Conclusiones

En esta sesión hemos visto cómo desplegar una red ROS distribuida donde se ejecutan nodos en nuestro PC y en una Raspberry Pi. Esto tiene un impacto más que evidente en el desempeño de la red/proyecto.

Hemos realizado el mapa de manera autónoma mediante un algoritmo de seguimiento de pared para, posteriormente, poder navegar automáticamente de punto a punto del mapa utilizando la ruta de menor coste.

Finalmente, hemos programado una ruta a seguir por GPG3 de tal modo que en cada punto de la misma el robot tome una fotografía.
