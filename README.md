## Despliegue de Camunda con Persistencia en Google Cloud Platform

A continuación se indica la manera de desplegar un servidor tomcat con el motor de Camunda. Los pasos mencionados los puede realizar directamente desde la shell de GCP o si tiene configurado el Google Cloud SDK en su Laptop tambien se puede hacer desde allí. En caso de que desee hacer el despliegue con un namespace personalizado dentro del cluster, deberá tener en cuenta que los elementos creados estén asociados a dicho namespace.

1. Subir imagen oficial de Camunda BPM Platform Edition Community al repositorio GCP. [Acá](https://hub.docker.com/r/camunda/camunda-bpm-platform) encuentra la imagen de docker.
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
    - Crear un *Kubernetes Secret* a partir de la llave de la cuenta de servicio.  
    `kubectl create secret generic cloudsql-instance-credentials --from-file=credentials.json=key.json`
    - Crear un *kubernetes Secret* a partir de valores literales en linea. Se puede agregar n valores sensibles en dicho *Secret*.  
    `kubectl create secret generic cloudsql-db-credentials --from-literal=username=[DB_USER] --from-literal=password=[DB_PASS]`
6. Usar el archivo de configmap con los valores previamente ajustados según la necesidad.  
`kubectl create -f [FILE_CONFIGMAP].yaml`
7. Usar el archivo de despliegue con los valores previamente ajustados según considere necesario.  
`kubectl create -f [FILE_DEPLOYMENT].yaml`
8. Revisar si el despliegue se realizó de manera correcta.  
`kubectl get pods`
9. Exponer el despliegue realizado usando el archivo de servicio con los valores previamente ajustados.  
`kubectl create -f [FILE_SERVICE].yaml`
