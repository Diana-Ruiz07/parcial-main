# Informe Técnico: Arquitectura Multi-Contenedor y Análisis del Modelo OSI

**Imtegrantes:** Fedrico Castro, Daniela Niño, Diana Ruiz
**Asignatura:** Comunicaciones
**Carrera:** Ingeniería Mecatrónica
**Universidad Militar Nueva Granada**

---

# Sección 1: Topología y Flujo de Información

## 1.1 Diagrama de Arquitectura

La solución está compuesta por cinco contenedores Docker: **Nginx, Joomla, PostgreSQL, Jupyter y Grafana**. Los contenedores se distribuyen en dos redes tipo `bridge`: `frontend_net` y `backend_net`.

```text
                         USUARIO / NAVEGADOR
                                |
                                | HTTP :80
                                v
                    +------------------------+
                    |         NGINX          |
                    |    Reverse Proxy       |
                    |     Puerto 80          |
                    +-----------+------------+
                                |
              +-----------------+------------------+
              |                 |                  |
              | HTTP            | HTTP/WebSocket   | HTTP
              v                 v                  v
       +-------------+    +-------------+    +-------------+
       |   JOOMLA    |    |   JUPYTER   |    |   GRAFANA   |
       |    :80      |    |    :8888    |    |    :3000    |
       +------+------+    +-------------+    +------+------+
              |                                    |
              | TCP :5432                           |
              |                                    |
              +----------------+-------------------+
                               |
                               v
                     +-------------------+
                     |     DATABASE      |
                     |   PostgreSQL :5432|
                     +-------------------+

       ================= REDES DOCKER =================

       frontend_net:
       Nginx <----> Joomla
          |-----> Jupyter
          |-----> Grafana

       backend_net:
       Joomla <----> PostgreSQL
          |
          +-------> Grafana
```

### Redes utilizadas

| Contenedor | frontend_net | backend_net | Puerto interno |
| ---------- | ------------ | ----------- | -------------- |
| Nginx      | Sí           | No          | 80             |
| Joomla     | Sí           | Sí          | 80             |
| Jupyter    | Sí           | No          | 8888           |
| Grafana    | Sí           | Sí          | 3000           |
| PostgreSQL | No           | Sí          | 5432           |

El único contenedor que publica un puerto hacia el equipo host es **Nginx**, mediante:

```yaml
ports:
  - "80:80"
```

Por esta razón, el usuario accede a los diferentes servicios a través del proxy inverso:

```text
http://localhost/
http://localhost/jupyter/
http://localhost/grafana/
```

PostgreSQL no publica directamente el puerto `5432` hacia el host, por lo que permanece disponible únicamente dentro de la red interna de Docker.

---

## 1.2 Flujo de información

Cuando un usuario accede a `http://localhost/`, la solicitud llega al puerto 80 del contenedor Nginx. Nginx identifica la ruta solicitada y la reenvía al servicio correspondiente.

Para Joomla:

```text
Navegador
   |
   | HTTP :80
   v
Nginx
   |
   | HTTP
   v
Joomla :80
   |
   | PostgreSQL TCP :5432
   v
Database
```

Para Jupyter:

```text
Navegador
   |
   | HTTP/WebSocket
   v
Nginx
   |
   | HTTP :8888
   v
Jupyter
```

Para Grafana:

```text
Navegador
   |
   | HTTP :80
   v
Nginx
   |
   | HTTP :3000
   v
Grafana
   |
   | PostgreSQL :5432
   v
Database
```

---

## 1.3 Recolección de métricas y datos para Grafana

En esta implementación, Grafana obtiene los datos directamente desde **PostgreSQL**, utilizando un datasource configurado automáticamente mediante provisioning.

El datasource está definido en:

```text
grafana/provisioning/datasources/
```

y utiliza:

```text
database:5432
```

como dirección del servidor PostgreSQL.

Las credenciales corresponden a las variables configuradas para Joomla y PostgreSQL:

```text
Base de datos: joomladb
Usuario: joomlauser
Contraseña: joomlapassword
```

De esta manera, Grafana puede realizar consultas SQL sobre las tablas creadas por Joomla.

Por ejemplo, el panel de sesiones utiliza:

```sql
SELECT count(*) AS total_sesiones
FROM fl3x2_session;
```

Mientras que el panel de contenidos utiliza:

```sql
SELECT state AS estado, count(*) AS cantidad
FROM fl3x2_content
GROUP BY state;
```

El prefijo `#__` utilizado en Joomla corresponde al mecanismo de prefijos de tablas del CMS. En la instalación realizada, el prefijo efectivo es `fl3x2_`.

Además, Jupyter utiliza Python y librerías como `psycopg2`, `SQLAlchemy` y `pandas` para realizar consultas y procesamiento de información de PostgreSQL.

