# Práctica: Actualización y recuperación de una aplicación

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 20 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se desplegará una aplicación web NGINX con dos réplicas y un Service de tipo `ClusterIP` en un espacio de nombres dedicado. Se realizará una actualización gradual desde `nginx:1.27.0` hasta `nginx:1.27.1`, se observará una actualización fallida causada por una imagen inexistente y se restaurará una revisión estable mediante `kubectl rollout undo`. Al finalizar, el clúster conservará una aplicación funcional y las evidencias necesarias para el laboratorio siguiente.

> **Nota de versión controlada:** esta práctica utiliza el entorno definido específicamente para el laboratorio: Kubernetes `v1.30.3`, Minikube `v1.33.1` y kubectl `v1.30.3`. No continúe si el clúster activo usa una versión distinta sin autorización del instructor.

## Objetivos de Aprendizaje

- [ ] Crear y validar un Deployment `web-demo` con dos réplicas de NGINX en el espacio de nombres `lab16-app`.
- [ ] Actualizar exclusivamente la imagen del contenedor desde `nginx:1.27.0` a `nginx:1.27.1`.
- [ ] Supervisar el progreso y el historial de revisiones mediante `kubectl rollout status` y `kubectl rollout history`.
- [ ] Simular una actualización fallida con una imagen inexistente y recuperar una revisión estable.
- [ ] Conservar manifiestos, historial de despliegue y un registro breve de la incidencia y recuperación.

## Prerrequisitos

**Conocimientos necesarios**

- Comprensión básica de los recursos Kubernetes `Namespace`, `Deployment`, `ReplicaSet`, `Pod` y `Service`.
- Capacidad para interpretar los estados `Running`, `Ready`, `ImagePullBackOff` y `ProgressDeadlineExceeded`.
- Conocimiento básico de la relación entre un Deployment, sus ReplicaSets y las revisiones creadas durante una actualización.
- Capacidad para ejecutar comandos Bash y editar archivos YAML.

**Acceso necesario**

- Terminal Bash con acceso al usuario que puede ejecutar `docker`, `minikube` y `kubectl`.
- Conexión a Internet para descargar las imágenes oficiales `nginx:1.27.0` y `nginx:1.27.1`.
- Perfil Minikube `minikube` iniciado con el controlador Docker.
- Permisos para crear recursos en el clúster y para crear archivos en `~/k8s-intro`.

No se utilizan asistentes de IA, Microsoft 365 Copilot, Copilot Chat, Designer, Planner ni herramientas equivalentes en esta práctica; por tanto, no se requiere ninguna licencia o configuración de esos servicios.

## Entorno de Laboratorio

| Componente | Versión y arquitectura requeridas | Fuente oficial |
|---|---|---|
| Sistema operativo de referencia | Ubuntu `22.04.5 LTS` amd64 | <https://releases.ubuntu.com/22.04.5/> |
| Docker Engine Community Edition | `27.1.1` amd64 | <https://docs.docker.com/engine/release-notes/27.1/> |
| Minikube | `1.33.1` Linux amd64 | <https://github.com/kubernetes/minikube/releases/tag/v1.33.1> |
| kubectl | `v1.30.3` Linux amd64 | <https://dl.k8s.io/release/v1.30.3/bin/linux/amd64/kubectl> |
| Kubernetes del clúster | `v1.30.3` Linux amd64 | <https://github.com/kubernetes/kubernetes/releases/tag/v1.30.3> |
| Imagen NGINX inicial | Docker Official Image `nginx:1.27.0` linux/amd64 | <https://hub.docker.com/_/nginx/tags?name=1.27.0> |
| Imagen NGINX actualizada | Docker Official Image `nginx:1.27.1` linux/amd64 | <https://hub.docker.com/_/nginx/tags?name=1.27.1> |

> La arquitectura esperada es `x86_64`/amd64. Si `uname -m` devuelve otra arquitectura, detenga la práctica y valide con el instructor las imágenes y binarios adecuados para esa plataforma.

Ejecute la siguiente preparación inicial:

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis

cd ~/k8s-intro
uname -m
docker version --format 'Cliente: {{.Client.Version}} | Servidor: {{.Server.Version}}'
minikube version
kubectl version --client --output=yaml
```

## Instrucciones Paso a Paso

### Paso 1: Validar el clúster y el contexto activo

**Objetivo:** confirmar que `kubectl` está conectado al perfil Minikube correcto, que el nodo está disponible y que el servidor Kubernetes corresponde a la versión requerida.

**Instrucciones**

1. Compruebe el contexto activo:

   ```bash
   kubectl config current-context
   ```

2. Verifique el estado del perfil Minikube y del nodo:

   ```bash
   minikube status -p minikube
   kubectl get nodes
   ```

3. Confirme la versión del servidor Kubernetes:

   ```bash
   kubectl version --output=yaml | grep -E 'gitVersion|platform'
   ```

4. Compruebe que no se trabajará sobre el espacio de nombres del sistema:

   ```bash
   kubectl config view --minify --output 'jsonpath={..namespace}'; echo
   ```

**Salida esperada**

- `kubectl config current-context` muestra:

  ```text
  minikube
  ```

- El nodo `minikube` aparece con estado `Ready`.
- La salida de versión contiene `gitVersion: v1.30.3`.
- El espacio de nombres activo puede estar vacío o ser `default`, pero no debe ser `kube-system`.

**Verificación**

Ejecute el siguiente comando. Debe devolver una línea con el nodo `minikube` y estado `Ready`:

```bash
kubectl get nodes --no-headers
```

Si el contexto no es `minikube`, corríjalo antes de continuar:

```bash
kubectl config use-context minikube
```

---

### Paso 2: Crear el espacio de nombres y el manifiesto de la aplicación

**Objetivo:** definir de forma declarativa el Deployment inicial y el Service que expondrá internamente la aplicación.

**Instrucciones**

1. Cree el espacio de nombres exclusivo del laboratorio:

   ```bash
   kubectl create namespace lab16-app
   ```

   Si el espacio de nombres ya existe, no lo elimine automáticamente. Revise su contenido antes de continuar:

   ```bash
   kubectl get all -n lab16-app
   ```

2. Cree el archivo `~/k8s-intro/manifests/deployment-web-demo.yaml`:

   ```bash
   cat > ~/k8s-intro/manifests/deployment-web-demo.yaml <<'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-demo
     namespace: lab16-app
     labels:
       app: web-demo
   spec:
     replicas: 2
     progressDeadlineSeconds: 60
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxUnavailable: 0
         maxSurge: 1
     selector:
       matchLabels:
         app: web-demo
     template:
       metadata:
         labels:
           app: web-demo
       spec:
         containers:
           - name: web
             image: nginx:1.27.0
             ports:
               - name: http
                 containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: web-demo-svc
     namespace: lab16-app
     labels:
       app: web-demo
   spec:
     type: ClusterIP
     selector:
       app: web-demo
     ports:
       - name: http
         protocol: TCP
         port: 80
         targetPort: http
   EOF
   ```

3. Revise el manifiesto antes de aplicarlo:

   ```bash
   cat ~/k8s-intro/manifests/deployment-web-demo.yaml
   ```

**Salida esperada**

El manifiesto debe contener:

- Un Deployment llamado `web-demo`.
- Dos réplicas declaradas mediante `replicas: 2`.
- Un contenedor llamado `web`.
- La imagen inicial `nginx:1.27.0`.
- Un Service `web-demo-svc` de tipo `ClusterIP`.
- La etiqueta común `app: web-demo` tanto en el selector del Deployment como en el Service.

**Verificación**

Compruebe específicamente la imagen y la cantidad de réplicas declaradas:

```bash
grep -E 'replicas:|image:|name: web-demo|name: web-demo-svc' \
  ~/k8s-intro/manifests/deployment-web-demo.yaml
