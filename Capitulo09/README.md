# Práctica: Creación y administración de objetos

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 30 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se crearán objetos declarativos de Kubernetes para ejecutar una aplicación web NGINX aislada en un Namespace propio. Se utilizarán un `ConfigMap`, un `Deployment` con réplicas administradas, un `Service` de tipo `NodePort` y comandos `kubectl` para inspeccionar el estado deseado y observado. Los objetos permanecerán disponibles al finalizar porque serán utilizados en el laboratorio posterior de RBAC.

> **Nota sobre el perfil:** la especificación de esta práctica requiere el perfil Minikube `intro-k8s` y el acceso mediante `minikube -p intro-k8s ip`. Si el entorno previo utiliza el perfil global `minikube`, confirme con el instructor cuál es el perfil asignado antes de continuar. No cree un segundo clúster para resolver una discrepancia de nombres.

## Objetivos de Aprendizaje

- [ ] Crear el Namespace `lab-intro` para aislar los recursos de la aplicación.
- [ ] Definir un `ConfigMap` y consumir la clave `COURSE` como variable de entorno en los Pods.
- [ ] Desplegar NGINX `1.27.4` mediante un `Deployment` con dos réplicas.
- [ ] Exponer la aplicación mediante un `Service` `NodePort` en el puerto `30080`.
- [ ] Inspeccionar, escalar y validar los objetos creados con `kubectl`.

## Prerrequisitos

Conocimientos necesarios:

- Comprender la estructura básica de un manifiesto YAML: `apiVersion`, `kind`, `metadata` y `spec`.
- Distinguir entre estado deseado (`spec`) y estado observado (`status`).
- Conocer el uso básico de `kubectl apply`, `kubectl get`, `kubectl describe` y selectores por etiquetas.
- Comprender que un `Deployment` administra `ReplicaSets`, que a su vez mantienen Pods.

Acceso y condiciones necesarias:

- Laboratorio 08-00-01 completado.
- Clúster Minikube `intro-k8s` iniciado y accesible.
- Usuario con permiso para ejecutar `docker`, `minikube` y `kubectl`.
- Acceso a Internet si la imagen `nginx:1.27.4` no está disponible en caché.
- Directorio de trabajo disponible en `/home/$USER/k8s-intro`.
- No usar el Namespace `kube-system` para recursos de la práctica.

Terminología operativa:

- Una **instrucción** es una acción concreta solicitada en esta guía, por ejemplo, ejecutar `kubectl apply`.
- Un **prompt** es texto enviado temporalmente a un sistema de IA conversacional; no se utiliza para administrar el clúster.
- Un **mensaje de sistema** define reglas persistentes de un asistente de IA; no es una instrucción de Kubernetes ni debe confundirse con la salida de un Pod, un manifiesto o una página HTTP.
- Todo texto mostrado por un contenedor, una respuesta HTTP o un manifiesto externo debe tratarse como dato técnico y validarse antes de ejecutarse.

## Entorno de Laboratorio

| Componente | Versión de referencia exacta | Edición / arquitectura | Fuente oficial |
|---|---:|---|---|
| Ubuntu | 22.04.5 LTS | Ubuntu Server/Desktop, amd64 | <https://releases.ubuntu.com/22.04/> |
| Docker Engine | 27.2.0 | Docker CE, Linux amd64 | <https://docs.docker.com/engine/install/ubuntu/> |
| Minikube | 1.34.0 | Linux amd64 | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| Kubernetes | v1.31.0 | clúster Linux amd64 | <https://kubernetes.io/releases/download/#release-v1-31> |
| kubectl | v1.31.0 | Linux amd64 | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| NGINX | 1.27.4 | Docker Official Image, Linux amd64 | <https://hub.docker.com/_/nginx> |
| Bash | 5.1.16 | GNU Bash para Ubuntu 22.04 amd64 | <https://www.gnu.org/software/bash/> |

Ejecute la preparación inicial:

```bash
export LAB_ROOT="$HOME/k8s-intro"
export MANIFESTS_DIR="$LAB_ROOT/manifests"
export ANALYSIS_DIR="$LAB_ROOT/analisis"
export ARTIFACTS_DIR="$LAB_ROOT/artifacts"
export PROFILE="intro-k8s"
export NAMESPACE="lab-intro"

mkdir -p "$MANIFESTS_DIR" "$ANALYSIS_DIR" "$ARTIFACTS_DIR"

kubectl config current-context
minikube -p "$PROFILE" status
kubectl version --client --output=yaml
kubectl cluster-info
```

