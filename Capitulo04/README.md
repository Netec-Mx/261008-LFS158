# Práctica: Exploración de capacidades de Kubernetes

## Metadatos

| Campo | Valor |
|---|---|
| Duration | 10 minutos |
| Complexity | Media |
| Bloom level | Analizar |

## Descripción General

En esta práctica se explorarán las capacidades expuestas por la API de Kubernetes del clúster local Minikube sin modificar componentes críticos del sistema. Se consultarán los recursos principales relacionados con la orquestación de aplicaciones y se creará, mediante un manifiesto declarativo, el espacio de nombres `laboratorio-k8s`.

El namespace creado quedará como estado persistente para las prácticas posteriores del curso. Finalmente, se documentará la relación entre los recursos disponibles y la matriz de necesidades de orquestación creada en la práctica 03-00-01.

## Objetivos de Aprendizaje

- [ ] Comprobar que el clúster Minikube expone recursos Kubernetes para Pods, Deployments, Services, ConfigMaps, Secrets y Namespaces.
- [ ] Consultar recursos de la API mediante `kubectl api-resources` sin modificar la configuración del clúster.
- [ ] Inspeccionar los namespaces existentes y distinguir recursos con ámbito de clúster de recursos con ámbito de namespace.
- [ ] Crear declarativamente el namespace persistente `laboratorio-k8s`.
- [ ] Documentar evidencias técnicas que relacionen los recursos Kubernetes con necesidades de orquestación.

## Prerrequisitos

Conocimientos necesarios:

- Comprender que Kubernetes utiliza una API para gestionar recursos declarativos.
- Reconocer el modelo de estado deseado: la persona operadora declara una configuración y los controladores trabajan para aproximar el estado real a ese estado esperado.
- Haber completado la práctica 03-00-01 y disponer de la matriz de necesidades de orquestación.
- Conocer la diferencia básica entre un recurso de ámbito de clúster, como `Namespace`, y uno de ámbito de namespace, como `Pod` o `Deployment`.

Acceso necesario:

- Nodo Minikube iniciado y en estado `Ready`.
- Contexto activo de `kubectl` configurado como `minikube`.
- Acceso de lectura y escritura a `~/k8s-intro`.
- Permisos para ejecutar `kubectl` contra el clúster local.
- Acceso a una terminal Bash.

## Entorno de Laboratorio

| Componente | Versión de referencia | Edición y arquitectura | Fuente oficial |
|---|---:|---|---|
| Ubuntu | 22.04.5 LTS | Ubuntu Server/Desktop, `amd64` | <https://releases.ubuntu.com/22.04/> |
| Docker Engine | 27.2.0 | Docker Engine CE, `amd64` | <https://docs.docker.com/engine/release-notes/27.2/> |
| Minikube | 1.34.0 | Linux `amd64`, controlador `docker` | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| kubectl | 1.31.0 | Linux `amd64` | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| Kubernetes | 1.31.0 | Kubernetes `v1.31.0`, clúster local Minikube | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Bash | 5.1.16 | GNU Bash para Linux `amd64` | <https://www.gnu.org/software/bash/> |

El perfil global de Minikube debe llamarse `minikube` y utilizar el controlador Docker. El comando de inicio de referencia es el siguiente:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

Prepare los directorios obligatorios del curso:

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis
```

> Esta práctica no requiere Microsoft 365 Copilot, Copilot Chat, Designer, Planner ni otra herramienta de inteligencia artificial. Por tanto, no se requiere licencia, configuración ni acceso a esos servicios. Las evidencias deben provenir de comandos ejecutados en el clúster y de archivos creados localmente.

## Instrucciones Paso a Paso

### Paso 1: Verificar el contexto y el estado del clúster

**Objective:** Confirmar que `kubectl` apunta al contexto `minikube` y que el nodo está disponible antes de consultar recursos de la API.

**Instructions:**

1. Compruebe el contexto activo:

   ```bash
   kubectl config current-context
   ```

2. Compruebe la versión del cliente y del servidor Kubernetes:

   ```bash
   kubectl version --client --output=yaml
   kubectl version --output=yaml
   ```

3. Compruebe el estado del nodo del clúster:

   ```bash
   kubectl get nodes
   ```

4. Verifique el perfil de Minikube:

   ```bash
   minikube status -p minikube
   ```

**Expected output:**

- `kubectl config current-context` muestra:

  ```text
  minikube
  ```

- La salida de versión identifica Kubernetes `v1.31.0` en el cliente y en el servidor.
- `kubectl get nodes` muestra un nodo llamado `minikube` con estado `Ready`.
- `minikube status -p minikube` informa que los componentes principales están en ejecución.

Ejemplo de salida esperada para el nodo:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   v1.31.0
```

