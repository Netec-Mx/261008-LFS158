# Práctica: Publicación de aplicaciones mediante Ingress

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 30 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica instalarás el controlador `ingress-nginx` con la versión fija `v1.12.0` y crearás un recurso `Ingress` para publicar la aplicación existente `webapp`. El tráfico HTTP dirigido al host `webapp.local.test` será enrutado hacia el Service interno `webapp-svc`, de tipo `ClusterIP`, sin exponer directamente los Pods de la aplicación mediante `NodePort`.

La práctica refuerza la diferencia entre un Service, que ofrece conectividad estable dentro del clúster, y un Ingress, que define reglas HTTP de capa 7 basadas en hosts y rutas.

## Objetivos de Aprendizaje

- [ ] Instalar y verificar el controlador `ingress-nginx` versión `v1.12.0`.
- [ ] Crear un recurso `Ingress` con `ingressClassName: nginx`.
- [ ] Enrutar solicitudes HTTP para `webapp.local.test/` hacia el Service `webapp-svc`.
- [ ] Validar que la aplicación permanece publicada internamente mediante un Service `ClusterIP`.
- [ ] Obtener evidencias verificables del estado del controlador, del Ingress y de la respuesta HTTP.

## Prerrequisitos

### Conocimientos necesarios

Antes de comenzar, debes comprender los siguientes conceptos:

- Pods efímeros y Services como puntos de acceso estables.
- Diferencias entre Services `ClusterIP`, `NodePort` y `LoadBalancer`.
- Namespace, Deployment, Pod, ConfigMap, PersistentVolumeClaim y Service.
- Sintaxis básica de manifiestos YAML de Kubernetes.
- Uso de `kubectl get`, `kubectl describe`, `kubectl apply` y `kubectl wait`.
- Resolución local de nombres mediante `/etc/hosts` o mediante `curl --resolve`.

### Acceso y estado previo obligatorio

Debes disponer de lo siguiente:

- Laboratorio `14-00-01` completado.
- Namespace de trabajo `laboratorio-k8s`.
- Recursos existentes en el namespace de trabajo:
  - Deployment `webapp`.
  - Service `webapp-svc`.
  - PersistentVolumeClaim `webapp-content-pvc`.
  - ConfigMap `webapp-config`.
  - Pods de `webapp` en estado `Running` y `Ready`.
- Conexión a Internet para descargar el manifiesto oficial del controlador.
- Permisos para ejecutar `sudo` únicamente si eliges modificar `/etc/hosts`.
- Perfil de Minikube llamado `minikube`.
- Contexto activo de `kubectl`: `minikube`.

> **Nota de coherencia de namespace:** el alcance inicial de esta práctica puede mencionar el namespace `curso-k8s`; sin embargo, el entorno global del curso reserva `laboratorio-k8s`. En esta guía se utiliza exclusivamente `laboratorio-k8s` para mantener el Service existente y el recurso Ingress en el mismo namespace. Un Ingress solo puede dirigir tráfico a Services del mismo namespace.

## Entorno de Laboratorio

### Versiones de referencia

| Componente | Versión, edición y arquitectura | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, AMD64 | https://releases.ubuntu.com/22.04/ |
| Docker Engine | 27.2.0, Linux AMD64 | https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/ |
| Minikube | 1.34.0, Linux AMD64 | https://github.com/kubernetes/minikube/releases/tag/v1.34.0 |
| kubectl | 1.31.0, Linux AMD64 | https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl |
| Kubernetes | v1.31.0, Linux AMD64 | https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0 |
| Controlador ingress-nginx | v1.12.0, imagen Linux AMD64 mediante manifiesto oficial | https://github.com/kubernetes/ingress-nginx/releases/tag/controller-v1.12.0 |
| NGINX de la aplicación existente | 1.27.4, imagen Linux AMD64 | https://hub.docker.com/_/nginx |
| curl | `[VERSIÓN POR VALIDAR]`, paquete Linux AMD64 instalado en el equipo | https://curl.se/download.html |

> No se utiliza Microsoft 365 Copilot, Copilot Chat, Microsoft Designer, Microsoft Planner ni ninguna herramienta de IA en esta práctica. Por tanto, no se requiere licencia, configuración ni permisos para esos productos.

### Preparación inicial

1. Abre una terminal Bash.
2. Crea los directorios obligatorios del curso si aún no existen.
3. Verifica que Docker, Minikube y Kubernetes están disponibles.
4. Confirma que el contexto de `kubectl` es `minikube`.

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis

docker --version
minikube version
kubectl version --client=true

kubectl config current-context
minikube status --profile=minikube
```

El contexto esperado es:

```text
minikube
```

Si Minikube no está iniciado, utiliza el comando de referencia del curso:

```bash
minikube start \
  --driver=docker \
  --kubernetes-version=v1.31.0 \
  --cpus=2 \
  --memory=4096mb \
  --profile=minikube
```

Define variables para reducir errores de escritura:

```bash
export LAB_NS="laboratorio-k8s"
export INGRESS_NS="ingress-nginx"
export APP_SERVICE="webapp-svc"
export APP_HOST="webapp.local.test"
```

## Instrucciones Paso a Paso

### Paso 1: Verificar el clúster y la aplicación reutilizada

**Objetivo:** confirmar que el clúster está disponible, que el contexto es correcto y que los recursos creados en laboratorios anteriores existen antes de instalar el controlador Ingress.

**Instrucciones:**

1. Verifica el contexto activo de `kubectl`.

   ```bash
   kubectl config current-context
   ```

2. Comprueba el estado general de Minikube.

   ```bash
   minikube status --profile=minikube
   ```

3. Comprueba que existe el namespace de trabajo.

   ```bash
   kubectl get namespace "${LAB_NS}"
   ```

4. Comprueba el estado de los Pods de la aplicación.

   ```bash
   kubectl get pods -n "${LAB_NS}" -o wide
   ```

5. Comprueba que el Service `webapp-svc` existe y es de tipo `ClusterIP`.

   ```bash
   kubectl get service "${APP_SERVICE}" -n "${LAB_NS}"
   kubectl get service "${APP_SERVICE}" -n "${LAB_NS}" \
     -o jsonpath='{.spec.type}{"\n"}'
   ```

6. Confirma que el Service tiene endpoints asociados a Pods disponibles.

   ```bash
   kubectl get endpoints "${APP_SERVICE}" -n "${LAB_NS}"
   ```

7. Guarda una evidencia inicial del estado de la aplicación.

   ```bash
   {
     echo "Fecha UTC:"
     date -u
     echo
     echo "Contexto:"
     kubectl config current-context
     echo
     echo "Pods:"
     kubectl get pods -n "${LAB_NS}" -o wide
     echo
     echo "Service:"
     kubectl get service "${APP_SERVICE}" -n "${LAB_NS}"
     echo
     echo "Endpoints:"
     kubectl get endpoints "${APP_SERVICE}" -n "${LAB_NS}"
   } | tee ~/k8s-intro/analisis/15-00-01-estado-inicial.txt
   ```

**Resultado esperado:**

- El contexto activo es `minikube`.
- Minikube muestra componentes en estado `Running`.
- El namespace `laboratorio-k8s` existe.
- Los Pods de `webapp` están en estado `Running` y con `READY` completo, por ejemplo `1/1`.
- `webapp-svc` es de tipo `ClusterIP`.
- El recurso `Endpoints` contiene al menos una dirección IP y el puerto de la aplicación.

**Verificación:**

Ejecuta el siguiente comando:

```bash
kubectl get service "${APP_SERVICE}" -n "${LAB_NS}" \
  -o jsonpath='Nombre={.metadata.name} Tipo={.spec.type} ClusterIP={.spec.clusterIP}{"\n"}'
