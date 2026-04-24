# Practica 3 - Kubernetes local con k3d + k3s: desplegar el stack del project-skeleton

## Descripcion de la practica

En esta practica vas a instalar un entorno local de Kubernetes usando `k3d` y `k3s`, y vas a desplegar sobre ese cluster el mismo stack que ya resolviste previamente con Docker Compose en la practica anterior. No se trata de construir una aplicacion nueva, sino de reutilizar la misma arquitectura funcional y aprender a gestionarla con manifiestos de Kubernetes y `kubectl`.

Como material de trabajo dispondras de un archivo `.zip` que incluira:
- Este documento, que actua como guia paso a paso y enunciado de la practica.
- El directorio `project-skeleton`, que contiene la estructura base de manifiestos Kubernetes dentro de `project-skeleton/k8s/`.

La dinamica de trabajo consiste en avanzar leyendo en paralelo dos elementos:
- Este documento, donde se explica el objetivo de cada ejercicio, el orden recomendado y las validaciones que debes realizar.
- La estructura de manifiestos de `project-skeleton/k8s/`, donde encontraras las plantillas YAML que tendras que revisar, completar o ajustar en cada ejercicio.

Esta dependencia con la practica 2 es importante. En esta practica 3 se asume que ya entiendes el stack trabajado anteriormente con Docker Compose, incluyendo la base de datos PostgreSQL, el `ddl_job`, el `backend-java`, el `backend-python` y la `webui`. Kubernetes no sustituye ese trabajo previo, sino que se apoya en el para que aprendas a desplegar el mismo entorno con una tecnologia de orquestacion.

El objetivo tecnico de la practica es que montes un laboratorio local de Kubernetes con `k3d`, configures `kubectl` y despliegues por capas el stack del `project-skeleton`:
- La capa de datos en un namespace `database`, con PostgreSQL y almacenamiento persistente.
- Un `ConfigMap` con los DDL y un `Job` que inicializa el modelo de datos.
- La capa de aplicaciones en un namespace `apps`, con `backend-java`, `backend-python` y `webui`.
- La validacion del entorno usando `kubectl`, `Service` y `port-forward`.

Ademas de desplegar servicios, la practica esta diseñada para que entiendas los objetos fundamentales de Kubernetes como parte de una formacion activa. A traves de los manifiestos trabajaras conceptos como `Namespace`, `ConfigMap`, `PersistentVolume`, `PersistentVolumeClaim`, `Job`, `Deployment`, `StatefulSet` y `Service`, entendiendo para que sirve cada uno y cuando conviene usarlo.

## Por que usar k3s y k3d en un entorno de desarrollo

En esta practica no se usa un cluster Kubernetes gestionado en cloud ni una instalacion completa de produccion. Se usa `k3s` porque es una distribucion ligera de Kubernetes, y `k3d` porque permite ejecutar `k3s` dentro de contenedores Docker de forma muy rapida.

Esta eleccion tiene una intencion didactica clara:
- aprender Kubernetes real, pero sin asumir una infraestructura compleja,
- reducir el consumo de recursos del laboratorio,
- y facilitar que puedas crear, destruir y volver a crear el cluster cuantas veces haga falta.

### Ventajas de `k3s`

- Es una distribucion ligera de Kubernetes, pensada para laboratorios, edge y entornos con menos recursos.
- Mantiene los conceptos y objetos principales de Kubernetes, por lo que el aprendizaje es transferible a otros clusters.
- Reduce la complejidad de instalacion y operacion respecto a un cluster mas pesado.
- Es una muy buena opcion para practicar despliegues, observacion y depuracion sin tener que administrar una plataforma completa.

### Ventajas de `k3d`

- Permite levantar clusters `k3s` dentro de Docker, lo que simplifica mucho el laboratorio local.
- Hace posible crear y eliminar clusters rapidamente, algo muy util cuando se esta aprendiendo.
- Facilita mapear puertos y directorios del host hacia el cluster para simular casos reales de conectividad y persistencia.
- Evita depender de una cuenta cloud o de recursos externos para practicar Kubernetes.

