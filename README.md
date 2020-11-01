Table of contents
=================

<!--ts-->
   * [Backup MongoDB - AWS Lambda & S3](#Backup-MongoDB---AWS-Lambda-&-S3)
   * [Healthcheck - AWS Route 53](#Healthcheck---AWS-Route-53)
<!--te-->

# Backup MongoDB - AWS Lambda & S3

AWS Lambda es un sistema capaz de ejecutar tareas en respuesta a eventos sin aprovisionar ni administrar servidores.

En este caso se va a usar para crear un *backup automático* de forma periódica de una base de datos Mongo.

## 1 - Crear una instancia S3

En la instancia S3 se van almacenar los backups de la base de datos y el script que permite llevar acabo esta operacion.

### 1.1 - Crear bucket

Crear el bucket y asegurarse de que la region seleccionada es la misma que la de los otros componentes de la arquitectura. Ademas se debe bloquear el acceso publico seleccionando *Block all public access*

Una vez creado el bucket crear una carpeta *backups* donde se almacenaran los backups creados.

[Amazon S3](
https://s3.console.aws.amazon.com/s3/home?region=us-east-2 "Amazon S3")

### 1.2 - Subir el script al S3

- Clonar este repositorio. (`nodejs` is required)
- Ejecutar `npm install`.
- Crear un zip y asegurar se comprimir el contenido y no el directorio en si. Para esto crear el zip desde linea de comandos y desde dentro del directorio ejecutar el comando `zip -r backup_script.zip .`
- Subir el zip al S3.

IMG 01

### 1.3 - Obtener el link del zip del sript en el S3

Seleccionar en S3 el zip y dentro de la pantalla de detalle en la solapa de *Overview* se encuentra el *Object URL* tomar nota del mismo, luego se usara para cargarlo en la funcion Lambda.

IMG 02

En el siguiente link se podran encontrar algunos ejemplos de scripts:
[Amazon Examples](https://github.com/awsdocs/aws-doc-sdk-examples "Amazon Examples")


## 2 - Crear Rol y Policy

Cómo todo componente de AWS es necesario establecer unos permisos para poder interactuar entre los demás componentes, en este ejemplo necesitamos tener permisos para que *Lambda* interactue con *EC2* y también con *CloudWatch* por si queremos ver a través de los log las operaciones realizadas.

El primer paso es ir a la seccion de [Identity and Access Management (IAM)](https://console.aws.amazon.com/iam "Identity and Access Management (IAM)") donde se administran los roles y policies.

### 2.1 Crear Policy

Crear una Policy selecionando como servicio *S3*, action *PutObject* y resources *All Resources* de este modo la funcion Lambda tendra los permisos para subir los backups al bucket. En este ejemplo la policy se llamara *S3-Backup*.

IMG 03
[Amazon Policies](https://console.aws.amazon.com/iam/home?region=us-east-2#/policies "Amazon Policies")

### 2.2 Crear Rol

Ahora lo siguiente va a ser crear el Rol, se debe selecionar como tipo de rol *AWS Service* y use case *Lambda*.

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


Las ultimas tres policies permiten registrar logs, monitorear la funcion lambda y crear alertas.

Por ultimo se debe asignarle un nombre, en este ejemplo se llamara *S3BackupRole*.

## 3 - Crear funcion Lambda

Finalmente se debe crear la funcional lambda que ejecutara periodicamente el script que realiza el backup de la base de datos.

[Amazon Lambda](https://us-east-2.console.aws.amazon.com/lambda "Amazon Lambda")

### 3.1 Información basica
Se debera crear a partir del template basico, se le debera dar un nombre, asignar la version de node y asignarle los permisos anteriormente creados.

### 3.2 Agregar Trigger

El trigger es el evento por el cual se ejecuta la funcion lambda. En este caso vamos a querer que el backup se ejecute periodicamente por ende se debe seleccionar el tipo *EventBridge (CloudWatch Events)*

El evento puede configurarse para que se dispare a una hora y dia especifico, por ejemplo si se quiere que se ejecute a las 2am UTC todos los dias `cron(0 2 * * ? *)` 

Minute - Hora - Day of month	- Month	 - Day of week	- Year

Todas las opciones estan descriptas en el siguiente link:

[Amazon Schedule Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html "Amazon Schedule Events")

### 3.3 Agregar codigo

En la seccion de *Function code* desde actions puede subirse el codigo desde un archivo o desde S3. Como el zip pesa mas de 10mb debe hacerse desde S3. El link del zip se obtuvo en el paso 1.3

### 3.4 Environment variables

El script requiere saber como conectarse a la base de datos y en donde se deben guardar los backups. Esta información se le pasa por medio de las variables de entorno.

- MONGO_URL `mongodb://<user>:<password>@<host>:<port>/<database>`
- S3_PATH `<s3bucket>/<folder>`

### 3.5 Basic settings

Desde esta seccion se debe configurar la memoria y el timeout de la funcion lambda.

### 3.6 Monitoring tools

Desde esta sección se puede habilitar *CloudWatch Lambda Insights* para trackear informacion de ejecución y performance de la función lambda.

## 4 - Conclusión

Tras completar la configuración anterior, ya tendríamos todos los requisitos para que funcionara la función de AWS Lambda. El propio panel de configuración nos da la opción de probar la función directamente (sin tener que esperar a que se ejecute por los “triggers” que se hayan establecido). El primer millón de llamadas a las funciones cada mes es gratuito.


# Healthcheck - AWS Route 53

En la siguiente seccion se describe como usar Amazon Route 53 para controlar de manera independiente el estado de la aplicación y sus puntos de enlace y ser notificado ante una eventual caida del server. Los costos de este servicio se encuentran en [Amazon Route 53 pricing](https://aws.amazon.com/es/route53/pricing/ "Amazon Route 53 pricing")


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

IMG 05

## 3 - Configurar Alarm

Una vez creado el Health checks se puede configurarle un Alarm para que se envie un mail o un sms y notifique el problema.

En la pantalla de creacion debe seleccionarse **Send notification** y seleccionar o crear un SNS topic y asignar los emails. Esto permitira crear una configuracion basica que permite ser notificados por email pero luego desde [Simple Notification Service (Amazon SNS)]( https://console.aws.amazon.com/sns/v3/home?region=us-east-1#/topics "Simple Notification Service (Amazon SNS)s") es posible realizar configuraciones mas avanzadas y seleccionar otros tipos de notificaciones.

IMG 06



