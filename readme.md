# Shell toolkit GCP + Kubernetes + Logs

Conjunto de funciones y alias para `zsh` orientado a trabajo diario con:

- GCP / GKE (proyectos y clusters).
- Kubernetes (pods, logs, exec, describe, port-forward, trazas).
- Logs locales (`lnav`).

El objetivo es:

- Saber siempre en qué **cluster** y **namespace** estás.
- Llegar rápido a **pods**, **logs**, **configuración** y **actuator**.
- Subir/bajar el **nivel de trazas** sin pelearte con YAML.

---

## 0. Chuleta rápida (necesidad → alias)

| Necesidad rápida                                                 | Alias / comando                          | Ejemplo                                                      |
|------------------------------------------------------------------|------------------------------------------|--------------------------------------------------------------|
| Saber en qué cluster y namespace estoy y ver pods                | `kpods`                                   | `kpods`                                                      |
| Ver pods con cluster/ns/nodo en tiempo real                      | `kpods-env`                               | `kpods-env`                                                  |
| Ver todos los namespaces                                         | `knamespaces`                             | `knamespaces`                                                |
| Cambiar el namespace actual                                      | `knamespace-s <ns>`                       | `knamespace-s backend`                                       |
| Ver logs de un pod una vez                                       | `klogs <pattern> [ns]`                    | `klogs link-issuer backend`                                  |
| Ver logs de un pod en streaming                                  | `klogsf <pattern> [ns]`                   | `klogsf api-gateway backend`                                 |
| Guardar logs de un pod a fichero                                 | `klog-save <pattern> [ns]`                | `klog-save link-issuer backend`                              |
| Ir al directorio de logs                                         | `ksaved-logs`                             | `ksaved-logs`                                                |
| Abrir el último log con lnav                                     | `last-log`                                | `last-log`                                                   |
| Entrar en un pod con shell                                       | `kit <pattern> [ns]`                      | `kit link-issuer backend`                                    |
| Ver la descripción completa de un pod                            | `kdesc <pattern> [ns]`                    | `kdesc link-issuer backend`                                  |
| Hacer port-forward a un pod (8080→8080 por defecto)              | `kpf <pattern> [lport] [rport] [ns]`      | `kpf link-issuer 9000 8080 backend`                          |
| Ver todos los deployments                                        | `kdeployments`                            | `kdeployments`                                               |
| Ver todos los services                                           | `kservices`                               | `kservices`                                                  |
| Ver el YAML de un deployment por patrón                          | `kdeployment-y <pattern> [ns]`            | `kdeployment-y api-gateway backend`                          |
| Subir nivel de logs (persistente) de un deployment               | `ktrace <deploy-pattern> [level] [ns]`    | `ktrace link-issuer DEBUG backend`                           |
| Cambiar nivel de logs temporalmente en un pod vía Actuator       | `ktrace-l <pod-pattern> [level] [ns]`     | `ktrace-l link-issuer DEBUG backend`                         |
| Cambiar a cluster Zertiban neocore TEST                          | `zb-test`                                 | `zb-test`                                                    |
| Cambiar a cluster Zertiban neocore UAT                           | `zb-uat`                                  | `zb-uat`                                                     |
| Cambiar a cluster Zertipayments neocore TEST                     | `zp-test`                                 | `zp-test`                                                    |
| Cambiar a cluster Zertipayments neocore UAT                      | `zp-uat`                                  | `zp-uat`                                                     |
| Ver proyecto GCP actual                                          | `gcp-project`                             | `gcp-project`                                                |
| Listar proyectos GCP                                             | `gcp-projects`                            | `gcp-projects`                                               |

- Ver dónde estoy → `kcluster`, `knamespace`, `kpods`
- Ver logs → `klogs`, `klogsf`, `klog-save`, `ksaved-logs`, `last-log`
- Entrar a un pod → `kit`
  - Una vez dentro del pod:
    - `ps aux` → ver qué procesos se están ejecutando (ej. `java -jar /app.jar`)
    - `env | sort` → ver todas las variables de entorno efectivas
- Ver detalles de un pod → `kdesc`
- Port-forward → `kpf`
- Ver deployments / services → `kdeployments`, `kservices`, `kdeployment-y`
- Cambiar trazas → `ktrace`, `ktrace-l`
- Cambiar de cluster → `zb-test`, `zb-uat`, `zp-test`, `zp-uat`, `*-legacy-*`


## 1. Instalación y estructura