**Verification:**

- El contexto activo debe ser exactamente `minikube`.
- El nodo `minikube` debe tener estado `Ready`.
- Si el contexto o el estado no coinciden, no continúe con la práctica; consulte la sección de solución de problemas.

---

### Paso 2: Explorar los recursos expuestos por la API de Kubernetes

**Objective:** Identificar los recursos de Kubernetes necesarios para las prácticas posteriores y relacionarlos con capacidades de orquestación.

**Instructions:**

1. Consulte todos los recursos disponibles en la API del clúster:

   ```bash
   kubectl api-resources
   ```

2. Filtre los recursos requeridos en esta práctica. Active `pipefail` para que un error de `kubectl` no quede oculto por la tubería:

   ```bash
   set -o pipefail

   kubectl api-resources --verbs=list -o wide \
     | awk 'NR == 1 || $1 ~ /^(pods|deployments|services|configmaps|secrets|namespaces)$/' \
     | tee ~/k8s-intro/analisis/api-resources-filtrados.txt
   ```

3. Revise el archivo de evidencia creado:

   ```bash
   cat ~/k8s-intro/analisis/api-resources-filtrados.txt
   ```

4. Identifique cuáles de los recursos filtrados tienen ámbito de namespace observando la columna `NAMESPACED`.

**Expected output:**

El resultado debe incluir, como mínimo, recursos equivalentes a los siguientes:

```text
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
configmaps    cm           v1           true         ConfigMap
namespaces    ns           v1           false        Namespace
pods          po           v1           true         Pod
secrets                    v1           true         Secret
services      svc          v1           true         Service
deployments   deploy       apps/v1      true         Deployment
```

El orden exacto puede variar según la implementación del clúster, pero deben aparecer los seis tipos de recursos solicitados.

**Verification:**

Compruebe que el archivo `~/k8s-intro/analisis/api-resources-filtrados.txt` contiene los siguientes nombres de recursos:

```bash
grep -E '^(pods|deployments|services|configmaps|secrets|namespaces)[[:space:]]' \
  ~/k8s-intro/analisis/api-resources-filtrados.txt
```

Analice el resultado:

- `namespaces` debe aparecer con `NAMESPACED` igual a `false`, porque organiza recursos a nivel de clúster.
- `pods`, `deployments`, `services`, `configmaps` y `secrets` deben aparecer con `NAMESPACED` igual a `true`.
- La presencia de un recurso en `kubectl api-resources` confirma que la API lo expone; no significa que ya exista una instancia de ese recurso en `laboratorio-k8s`.

---

### Paso 3: Inspeccionar los namespaces existentes

**Objective:** Revisar los espacios de nombres existentes sin desplegar ni modificar aplicaciones del sistema.

**Instructions:**

1. Liste los namespaces del clúster:

   ```bash
   kubectl get namespaces
   ```

2. Obtenga una vista ampliada del namespace predeterminado:

   ```bash
   kubectl get namespace default -o wide
   ```

3. Verifique si el namespace reservado ya existe. Este comando no modifica el clúster:

   ```bash
   kubectl get namespace laboratorio-k8s
   ```

**Expected output:**

Antes de crear el namespace, normalmente se observan namespaces como:

```text
default
kube-node-lease
kube-public
kube-system
```