```

La salida debe contener:

```text
Nombre=webapp-svc Tipo=ClusterIP
```

No debe aparecer `NodePort` como tipo del Service de la aplicación.

---

### Paso 2: Instalar y verificar ingress-nginx Controller v1.12.0

**Objetivo:** instalar el controlador que interpreta los recursos Ingress y aplica las reglas de enrutamiento HTTP.

**Instrucciones:**

1. Descarga el manifiesto oficial de `ingress-nginx` fijado a la versión `controller-v1.12.0`.

   ```bash
   curl -fL \
     -o ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml \
     https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
   ```

2. Comprueba que el archivo se descargó correctamente y que no está vacío.

   ```bash
   ls -lh ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml
   grep -n "controller:v1.12.0" \
     ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml
   ```

3. Aplica el manifiesto oficial.

   ```bash
   kubectl apply \
     -f ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml
   ```

4. Espera hasta que el Deployment del controlador esté disponible. El límite de espera evita continuar si la imagen no puede descargarse o si el Pod no inicia correctamente.

   ```bash
   kubectl wait \
     --namespace="${INGRESS_NS}" \
     --for=condition=Available \
     deployment/ingress-nginx-controller \
     --timeout=180s
   ```

5. Consulta los Pods del controlador.

   ```bash
   kubectl get pods -n "${INGRESS_NS}" -o wide
   ```

6. Consulta el Service que expone al controlador.

   ```bash
   kubectl get service -n "${INGRESS_NS}" ingress-nginx-controller
   ```

7. Comprueba que existe la clase de Ingress `nginx`.

   ```bash
   kubectl get ingressclass
   kubectl describe ingressclass nginx
   ```

8. Guarda evidencias técnicas de la instalación.

   ```bash
   {
     echo "Pods del controlador:"
     kubectl get pods -n "${INGRESS_NS}" -o wide
     echo
     echo "Deployment del controlador:"
     kubectl get deployment ingress-nginx-controller -n "${INGRESS_NS}"
     echo
     echo "Service del controlador:"
     kubectl get service ingress-nginx-controller -n "${INGRESS_NS}"
     echo
     echo "IngressClass:"
     kubectl get ingressclass
   } | tee ~/k8s-intro/analisis/15-00-01-ingress-controller.txt
   ```

**Resultado esperado:**

- El namespace `ingress-nginx` se crea.
- El Deployment `ingress-nginx-controller` alcanza disponibilidad.
- El Pod del controlador se muestra como `Running`.
- La clase `nginx` aparece al ejecutar `kubectl get ingressclass`.
- El Service `ingress-nginx-controller` queda configurado como punto de entrada del controlador.

Una salida de referencia parcial es:

```text
deployment.apps/ingress-nginx-controller condition met
```

Y para los Pods:

```text
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          ...
```

**Verificación:**

Comprueba la imagen utilizada por el controlador:

```bash
kubectl get deployment ingress-nginx-controller \
  -n "${INGRESS_NS}" \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

La salida debe contener:

```text
registry.k8s.io/ingress-nginx/controller:v1.12.0
```

> El recurso `Ingress` es una definición declarativa de rutas. El controlador `ingress-nginx` es el software que observa esa definición y configura el enrutamiento HTTP. Crear un Ingress sin un controlador no publica la aplicación.

---

### Paso 3: Crear el recurso Ingress para webapp

**Objetivo:** definir una regla de enrutamiento HTTP que envíe las solicitudes dirigidas a `webapp.local.test/` hacia el Service interno `webapp-svc` por el puerto `80`.

**Instrucciones:**

1. Crea el manifiesto `webapp-ingress.yaml`.

   ```bash
   cat <<'EOF' > ~/k8s-intro/manifests/webapp-ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: webapp-ingress
     namespace: laboratorio-k8s
     labels:
       app.kubernetes.io/name: webapp
       app.kubernetes.io/component: ingress
   spec:
     ingressClassName: nginx
     rules:
       - host: webapp.local.test
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: webapp-svc
                   port:
                     number: 80
   EOF
   ```

2. Revisa el manifiesto antes de aplicarlo.

   ```bash
   cat ~/k8s-intro/manifests/webapp-ingress.yaml
   ```

