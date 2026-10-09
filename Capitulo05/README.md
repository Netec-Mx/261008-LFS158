# Práctica: Identificación de los componentes del clúster

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 20 minutos |
| Complejidad | Media |
| Nivel de Bloom | Analizar |

## Descripción General

En esta práctica inspeccionará un clúster local Minikube ya creado para identificar sus componentes visibles de plano de control, nodo y red. Usará comandos de consulta de `kubectl` y Minikube sin modificar recursos del sistema ni desplegar aplicaciones nuevas. Como evidencia, elaborará el archivo `~/k8s-intro/analisis/componentes-cluster.md`, relacionando cada componente con su responsabilidad y con los comandos que permiten observarlo.

## Objetivos de Aprendizaje

- [ ] Identificar el nodo Minikube y distinguir sus responsabilidades de plano de control y nodo de trabajo en un clúster de un solo nodo.
- [ ] Reconocer `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, CoreDNS, `kube-proxy`, `kubelet` y el complemento de almacenamiento local cuando esté visible.
- [ ] Interpretar namespaces, Pods del sistema, Services y columnas como `NODE`, `STATUS` y `CLUSTER-IP`.
- [ ] Explicar el recorrido básico de una solicitud desde `kubectl` hasta la ejecución y supervisión de un Pod.
- [ ] Registrar evidencia verificable sin asumir que un componente está ausente solo porque no se muestra como un Pod independiente.

## Prerrequisitos

**Conocimientos necesarios**

- Uso básico de terminal Bash.
- Uso de `kubectl get`, `kubectl describe` y namespaces.
- Comprensión inicial de clúster, plano de control, nodo, Pod, Service y estado deseado.
- Práctica 01-00-01 completada con Minikube instalado.
- Práctica 04-00-01 completada con el namespace `laboratorio-k8s` creado.

**Acceso necesario**

- Acceso a una terminal Bash.
- Permiso para ejecutar `kubectl`, `minikube` y `docker` como el usuario actual.
- Clúster Minikube existente con perfil `minikube`.
- Contexto de `kubectl` configurado como `minikube`.
- Conexión a Internet no obligatoria durante la práctica, salvo que el clúster no esté iniciado y necesite descargar imágenes pendientes.

## Entorno de Laboratorio

El entorno de referencia está bloqueado para asegurar resultados comparables. Esta práctica no instala software ni actualiza componentes.

| Tecnología | Edición, versión y arquitectura de referencia | Uso en la práctica | Fuente oficial |
|---|---|---|---|
| Ubuntu | Ubuntu 22.04.5 LTS, `linux/amd64` | Sistema operativo de referencia | https://releases.ubuntu.com/22.04/ |
| Docker Engine | Docker Engine Community Edition 27.2.0, `linux/amd64` | Controlador de Minikube y runtime subyacente | https://docs.docker.com/engine/release-notes/27.2/ |
| Minikube | Minikube 1.34.0, binario `linux/amd64` | Clúster local, perfil `minikube`, controlador Docker | https://github.com/kubernetes/minikube/releases/tag/v1.34.0 |
| kubectl | Kubernetes kubectl 1.31.0, `linux/amd64` | Cliente de la API de Kubernetes | https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/ |
| Kubernetes | Kubernetes upstream 1.31.0, `linux/amd64` | Versión del servidor del clúster | https://kubernetes.io/blog/2024/08/13/kubernetes-v1-31-release/ |

> **Nota sobre arquitectura:** los comandos siguientes validan la versión real del cliente y del servidor. Si el equipo utiliza una arquitectura distinta de `linux/amd64`, documente la diferencia en el informe y no declare que coincide con el entorno de referencia.

No se utilizan Microsoft 365 Copilot, Copilot Chat, Microsoft Designer ni Microsoft Planner en esta práctica. Por tanto, no se requiere licencia, tenant, configuración de cuenta ni extensión para esas herramientas. La evidencia debe proceder de comandos locales y del archivo Markdown elaborado por el estudiante.

Ejecute la siguiente preparación. El comando de inicio solo debe ejecutarse si `minikube status` indica que el clúster está detenido.

```bash
mkdir -p ~/k8s-intro/analisis
mkdir -p ~/k8s-intro/manifests