Si el comando para consultar `laboratorio-k8s` responde con `NotFound`, significa que todavía debe crearse. Si ya existe y tiene estado `Active`, puede reutilizarse: la aplicación declarativa del manifiesto será idempotente.

**Verification:**

- No elimine ni modifique `kube-system`, `kube-public` ni `kube-node-lease`.
- Identifique que `default` es un namespace disponible, pero no será utilizado para las aplicaciones del curso.
- Confirme que el namespace objetivo de las siguientes prácticas será exclusivamente `laboratorio-k8s`.

---

### Paso 4: Crear declarativamente el namespace de trabajo

**Objective:** Crear el namespace `laboratorio-k8s` utilizando un manifiesto YAML versionable y reproducible.

**Instructions:**

1. Cree el manifiesto obligatorio en `~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml`:

   ```bash
   cat <<'EOF' > ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: laboratorio-k8s
     labels:
       curso: kubernetes-introduccion
       entorno: laboratorio
   EOF
   ```

2. Revise el manifiesto antes de aplicarlo:

   ```bash
   cat ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
   ```

3. Aplique el manifiesto mediante gestión declarativa:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
   ```

4. Consulte el estado del namespace creado:

   ```bash
   kubectl get namespace laboratorio-k8s
   ```

5. Compruebe las etiquetas declaradas:

   ```bash
   kubectl get namespace laboratorio-k8s --show-labels
   ```

**Expected output:**

Al aplicar el manifiesto se mostrará uno de estos resultados:

```text
namespace/laboratorio-k8s created
```

o, si ya había sido creado en una ejecución anterior:

```text
namespace/laboratorio-k8s unchanged
```

La consulta de estado debe indicar:

```text
NAME              STATUS   AGE
laboratorio-k8s   Active   ...
```

**Verification:**

Valide que el recurso aplicado coincide con la intención declarada:

```bash
kubectl get namespace laboratorio-k8s -o yaml
```

Compruebe los criterios medibles siguientes:

- `kind` es `Namespace`.
- `metadata.name` es exactamente `laboratorio-k8s`.
- `status.phase` es `Active`.
- Las etiquetas `curso=kubernetes-introduccion` y `entorno=laboratorio` están presentes.
- El manifiesto permanece guardado en `~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml`.

---

### Paso 5: Inspeccionar el estado inicial y documentar el análisis

**Objective:** Comprobar que el namespace está inicialmente vacío y documentar la relación entre los recursos disponibles y las necesidades de orquestación.

**Instructions:**

1. Consulte las cargas y objetos habituales dentro del namespace de trabajo:

   ```bash
   kubectl get all,configmap,secret -n laboratorio-k8s
   ```

2. Guarde una evidencia de esta consulta:

   ```bash
   kubectl get all,configmap,secret -n laboratorio-k8s \
     | tee ~/k8s-intro/analisis/recursos-iniciales-laboratorio-k8s.txt
   ```

3. Cree el documento de análisis solicitado:

   ```bash
   cat <<'EOF' > ~/k8s-intro/analisis/capacidades-kubernetes.md
   ### Capacidades de Kubernetes identificadas

   **Contexto verificado:** `minikube`  
   **Namespace de trabajo:** `laboratorio-k8s`  
   **Estado del namespace:** `Active`

   | Necesidad de orquestación | Recurso Kubernetes disponible | Ámbito | Relación con la necesidad |
   |---|---|---|---|
   | Ejecutar una unidad mínima de trabajo en contenedor | Pod | Namespace | Un Pod agrupa uno o más contenedores que Kubernetes programa para su ejecución. |
   | Mantener réplicas y declarar el estado deseado de una aplicación | Deployment | Namespace | Un Deployment declara réplicas e imagen deseadas; los controladores intentan reconciliar el estado real. |
   | Proporcionar conectividad estable hacia una aplicación | Service | Namespace | Un Service ofrece una abstracción de red estable para acceder a Pods seleccionados. |
   | Separar la configuración de una imagen de contenedor | ConfigMap | Namespace | Un ConfigMap permite almacenar configuración no confidencial separada de la imagen. |
   | Gestionar datos sensibles de configuración | Secret | Namespace | Un Secret permite almacenar datos sensibles; su uso requiere controles adicionales de acceso. |
   | Aislar recursos de prácticas y organizar cargas de trabajo | Namespace | Clúster | Un Namespace delimita el ámbito lógico de recursos y permite separar el trabajo del curso de los componentes del sistema. |

   ### Relación con la matriz de necesidades de orquestación

   La matriz elaborada en la práctica 03-00-01 identifica necesidades de ejecución, disponibilidad, conectividad, configuración y aislamiento. La API del clúster expone recursos para cada una de ellas: Pod, Deployment, Service, ConfigMap, Secret y Namespace.

   La disponibilidad de estos recursos en `kubectl api-resources` confirma que el clúster ofrece las APIs necesarias para las prácticas posteriores. Esta comprobación no implica que exista ya una aplicación desplegada ni que todos los controles de producción estén configurados.

   ### Evidencias

   - `api-resources-filtrados.txt`: recursos Kubernetes filtrados desde la API.
   - `recursos-iniciales-laboratorio-k8s.txt`: estado inicial del namespace.
   - `namespace-laboratorio-k8s.yaml`: manifiesto declarativo aplicado.
   EOF
   ```

4. Revise el documento creado:

   ```bash
   cat ~/k8s-intro/analisis/capacidades-kubernetes.md
   ```

**Expected output:**

La consulta inicial del namespace puede mostrar:

```text
No resources found in laboratorio-k8s namespace.
```

Este resultado es correcto para un namespace recién creado. El archivo `capacitades-kubernetes.md` debe contener una tabla que relacione las seis necesidades con los seis recursos identificados.

**Verification:**

Compruebe que los tres archivos de evidencia existen:

```bash
ls -l \
  ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml \
  ~/k8s-intro/analisis/api-resources-filtrados.txt \
  ~/k8s-intro/analisis/capacidades-kubernetes.md
