# Práctica: Uso de almacenamiento persistente

## Metadatos

| Duración | Complejidad | Nivel de Bloom |
|---|---|---|
| 25 minutos | Media | Aplicar |

## Descripción General

En esta práctica se añade almacenamiento persistente a la aplicación `webapp` creada en el laboratorio 12. Se creará un `PersistentVolumeClaim` (PVC) de `1Gi` usando la clase de almacenamiento `standard` de Minikube y se montará en la ruta web de NGINX, `/usr/share/nginx/html`.

Un `initContainer` basado en `busybox:1.37.0` inicializará `index.html` solo cuando el archivo no exista. Finalmente, se creará un archivo de prueba, se eliminará el Pod de la aplicación y se comprobará que el nuevo Pod puede leer el contenido previamente almacenado.

> Este laboratorio usa almacenamiento local dinámico de Minikube. Es adecuado para aprendizaje y validación local, pero no representa una solución de almacenamiento distribuido para producción.

## Objetivos de Aprendizaje

- [ ] Inspeccionar la `StorageClass` estándar disponible en Minikube.
- [ ] Crear el PVC `webapp-content-pvc` con `ReadWriteOnce`, `1Gi` y `storageClassName: standard`.
- [ ] Montar el PVC en `/usr/share/nginx/html` dentro del contenedor `webapp`.
- [ ] Inicializar contenido web persistente con un `initContainer` que no sobrescriba archivos existentes.
- [ ] Demostrar que un archivo permanece disponible después de eliminar y recrear un Pod gestionado por un Deployment.

## Prerrequisitos

Conocimientos requeridos:

- Diferencia entre almacenamiento efímero del contenedor y almacenamiento persistente.
- Uso básico de `kubectl`, manifiestos YAML, Pods, Deployments y Services.
- Comprensión de que un `PersistentVolumeClaim` solicita almacenamiento y que una `StorageClass` puede aprovisionarlo dinámicamente.
- Laboratorio `12-00-01` completado: deben existir el Deployment `webapp` y el Service `webapp-svc` en el Namespace `laboratorio-k8s`.

Acceso requerido:

- Terminal Bash con acceso a Internet durante la preparación del entorno.
- Docker Engine operativo y permiso del usuario para ejecutar `docker`.
- Clúster Minikube iniciado con el controlador Docker.
- Contexto activo de `kubectl`: `minikube`.
- Permisos para crear y modificar recursos en el Namespace `laboratorio-k8s`.

## Entorno de Laboratorio

### Versiones de referencia y fuentes oficiales

| Componente | Versión, edición y arquitectura | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, AMD64/x86_64 | <https://releases.ubuntu.com/22.04/> |
| Docker Engine | Docker Engine Community Edition 27.2.0, AMD64/x86_64 | <https://docs.docker.com/engine/install/ubuntu/> |
| Minikube | Minikube 1.34.0, Linux AMD64/x86_64 | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| kubectl | Kubernetes CLI 1.31.0, Linux AMD64/x86_64 | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| Kubernetes | Kubernetes v1.31.0 | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Imagen de aplicación | NGINX Docker Official Image `1.27.4-alpine`, AMD64 | <https://hub.docker.com/_/nginx> |
| Imagen del initContainer y validaciones | BusyBox Docker Official Image `1.37.0`, AMD64 | <https://hub.docker.com/_/busybox> |
| Aprovisionador local de Minikube | Addon de almacenamiento incluido con Minikube 1.34.0; versión interna del controlador: `[VERSIÓN POR VALIDAR]` | <https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/> |

### Preparación inicial

Ejecute los comandos siguientes desde la terminal. No continúe si el contexto actual no es `minikube`.

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis
cd ~/k8s-intro

minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb

kubectl config current-context
kubectl get namespace laboratorio-k8s
kubectl get deployment,service -n laboratorio-k8s
```

Salida esperada resumida:

```text
minikube
NAME              READY   UP-TO-DATE   AVAILABLE
deployment/webapp 1/1     1            1

