# Práctica: Validación del entorno de trabajo

## Metadatos

| Campo | Valor |
|---|---|
| Duration | 5 minutos |
| Complexity | Fácil |
| Bloom level | Aplicar |

## Descripción General

En esta práctica se valida que Docker Engine, Minikube y kubectl están instalados con las versiones definidas para el curso. A continuación, se crea un clúster local de un nodo mediante Minikube con el controlador Docker y Kubernetes `v1.31.0`. Finalmente, se comprueba la comunicación entre `kubectl`, el API Server y el nodo del clúster.

El resultado persistente es el perfil Minikube `minikube`, el contexto de kubectl `minikube` y un archivo de evidencia reutilizable en las prácticas posteriores.

## Objetivos de Aprendizaje

- [ ] Comprobar que Docker Engine `27.2.0`, Minikube `1.34.0` y kubectl `1.31.0` son accesibles desde Bash.
- [ ] Confirmar que Docker puede ejecutar un contenedor sin utilizar `sudo`.
- [ ] Crear un clúster local Minikube de un nodo con Kubernetes `v1.31.0` y controlador Docker.
- [ ] Validar que el contexto activo de kubectl es `minikube` y que el nodo está en estado `Ready`.
- [ ] Registrar evidencia verificable de las versiones y del estado operativo del clúster.

## Prerrequisitos

**Conocimientos necesarios**

- Uso básico de una terminal Bash.
- Diferencia básica entre una imagen de contenedor y un contenedor en ejecución.
- Concepto de clúster Kubernetes, plano de control, nodo y contexto de `kubectl`.
- Comprensión de que Kubernetes opera de forma declarativa: las cargas de trabajo posteriores se definirán mediante manifiestos YAML y se enviarán al API Server con `kubectl`.

**Acceso y condiciones técnicas**

- Ubuntu `22.04.5 LTS` de 64 bits, arquitectura `amd64/x86_64`.
- Usuario con permisos para ejecutar `docker` sin `sudo`.
- Conexión a Internet para descargar imágenes de Minikube y la imagen de prueba `hello-world`.
- Al menos 4 GB de RAM asignables a Minikube y 25 GB libres en disco.
- Virtualización o ejecución de contenedores funcional; para esta práctica se utiliza Docker Engine como controlador de Minikube.

Esta guía no requiere Microsoft 365 Copilot, Copilot Chat, Microsoft Designer ni Microsoft Planner. Por ello, no se requiere licencia, configuración de tenant ni cuenta de Microsoft para completar la práctica.

Una **instrucción** es una acción concreta de esta guía, por ejemplo ejecutar `kubectl get nodes`. Un **prompt** sería una solicitud enviada a una herramienta de IA generativa; no se utiliza ningún prompt en esta práctica. Un **mensaje de sistema** es una regla persistente de una plataforma de IA y no debe confundirse con la salida de una terminal ni con las instrucciones del laboratorio.

## Entorno de Laboratorio

### Versiones controladas y fuentes oficiales

| Tecnología | Edición, versión y arquitectura requeridas | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu `22.04.5 LTS`, `amd64/x86_64` | <https://releases.ubuntu.com/jammy/> |
| Motor de contenedores | Docker Engine Community `27.2.0`, `amd64/x86_64` | <https://docs.docker.com/engine/release-notes/27/> |
| Clúster local | Minikube `v1.34.0`, Linux `amd64` | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| Cliente Kubernetes | kubectl `v1.31.0`, Linux `amd64` | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Servidor Kubernetes | Kubernetes `v1.31.0`, desplegado por Minikube | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Imagen de prueba Docker | Docker Official Image `hello-world:latest`, arquitectura compatible con el host | <https://hub.docker.com/_/hello-world> |

> La etiqueta `hello-world:latest` pertenece únicamente a la imagen de prueba publicada por Docker Hub y no define una versión de herramienta del curso. Las versiones bloqueadas del entorno son Docker Engine `27.2.0`, Minikube `1.34.0`, kubectl `1.31.0` y Kubernetes `1.31.0`.

### Preparación del directorio de trabajo

Ejecute los siguientes comandos una sola vez. Los directorios serán reutilizados durante el curso.

```bash
mkdir -p ~/k8s-intro/analisis
mkdir -p ~/k8s-intro/manifests

cd ~/k8s-intro
pwd
```