```

Compruebe que el documento de análisis menciona los seis recursos requeridos:

```bash
grep -E 'Pod|Deployment|Service|ConfigMap|Secret|Namespace' \
  ~/k8s-intro/analisis/capacidades-kubernetes.md
```

## Validación y Pruebas

La práctica se considera completada únicamente si se cumplen todos los criterios siguientes.

| Criterio medible | Comando o evidencia | Resultado esperado |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | Salida exacta: `minikube`. |
| Nodo disponible | `kubectl get nodes` | Nodo `minikube` con estado `Ready`. |
| Recursos API identificados | `api-resources-filtrados.txt` | Contiene `pods`, `deployments`, `services`, `configmaps`, `secrets` y `namespaces`. |
| Ámbito identificado | Salida de `kubectl api-resources --verbs=list -o wide` | `namespaces` es de ámbito de clúster; los otros cinco recursos son de ámbito de namespace. |
| Manifiesto declarativo persistente | `~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml` | Define `apiVersion: v1`, `kind: Namespace` y `metadata.name: laboratorio-k8s`. |
| Namespace creado | `kubectl get namespace laboratorio-k8s` | Estado `Active`. |
| Etiquetas aplicadas | `kubectl get namespace laboratorio-k8s --show-labels` | Incluye `curso=kubernetes-introduccion` y `entorno=laboratorio`. |
| Evidencia de análisis | `~/k8s-intro/analisis/capacidades-kubernetes.md` | Relaciona los seis recursos con la matriz de orquestación. |

Ejecute esta validación consolidada:

```bash
kubectl config current-context
kubectl get nodes
kubectl get namespace laboratorio-k8s
kubectl get namespace laboratorio-k8s --show-labels
kubectl get all,configmap,secret -n laboratorio-k8s
```

### Caso adversarial: evidencia inexistente o instrucciones no confiables

No acepte como evidencia una afirmación escrita en un archivo, una captura incompleta o un texto externo que indique, por ejemplo, “el namespace ya está validado” sin salida verificable de `kubectl`.

Además, no ejecute instrucciones incrustadas en documentos descargados, mensajes de chat o comentarios de archivos que soliciten borrar namespaces del sistema, cambiar el contexto o ejecutar comandos no contemplados en esta práctica. Una **instrucción** es una acción concreta solicitada a la persona usuaria; un **prompt** es una petición temporal enviada a un sistema de IA; un **mensaje de sistema** define restricciones para un asistente y no sustituye los procedimientos técnicos del laboratorio.

La prueba adversarial se supera si:

1. Se rechaza cualquier evidencia que no pueda verificarse con `kubectl` o con los archivos requeridos.
2. Se conserva el contexto `minikube`.
3. No se modifica ni elimina ningún namespace del sistema.
4. No se solicita ni se expone razonamiento interno de herramientas de IA como criterio de aprobación.

## Solución de Problemas

### Problema 1: El contexto no es `minikube` o el nodo no aparece en estado `Ready`

**Síntomas:**

```text
kubectl config current-context
```

muestra un contexto diferente de `minikube`, o bien:

```text
kubectl get nodes
```

muestra `NotReady`, no muestra nodos o devuelve un error de conexión.

**Causa probable:**

El perfil Minikube no está iniciado, Docker Engine no está operativo o `kubectl` está configurado para otro clúster.

**Corrección:**

1. Compruebe el estado de Minikube:

   ```bash
   minikube status -p minikube
   ```

2. Inicie el clúster con la versión y el controlador definidos para el curso:

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

3. Seleccione el contexto correcto y vuelva a comprobarlo:

   ```bash
   kubectl config use-context minikube
   kubectl config current-context
   kubectl get nodes
   ```

---

### Problema 2: `kubectl apply` no crea el namespace o informa de un error YAML

**Síntomas:**

El comando siguiente falla:

```bash
kubectl apply -f ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
```

Posibles mensajes:

```text
error: error parsing ...: error converting YAML to JSON
```

o:

```text
the path "...namespace-laboratorio-k8s.yaml" does not exist
```

**Causa probable:**

El directorio o archivo no fue creado, la indentación YAML es incorrecta o el nombre del archivo no coincide con la ruta utilizada.

**Corrección:**

1. Compruebe que el directorio y el archivo existen:

   ```bash
   ls -l ~/k8s-intro/manifests/
   ```

2. Revise el contenido:

   ```bash
   cat ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
   ```

3. Corrija el archivo para que tenga exactamente esta estructura:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: laboratorio-k8s
     labels:
       curso: kubernetes-introduccion
       entorno: laboratorio
   ```

