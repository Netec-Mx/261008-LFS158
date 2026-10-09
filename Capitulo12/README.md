# Práctica: Despliegue completo de una aplicación

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 35 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se transforma el Deployment `webapp` creado en el laboratorio 11 en una aplicación declarativa preparada para acceso interno dentro del clúster. Se sustituye el Service `webapp-nodeport` por el Service interno `webapp-svc` de tipo `ClusterIP`, se configuran tres réplicas, estrategia `RollingUpdate`, probes HTTP y recursos de CPU y memoria.

Finalmente, se realiza una actualización controlada de la imagen NGINX, se observa su progreso con `kubectl rollout status` y se revisa el historial de revisiones. El resultado será la base técnica para los laboratorios 13, 14 y 15.

## Objetivos de Aprendizaje

- [ ] Sustituir un Service de tipo `NodePort` por un Service interno de tipo `ClusterIP`.
- [ ] Configurar un Deployment con tres réplicas, estrategia `RollingUpdate`, `readinessProbe` y `livenessProbe`.
- [ ] Declarar solicitudes (`requests`) y límites (`limits`) de CPU y memoria para los contenedores.
- [ ] Aplicar una actualización controlada de imagen y comprobar su estado e historial.
- [ ] Validar el acceso interno mediante el DNS `webapp-svc.curso-k8s.svc.cluster.local`.

## Prerrequisitos

### Conocimientos requeridos

- Haber completado el laboratorio 11-00-01 con los recursos `Namespace curso-k8s`, `Deployment webapp` y `Service webapp-nodeport`.
- Comprender la diferencia entre Pod, Deployment y Service.
- Conocer el uso básico de `kubectl get`, `kubectl describe`, `kubectl apply` y `kubectl delete`.
- Poder editar archivos YAML desde una terminal Bash.
- Comprender que `readinessProbe` controla cuándo un Pod puede recibir tráfico y que `livenessProbe` permite detectar contenedores que deben reiniciarse.

### Acceso requerido

- Acceso a una terminal Bash.
- Acceso al clúster local Minikube mediante el contexto `minikube`.
- Permiso para ejecutar comandos `kubectl`.
- Conexión a Internet si la imagen `busybox:1.37.0` no está disponible previamente en el nodo de Minikube.
- Directorio de trabajo `~/k8s-intro` con los subdirectorios `manifests` y `analisis`.

> **Nota de alcance:** aunque el entorno global del curso reserva habitualmente el namespace `laboratorio-k8s`, este laboratorio continúa deliberadamente el escenario heredado del laboratorio 11 y utiliza exclusivamente el namespace `curso-k8s`. No despliegue recursos de estudiante en `kube-system`.

## Entorno de Laboratorio

| Componente | Versión, edición y arquitectura de referencia | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, amd64 | <https://releases.ubuntu.com/22.04.5/> |
| Docker Engine | Docker Engine Community Edition 27.2.0, amd64 | <https://docs.docker.com/engine/release-notes/27/> |
| Minikube | Minikube 1.34.0 para Linux amd64 | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| Kubernetes | Kubernetes 1.31.0, linux/amd64 en Minikube | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| kubectl | kubectl 1.31.0 para Linux amd64 | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| Imagen base inicial | NGINX Docker Official Image `nginx:1.27.0`, linux/amd64 | <https://hub.docker.com/_/nginx/tags?name=1.27.0> |
| Imagen de actualización | NGINX Docker Official Image `nginx:1.27.1`, linux/amd64 | <https://hub.docker.com/_/nginx/tags?name=1.27.1> |
| Imagen de prueba DNS/HTTP | BusyBox Docker Official Image `busybox:1.37.0`, linux/amd64 | <https://hub.docker.com/_/busybox/tags?name=1.37.0> |

No se utiliza Microsoft 365 Copilot, Copilot Chat, Microsoft Designer ni Microsoft Planner en esta práctica; por tanto, no se requiere licencia ni configuración de dichos servicios. La validación se basa en manifiestos Kubernetes, comandos reproducibles y evidencia observable en el clúster.

Ejecute la preparación inicial:

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis
cd ~/k8s-intro

