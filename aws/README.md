# Despliegue de Camunda en Amazon Web Services

## Introducción
---
Los pasos mencionados a continuación utilizan una instancia SQL de PostgreSQL en RDS para la base de datos de la plataforma y un cluster de kubernetes EKS. Estas instrucciones las puede realizar utilizando la linea de comandos o la consola de AWS.

Por otro lado debe tener en cuenta los permisos que requiere el usuario con el que va a realizar el despliegue ya que algunos comandos requieren permisos avanzados o minimo solicitar la creación de la instancia de base de datos y el cluster de kubernetes a un administrador de la nube.

## Requisitos
---
Los requisitos para ejecutar las instrucciones de manera correcta son los siguientes:

* Cuenta AWS con los respectivos permisos.
* [aws cli](https://docs.aws.amazon.com/es_es/cli/latest/userguide/install-cliv2.html).
* [Cliente PostgreSQL](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) o [PGAdmin](https://www.pgadmin.org/download/).
* [kubernetes cli](https://kubernetes.io/docs/tasks/tools/).

## Instrucciones
---
1. Subir imagen oficial de Camunda BPM Platform Edition Community al ECR de AWS. [Acá](https://hub.docker.com/r/camunda/camunda-bpm-platform) encuentra la imagen oficial de docker o puede utilizar una imagen personalizada [aquí](https://hub.docker.com/repository/docker/sadalmelik828/camunda-bpm-platform).
2. Crear una instancia SQL.
    - Ejecutar el siguiente comando para crear una instancia de PostgreSQL en RDS:  
    `aws rds create-db-instance --engine postgres --engine-version 13.4 --db-instance-identifier [INSTANCE_DB_NAME] --allocated-storage 50 --storage-type gp2 --db-instance-class db.t3.medium --db-parameter-group-name default.postgres13 --option-group-name default.postgres13 --storage-encrypted true --kms-key-id aws/rds --enable-cloudwatch-logs-exports postgresql upgrade --vpc-security-group-ids [IDS_SECURITY_GROUPS] --db-subnet-group [SUBNET_GROUP_NAME] --master-username [MASTER_USER] --master-user-password [PASSWD_MASTER_USER] --backup-retention-period 7 --availability-zone us-west-2b`
    - Conectarse a la base de datos.  
    > Para conectarse a la DB creada, se requiere el cliente de PostgreSQL o el pgadmin. Los pasos para habilitar una conexión con PostgreSQL RDS los puede encontrar [aqui](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.PostgreSQL.html) y [aqui](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html).  
    - Una vez conectado a la base de datos, ejecutar los siguientes comandos SQL:  
    `CREATE DATABASE [CAMUNDA_DB_NAME];`  
    `CREATE USER [CAMUNDA_DB_USER] WITH PASSWORD '[PASSWORD]';`  
    `ALTER ROLE [CAMUNDA_DB_USER] SET default_transaction_isolation TO 'read committed';`  
    `ALTER ROLE [CAMUNDA_DB_USER] SET TIMEZONE TO 'America/Bogota';`  
    `GRANT ALL PRIVILEGES ON DATABASE [CAMUNDA_DB_NAME] TO [CAMUNDA_DB_USER];`  
    Con las sentencias anteriores, se crea una base de datos, usuario, zona horaria y se define la metodología de aislamiento de transacciones exclusivo para Camunda.
    - En caso de que no desee realizar los pasos anteriores para la creación de una instancia SQL, puede usar la consola gráfica de AWS. [Acá](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.PostgreSQL.html) puede obtener mas información.
3. Desplegar el contenedor de camunda en **Google Kubernetes Engine**.
    - En este paso se requiere la previa creación de un *cluster Kubernetes*. Si aún no lo tiene, puede crearlo con el siguiente comando:  
    `gcloud container clusters create [CLUSTER_NAME] --zone us-central1-a --machine-type=n1-standard-2 --enable-autorepair --enable-autoscaling --max-nodes=10 --min-nodes=1`
    - Conectarse al cluster previamente creado.  
    `gcloud container clusters get-credentials [CLUSTER_NAME] --zone us-central1-a`
    - Crear *namespace* para separar despliegue.  
    `kubectl create namespace camunda`
    - Crear un *Kubernetes Secret* a partir de la llave de la cuenta de servicio.  
    `kubectl create secret generic cloudsql-instance-credentials --from-file=credentials.json=key.json -n camunda`
    - Crear un *kubernetes Secret* a partir de valores literales en linea. Se puede agregar n valores sensibles en dicho *Secret*.  
    `kubectl create secret generic cloudsql-db-credentials --from-literal=username=[DB_USER] --from-literal=password=[DB_PASS] -n camunda`
4. Usar el archivo de configmap con los valores previamente ajustados según la necesidad.  
    `kubectl create -f [FILE_CONFIGMAP].yaml`
5. Usar el archivo de despliegue con los valores previamente ajustados según considere necesario.  
    `kubectl create -f [FILE_DEPLOYMENT].yaml`
6. Revisar si el despliegue se realizó de manera correcta.  
    `kubectl get pods -n camunda`
7. Exponer el despliegue realizado usando el archivo de servicio con los valores previamente ajustados.  
    `kubectl create -f [FILE_SERVICE].yaml`
8. Revisar los endpoints generados para exposición.  
    `kubectl get svc -n camunda`
