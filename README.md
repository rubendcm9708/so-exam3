### Examen 3
**Universidad ICESI**  
**Curso:** Sistemas Operativos  
**Docente:** Daniel Barragán C.  
**Tema:** Descubrimiento de servicios, Microservicios  
**Correo:** daniel.barragan at correo.icesi.edu.co

**Nombre:** Rubén Darío Ceballos Muriel  
**Código:** A00054636  
**Correo:** rubendcm9708@gmail.com  
**Git URL:** https://github.com/rubendcm9708/so-exam3/tree/A00054636/so-exam3  

### Objetivos
* Implementar servicios web que puedan ser consumidos por usuarios o aplicaciones
* Conocer y emplear tecnologías de descubrimiento de servicio

### Prerrequisitos
* Virtualbox o WMWare
* Máquina virtual con sistema operativo CentOS7
* Framework consul, zookeper o etcd

### Descripción
El tercer parcial del curso sistemas operativos trata sobre la creación de servicios web y el uso de tecnologías para el descubrimiento de servicio

![][1]
**Figura 1.** Despliegue básico de microservicios

### Actividades
1. Incluir nombre, código (5%)
2. Ortografía y redacción cuando sea necesario (5%)
3. Despliegue un esquema como el mostrado en la **figura 1**. Empleen un servicio web de su preferencia (puede usar alguno de los ejemplos de clase). No es necesario incluir los componentes para monitoreo (Elasticsearch, Kibana, Logstash) (30%)
4. Adicione un microservicio igual al ya desplegado. Muestre a través de evidencias como las peticiones realizadas al balanceador son dirigidas a la replica del microservicio (30%)
5. Describa los cambios o adiciones necesarias en el diagrama de la **figura 1** para adicionar un microservicio diferente al ya desplegado en el ambiente, tenga en cuenta los siguientes conceptos en su descripción: API Gateway, paradigma reactivo, load balancer, protocolo publicador/suscriptor (interconexión de microservicios) (20%)
6. El informe debe ser entregado en formato pdf a través del moodle y el informe en formato README.md debe ser subido a un repositorio de github. El repositorio de github debe ser un fork de https://github.com/ICESI-Training/so-exam3 y para la entrega deberá hacer un Pull Request (PR) respetando la estructura definida. El código fuente y la url de github deben incluirse en el informe (10%)  

### Desarrollo  
3. Para poder desplegar un esquema como el mostrado en la **figura 1**, se necesitaron:  
  -1 Consul Server funcionando como Discovery Service  
  -1 Load Balance  
  -4 Consul Clients, cada uno con un microservicio igual (Despliegue página web), pero con contenido diferente en su cuerpo  
 
**PASO 1:** Empezando mostrando la implementación de los clientes. Primero que nada se implementa el código del microservicio en python, creamos un archivo con extensión .py como el siguiente:
 
```
from flask import flask
app = Flask(_name_)

@app.route("/health")
def health():
  return "Mensaje de estado"
  
@app.route("/")
def microservice():
  return "Mensaje que quiero mostrar en la página"

if__name__ == "__main__":
  app.run(host="0.0.0.0",port=8080,debug="True")
```
**PASO 2:** Después de tener el código del microservicio, pasamos a crear un ambiente de trabajo, para ello necesitamos instalar las siguientes librerias y dependencias. Este paso es necesario en los **clientes**. Con esto nos cersioramos que no se instalen cosas innecesarias en otros ambientes dentro de la misma máquina.

```
# yum install -y wget
# wget https://bootstrap.pypa.io/get-pip.py -P /tmp
# python /tmp/get-pip.py
# pip install virtualenv
# su microservices
$ pip install --user virtualenvwrapper
``` 

Editamos el bashrc para que exporte automáticamente las librerias de *virtualenv* al iniciar la máquina
``` 
$ vi ~/.bashrc
``` 
Y pegamos las siguientes líneas dentro
``` 
export WORKON_HOME=~/.virtualenvs
source /home/microservices/.local/bin/virtualenvwrapper.sh
``` 
Guardamos cambios con
``` 
$ source ~/.bashrc
``` 

Después de estos pasos, podremos crear nuestro ambiente de trabajo. Y a continuación de entrar en el virtual environment, instalamos **Flask** que nos va a permitir desplegar nuestra página web.  

``` 
mkvirtualenv parcial3
workon parcial3
pip install Flask
``` 
**PASO 3:** Con nuestro ambiente y Flask instalado, podemos proceder a instalar Consul, que es la herramienta que nos permitirá unificar las 5 máquinas y asignar los distintos roles dentro de la topología.

Instalamos las dependencias necesarias:
``` 
# yum install -y wget unzip
# wget https://releases.hashicorp.com/consul/1.0.0/consul_1.0.0_linux_amd64.zip -P /tmp
# unzip /tmp/consul_1.0.0_linux_amd64.zip -d /tmp
# mv /tmp/consul /usr/bin
# mkdir /etc/consul.d
# mkdir -p /etc/consul/data
```

Abrimos los puertos necesarios para la comunicación:
```
# firewall-cmd --zone=public --add-port=8301/tcp --permanent
# firewall-cmd --zone=public --add-port=8300/tcp --permanent
# firewall-cmd --zone=public --add-port=8500/tcp --permanent
# firewall-cmd --reload
```
**PASO 4:** Con los puertos abiertos, ya podremos hacer la conexión al servidor. Primero creamos una sesión de *screen*, ya que en la actual, solo podremos ver los mensajes de *health* que nos informan el estado de las consultas. Y segundo, ejectuamos el microservicio