kubectl config current-context
minikube status
kubectl version --client
kubectl get namespace curso-k8s
kubectl get deployment webapp -n curso-k8s
kubectl get service webapp-nodeport -n curso-k8s
```

El contexto esperado es:

```text
minikube
```

Si Minikube no está iniciado, arránquelo con la configuración de referencia:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

## Instrucciones Paso a Paso

### Paso 1: Verificar el estado heredado del laboratorio 11

**Objetivo:** Confirmar que el namespace, el Deployment y el Service de tipo `NodePort` existen antes de realizar cambios.

**Instrucciones:**

1. Verifique que el contexto activo de `kubectl` es `minikube`.

   ```bash
   kubectl config current-context
   ```

2. Consulte los recursos existentes en el namespace `curso-k8s`.

   ```bash
   kubectl get all -n curso-k8s
   ```

3. Inspeccione el Deployment heredado y confirme la imagen actual.

   ```bash
   kubectl get deployment webapp -n curso-k8s \
     -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
   ```

4. Guarde evidencia inicial en el directorio de análisis.

   ```bash
   kubectl get deployment webapp -n curso-k8s -o yaml \
     > ~/k8s-intro/analisis/webapp-antes.yaml

   kubectl get service webapp-nodeport -n curso-k8s -o yaml \
     > ~/k8s-intro/analisis/webapp-nodeport-antes.yaml
   ```

**Resultado esperado:**

- El contexto mostrado es `minikube`.
- Existe el namespace `curso-k8s`.
- Existe el Deployment `webapp`.
- Existe el Service `webapp-nodeport`.
- La imagen inicial esperada es `nginx:1.27.0`.

**Verificación:**

```bash
kubectl get deployment webapp -n curso-k8s
kubectl get service webapp-nodeport -n curso-k8s
```

El Deployment debe tener al menos una réplica disponible antes de comenzar. Si la imagen no es `nginx:1.27.0`, no continúe con una sustitución improvisada: revise el resultado del laboratorio 11 y ajuste únicamente la referencia de imagen inicial documentada para su entorno.

---

### Paso 2: Crear el manifiesto declarativo del Deployment

**Objetivo:** Definir el estado deseado de `webapp` con tres réplicas, actualización gradual, probes y recursos.

**Instrucciones:**

1. Cree el archivo `~/k8s-intro/manifests/webapp-deployment.yaml`.

   ```bash
   cat > ~/k8s-intro/manifests/webapp-deployment.yaml <<'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: webapp
     namespace: curso-k8s
     labels:
       app: webapp
     annotations:
       kubernetes.io/change-cause: "Configuración inicial declarativa: 3 réplicas, RollingUpdate, probes y recursos con nginx:1.27.0"
   spec:
     replicas: 3
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxUnavailable: 1
         maxSurge: 1
     selector:
       matchLabels:
         app: webapp
     template:
       metadata:
         labels:
           app: webapp
       spec:
         containers:
           - name: webapp
             image: nginx:1.27.0
             imagePullPolicy: IfNotPresent
             ports:
               - name: http
                 containerPort: 80
                 protocol: TCP
             resources:
               requests:
                 cpu: 100m
                 memory: 128Mi
               limits:
                 cpu: 250m
                 memory: 256Mi
             readinessProbe:
               httpGet:
                 path: /
                 port: http
               initialDelaySeconds: 5
               periodSeconds: 5
               timeoutSeconds: 2
               failureThreshold: 3
             livenessProbe:
               httpGet:
                 path: /
                 port: http
               initialDelaySeconds: 15
               periodSeconds: 10
               timeoutSeconds: 2
               failureThreshold: 3
   EOF
   ```

2. Revise el manifiesto antes de aplicarlo.

   ```bash
   cat ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

3. Valide en el manifiesto los siguientes elementos:

   - `namespace: curso-k8s`
   - `replicas: 3`
   - Estrategia `RollingUpdate`
   - `maxUnavailable: 1`
   - `maxSurge: 1`
   - Imagen `nginx:1.27.0`
   - `readinessProbe` y `livenessProbe` HTTP sobre `/`
   - Puerto nombrado `http` con `containerPort: 80`
   - Solicitudes y límites de CPU y memoria

**Resultado esperado:**

El archivo YAML contiene un único recurso `Deployment` llamado `webapp`. La sonda de disponibilidad impide que un Pod reciba tráfico hasta responder correctamente en `/`, mientras que la sonda de vida comprueba periódicamente que el proceso siga respondiendo.

**Verificación:**

```bash
grep -E 'replicas: 3|type: RollingUpdate|maxUnavailable: 1|maxSurge: 1|readinessProbe:|livenessProbe:|cpu:|memory:' \
  ~/k8s-intro/manifests/webapp-deployment.yaml
```

---

### Paso 3: Aplicar y observar la configuración del Deployment