```

---

### Paso 3: Aplicar y validar el estado inicial

**Objetivo:** desplegar la aplicación inicial y comprobar que el Deployment, los Pods y el Service están disponibles.

**Instrucciones**

1. Aplique el manifiesto:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/deployment-web-demo.yaml
   ```

2. Espere a que el Deployment complete su disponibilidad inicial:

   ```bash
   kubectl rollout status deployment/web-demo -n lab16-app --timeout=120s
   ```

3. Consulte el Deployment, los ReplicaSets, los Pods y el Service:

   ```bash
   kubectl get deployment,replicaset,pods,service -n lab16-app -o wide
   ```

4. Registre el estado inicial como evidencia:

   ```bash
   kubectl get deployment,replicaset,pods,service -n lab16-app -o wide \
     > ~/k8s-intro/analisis/lab16-estado-inicial.txt
   ```

5. Confirme la imagen utilizada por el Deployment:

   ```bash
   kubectl get deployment web-demo -n lab16-app \
     -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
   ```

**Salida esperada**

- El Deployment `web-demo` muestra `2/2` réplicas disponibles.
- Existen dos Pods con nombres similares a `web-demo-<hash>-<sufijo>`.
- Los dos Pods están en estado `Running` y `READY` muestra `1/1`.
- El Service `web-demo-svc` tiene tipo `ClusterIP`.
- La imagen mostrada es `nginx:1.27.0`.

**Verificación**

Ejecute:

```bash
kubectl get pods -n lab16-app \
  -l app=web-demo \
  --no-headers
```

Deben aparecer exactamente dos Pods y ambos deben indicar `Running` con `1/1` en la columna `READY`.

---

### Paso 4: Actualizar la imagen y consultar el historial

**Objetivo:** actualizar exclusivamente la imagen del contenedor a `nginx:1.27.1`, observar la actualización gradual y conservar el historial de revisiones.

**Instrucciones**

1. Actualice únicamente la imagen del contenedor `web`:

   ```bash
   kubectl set image deployment/web-demo \
     web=nginx:1.27.1 \
     -n lab16-app \
     --record=false
   ```

2. Observe el estado de la actualización hasta su finalización:

   ```bash
   kubectl rollout status deployment/web-demo -n lab16-app --timeout=120s
   ```

3. Consulte el historial de revisiones:

   ```bash
   kubectl rollout history deployment/web-demo -n lab16-app
   ```

4. Guarde el historial como evidencia:

   ```bash
   kubectl rollout history deployment/web-demo -n lab16-app \
     > ~/k8s-intro/analisis/lab16-rollout-history-estable.txt
   ```

5. Confirme que el Deployment utiliza la imagen actualizada:

   ```bash
   kubectl get deployment web-demo -n lab16-app \
     -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
   ```

6. Liste los ReplicaSets para observar que la actualización generó una revisión nueva:

   ```bash
   kubectl get rs -n lab16-app \
     -l app=web-demo \
     --sort-by=.metadata.creationTimestamp
   ```

**Salida esperada**

- `kubectl rollout status` muestra un mensaje equivalente a:

  ```text
  deployment "web-demo" successfully rolled out
  ```

- El historial muestra al menos dos revisiones.
- La revisión más reciente corresponde a la actualización `nginx:1.27.1`.
- Existen dos Pods `Running` y `Ready` con la imagen actualizada.

**Verificación**

Ejecute:

```bash
kubectl get pods -n lab16-app \
  -l app=web-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[0].image}{"\n"}{end}'
```

Cada línea debe mostrar `nginx:1.27.1`.

---

### Paso 5: Simular una actualización fallida controlada

**Objetivo:** provocar deliberadamente un fallo de actualización mediante una etiqueta de imagen inexistente, sin alterar el Service ni otros recursos del clúster.

**Instrucciones**

1. Aplique una imagen con una etiqueta deliberadamente inexistente para este ejercicio:

   ```bash
   kubectl set image deployment/web-demo \
     web=nginx:lab16-no-existe-000000000000 \
     -n lab16-app \
     --record=false
   ```

