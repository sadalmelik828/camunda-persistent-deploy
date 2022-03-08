# Despliegue de Camunda en Google Cloud Platform

## Introducción

Los pasos mencionados a continuación utilizan una instancia SQL para la base de datos de la plataforma y un cluster de kubernetes. Estas instrucciones las puede realizar utilizando la shell de GCP o el SDK Google Cloud. Tambien se puede ayudar con las diferentes opciones que se encuentra en la consola de GCP.

Por otro lado debe tener en cuenta los permisos que requiere el usuario con el que va a realizar el despliegue ya que algunos comandos requieren permisos avanzados o minimo solicitar la creación de la instancia de base de datos y el cluster de kubernetes a un administrador de la nube.

## Requisitos

Los requisitos para ejecutar las instrucciones de manera correcta son los siguientes:

* Cuenta GCP con los respectivos permisos.
* [Google Cloud SDK](https://cloud.google.com/sdk/docs/install).
* [Cliente PostgreSQL](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) o [PGAdmin](https://www.pgadmin.org/download/).
* [kubernetes cli](https://kubernetes.io/docs/tasks/tools/).
## Instrucciones

1. Subir imagen oficial de Camunda BPM Platform Edition Community al repositorio GCP. [Acá](https://hub.docker.com/r/camunda/camunda-bpm-platform) encuentra la imagen oficial de docker o puede utilizar una imagen personalizada [aquí](https://hub.docker.com/repository/docker/sadalmelik828/camunda-bpm-platform) con algunas caracteristicas habilitadas.
2. Activar el API de SQL Admin GCP.
    - Conectarse al proyecto en cuestión.  
    `gcloud config set project [PROJECT_ID]`
    - Activar el API.  
    `gcloud services enable sqladmin.googleapis.com`
    - Verificar que queda activo SQL Admin API.  
    `gcloud services list --enabled`
3. Crear una instancia SQL.
    - Ejecutar el siguiente comando para crear una instancia de PostgreSQL:  
    `gcloud sql instances create --gce-zone us-central1-a --database-version POSTGRES_11 --memory 4 --cpu 1 [NAME_INSTANCE]`
    - Definir la contraseña para el usuario postgres:  
    `gcloud sql users set-password postgres --instance [INSTANCE_ID] --password [PASSWORD]`
    - Conectarse a la base de datos:  
    `gcloud sql connect [INSTANCE_ID] --user=postgres`
    - Una vez dentro de la base de datos, ejecutar los siguientes comandos SQL:  
    `CREATE DATABASE [CAMUNDA_DB_NAME];`  
    `CREATE USER [CAMUNDA_DB_USER] WITH PASSWORD '[PASSWORD]';`  
    `ALTER ROLE [CAMUNDA_DB_USER] SET default_transaction_isolation TO 'read committed';`  
    `ALTER ROLE [CAMUNDA_DB_USER] SET TIMEZONE TO 'America/Bogota';`  
    `GRANT ALL PRIVILEGES ON DATABASE [CAMUNDA_DB_NAME] TO [CAMUNDA_DB_USER];`  
    Con las sentencias anteriores, se crea una base de datos, usuario, zona horaria y se define la metodología de aislamiento de transacciones exclusivo para Camunda.
    - En caso de que no desee realizar los pasos anteriores para la creación de una instancia SQL, puede usar la consola gráfica de GCP. [Acá](https://cloud.google.com/sql/docs/postgres/create-instance) puede obtener mas información.
4. Crear una cuenta de servicio para usarlas a traves del proxy.
    - Ejecutar el siguiente comando para crear una cuenta de servicio:  
    `gcloud iam service-accounts create proxy-user --display-name "proxy-user"`
    - Si desea verifica la creación de la cuenta de servicio, puede ejecutar el siguiente comando:  
    `gcloud iam service-accounts list`
    - Ejecute el siguiente comando para otorgar permisos de acceso al role de **CloudSQL Client** en dicha cuenta de servicio:  
    `gcloud projects add-iam-policy-binding [PROJECT_ID] --member serviceAccount:[SERVICE_ACCOUNT_EMAIL] --role roles/cloudsql.client`
    - Ejecute el siguiente comando para crear una llave de autenticación con la cuenta de servicio:  
    `gcloud iam service-accounts keys create key.json --iam-account [SERVICE_ACCOUNT_EMAIL]`
5. Desplegar el contenedor de camunda en **Google Kubernetes Engine**.
    - En este paso se requiere la previa creación de un *cluster Kubernetes*. Si aún no lo tiene, puede crearlo con el siguiente comando:  
    `gcloud container clusters create [CLUSTER_NAME] --zone us-central1-a --machine-type=n1-standard-2 --enable-autorepair --enable-autoscaling --max-nodes=10 --min-nodes=1`
    - Conectarse al cluster previamente creado.  
    `gcloud container clusters get-credentials [CLUSTER_NAME] --zone us-central1-a`
    - Crear *namespace* para separar despliegue.  
    `kubectl create namespace camunda`
    - Crear un *Kubernetes Secret* a partir de la llave de la cuenta de servicio.  
    `kubectl create secret generic cloudsql-instance-credentials --from-file=credentials.json=key.json -n camunda`
    - Crear un *kubernetes Secret* a partir de valores literales en linea para el usuario y contraseña de base de datos. Se puede agregar n valores sensibles en dicho *Secret*.  
    `kubectl create secret generic cloudsql-db-credentials --from-literal=username=[DB_USER] --from-literal=password=[DB_PASS] -n camunda`
6. Usar el archivo de [configmap](https://github.com/sadalmelik828/camunda-persistent-deploy/blob/master/gcp/configMaps/camunda-full-persistence-configmap.yaml) con los valores previamente ajustados según la necesidad.  
    `kubectl create -f [FILE_CONFIGMAP].yaml`
7. Usar el archivo de [despliegue](https://github.com/sadalmelik828/camunda-persistent-deploy/blob/master/gcp/deployments/camunda-full-persistence-deployment.yaml) con los valores previamente ajustados según considere necesario.  
    `kubectl create -f [FILE_DEPLOYMENT].yaml`
8. Revisar si el despliegue se realizó de manera correcta.  
    `kubectl get pods -n camunda`
9. Exponer el despliegue realizado usando el archivo de [servicio](https://github.com/sadalmelik828/camunda-persistent-deploy/blob/master/gcp/services/camunda-full-persistence-service.yml) con los valores previamente ajustados.  
    `kubectl create -f [FILE_SERVICE].yaml`
10. Revisar los endpoints generados que expone el servidor de camunda.  
    `kubectl get svc -n camunda`