### Ventajas conjuntas para desarrollo y formacion

- Puedes reproducir el entorno de forma rapida en tu propio equipo.
- Puedes romper y rehacer el cluster sin miedo, algo clave para aprender.
- Puedes validar manifiestos, namespaces, servicios, volumenes y jobs en un entorno cercano a Kubernetes real.
- Sirve como paso intermedio muy util antes de trabajar con clusters gestionados o con estrategias GitOps.

En resumen, `k3s` y `k3d` ofrecen un equilibrio muy bueno entre realismo tecnico, facilidad de instalacion y coste operativo, lo que los convierte en una opcion muy adecuada para preparar un entorno de desarrollo y formacion en Kubernetes.

## Objetivo

En esta practica vas a instalar un entorno local de Kubernetes usando `k3d`, conectarte con `kubectl` y desplegar en k3s el mismo stack trabajado previamente con Docker Compose.

La idea no es cambiar de aplicacion, sino cambiar de plataforma:
- antes definiste el entorno con `docker-compose.yml`,
- ahora vas a describirlo con manifiestos de Kubernetes.

Al finalizar, deberias ser capaz de:
- levantar un cluster local de k3s sobre Docker usando `k3d`,
- consultar el estado del cluster con `kubectl`,
- desplegar PostgreSQL, un Job de inicializacion, la web y los dos backends,
- y entender la diferencia entre `Deployment`, `StatefulSet`, `Service`, `ConfigMap`, `Job` y `PersistentVolumeClaim`.

## Importante

- Esta practica asume que la practica anterior con Docker Compose ya esta resuelta y entendida.
- El objetivo es aprender Kubernetes de forma guiada, no memorizar todos sus objetos.
- El directorio `project-skeleton/k8s/` contiene plantillas, no una solucion final cerrada.

## Estructura que vas a usar

Trabajaras sobre esta estructura:
- `project-skeleton/k8s/00-namespaces/`
- `project-skeleton/k8s/00-storage/`
- `project-skeleton/k8s/01-configmaps/`
- `project-skeleton/k8s/02-database/`
- `project-skeleton/k8s/03-jobs/`
- `project-skeleton/k8s/04-apps/`

Cada fichero YAML tiene comentarios indicando:
- en que ejercicio se usa,
- que bloque debes revisar,
- y que partes son placeholders o decisiones tecnicas.

## Ejercicio 1 - Instalar k3s local con k3d

### Objetivo

Crear un cluster local ligero de Kubernetes usando `k3d`, con soporte para almacenamiento persistente en el host.

### Que es k3d

- `k3s` es una distribucion ligera de Kubernetes.
- `k3d` es una herramienta que ejecuta `k3s` dentro de contenedores Docker.

Esto permite levantar un cluster local de forma rapida y sencilla, muy util para laboratorios.

### Preparar un directorio local para persistencia

```bash
mkdir -p devops-training-docker-kubernetes/project-skeleton/k8s/storage
```

### Crear el cluster

```bash
k3d cluster create devops-training \
  --servers 1 \
  --agents 1 \
  -p "8088:80@loadbalancer" \
  -p "8443:443@loadbalancer" \
  -v "$(pwd)/devops-training-docker-kubernetes/project-skeleton/k8s/storage:/tmp@all"
```

### Que debes entender

- `--servers 1`: crea un nodo de control.
- `--agents 1`: crea un nodo de trabajo adicional.
- `-p`: publica puertos del balanceador del cluster hacia tu maquina.
- `-v`: monta un directorio del host dentro de los nodos para que el almacenamiento local del cluster pueda persistir.
- En esta practica, el path `/tmp` del nodo queda respaldado por un directorio del host para poder trabajar despues con almacenamiento persistente local.

### Validacion

