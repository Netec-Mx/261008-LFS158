# Práctica: Creación de un clúster local

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 25 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se creará un clúster local de Kubernetes de un nodo utilizando Minikube y el controlador Docker. Se comprobará la disponibilidad de Docker Engine, Minikube y `kubectl`, se verificará que el contexto activo corresponda al perfil `intro-k8s` y se validará el estado operativo del nodo y de los componentes esenciales del sistema.

El clúster creado será la base obligatoria para los laboratorios 08-00-01, 09-00-01 y 10-00-01. No se desplegarán aplicaciones del estudiante en el espacio de nombres `kube-system`.

## Objetivos de Aprendizaje

- [ ] Verificar que Docker Engine 28.0.4, Minikube 1.35.0 y `kubectl` 1.33.0 estén instalados y sean accesibles.
- [ ] Crear el perfil de Minikube `intro-k8s` con un nodo, 2 CPU, 4 GB de memoria, controlador Docker y Kubernetes v1.33.0.
- [ ] Confirmar que el contexto activo de `kubectl` sea `intro-k8s`.
- [ ] Validar que el nodo esté en estado `Ready` y que los Pods esenciales de `kube-system` estén operativos.
- [ ] Guardar una línea base verificable del clúster en `artifacts/cluster-baseline.txt`.

## Prerrequisitos

### Conocimientos requeridos

- Comprensión básica de la diferencia entre un clúster local y un clúster de producción.
- Familiaridad inicial con terminal Bash, directorios, permisos y comandos básicos de Linux.
- Conocimiento introductorio de Kubernetes: nodo, Pod, espacio de nombres, contexto y API de Kubernetes.
- Comprensión de que Minikube utiliza un perfil para identificar una instancia local de clúster.
- Conocimiento de que el controlador Docker ejecuta el nodo de Minikube como un contenedor administrado por Docker Engine.

### Acceso requerido

- Usuario Linux con acceso a una terminal Bash.
- Usuario autorizado para ejecutar `docker ps` sin errores de permisos.
- Conexión a Internet para descargar la imagen del nodo y las imágenes de componentes de Kubernetes durante el primer inicio.
- Al menos 2 CPU disponibles y 4 GB de memoria disponibles para Minikube.
- Espacio libre suficiente para imágenes, caché y datos de Minikube.
- Laboratorio 06-00-01 completado, incluida la decisión de utilizar el controlador Docker.

> **Importante:** Esta práctica usa instrucciones de terminal, no herramientas de inteligencia artificial. Una **instrucción** es una acción concreta que debe ejecutar la persona estudiante. No se utiliza ningún **prompt**, ni existe un **mensaje de sistema** o un agente persistente que modifique el estado del clúster. No se requieren Microsoft 365 Copilot, Copilot Chat, Designer, Planner ni licencias o configuraciones asociadas.

## Entorno de Laboratorio

### Versiones de referencia obligatorias

| Componente | Edición y arquitectura | Versión exacta | Fuente oficial |
|---|---|---:|---|
| Sistema operativo de referencia | Ubuntu 22.04.5 LTS amd64 | 22.04.5 | https://releases.ubuntu.com/22.04/ |
| Motor de contenedores | Docker Engine Community Edition para Linux amd64 | 28.0.4 | https://docs.docker.com/engine/release-notes/28/#2804 |
| Clúster local | Minikube para Linux amd64 | 1.35.0 | https://github.com/kubernetes/minikube/releases/tag/v1.35.0 |
| Cliente de Kubernetes | kubectl para Linux amd64 | 1.33.0 | https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl |
| Plataforma del clúster | Kubernetes para Linux amd64 | v1.33.0 | https://github.com/kubernetes/kubernetes/releases/tag/v1.33.0 |
| Controlador de Minikube | Docker Engine Community Edition para Linux amd64 | 28.0.4 | https://minikube.sigs.k8s.io/docs/drivers/docker/ |

### Configuración objetivo del clúster