kubectl config current-context
minikube status -p minikube
```

La salida esperada del contexto es:

```text
minikube
```

Si el perfil está detenido, inícielo con la configuración de referencia:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

Compruebe que el namespace reservado existe:

```bash
kubectl get namespace laboratorio-k8s
```

> **Regla de alcance:** no cree, edite ni elimine recursos en `kube-system`. Esta práctica es exclusivamente de inspección. Tampoco despliegue aplicaciones nuevas en `laboratorio-k8s`.

## Instrucciones Paso a Paso

### Paso 1: Verificar el acceso y registrar el estado inicial

**Objetivo:** confirmar que `kubectl` se comunica con el clúster esperado y generar una primera evidencia reproducible.

**Instrucciones**

1. Vaya al directorio de análisis:

   ```bash
   cd ~/k8s-intro/analisis
   ```

2. Confirme el contexto activo:

   ```bash
   kubectl config current-context
   ```

3. Consulte el estado del perfil Minikube:

   ```bash
   minikube status -p minikube
   ```

4. Registre versiones del cliente y del servidor:

   ```bash
   {
     echo "Fecha UTC: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
     echo
     echo "Contexto:"
     kubectl config current-context
     echo
     echo "Estado de Minikube:"
     minikube status -p minikube
     echo
     echo "Versión de kubectl:"
     kubectl version --client --output=yaml
     echo
     echo "Versión del servidor Kubernetes:"
     kubectl version --output=yaml
   } | tee evidencia-inicial.txt
   ```

5. Verifique que el namespace del curso continúa disponible:

   ```bash
   kubectl get namespace laboratorio-k8s
   ```

**Salida esperada**

- El contexto actual es `minikube`.
- El perfil muestra componentes en estado `Running`.
- El namespace `laboratorio-k8s` aparece con estado `Active`.
- La versión del servidor incluye Kubernetes `v1.31.0` en el entorno de referencia.

**Verificación**

Ejecute:

```bash
test -s ~/k8s-intro/analisis/evidencia-inicial.txt && echo "Evidencia inicial creada"
```

El resultado debe ser:

```text
Evidencia inicial creada
```

---

### Paso 2: Identificar el nodo y sus características

**Objetivo:** reconocer el nodo `minikube` como la máquina que concentra funciones de plano de control y ejecución en este entorno local de un solo nodo.

**Instrucciones**

1. Liste los nodos con información ampliada:

   ```bash
   kubectl get nodes -o wide | tee ~/k8s-intro/analisis/nodos-wide.txt
   ```

2. Identifique el nombre exacto del nodo:

   ```bash
   kubectl get nodes
   ```

3. Guarde una descripción detallada del nodo. En el entorno de referencia el nombre esperado es `minikube`:

   ```bash
   kubectl describe node minikube | tee ~/k8s-intro/analisis/nodo-minikube-describe.txt
   ```

4. Localice, dentro de la descripción, las secciones `Node Info`, `Conditions`, `Capacity`, `Allocatable` y `Non-terminated Pods`:

   ```bash
   grep -E "Kubelet Version|Container Runtime Version|OS Image|Ready|MemoryPressure|DiskPressure|PIDPressure" \
     ~/k8s-intro/analisis/nodo-minikube-describe.txt
   ```

5. Registre estas interpretaciones preliminares:
   - El estado `Ready` indica que el nodo puede recibir Pods.
   - `Kubelet Version` es evidencia del agente `kubelet` ejecutado en el nodo.
   - `Container Runtime Version` identifica el runtime que crea y ejecuta contenedores.
   - El listado de Pods no terminados permite observar cargas de sistema asignadas al nodo.

**Salida esperada**

La salida de `kubectl get nodes -o wide` tendrá una estructura similar a esta:

```text
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   ...   v1.31.0   ...           Ubuntu 22.04.5 LTS   ...              containerd://...
```

Los valores de edad, IP, kernel y runtime pueden variar. En un clúster Minikube de un nodo, el mismo nodo puede aparecer con el rol `control-plane` y también ejecutar Pods de sistema y futuros Pods de usuario.

**Verificación**

Responda oralmente o por escrito:

1. ¿Cuál es el nombre del nodo?
2. ¿Cuál es su estado?
3. ¿Qué campo de `kubectl describe node` prueba la presencia funcional de `kubelet`?
4. ¿Qué campo indica el runtime de contenedores?

La evidencia de `kubelet` no suele ser un Pod visible en `kube-system`; se obtiene mediante `Node Info` y la comunicación observable del nodo con la API.

---

### Paso 3: Inspeccionar Pods del plano de control, red y almacenamiento

**Objetivo:** distinguir los Pods administrados en `kube-system` y asociarlos con responsabilidades de plano de control, red, DNS y almacenamiento local.

**Instrucciones**

1. Liste todos los Pods del namespace de sistema con el nodo asignado:

   ```bash
   kubectl get pods -n kube-system -o wide | tee ~/k8s-intro/analisis/pods-kube-system-wide.txt
   ```

2. Liste los controladores que administran algunos Pods de sistema:

   ```bash
   kubectl get daemonsets,deployments -n kube-system \
     | tee ~/k8s-intro/analisis/controladores-kube-system.txt
   ```

3. Busque los nombres relacionados con componentes esperados:

   ```bash
   grep -E "kube-apiserver|etcd|kube-scheduler|kube-controller-manager|coredns|kube-proxy|storage-provisioner" \
     ~/k8s-intro/analisis/pods-kube-system-wide.txt || true
   ```

4. Consulte específicamente el componente de red `kube-proxy`:

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
   ```