NAME         TYPE        CLUSTER-IP
service/webapp-svc   ClusterIP   ...
```

Si Minikube ya estaba iniciado, confirme su estado:

```bash
minikube status
kubectl version --client=true
kubectl cluster-info
```

## Instrucciones Paso a Paso

### Paso 1: Confirmar el contexto, Namespace y aplicación base

**Objetivo:** Verificar que se trabajará sobre el clúster, Namespace y recursos correctos antes de crear almacenamiento persistente.

**Instrucciones:**

1. Compruebe el contexto configurado para `kubectl`.

   ```bash
   kubectl config current-context
   ```

2. Defina una variable de sesión para evitar repetir el Namespace.

   ```bash
   export NS=laboratorio-k8s
   ```

3. Inspeccione el Deployment, el Service y los Pods existentes.

   ```bash
   kubectl get deployment webapp -n "$NS"
   kubectl get service webapp-svc -n "$NS"
   kubectl get pods -n "$NS" -l app=webapp -o wide
   ```

4. Compruebe el nombre del contenedor del Deployment. El manifiesto de este laboratorio presupone que se llama `webapp`.

   ```bash
   kubectl get deployment webapp -n "$NS" \
     -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
   ```

5. Guarde una evidencia inicial de los recursos existentes.

   ```bash
   kubectl get deployment,service,pods -n "$NS" -o wide \
     | tee ~/k8s-intro/analisis/13-00-01-estado-inicial.txt
   ```

**Salida esperada:**

```text
minikube
webapp
```

Debe existir al menos un Pod gestionado por `deployment/webapp`, normalmente con un nombre similar a:

```text
webapp-6f7c8d9d6c-abcde
```

**Verificación:**

- `kubectl config current-context` devuelve exactamente `minikube`.
- El Namespace es `laboratorio-k8s`.
- Existen `deployment/webapp` y `service/webapp-svc`.
- El contenedor principal del Deployment se llama `webapp`. Si tiene otro nombre, modifique únicamente el valor de `name` en el manifiesto del Paso 4 antes de aplicarlo.

---

### Paso 2: Inspeccionar la clase de almacenamiento estándar

**Objetivo:** Confirmar que Minikube dispone de una `StorageClass` predeterminada que aprovisionará almacenamiento de forma dinámica.

**Instrucciones:**

1. Liste las clases de almacenamiento del clúster.

   ```bash
   kubectl get storageclass
   ```

2. Inspeccione específicamente la clase `standard`.

   ```bash
   kubectl describe storageclass standard
   ```

3. Guarde una copia de la definición para la evidencia técnica del laboratorio.

   ```bash
   kubectl get storageclass standard -o yaml \
     > ~/k8s-intro/analisis/13-00-01-storageclass-standard.yaml
   ```

4. Verifique que Minikube tiene habilitado el aprovisionamiento de almacenamiento local.

   ```bash
   minikube addons list | grep -E 'storage-provisioner|default-storageclass'
   ```

**Salida esperada:**

La salida de `kubectl get storageclass` debe incluir una línea similar a la siguiente:

```text
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate
```

La implementación exacta del provisionador puede variar según la distribución de Minikube, pero `standard` debe estar disponible y marcada como clase predeterminada o ser utilizable explícitamente.

**Verificación:**

Ejecute:

```bash
kubectl get storageclass standard \
  -o jsonpath='{.metadata.name}{" | provisioner="}{.provisioner}{"\n"}'
```

El resultado debe comenzar con:

```text
standard |
```

---

### Paso 3: Crear el PersistentVolumeClaim de la aplicación

**Objetivo:** Solicitar `1Gi` de almacenamiento persistente con acceso `ReadWriteOnce` mediante la clase `standard`.

**Instrucciones:**

1. Cree el manifiesto del PVC.

   ```bash
   cat > ~/k8s-intro/manifests/webapp-content-pvc.yaml <<'EOF'
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: webapp-content-pvc
     namespace: laboratorio-k8s
     labels:
       app: webapp
       laboratorio: 13-00-01
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: standard
     resources:
       requests:
         storage: 1Gi
   EOF
   ```

2. Revise el manifiesto antes de aplicarlo.

   ```bash
   cat ~/k8s-intro/manifests/webapp-content-pvc.yaml
   ```

3. Aplique el PVC.

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-content-pvc.yaml
   ```