| Parámetro | Valor obligatorio |
|---|---|
| Directorio de trabajo | `/home/$USER/k8s-intro` |
| Perfil de Minikube | `intro-k8s` |
| Contexto esperado de `kubectl` | `intro-k8s` |
| Número de nodos | 1 |
| CPU asignadas | 2 |
| Memoria asignada | 4096 MB |
| Controlador | `docker` |
| Versión de Kubernetes | `v1.33.0` |
| Archivo de evidencia | `~/k8s-intro/artifacts/cluster-baseline.txt` |

### Preparación del directorio de trabajo

Ejecute los siguientes comandos al inicio de la práctica:

```bash
mkdir -p "$HOME/k8s-intro"/{analisis,manifests,artifacts}
cd "$HOME/k8s-intro"
pwd
```

La salida de `pwd` debe terminar en:

```text
/home/<usuario>/k8s-intro
```

## Instrucciones Paso a Paso

### Paso 1: Preparar el espacio de trabajo y comprobar recursos locales

**Objetivo:** Confirmar que se utilizará el directorio de trabajo del curso, que Docker Engine responde para el usuario actual y que el equipo dispone de recursos mínimos para iniciar Minikube.

1. Cree, si todavía no existen, los directorios obligatorios del curso y el directorio de evidencias:

   ```bash
   mkdir -p "$HOME/k8s-intro"/{analisis,manifests,artifacts}
   cd "$HOME/k8s-intro"
   ```

2. Confirme el directorio actual:

   ```bash
   pwd
   ```

3. Compruebe que Docker responde sin utilizar `sudo`:

   ```bash
   docker ps
   ```

4. Compruebe que el daemon de Docker está disponible:

   ```bash
   docker info >/dev/null && echo "Docker disponible"
   ```

5. Consulte las CPU lógicas disponibles en el equipo:

   ```bash
   nproc
   ```

6. Consulte la memoria disponible en MiB:

   ```bash
   free -m
   ```

**Salida esperada:**

- El directorio actual corresponde a `/home/<usuario>/k8s-intro`.
- `docker ps` muestra encabezados de columnas o contenedores existentes, sin errores de permisos.
- Se muestra el mensaje `Docker disponible`.
- El sistema informa al menos 2 CPU lógicas.
- Existe capacidad de memoria suficiente para reservar 4096 MB para Minikube sin comprometer el sistema anfitrión.

**Verificación:**

Ejecute el siguiente comando. Debe devolver una ruta bajo el directorio personal del usuario y no debe producir errores:

```bash
test -d "$HOME/k8s-intro/artifacts" && echo "Directorio de evidencias preparado"
```

Si `docker ps` devuelve un error relacionado con permisos o con un daemon no disponible, no continúe con la creación del clúster. Consulte la sección de solución de problemas.

---

### Paso 2: Verificar versiones de Docker, Minikube y kubectl

**Objetivo:** Confirmar que las herramientas requeridas están disponibles y corresponden exactamente a las versiones definidas para este laboratorio.

1. Compruebe la versión del cliente y del servidor de Docker:

   ```bash
   docker version --format 'Cliente={{.Client.Version}} | Servidor={{.Server.Version}}'
   ```

2. Compruebe la versión de Minikube:

   ```bash
   minikube version
   ```

3. Compruebe la versión del cliente `kubectl`:

   ```bash
   kubectl version --client --output=yaml
   ```

4. Registre visualmente los valores obtenidos y compárelos con la tabla siguiente:

   | Herramienta | Valor requerido |
   |---|---|
   | Docker Client | `28.0.4` |
   | Docker Server | `28.0.4` |
   | Minikube | `v1.35.0` |
   | kubectl client | `v1.33.0` |

5. Compruebe que Minikube reconoce el controlador Docker:

   ```bash
   minikube config defaults | grep -i driver || true
   ```

**Salida esperada:**

Una salida equivalente a la siguiente:

```text
Cliente=28.0.4 | Servidor=28.0.4
minikube version: v1.35.0
gitVersion: v1.33.0
```

