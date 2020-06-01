# DESPLIEGUE DE SISTEMA DE MONITORIZACIÓN PARA AEDUCAR

## Fuente para desplegar servicio en producción

Fuente: https://github.com/vegasbrianc/prometheus

[README orginal aqui](README_orig.md)

Cambios realizados:

* Se bloquean los puertos a los servicios salvo grafana
* Se añade soporte 80/443 para acceso a Grafana en producción (desarrollador original usa opción traefik)
* Activada la notificación a Slack por defecto. Parametrizar token para cada instalación
* Se actualiza el contenedor de Cadvisor
* Se añade la opción de monitorizar la BD asociada al proyecto

### Sistema Base con dashboard para docker

Pasos para desplegar el servicio de monitorización:

1. Usuario **administrador**

Contraseña de usuario de administración en grafana -> grafana/config.monitoring

2. Activar **docker swarm** en el "manager". `docker swarm init --advertise-addr interfaz_ip:2377 --listen-addr intefaz_ip:2377 

   1. El parámetro advertise es pod donde harán los anuncios

   2. El puerto de escucha (para unir workers - 2377 de la interfaz pública) es interesante también ocultarlo a la red pública. Para ello, usar --listen-addr interfaz:2377 (se puede usar interfaz física o dirección IP)

   3. Ver estado de swarm: `docker info `

   4. Ver estado del nodo: `docker node ls`

3. Unir los **"workers"** de swarm. Para ello puede ser bueno

   * Unir el equipo. El token se facilita al iniciar el docker en el manager: `docker swarm join --token XXXXXX IP:2377`
   * Si no recordamos el token, podemos hacer en el manager: ` docker swarm join-token worker`
   * Ver estado de swarm (ojo, solo en el manager): `docker node ls`

4. Configurar las **alertas** según aparece en la descripción. Viene parametrizado **para slack**. Para configurarlo

   1. En alertmanager/config.yml configurar el token necesario según se crea en slack, username que se usará y el canal donde se emitarán los mensajes
   2. Si no se quieren alertas a slack, dejarlo sin comentar.

5. **Arranque del sistema** (se cargarán los servicios necesarios en los workers para recolectar información)
En el caso de arrancar en producción, definir en docker-stack.override_prod.yml VIRTUAL_HOST,LETSENCRYPT_HOST y LETSENCRYPT_EMAIL acorde a tu infraestructura.
(cambiar el enlace simbólico si interesa prod o dev)
```bash
  docker stack deploy -c docker-stack.yml -c docker-stack.override.yml monitor
```

6. **Chequeo** del estado del despliegue

* Despliegue
```bash
  docker stack ps monitor
```
* Ver servicio en concreto
```bash
  docker service ACTION SERVICIO
```
* Publicar un puerto: docker service update --publish-add 8080:8080 monitor_cadvisor 

7. **Logging** de todos los servicios. No implementado en docker stack. Para ello usar el script:

```bash
  stacklogs [service]
```

8. **Parar y borrar** el depliegue

```bash
  docker stack rm monitor
```

9. **Añadir los dashboards** de docker necesarios vía interfaz web. Por defecto viene uno instalado. El resto los podemos encontrar en dashboards. Interesante el dashboard by node

### Monitorizar MaríaDB

Para monitorizar María DB debemos usar algún otro dashboard y/o usar una fuente de datos diferente a Prometheus.
Vemos un proyecto interesante en:
https://github.com/meob/my2Collector

1. Crear la BD y usuario/password para recolectar los datos.

   1. Modificar la password para el usuario my2 en my2Collector/my2.sql

   2. Crear la BD, usuario y "llenado" de datos. En la máquina con MariaDB: 

```bash
  mysql --user=root -p < my2.sql
```

2. En Grafana, añadir un DataSource:
    * Tipo Mysql
    * Host: IP_interna:3306 || user: my2 || Password: password || Name: A gusto

3. Importar Dashboard de Mysql, asociado al Datasource anterior

## TO-DO

* Secrets (.env no es válido en docker swarm)