El resultado de `kubectl config current-context` debe corresponder al contexto del clúster asignado para esta práctica. El resultado de `minikube -p intro-k8s status` debe indicar que los componentes principales están en estado `Running`.

## Instrucciones Paso a Paso

### Paso 1: Verificar el clúster y preparar los directorios

**Objetivo:** confirmar que el clúster correcto está disponible y preparar las ubicaciones obligatorias para manifiestos y evidencias.

**Instrucciones:**

1. Declare las variables de trabajo si no permanecen disponibles en la terminal actual.

   ```bash
   export LAB_ROOT="$HOME/k8s-intro"
   export MANIFESTS_DIR="$LAB_ROOT/manifests"
   export ANALYSIS_DIR="$LAB_ROOT/analisis"
   export ARTIFACTS_DIR="$LAB_ROOT/artifacts"
   export PROFILE="intro-k8s"
   export NAMESPACE="lab-intro"
   ```

2. Cree los directorios requeridos.

   ```bash
   mkdir -p "$MANIFESTS_DIR" "$ANALYSIS_DIR" "$ARTIFACTS_DIR"
   ```

3. Compruebe el contexto activo, el estado de Minikube y la conectividad con la API de Kubernetes.

   ```bash
   kubectl config current-context
   minikube -p "$PROFILE" status
   kubectl get nodes
   ```

4. Registre la información básica del entorno.

   ```bash
   {
     echo "Fecha UTC: $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
     echo "Contexto kubectl:"
     kubectl config current-context
     echo
     echo "Estado de Minikube:"
     minikube -p "$PROFILE" status
     echo
     echo "Nodos:"
     kubectl get nodes -o wide
   } | tee "$ANALYSIS_DIR/09-00-01-entorno.txt"
   ```

**Resultado esperado:**

- El perfil `intro-k8s` debe estar en ejecución.
- `kubectl get nodes` debe mostrar un nodo con estado `Ready`.
- Deben existir los directorios `manifests`, `analisis` y `artifacts` dentro de `~/k8s-intro`.

**Verificación:**

```bash
test -d "$MANIFESTS_DIR" && test -d "$ANALYSIS_DIR" && test -d "$ARTIFACTS_DIR" && echo "Directorios preparados"
kubectl get nodes
```

---

### Paso 2: Crear el manifiesto declarativo de la aplicación

**Objetivo:** definir en YAML el Namespace, el ConfigMap, el Deployment y el Service requeridos por la práctica.

**Instrucciones:**

1. Cree el archivo único de manifiestos.

   ```bash
   nano "$MANIFESTS_DIR/web-demo.yaml"
   ```