### 1.1. Layout recomendado

```text
~/.config/shell/
  aliases/
    gcp.aliases
    k8s.aliases
    git.aliases
    log.aliases
  functions/
    gcp.functions
    k8s.functions
    logs.functions
```

### 1.2. Carga desde .zshrc

En el fichero .zshrc se debe añadir

```shell
setopt aliases

SHELL_CONF_DIR="$HOME/.config/shell"

# Cargar funciones
if [ -d "$SHELL_CONF_DIR/functions" ]; then
  for f in "$SHELL_CONF_DIR"/functions/*(.N); do
    . "$f"
  done
fi

# Cargar alias
if [ -d "$SHELL_CONF_DIR/aliases" ]; then
  for f in "$SHELL_CONF_DIR"/aliases/*(.N); do
    . "$f"
  done
fi
```

### 1.3. Instalación básica

```shell
mkdir -p ~/.config/shell/functions
mkdir -p ~/.config/shell/aliases

# Copias tus ficheros:
#   ~/.config/shell/functions/gcp.functions
#   ~/.config/shell/functions/k8s.functions
#   ~/.config/shell/functions/logs.functions
#   ~/.config/shell/aliases/gcp.aliases
#   ~/.config/shell/aliases/k8s.aliases
#   ~/.config/shell/aliases/git.aliases
#   ~/.config/shell/aliases/log.aliases

source ~/.zshrc
```
## 2. Alias Kubernetes

Alias para trabajar con Kubernetes agrupados por funcionalidad.  
Las funciones internas (`klog`, `klogf`, etc.) no se documentan aquí, sólo los alias que usas en el día a día.

### 2.1. Índice rápido de alias Kubernetes

- **Contexto / recursos**
  - `kclusters` – lista clusters GKE del proyecto de develop
  - `kcluster` – muestra el cluster actual de `kubectl`
  - `knamespaces` – lista namespaces
  - `knamespace` – muestra el namespace actual
  - `knamespace-s` – cambia el namespace actual
  - `kpods` – lista pods con info ampliada (cluster/namespace)
  - `kpods-env` – lista pods con columnas CLUSTER/NAMESPACE/NAME/STATUS/IP/NODE en modo watch
  - `kdeployments` – lista deployments
  - `kservices` – lista services
  - `kdeployment-y` – muestra el YAML de un deployment

- **Logs**
  - `klogs`, `klogsf` – logs de pods (una vez / en streaming)
  - `kslogs`, `kslogsf` – logs “por service” (mismo patrón de resolución)
  - `klog-save` – guardar logs a fichero
  - `ksaved-logs` – ir al directorio de logs
  - `last-log` – abrir último log con `lnav`

- **Pods / debugging**
  - `kit` – entrar en un pod con `sh`
  - `kdesc` – `kubectl describe pod`
  - `kpf` – `port-forward` a un pod

- **Trazas**
  - `ktrace` – cambio persistente de nivel de trazas (Deployment + rollout)
  - `ktrace-l` – cambio temporal de nivel de trazas en un pod vía Actuator

---

### 2.2. Contexto de cluster y namespace

```shell
kclusters
```

- Lista los clusters GKE del proyecto `zertiban-neocore-develop`.

- Equivalente a:

  ```shell
  gcloud container clusters list --project zertiban-neocore-develop
  ```

```shell
kcluster
```

- Muestra el **cluster actual** configurado en `kubectl`.

- Equivalente a:

  ```shell
  kubectl config view --minify -o jsonpath='{.contexts[0].context.cluster}'
  ```

```shell
knamespaces
```

- Lista todos los namespaces del cluster actual.

- Equivalente a:

  ```shell
  kubectl get namespaces
  ```

```shell
knamespace
```

- Muestra el **namespace actual** del contexto de `kubectl`.

- Equivalente a:

  ```shell
  kubectl config view --minify -o jsonpath='{.contexts[0].context.namespace}'
  ```

```shell
knamespace-s <namespace>
```

- Cambia el namespace por defecto del contexto actual.

- Equivalente a:

  ```shell
  kubectl config set-context --current --namespace <namespace>
  ```

```shell
kpods
```

- Muestra:
  - El nombre simplificado del **cluster** actual.
  - El **namespace** actual (o `default` si no está definido).
  - El listado de pods (`kubectl get pods -o wide`) con información de IP, nodo, etc.

- Equivalente aproximado a:

  ```shell
  kubectl get pods -o wide
  ```