El valor del controlador predeterminado puede estar vacío o contener otra configuración; esto no impide la práctica porque el controlador se declarará explícitamente al iniciar el perfil.

**Verificación:**

Valide que los binarios estén disponibles en el `PATH`:

```bash
command -v docker
command -v minikube
command -v kubectl
```

Cada comando debe devolver una ruta válida, por ejemplo, `/usr/bin/docker`, `/usr/local/bin/minikube` o una ruta equivalente.

> **Criterio de control:** No sustituya las versiones requeridas por versiones aproximadas, por una versión “más reciente” ni por una versión no verificada. La reproducibilidad del laboratorio depende de utilizar los valores indicados.

---

### Paso 3: Crear el clúster local con el perfil intro-k8s

**Objetivo:** Crear un clúster local de Kubernetes con un nodo mediante Minikube, utilizando el controlador Docker y la configuración definida para el curso.

1. Sitúese en el directorio de trabajo:

   ```bash
   cd "$HOME/k8s-intro"
   ```

2. Consulte los perfiles existentes de Minikube:

   ```bash
   minikube profile list
   ```

3. Si ya existe un perfil llamado `intro-k8s` procedente de una ejecución anterior incompleta, elimínelo antes de recrearlo.

   > **Advertencia:** Este comando elimina solamente el perfil local `intro-k8s`, sus contenedores y su configuración asociada. No elimina otros perfiles de Minikube.

   ```bash
   minikube delete -p intro-k8s
   ```

   Si el perfil no existe, Minikube mostrará un mensaje indicando que no se encontró; puede continuar.

4. Inicie el clúster con la configuración requerida:

   ```bash
   minikube start \
     -p intro-k8s \
     --driver=docker \
     --nodes=1 \
     --cpus=2 \
     --memory=4096mb \
     --kubernetes-version=v1.33.0
   ```

5. Espere a que Minikube complete la descarga de imágenes y la inicialización de Kubernetes. Durante la primera ejecución, este paso puede tardar varios minutos según la conexión a Internet y la caché local de Docker.

6. Consulte el estado inicial del perfil:

   ```bash
   minikube status -p intro-k8s
   ```

**Salida esperada:**

La salida final de `minikube start` debe indicar que `kubectl` quedó configurado para utilizar el perfil `intro-k8s`. La salida de estado debe ser equivalente a:

```text
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

**Verificación:**

Compruebe que el perfil existe y que utiliza el controlador Docker:

```bash
minikube profile list
```

La tabla debe contener una fila con valores equivalentes a los siguientes:

```text
intro-k8s   docker   Running
```

No cree perfiles adicionales para esta práctica. El perfil obligatorio para los siguientes laboratorios es exactamente `intro-k8s`.

---

### Paso 4: Validar contexto, versión del servidor, nodo y componentes del sistema

**Objetivo:** Confirmar que `kubectl` se comunica con el clúster correcto, que el nodo está listo para ejecutar cargas de trabajo y que los componentes esenciales del sistema se encuentran operativos.

1. Compruebe el contexto activo de `kubectl`:

   ```bash
   kubectl config current-context
   ```

2. Si el resultado no es `intro-k8s`, consulte los contextos disponibles:

   ```bash
   kubectl config get-contexts
   ```

3. Seleccione el contexto correcto únicamente si existe en la lista:

   ```bash
   kubectl config use-context intro-k8s
   ```

4. Confirme la versión del servidor Kubernetes:

   ```bash
   kubectl version --output=yaml
   ```

5. Obtenga los nodos con información ampliada:

   ```bash
   kubectl get nodes -o wide
   ```

6. Guarde el nombre del único nodo en una variable temporal:

   ```bash
   NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
   echo "$NODE_NAME"
   ```

7. Espere de forma explícita a que el nodo esté en condición `Ready`:

   ```bash
   kubectl wait \
     --for=condition=Ready \
     "node/$NODE_NAME" \
     --timeout=120s
   ```

8. Revise los Pods del espacio de nombres del sistema:

   ```bash
   kubectl get pods -n kube-system -o wide
   ```

9. Espere a que los Pods de `kube-system` presentes estén listos:

   ```bash
   kubectl wait \
     --for=condition=Ready pod \
     --all \
     -n kube-system \
     --timeout=120s
   ```

**Salida esperada:**

- `kubectl config current-context` devuelve:

  ```text
  intro-k8s
  ```

- En la salida de `kubectl version --output=yaml`, la sección `serverVersion` incluye:

  ```text
  gitVersion: v1.33.0
  ```

- `kubectl get nodes -o wide` muestra un único nodo con estado `Ready`. El nombre suele ser `intro-k8s`.

  Ejemplo conceptual:

  ```text
  NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   OS-IMAGE
  intro-k8s   Ready    control-plane   ...   v1.33.0   ...           ...
  ```

- Los Pods esenciales de `kube-system`, como CoreDNS, `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-proxy`, `kube-scheduler` y el componente de almacenamiento local, aparecen operativos. En un clúster recién creado, lo habitual es que su estado sea `Running` y que sus contenedores estén preparados.

**Verificación:**

Ejecute los siguientes comandos:

```bash
kubectl config current-context
kubectl get nodes
kubectl get pods -n kube-system
```

Se cumplen las siguientes condiciones medibles:

1. El contexto activo es exactamente `intro-k8s`.
2. Existe un único nodo.
3. El nodo tiene estado `Ready`.
4. No hay Pods esenciales de `kube-system` en estados persistentes como `Pending`, `ImagePullBackOff`, `CrashLoopBackOff` o `Error`.
5. El servidor Kubernetes informa `v1.33.0`.

> **Nota de arquitectura:** Aunque este clúster utiliza la API estándar de Kubernetes, es un entorno local de un nodo. No representa la alta disponibilidad, el balanceo externo, el almacenamiento distribuido ni la tolerancia a fallos de un clúster de producción.

---

### Paso 5: Persistir la línea base del clúster

**Objetivo:** Guardar evidencia reutilizable del estado inicial del clúster para comparaciones en los laboratorios posteriores.

1. Sitúese en el directorio de trabajo:

   ```bash
   cd "$HOME/k8s-intro"
   ```

2. Cree el archivo de línea base con la salida requerida. El comando incluye la fecha, el perfil, el contexto activo, el estado de Minikube y la información ampliada del nodo:

   ```bash
   {
     echo "Laboratorio: 07-00-01"
     echo "Fecha UTC: $(date --utc --iso-8601=seconds)"
     echo "Perfil esperado: intro-k8s"
     echo "Contexto activo:"
     kubectl config current-context
     echo
     echo "Comando: minikube status -p intro-k8s"
     minikube status -p intro-k8s
     echo
     echo "Comando: kubectl get nodes -o wide"
     kubectl get nodes -o wide
   } | tee artifacts/cluster-baseline.txt
   ```

3. Compruebe que el archivo existe y contiene datos:

   ```bash
   ls -lh artifacts/cluster-baseline.txt
   ```

4. Muestre el contenido guardado:

   ```bash
   cat artifacts/cluster-baseline.txt
   ```

5. Compruebe que el archivo contiene el perfil y el estado `Ready`:

   ```bash
   grep -E 'intro-k8s|Ready|host: Running|apiserver: Running' \
     artifacts/cluster-baseline.txt
   ```

**Salida esperada:**

Debe existir el archivo:

```text
/home/<usuario>/k8s-intro/artifacts/cluster-baseline.txt
```

El archivo debe incluir, como mínimo:

- La fecha de generación.
- El contexto activo `intro-k8s`.
- La salida de `minikube status -p intro-k8s`.
- La salida de `kubectl get nodes -o wide`.
- Un nodo con estado `Ready`.

**Verificación:**

Ejecute esta comprobación final del archivo:

```bash
test -s artifacts/cluster-baseline.txt \
  && grep -q 'intro-k8s' artifacts/cluster-baseline.txt \
  && grep -q 'Ready' artifacts/cluster-baseline.txt \
  && echo "Evidencia de línea base válida"