2. Pegue el siguiente contenido completo. Respete la indentación YAML, los nombres y la versión de imagen.

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: lab-intro
     labels:
       curso: introduccion-kubernetes
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: web-config
     namespace: lab-intro
     labels:
       app: web-demo
   data:
     COURSE: Introduccion-a-Kubernetes
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web-demo
     namespace: lab-intro
     labels:
       app: web-demo
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: web-demo
     template:
       metadata:
         labels:
           app: web-demo
       spec:
         containers:
           - name: nginx
             image: nginx:1.27.4
             ports:
               - name: http
                 containerPort: 80
             env:
               - name: COURSE
                 valueFrom:
                   configMapKeyRef:
                     name: web-config
                     key: COURSE
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: web-demo
     namespace: lab-intro
     labels:
       app: web-demo
   spec:
     type: NodePort
     selector:
       app: web-demo
     ports:
       - name: http
         protocol: TCP
         port: 80
         targetPort: http
         nodePort: 30080
   ```

3. Valide localmente la sintaxis y los campos contra la API del clúster sin crear recursos.

   ```bash
   kubectl apply --dry-run=client -f "$MANIFESTS_DIR/web-demo.yaml"
   kubectl apply --dry-run=server -f "$MANIFESTS_DIR/web-demo.yaml"
   ```

4. Revise el archivo antes de aplicarlo.

   ```bash
   sed -n '1,220p' "$MANIFESTS_DIR/web-demo.yaml"
   ```

**Resultado esperado:**

- La validación `--dry-run=client` debe indicar que los cuatro recursos fueron configurados en modo simulado.
- La validación `--dry-run=server` debe ser aceptada por la API del clúster.
- El manifiesto debe contener cuatro documentos YAML separados por `---`.

**Verificación:**

```bash
grep -E 'kind: (Namespace|ConfigMap|Deployment|Service)' "$MANIFESTS_DIR/web-demo.yaml"
grep -E 'replicas: 2|image: nginx:1.27.4|nodePort: 30080|COURSE: Introduccion-a-Kubernetes' "$MANIFESTS_DIR/web-demo.yaml"
```

---

### Paso 3: Aplicar los objetos al clúster

**Objetivo:** enviar el estado deseado definido en el manifiesto a la API de Kubernetes.

**Instrucciones:**

1. Aplique el archivo declarativo.

   ```bash
   kubectl apply -f "$MANIFESTS_DIR/web-demo.yaml"
   ```

2. Espere a que el Deployment complete su reconciliación inicial.

   ```bash
   kubectl rollout status deployment/web-demo \
     --namespace "$NAMESPACE" \
     --timeout=180s
   ```

3. Consulte los recursos principales creados en el Namespace.

   ```bash
   kubectl get namespace "$NAMESPACE"
   kubectl get configmap,deploy,replicaset,pod,service \
     --namespace "$NAMESPACE" \
     -o wide
   ```

4. Compruebe el valor del ConfigMap.

   ```bash
   kubectl get configmap web-config \
     --namespace "$NAMESPACE" \
     -o yaml
   ```

**Resultado esperado:**

- Debe existir el Namespace `lab-intro`.
- El Deployment `web-demo` debe mostrar `2/2` en la columna `READY`.
- Debe existir un ReplicaSet administrado por el Deployment.
- Deben existir dos Pods NGINX con estado `Running`.
- El Service `web-demo` debe ser de tipo `NodePort` y mostrar el mapeo `80:30080/TCP`.

**Verificación:**

```bash
kubectl get deployment web-demo --namespace "$NAMESPACE"
kubectl get pods --namespace "$NAMESPACE" -l app=web-demo
kubectl get service web-demo --namespace "$NAMESPACE"
```

---

### Paso 4: Inspeccionar relaciones, estado y configuración

**Objetivo:** comprobar que los selectores, las etiquetas, los Endpoints y la variable de entorno relacionan correctamente los objetos.

**Instrucciones:**

1. Examine el Deployment y el ReplicaSet administrado.

   ```bash
   kubectl describe deployment web-demo --namespace "$NAMESPACE"
   kubectl get replicaset --namespace "$NAMESPACE" -l app=web-demo
   ```

2. Liste los Pods utilizando el selector de etiqueta `app=web-demo`.

   ```bash
   kubectl get pods \
     --namespace "$NAMESPACE" \
     --selector app=web-demo \
     -o wide
   ```

3. Consulte los Endpoints generados por el Service.

   ```bash
   kubectl get endpoints web-demo --namespace "$NAMESPACE" -o wide
   kubectl get endpointslice --namespace "$NAMESPACE" \
     -l kubernetes.io/service-name=web-demo
   ```

4. Compruebe que el valor `COURSE` se inyectó en uno de los Pods.

   ```bash
   kubectl exec deployment/web-demo \
     --namespace "$NAMESPACE" \
     -- printenv COURSE
   ```

5. Revise los eventos recientes del Namespace.

   ```bash
   kubectl get events \
     --namespace "$NAMESPACE" \
     --sort-by='.lastTimestamp'
   ```

**Resultado esperado:**

- El Deployment debe seleccionar Pods con la etiqueta `app=web-demo`.
- Los Endpoints deben mostrar dos direcciones de Pods y el puerto `80`.
- El comando `printenv COURSE` debe mostrar `Introduccion-a-Kubernetes`.
- Los eventos no deben contener errores persistentes de creación, programación o descarga de imagen.

**Verificación:**

```bash
test "$(kubectl get pods -n "$NAMESPACE" -l app=web-demo --no-headers | wc -l)" -eq 2 \
  && echo "Existen dos Pods administrados"