Por lo tanto, el flujo de datos para los paneles es:

```text
Joomla
   |
   | almacena información
   v
PostgreSQL
   |
   | consultas SQL
   v
Grafana
   |
   | HTTP
   v
Nginx
   |
   v
Navegador
```

---

# Sección 2: Análisis Detallado del Modelo OSI en la Solución

## 2.1 Capa 7 — Aplicación

La capa de aplicación corresponde a los protocolos y servicios utilizados directamente por las aplicaciones. En este proyecto participan principalmente **HTTP, WebSocket y PostgreSQL**.

### Cabeceras HTTP utilizadas por Nginx

Nginx funciona como reverse proxy y agrega o conserva información mediante diferentes cabeceras HTTP.

Se utiliza:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

### Host

La cabecera:

```text
Host
```

permite indicar el nombre del host solicitado originalmente por el cliente.

Esto permite que Joomla conozca el host utilizado por el usuario aunque la petición haya pasado previamente por Nginx.

### X-Real-IP

La cabecera:

```text
X-Real-IP
```

permite enviar al servidor backend la dirección IP del cliente que realizó la solicitud.

### X-Forwarded-For

La cabecera:

```text
X-Forwarded-For
```

mantiene información sobre las direcciones IP involucradas en el recorrido de la solicitud. Es especialmente útil cuando existen uno o varios proxies.

### X-Forwarded-Proto

La cabecera:

```text
X-Forwarded-Proto
```

indica el protocolo utilizado originalmente por el cliente, por ejemplo:

```text
http
```

o:

```text
https
```

Esto permite que las aplicaciones backend conozcan el protocolo original aunque se encuentren detrás de un proxy.

---

## HTTP Upgrade y WebSockets en Jupyter

Jupyter utiliza conexiones WebSocket para determinadas comunicaciones interactivas entre el navegador y el servidor, especialmente para mantener comunicación en tiempo real con el kernel.

Por esta razón, Nginx utiliza:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

La cabecera `Upgrade` permite solicitar el cambio de una conexión HTTP a un protocolo de comunicación como WebSocket.

Sin esta configuración, algunas funciones interactivas de Jupyter pueden no funcionar correctamente al acceder mediante el proxy inverso.

También se establece:

```nginx
proxy_read_timeout 86400;
```

para evitar que Nginx cierre prematuramente conexiones de larga duración.

---

## PostgreSQL como protocolo cliente/servidor

PostgreSQL utiliza una arquitectura cliente/servidor. En este proyecto:

```text
Joomla / Grafana / Jupyter
          |
          | PostgreSQL protocol
          v
      PostgreSQL
```

La comunicación se realiza mediante TCP sobre el puerto:

```text
5432
```

Joomla utiliza PostgreSQL como sistema de almacenamiento de información del CMS, mientras que Grafana utiliza el mismo servidor como fuente de datos.

Jupyter también puede conectarse mediante Python utilizando librerías como `psycopg2` o `SQLAlchemy`.

---

## Logs generados por Joomla

Joomla utiliza el servidor Apache dentro del contenedor para atender las solicitudes HTTP. Los registros pueden almacenarse en:

```text
/var/log/apache2
```

En este proyecto dicho directorio se comparte mediante un bind mount con:

```text
./joomla_logs
```

Esto permite conservar los archivos de registro fuera del sistema de archivos efímero del contenedor.

Los logs de Apache contienen información asociada a las solicitudes HTTP, como la dirección del cliente, fecha y hora, método HTTP, recurso solicitado y código de respuesta.

---

# 2.2 Capa 4 — Transporte

La capa de transporte se encarga de la comunicación extremo a extremo entre los servicios.

En esta arquitectura se utiliza principalmente **TCP**.

Los puertos involucrados son:

| Servicio   | Puerto TCP | Función                           |
| ---------- | ---------: | --------------------------------- |
| Nginx      |         80 | Entrada HTTP desde el host        |
| Joomla     |         80 | Servidor web interno              |
| PostgreSQL |       5432 | Comunicación con la base de datos |
| Jupyter    |       8888 | Servidor Notebook                 |
| Grafana    |       3000 | Interfaz y API de Grafana         |

El usuario solamente accede directamente al host mediante:

```text
localhost:80
```

Los demás puertos son utilizados internamente entre contenedores.

---

## Conexiones concurrentes

Nginx puede mantener múltiples conexiones TCP simultáneas con diferentes clientes y servidores backend.

Por ejemplo:

```text
Cliente 1 ──┐
Cliente 2 ──┼──> Nginx ──> Joomla
Cliente 3 ──┘
```

De manera similar, Joomla puede establecer conexiones TCP hacia PostgreSQL para realizar consultas y operaciones sobre la base de datos.