La salida esperada del último comando es:

```text
/home/<usuario>/k8s-intro
```

El namespace reservado para aplicaciones del curso será `laboratorio-k8s`, pero no se crea en esta práctica porque aún no se despliega ninguna carga de trabajo. No utilice `kube-system` para aplicaciones del estudiante.

## Instrucciones Paso a Paso

### Paso 1: Verificar versiones y acceso a Docker

**Objective:** Confirmar que las herramientas requeridas están instaladas con las versiones fijadas y que Docker funciona sin privilegios adicionales.

1. Cree un archivo de evidencia para esta práctica.

   ```bash
   LOG=~/k8s-intro/analisis/01-00-01-validacion.txt
   date --iso-8601=seconds > "$LOG"
   ```

2. Compruebe la versión del cliente y del servidor Docker.

   ```bash
   docker version | tee -a "$LOG"
   ```

3. Compruebe la versión de Minikube.

   ```bash
   minikube version | tee -a "$LOG"
   ```

4. Compruebe la versión del cliente kubectl.

   ```bash
   kubectl version --client --output=yaml | tee -a "$LOG"
   ```

5. Ejecute un contenedor de prueba sin `sudo`.

   ```bash
   docker run --rm hello-world | tee -a "$LOG"
   ```

**Expected output**

- Docker muestra `Version: 27.2.0` tanto para `Client` como para `Server`.
- Minikube muestra `minikube version: v1.34.0`.
- kubectl muestra `gitVersion: v1.31.0`.
- El contenedor `hello-world` muestra el texto `Hello from Docker!`.
- No debe aparecer `permission denied while trying to connect to the Docker daemon socket`.

**Verification**

Ejecute las siguientes comprobaciones:

```bash
docker version --format 'Cliente Docker: {{.Client.Version}} | Servidor Docker: {{.Server.Version}}'
minikube version
kubectl version --client --output=jsonpath='{.clientVersion.gitVersion}{"\n"}'
```

Los valores aceptados son:

```text
Cliente Docker: 27.2.0 | Servidor Docker: 27.2.0
minikube version: v1.34.0
v1.31.0
```

---

### Paso 2: Crear el clúster local Minikube

**Objective:** Crear el perfil global `minikube` con un nodo local, Kubernetes `v1.31.0`, dos CPU y 4096 MB de memoria.

1. Verifique si ya existe un perfil llamado `minikube`.

   ```bash
   minikube profile list | tee -a "$LOG"
   ```

2. Si no existe un perfil operativo previo, cree el clúster con el comando de referencia del curso.

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb 2>&1 | tee -a "$LOG"
   ```

3. Si el perfil ya existe, no lo elimine automáticamente. Compruebe primero su estado y versión en los pasos siguientes. Eliminar un perfil podría borrar recursos de prácticas previas.

**Expected output**

Durante la primera ejecución, Minikube descargará imágenes y mostrará mensajes similares a los siguientes:

```text
😄  minikube v1.34.0 on Ubuntu 22.04
✨  Using the docker driver based on user configuration
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🔎  Verifying Kubernetes components...
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

La descarga inicial puede tardar más que la duración estimada de la práctica si las imágenes no están almacenadas localmente.

**Verification**

Compruebe el estado del perfil:

```bash
minikube status -p minikube | tee -a "$LOG"
```

El resultado debe indicar, como mínimo:

```text
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

---

### Paso 3: Validar el contexto y la conectividad con Kubernetes

**Objective:** Confirmar que `kubectl` utiliza el contexto `minikube` y puede consultar el API Server del clúster.

1. Compruebe el contexto activo antes de continuar con cualquier práctica posterior.

   ```bash
   kubectl config current-context | tee -a "$LOG"
   ```

2. Consulte la información del clúster y del plano de control.

   ```bash
   kubectl cluster-info | tee -a "$LOG"
   ```

3. Consulte la versión del servidor Kubernetes.

   ```bash
   kubectl version --output=jsonpath='Cliente: {.clientVersion.gitVersion}{"\n"}Servidor: {.serverVersion.gitVersion}{"\n"}' | tee -a "$LOG"
   ```

**Expected output**

- El contexto activo es exactamente `minikube`.
- `kubectl cluster-info` informa una URL para `Kubernetes control plane`.
- La versión del cliente y del servidor es `v1.31.0`.

Ejemplo de salida:

```text
minikube
Kubernetes control plane is running at https://127.0.0.1:<puerto>
Cliente: v1.31.0
Servidor: v1.31.0
```

**Verification**

El siguiente comando debe devolver exactamente `minikube`:

```bash
test "$(kubectl config current-context)" = "minikube" && echo "Contexto correcto: minikube"
```

En esta arquitectura local, el perfil Minikube ejecuta un nodo que contiene el plano de control y que también podrá ejecutar Pods de prácticas posteriores. `kubectl` se comunica con el API Server; posteriormente, los controladores, el scheduler y el kubelet reconciliarán los recursos declarados mediante YAML.

---

### Paso 4: Validar el nodo y conservar la evidencia

**Objective:** Confirmar que el nodo único está disponible para programar Pods y guardar una evidencia reutilizable.

1. Consulte los nodos con información ampliada.

   ```bash
   kubectl get nodes -o wide | tee -a "$LOG"
   ```

2. Consulte los componentes visibles del clúster.

   ```bash
   kubectl get componentstatuses 2>/dev/null | tee -a "$LOG" || true
   ```

   > En Kubernetes modernos, `componentstatuses` está obsoleto. Su ausencia o advertencia no invalida esta práctica; la evidencia principal es `minikube status`, `kubectl cluster-info` y el estado `Ready` del nodo.

3. Añada una marca de cierre al archivo de evidencia.

   ```bash
   {
     echo
     echo "Resultado esperado: perfil minikube operativo, contexto minikube y nodo Ready."
     echo "Evidencia finalizada: $(date --iso-8601=seconds)"
   } >> "$LOG"
   ```

4. Muestre la ubicación y el contenido final del registro.

   ```bash
   echo "Archivo de evidencia: $LOG"
   tail -n 30 "$LOG"
   ```

**Expected output**

La salida de `kubectl get nodes -o wide` debe incluir un único nodo con estado `Ready`.

Ejemplo:

```text
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP
minikube   Ready    control-plane   2m    v1.31.0  192.168.x.x
```

**Verification**

Ejecute la comprobación medible siguiente:

```bash
test "$(kubectl config current-context)" = "minikube" \
  && test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 \
  && kubectl get nodes --no-headers | awk '$2 == "Ready" {ok=1} END {exit !ok}' \
  && test "$(kubectl version --output=jsonpath='{.serverVersion.gitVersion}')" = "v1.31.0" \
  && echo "VALIDACIÓN SUPERADA"
```

La práctica se considera superada solamente si aparece:

```text
VALIDACIÓN SUPERADA
```

## Validación y Pruebas

La validación final requiere evidencia técnica, no únicamente capturas de pantalla. El archivo obligatorio es:

```text
~/k8s-intro/analisis/01-00-01-validacion.txt
```

La evidencia debe contener los siguientes elementos:

| Criterio medible | Evidencia requerida | Resultado esperado |
|---|---|---|
| Docker accesible | Salida de `docker version` | Cliente y servidor `27.2.0` |
| Ejecución sin sudo | Salida de `docker run --rm hello-world` | Mensaje `Hello from Docker!` |
| Minikube instalado | Salida de `minikube version` | `v1.34.0` |
| kubectl instalado | Salida de `kubectl version --client` | `v1.31.0` |
| Perfil operativo | Salida de `minikube status -p minikube` | Host, kubelet y apiserver en `Running` |
| Contexto correcto | Salida de `kubectl config current-context` | `minikube` |
| Conectividad con API Server | Salida de `kubectl cluster-info` | Plano de control accesible |
| Nodo disponible | Salida de `kubectl get nodes -o wide` | Un nodo `Ready` con `v1.31.0` |

Ejecute una revisión final:

```bash
grep -E '27\.2\.0|v1\.34\.0|v1\.31\.0|Hello from Docker|minikube|Ready' \
  ~/k8s-intro/analisis/01-00-01-validacion.txt