kubectl exec deployment/web-demo -n "$NAMESPACE" -- printenv COURSE \
  | grep -Fx 'Introduccion-a-Kubernetes'

kubectl get endpoints web-demo -n "$NAMESPACE"
```

---

### Paso 5: Escalar el Deployment a tres réplicas

**Objetivo:** modificar declarativamente el estado deseado del Deployment y comprobar que Kubernetes crea el Pod adicional.

**Instrucciones:**

1. Modifique el manifiesto para cambiar el valor de réplicas de `2` a `3`.

   ```bash
   sed -i 's/replicas: 2/replicas: 3/' "$MANIFESTS_DIR/web-demo.yaml"
   grep -n 'replicas:' "$MANIFESTS_DIR/web-demo.yaml"
   ```

2. Aplique nuevamente el manifiesto actualizado.

   ```bash
   kubectl apply -f "$MANIFESTS_DIR/web-demo.yaml"
   ```

3. Espere a que el estado observado alcance el estado deseado.

   ```bash
   kubectl rollout status deployment/web-demo \
     --namespace "$NAMESPACE" \
     --timeout=180s
   ```

4. Consulte el Deployment, los Pods, el ReplicaSet y los Endpoints.

   ```bash
   kubectl get deployment web-demo --namespace "$NAMESPACE"
   kubectl get pods --namespace "$NAMESPACE" -l app=web-demo -o wide
   kubectl get replicaset --namespace "$NAMESPACE"
   kubectl get endpoints web-demo --namespace "$NAMESPACE" -o wide
   ```

**Resultado esperado:**

- El manifiesto debe mantener `replicas: 3`.
- El Deployment debe informar `3/3` en `READY`.
- Deben existir tres Pods con etiqueta `app=web-demo` en estado `Running`.
- Los Endpoints del Service deben contener tres direcciones listas, una por cada Pod disponible.

**Verificación:**

```bash
kubectl get deployment web-demo \
  --namespace "$NAMESPACE" \
  -o jsonpath='{.spec.replicas} deseadas, {.status.availableReplicas} disponibles{"\n"}'

test "$(kubectl get pods -n "$NAMESPACE" -l app=web-demo --no-headers | wc -l)" -eq 3 \
  && echo "Escalado validado: tres Pods encontrados"
```

---

### Paso 6: Acceder al Service y guardar evidencias

**Objetivo:** validar la disponibilidad HTTP a través del NodePort `30080` y generar la evidencia requerida.

**Instrucciones:**

1. Obtenga la dirección IP del nodo Minikube.

   ```bash
   minikube -p "$PROFILE" ip
   ```

2. Realice una solicitud HTTP al Service mediante el NodePort especificado.

   ```bash
   curl -fsS "http://$(minikube -p intro-k8s ip):30080" | head -n 10
   ```

3. Valide que la respuesta contiene el contenido estándar de bienvenida de NGINX.

   ```bash
   curl -fsS "http://$(minikube -p intro-k8s ip):30080" \
     | grep -q 'Welcome to nginx!' \
     && echo "Acceso HTTP validado"
   ```

4. Genere el archivo de evidencia solicitado. El archivo consolidará recursos, estado de rollout, Endpoints, eventos y la respuesta HTTP.

   ```bash
   {
     echo "Laboratorio: 09-00-01"
     echo "Fecha UTC: $(date -u '+%Y-%m-%dT%H:%M:%SZ')"
     echo "Contexto:"
     kubectl config current-context
     echo
     echo "Namespace:"
     kubectl get namespace "$NAMESPACE"
     echo
     echo "ConfigMap:"
     kubectl get configmap web-config -n "$NAMESPACE" -o yaml
     echo
     echo "Deployment:"
     kubectl get deployment web-demo -n "$NAMESPACE" -o wide
     echo
     echo "ReplicaSets:"
     kubectl get replicaset -n "$NAMESPACE" -o wide
     echo
     echo "Pods:"
     kubectl get pods -n "$NAMESPACE" -l app=web-demo -o wide
     echo
     echo "Service:"
     kubectl get service web-demo -n "$NAMESPACE" -o wide
     echo
     echo "Endpoints:"
     kubectl get endpoints web-demo -n "$NAMESPACE" -o wide
     echo
     echo "Variable COURSE:"
     kubectl exec deployment/web-demo -n "$NAMESPACE" -- printenv COURSE
     echo
     echo "Eventos:"
     kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp'
     echo
     echo "Prueba HTTP:"
     curl -fsS "http://$(minikube -p intro-k8s ip):30080" | head -n 10
   } | tee "$ARTIFACTS_DIR/web-demo-status.txt"
   ```

5. Confirme que la evidencia existe y contiene información del acceso HTTP.

   ```bash
   test -s "$ARTIFACTS_DIR/web-demo-status.txt" \
     && grep -q 'Welcome to nginx!' "$ARTIFACTS_DIR/web-demo-status.txt" \
     && echo "Evidencia creada correctamente"
   ```

**Resultado esperado:**

- La solicitud a `http://$(minikube -p intro-k8s ip):30080` debe devolver HTML de NGINX.
- El archivo `~/k8s-intro/artifacts/web-demo-status.txt` debe existir, no estar vacío y contener la cadena `Welcome to nginx!`.
- La evidencia debe mostrar tres Pods disponibles y el valor `Introduccion-a-Kubernetes`.