El uso de conexiones persistentes y mecanismos de reutilización de conexiones permite reducir el costo de establecer repetidamente nuevas conexiones TCP.

Grafana también mantiene conexiones con PostgreSQL para ejecutar las consultas requeridas por sus paneles.

---

# 2.3 Capa 3 — Red

La capa 3 se encarga del direccionamiento IP y del encaminamiento de los paquetes entre las diferentes redes.

Docker crea redes virtuales tipo `bridge`:

```text
frontend_net
backend_net
```

La distribución utilizada es:

```text
frontend_net:
    nginx
    joomla
    jupyter
    grafana

backend_net:
    joomla
    database
    grafana
```

Esta separación permite aislar el servidor PostgreSQL de la red externa.

---

## Direccionamiento IP

Los contenedores reciben direcciones IP privadas dentro de las redes Docker.

Estas direcciones son administradas automáticamente por Docker.

Sin embargo, las aplicaciones no necesitan conocer directamente las direcciones IP de los contenedores, ya que pueden utilizar sus nombres de servicio.

Por ejemplo:

```text
database
joomla
jupyter
grafana
```

---

## DNS embebido de Docker

Docker proporciona un servidor DNS interno, normalmente accesible desde los contenedores mediante:

```text
127.0.0.11
```

Este servicio permite resolver los nombres de los servicios definidos en `docker-compose.yml`.

Por ejemplo:

```text
database
```

se resuelve hacia la dirección IP interna correspondiente al contenedor PostgreSQL.

Por esta razón, Joomla utiliza:

```text
JOOMLA_DB_HOST=database
```

en lugar de una dirección IP fija.

De igual forma, Nginx puede utilizar:

```nginx
proxy_pass http://joomla:80;
proxy_pass http://jupyter:8888;
proxy_pass http://grafana:3000;
```

---

## NAT y reenvío de paquetes

Docker configura reglas de red en el sistema operativo host para permitir la comunicación entre las redes virtuales y el host.

Cuando un usuario accede a:

```text
http://localhost/
```

la conexión llega al puerto 80 publicado por Docker y posteriormente es dirigida al puerto 80 del contenedor Nginx.

El flujo puede representarse como:

```text
Host
  |
  | Puerto 80
  v
NAT / reglas de Docker
  |
  v
Nginx :80
```

Los mecanismos de red del kernel permiten además que los contenedores pertenecientes a una misma red virtual puedan comunicarse entre sí.

---

# 2.4 Capa 2 — Enlace de Datos

En Docker, las redes tipo `bridge` permiten la comunicación de los contenedores mediante interfaces virtuales.

Cada contenedor posee una interfaz de red virtual conectada a la infraestructura de red de Docker.

Conceptualmente:

```text
Contenedor
    |
   veth
    |
    v
bridge Docker
    |
    +---- veth ---- Contenedor
    |
    +---- veth ---- Contenedor
```

Las interfaces `veth*` funcionan como pares virtuales: un extremo se encuentra asociado al contenedor y el otro al bridge administrado por Docker.

---

## Bridges de Docker

Para este proyecto Docker crea bridges asociados a:

```text
frontend_net
backend_net
```

Los bridges permiten transportar tramas Ethernet entre los contenedores pertenecientes a la misma red.

---

## Resolución ARP

Cuando un contenedor necesita comunicarse con otro dentro de la misma red bridge, necesita conocer la dirección MAC correspondiente a la dirección IP destino.

Para ello puede utilizar ARP.

El proceso puede representarse como:

```text
Contenedor A
    |
    | ARP
    | "¿Quién tiene esta IP?"
    v
Bridge Docker
    |
    v
Contenedor B
```

Una vez conocida la dirección MAC, las tramas Ethernet pueden ser enviadas hacia el destino correspondiente.

De esta manera, las redes virtuales de Docker proporcionan una infraestructura de comunicación similar, conceptualmente, a una red Ethernet física.

---

# Sección 3: Guía de Verificación y Demostración

## 3.1 Verificación de los contenedores

Desde la raíz del proyecto se ejecuta:

```powershell
docker compose up -d
```

El despliegue debe iniciar los cinco servicios.

Para comprobar su estado:

```powershell
docker compose ps
```

Se debe observar:

```text
database   Up (healthy)
grafana    Up
joomla     Up
jupyter    Up (healthy)
nginx      Up
```

El único servicio que debe mostrar un puerto publicado hacia el host es Nginx:

```text
0.0.0.0:80->80/tcp
```

---

## 3.2 Verificación de Joomla

Abrir un navegador y acceder a:

```text
http://localhost/
```

La solicitud llega primero a Nginx y posteriormente es enviada al contenedor Joomla.

Para generar tráfico HTTP se puede navegar por diferentes páginas del sitio.

Por ejemplo:

```text
Página principal
    ↓
Menús
    ↓
Artículos
    ↓
Diferentes recursos del sitio
```

