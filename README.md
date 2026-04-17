# Del filtrado DNS a una nube privada accesible mediante Tailscale

## Indice
- [Introduccion](#introduccion)
- [Fundamento teorico](#fundamento-teorico)
  - [1. Tailscale - la muerte del modelo Hub-and-Spoke y el ascenso de la comunicacion peer-to-peer con WireGuard](#1-tailscale---la-muerte-del-modelo-hub-and-spoke-y-el-ascenso-de-la-comunicacion-peer-to-peer-con-wireguard)
  - [2. Tailscale y su magia para realizar conexiones a traves de NATs y firewalls hogarenos](#2-tailscale-y-su-magia-para-realizar-conexiones-a-traves-de-nats-y-firewalls-hogarenos)
- [Fase de implementacion](#fase-de-implementacion)
  - [1. Blindaje del DNS (Pi-hole y Unbound DoT)](#1-blindaje-del-dns-pi-hole-y-unbound-dot)
  - [2. Implementacion de Nextcloud AIO - conflicto de puertos y externalizacion del perimetro mediante proxy inverso](#2-implementacion-de-nextcloud-aio---conflicto-de-puertos-y-externalizacion-del-perimetro-mediante-proxy-inverso)
  - [3. Implementacion de Immich - acceso seguro, proxy inverso, cifrado TLS y eleccion de proveedor DNS](#3-implementacion-de-immich---acceso-seguro-proxy-inverso-cifrado-tls-y-eleccion-de-proveedor-dns)
- [4. Gestion del hardware - Raspberry Pi al limite](#4-gestion-del-hardware---raspberry-pi-al-limite)
  - [4.1. La ingesta de datos - como importar cientos de gigas a Immich](#41-la-ingesta-de-datos---como-importar-cientos-de-gigas-a-immich)
  - [4.2. Limitar los contenedores para no colapsar el hardware](#42-limitar-los-contenedores-para-no-colapsar-el-hardware)

## Introduccion
Antes de la evolución de este proyecto, la Raspberry Pi 5 equipada con 8 GB de memoria RAM desempeñaba una única dentro del entorno doméstico: actuar como servidor DNS mediante la combinación de Pi-hole y Unbound. Esta arquitectura permitía resolver consultas DNS de manera recursiva directamente contra los servidores raíz de Internet, eliminando intermediarios y proporcionando un mayor control sobre la privacidad del tráfico, además de incorporar filtrado de dominios a nivel de red.
La dirección que toma este servidor a partir de ahora es adoptar la filosofía “Zero Trust”, principio arquitectónico que parte de una premisa fundamental: ninguna red, ni siquiera la local, debe considerarse intrínsecamente segura. Este cambio conceptual implica abandonar los modelos tradicionales basados en exposición de servicios mediante apertura de puertos y sustituirlos por mecanismos de autenticación fuerte, cifrado extremo a extremo y control de acceso basado en identidad.
Bajo este marco, el objetivo del proyecto es diseñar e implementar una nube privada plenamente controlada por el usuario, capaz de ofrecer servicios accesibles tanto desde la red local como desde fuera de ella, sin necesidad de exponer puertos ni comprometer la superficie de ataque.

## Fundamento teorico
### 1. Tailscale - la muerte del modelo Hub-and-Spoke y el ascenso de la comunicacion peer-to-peer con WireGuard
Históricamente las VPN adoptaban topologías hub-and-spoke: todos los clientes remotos se conectaban a un servidor central, y éste reenviaba el tráfico entre ellos. Esto causaba cuellos de botella y latencias innecesarias, ya que los datos viajaban de extremo a extremo pasando por el hub. 
<img width="639" height="406" alt="image" src="https://github.com/user-attachments/assets/c4ee7eb1-f39a-4975-a33e-07f7a60a67ae" />
En contraste, Tailscale introduce una arquitectura radicalmente distinta basada en redes mesh o de malla, donde cada nodo es capaz de comunicarse directamente con cualquier otro nodo autorizado. Este modelo elimina la necesidad de encaminamiento indirecto, permitiendo que los datos fluyan por la ruta más corta posible entre origen y destino. El resultado es una reducción significativa de la latencia, una mejora en el rendimiento global y una distribución más eficiente de la carga a medida que crece el número de dispositivos.
<img width="672" height="454" alt="image" src="https://github.com/user-attachments/assets/fb494dc7-a743-49b5-b0d4-e6b33605aeb8" />
Desde el punto de vista arquitectónico, Tailscale separa claramente dos planos funcionales. Por un lado, el plano de control, gestionado por un servidor de coordinación (conocido como coordination server), cuya función es facilitar el descubrimiento entre nodos mediante el intercambio de claves públicas y metadatos de conectividad. Este servidor no participa en el flujo de datos ni tiene acceso al contenido de las comunicaciones.
Por otro lado, el plano de datos se construye sobre WireGuard, un protocolo de VPN moderno caracterizado por su simplicidad, eficiencia y seguridad criptográfica. WireGuard utiliza criptografía de última generación (Curve25519, ChaCha20, Poly1305) para establecer túneles cifrados punto a punto, minimizando la sobrecarga y maximizando el rendimiento. A diferencia de soluciones tradicionales, su diseño minimalista reduce la superficie de ataque y facilita auditorías de seguridad.
En este contexto, cada nodo mantiene su propia clave privada, que nunca abandona el dispositivo, mientras que las claves públicas se distribuyen a través del plano de control. La autenticación de los dispositivos se realiza mediante mecanismos modernos como OAuth, lo que permite integrar identidades externas sin necesidad de gestionar credenciales manualmente. El resultado es una red privada distribuida donde la confidencialidad y la integridad de los datos están garantizadas extremo a extremo.

### 2. Tailscale y su magia para realizar conexiones a traves de NATs y firewalls hogarenos
Uno de los desafíos fundamentales en la construcción de redes privadas distribuidas es la conectividad entre nodos situados detrás de dispositivos de traducción de direcciones de red (NAT) y firewalls con estado (stateful firewalls). En escenarios domésticos y móviles, es habitual que los dispositivos carezcan de direcciones IP públicas accesibles, especialmente cuando operan bajo esquemas de CGNAT (Carrier-Grade NAT).
En estos entornos, los firewalls aplican una regla básica: únicamente se permite la entrada de paquetes que correspondan a una conexión previamente iniciada desde el interior de la red. Esto impide, en principio, que dos dispositivos situados tras NAT independientes puedan establecer comunicación directa.
<img width="886" height="346" alt="image" src="https://github.com/user-attachments/assets/d6c7b189-11ce-4d0c-9827-dd9d6e89be16" />
Tailscale resuelve este problema mediante técnicas avanzadas de NAT traversal, apoyándose en protocolos como STUN (Session Traversal Utilities for NAT). A través de este mecanismo, cada nodo es capaz de determinar su dirección IP y puerto externos tal como son percibidos desde Internet. Esta información se comparte a través del plano de control, permitiendo a los nodos intentar establecer conexiones directas.
El proceso clave es el denominado UDP hole punching, técnica que consiste en que ambos extremos inicien simultáneamente el envío de paquetes hacia las direcciones y puertos del otro nodo. Debido al comportamiento de los NAT, esta acción crea entradas temporales en las tablas de estado de los firewalls, permitiendo que los paquetes entrantes sean aceptados.
<img width="886" height="346" alt="image" src="https://github.com/user-attachments/assets/6cbaaba9-de0b-466f-a0ed-6b7691b45604" />
En situaciones más complejas, como NAT simétricos o entornos altamente restrictivos, el establecimiento de conexiones directas puede no ser viable. En estos casos, Tailscale recurre a mecanismos de retransmisión mediante nodos intermedios, garantizando la conectividad a costa de un ligero incremento en la latencia.
Este enfoque híbrido, que prioriza siempre la conexión directa pero incorpora mecanismos de respaldo, permite mantener un equilibrio óptimo entre rendimiento, fiabilidad y compatibilidad con entornos de red adversos.
Para más información sobre el funcionamiento de Tailscale, que en este documento se ha expuesto de forma muy resumida, recomiendo visitar:
-	https://tailscale.com/blog/how-tailscale-works 
-	https://tailscale.com/blog/how-nat-traversal-works 

## Fase de implementacion
### 1. Blindaje del DNS (Pi-hole y Unbound DoT)
El primer paso fue reforzar la privacidad desde la raíz: el DNS. Sin cifrado, el operador o cualquier observador en la ruta conocería cada dominio que resolvemos. Por eso instalamos Pi-hole como servidor DNS local de la LAN, junto con Unbound configurado como recursivo que realiza consultas DNS-over-TLS (DoT) a Quad9. Quad9 es un servicio DNS público gratuito centrado en seguridad y privacidad.
En la configuración de Unbound se añade una forward-zone al dominio raíz con forward-tls-upstream: yes hacia 9.9.9.9@853#dns.quad9.net (y secundario 149.112.112.112@853#dns.quad9.net). Con esto todo DNS saliente desde la Pi va cifrado por TLS al resolver de Quad9, impidiendo que el ISP vea nuestras consultas:</br>
**/etc/unbound/unbound.conf.d/pi-hole.conf**
```
forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 9.9.9.9@853#dns.quad9.net
    forward-addr: 149.112.112.112@853#dns.quad9.net
```
Habilitar DoT junto con DNSSEC introdujo un reto: las respuestas DNS validadas por DNSSEC son mucho más grandes de 512 bytes, lo que puede fragmentarlas y provocar timeouts. La solución técnica fue aumentar el buffer DNS: siguiendo la documentación oficial de Unbound, fijamos edns-buffer-size: 1232 para evitar fragmentación. Además, en Pi-hole activamos EDNS0=true en su configuración (/etc/pihole/pihole.toml), lo que permite manejar estas consultas ampliadas sin caídas.
Por último, aislamos al nodo principal de las DNS globales de Tailscale. Tailscale MagicDNS asigna nombres internos y puede aprovisionar DNS (por ejemplo mi-node.tailnet), lo que se usará más adelante para Nextcloud, pero no queremos que nuestra Pi consulte a sí misma a través del túnel VPN y entre en bucles. Entonces ejecutamos:</br>

**sudo tailscale up --accept-dns=false --advertise-routes=192.168.1.0/24'**</br>

El flag --accept-dns=false ordena al nodo ignorar cualquier DNS del tailnet, usando exclusivamente su resolver local. Y --advertise-routes=192.168.1.0/24 anuncia la ruta de la LAN al tailnet. Así, los dispositivos remotos sí ven la Pi-hole como DNS (gracias al anuncio de ruta), pero la Raspberry mantiene su cordura resolviendo internamente con Quad9 encripta directamente. En síntesis, el DNS quedó “blindado”: cifrado en el tránsito (DoT+DNSSEC) y deslindado del MagicDNS de Tailscale, garantizando privacidad total.

### 2. Implementacion de Nextcloud AIO - conflicto de puertos y externalizacion del perimetro mediante proxy inverso
En esta fase se introdujo uno de los componentes más relevantes de la infraestructura: Nextcloud en su variante Nextcloud All-in-One (AIO). Esta solución, diseñada para simplificar despliegues complejos, incorpora un contenedor maestro que orquesta múltiples servicios internos (base de datos, almacenamiento, servidor web, etc.). Sin embargo, su diseño presenta una limitación crítica desde el punto de vista arquitectónico: asume el control absoluto de los puertos 80 y 443 del host.
Para materializar el desacoplamiento de ambos puertos, se identificó que el contenedor maestro encapsula un servidor Apache HTTP Server que, por diseño, escucha en el puerto 11000 dentro del entorno de los contenedores. Entonces se modificó la configuración del contenedor mediante variables de entorno para forzar el uso de ese puerto interno y de esta forma hacer que Nextcloud pase a ser accesible únicamente desde el backend en lugar de estar “de cara al usuario”, lo que además introduce una capa de seguridad intermedia.</br>
```
-e APACHE_PORT=11000 \
-e APACHE_IP_BINDING=0.0.0.0 \
-e DOCKER_API_VERSION=1.41 (añadimos también esta variable ya que el contenedor maestro de Nextcloud utiliza esa versión específica)
```

**El papel de Tailscale en la gestión de identidad y el cifrado**
A diferencia de un despliegue tradicional basado en dominios públicos y certificados obtenidos mediante validación HTTP, en este caso la conectividad se realiza exclusivamente a través de la red privada mesh.
Tailscale proporciona a cada nodo un nombre de dominio interno resoluble mediante MagicDNS, lo que permite acceder a los servicios utilizando identificadores consistentes sin depender de DNS públicos. Por otro, permite la emisión de certificados TLS válidos asociados a dichos nombres mediante su propio mecanismo de provisión de certificados.
Esto implica que Nextcloud, que requiere utilizar HTTPS, puede operar bajo este protocolo sin necesidad de exponer ningún endpoint al exterior, ya que el tráfico nunca abandona la red cifrada de Tailscale. Desde el punto de vista criptográfico, la autenticidad y confidencialidad están garantizadas tanto a nivel de transporte (WireGuard) como a nivel de aplicación (TLS).

**Nginx Proxy Manager como punto de entrada lógico**
La decisión de incorporar un proxy inverso responde a varias necesidades estructurales:
En primer lugar, permite centralizar el enrutamiento de tráfico HTTP/HTTPS en un único punto, desacoplando completamente los servicios backend de la capa de acceso. En segundo lugar, facilita la gestión de certificados, dominios y reglas de redirección sin modificar la configuración interna de cada aplicación. Finalmente, introduce una capa adicional de control que puede evolucionar independientemente del resto del sistema.
En este contexto, NPM actúa como un proxy interno dentro de la red Tailscale.

**Flujo completo de acceso tras la reconfiguración**
<img width="1771" height="797" alt="nextcloud1" src="https://github.com/user-attachments/assets/20ff5fd7-19d8-4038-8216-bd101076f3bb" />

-	Tailscale gestiona la identidad, autenticación y cifrado de red
-	NPM gestiona el enrutamiento y la terminación TLS
-	Nextcloud actúa exclusivamente como servicio de aplicación

### 3. Implementacion de Immich - acceso seguro, proxy inverso, cifrado TLS y eleccion de proveedor DNS
A pesar de que Nextcloud es una solución muy eficaz para el almacenamiento de archivos, se decidió implementar también Immich como repositorio principal de fotos y vídeos.
A diferencia de Nextcloud, Immich no impone restricciones estrictas sobre el uso de HTTPS ni exige certificados válidos para su funcionamiento. Desde un punto de vista puramente funcional, podría operar perfectamente sobre HTTP dentro de una red local. Sin embargo, esta aproximación resulta insuficiente en el contexto de este proyecto ya que, el acceso a Immich no se limitará al entorno local, sino que se realizará de forma habitual desde redes externas (datos móviles, redes Wi-Fi públicas, etc.) a través de Tailscale. Aunque Tailscale ya proporciona cifrado extremo a extremo a nivel de red mediante WireGuard, confiar exclusivamente en esta capa implicaría depender de un único mecanismo de seguridad.

**Limitaciones del enfoque nativo de Tailscale para certificados**
En este punto, una solución aparentemente natural sería utilizar los certificados generados por Tailscale mediante su funcionalidad integrada (tailscale cert), como se hizo en el caso de Nextcloud. Sin embargo, los certificados de Tailscale están estrictamente ligados al nombre del nodo, lo que dificulta exponer múltiples servicios bajo distintos subdominios (por ejemplo, immich.midominio, nextcloud.midominio, etc.). En consecuencia, se descarta este enfoque en favor de una solución más modular y escalable.

**Introducción del proxy inverso como punto de entrada unificado**
Para resolver la limitación presentada por Tailscale, se consolida el uso de Nginx Proxy Manager como punto de entrada único a todos los servicios. En este modelo, NPM se convierte en el componente encargado de:
-	Terminar las conexiones TLS (gestión de certificados)
-	Resolver nombres de dominio
-	Redirigir el tráfico a los servicios internos correspondientes
Esto permite desacoplar completamente la capa de acceso de los servicios backend, que permanecen aislados dentro de la red Docker, al igual que se hizo con Nextcloud.
En el caso concreto de Immich, el servicio pasa a exponerse únicamente a través de NPM, dejando de estar accesible mediante su puerto interno (2283) desde el exterior del entorno de contenedores.

**El desafío del certificado sin apertura de puertos**
El siguiente obstáculo es la obtención de certificados TLS válidos. En un despliegue tradicional, esto se resolvería mediante el desafío HTTP-01 de Let's Encrypt, que requiere exponer el puerto 80 para validar la propiedad del dominio.
Sin embargo, esta opción contradice directamente el principio fundamental del proyecto: no abrir puertos en el router bajo ninguna circunstancia.
Para resolver esta limitación, se opta por el mecanismo de validación DNS-01, que permite demostrar la propiedad de un dominio mediante la creación de registros DNS específicos, sin necesidad de exponer ningún servicio.

**Elección de DuckDNS como proveedor DNS**
DuckDNS se selecciona por varias razones:
-	Ofrece una API extremadamente sencilla que permite automatizar la creación y modificación de registros DNS mediante un token de autenticación.
-	Elimina la necesidad de disponer de un dominio propio de pago.
-	Finalmente, su integración con Nginx Proxy Manager está bien soportada, lo que simplifica la configuración.
Únicamente hay que registrar un dominio .duckdns.org desde https://www.duckdns.org/, que actuará como que actuará como identificador público del sistema, ya que establecemos que redirija a la IP de Tailscale asociada al servidor, por lo que solo podrá acceder alguien conectado y con acceso al nodo de Tailscale.

**Obtención y gestión del certificado en NPM**
Nginx Proxy Manager se configura para solicitar certificados a Let's Encrypt utilizando el desafío DNS-01. Durante este proceso, NPM utiliza el token de DuckDNS para crear automáticamente los registros TXT necesarios en la zona DNS del dominio.
Let's Encrypt valida estos registros y emite el certificado correspondiente, sin necesidad de establecer ninguna conexión entrante hacia la infraestructura.
Una vez emitido, el certificado queda almacenado en NPM, que lo utilizará para todas las conexiones entrantes hacia los servicios configurados.

**Flujo completo de conexión a Immich**
<img width="1921" height="820" alt="immich" src="https://github.com/user-attachments/assets/0b683a33-bf3f-4c04-9fdd-4cb0e90b9f2d" />

-	Tailscale gestiona la identidad de los nodos, la autenticación y el cifrado de red mediante túneles basados en WireGuard, permitiendo que el acceso a Immich se realice exclusivamente dentro de una red privada sin exposición pública.
-	Pi-hole se encarga de la resolución DNS interna (override), asociando el dominio configurado a la IP de Tailscale del servidor, evitando así dependencias del DNS público en tiempo de ejecución.
-	Nginx Proxy Manager actúa como punto único de entrada, gestionando el enrutamiento hacia el servicio correspondiente y la terminación TLS mediante certificados emitidos por Let's Encrypt y validados a través del desafío DNS-01 con DuckDNS.
-	Immich opera como servicio de aplicación backend, accesible únicamente dentro de la red Docker a través de su puerto interno, sin exposición directa ni a la red local ni a Internet.

## 4. Gestion del hardware - Raspberry Pi al limite
### 4.1. La ingesta de datos - como importar cientos de gigas a Immich
De cara a la importación de las imágenes y vídeos presentes en un disco duro externo que se dejará como respaldo, se utilizará la CLI oficial de Immich en Docker, ya que es la herramienta mejor optimizada para realizar este proceso.
Tras montar el disco externo, se ejecuta el comando:
```
sudo docker run -it --rm \
  -v "/media/ruta_al_punto_de_montaje EXT:/import" \
  -e IMMICH_INSTANCE_URL=http://IP_INTERNA:2283 \
  -e IMMICH_API_KEY=XXX \
  ghcr.io/immich-app/immich-cli upload \
  --recursive --album /import
```
Esto recorrió recursivamente todas las carpetas en /import. La opción --recursive incluye subdirectorios, y --album hace que el CLI cree automáticamente álbumes basados en el nombre de cada carpeta.
Se empleó tmux para ejecutar este comando para que la transferencia siguiera en marcha aun si la sesión SSH se desconectaba:
1.	tmux new -s subida
2.	tmux attach -t subida
3.	/ejecutar el comando anterior de Docker/
4.	Para salir de tmux: CTRL+B, seguido de D

### 4.2. Limitar los contenedores para no colapsar el hardware
Durante la realización no se tuvieron en cuenta los límites del hardware de la Rapberry Pi y, cuando a penas había terminado el 7% de la subida, el sistema colapsó. Al realizar una inspección de los procesos con htop se observaba una carga del procesador del 350%.
Los procesos culpables son los contenedores de immich-server y el de Machine Learning, que escanea las fotos mientras se suben al servidor. Por este motivo, es importante desplegar los contenedores con límites de hardware establecidos en el docker-compose.yml:</br>
-	En el contenedor principal:
```
deploy:
  resources:
    limits:
      cpus: '1'
      memory: 2048M'
```

-	En el contenedor de Machine Learning:
```
deploy:
  resources:
    limits:
      cpus: '0.25'
      memory: 512M
```
