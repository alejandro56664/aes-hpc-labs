# Balanceadores de carga

## Contenido

- [Introducción](#introducción)
- [Funcionamiento](#funcionamiento)
- [Preparación ambiente pruebas](#preparación-ambiente-pruebas)
- [Experimentos](#experimentos)
- [Conclusiones](#conclusiones)
- [Referencias](#referencias)

## Introducción

En este documento se pretende mostrar el funcionamiento de los balanceadores de carga a través de una aproximación teorico-practica utilizando como principal herramienta el servidor de aplicaciones NGINX.
En la sección _Funcionamiento_ se describe brevemente como funcionan los balanceadores y que tipo de algoritmos usan. Luego en la sección _Preparación ambiente pruebas_ se describe paso a paso como instalar y configurar un clúster local usando Hyper-V en Windows 10 y Alpine Linux y NGINX para realizar los experimentos. En la sección _Experimentos_ se presenta el procedimiento para realizar las pruebas de balanceo cambiando la configuración del nodo director de forma tal que se pueda probar el comportamiento de los diferentes algoritmos de balanceo y su verificación analizando el trafico generado usando la herramienta Wireshark. Finalmente en la sección _Conclusiones_ se revisa las ventajas y desventajas de cada algoritmo y sus posibles casos de uso en la practica.

## Funcionamiento

El balanceo de carga consiste en distribuir un conjunto de tareas sobre un conjunto de recursos (unidades informáticas), para hacer más eficiente su procesamiento general. Existen dos enfoques principales:

1. algoritmos estáticos, que no tienen en cuenta el estado de las diferentes máquinas
2. algoritmos dinámicos, que suelen ser más generales y más eficientes, pero requieren intercambios de información entre las diferentes unidades informáticas, a riesgo de un pérdida de eficiencia.

![esquema básico balanceo de carga](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/esquema%20basico.PNG?raw=true)

### Objetivos

- Distribuir las solicitudes de los clientes o la carga de la red de manera eficiente en varios servidores.
- Garantizar una alta disponibilidad y confiabilidad al enviar solicitudes solo a los servidores que están en línea.
- Proporcionar la flexibilidad de agregar o quitar servidores según lo requiera la demanda.

### Aproximaciones

- Round Robin: las solicitudes se distribuyen en el grupo de servidores de forma secuencial.

- Least Connections: se envía una nueva solicitud al servidor con la menor cantidad de conexiones actuales a los clientes. La capacidad informática relativa de cada servidor se tiene en cuenta para determinar cuál tiene menos conexiones.

- Least time: envía solicitudes al servidor seleccionado por una fórmula que combina
el tiempo de respuesta más rápido y la menor cantidad de conexiones activas.

- Hash: distribuye las solicitudes en función de una clave que defina, como la dirección IP del cliente o
la URL de la solicitud.

- Hash IP: la dirección IP del cliente se utiliza para determinar qué servidor recibe la solicitud.
Aleatorio con dos opciones: elige dos servidores al azar y envía la solicitud al
uno que se selecciona aplicando el algoritmo Least Connections.

TODO: complementar funcionamiento

## Preparación ambiente pruebas

Para realizar pruebas con el balanceador de carga en un ambiente windows 10 es necesario tener activado el modo Hyper-V. Para habilitar el modo puede consultar: <https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v> .

El sistema operativo de las maquinas virtuales se va a utilizar Alpine Linux por ser una distribución liviana. Primero debe descargar el iso en el siguiente enlace: <https://www.alpinelinux.org/downloads/>, en este laboratorio se uso la versión estandar x86_64.

### Creación y configuración maquinas virtuales nodos del clúster

En este punto se va a explicar como crear y configurar las maquinas virtuales.
Ingrese al _Administrador de Hyper-V_

Pasos para crear una maquina virtual

![crea vm step1](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step1.PNG?raw=true)

![crea vm step2](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step2.PNG?raw=true)

![crea vm step3](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step3.PNG?raw=true)

![crea vm step4](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step4.PNG?raw=true)

![crea vm step5](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step5.PNG?raw=true)

![crea vm step6](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step6.PNG?raw=true)

![crea vm step7](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step7.PNG?raw=true)

![crea vm step8](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/crear%20vm%20step8.PNG?raw=true)

Pasos para configurar la maquina virtual:

![configuar step1](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/configurar%20vm%20step1.PNG?raw=true)

![configuar step2](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/configurar%20vm%20step2.PNG?raw=true)

Debe iniciar sesión con el usuario por defecto: _root_ y luego ejecutar el asistente de configuración de Alpine:

```sh
setup-alpine
```

Esto iniciara un asistente, se recomiendan las siguientes configuraciones, las demas las puede dejar por defecto:

- idioma teclado: _es_ luego _es-winkeys_
- host: vm0x _donde x: 0, 1, n nodo_
- pass: lab@vm0x _donde x: 0, 1, n nodo_

![configuar step3](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/configurar%20vm%20step3.PNG?raw=true)

Instalación nginx

```sh
apk update

apk add nginx
```

![configuar step4](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/configurar%20vm%20step4.PNG?raw=true)

Para configurar la página html que se va a presentar:

```sh

cd ..

mkdir /home/www

cd /home/www

vi index.html
```

Recuerde:

- Para insertar texto presione la tecla _i_
- Cuando termine de editar presione _[esc]_ y luego escriba _:wq_
- Si desea salir sin guargar, presione _[esc]_ y luego escriba _:q!_

Este es el texto que debe agregar a index.html (recuerde x: 0, 1, n nodo):

```html

<!DOCTYPE html>
<html>
<body>

<h1>Hola desde VM0x</h1>
<p>Integrantes: </p>
<ul>
  <li>Integrante 1</li>
  <li>Integrante 2</li>
  <li>...</li>
</ul>
</body>
</html>
```

Ahora debe configurar el nginx para que sirva contenido estatico, para ello debe editar la configuración del servidor:

```sh
vi /etc/nginx/conf.d/default.conf
```

Esta es la configuración que debe quedar:

```js
server {
      listen 80 default_server;
      listen [::]:80 default_server;
      root /home/www;

      # log files
      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/error.log;

      location / {
          }
      }
```

Fuente: <https://docs.nginx.com/nginx/admin-guide/web-server/serving-static-content/>

Guarde: [esc] :wq

Ahora es necesario configurar el servicio de nginx para que inicie cada vez que inicie el sistema operativo, puede obtener mayor información sobre el comando (rc-update)[https://manpages.debian.org/testing/openrc/rc-update.8.en.html]

```sh
rc-update add nginx default
```

Esto permite iniciar, detener y reiniciar el servicio desde la linea de comandos, se utiliza el comando rc-service, el cuál permite localizar un servicio y enviarle comandos. puede obtener mayor información sobre el comando (rc-update)[https://manpages.debian.org/testing/openrc/rc-update.8.en.html] :

```sh
rc-service nginx start

rc-service nginx stop

rc-service nginx restart
```

Si el servicio levanto correctamente, puede ingresar al navegador con la ip privada de la maquina virtual recien creada y se podrá visualizar el html creado inicialmente.

Para ver la ip de la maquina virtual puede usar el siguiente comando:

```sh
ip a
```

![configuar step5](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/configurar%20vm%20step5.PNG?raw=true)

*NOTA:*
Estos pasos se repiten con todas las maquinas nodos que se desean crear para los nodos
Para la creación del nodo balanceador se recomienda nombrarlo 'vm-director01'

La configuración inicial del nginx del nodo director es diferente y se detalla a continuación:

Primero debe editar el servidor por defecto:

```sh
vi /etc/nginx/conf.d/default.conf
```

```js
server {
      listen 80 default_server;
      listen [::]:80 default_server;

      # log files
      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/error.log;
      location / {
      proxy_pass http://cluster;
          }
      }
```

Guarde: [esc] :wq

Note que aquí el campo _location_ va a servir de proxy para un servicio http. El nombre 'cluster' es completamento arbitrario, pero debe coincider con el _upstream_ configurado en el archivo de configuración principal del nginx.

```sh
vi /etc/nginx/nginx.conf
```

```js
http {
    upstream cluster {
    server 192.168.x.x:80 weight=2;
    server 192.168.x.x:80;
  }
  # ... demas configuraciones
}
```

Fuente: <https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/>

Guarde: [esc] :wq

Note que aquí se configuran las ip privadas de los servidores a los cuales se van a redirigir las peticiones.
Así que *recuerde poner las ips de las maquinas vm01 y vm02*

Una vez modificados los archivos de configuración adecuados, termine la configuración del nginx como en los demas nodos:

```sh
rc-update add nginx default

rc-service nginx start

ip a
```

Entre al navegador con la ip indicada y podrá ver el html (recargue varias veces la página y verá que algunas veces atiende la vm01 y otras la vm02)

En este punto ya tiene configurado un ambiente básico para realizar pruebas de balanceo de carga en una maquina local con Windows 10.

## Experimentos

Para la ejecución de los experimentos propuestos se requiere tener instalado el software *tcpdump*, el cuál puede descargar e instalarse en este caso en la maquina director.
Para mas información visite el sitio oficial: https://www.tcpdump.org/manpages/tcpdump.1.html

```sh
apk add tcpdump
```

Una vez instalado y configurado el clúster de manera local, se procede a realizar las siguientes pruebas.

Para verificar el acceso a cada servidor se vigilará los logs de cada servidor, para ello puede utilizar el comando:

```sh
tail -n 1000 /var/log/nginx/access.log -f
```
El cuál permite vigilar el documento a medida que crece (parametro -f) y limita a 1000 el número de lineas mostradas (parametro -n)

### Balanceo tipo Round-robin con pesos

Procedimiento: 

1. Se configura el ngnix del director de la siguiente manera:

```js
http {
    upstream cluster {
    server 192.168.x.x:80 weight=1;
    server 192.168.x.x:80 weight=1;
    server 192.168.x.x:80 weight=1;
  }
  # ... demas configuraciones
}
```

2. Ejecute el siguiente comando:

En la siguiente imagen obtenida del _tcpdump_ se puede observar las diferentes solicitudes en el tiempo:

```sh
tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

Este comando permite imprimir todos los paquetes HTTP IPv4 hacia y desde el puerto 80, es decir, imprimir solo paquetes que contienen datos, no, por ejemplo, paquetes SYN y FIN y paquetes solo ACK.

3. Ingrese a la ip de la vm director en un navegador y recargue varias veces la página.

Resultados:

Podrá ver como el _tcpdump_ muestra las peticiones recibidas y las redirecciones a cada uno de los servidores del clúster. En color rojo esta el trafico relacionado con el servidor vm02, en verde el servidor vm03, en azul el servidor vm01:

![experimento1 step1](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/experimento1%20step1.PNG?raw=true)

En el siguiente video puede observar el comportamiento dinámico del balanceo.

[![IMAGE ALT TEXT](http://img.youtube.com/vi/YOUTUBE_VIDEO_ID_HERE/0.jpg)](http://www.youtube.com/watch?v=YOUTUBE_VIDEO_ID_HERE "Video Title")

2. Se bajan cada uno de los servidores en orden y se verifica el comportamiento del balanceador.

Podrá ver como el _tcpdump_ muestra las peticiones recibidas y las redirecciones a cada uno de los servidores del clúster (menos al servidor apagado, en este caso vm01)

![experimento1 step2](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/experimento1%20step2.PNG?raw=true)

Ahora se apaga el servidor vm02 y se obtiene el siguiente patrón:

![experimento1 step3](https://github.com/alejandro56664/aes-hpc-labs/blob/main/load-balancing/doc/assets/experimento1%20step3.PNG?raw=true)

En el siguiente video puede ver el comportamiento dinamico del balanceador:

[![IMAGE ALT TEXT](http://img.youtube.com/vi/Zu-hVSY3Svo/0.jpg)](https://youtu.be/Zu-hVSY3Svo "Pruebas laboratorio balanceador de carga")


## Conclusiones

TODO presentar un resumen de las ventajas y desventajas de cada uno de los algoritmos.

## Referencias

- <https://nginx.org/en/docs/http/ngx_http_upstream_module.html>
- <https://www.nginx.com/resources/glossary/load-balancing/>
- <https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/>
- <https://wiki.alpinelinux.org/wiki/Alpine_Install:_from_a_iso_to_a_virtualbox_machine_with_external_disc>
- <https://www.cyberciti.biz/faq/how-to-install-nginx-web-server-on-alpine-linux/>
- <https://nginx.org/en/docs/http/ngx_http_upstream_module.html>