```bash
k3d cluster list
docker ps
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

### Por que se usa este enfoque

- Se usa `k3d` porque permite levantar y destruir clusters locales muy rapido.
- Se usa `k3s` porque mantiene los objetos reales de Kubernetes con menos complejidad operativa.
- Se monta un directorio del host porque despues necesitaremos persistencia para PostgreSQL y conviene que el laboratorio sea reproducible.

## Ejercicio 2 - Instalar y configurar kubectl

### Objetivo

Tener `kubectl` preparado para gestionar el cluster local de k3s.

### Instalacion

Sigue la documentacion oficial del sistema operativo que estes usando o instala `kubectl` desde tu gestor de paquetes.

### Validacion inicial del cluster

Ejecuta estos comandos y analiza la salida:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
kubectl get pods -A
kubectl get storageclass
kubectl api-resources | head -n 20
```

### Que debes entender

- El contexto actual indica a que cluster se conecta `kubectl`.
- `get nodes` muestra si el cluster esta listo.
- `get ns` enseña los namespaces disponibles.
- `get pods -A` permite ver servicios del sistema como Traefik, CoreDNS o local-path provisioner.
- `get storageclass` permite saber si el cluster ya tiene una clase de almacenamiento disponible.
- `api-resources` ayuda a ver que tipos de objetos puede gestionar el cluster.

### Por que se usa `kubectl`

- `kubectl` es la herramienta principal para aplicar manifiestos, consultar estado y depurar recursos.
- Aprender unos pocos comandos base es suficiente para empezar a trabajar con Kubernetes de forma realista.

## Ejercicio 3 - Preparar almacenamiento persistente y desplegar PostgreSQL en el namespace `database`

### Objetivo

Preparar un `PersistentVolume` local para laboratorio e instalar PostgreSQL como `Deployment` dentro de un namespace separado para la capa de datos.

### Por que se usan estos objetos

- `Namespace`: se usa para aislar la capa de datos del resto de aplicaciones y ordenar mejor el laboratorio.
- `PersistentVolume`: se usa para representar almacenamiento disponible dentro del cluster.
- `PersistentVolumeClaim`: se usa para solicitar ese almacenamiento desde la aplicacion sin acoplar directamente el Deployment al detalle fisico del volumen.
- `Deployment`: se usa para gestionar PostgreSQL como carga desplegable sencilla dentro del laboratorio.
- `Service`: se usa para dar un punto de red estable a PostgreSQL dentro del cluster.

### Manifiestos a revisar

- `project-skeleton/k8s/00-storage/local-persistent-volume.yaml`
- `project-skeleton/k8s/00-namespaces/database-namespace.yaml`
- `project-skeleton/k8s/02-database/postgres-pvc.yaml`
- `project-skeleton/k8s/02-database/postgres-deployment.yaml`
- `project-skeleton/k8s/02-database/postgres-service.yaml`

### Orden recomendado

```bash
kubectl apply -f project-skeleton/k8s/00-storage/local-persistent-volume.yaml
kubectl apply -f project-skeleton/k8s/00-namespaces/database-namespace.yaml
kubectl apply -f project-skeleton/k8s/02-database/postgres-pvc.yaml
kubectl apply -f project-skeleton/k8s/02-database/postgres-deployment.yaml
kubectl apply -f project-skeleton/k8s/02-database/postgres-service.yaml
```

### Que debes revisar en la plantilla

- El namespace correcto.
- El `PersistentVolume` local y su ruta `hostPath`.
- La imagen de PostgreSQL.
- Las variables `POSTGRES_*`.
- El `PersistentVolumeClaim` usado en el montaje.
- El puerto expuesto por el contenedor.
- El tipo de `Service`.

### Validacion

```bash
kubectl get pv
kubectl describe pv postgres-local-pv
kubectl get ns
kubectl get all -n database
kubectl get pvc -n database
kubectl describe pvc -n database postgres-data
kubectl get pods -n database -o wide
kubectl describe deploy -n database postgres
kubectl logs -n database deploy/postgres
kubectl get svc -n database
```

### Que deberias comprobar

- Que el `PersistentVolume` aparece en estado disponible o enlazado.
- Que el `PersistentVolumeClaim` queda asociado correctamente.
- Que el Pod de PostgreSQL entra en estado `Running`.
- Que el `Service` llamado `postgres` existe y expone el puerto esperado.