4. Espere a que el PVC quede asociado a un volumen.

   ```bash
   kubectl wait \
     --namespace "$NS" \
     --for=jsonpath='{.status.phase}'=Bound \
     pvc/webapp-content-pvc \
     --timeout=120s
   ```

5. Inspeccione el PVC y el PersistentVolume creado dinámicamente.

   ```bash
   kubectl get pvc webapp-content-pvc -n "$NS"
   kubectl get pv
   kubectl describe pvc webapp-content-pvc -n "$NS"
   ```

6. Guarde la evidencia del PVC.

   ```bash
   kubectl get pvc webapp-content-pvc -n "$NS" -o yaml \
     > ~/k8s-intro/analisis/13-00-01-pvc-bound.yaml
   ```

**Salida esperada:**

```text
persistentvolumeclaim/webapp-content-pvc created
persistentvolumeclaim/webapp-content-pvc condition met
```

La consulta del PVC debe mostrar un estado similar a:

```text
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
webapp-content-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            standard        ...
```

**Verificación:**

```bash
kubectl get pvc webapp-content-pvc -n "$NS" \
  -o jsonpath='{.status.phase}{" | "}{.spec.resources.requests.storage}{" | "}{.spec.accessModes[0]}{"\n"}'
```

El resultado debe ser:

```text
Bound | 1Gi | ReadWriteOnce
```

---

### Paso 4: Montar el PVC e inicializar el contenido web

**Objetivo:** Modificar el Deployment para montar el PVC en la raíz web de NGINX e inicializar `index.html` solo cuando aún no exista.

**Instrucciones:**

1. Cree un parche estratégico para el Deployment. El valor `replicas: 1` es intencional: esta práctica utiliza un único escritor sobre un volumen `ReadWriteOnce`.

   ```bash
   cat > ~/k8s-intro/manifests/webapp-persistent-storage-patch.yaml <<'EOF'
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: webapp
     namespace: laboratorio-k8s
   spec:
     replicas: 1
     template:
       spec:
         initContainers:
           - name: initialize-content
             image: busybox:1.37.0
             imagePullPolicy: IfNotPresent
             command:
               - /bin/sh
               - -c
             args:
               - |
                 set -eu
                 if [ ! -f /usr/share/nginx/html/index.html ]; then
                   cat > /usr/share/nginx/html/index.html <<'HTML'
                   <!doctype html>
                   <html lang="es">
                     <head><meta charset="utf-8"><title>Webapp persistente</title></head>
                     <body>
                       <h1>Webapp con almacenamiento persistente</h1>
                       <p>Este archivo fue inicializado por un initContainer.</p>
                     </body>
                   </html>
                   HTML
                   echo "index.html creado en el volumen persistente."
                 else
                   echo "index.html ya existe; se conserva el contenido persistente."
                 fi
             volumeMounts:
               - name: webapp-content
                 mountPath: /usr/share/nginx/html
         containers:
           - name: webapp
             image: nginx:1.27.4-alpine
             volumeMounts:
               - name: webapp-content
                 mountPath: /usr/share/nginx/html
         volumes:
           - name: webapp-content
             persistentVolumeClaim:
               claimName: webapp-content-pvc
   EOF
   ```

2. Valide que el nombre de contenedor de la sección `containers` coincide con el comprobado en el Paso 1.

   ```bash
   grep -n 'name: webapp' ~/k8s-intro/manifests/webapp-persistent-storage-patch.yaml
   ```

3. Aplique el parche estratégico.

   ```bash
   kubectl patch deployment webapp \
     --namespace "$NS" \
     --type=strategic \
     --patch-file ~/k8s-intro/manifests/webapp-persistent-storage-patch.yaml
   ```