```
python microservicio_a
```
Y creamos un archivo de configuración para el microservicio
```
echo '{"service": {"name": "microservice-a", "tags": ["flask"], "port": 8080,
  "check": {"script": "curl localhost:8080/health >/dev/null 2>&1", "interval": "10s"}}}' >/etc/consul.d/microservice_a.json
```
Ahora, iniciamos el consul para el modo cliente, reemplazamos los campos con nuestra IP e identificador deseado.
```
consul agent -data-dir=/etc/consul/data -node=CODIGO_ESTUDIANTE \
    -bind=DIRECCION_IP -enable-script-checks=true -config-dir=/etc/consul.d
```
Antes de poder unirnos al **Discovery service**, este debe ejecutar el **PASO 3**, e iniciar Consul como modo servidor. Para ello, ejecuta el siguiente comando
```
consul agent -server -bootstrap-expect=1 \
   -data-dir=/etc/consul/data -node=agent-server -bind=DIRECCION_IP \
   -enable-script-checks=true -config-dir=/etc/consul.d -client 0.0.0.0
```
Ahora, los clientes pueden unirse al **Discovery Service**
```
consul join DIRECCION_IP_SERVIDOR
```

Todo lo anterior, se implementó con 4 máquinas conectadas a la misma red, el restado de esto fue que al ejecutar
```
consul members
```
Pudimos observar una lista de todas las máquinas conectadas usando consul, que es la siguiente
![][2]

Y al usar un navegador y poner la dirección IP de cualquiera de los clientes, nos desplegaría un mensaje que es el que configuramos en un inicio. Por ejemplo uno de ellos nos muestra  
![][3]

Consul, nos va mostrando los sucesos en la conexión, esta es una vista desde el servidor 
![][4]  

Ahora, implementaremos el **balanceador de carga**, que tiene como tarea controlar el flujo de las solicitudes externas a los consul clients, registrando el servidor y haciendo comunicación con el. 

Desde el **balanceador** instalamos lo siguiente

```
# yum install -y wget haproxy unzip
# wget https://releases.hashicorp.com/consul-template/0.19.4/consul-template_0.19.4_linux_amd64.zip -P /tmp
# unzip /tmp/consul-template_0.19.4_linux_amd64.zip -d /tmp
# mv /tmp/consul-template /usr/bin
# mkdir /etc/consul-template
```

Abrimos el puerto 5000
```
# firewall-cmd --zone=public --add-port=5000/tcp --permanent
# firewall-cmd --reload
```
Ahora configuramos el archivo haproxy.pl
```
vi /etc/consul-template/haproxy.tpl
```
Tomando de referencia lo siguiente

![][5]  

Finalmente, reiniciamos el servicio

```
sudo systemctl restart haproxy
```
Y podremos observar el **balanceador** ejecutando

![][6]  

4. Para probar que el **balanceador** está asignando las solicitudes a los consul clientes, hacemos un curl con la dirección IP del balanceador, y obtendremos distintos contenidos al pedir el microservicio, que son lo que configuraron los 4 clientes en un inicio en el archivo .py

![][7]  

5. Para agregar un microservicio diferente al ambiente ya existente, debemos tener en cuenta lo siguiente:

**API-GATEWAY:** Con un API-GATEWAY, evitariamos que implementar el código del mismo microservicio en varias máquinas. Lo que haría un consul client en un ambiente con esta herramienta implementada, es consultar el código del microservicio que se le encargó en este tipo de GATEWAY. Con esto, no tendremos que hacer **n** actualizaciones del microservicio en **n** máquinas, si no una sola vez en el API-GATEWAY. Adicionalmente, se optimiza espacio y tiempo, ya que todas las máquinas podrían ejecutar todos los microservicios.

**REACTIVE PARADIGM:** Si el nuevo servicio está involucrado directamente con el ya implementado, como que el output del nuevo microservicio B se despliegue en el microservicio A, es posible aprovechar la programación reactiva, donde todos los microservicios A se actualizarán inmediatamente el microservicio B se ejecute.

**Load Balancer:** Es nuestro balanceador de carga ya implementado, va a ser necesario para distribuir las solicitudes a las máquinas, ya que no hay forma de identificar que microservicio se requiere a menos que el **balanceador** exista.

**Prótocolo publicador/suscriptor:** Es la metodología que implementamos, dónde las máquinas con los microservicios se suscriben al **Discovery server**, para así tener un control sobre ellas y su ejecución.

Con estas herramientas y conceptos podremos implementar un servicio así:

1. Si queremos y estamos en la capacidad, podremos configurar un API-GATEWAY e implementaremos allí el viejo y nuevo código del microservicio. De lo contrario, implementaremos el nuevo microservicio en las máquinas ya existentes.

2. Registramos el nuevo servicio en el load-balancer, para que los clientes que hagan las solicitudes, puedan consumir el viejo y nuevo microservicio.

Con esto, finalmente podremos tener 2 microservicios distintos en nuestro ambiente.

### Referencias
https://github.com/ICESI/so-microservices-python  
http://microservices.io/patterns/microservices.html  
https://www.upcloud.com/support/haproxy-load-balancer-centos/  
http://microservices.io/patterns/apigateway.html
https://en.wikipedia.org/wiki/Reactive_programming

[1]: images/Microservices_Deployment.png
[2]: images/consul_members.PNG
[3]: images/browser_coni.PNG
[4]: images/consul_logs.PNG
[5]: images/configuracionConsulTemplates.png
[6]: images/BalanceadorCorriendo.png
[7]: images/curl_balancer.PNG