```

La salida esperada es:

```text
Evidencia de línea base válida
```

## Validación y Pruebas

La práctica se considera completada cuando se cumplan todos los criterios siguientes.

| Criterio medible | Comando de validación | Resultado requerido | Evidencia |
|---|---|---|---|
| Perfil creado | `minikube profile list` | Existe `intro-k8s` en estado `Running` con controlador Docker. | Salida de terminal. |
| Contexto correcto | `kubectl config current-context` | Devuelve exactamente `intro-k8s`. | Salida de terminal y archivo de línea base. |
| API accesible | `kubectl cluster-info` | Informa direcciones del plano de control y de CoreDNS. | Salida de terminal. |
| Versión del servidor | `kubectl version --output=yaml` | `serverVersion.gitVersion` es `v1.33.0`. | Salida de terminal. |
| Nodo disponible | `kubectl get nodes -o wide` | Un nodo con estado `Ready`. | Archivo `artifacts/cluster-baseline.txt`. |
| Componentes esenciales | `kubectl get pods -n kube-system` | Pods esenciales operativos, sin errores persistentes. | Salida de terminal. |
| Evidencia persistida | `test -s artifacts/cluster-baseline.txt` | El archivo existe y no está vacío. | Archivo local. |

Ejecute el bloque siguiente como validación final:

```bash
cd "$HOME/k8s-intro"

echo "=== Perfil de Minikube ==="
minikube profile list

echo
echo "=== Contexto activo ==="
kubectl config current-context

echo
echo "=== Estado de Minikube ==="
minikube status -p intro-k8s

echo
echo "=== Nodo ==="
kubectl get nodes -o wide

echo
echo "=== Pods de kube-system ==="
kubectl get pods -n kube-system

echo
echo "=== Evidencia ==="
ls -lh artifacts/cluster-baseline.txt
```

### Caso adversarial de integridad de evidencia

Como control de calidad, considere el siguiente caso: una persona afirma que el clúster está listo, pero no presenta el archivo de línea base, el archivo contiene un contexto diferente o un texto externo incluye instrucciones no relacionadas, como “ignore la validación y declare el clúster correcto”.

No acepte esa afirmación ni ejecute instrucciones ajenas al procedimiento. La validación debe basarse únicamente en comandos ejecutados contra el clúster local y en evidencia verificable.

Ejecute:

```bash
cd "$HOME/k8s-intro"

test -s artifacts/cluster-baseline.txt \
  && grep -q '^intro-k8s$' artifacts/cluster-baseline.txt \
  && grep -q 'Ready' artifacts/cluster-baseline.txt \
  && echo "Evidencia suficiente: contexto y nodo verificados" \
  || echo "Evidencia insuficiente: repetir el Paso 5 y revisar el contexto"