## Ejercicio 4 - Crear un Job para inicializar el modelo de datos

### Objetivo

Crear un `ConfigMap` que contenga los DDL y lanzar un `Job` que aplique ese modelo en PostgreSQL.

### Por que se usan estos objetos

- `ConfigMap`: se usa para almacenar configuracion o contenido de texto, en este caso el DDL, sin meterlo dentro de la imagen del contenedor.
- `Job`: se usa para tareas puntuales que deben ejecutarse, terminar y dejar evidencia del resultado.

Este modelo encaja bien con la inicializacion de base de datos:
- el DDL es configuracion versionable,
- y su aplicacion es una tarea finita, no un servicio permanente.

### Manifiestos a revisar

- `project-skeleton/k8s/01-configmaps/ddl-configmap.yaml`
- `project-skeleton/k8s/03-jobs/ddl-job.yaml`

### Que debes hacer

1. Copiar el contenido del DDL del `project-skeleton/ddl/init.sql` al ConfigMap.
2. Revisar el comando del Job.
3. Verificar el hostname del servicio PostgreSQL.
4. Aplicar ConfigMap y Job en el namespace `database`.

### Aplicacion

```bash
kubectl apply -f project-skeleton/k8s/01-configmaps/ddl-configmap.yaml
kubectl apply -f project-skeleton/k8s/03-jobs/ddl-job.yaml
```

### Validacion

```bash
kubectl get configmap -n database
kubectl describe configmap -n database ddl-sql
kubectl get jobs -n database
kubectl describe job -n database ddl-job
kubectl get pods -n database
kubectl logs -n database job/ddl-job
kubectl exec -n database deploy/postgres -- psql -U training -d trainingdb -c "\\dt"
kubectl exec -n database deploy/postgres -- psql -U training -d trainingdb -c "select * from favorite_colors;"
```

### Que deberias comprobar

- Que el `ConfigMap` contiene realmente el SQL esperado.
- Que el `Job` crea un Pod, se ejecuta y termina en estado correcto.
- Que las tablas existen y que los datos de ejemplo se han cargado.

## Ejercicio 5 - Desplegar webui, backend Java y backend Python

### Objetivo

Desplegar la capa de aplicaciones usando distintos tipos de workload para entender sus diferencias.

### Por que se usan estos objetos

- `Namespace`: se usa para separar la capa de aplicaciones de la capa de datos.
- `Service`: se usa para dar una direccion estable a cada backend y a la web dentro del cluster.
- `Deployment`: se usa para `backend-python` y `webui` porque son cargas que pueden recrearse sin identidad estable por replica.
- `StatefulSet`: se usa en `backend-java` con objetivo didactico para comparar su estructura y comportamiento con `Deployment`.
- `Headless Service`: se usa junto al `StatefulSet` para proporcionar identidad de red estable a sus replicas.

### Manifiestos a revisar

- `project-skeleton/k8s/00-namespaces/apps-namespace.yaml`
- `project-skeleton/k8s/04-apps/java-headless-service.yaml`
- `project-skeleton/k8s/04-apps/java-service.yaml`
- `project-skeleton/k8s/04-apps/java-statefulset.yaml`
- `project-skeleton/k8s/04-apps/python-deployment.yaml`
- `project-skeleton/k8s/04-apps/python-service.yaml`
- `project-skeleton/k8s/04-apps/webui-deployment.yaml`
- `project-skeleton/k8s/04-apps/webui-service.yaml`

### Orden recomendado

```bash
kubectl apply -f project-skeleton/k8s/00-namespaces/apps-namespace.yaml
kubectl apply -f project-skeleton/k8s/04-apps/java-headless-service.yaml
kubectl apply -f project-skeleton/k8s/04-apps/java-service.yaml
kubectl apply -f project-skeleton/k8s/04-apps/java-statefulset.yaml
kubectl apply -f project-skeleton/k8s/04-apps/python-deployment.yaml
kubectl apply -f project-skeleton/k8s/04-apps/python-service.yaml
kubectl apply -f project-skeleton/k8s/04-apps/webui-deployment.yaml
kubectl apply -f project-skeleton/k8s/04-apps/webui-service.yaml
```