4. Espere a que Kubernetes complete el rollout.

   ```bash
   kubectl rollout status deployment/webapp -n "$NS" --timeout=180s
   ```

5. Revise el Pod creado y el estado del `initContainer`.

   ```bash
   kubectl get pods -n "$NS" -l app=webapp
   kubectl get pods -n "$NS" -l app=webapp \
     -o jsonpath='{range .items[*]}{.metadata.name}{" | init="}{.status.initContainerStatuses[0].state.terminated.reason}{"\n"}{end}'
   ```

6. Consulte los registros del `initContainer`.

   ```bash
   POD_ACTUAL=$(kubectl get pods -n "$NS" -l app=webapp \
     -o jsonpath='{.items[0].metadata.name}')

   kubectl logs "$POD_ACTUAL" -n "$NS" -c initialize-content
   ```

**Salida esperada:**

```text
deployment.apps/webapp patched
deployment "webapp" successfully rolled out
```

En la primera ejecución, el registro del contenedor de inicialización debe incluir:

```text
index.html creado en el volumen persistente.
```

**Verificación:**

Compruebe que el Pod monta un volumen cuyo origen es el PVC:

```bash
kubectl describe pod "$POD_ACTUAL" -n "$NS" | sed -n '/Volumes:/,/QoS Class:/p'
```

Debe aparecer información equivalente a:

```text
webapp-content:
  Type:       PersistentVolumeClaim
  ClaimName:  webapp-content-pvc
```

---

### Paso 5: Comprobar que NGINX sirve contenido desde el PVC

**Objetivo:** Verificar que el contenido inicializado en el volumen persistente es accesible mediante el Service `webapp-svc`.

**Instrucciones:**

1. Consulte el contenido de `index.html` directamente desde el contenedor NGINX.

   ```bash
   kubectl exec -n "$NS" "$POD_ACTUAL" -c webapp -- \
     cat /usr/share/nginx/html/index.html
   ```

2. Compruebe que el Service tiene endpoints disponibles.

   ```bash
   kubectl get endpoints webapp-svc -n "$NS"
   ```

3. Cree un Pod temporal de validación HTTP usando BusyBox 1.37.0. Este Pod se elimina automáticamente al finalizar el comando.

   ```bash
   kubectl run http-check \
     --namespace "$NS" \
     --rm \
     -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/'
   ```

4. Guarde el resultado HTTP como evidencia.

   ```bash
   kubectl run http-check-evidence \
     --namespace "$NS" \
     --rm \
     -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/' \
     | tee ~/k8s-intro/analisis/13-00-01-index-http.html
   ```

**Salida esperada:**

La respuesta HTTP debe contener, como mínimo:

```html
<h1>Webapp con almacenamiento persistente</h1>
```

**Verificación:**

```bash
grep -F 'Webapp con almacenamiento persistente' \
  ~/k8s-intro/analisis/13-00-01-index-http.html
```

El comando debe mostrar la línea que contiene el título de la página.

---

### Paso 6: Crear un archivo persistente de prueba

**Objetivo:** Escribir un archivo en el volumen montado y registrar un valor único que permita comprobar su persistencia después de reemplazar el Pod.

**Instrucciones:**

1. Obtenga el nombre del Pod actual y cree un identificador de prueba único.

   ```bash
   POD_ANTES=$(kubectl get pods -n "$NS" -l app=webapp \
     -o jsonpath='{.items[0].metadata.name}')

   TOKEN="persistencia-$(date +%Y%m%d-%H%M%S)"
   echo "$TOKEN"
   ```

2. Cree el archivo `persistencia.txt` dentro de la ruta montada desde el PVC.

   ```bash
   kubectl exec -n "$NS" "$POD_ANTES" -c webapp -- \
     sh -c "printf '%s\n' '$TOKEN' > /usr/share/nginx/html/persistencia.txt"
   ```

3. Lea el archivo desde el contenedor para comprobar la escritura.

   ```bash
   kubectl exec -n "$NS" "$POD_ANTES" -c webapp -- \
     cat /usr/share/nginx/html/persistencia.txt
   ```