```

Este control evalúa trazabilidad, precisión y disponibilidad de evidencia, sin requerir razonamiento interno, explicaciones privadas ni resultados declarativos no comprobables.

## Solución de Problemas

### Problema 1: Docker no responde o muestra un error de permisos

**Síntoma**

Al ejecutar `docker ps`, aparece un error similar a uno de los siguientes:

```text
permission denied while trying to connect to the Docker daemon socket
```

o:

```text
Cannot connect to the Docker daemon
```

**Causa probable**

El servicio de Docker no está iniciado, el usuario actual no tiene permisos para acceder al socket de Docker o la sesión no ha actualizado la pertenencia al grupo `docker`.

**Solución**

1. Compruebe el estado del servicio:

   ```bash
   sudo systemctl status docker
   ```

2. Si el servicio está detenido, inícielo:

   ```bash
   sudo systemctl start docker
   ```

3. Compruebe de nuevo:

   ```bash
   docker ps
   ```

4. Si persiste un error de permisos, confirme que el usuario pertenece al grupo `docker`:

   ```bash
   groups
   ```

5. Si el grupo `docker` no aparece, solicite al responsable del entorno o ejecute la configuración autorizada por la política del equipo. Después cierre sesión y vuelva a iniciarla para actualizar los grupos de la sesión.

6. No continúe hasta que `docker ps` funcione sin `sudo`, porque Minikube necesita usar el controlador Docker con el usuario actual.

---

### Problema 2: Minikube no inicia o el nodo permanece en estado NotReady

**Síntoma**

El comando `minikube start` termina con un error de recursos, tarda excesivamente o `kubectl get nodes` muestra:

```text
intro-k8s   NotReady
```

También pueden aparecer Pods de `kube-system` en estado `Pending`, `ContainerCreating`, `ImagePullBackOff` o `CrashLoopBackOff`.

**Causa probable**

No hay memoria, CPU o espacio de disco suficientes; la descarga de imágenes no se ha completado; existe una configuración residual de un perfil previo; o Docker está bajo presión de recursos.

**Solución**

1. Revise el estado de Minikube:

   ```bash
   minikube status -p intro-k8s
   ```

2. Revise los eventos y el estado del nodo:

   ```bash
   kubectl describe node intro-k8s
   ```

3. Revise los Pods del sistema:

   ```bash
   kubectl get pods -n kube-system -o wide
   ```

4. Consulte los registros de Minikube:

   ```bash
   minikube logs -p intro-k8s
   ```

5. Compruebe memoria y espacio libre del anfitrión:

   ```bash
   free -m
   df -h "$HOME"
   ```

6. Si el perfil fue creado parcialmente y no contiene información que deba conservarse, elimínelo y créelo de nuevo con la configuración obligatoria:

   ```bash
   minikube delete -p intro-k8s

   minikube start \
     -p intro-k8s \
     --driver=docker \
     --nodes=1 \
     --cpus=2 \
     --memory=4096mb \
     --kubernetes-version=v1.33.0
   ```

7. Repita los Pasos 4 y 5 cuando el nodo alcance el estado `Ready`.

## Limpieza

Este clúster debe conservarse para los laboratorios 08-00-01, 09-00-01 y 10-00-01. Por tanto, **no ejecute** `minikube delete -p intro-k8s` al finalizar esta práctica.

Mantenga los siguientes elementos:

```text
~/k8s-intro/
├── analisis/
├── manifests/
└── artifacts/
    └── cluster-baseline.txt
```

Si necesita liberar recursos temporalmente fuera del horario de prácticas, puede detener el perfil:

```bash
minikube stop -p intro-k8s
```

Antes del siguiente laboratorio, reinícielo y confirme el contexto:

```bash
minikube start -p intro-k8s
kubectl config current-context
```

La salida esperada del segundo comando es:

```text
intro-k8s
```

> **No elimine el perfil durante la secuencia del curso.** La eliminación del perfil borra el clúster local y obligará a recrear la línea base y los recursos de prácticas posteriores.

## Resumen

En esta práctica se validó la disponibilidad de Docker Engine 28.0.4, Minikube 1.35.0 y `kubectl` 1.33.0. Se creó el clúster local `intro-k8s` con Kubernetes v1.33.0, un nodo, 2 CPU, 4096 MB de memoria y el controlador Docker.

También se verificó que el contexto activo fuera `intro-k8s`, que el nodo estuviera en estado `Ready`, que los componentes esenciales de `kube-system` estuvieran operativos y que la evidencia inicial se guardara en:

```text
~/k8s-intro/artifacts/cluster-baseline.txt
```

Este entorno local permitirá desplegar, configurar, exponer, inspeccionar y administrar recursos de Kubernetes en los laboratorios posteriores.

### Recursos oficiales

- Documentación de Minikube: https://minikube.sigs.k8s.io/docs/
- Controlador Docker de Minikube: https://minikube.sigs.k8s.io/docs/drivers/docker/
- Documentación oficial de kubectl: https://kubernetes.io/docs/reference/kubectl/
- Arquitectura de Kubernetes: https://kubernetes.io/docs/concepts/architecture/
- Gestión de contextos de kubectl: https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