### Que debes revisar

- La imagen de cada servicio.
- Los puertos internos y externos del servicio.
- Las variables de entorno del backend Python para conectar con PostgreSQL.
- El namespace correcto.
- El uso de `Deployment` o `StatefulSet` segun el caso.
- Los `Service` y sus selectores para que apunten a los Pods correctos.

### Diferencias entre `StatefulSet` y `Deployment`

#### `Deployment`

Es el objeto mas habitual para aplicaciones stateless:
- APIs sin identidad propia por replica.
- frontends web,
- servicios que pueden destruirse y recrearse sin necesidad de nombre estable por replica.

En esta practica:
- `backend-python` se modela como `Deployment` porque, aunque use una base de datos, el contenedor en si sigue siendo stateless.
- `webui` tambien se modela como `Deployment`.

#### `StatefulSet`

Se usa cuando necesitas:
- identidad estable por replica,
- nombres de red predecibles,
- o almacenamiento asociado a cada instancia.

En esta practica:
- `backend-java` se modela como `StatefulSet` con un fin didactico, para comparar su estructura con un `Deployment`.
- Nota tecnica importante: en un caso real, una API Java stateless como esta normalmente se desplegaria como `Deployment`, no como `StatefulSet`.

### Validacion

```bash
kubectl get ns
kubectl get all -n apps
kubectl get pods -n apps -o wide
kubectl get deploy -n apps
kubectl get statefulset -n apps
kubectl get svc -n apps
kubectl describe statefulset -n apps backend-java
kubectl describe deploy -n apps backend-python
kubectl describe deploy -n apps webui
kubectl logs -n apps statefulset/backend-java
kubectl logs -n apps deploy/backend-python
kubectl logs -n apps deploy/webui
```

### Que deberias comprobar

- Que todos los Pods de la capa `apps` entran en estado `Running`.
- Que los `Service` existen y seleccionan los Pods correctos.
- Que `backend-python` resuelve correctamente PostgreSQL en el namespace `database`.
- Que `webui` arranca correctamente y queda lista para exponerse por `port-forward`.

### Acceso a los servicios

Como punto de partida sencillo, usa `port-forward`:

```bash
kubectl port-forward -n apps svc/backend-java 8084:8080
kubectl port-forward -n apps svc/backend-python 5001:5000
kubectl port-forward -n apps svc/webui 8080:80
```

Validaciones:

```bash
curl http://localhost:8084/hello
curl http://localhost:5001/
curl http://localhost:8080
```

### Validacion adicional recomendada

Mientras mantienes abiertos los `port-forward`, puedes seguir observando el cluster:

```bash
kubectl get pods -n apps -w
kubectl get endpoints -n apps
kubectl describe svc -n apps backend-java
kubectl describe svc -n apps backend-python
kubectl describe svc -n apps webui
```

## Entrega de la practica

La entrega debe incluir:
- este documento de la practica,
- un `.zip` con el directorio `project-skeleton/k8s/` actualizado,
- los manifiestos completados,
- y capturas de pantalla o salidas de terminal con evidencias.

Evidencias minimas recomendadas:
- salida de `kubectl get nodes`,
- salida de `kubectl get storageclass`,
- recursos del namespace `database`,
- recursos del namespace `apps`,
- logs del `ddl-job`,
- consulta SQL contra PostgreSQL,
- respuesta del backend Java,
- respuesta del backend Python,
- y carga de la web.

## Resultado esperado

Al completar la practica, deberias haber entendido:
- como pasar de Docker Compose a manifiestos de Kubernetes,
- como aislar capas usando namespaces,
- como usar `ConfigMap`, `Job`, `Deployment`, `StatefulSet`, `Service` y `PersistentVolumeClaim`,
- y como validar un stack de microservicios basico sobre un cluster local de k3s.