```shell
kpods-env
```

- Ejecuta `kubectl get pods` con un conjunto de columnas personalizado:
  - `CLUSTER`, `NAMESPACE`, `NAME`, `STATUS`, `IP`, `NODE`.
- Se ejecuta con `-w`, por lo que queda en modo *watch* mostrando cambios en tiempo real hasta que pulses `Ctrl+C`.

- Equivalente aproximado a:

  ```shell
  kubectl get pods \
    -o custom-columns="CLUSTER:\"$CLUSTER\",NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP,NODE:.spec.nodeName" \
    -w
  ```
  
Ejemplo rápido de chequeo de entorno:

```shell
kcluster
knamespace
kpods
kpods-env
```

---

### 2.3. Listado de recursos básicos

```shell
kdeployments
```

- Lista todos los Deployments del namespace actual.

- Equivalente a:

  ```shell
  kubectl get deployments
  ```

```shell
kservices
```

- Lista todos los Services del namespace actual.

- Equivalente a:

  ```shell
  kubectl get services
  ```

```shell
kdeployment-y <pattern> [namespace]
```

- Muestra el YAML del Deployment cuyo nombre contenga `<pattern>`.
- Si se indica `namespace`, busca sólo en ese namespace; si no, utiliza el namespace actual.
- Valida que haya exactamente un Deployment que cumpla el patrón.

- Equivalente aproximado a:

  ```shell
  kubectl get deploy <nombre-completo> -o yaml [ -n <namespace> ]
  ```

Ejemplos:

```shell
kdeployments
kservices
kdeployment-y api-gateway backend
```

---

### 2.4. Logs de pods (`klogs`, `klogsf`, `kslogs`, `kslogsf`)

Alias principales:

```shell
klogs  <pod-pattern> [namespace]
klogsf <pod-pattern> [namespace]
```

- `klogs`:
  - Muestra una vez los logs del pod (equivalente a `kubectl logs`).
- `klogsf`:
  - Sigue los logs en tiempo real (equivalente a `kubectl logs -f`).

- Equivalente a:

  ```shell
  kubectl logs [ -n <namespace> ] <pod>
  kubectl logs -f [ -n <namespace> ] <pod>
  ```

Alias equivalentes orientados a “service”:

```shell
kslogs  <pod-pattern> [namespace]
kslogsf <pod-pattern> [namespace]
```

- Semánticamente los puedes usar cuando piensas en servicios en lugar de pods, pero resuelven por el mismo patrón de nombre de pod.

- Internamente usan el mismo comando base:

  ```shell
  kubectl logs [ -n <namespace> ] <pod>
  kubectl logs -f [ -n <namespace> ] <pod>
  ```

Ejemplos:

```shell
klogs link-issuer
klogs link-issuer backend

klogsf link-issuer
klogsf link-issuer backend
```

Se pueden combinar con `lnav` en streaming:

```shell
klogsf link-issuer | lnav -
```

---

### 2.5. Guardado de logs y exploración con lnav (`klog-save`, `ksaved-logs`, `last-log`)

```shell
klog-save <pod-pattern> [namespace]
```

- Resuelve el pod por patrón.
- Valida que haya un único pod que cumpla el patrón.
- Guarda los logs en un fichero local dentro de `~/logs/k8s`.
- El nombre del fichero incluye:
  - Nombre del pod.
  - Namespace (o `current` si no se pasó).
  - Timestamp.

- Equivalente aproximado a:

  ```shell
  kubectl logs [ -n <namespace> ] <pod> > ~/logs/k8s/<pod>-<namespace>-<timestamp>.log
  ```

Ejemplo:

```shell
klog-save link-issuer backend
# Genera algo como:
#   ~/logs/k8s/zertiban-link-issuer-service-backend-20251123-180000.log
```

```shell
ksaved-logs
```

- Cambia el directorio actual a `~/logs/k8s`, donde se guardan los logs capturados con `klog-save`.

- Equivalente a:

  ```shell
  cd ~/logs/k8s
  ```

```shell
last-log
```

- Abre con `lnav` el último fichero `.log` del directorio de logs (por defecto `~/logs/k8s`).
- Permite navegar, filtrar, buscar y ver histogramas sobre los logs.

Flujo típico:

```shell
klog-save link-issuer backend   # 1. Guardas los logs
ksaved-logs                     # 2. Vas al directorio de logs
last-log                        # 3. Abres el último log con lnav
```