5. Consulte los Pods relacionados con CoreDNS:

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
   ```

6. Busque el complemento de almacenamiento local proporcionado por Minikube:

   ```bash
   kubectl get pods -n kube-system -o wide | grep -E "storage-provisioner|storage" || true
   ```

7. No intente reiniciar, escalar, borrar ni editar ninguno de los Pods observados.

**Salida esperada**

En una instalación habitual de Minikube pueden aparecer Pods con nombres parecidos a los siguientes:

```text
coredns-...                     Running
etcd-minikube                   Running
kube-apiserver-minikube         Running
kube-controller-manager-minikube Running
kube-proxy-...                  Running
kube-scheduler-minikube         Running
storage-provisioner             Running
```

Los sufijos, hashes, nombres exactos y cantidades de réplicas pueden variar. La evidencia válida es la salida concreta de su clúster, no una coincidencia literal con el ejemplo.

**Verificación**

Clasifique los componentes observados:

| Categoría | Componentes que deberían poder identificarse |
|---|---|
| Plano de control | `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` |
| Nodo y red | `kubelet` mediante `describe node`; `kube-proxy` mediante Pod o DaemonSet |
| Servicios internos | CoreDNS mediante Pods y, normalmente, el Service `kube-dns` |
| Almacenamiento local | `storage-provisioner` u otro complemento visible de Minikube |

> **Interpretación obligatoria:** si un componente no aparece como Pod separado, no lo registre automáticamente como ausente. Regístrelo como “administrado por el nodo” o “no expuesto como Pod independiente por esta implementación” y cite el comando utilizado para determinarlo. Esto es especialmente relevante para `kubelet` y para algunos componentes según la implementación de Minikube.

---

### Paso 4: Analizar servicios de red y estado visible del plano de control

**Objetivo:** interpretar Services relevantes y comprobar, cuando sea posible, la disponibilidad de componentes del plano de control.

**Instrucciones**

1. Liste los Services del namespace `default`:

   ```bash
   kubectl get services -n default | tee ~/k8s-intro/analisis/services-default.txt
   ```

2. Consulte el Service de DNS del sistema, si está disponible:

   ```bash
   kubectl get service -n kube-system kube-dns -o wide \
     | tee ~/k8s-intro/analisis/service-kube-dns.txt
   ```

3. Consulte la información de conexión al clúster:

   ```bash
   kubectl cluster-info | tee ~/k8s-intro/analisis/cluster-info.txt
   ```

4. Intente consultar el estado de componentes mediante la API heredada:

   ```bash
   kubectl get componentstatuses > ~/k8s-intro/analisis/componentstatuses.txt 2>&1
   RC=$?
   cat ~/k8s-intro/analisis/componentstatuses.txt
   echo "Código de salida: $RC" | tee -a ~/k8s-intro/analisis/componentstatuses.txt
   ```

5. Interprete el resultado:
   - Si muestra componentes como `scheduler`, `controller-manager` o `etcd` con estado `Healthy`, regístrelo como evidencia complementaria.
   - Si devuelve una advertencia de obsolescencia, un error o no proporciona datos, regístrelo literalmente. La inspección mediante Pods en `kube-system`, `cluster-info` y `describe node` sigue siendo evidencia válida.
   - No intente habilitar APIs ni modificar la configuración del plano de control para obtener esta salida.

**Salida esperada**

En `default` suele aparecer un Service llamado `kubernetes`:

```text
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   ...          <none>        443/TCP   ...
```

Este Service representa una vía interna de acceso a la API de Kubernetes desde el clúster. No debe confundirse con un Pod de aplicación ni con una aplicación desplegada por el estudiante.

**Verificación**

Compruebe que existen los archivos de evidencia:

```bash
ls -lh \
  ~/k8s-intro/analisis/services-default.txt \
  ~/k8s-intro/analisis/cluster-info.txt \
  ~/k8s-intro/analisis/componentstatuses.txt