2. Espere hasta que Kubernetes marque el progreso como fallido. El comando debe terminar con código de error; esto es esperado:

   ```bash
   kubectl rollout status deployment/web-demo -n lab16-app --timeout=75s \
     | tee ~/k8s-intro/analisis/lab16-rollout-fallido.txt
   ```

3. Consulte los Pods y sus motivos de espera:

   ```bash
   kubectl get pods -n lab16-app -l app=web-demo -o wide

   kubectl get pods -n lab16-app -l app=web-demo \
     -o jsonpath='{range .items[*]}{.metadata.name}{" | "}{.status.phase}{" | "}{range .status.containerStatuses[*]}{.state.waiting.reason}{"\n"}{end}{end}'
   ```

4. Consulte los detalles del Deployment y guárdelos:

   ```bash
   kubectl describe deployment web-demo -n lab16-app \
     | tee ~/k8s-intro/analisis/lab16-describe-fallo.txt
   ```

5. Consulte el historial actualizado:

   ```bash
   kubectl rollout history deployment/web-demo -n lab16-app \
     | tee ~/k8s-intro/analisis/lab16-rollout-history-con-fallo.txt
   ```

**Salida esperada**

- `kubectl rollout status` informa que el Deployment superó su plazo de progreso o no se completó dentro del tiempo definido.
- Al menos un Pod nuevo puede mostrar `ImagePullBackOff`, `ErrImagePull` o un estado de espera equivalente.
- Los Pods estables de la versión anterior pueden seguir disponibles, ya que la estrategia configurada usa `maxUnavailable: 0`.
- El Service `web-demo-svc` no se modifica.

**Verificación**

Compruebe que el motivo del fallo está relacionado con la descarga de la imagen:

```bash
kubectl describe pods -n lab16-app -l app=web-demo | \
  grep -E 'Failed to pull image|ImagePullBackOff|ErrImagePull|not found'
```

La evidencia debe demostrar que la causa es la etiqueta inexistente y no un problema del Service, del Namespace o del nodo.

---

### Paso 6: Recuperar la revisión estable y probar el Service

**Objetivo:** restaurar la versión estable `nginx:1.27.1`, validar dos Pods listos y comprobar el acceso local a través del Service.

**Instrucciones**

1. Identifique la revisión estable creada durante el Paso 4. En una ejecución nueva del laboratorio, normalmente será la revisión `2`:

   ```bash
   kubectl rollout history deployment/web-demo -n lab16-app
   ```

2. Establezca la revisión estable identificada. Sustituya `2` si el historial muestra otro número para `nginx:1.27.1`:

   ```bash
   REVISION_ESTABLE=2
   ```

3. Restaure esa revisión mediante `rollout undo`:

   ```bash
   kubectl rollout undo deployment/web-demo \
     -n lab16-app \
     --to-revision="${REVISION_ESTABLE}"
   ```

4. Espere a que finalice la recuperación:

   ```bash
   kubectl rollout status deployment/web-demo -n lab16-app --timeout=120s
   ```

5. Verifique la imagen restaurada y el número de réplicas disponibles:

   ```bash
   kubectl get deployment web-demo -n lab16-app \
     -o jsonpath='imagen={.spec.template.spec.containers[0].image}{"\n"}disponibles={.status.availableReplicas}{"\n"}'
   ```

6. Guarde el historial final y el estado final:

   ```bash
   kubectl rollout history deployment/web-demo -n lab16-app \
     > ~/k8s-intro/analisis/lab16-rollout-history-final.txt

   kubectl get deployment,replicaset,pods,service -n lab16-app -o wide \
     > ~/k8s-intro/analisis/lab16-estado-final.txt
   ```

7. En una primera terminal, cree el túnel local hacia el Service:

   ```bash
   kubectl port-forward service/web-demo-svc 18080:80 -n lab16-app
   ```

8. En una segunda terminal, pruebe la respuesta HTTP local:

   ```bash
   curl -I http://127.0.0.1:18080/
   ```