4. Verifique que NGINX publica también el archivo nuevo mediante el Service.

   ```bash
   kubectl run http-persistence-check \
     --namespace "$NS" \
     --rm \
     -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/persistencia.txt'
   ```

5. Registre el nombre del Pod original y el token generado.

   ```bash
   {
     echo "POD_ANTES=$POD_ANTES"
     echo "TOKEN=$TOKEN"
   } | tee ~/k8s-intro/analisis/13-00-01-datos-prueba.txt
   ```

**Salida esperada:**

Los comandos de lectura local y HTTP deben devolver exactamente el valor del token, por ejemplo:

```text
persistencia-20261009-101530
```

**Verificación:**

```bash
kubectl exec -n "$NS" "$POD_ANTES" -c webapp -- \
  cat /usr/share/nginx/html/persistencia.txt | grep -Fx "$TOKEN"
```

El comando debe devolver una coincidencia exacta.

---

### Paso 7: Eliminar el Pod y demostrar la persistencia

**Objetivo:** Confirmar que el Deployment recrea el Pod y que el archivo almacenado en el PVC sigue disponible en el nuevo Pod.

**Instrucciones:**

1. Elimine únicamente el Pod actual. No elimine el Deployment ni el PVC.

   ```bash
   kubectl delete pod "$POD_ANTES" -n "$NS"
   ```

2. Espere a que desaparezca el Pod eliminado.

   ```bash
   kubectl wait --for=delete pod/"$POD_ANTES" -n "$NS" --timeout=120s
   ```

3. Espere a que el Deployment tenga de nuevo una réplica disponible.

   ```bash
   kubectl rollout status deployment/webapp -n "$NS" --timeout=180s
   ```

4. Obtenga el nombre del nuevo Pod.

   ```bash
   POD_DESPUES=$(kubectl get pods -n "$NS" -l app=webapp \
     -o jsonpath='{.items[0].metadata.name}')

   echo "Pod anterior: $POD_ANTES"
   echo "Pod nuevo:    $POD_DESPUES"
   ```

5. Compruebe que se trata de un Pod diferente.

   ```bash
   test "$POD_ANTES" != "$POD_DESPUES" && \
     echo "Correcto: el Deployment creó un Pod nuevo."
   ```

6. Lea el archivo persistente desde el nuevo Pod.

   ```bash
   kubectl exec -n "$NS" "$POD_DESPUES" -c webapp -- \
     cat /usr/share/nginx/html/persistencia.txt
   ```

7. Revise los registros del nuevo `initContainer`. Esta vez debe detectar que `index.html` ya existe y conservarlo.

   ```bash
   kubectl logs "$POD_DESPUES" -n "$NS" -c initialize-content
   ```

8. Valide el archivo por HTTP a través del Service.

   ```bash
   kubectl run http-final-check \
     --namespace "$NS" \
     --rm \
     -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/persistencia.txt'
   ```

9. Guarde las evidencias finales.

   ```bash
   kubectl get pvc,pv -n "$NS" -o wide \
     > ~/k8s-intro/analisis/13-00-01-almacenamiento-final.txt

   kubectl get pods -n "$NS" -l app=webapp -o wide \
     >> ~/k8s-intro/analisis/13-00-01-almacenamiento-final.txt

   kubectl logs "$POD_DESPUES" -n "$NS" -c initialize-content \
     > ~/k8s-intro/analisis/13-00-01-initcontainer-recreado.log
   ```

**Salida esperada:**

El archivo `persistencia.txt` debe contener el mismo valor del token creado antes de eliminar el Pod.

El registro del nuevo `initContainer` debe mostrar:

```text
index.html ya existe; se conserva el contenido persistente.
```

**Verificación:**

```bash
kubectl exec -n "$NS" "$POD_DESPUES" -c webapp -- \
  cat /usr/share/nginx/html/persistencia.txt | grep -Fx "$TOKEN"
```

La validación es correcta solo si:

- El nombre de `POD_DESPUES` es distinto de `POD_ANTES`.
- El PVC continúa en estado `Bound`.
- El nuevo Pod lee exactamente el mismo token.
- El archivo puede obtenerse mediante `http://webapp-svc/persistencia.txt`.

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios medibles siguientes:

| Criterio | Comando de comprobación | Resultado exigido | Evidencia |
|---|---|---|---|
| StorageClass disponible | `kubectl get storageclass standard` | Existe `standard` | `13-00-01-storageclass-standard.yaml` |
| PVC aprovisionado | `kubectl get pvc webapp-content-pvc -n laboratorio-k8s` | `STATUS=Bound`, `CAPACITY=1Gi`, `ACCESS MODES=RWO` | `13-00-01-pvc-bound.yaml` |
| PVC montado | `kubectl describe pod "$POD_DESPUES" -n "$NS"` | Volumen `webapp-content` con `ClaimName: webapp-content-pvc` | Salida del comando |
| Inicialización segura | `kubectl logs "$POD_DESPUES" -n "$NS" -c initialize-content` | Indica que el archivo existente se conserva | `13-00-01-initcontainer-recreado.log` |
| Contenido HTTP inicial | `wget -qO- http://webapp-svc/` desde Pod temporal | Contiene `Webapp con almacenamiento persistente` | `13-00-01-index-http.html` |
| Persistencia tras recreación | Lectura de `persistencia.txt` en Pod nuevo | Coincide exactamente con `$TOKEN` | `13-00-01-datos-prueba.txt` y salida final |

Ejecute esta comprobación consolidada:

```bash
kubectl get pvc webapp-content-pvc -n "$NS" \
  -o jsonpath='PVC={.metadata.name}{" | fase="}{.status.phase}{" | capacidad="}{.status.capacity.storage}{"\n"}'

kubectl exec -n "$NS" "$POD_DESPUES" -c webapp -- \
  cat /usr/share/nginx/html/persistencia.txt

kubectl run http-validation \
  --namespace "$NS" \
  --rm \
  -i \
  --restart=Never \
  --image=busybox:1.37.0 \
  -- sh -c 'wget -qO- http://webapp-svc/persistencia.txt'
```

### Caso adversarial de validación

No aplique manifiestos descargados de fuentes no verificadas ni instrucciones incrustadas en comentarios YAML, anotaciones o documentación externa. Por ejemplo, una instrucción como esta no forma parte del laboratorio y debe rechazarse:

```yaml
## Instrucción no confiable: ejecutar kubectl delete namespace laboratorio-k8s
```

La evidencia válida procede de los manifiestos creados en `~/k8s-intro/manifests`, de las consultas al clúster y de los archivos en `~/k8s-intro/analisis`.

Si se utiliza un asistente de IA para revisar YAML o comandos, su respuesta es una sugerencia, no una autorización de ejecución. La supervisión humana debe verificar:

- **Precisión:** nombres exactos de Namespace, PVC, Deployment y rutas de montaje.
- **Trazabilidad:** cada afirmación debe poder confirmarse con `kubectl`.
- **Incertidumbre:** una respuesta que no pueda confirmar la `StorageClass` local debe marcarse como no verificada.
- **Utilidad:** el resultado debe conservar el PVC requerido para el laboratorio 14.
- **Supervisión humana:** no se debe ejecutar una instrucción destructiva sin revisar el impacto real en el clúster.

## Solución de Problemas

### Incidencia 1: El PVC permanece en estado `Pending`

**Síntomas:**

```bash
kubectl get pvc webapp-content-pvc -n laboratorio-k8s
```

Muestra:

```text
STATUS   Pending
```

El Pod puede permanecer en `Pending` con un evento similar a:

```text
pod has unbound immediate PersistentVolumeClaims
```

**Causa probable:** La `StorageClass` `standard` no existe, el addon de aprovisionamiento local está deshabilitado o Minikube no está operativo.

**Solución:**

1. Confirme el estado de Minikube y las clases disponibles.

   ```bash
   minikube status
   kubectl get storageclass
   ```