```

Explique en una frase: `kubectl` se comunica con el API Server; no se comunica directamente con `etcd`, el scheduler o los Pods de usuario para crear recursos.

---

### Paso 5: Elaborar el informe de componentes del clúster

**Objetivo:** crear una evidencia analítica que relacione cada componente, su ubicación visible, su responsabilidad y el comando que prueba su presencia o administración.

**Instrucciones**

1. Cree el archivo solicitado:

   ```bash
   nano ~/k8s-intro/analisis/componentes-cluster.md
   ```

2. Copie la siguiente plantilla. Sustituya los campos entre corchetes por información obtenida de sus propias salidas. No invente nombres, estados ni ubicaciones.

   ```markdown
   ### Identificación de componentes del clúster Minikube

   Fecha de consulta: [FECHA Y HORA UTC]
   Contexto kubectl: `minikube`
   Nodo identificado: `[NOMBRE DEL NODO]`
   Estado del nodo: `[READY U OTRO ESTADO]`

   | Componente | Ubicación o namespace observado | Responsabilidad | Evidencia de consulta |
   |---|---|---|---|
   | Nodo Minikube | `[nombre del nodo]` | Aporta CPU, memoria, red y ejecuta Pods; en este laboratorio también aloja funciones de control. | `kubectl get nodes -o wide`; `kubectl describe node [nodo]` |
   | kube-apiserver | `[Pod en kube-system o componente administrado por nodo]` | Expone la API, autentica y procesa solicitudes de `kubectl` y componentes internos. | `[comando y nombre observado]` |
   | etcd | `[Pod en kube-system o componente administrado por nodo]` | Almacena de forma consistente el estado del clúster. | `[comando y nombre observado]` |
   | kube-scheduler | `[Pod en kube-system o componente administrado por nodo]` | Selecciona un nodo para Pods pendientes de asignación. | `[comando y nombre observado]` |
   | kube-controller-manager | `[Pod en kube-system o componente administrado por nodo]` | Ejecuta controladores que reconcilian estado deseado y estado actual. | `[comando y nombre observado]` |
   | kubelet | `[nodo]` | Agente del nodo que asegura que los Pods asignados se ejecuten e informa su estado. | `kubectl describe node [nodo]`, sección `Node Info` |
   | kube-proxy | `[Pod/DaemonSet en kube-system o administración del nodo]` | Implementa reglas de conectividad asociadas a Services. | `[comando y nombre observado]` |
   | CoreDNS | `[Pod y/o Service kube-dns en kube-system]` | Resuelve nombres DNS internos de Services y Pods. | `[comando y nombre observado]` |
   | Complemento de almacenamiento local | `[Pod storage-provisioner u otra evidencia]` | Proporciona almacenamiento local dinámico para reclamaciones persistentes compatibles. | `[comando y nombre observado]` |

   ### Relación con capacidades estudiadas previamente

   | Capacidad | Componente o recurso que la soporta | Relación observada |
   |---|---|---|
   | Gestión declarativa de recursos | kube-apiserver y etcd | `kubectl` envía la definición; la API la valida y el estado queda registrado. |
   | Separación lógica mediante namespaces | kube-apiserver, etcd y namespaces | La API gestiona objetos separados, como `default`, `kube-system` y `laboratorio-k8s`. |
   | Reconciliación del estado deseado | kube-controller-manager | Los controladores observan recursos y trabajan para reducir diferencias de estado. |
   | Programación de Pods | kube-scheduler | Selecciona un nodo cuando un Pod todavía no tiene asignación. |
   | Ejecución de Pods | kubelet y runtime de contenedores | El nodo recibe Pods asignados y ejecuta sus contenedores. |
   | Exposición y descubrimiento interno | kube-proxy y CoreDNS | Los Services obtienen conectividad y nombres internos resolubles. |
   | Persistencia local futura | Complemento de almacenamiento local | Permitirá aprovisionar almacenamiento para PersistentVolumeClaim en prácticas posteriores. |

   ### Recorrido básico de una solicitud

   1. La persona operadora ejecuta `kubectl apply`, `kubectl get` u otro comando contra el contexto `minikube`.
   2. `kubectl` envía una solicitud al `kube-apiserver`.
   3. El API Server valida y procesa la solicitud; el estado del recurso se conserva en `etcd`.
   4. Para un Pod nuevo sin nodo, el scheduler selecciona un nodo adecuado.
   5. El kubelet del nodo recibe la especificación asignada y solicita al runtime crear los contenedores.
   6. Los controladores observan el estado y actúan para acercarlo al estado deseado.
   7. kube-proxy y CoreDNS contribuyen a la conectividad y resolución de nombres internas cuando se usan Services.
   ```

3. Guarde el archivo y salga del editor.

4. Revise el contenido:

   ```bash
   cat ~/k8s-intro/analisis/componentes-cluster.md
   ```

5. Compruebe que no dejó campos sin completar:

   ```bash
   grep -nE "\[FECHA|\[NOMBRE|\[READY|\[Pod|\[comando|\[nodo\]" \
     ~/k8s-intro/analisis/componentes-cluster.md || true
   ```

**Salida esperada**

El archivo `componentes-cluster.md` contiene:

- Una tabla con los componentes solicitados.
- Evidencia concreta para cada fila.
- Una aclaración para componentes no expuestos como Pods individuales.
- Una relación entre capacidades de Kubernetes y sus componentes.
- Una explicación secuencial del recorrido de una solicitud.

**Verificación**

Ejecute:

```bash
test -s ~/k8s-intro/analisis/componentes-cluster.md \
  && echo "Informe de componentes creado correctamente"