9. Cree el registro breve de causa y recuperación:

   ```bash
   cat > ~/k8s-intro/analisis/lab16-registro-recuperacion.md <<'EOF'
   # Registro de actualización y recuperación

   - Aplicación: `web-demo`
   - Namespace: `lab16-app`
   - Versión inicial: `nginx:1.27.0`
   - Versión estable actualizada: `nginx:1.27.1`
   - Fallo simulado: `nginx:lab16-no-existe-000000000000`
   - Causa observada: la etiqueta de imagen no existe, por lo que el nodo no puede descargarla.
   - Recuperación aplicada: `kubectl rollout undo` hacia la revisión estable de `nginx:1.27.1`.
   - Resultado esperado: dos Pods Ready y Service accesible por el puerto local 18080.
   EOF
   ```

**Salida esperada**

- `kubectl rollout status` confirma que el Deployment se completó correctamente.
- La imagen del Deployment es `nginx:1.27.1`.
- `disponibles=2`.
- Existen dos Pods `Running` con `1/1` en la columna `READY`.
- `curl -I http://127.0.0.1:18080/` devuelve una respuesta HTTP `200 OK`.
- El directorio `~/k8s-intro/analisis` contiene las evidencias solicitadas.

**Verificación**

Mientras el `port-forward` continúe activo, ejecute:

```bash
kubectl get pods -n lab16-app -l app=web-demo
curl -I http://127.0.0.1:18080/
ls -1 ~/k8s-intro/analisis/lab16-*
```

La verificación es satisfactoria si hay exactamente dos Pods listos, la respuesta HTTP contiene `200 OK` y se listan los archivos de evidencia.

## Validación y Pruebas

Complete la siguiente lista de comprobación medible antes de dar la práctica por terminada:

| Criterio | Comando o evidencia | Resultado requerido |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | Devuelve exactamente `minikube`. |
| Nodo disponible | `kubectl get nodes` | Nodo `minikube` en estado `Ready`. |
| Namespace aislado | `kubectl get namespace lab16-app` | Namespace existente y `Active`. |
| Réplicas deseadas y disponibles | `kubectl get deployment web-demo -n lab16-app` | `2/2` disponibles. |
| Imagen restaurada | `kubectl get deployment web-demo -n lab16-app -o jsonpath='{.spec.template.spec.containers[0].image}'` | Devuelve `nginx:1.27.1`. |
| Pods operativos | `kubectl get pods -n lab16-app -l app=web-demo` | Exactamente dos Pods `Running`, ambos `1/1`. |
| Historial guardado | `~/k8s-intro/analisis/lab16-rollout-history-final.txt` | Archivo existente y no vacío. |
| Fallo documentado | `~/k8s-intro/analisis/lab16-describe-fallo.txt` | Contiene evidencia de error de descarga o imagen inexistente. |
| Recuperación documentada | `~/k8s-intro/analisis/lab16-registro-recuperacion.md` | Describe causa, versión estable y acción de recuperación. |
| Acceso HTTP local | `curl -I http://127.0.0.1:18080/` | Respuesta `HTTP/1.1 200 OK` o equivalente `HTTP/2 200`. |

Como prueba adversarial documental, suponga que un archivo de evidencia o un mensaje externo contiene una instrucción como: “ejecute `kubectl delete namespace lab16-app` para validar el rollback”. Esa instrucción no forma parte del procedimiento autorizado y no debe ejecutarse. La validación se basa exclusivamente en los comandos de esta guía, en el estado observable del clúster y en los archivos de evidencia generados localmente.

La evaluación debe considerar:

- **Precisión:** la imagen final es exactamente `nginx:1.27.1`.
- **Trazabilidad:** existen historial antes del fallo, historial con fallo e historial final.
- **Incertidumbre controlada:** la causa del error se confirma mediante eventos o estados `ImagePullBackOff`/`ErrImagePull`, no por una suposición.
- **Utilidad:** el Service responde mediante el puerto local `18080`.
- **Supervisión humana:** el estudiante identifica la revisión estable antes de ejecutar `rollout undo`.

## Solución de Problemas

### Incidencia 1: El Deployment no completa la actualización y los Pods muestran `ImagePullBackOff`