3. Aplica el recurso Ingress.

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-ingress.yaml
   ```

4. Consulta el Ingress creado.

   ```bash
   kubectl get ingress webapp-ingress -n "${LAB_NS}"
   ```

5. Describe el Ingress para revisar las reglas, el backend y los eventos.

   ```bash
   kubectl describe ingress webapp-ingress -n "${LAB_NS}"
   ```

6. Comprueba que el backend asociado sigue siendo el Service interno correcto.

   ```bash
   kubectl get ingress webapp-ingress -n "${LAB_NS}" \
     -o jsonpath='{.spec.rules[0].host}{" -> "}{.spec.rules[0].http.paths[0].backend.service.name}{":"}{.spec.rules[0].http.paths[0].backend.service.port.number}{"\n"}'
   ```

7. Guarda la descripción como evidencia.

   ```bash
   kubectl describe ingress webapp-ingress -n "${LAB_NS}" \
     | tee ~/k8s-intro/analisis/15-00-01-describe-ingress.txt
   ```

**Resultado esperado:**

El Ingress debe contener una regla equivalente a la siguiente:

```text
Host: webapp.local.test
Path: /
Path Type: Prefix
Backend: webapp-svc:80
```

La descripción del recurso debe indicar que la clase utilizada es `nginx`.

**Verificación:**

Ejecuta:

```bash
kubectl get ingress webapp-ingress -n "${LAB_NS}" -o yaml
```

Confirma manualmente los siguientes criterios medibles:

- `kind: Ingress`.
- `metadata.name: webapp-ingress`.
- `metadata.namespace: laboratorio-k8s`.
- `spec.ingressClassName: nginx`.
- `host: webapp.local.test`.
- `path: /`.
- `pathType: Prefix`.
- Backend `webapp-svc`.
- Puerto de backend `80`.

---

### Paso 4: Obtener la dirección de entrada y probar el enrutamiento HTTP

**Objetivo:** acceder a la aplicación mediante el host `webapp.local.test` y comprobar que el controlador dirige la solicitud al Service `webapp-svc`.

**Instrucciones:**

1. En una segunda terminal, inicia el túnel de Minikube. Mantén esta terminal abierta durante las pruebas.

   ```bash
   minikube tunnel --profile=minikube
   ```

   Si el sistema solicita privilegios administrativos para crear rutas de red, proporciona la contraseña de `sudo`.

2. En la primera terminal, espera a que el Service del controlador tenga una dirección externa asignada.

   ```bash
   kubectl get service ingress-nginx-controller \
     -n "${INGRESS_NS}" \
     --watch
   ```

   Detén la observación con `Ctrl+C` cuando la columna `EXTERNAL-IP` ya no muestre `<pending>`.

3. Obtén y guarda la dirección IP externa del controlador.

   ```bash
   export INGRESS_IP="$(kubectl get service ingress-nginx-controller \
     -n "${INGRESS_NS}" \
     -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"

   echo "${INGRESS_IP}"
   ```

4. Si la variable `INGRESS_IP` está vacía, consulta la IP de Minikube como alternativa de diagnóstico.

   ```bash
   minikube ip --profile=minikube
   ```

5. Realiza una solicitud HTTP sin modificar `/etc/hosts`, usando `curl --resolve`. Esta opción es la recomendada porque limita el cambio de resolución al comando ejecutado.

   ```bash
   curl --verbose \
     --resolve "${APP_HOST}:80:${INGRESS_IP}" \
     "http://${APP_HOST}/"
   ```

6. Realiza una segunda solicitud y muestra solamente la cabecera HTTP y las primeras líneas del contenido.

   ```bash
   curl --silent --show-error --include \
     --resolve "${APP_HOST}:80:${INGRESS_IP}" \
     "http://${APP_HOST}/" \
     | head -n 20
   ```

7. Como alternativa opcional, añade una entrada temporal en `/etc/hosts`. Usa este método solo si deseas probar desde un navegador local.

   ```bash
   echo "${INGRESS_IP} ${APP_HOST}" | sudo tee -a /etc/hosts
   ```

8. Si modificaste `/etc/hosts`, prueba el acceso mediante el nombre configurado.

   ```bash
   curl --verbose "http://${APP_HOST}/"
   ```

9. Guarda las evidencias de la prueba HTTP.

   ```bash
   {
     echo "IP de entrada: ${INGRESS_IP}"
     echo
     echo "Respuesta HTTP:"
     curl --silent --show-error --include \
       --resolve "${APP_HOST}:80:${INGRESS_IP}" \
       "http://${APP_HOST}/" \
       | head -n 30
   } | tee ~/k8s-intro/analisis/15-00-01-prueba-http.txt
   ```

**Resultado esperado:**

La respuesta debe incluir un código HTTP exitoso:

```text
HTTP/1.1 200 OK
```

El contenido HTML dependerá de la aplicación `webapp` creada en laboratorios anteriores. Debe corresponder al contenido servido por NGINX y almacenado o configurado mediante los recursos previamente creados.

**Verificación:**

Ejecuta la prueba de estado HTTP:

```bash
curl --silent --output /dev/null --write-out "%{http_code}\n" \
  --resolve "${APP_HOST}:80:${INGRESS_IP}" \
  "http://${APP_HOST}/"
```

El resultado esperado es:

```text
200
```

Confirma adicionalmente que la aplicación no fue convertida en `NodePort`:

```bash
kubectl get service "${APP_SERVICE}" -n "${LAB_NS}" \
  -o jsonpath='{.spec.type}{"\n"}'