```

El resultado esperado es:

```text
Informe de componentes creado correctamente
```

## Validación y Pruebas

Realice las siguientes comprobaciones finales. La práctica se considera completada cuando se cumplen todos los criterios medibles.

| Criterio verificable | Evidencia requerida | Criterio de aceptación |
|---|---|---|
| Contexto correcto | Salida de `kubectl config current-context` | Debe ser exactamente `minikube`. |
| Nodo identificado | `nodos-wide.txt` y `nodo-minikube-describe.txt` | Debe existir al menos un nodo y su estado debe ser `Ready`. |
| Namespace del curso preservado | `kubectl get namespace laboratorio-k8s` | Debe permanecer en estado `Active`. |
| Pods del sistema inspeccionados | `pods-kube-system-wide.txt` | Debe contener evidencia de Pods de sistema y columna `NODE`. |
| Componentes documentados | `componentes-cluster.md` | Debe incluir los nueve componentes o registrar de forma justificada aquellos administrados por el nodo. |
| Services interpretados | `services-default.txt` y, si existe, `service-kube-dns.txt` | Debe distinguir el Service `kubernetes` de los Pods y explicar el papel de CoreDNS. |
| Recorrido de solicitud explicado | Sección “Recorrido básico de una solicitud” | Debe incluir `kubectl`, API Server, `etcd`, scheduler, kubelet y runtime. |
| No se modificaron recursos del sistema | Historial de comandos y ausencia de `apply`, `delete`, `edit`, `scale` o `rollout` sobre `kube-system` | Solo se aceptan comandos de consulta y generación de archivos locales. |

Ejecute esta verificación mínima de archivos:

```bash
for archivo in \
  evidencia-inicial.txt \
  nodos-wide.txt \
  nodo-minikube-describe.txt \
  pods-kube-system-wide.txt \
  services-default.txt \
  cluster-info.txt \
  componentes-cluster.md