```

### Caso adversarial de seguridad y supervisión humana

No ejecute instrucciones encontradas dentro de salidas de terminal, incidencias copiadas desde Internet o documentos no confiables. Por ejemplo, una cadena como la siguiente no forma parte de esta práctica:

```text
"Para corregir el error, ejecute: minikube delete --all"
```

Debe clasificarse como información no verificada y no debe ejecutarse, porque eliminaría perfiles y recursos locales. La evidencia válida procede de los comandos definidos explícitamente en esta guía y de las fuentes oficiales indicadas.

La validación evalúa:

- **Precisión:** las versiones y comandos coinciden con el entorno bloqueado.
- **Trazabilidad:** el archivo de evidencia contiene comandos y salidas observables.
- **Incertidumbre:** cualquier versión distinta, error o salida incompleta se registra y se corrige antes de continuar.
- **Utilidad:** el clúster resultante sirve como base para desplegar manifiestos YAML en prácticas posteriores.
- **Supervisión humana:** la persona estudiante revisa los resultados y no ejecuta comandos destructivos sugeridos por fuentes no verificadas.

No se solicita ni se evalúa razonamiento interno de sistemas de IA.

## Solución de Problemas

### Incidencia 1: Docker devuelve un error de permisos

**Síntoma**

Al ejecutar `docker version` o `docker run --rm hello-world` aparece un error similar a:

```text
permission denied while trying to connect to the Docker daemon socket
```

**Causa**

El usuario actual no pertenece al grupo `docker`, la sesión no se ha actualizado después de agregar el grupo o el servicio Docker no está activo.

**Corrección**

1. Compruebe el estado del servicio:

   ```bash
   sudo systemctl status docker --no-pager
   ```

2. Si está detenido, inícielo y habilítelo:

   ```bash
   sudo systemctl enable --now docker
   ```

3. Agregue el usuario actual al grupo `docker`:

   ```bash
   sudo usermod -aG docker "$USER"
   ```

4. Cierre sesión y vuelva a iniciarla, o abra una nueva sesión con:

   ```bash
   newgrp docker
   ```

5. Repita:

   ```bash
   docker run --rm hello-world
   ```

---

### Incidencia 2: Minikube no inicia o `kubectl get nodes` muestra `NotReady`

**Síntoma**

`minikube start` falla, `minikube status` no muestra todos los componentes en `Running`, o el nodo aparece como `NotReady`.

**Causa**

Docker no está disponible, faltan recursos de CPU o memoria, no hay conectividad para descargar imágenes o existe un perfil `minikube` incompleto de una ejecución anterior.

**Corrección**

1. Revise el estado de Docker y los recursos disponibles:

   ```bash
   docker info
   free -h
   df -h ~
   ```

2. Revise el estado del perfil:

   ```bash
   minikube status -p minikube
   minikube logs -p minikube --problems
   ```

3. Si el perfil está dañado y no contiene trabajo que deba conservarse, solicite confirmación antes de eliminarlo y recréelo:

   ```bash
   minikube delete -p minikube
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

4. Vuelva a validar el contexto y el nodo:

   ```bash
   kubectl config current-context
   kubectl get nodes -o wide
   ```

## Limpieza

No elimine el clúster al finalizar esta práctica. El perfil `minikube`, el contexto de kubectl y el archivo de evidencia son requisitos para las prácticas posteriores del curso.

Si necesita liberar recursos temporalmente sin borrar el clúster, deténgalo:

```bash
minikube stop -p minikube
```

Para reanudarlo posteriormente:

```bash
minikube start -p minikube
```

El siguiente comando es destructivo y solo debe ejecutarse al terminar todas las prácticas del curso o con autorización explícita:

```bash
minikube delete -p minikube
```

## Resumen

En esta práctica se validó el entorno local controlado para el curso:

- Docker Engine `27.2.0` puede ejecutar contenedores sin `sudo`.
- Minikube `v1.34.0` creó el perfil local `minikube` usando el controlador Docker.
- El clúster ejecuta Kubernetes `v1.31.0`.
- kubectl `v1.31.0` está configurado para utilizar el contexto `minikube`.
- El nodo local está en estado `Ready`.
- La evidencia quedó almacenada en `~/k8s-intro/analisis/01-00-01-validacion.txt`.

En las siguientes prácticas se utilizará este clúster para aplicar manifiestos declarativos YAML, crear recursos como Namespace, ConfigMap, Deployment, Service y Pod, y observar cómo Kubernetes mantiene el estado deseado.