```

El resultado debe ser:

```text
ClusterIP
```

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios siguientes.

| Criterio | Comando de validación | Evidencia esperada |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | `minikube` |
| Aplicación disponible | `kubectl get pods -n laboratorio-k8s` | Pods `webapp` con estado `Running` y `READY` completo |
| Service interno | `kubectl get svc webapp-svc -n laboratorio-k8s` | Tipo `ClusterIP` |
| Controlador disponible | `kubectl get deployment -n ingress-nginx ingress-nginx-controller` | `AVAILABLE` igual a las réplicas deseadas |
| Clase de Ingress | `kubectl get ingressclass nginx` | Recurso `nginx` existente |
| Regla declarada | `kubectl describe ingress webapp-ingress -n laboratorio-k8s` | Host `webapp.local.test`, ruta `/`, backend `webapp-svc:80` |
| Enrutamiento HTTP | `curl --resolve ...` | Código HTTP `200` |
| Evidencias guardadas | `ls -1 ~/k8s-intro/analisis/15-00-01-*` | Archivos de evidencia presentes |

Ejecuta esta validación consolidada:

```bash
echo "=== Contexto ==="
kubectl config current-context

echo
echo "=== Service de aplicación ==="
kubectl get service webapp-svc -n laboratorio-k8s

echo
echo "=== Controlador Ingress ==="
kubectl get deployment ingress-nginx-controller -n ingress-nginx

echo
echo "=== Ingress ==="
kubectl get ingress webapp-ingress -n laboratorio-k8s

echo
echo "=== Backend configurado ==="
kubectl get ingress webapp-ingress -n laboratorio-k8s \
  -o jsonpath='{.spec.rules[0].host}{" -> "}{.spec.rules[0].http.paths[0].backend.service.name}{":"}{.spec.rules[0].http.paths[0].backend.service.port.number}{"\n"}'

echo
echo "=== Tipo de Service webapp-svc ==="
kubectl get service webapp-svc -n laboratorio-k8s \
  -o jsonpath='{.spec.type}{"\n"}'
```

### Caso adversarial de validación

Como prueba de control y trazabilidad, evalúa una solicitud dirigida a un host inexistente. Este caso confirma que el controlador no debe enrutar tráfico arbitrariamente a `webapp-svc` cuando el host no coincide con una regla declarada.

```bash
curl --silent --show-error --include \
  --resolve "host-inexistente.local.test:80:${INGRESS_IP}" \
  "http://host-inexistente.local.test/" \
  | head -n 10