---

### 2.6. Shell y descripción de pods (`kit`, `kdesc`)

```shell
kit <pod-pattern> [namespace]
```

- Entra en una shell `sh` dentro del primer pod cuyo nombre contenga `<pod-pattern>`.
- Si se indica `namespace`, busca en ese namespace; si no, usa el namespace actual.

- Equivalente aproximado a:

  ```shell
  kubectl exec -it [ -n <namespace> ] <pod> -- sh
  ```

Ejemplos:

```shell
kit link-issuer
kit link-issuer backend
```

Desde dentro del pod:

- `ps aux` → ver el proceso que se está ejecutando (por ejemplo `java -jar /app.jar`).
- `env | sort` → ver todas las variables de entorno efectivas.

```shell
kdesc <pod-pattern> [namespace]
```

- Muestra la salida de `kubectl describe pod` para el primer pod que cumpla el patrón.
- Información útil:
  - Variables de entorno (y su origen en ConfigMaps/Secrets).
  - Probes (liveness/readiness).
  - Imagen, args, recursos, tolerations, etc.
  - Eventos recientes del pod.

- Equivalente a:

  ```shell
  kubectl describe pod [ -n <namespace> ] <pod>
  ```

Ejemplos:

```shell
kdesc link-issuer
kdesc link-issuer backend
```

Además, hay algunos comandos manuales útiles cuando quieres inspeccionar el `jar`:

```shell
kubectl cp <namespace>/<pod>:/app.jar ./app.jar
jar tf app.jar | grep application
```

---

### 2.7. Port-forward a pods (`kpf`)

```shell
kpf <pod-pattern> [local-port=8080] [remote-port=8080] [namespace]
```

- Abre un `kubectl port-forward` desde `localhost:<local-port>` hasta `pod/<pod-name>:<remote-port>`.
- Resuelve el pod por patrón de nombre.
- Si no se especifican puertos, usa `8080` tanto local como remoto.
- Si no se indica `namespace`, utiliza el namespace actual.

- Equivalente a:

  ```shell
  kubectl port-forward [ -n <namespace> ] pod/<pod> <local-port>:<remote-port>
  ```

Ejemplos:

```shell
kpf link-issuer
# localhost:8080 -> pod/link-issuer-...:8080 (namespace actual)

kpf link-issuer 9000
# localhost:9000 -> pod/link-issuer-...:8080

kpf link-issuer 9000 8081 backend
# localhost:9000 -> pod/link-issuer-...:8081 en namespace backend
```

Útil para:

- Probar endpoints internos (`/actuator`, APIs, health checks) desde tu máquina local.
- Depurar sin tocar Ingress, NodePort ni exponer nada adicional en el cluster.

---

### 2.8. Cambio de nivel de trazas (`ktrace`, `ktrace-l`)

Hay dos alias principales para ajustar el nivel de logs de forma controlada.

```shell
ktrace <deploy-pattern> [level=DEBUG] [namespace]
```

- Cambia la variable `LOGGING_LEVEL_ROOT` en el Deployment cuyo nombre contenga `<deploy-pattern>`.
- Lanza un `kubectl rollout restart` del Deployment afectado.
- El cambio es **persistente**: afecta a todos los pods nuevos de ese Deployment.

- Equivalente aproximado a:

  ```shell
  kubectl set env deployment/<deploy> LOGGING_LEVEL_ROOT=<LEVEL> [ -n <namespace> ]
  kubectl rollout restart deployment/<deploy> [ -n <namespace> ]
  ```

Ejemplos:

```shell
ktrace link-issuer
# LOGGING_LEVEL_ROOT=DEBUG en el Deployment que contenga "link-issuer" (ns actual)

ktrace link-issuer INFO backend
# LOGGING_LEVEL_ROOT=INFO en el namespace backend
```

```shell
ktrace-l <pod-pattern> [level=DEBUG] [namespace]
```

- Ajusta el nivel de logs de un **pod concreto** usando Spring Boot Actuator.
- Abre un `port-forward` temporal hacia el pod.
- Llama al endpoint `/actuator/loggers/ROOT` con el nivel indicado.
- Cierra el `port-forward` al terminar.
- El cambio es **temporal** y sólo afecta a ese pod mientras siga vivo.

- Equivalente aproximado a:

  ```shell
  kubectl port-forward [ -n <namespace> ] pod/<pod> 18080:8080
  curl -X POST "http://127.0.0.1:18080/actuator/loggers/ROOT" \
    -H "Content-Type: application/json" \
    -d '{"configuredLevel":"<LEVEL>"}'
  ```

