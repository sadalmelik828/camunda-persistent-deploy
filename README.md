# Despliegue de plataforma Camunda con persistencia

En el presente repositorio se muestra una serie de pasos para desplegar un servidor tomcat con el motor de Camunda en la modalidad *standalone* utilizando una base de datos para persistir metadata e información de instancias de procesos. Al finalizar el despliegue, La instancia queda con los siguientes componentes disponibles: 

* Motor de procesos BPMN 2.0, DMN 1.3 y CMMN 1.1.
* REST API.
* GUI.
    - [Admin](https://docs.camunda.org/manual/7.16/webapps/admin/)
    - [Cockpit](https://docs.camunda.org/manual/7.16/webapps/cockpit/)
    - [Tasklist](https://docs.camunda.org/manual/7.16/webapps/tasklist/)

## Instrucciones

Las indicaciones se ha separado de la siguiente manera:

- [AWS](https://github.com/sadalmelik828/camunda-persistent-deploy/tree/master/aws)
- [GCP](https://github.com/sadalmelik828/camunda-persistent-deploy/tree/master/gcp)
- Helm Chart

## Información adicional

Si desea conocer otros modos de despliegue de camunda, [aquí](https://docs.camunda.org/manual/7.16/introduction/architecture/) encontrará mas información o si desea profundizar en la plataforma camunda, puede dirigirse a la [documentación](https://docs.camunda.org/manual/7.16/).