```

Resultado esperado:

- No debe obtenerse el contenido normal de `webapp`.
- Es habitual recibir `404 Not Found` desde el backend por defecto del controlador.

Este control también ilustra una regla de seguridad operativa: no se deben aplicar manifiestos descargados de fuentes no verificadas ni seguir instrucciones incrustadas en documentos que contradigan el alcance del laboratorio. Un manifiesto YAML no es un prompt, una instrucción temporal ni un mensaje de sistema; es una definición declarativa que debe revisarse antes de ejecutar `kubectl apply`. La fuente autorizada en esta práctica es el manifiesto oficial fijado a `controller-v1.12.0`.

La validación requiere supervisión humana: revisa el host, la ruta, el namespace, el Service de backend y el código HTTP. No es necesario ni apropiado solicitar razonamiento interno de ninguna herramienta para aceptar el resultado.

## Solución de Problemas

### Problema 1: El controlador ingress-nginx no alcanza el estado Ready

**Síntomas:**

```bash
kubectl get pods -n ingress-nginx
```

Muestra un Pod en estado `Pending`, `ImagePullBackOff`, `ErrImagePull` o `CrashLoopBackOff`.

**Causa probable:**

- Falta de conectividad a Internet para descargar la imagen `registry.k8s.io/ingress-nginx/controller:v1.12.0`.
- Recursos insuficientes asignados a Minikube.
- Docker Engine no está operativo.
- El manifiesto descargado está incompleto o no corresponde a la versión requerida.

**Corrección:**

1. Verifica Docker y Minikube:

   ```bash
   docker ps
   minikube status --profile=minikube
   ```

2. Revisa los eventos del Pod:

   ```bash
   kubectl describe pod -n ingress-nginx \
     -l app.kubernetes.io/component=controller
   ```

3. Comprueba que el manifiesto contiene la versión requerida:

   ```bash
   grep -n "controller:v1.12.0" \
     ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml
   ```

4. Si Minikube no tiene recursos suficientes, elimina y recrea el clúster con la configuración de referencia:

   ```bash
   minikube delete --profile=minikube

   minikube start \
     --driver=docker \
     --kubernetes-version=v1.31.0 \
     --cpus=2 \
     --memory=4096mb \
     --profile=minikube
   ```

5. Restaura los recursos de laboratorios anteriores antes de continuar con esta práctica.

### Problema 2: curl no responde con HTTP 200 o la IP externa permanece en `<pending>`

**Síntomas:**

- `kubectl get service ingress-nginx-controller -n ingress-nginx` muestra `EXTERNAL-IP` como `<pending>`.
- `curl --resolve` devuelve `Connection refused`, `Could not resolve host`, tiempo de espera o `404 Not Found`.
- La variable `INGRESS_IP` está vacía.

**Causa probable:**

- El comando `minikube tunnel --profile=minikube` no está ejecutándose.
- Se utilizó una IP vacía, incorrecta o no accesible.
- El host enviado por `curl` no es exactamente `webapp.local.test`.
- El Ingress apunta a un Service inexistente, a un puerto incorrecto o a un namespace distinto.
- Los Pods seleccionados por `webapp-svc` no están disponibles.

**Corrección:**

1. Inicia el túnel en otra terminal y mantenlo activo:

   ```bash
   minikube tunnel --profile=minikube
   ```

2. Comprueba el Service del controlador:

   ```bash
   kubectl get service ingress-nginx-controller -n ingress-nginx
   ```

3. Obtén nuevamente la IP:

   ```bash
   export INGRESS_IP="$(kubectl get service ingress-nginx-controller \
     -n ingress-nginx \
     -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"

   echo "${INGRESS_IP}"
   ```

4. Revisa la regla de Ingress:

   ```bash
   kubectl describe ingress webapp-ingress -n laboratorio-k8s
   ```

5. Revisa los endpoints de la aplicación:

   ```bash
   kubectl get endpoints webapp-svc -n laboratorio-k8s
   ```

6. Repite la prueba usando exactamente el host definido:

   ```bash
   curl --verbose \
     --resolve "webapp.local.test:80:${INGRESS_IP}" \
     http://webapp.local.test/
   ```

## Limpieza

La limpieza debe preservar la aplicación `webapp`, su PVC, ConfigMap, Deployment y Service si serán reutilizados en laboratorios posteriores.

1. Si agregaste una entrada en `/etc/hosts`, elimínala de forma controlada:

   ```bash
   sudo sed -i '\|webapp.local.test|d' /etc/hosts
   ```

2. Detén el proceso `minikube tunnel` con `Ctrl+C` en la terminal donde está ejecutándose.

3. Si necesitas eliminar únicamente la regla de publicación de esta práctica, elimina el recurso Ingress:

   ```bash
   kubectl delete ingress webapp-ingress -n laboratorio-k8s
   ```

4. Conserva los archivos de evidencia para revisión:

   ```bash
   ls -lh ~/k8s-intro/analisis/15-00-01-*
   ```

5. Solo si el controlador Ingress no será necesario en prácticas posteriores, elimínalo usando el mismo manifiesto versionado:

   ```bash
   kubectl delete \
     -f ~/k8s-intro/manifests/ingress-nginx-controller-v1.12.0.yaml
   ```

> No ejecutes `minikube delete` como limpieza normal de esta práctica, porque eliminaría el clúster completo y los recursos creados en laboratorios anteriores.

## Resumen

En esta práctica instalaste `ingress-nginx Controller v1.12.0`, verificaste la clase `nginx` y creaste el recurso `webapp-ingress`. La regla configurada asocia el host `webapp.local.test` y la ruta `/` con el Service interno `webapp-svc:80`.

La validación demostró que:

- Los Pods de la aplicación no necesitan exponerse directamente.
- `webapp-svc` permanece como `ClusterIP`.
- El controlador Ingress actúa como punto de entrada HTTP compartido.
- El enrutamiento se realiza por nombre de host y ruta HTTP, capacidades propias de capa 7.
- La evidencia técnica puede revisarse en `~/k8s-intro/analisis/`.

### Recursos oficiales

- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes Service: https://kubernetes.io/docs/concepts/services-networking/service/
- ingress-nginx Controller v1.12.0: https://github.com/kubernetes/ingress-nginx/releases/tag/controller-v1.12.0
- Manifiesto oficial fijado a `controller-v1.12.0`: https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml
- Minikube tunnel: https://minikube.sigs.k8s.io/docs/commands/tunnel/