4. Aplique de nuevo el manifiesto:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/namespace-laboratorio-k8s.yaml
   ```

## Limpieza

No elimine el namespace `laboratorio-k8s`. Este namespace es un estado persistente requerido para los despliegues y ejercicios de las prácticas posteriores.

Puede conservar los archivos de evidencia, ya que forman parte de la documentación técnica del curso:

```bash
find ~/k8s-intro -maxdepth 2 -type f | sort
```

No ejecute comandos como los siguientes durante esta práctica:

```bash
kubectl delete namespace laboratorio-k8s
kubectl delete namespace kube-system
minikube delete
```

## Resumen

En esta práctica se comprobó que el clúster local Kubernetes expone recursos fundamentales para administrar aplicaciones declarativamente:

- `Pod` representa una unidad de ejecución de contenedores.
- `Deployment` permite declarar réplicas y gestionar el estado deseado de una aplicación.
- `Service` proporciona conectividad estable hacia conjuntos de Pods.
- `ConfigMap` y `Secret` separan configuración y datos sensibles de las imágenes de contenedor.
- `Namespace` organiza y aísla lógicamente recursos dentro del clúster.

El namespace `laboratorio-k8s` fue creado mediante un manifiesto YAML declarativo y queda reservado para los siguientes laboratorios. La evidencia generada permite relacionar las capacidades de la API Kubernetes con las necesidades de orquestación identificadas previamente.