Ejemplos:

```shell
ktrace-l link-issuer
# Sube a DEBUG el nivel de logs del pod que contenga "link-issuer" (ns actual)

ktrace-l link-issuer INFO backend
# Cambia a INFO en un pod del namespace backend
```

Resumen:

- `ktrace`  → cambio persistente a nivel de Deployment (requiere rollout, afecta a todos los pods).
- `ktrace-l` → cambio puntual en un pod vía Actuator (sin tocar el Deployment).

---

## 3. Alias GCP / Clusters Zertiban

Alias relacionados con GCP y con el cambio rápido entre clusters de Zertiban y Zertipayments.

### 3.1. Proyecto actual y listado de proyectos

```shell
gcp-project
```

- Muestra el **ID de proyecto** actualmente configurado en `gcloud`.

```shell
gcp-projects
```

- Lista los proyectos disponibles en la cuenta, mostrando:
  - `projectId`
  - `name`
  - `projectNumber`

---

### 3.2. Cambio rápido entre clusters Zertiban / Zertipayments

Alias que combinan:

- `gcloud config set project <proyecto>`
- `gcloud container clusters get-credentials <cluster> --zone ... --project ...`
- `kubectl config set-context --current --namespace=backend`

Entornos Zertiban neocore:

```shell
zb-test
zb-uat
```

- `zb-test`:
  - Proyecto: `zertiban-neocore-develop`
  - Cluster: `zertiban-neocore-test` (`europe-southwest1-a`)
  - Namespace por defecto: `backend`.

- `zb-uat`:
  - Proyecto: `zertiban-neocore-develop`
  - Cluster: `zertiban-neocore-uat` (`europe-southwest1-a`)
  - Namespace por defecto: `backend`.

Entornos Zertipayments neocore:

```shell
zp-test
zp-uat
```

- `zp-test`:
  - Proyecto: `zertipayments-neocore-develop`
  - Cluster: `zertipayments-neocore-test` (`europe-southwest1-a`)
  - Namespace por defecto: `backend`.

- `zp-uat`:
  - Proyecto: `zertipayments-neocore-develop`
  - Cluster: `zertipayments-neocore-uat` (`europe-southwest1-a`)
  - Namespace por defecto: `backend`.

Entornos Zertiban legacy:

```shell
zb-legacy-production
zb-legacy-test
```

- `zb-legacy-production`:
  - Proyecto: `zertiban-production`
  - Cluster: `zertiban-production` (`europe-west1`)
  - Namespace por defecto: `backend`.

- `zb-legacy-test`:
  - Proyecto: `zertiban-development`
  - Cluster: `zertiban-development` (`europe-west1`)
  - Namespace por defecto: `backend`.

Entornos Zertipayments legacy:

```shell
zp-legacy-production
zp-legacy-test
```

- `zp-legacy-production`:
  - Proyecto: `zertipayments-production`
  - Cluster: `zertipayments-production` (`europe-southwest1`)
  - Namespace por defecto: `backend`.

- `zp-legacy-test`:
  - Proyecto: `zertipayments-development`
  - Cluster: `zertipayments-development` (`europe-southwest1`)
  - Namespace por defecto: `backend`.

Uso típico:

```shell
zb-test
# Trabajar en Zertiban TEST (neocore), ns backend

zp-test
# Trabajar en Zertipayments TEST (neocore), ns backend

zb-uat
# Trabajar en Zertiban UAT (neocore), ns backend

zp-uat
# Trabajar en Zertipayments UAT (neocore), ns backend
```

Una vez elegido el entorno con estos alias, todos los alias de Kubernetes (`kpods`, `klogs`, `kit`, etc.) operan sobre ese cluster/namespace.

---

## 4. Alias de logs locales

```shell
last-log
```

- Abre con `lnav` el último fichero `.log` encontrado en el directorio de logs configurado (por defecto `~/logs/k8s`).
- Es el complemento natural de:
  - `klog-save` → guardar logs de un pod.
  - `ksaved-logs` → ir al directorio donde se guardan.

Flujo completo típico de trabajo con logs:

```shell
# 1. Guardas los logs de un pod
klog-save link-issuer backend

# 2. Vas al directorio de logs
ksaved-logs

# 3. Abres el último log con lnav
last-log
```
# shell
# shell
# shell