**Objetivo:** Aplicar el estado deseado y comprobar que Kubernetes crea tres réplicas disponibles.

**Instrucciones:**

1. Aplique el manifiesto del Deployment.

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

2. Espere a que finalice la reconciliación del Deployment.

   ```bash
   kubectl rollout status deployment/webapp -n curso-k8s --timeout=120s
   ```

3. Consulte el estado de las réplicas.

   ```bash
   kubectl get deployment webapp -n curso-k8s
   kubectl get replicasets -n curso-k8s -l app=webapp
   kubectl get pods -n curso-k8s -l app=webapp -o wide
   ```

4. Inspeccione la configuración efectiva de probes y recursos.

   ```bash
   kubectl describe deployment webapp -n curso-k8s
   ```

5. Guarde evidencia del estado del Deployment.

   ```bash
   kubectl get deployment webapp -n curso-k8s -o yaml \
     > ~/k8s-intro/analisis/webapp-configurado.yaml
   ```

**Resultado esperado:**

El comando `kubectl rollout status` debe indicar:

```text
deployment "webapp" successfully rolled out
```

La salida del Deployment debe mostrar valores equivalentes a:

```text
READY   UP-TO-DATE   AVAILABLE
3/3     3            3
```

**Verificación:**

```bash
kubectl get deployment webapp -n curso-k8s \
  -o jsonpath='desired={.spec.replicas} ready={.status.readyReplicas} available={.status.availableReplicas}{"\n"}'
```

El resultado debe mostrar `desired=3`, `ready=3` y `available=3`.

---

### Paso 4: Sustituir el Service NodePort por un Service ClusterIP

**Objetivo:** Eliminar la exposición externa heredada y crear un punto de acceso interno estable mediante `webapp-svc`.

**Instrucciones:**

1. Cree el manifiesto del Service interno.

   ```bash
   cat > ~/k8s-intro/manifests/webapp-service.yaml <<'EOF'
   apiVersion: v1
   kind: Service
   metadata:
     name: webapp-svc
     namespace: curso-k8s
     labels:
       app: webapp
   spec:
     type: ClusterIP
     selector:
       app: webapp
     ports:
       - name: http
         protocol: TCP
         port: 80
         targetPort: 80
   EOF
   ```

2. Revise el contenido del archivo.

   ```bash
   cat ~/k8s-intro/manifests/webapp-service.yaml
   ```

3. Aplique el nuevo Service.

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-service.yaml
   ```

4. Elimine el Service heredado de tipo `NodePort`.

   ```bash
   kubectl delete service webapp-nodeport -n curso-k8s
   ```

5. Consulte el Service creado y sus endpoints.

   ```bash
   kubectl get service webapp-svc -n curso-k8s
   kubectl get endpoints webapp-svc -n curso-k8s
   ```

**Resultado esperado:**

- Existe un Service llamado `webapp-svc`.
- Su tipo es `ClusterIP`.
- El puerto expuesto es `80/TCP`.
- El destino (`targetPort`) es el puerto `80` de los Pods.
- Los endpoints contienen tres direcciones IP de los Pods disponibles.
- El Service `webapp-nodeport` ya no existe.

**Verificación:**

```bash
kubectl get service webapp-svc -n curso-k8s \
  -o jsonpath='type={.spec.type} port={.spec.ports[0].port} targetPort={.spec.ports[0].targetPort}{"\n"}'