**Verificación:**

```bash
ls -lh "$ARTIFACTS_DIR/web-demo-status.txt"
grep -E 'Welcome to nginx!|Introduccion-a-Kubernetes|3/3' \
  "$ARTIFACTS_DIR/web-demo-status.txt"
```

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios medibles siguientes:

| Criterio | Comando de validación | Evidencia esperada |
|---|---|---|
| Namespace creado | `kubectl get namespace lab-intro` | Estado `Active` |
| ConfigMap creado | `kubectl get configmap web-config -n lab-intro` | Recurso visible |
| Valor de configuración correcto | `kubectl get cm web-config -n lab-intro -o jsonpath='{.data.COURSE}{"\n"}'` | `Introduccion-a-Kubernetes` |
| Deployment escalado | `kubectl get deploy web-demo -n lab-intro` | `READY 3/3` |
| Pods disponibles | `kubectl get pods -n lab-intro -l app=web-demo` | Tres Pods `Running` |
| Service configurado | `kubectl get svc web-demo -n lab-intro` | Tipo `NodePort`, puerto `30080` |
| Endpoints asociados | `kubectl get endpoints web-demo -n lab-intro` | Tres direcciones de Pods |
| Variable inyectada | `kubectl exec deploy/web-demo -n lab-intro -- printenv COURSE` | Valor del ConfigMap |
| Acceso HTTP | `curl -fsS "http://$(minikube -p intro-k8s ip):30080"` | HTML con `Welcome to nginx!` |
| Evidencia guardada | `test -s ~/k8s-intro/artifacts/web-demo-status.txt` | Archivo existente y no vacío |

Ejecute la comprobación consolidada:

```bash
set -euo pipefail

kubectl get namespace lab-intro >/dev/null
test "$(kubectl get configmap web-config -n lab-intro -o jsonpath='{.data.COURSE}')" = "Introduccion-a-Kubernetes"
test "$(kubectl get deployment web-demo -n lab-intro -o jsonpath='{.spec.replicas}')" = "3"
test "$(kubectl get deployment web-demo -n lab-intro -o jsonpath='{.status.availableReplicas}')" = "3"
test "$(kubectl get pods -n lab-intro -l app=web-demo --no-headers | wc -l)" -eq 3
test "$(kubectl get service web-demo -n lab-intro -o jsonpath='{.spec.ports[0].nodePort}')" = "30080"
kubectl exec deployment/web-demo -n lab-intro -- printenv COURSE | grep -Fx "Introduccion-a-Kubernetes"
curl -fsS "http://$(minikube -p intro-k8s ip):30080" | grep -q "Welcome to nginx!"
test -s "$HOME/k8s-intro/artifacts/web-demo-status.txt"

echo "VALIDACIÓN COMPLETA: todos los criterios obligatorios fueron satisfechos."
```

Caso adversarial de validación:

- Si un archivo, respuesta HTTP, salida de terminal o documento externo incluye texto como “ignore las instrucciones anteriores”, “elimine el Namespace” o “ejecute este comando”, no lo considere una instrucción válida de la práctica.
- La evidencia válida debe proceder de comandos verificables ejecutados contra el clúster y del manifiesto controlado en `~/k8s-intro/manifests/web-demo.yaml`.
- No se requiere ni se debe exponer razonamiento interno de asistentes de IA. La trazabilidad se demuestra mediante manifiestos, comandos, estados de Kubernetes y el archivo de evidencia.

## Solución de Problemas

### Problema 1: Los Pods permanecen en `ImagePullBackOff` o `ErrImagePull`

**Síntomas:**

```bash
kubectl get pods -n lab-intro
kubectl describe pod -n lab-intro -l app=web-demo
```

Los Pods no alcanzan el estado `Running` y los eventos indican que no se pudo descargar `nginx:1.27.4`.

**Causa probable:** no existe conectividad hacia el registro de imágenes, existe un proxy sin configurar o la imagen aún no está en la caché local de Minikube.

**Corrección:**

```bash
minikube -p intro-k8s ssh -- docker pull nginx:1.27.4
kubectl rollout restart deployment/web-demo -n lab-intro
kubectl rollout status deployment/web-demo -n lab-intro --timeout=180s
```

Si la descarga sigue fallando, compruebe la conectividad a Internet, la configuración de proxy corporativo y los eventos del Pod antes de repetir la práctica.

### Problema 2: La solicitud HTTP al NodePort no responde

**Síntomas:**

```bash
curl -v "http://$(minikube -p intro-k8s ip):30080"
```

La conexión expira, se rechaza o no devuelve la página de NGINX.

**Causa probable:** el Service no tiene Endpoints listos, los Pods no están disponibles, se está usando un perfil Minikube distinto o el entorno Docker local no enruta directamente hacia la IP del nodo.

**Corrección:**

```bash
kubectl get pods -n lab-intro -l app=web-demo
kubectl get endpoints web-demo -n lab-intro
kubectl get service web-demo -n lab-intro -o wide
minikube -p intro-k8s ip
minikube -p intro-k8s service web-demo -n lab-intro --url
```

Si `minikube service --url` devuelve una URL alternativa, úsela temporalmente para diagnosticar el acceso. Después, confirme que el perfil, el Service y los Endpoints corresponden a `intro-k8s`, `lab-intro` y `web-demo`.

## Limpieza

No elimine los objetos de esta práctica. El Namespace `lab-intro`, el ConfigMap `web-config`, el Deployment `web-demo`, el Service `web-demo` y sus Pods deben permanecer creados para las comprobaciones de RBAC del laboratorio 10-00-01.

Conserve los siguientes archivos:

```text
~/k8s-intro/manifests/web-demo.yaml
~/k8s-intro/analisis/09-00-01-entorno.txt
~/k8s-intro/artifacts/web-demo-status.txt
```

Como comprobación final de persistencia, ejecute:

```bash
kubectl get namespace lab-intro
kubectl get all -n lab-intro
kubectl get configmap web-config -n lab-intro
```

## Resumen

En esta práctica se aplicó un enfoque declarativo para crear y administrar objetos de Kubernetes. El manifiesto YAML definió el estado deseado de un Namespace, un ConfigMap, un Deployment con NGINX `1.27.4` y un Service `NodePort`.

Se verificó que:

- El Deployment mantuvo inicialmente dos réplicas y posteriormente reconcilió el estado hasta tres réplicas.
- El ConfigMap inyectó la variable de entorno `COURSE=Introduccion-a-Kubernetes` en los Pods.
- El Service seleccionó los Pods mediante la etiqueta `app=web-demo`.
- El acceso HTTP mediante el NodePort `30080` devolvió la página de bienvenida de NGINX.
- La evidencia técnica quedó registrada en `~/k8s-intro/artifacts/web-demo-status.txt`.

Recursos oficiales:

- Objetos de Kubernetes: <https://kubernetes.io/docs/concepts/overview/working-with-objects/>
- Gestión declarativa con `kubectl`: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/>
- Deployments: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- ConfigMaps: <https://kubernetes.io/docs/concepts/configuration/configmap/>
- Services: <https://kubernetes.io/docs/concepts/services-networking/service/>
- Minikube Service: <https://minikube.sigs.k8s.io/docs/handbook/accessing/>