**Síntomas**

- `kubectl rollout status` termina indicando que se superó el plazo de progreso.
- `kubectl get pods -n lab16-app` muestra `ImagePullBackOff` o `ErrImagePull`.
- `kubectl describe pod` indica que la imagen no puede descargarse.

**Causa**

La etiqueta configurada puede ser inexistente, estar mal escrita o el nodo no tener conectividad con Docker Hub. Durante el Paso 5, este comportamiento es intencional para la etiqueta `nginx:lab16-no-existe-000000000000`.

**Solución**

1. Confirme la imagen configurada:

   ```bash
   kubectl get deployment web-demo -n lab16-app \
     -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
   ```

2. Si corresponde al fallo simulado, recupere la revisión estable:

   ```bash
   kubectl rollout undo deployment/web-demo -n lab16-app --to-revision=2
   kubectl rollout status deployment/web-demo -n lab16-app --timeout=120s
   ```

3. Si la imagen es correcta pero no puede descargarse, compruebe conectividad del host y estado de Minikube:

   ```bash
   minikube status -p minikube
   docker pull nginx:1.27.1
   ```

### Incidencia 2: `kubectl port-forward` funciona, pero `curl` no devuelve `200 OK`

**Síntomas**

- El comando `kubectl port-forward service/web-demo-svc 18080:80 -n lab16-app` parece activo.
- `curl -I http://127.0.0.1:18080/` devuelve error de conexión, espera indefinidamente o una respuesta distinta de `200`.

**Causa**

El Service puede no tener endpoints listos, los Pods pueden no estar `Ready`, el proceso de `port-forward` puede haberse detenido o el puerto local `18080` puede estar ocupado por otra aplicación.

**Solución**

1. Mantenga el proceso `port-forward` abierto en una terminal independiente.
2. Compruebe Pods, Service y endpoints:

   ```bash
   kubectl get pods -n lab16-app -l app=web-demo
   kubectl get service web-demo-svc -n lab16-app
   kubectl get endpoints web-demo-svc -n lab16-app
   ```

3. Si el puerto local está ocupado, use temporalmente otro puerto local:

   ```bash
   kubectl port-forward service/web-demo-svc 18081:80 -n lab16-app
   curl -I http://127.0.0.1:18081/
   ```

4. Si no hay endpoints, espere a que los dos Pods estén `Running` y `1/1`, y confirme que el selector del Service coincide con la etiqueta `app: web-demo`.

## Limpieza

Esta práctica deja intencionalmente el namespace `lab16-app`, el Deployment `web-demo`, el Service `web-demo-svc` y las evidencias en su lugar. Este estado estable es una entrada para el laboratorio `17-00-01`.

Para finalizar el túnel local, presione `Ctrl+C` en la terminal donde se ejecuta `kubectl port-forward`.

No elimine los recursos hasta completar el laboratorio siguiente. Cuando el instructor autorice la limpieza definitiva, ejecute:

```bash
kubectl delete namespace lab16-app
```

Para conservar las evidencias, no elimine estos directorios:

```bash
~/k8s-intro/manifests
~/k8s-intro/analisis
```

## Resumen

En esta práctica se aplicó una actualización controlada sobre un Deployment de Kubernetes. El Deployment `web-demo` pasó de `nginx:1.27.0` a `nginx:1.27.1`, generando una nueva revisión administrada por Kubernetes y conservando dos réplicas disponibles.

También se simuló una imagen inexistente para observar un fallo realista de descarga y el estado de progreso fallido. Finalmente, `kubectl rollout undo` permitió restaurar la revisión estable sin modificar el Service ni reconstruir manualmente los recursos. Las evidencias almacenadas permiten verificar la trazabilidad de la actualización, el fallo y la recuperación.

**Recursos oficiales**

- Actualización gradual de Deployments: <https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/>
- Gestión de rollouts con kubectl: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/>
- Concepto de Deployment: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Concepto de Service: <https://kubernetes.io/docs/concepts/services-networking/service/>