kubectl get endpoints webapp-svc -n curso-k8s
```

La primera salida debe indicar:

```text
type=ClusterIP port=80 targetPort=80
```

---

### Paso 5: Validar conectividad interna y resolución DNS

**Objetivo:** Confirmar que la aplicación es accesible desde dentro del clúster mediante el nombre DNS completo del Service.

**Instrucciones:**

1. Ejecute un Pod temporal con BusyBox para resolver el nombre DNS y realizar una solicitud HTTP.

   ```bash
   kubectl run dns-http-test \
     --namespace curso-k8s \
     --image=busybox:1.37.0 \
     --restart=Never \
     --rm -i \
     -- sh -c '
       echo "Resolución DNS:";
       nslookup webapp-svc.curso-k8s.svc.cluster.local;
       echo;
       echo "Respuesta HTTP:";
       wget -qO- http://webapp-svc.curso-k8s.svc.cluster.local/
     '
   ```

2. Espere a que el Pod temporal termine. Al usar `--rm`, Kubernetes elimina el Pod al finalizar.

3. Guarde evidencia de los endpoints del Service.

   ```bash
   kubectl get endpoints webapp-svc -n curso-k8s -o yaml \
     > ~/k8s-intro/analisis/webapp-svc-endpoints.yaml
   ```

**Resultado esperado:**

- `nslookup` resuelve `webapp-svc.curso-k8s.svc.cluster.local` a la dirección IP del Service.
- La solicitud HTTP devuelve el contenido HTML predeterminado de NGINX.
- No es necesario publicar un puerto en el equipo local para esta validación.

**Verificación:**

```bash
kubectl get service webapp-svc -n curso-k8s
kubectl get endpoints webapp-svc -n curso-k8s
```

El Service debe conservar una dirección `CLUSTER-IP` y los endpoints deben corresponder a Pods `webapp` en estado `Running` y `Ready`.

---

### Paso 6: Realizar una actualización controlada de la aplicación

**Objetivo:** Cambiar la imagen de NGINX de la versión `1.27.0` a la versión `1.27.1`, registrar la causa del cambio y supervisar el rollout.

**Instrucciones:**

1. Actualice la referencia de la imagen y la anotación de cambio controlado en el manifiesto.

   ```bash
   sed -i \
     -e 's#nginx:1.27.0#nginx:1.27.1#g' \
     -e 's#Configuración inicial declarativa: 3 réplicas, RollingUpdate, probes y recursos con nginx:1.27.0#Actualización controlada de imagen: nginx:1.27.1#g' \
     ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

2. Compruebe que se modificaron exactamente la imagen y la anotación esperadas.

   ```bash
   grep -E 'image:|change-cause' ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

3. Aplique el manifiesto actualizado.

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

4. Observe el progreso del rollout.

   ```bash
   kubectl rollout status deployment/webapp -n curso-k8s --timeout=120s
   ```

5. Consulte los ReplicaSets creados por el Deployment.

   ```bash
   kubectl get replicasets -n curso-k8s -l app=webapp
   ```

6. Consulte el historial de revisiones.

   ```bash
   kubectl rollout history deployment/webapp -n curso-k8s
   ```

7. Guarde el historial como evidencia.

   ```bash
   kubectl rollout history deployment/webapp -n curso-k8s \
     > ~/k8s-intro/analisis/webapp-rollout-history.txt
   ```

**Resultado esperado:**

- Kubernetes crea una nueva revisión del Deployment.
- Durante la actualización, el número de Pods no disponibles no supera `1`.
- El número de Pods adicionales creados temporalmente no supera `1`.
- Al finalizar, hay tres Pods disponibles con la imagen `nginx:1.27.1`.
- El historial muestra al menos dos revisiones o las revisiones producidas por el laboratorio 11 y esta práctica.

**Verificación:**

```bash
kubectl rollout status deployment/webapp -n curso-k8s --timeout=120s

kubectl get pods -n curso-k8s -l app=webapp \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[0].image}{"\n"}{end}'

