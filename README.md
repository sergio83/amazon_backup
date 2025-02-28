Table of contents
=================

<!--ts-->
   * [Backup MongoDB - AWS Lambda & S3](#Backup-MongoDB---AWS-Lambda--S3)
   * [Healthcheck - AWS Route 53](#Healthcheck---AWS-Route-53)
   * [Keep server online 24&#47;7 - PM2](#Keep-server-online-247---PM2)
<!--te-->

# Backup MongoDB - AWS Lambda & S3

AWS Lambda es un sistema capaz de ejecutar tareas en respuesta a eventos sin aprovisionar ni administrar servidores.

En este caso se va a usar para crear un **backup automático** de forma periódica de una base de datos Mongo.

## 1 - Crear una instancia S3

En la instancia S3 se van a almacenar los backups de la base de datos y el script que permite llevar acabo esta operación.

### 1.1 Crear bucket

Crear el bucket y asegurarse de que la región seleccionada es la misma que la de los otros componentes de la arquitectura. Ademas se debe bloquear el acceso publico seleccionando **Block all public access**

Una vez creado el bucket crear una carpeta *backups* donde se almacenaran los backups creados.

[Amazon S3](
https://s3.console.aws.amazon.com/s3/home?region=us-east-2 "Amazon S3")

### 1.2 Subir el script al S3

- Clonar este repositorio. (`nodejs` is required)
- Ejecutar `npm install`.
- Crear un zip y asegurarse de comprimir el contenido y no el directorio en si. Para esto crear el zip desde línea de comandos y desde dentro del directorio ejecutar el comando `zip -r backup_script.zip .`
- Subir el zip al S3.

<br />

![01](img/01.png)

<br />

### 1.3 Obtener el link del zip del sript en el S3

Seleccionar en S3 el zip y dentro de la pantalla de detalle en la solapa de **Overview** se encuentra el **Object URL** tomar nota del mismo, luego se usara para cargarlo en la función Lambda.

<br />

![02](img/02.png)

<br />

En el siguiente link se podrán encontrar algunos ejemplos de scripts:
[Amazon Examples](https://github.com/awsdocs/aws-doc-sdk-examples "Amazon Examples")


## 2 - Crear Rol y Policy

Cómo todo componente de AWS es necesario establecer unos permisos para poder interactuar entre los demás componentes, en este ejemplo necesitamos tener permisos para que *Lambda* interactúe con **EC2** y también con **CloudWatch** por si queremos ver a través de los log las operaciones realizadas.

El primer paso es ir a la sección de [Identity and Access Management (IAM)](https://console.aws.amazon.com/iam "Identity and Access Management (IAM)") donde se administran los roles y policies.

### 2.1 Crear Policy

Crear una Policy selecionando como servicio **S3**, actión **PutObject** y resources **All Resources** de este modo la función Lambda tendrá los permisos para subir los backups al bucket. En este ejemplo la policy se llamara **S3-Backup**.

[Amazon Policies](https://console.aws.amazon.com/iam/home?region=us-east-2#/policies "Amazon Policies")

<br />

![03](img/03.png)

<br />

### 2.2 Crear Rol

Ahora lo siguiente va a ser crear el Rol, se debe seleccionar como tipo de rol **AWS Service* y use case *Lambda**.

Ademas se deben asignar esta las siguientes policies:
- S3-Backup
- CloudWatchAgentServerPolicy
- AWSLambdaBasicExecutionRole
- CloudWatchLambdaInsightsExecutionRolePolicy

| Policy  | Description |
| ------------- | ------------- |
| S3-Backup  | Provides write permissions to S3  |
| CloudWatchAgentServerPolicy  | Permissions required to use AmazonCloudWatchAgent on servers  |
| AWSLambdaBasicExecutionRole  | Provides write permissions to CloudWatch Logs |
| CloudWatchLambdaInsightsExecutionRolePolicy  | Policy required for the Lambda Insights Extension  |


Las ultimas tres policies permiten registrar logs, monitorear la función lambda y crear alertas.

Por último se debe asignarle un nombre, en este ejemplo se llamara *S3BackupRole*.

## 3 - Crear función Lambda

Finalmente se debe crear la funcional lambda que ejecutara periódicamente el script que realiza el backup de la base de datos.

[Amazon Lambda](https://us-east-2.console.aws.amazon.com/lambda "Amazon Lambda")

### 3.1 Información básica
Se deberá crear a partir del template básico, se le deberá dar un nombre, asignar la versión de node y asignarle los permisos anteriormente creados.


<br />

![04](img/04.png)

<br />

### 3.2 Agregar Trigger

El trigger es el evento por el cual se ejecuta la función lambda. En este caso vamos a querer que el backup se ejecute periódicamente por ende se debe seleccionar el tipo *EventBridge (CloudWatch Events)*

El evento puede configurarse para que se dispare a una hora y día especifico, por ejemplo si se quiere que se ejecute a las 2am UTC todos los días `cron(0 2 * * ? *)` 

Minute - Hora - Day of month	- Month	 - Day of week	- Year

Todas las opciones estan descriptas en el siguiente link:

[Amazon Schedule Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html "Amazon Schedule Events")

### 3.3 Agregar código

En la sección de **Function code** desde actions puede subirse el código desde un archivo o desde S3. Como el zip pesa más de 10mb debe hacerse desde S3. El link del zip se obtuvo en el paso 1.3

### 3.4 Environment variables

El script requiere saber como conectarse a la base de datos y en donde se deben guardar los backups. Esta información se le pasa por medio de las variables de entorno.

- MONGO_URL `mongodb://<user>:<password>@<host>:<port>/<database>`
- S3_PATH `<s3bucket>/<folder>`

### 3.5 Basic settings

Desde esta sección se debe configurar la memoria y el timeout de la función lambda.

### 3.6 Monitoring tools

Desde esta sección se puede habilitar **CloudWatch Lambda Insights** para trackear información de ejecución y performance de la función lambda.

## 4 - Conclusión

Tras completar la configuración anterior, ya tendríamos todos los requisitos para que funcionara la función de AWS Lambda. El propio panel de configuración nos da la opción de probar la función directamente (sin tener que esperar a que se ejecute por los “triggers” que se hayan establecido). El primer millón de llamadas a las funciones cada mes es gratuito.

<br />

------------

# Healthcheck - AWS Route 53

En la siguiente sección se describe como usar Amazon Route 53 para controlar de manera independiente el estado de la aplicación y sus puntos de enlace y ser notificado ante una eventual caída del server. Los costos de este servicio se encuentran en [Amazon Route 53 pricing](https://aws.amazon.com/es/route53/pricing/ "Amazon Route 53 pricing")


## 1 - Crear un el Healthcheck endpoint

Se debe exponer en la instancia EC2 que se quiere validar un servicio que simplemente devuelva un status code 200.

```javascript
const router = require("express").Router();

router.get("/", async function (req, res) {
	res.status(200).send();
});

module.exports = router;
```
## 2 - Crear el Health checks

Se debe configurar Route 53 para que llame cada 30 segundos al endpoint expuesto en EC2. 

<br />

![05](img/05.png)

<br />

## 3 - Configurar Alarm

Una vez creado el Health checks se puede configurarle un Alarm para que se envíe un mail o un sms y notifique el problema.

En la pantalla de creación debe seleccionarse **Send notification** y seleccionar o crear un SNS topic y asignar los emails. Esto permitirá crear una configuración básica que permite ser notificados por email pero luego desde [Simple Notification Service (Amazon SNS)]( https://console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics "Simple Notification Service (Amazon SNS)s") es posible realizar configuraciones más avanzadas y seleccionar otros tipos de notificaciones.

<br />

![06](img/06.png)

<br />

------------

# Keep server online 24/7 - PM2

[PM2](https://pm2.keymetrics.io/ "PM2") es un daemon que permite manejar procesos y restablecerlos ante eventuales fallos y caídas de los mismos. Ademas permite de forma fácil implementar balanceo de carga y monitoreo de performance y estado de los servidores.

En el caso en que sé este usando [Docker)](https://www.docker.com/ "Docker") los containers se restablecen según las restart policy [restart policy)](https://docs.docker.com/compose/compose-file/#restart "restart policy") configuradas en el **docker-compose.yml** sin embargo para que los servidores que corren dentro de cada container se restablezcan es necesario usar algún manejador de procesos como **PM2**

## PM2 Instalación y Configuración

En la [documentación oficial](https://pm2.keymetrics.io/docs/usage/docker-pm2-nodejs/ "documentación oficial") se describen los pasos básicos de la integración de PM2.

El primer paso es instalar PM2 esto debe hacerse en el **Dokerfile**:
```javascript
FROM node:12-slim
WORKDIR /app

COPY ./package.json ./package-lock.json ./
RUN npm ci
RUN npm install pm2 -g

COPY . .
CMD ["./run.sh"]
```

luego debe ejecutarse desde **run.sh**:

```javascript
#!/bin/sh

pm2-runtime start ecosystem.config.js
```

por ultimo se debe configurar PM2 esto se hace por medio del archivo de configuración **ecosystem.config.js**:

```javascript
module.exports = [{
  script: 'server.js',
  name: 'bettervet_api',
  exec_mode: 'cluster',
  instances: 2
}]
```

Las opciones básicas son:

| Policy  | Description |
| ------------- | ------------- |
| script  | script path relative to pm2 start  |
| name  | application name (default to script filename without extension)  |
| exec_mode  | mode to start your app, can be “cluster” or “fork”, default fork |
| instances  | number of app instance to be launched  |

La lista completa de opciones disponibles se encuentran en el siguiente [link)](https://pm2.keymetrics.io/docs/usage/application-declaration/#attributes-available "link")

## Load Balancing

Si el servidor es **Stateless Application** (el servidor no tiene un estado interno en memoria) entonces es posible configurar PM2 para que ejecute más de una instancia y haga [balanceo de carga)](https://pm2.io/docs/runtime/guide/load-balancing/ "balanceo de carga"). Para esto debe configurarse **exec_mode** como **cluster** por otro lado se puede configurar la cantidad de **instances** que se van a correr. 

Las opciones posibles son:
- **max** PM2 detectará automáticamente la cantidad de CPU disponibles y ejecutará tantos procesos como sea posible
- **integer** cantidad de procesos a ejecutar


## Monitoring & Logs

Para poder monitorear y ver los estados es necesario ingresar dentro del Docker container en el que está corriendo el servidor. 

Los pasos a seguir son:

#### 1 - Ingresar al servidor por ssh

`ssh -i certificate.pem ubuntu@ip`

#### 2 - Ver las containers que hay corriendo

`docker ps`

#### 3 - Ingresar al container

`docker exec -it container_name bash`

#### 4 - PM2 status

Una vez dentro del container se tiene acceso a PM2, ejecutando el siguiente comando es posible ver el estado del servidor:

`pm2 status`

<br />

![07](img/07.png)

<br />


#### 5 - PM2 logs

`pm2 logs --json` 