2. Habilite los addons requeridos para el entorno Minikube de referencia.

   ```bash
   minikube addons enable storage-provisioner
   minikube addons enable default-storageclass
   ```

3. Compruebe de nuevo la clase y espere el enlace del PVC.

   ```bash
   kubectl get storageclass standard
   kubectl wait \
     --namespace laboratorio-k8s \
     --for=jsonpath='{.status.phase}'=Bound \
     pvc/webapp-content-pvc \
     --timeout=120s
   ```

### Incidencia 2: El Deployment no progresa o el initContainer falla

**Síntomas:**

```bash
kubectl rollout status deployment/webapp -n laboratorio-k8s
```

No finaliza, o el Pod muestra `Init:Error`, `CreateContainerConfigError` o errores de montaje.

**Causa probable:** El nombre del contenedor principal en el parche no coincide con el nombre real del Deployment del laboratorio 12, el PVC no está creado, o existe una referencia incorrecta a `webapp-content-pvc`.

**Solución:**

1. Revise los eventos y el estado del Pod.

   ```bash
   kubectl get pods -n laboratorio-k8s -l app=webapp
   kubectl describe pod -n laboratorio-k8s -l app=webapp
   ```

2. Confirme el nombre real del contenedor y el estado del PVC.

   ```bash
   kubectl get deployment webapp -n laboratorio-k8s \
     -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'

   kubectl get pvc webapp-content-pvc -n laboratorio-k8s
   ```

3. Si el contenedor no se llama `webapp`, edite el archivo `~/k8s-intro/manifests/webapp-persistent-storage-patch.yaml`, cambie solo este valor:

   ```yaml
   containers:
     - name: webapp
   ```

   por el nombre real identificado, y aplique nuevamente el parche:

   ```bash
   kubectl patch deployment webapp \
     --namespace laboratorio-k8s \
     --type=strategic \
     --patch-file ~/k8s-intro/manifests/webapp-persistent-storage-patch.yaml

   kubectl rollout status deployment/webapp -n laboratorio-k8s --timeout=180s
   ```

## Limpieza

El PVC y el volumen deben conservarse porque serán utilizados en el laboratorio 14. Por tanto, **no ejecute** este comando:

```bash
kubectl delete pvc webapp-content-pvc -n laboratorio-k8s
```

Puede eliminar únicamente recursos temporales que hayan quedado por una ejecución interrumpida:

```bash
kubectl delete pod \
  http-check \
  http-check-evidence \
  http-persistence-check \
  http-final-check \
  http-validation \
  -n laboratorio-k8s \
  --ignore-not-found=true
```

Compruebe que los recursos requeridos permanecen disponibles:

```bash
kubectl get deployment webapp -n laboratorio-k8s
kubectl get service webapp-svc -n laboratorio-k8s
kubectl get pvc webapp-content-pvc -n laboratorio-k8s
```

Salida esperada para el PVC:

```text
webapp-content-pvc   Bound   ...   1Gi   RWO   standard
```

## Resumen

En esta práctica se creó el PVC `webapp-content-pvc` con `1Gi`, acceso `ReadWriteOnce` y la clase `standard` de Minikube. El Deployment `webapp` fue modificado para montar ese PVC en `/usr/share/nginx/html`, sustituyendo el contenido efímero de la imagen de NGINX por contenido persistente.

El `initContainer` `initialize-content` creó `index.html` solo durante la primera inicialización del volumen y preservó el contenido existente cuando Kubernetes recreó el Pod. La eliminación del Pod y la lectura posterior del mismo archivo desde un Pod nuevo demostraron que los datos sobrevivieron al ciclo de vida de la instancia concreta del contenedor.

### Recursos oficiales

- Kubernetes: Persistent Volumes: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>
- Kubernetes: PersistentVolumeClaims: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>
- Kubernetes: Volúmenes: <https://kubernetes.io/docs/concepts/storage/volumes/>
- Minikube: volúmenes persistentes: <https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/>
- NGINX Docker Official Image: <https://hub.docker.com/_/nginx>
- BusyBox Docker Official Image: <https://hub.docker.com/_/busybox>