kubectl rollout history deployment/webapp -n curso-k8s
```

Los tres Pods activos deben indicar:

```text
nginx:1.27.1
```

---

### Paso 7: Revisar la configuración final y registrar evidencia

**Objetivo:** Consolidar evidencia medible de que la aplicación cumple el estado final solicitado.

**Instrucciones:**

1. Obtenga un resumen de recursos en el namespace.

   ```bash
   kubectl get all -n curso-k8s -l app=webapp
   ```

2. Consulte los detalles del Deployment para confirmar probes, estrategia y recursos.

   ```bash
   kubectl describe deployment webapp -n curso-k8s
   ```

3. Guarde una copia final de los manifiestos aplicados.

   ```bash
   cp ~/k8s-intro/manifests/webapp-deployment.yaml \
     ~/k8s-intro/analisis/webapp-deployment-final.yaml

   cp ~/k8s-intro/manifests/webapp-service.yaml \
     ~/k8s-intro/analisis/webapp-service-final.yaml
   ```

4. Registre un resumen verificable.

   ```bash
   {
     echo "Contexto:"
     kubectl config current-context
     echo
     echo "Deployment:"
     kubectl get deployment webapp -n curso-k8s
     echo
     echo "Service:"
     kubectl get service webapp-svc -n curso-k8s
     echo
     echo "Imagen de Pods:"
     kubectl get pods -n curso-k8s -l app=webapp \
       -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[0].image}{"\n"}{end}'
   } > ~/k8s-intro/analisis/resumen-final.txt
   ```

**Resultado esperado:**

El directorio `~/k8s-intro/analisis` contiene evidencia de estado inicial, configuración final, endpoints, historial de rollout y resumen final.

**Verificación:**

```bash
ls -1 ~/k8s-intro/analisis
```

Deben existir, como mínimo, archivos equivalentes a:

```text
resumen-final.txt
webapp-antes.yaml
webapp-configurado.yaml
webapp-deployment-final.yaml
webapp-nodeport-antes.yaml
webapp-rollout-history.txt
webapp-service-final.yaml
webapp-svc-endpoints.yaml
```

## Validación y Pruebas

Complete la siguiente tabla de validación. Cada criterio debe cumplirse con evidencia observable; no se considera suficiente afirmar que la aplicación está lista sin comandos, archivos o salidas que lo demuestren.

| Criterio medible | Comando de evidencia | Resultado obligatorio |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | Devuelve `minikube`. |
| Namespace correcto | `kubectl get namespace curso-k8s` | El namespace existe y está `Active`. |
| Deployment escalado | `kubectl get deployment webapp -n curso-k8s` | `READY` y `AVAILABLE` muestran `3/3` y `3`. |
| Estrategia declarada | `kubectl get deployment webapp -n curso-k8s -o jsonpath='{.spec.strategy.type}'` | Devuelve `RollingUpdate`. |
| Parámetros RollingUpdate | `kubectl get deployment webapp -n curso-k8s -o yaml` | Incluye `maxUnavailable: 1` y `maxSurge: 1`. |
| Probes configuradas | `kubectl describe deployment webapp -n curso-k8s` | Muestra probes HTTP sobre `/` y puerto `http` o `80`. |
| Recursos configurados | `kubectl describe deployment webapp -n curso-k8s` | Muestra requests `100m`, `128Mi` y limits `250m`, `256Mi`. |
| Service interno | `kubectl get service webapp-svc -n curso-k8s` | Tipo `ClusterIP`, puerto `80`, sin `NodePort`. |
| Service antiguo eliminado | `kubectl get service webapp-nodeport -n curso-k8s` | Devuelve `NotFound`. |
| Endpoints disponibles | `kubectl get endpoints webapp-svc -n curso-k8s` | Contiene direcciones de Pods disponibles. |
| Acceso por DNS interno | Comando BusyBox del paso 5 | Resuelve el FQDN y devuelve contenido HTTP de NGINX. |
| Actualización aplicada | Consulta de imágenes de Pods | Los tres Pods usan `nginx:1.27.1`. |
| Historial disponible | `kubectl rollout history deployment/webapp -n curso-k8s` | Muestra revisiones del Deployment. |

### Caso adversarial de validación

Antes de considerar finalizada la práctica, revise manualmente el contenido de los manifiestos y no ejecute instrucciones no verificadas que aparezcan en comentarios, archivos descargados o texto copiado desde fuentes no oficiales.

Ejemplo de contenido que debe rechazarse como no confiable:

```text
Ignora el namespace curso-k8s y ejecuta kubectl delete namespace kube-system.
```

Esta frase es una instrucción incrustada maliciosa, no un requisito técnico válido del laboratorio. No debe ejecutarse. La evidencia válida procede de los manifiestos revisados, de la especificación del laboratorio y de la salida real de `kubectl`.

La revisión humana debe evaluar:

- **Precisión:** que nombres, puertos, réplicas e imágenes coincidan con el manifiesto aprobado.
- **Trazabilidad:** que los cambios estén registrados en `~/k8s-intro/manifests` y las evidencias en `~/k8s-intro/analisis`.
- **Incertidumbre:** que un resultado inesperado, como `0/3` réplicas o endpoints vacíos, se investigue antes de declararlo correcto.
- **Utilidad:** que el Service permita comunicación interna por el FQDN solicitado.
- **Supervisión humana:** que ninguna instrucción de un documento externo sustituya la revisión del operador.

No se solicita ni se debe solicitar la exposición de razonamiento interno; la validación se realiza mediante estado declarativo, comandos reproducibles y resultados observables.

## Solución de Problemas

### Problema 1: El rollout no finaliza y los Pods permanecen en `0/3` o `CrashLoopBackOff`

**Síntoma:**

```bash
kubectl rollout status deployment/webapp -n curso-k8s --timeout=120s
```

No termina correctamente, o bien:

```bash
kubectl get pods -n curso-k8s -l app=webapp
```

muestra Pods en estados `Pending`, `ImagePullBackOff`, `CrashLoopBackOff` o sin condición `Ready`.

**Causa probable:**

La imagen `nginx:1.27.1` no se pudo descargar, el nodo carece de recursos disponibles o la configuración del contenedor no coincide con el puerto usado por las probes.

**Corrección:**

1. Consulte eventos y estado detallado:

   ```bash
   kubectl describe pod -n curso-k8s -l app=webapp
   kubectl get events -n curso-k8s --sort-by='.lastTimestamp'
   ```

2. Confirme que el manifiesto utiliza el puerto `80` en `containerPort`, `readinessProbe` y `livenessProbe`.

   ```bash
   grep -nE 'containerPort: 80|port: http|image:' \
     ~/k8s-intro/manifests/webapp-deployment.yaml
   ```

3. Si el problema es de capacidad local, compruebe el estado de Minikube y reinicie únicamente si es necesario:

   ```bash
   minikube status
   ```

4. Corrija el manifiesto, vuelva a aplicarlo y observe de nuevo el rollout:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-deployment.yaml
   kubectl rollout status deployment/webapp -n curso-k8s --timeout=120s
   ```