Cada navegación genera nuevas solicitudes HTTP.

Los registros pueden verificarse en la carpeta:

```text
joomla_logs/
```

---

## 3.3 Verificación de Grafana

Acceder mediante:

```text
http://localhost/grafana/
```

Grafana se encuentra configurado para utilizar PostgreSQL como fuente de datos.

La configuración del datasource se realiza automáticamente mediante:

```text
grafana/provisioning/datasources/
```

Por lo tanto, no es necesario crear manualmente el datasource desde la interfaz gráfica.

El dashboard se carga automáticamente mediante:

```text
grafana/provisioning/dashboards/
```

El panel contiene, como mínimo, dos visualizaciones:

1. **Registro de Sesiones Activas en Joomla**
2. **Conteo de Contenidos Creados por Estado**

La primera utiliza una consulta SQL sobre:

```text
fl3x2_session
```

y la segunda utiliza:

```text
fl3x2_content
```

Las consultas se realizan directamente sobre PostgreSQL.

---

## 3.4 Verificación de Jupyter

Acceder mediante:

```text
http://localhost/jupyter/
```

El notebook se encuentra precargado mediante un bind mount:

```text
./jupyter/notebooks
```

hacia:

```text
/home/jovyan/work
```

Dentro de Jupyter se encuentra:

```text
analisis_datos.ipynb
```

El cuaderno contiene código Python para consultar y analizar información almacenada en PostgreSQL.

Las librerías utilizadas incluyen:

```text
pandas
matplotlib
psycopg2-binary
SQLAlchemy
```

El entorno fue comprobado mediante Jupyter y Python. La instalación de las librerías se realiza automáticamente mediante el archivo:

```text
jupyter/requirements.txt
```

Por ejemplo, una conexión a PostgreSQL puede utilizar la información:

```text
Host: database
Puerto: 5432
Base de datos: joomladb
Usuario: joomlauser
```

La consulta permite obtener información almacenada en las tablas de Joomla y procesarla mediante Python.

---

# Sección 4: Despliegue Desatendido

El proyecto está diseñado para realizar un despliegue Zero-Touch.

Los archivos principales se encuentran en la raíz:

```text
docker-compose.yml
.env.example
README.md
INFORME.md
```

Las variables de configuración se encuentran documentadas en:

```text
.env.example
```

Entre ellas:

```text
POSTGRES_DB=joomladb
POSTGRES_USER=joomlauser
POSTGRES_PASSWORD=joomlapassword

JOOMLA_DB_HOST=database
JOOMLA_DB_USER=joomlauser
JOOMLA_DB_PASSWORD=joomlapassword
JOOMLA_DB_NAME=joomladb
JOOMLA_DB_TYPE=pgsql
```

El objetivo es que el proyecto pueda iniciarse ejecutando únicamente:

```powershell
docker compose up -d
```

sin requerir la creación manual de datasources, dashboards o notebooks.

---

# Sección 5: Persistencia

La información de PostgreSQL se almacena mediante un volumen Docker:

```text
postgres_data
```

montado en:

```text
/var/lib/postgresql/data
```

Joomla utiliza un volumen Docker:

```text
joomla_data
```

montado en:

```text
/var/www/html
```

Grafana utiliza:

```text
grafana_data
```

montado en:

```text
/var/lib/grafana
```

El notebook de Jupyter utiliza un bind mount desde el repositorio:

```text
./jupyter/notebooks
```

hacia:

```text
/home/jovyan/work
```

De esta forma, los datos importantes y los archivos necesarios para el funcionamiento del proyecto no dependen exclusivamente del ciclo de vida individual de los contenedores.

---

# Sección 6: Conclusiones

La solución implementada integra cinco servicios mediante Docker Compose, utilizando Nginx como punto de entrada único para los servicios web.

La arquitectura utiliza dos redes virtuales para separar el tráfico externo del acceso interno a PostgreSQL. Esta distribución permite que la base de datos no tenga un puerto publicado directamente hacia el host.

Nginx realiza el encaminamiento de las solicitudes hacia Joomla, Jupyter y Grafana, incluyendo la configuración necesaria para soportar WebSockets utilizados por Jupyter.

PostgreSQL funciona como sistema de almacenamiento de Joomla y como fuente de datos para Grafana. Mediante el mecanismo de provisioning, Grafana obtiene automáticamente su datasource y dashboard al iniciar los contenedores.

Jupyter proporciona un entorno de análisis mediante Python y permite consultar y procesar la información almacenada en PostgreSQL.

Finalmente, la solución puede desplegarse mediante Docker Compose y mantiene la configuración, persistencia y contenido necesarios para que los servicios puedan ser verificados sin realizar configuraciones manuales dentro de las interfaces de Grafana o Jupyter.
