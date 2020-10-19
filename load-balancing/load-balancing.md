# Balanceadores de carga

## Contenido

- [Introducción](#introducción)
- [Funcionamiento](#funcionamiento)
- [Preparación ambiente pruebas](#preparación-ambiente-pruebas)
- [Experimentos](#experimentos)
- [Conclusiones](#conclusiones)
- [Referencias](#referencias)

## Introducción

TODO: agregar introducción

## Funcionamiento

TODO: agregar funcionamiento

## Preparación ambiente pruebas

Para realizar pruebas con el balanceador de carga en un ambiente windows 10 es necesario tener activado el modo Hyper-V. Para habilitar el modo puede consultar: https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v .

El sistema operativo de las maquinas virtuales se va a utilizar Alpine Linux por ser una distribución liviana. Primero debe descargar el iso en el siguiente enlace: https://www.alpinelinux.org/downloads/, en este laboratorio se uso la versión estandar x86_64.

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

Fuente: https://docs.nginx.com/nginx/admin-guide/web-server/serving-static-content/

Guarde: [esc] :wq

Ahora es necesario configurar el servicio de nginx para que inicie cada vez que inicie el sistema operativo

```sh
rc-update add nginx default
```

Esto permite iniciar, detener y reiniciar el servicio desde la linea de comandos:

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
    upstream myproject {
    server 192.168.x.x:80 weight=2;
    server 192.168.x.x:80;
  }
  # ... demas configuraciones
}
```

Fuente: https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/

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

En este punto ya tiene configurado un ambiente básico para realizar pruebas de balanceo de carga en un maquina local con Windows 10.

## Experimentos

## Conclusiones

## Referencias

- https://www.nginx.com/resources/wiki/start/topics/examples/loadbalanceexample/
- https://wiki.alpinelinux.org/wiki/Alpine_Install:_from_a_iso_to_a_virtualbox_machine_with_external_disc
- https://www.cyberciti.biz/faq/how-to-install-nginx-web-server-on-alpine-linux/