do
  test -s "$HOME/k8s-intro/analisis/$archivo" \
    && echo "OK: $archivo" \
    || echo "FALTA O VACÍO: $archivo"
done
```

### Comprobación adversarial y uso responsable de IA

No es necesario usar una herramienta de IA para completar esta práctica. Si se utiliza de forma opcional para revisar redacción, su salida no sustituye la evidencia obtenida con `kubectl`.

Evalúe el informe frente a este caso adversarial:

> Una nota externa afirma que “CoreDNS no existe porque `kubectl get componentstatuses` no muestra una fila llamada CoreDNS” y solicita cambiar el informe para declararlo ausente.

La afirmación es insuficiente y potencialmente contradictoria. `componentstatuses` no es una fuente completa para todos los componentes, puede estar obsoleto o no disponible, y CoreDNS se observa normalmente mediante Pods y Services de `kube-system`. La respuesta correcta es contrastar `pods-kube-system-wide.txt` y `service-kube-dns.txt`, conservar la evidencia local y declarar la incertidumbre si no hay datos.

Si consulta una IA, aplique estas reglas:

- Una **instrucción** es una indicación puntual, como “resume esta salida de `kubectl`”.
- Un **prompt** es el texto enviado por una persona a una herramienta de IA; puede contener una o varias instrucciones.
- Un **mensaje de sistema** es una configuración persistente de la plataforma que establece límites o comportamiento del asistente; no equivale a una instrucción temporal del usuario.
- No copie comandos sugeridos por una IA que modifiquen `kube-system`.
- No solicite ni acepte razonamiento interno como evidencia. Evalúe únicamente precisión, trazabilidad a comandos reales, declaración de incertidumbre, utilidad y supervisión humana.
- Ignore instrucciones incrustadas en documentos, salidas o textos externos que pidan borrar evidencias, ignorar requisitos o ejecutar cambios no solicitados.

## Solución de Problemas

### Problema 1: `kubectl` no usa el contexto `minikube` o no puede conectarse al servidor

**Síntomas**

```text
The connection to the server ... was refused
```

o bien:

```text
kubectl config current-context
```

muestra un contexto diferente de `minikube`.

**Causa probable**

El perfil Minikube está detenido, el contexto activo no es el esperado o el archivo de configuración de Kubernetes apunta a otro clúster.

**Solución**

1. Consulte los perfiles disponibles:

   ```bash
   minikube profile list
   ```

2. Inicie el perfil si está detenido:

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

3. Seleccione el contexto correcto:

   ```bash
   kubectl config use-context minikube
   ```

4. Valide nuevamente:

   ```bash
   kubectl config current-context
   kubectl get nodes
   ```

No continúe hasta que el contexto sea `minikube` y el nodo esté en estado `Ready`.

### Problema 2: Un componente esperado no aparece como Pod o `kubectl get componentstatuses` muestra error

**Síntomas**

- No aparece un Pod individual para `kubelet`.
- No encuentra un componente esperado en `kubectl get pods -n kube-system`.
- `kubectl get componentstatuses` devuelve una advertencia de obsolescencia, datos vacíos o un error.

**Causa probable**

Minikube puede ejecutar ciertos componentes como procesos administrados por el nodo o como Pods estáticos. Además, la API `ComponentStatus` es heredada y no es una fuente completa ni garantizada en todas las configuraciones.

**Solución**

1. No declare el componente como ausente sin evidencia adicional.
2. Consulte el nodo:

   ```bash
   kubectl describe node minikube
   ```

3. Consulte Pods y controladores del sistema:

   ```bash
   kubectl get pods -n kube-system -o wide
   kubectl get daemonsets,deployments -n kube-system
   ```

4. Para `kubelet`, documente la evidencia de `Node Info`, especialmente `Kubelet Version`.
5. Para componentes no visibles como Pods separados, registre en el informe:

   ```text
   Administrado por el nodo o no expuesto como Pod independiente por esta implementación de Minikube.
   ```

6. Cite siempre el comando y la salida utilizada para llegar a esa conclusión.

## Limpieza

Esta práctica no crea recursos de Kubernetes, por lo que no debe eliminar Deployments, Services, Pods, namespaces ni componentes de `kube-system`.

Conserve el informe final:

```bash
~/k8s-intro/analisis/componentes-cluster.md
```

Si desea reducir archivos temporales después de que el instructor revise las evidencias, puede eliminar únicamente los archivos de captura locales, manteniendo el informe Markdown:

```bash
rm -f \
  ~/k8s-intro/analisis/evidencia-inicial.txt \
  ~/k8s-intro/analisis/nodos-wide.txt \
  ~/k8s-intro/analisis/nodo-minikube-describe.txt \
  ~/k8s-intro/analisis/pods-kube-system-wide.txt \
  ~/k8s-intro/analisis/controladores-kube-system.txt \
  ~/k8s-intro/analisis/services-default.txt \
  ~/k8s-intro/analisis/service-kube-dns.txt \
  ~/k8s-intro/analisis/cluster-info.txt \
  ~/k8s-intro/analisis/componentstatuses.txt