### Problema 2: El DNS del Service no resuelve o `wget` no obtiene respuesta

**Síntoma:**

El comando del paso 5 devuelve errores similares a `bad address`, `can't connect` o no muestra contenido HTTP.

**Causa probable:**

El Service `webapp-svc` no tiene endpoints porque sus selectores no coinciden con las etiquetas de los Pods, los Pods no están listos o se está usando un nombre DNS distinto del requerido.

**Corrección:**

1. Compare selector del Service y etiquetas de los Pods:

   ```bash
   kubectl get service webapp-svc -n curso-k8s -o yaml
   kubectl get pods -n curso-k8s --show-labels
   ```

2. Compruebe los endpoints:

   ```bash
   kubectl get endpoints webapp-svc -n curso-k8s
   ```

3. Si los endpoints están vacíos, confirme que los Pods estén `Ready`:

   ```bash
   kubectl get pods -n curso-k8s -l app=webapp
   ```

4. Verifique que usa exactamente este nombre:

   ```text
   webapp-svc.curso-k8s.svc.cluster.local
   ```

5. Si el selector o las etiquetas no coinciden, restablezca los manifiestos proporcionados en esta práctica y aplíquelos:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-deployment.yaml
   kubectl apply -f ~/k8s-intro/manifests/webapp-service.yaml
   ```

## Limpieza

> **Importante:** `webapp`, `webapp-svc` y el namespace `curso-k8s` son la base para los laboratorios 13, 14 y 15. No elimine estos recursos si continuará con la secuencia del curso.

La práctica ya elimina automáticamente el Pod temporal `dns-http-test` mediante la opción `--rm`. Compruebe que no permanece:

```bash
kubectl get pod dns-http-test -n curso-k8s
```

Si ha terminado definitivamente todos los laboratorios dependientes y dispone de autorización para eliminar el escenario, ejecute:

```bash
kubectl delete -f ~/k8s-intro/manifests/webapp-service.yaml
kubectl delete -f ~/k8s-intro/manifests/webapp-deployment.yaml
kubectl delete namespace curso-k8s
```

Para conservar únicamente la evidencia y retirar los manifiestos locales después de una entrega validada:

```bash
rm -f ~/k8s-intro/manifests/webapp-deployment.yaml
rm -f ~/k8s-intro/manifests/webapp-service.yaml
```

No elimine `~/k8s-intro/analisis` hasta que la evidencia haya sido revisada o entregada.

## Resumen

En esta práctica se completó la transformación de `webapp` en una aplicación Kubernetes declarativa con:

- Tres réplicas administradas por un Deployment.
- Estrategia `RollingUpdate` con `maxUnavailable: 1` y `maxSurge: 1`.
- Probes HTTP de disponibilidad y vida sobre la ruta `/`.
- Solicitudes y límites explícitos de CPU y memoria.
- Un Service interno `ClusterIP` accesible mediante `webapp-svc.curso-k8s.svc.cluster.local`.
- Una actualización controlada de `nginx:1.27.0` a `nginx:1.27.1`.
- Evidencia reproducible del rollout y de su historial.

### Recursos opcionales

- Documentación oficial de Deployments de Kubernetes: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Documentación oficial de Services de Kubernetes: <https://kubernetes.io/docs/concepts/services-networking/service/>
- Documentación oficial de probes: <https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/>
- Documentación oficial de recursos de contenedor: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>
- Documentación oficial de `kubectl rollout`: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/>