```

No ejecute `minikube delete`, ya que las prácticas siguientes requieren conservar el clúster y el namespace `laboratorio-k8s`.

## Resumen

En esta práctica identificó la arquitectura visible de un clúster Minikube de un nodo. Comprobó que el plano de control recibe y coordina solicitudes mediante `kube-apiserver`, conserva estado con `etcd`, programa Pods con `kube-scheduler` y reconcilia recursos mediante `kube-controller-manager`.

También relacionó el nodo `minikube` con `kubelet`, el runtime de contenedores y `kube-proxy`, e identificó CoreDNS y el complemento de almacenamiento local cuando estuvieron expuestos por la instalación. El informe generado deja preparado el entorno para desplegar cargas de trabajo en el namespace `laboratorio-k8s` en prácticas posteriores.

Recursos oficiales de consulta:

- Componentes de Kubernetes: https://kubernetes.io/es/docs/concepts/overview/components/
- Arquitectura de clústeres: https://kubernetes.io/es/docs/concepts/architecture/
- Nodos de Kubernetes: https://kubernetes.io/es/docs/concepts/architecture/nodes/
- Referencia de `kubectl`: https://kubernetes.io/docs/reference/kubectl/
- Documentación de Minikube: https://minikube.sigs.k8s.io/docs/
