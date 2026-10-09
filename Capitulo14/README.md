# Práctica: Configuración de una aplicación

## Metadatos

| Campo | Valor |
|---|---|
| Duration | 25 minutos |
| Complexity | Fácil |
| Bloom level | Apply |

## Descripción General

En esta práctica se separará la configuración no sensible de la imagen de una aplicación web existente. Se creará el ConfigMap `webapp-config` con un archivo `config.json`, se montará exclusivamente esa clave mediante `subPath` en el Deployment `webapp` y se comprobará su publicación mediante el Service interno `webapp-svc`.

Posteriormente, se actualizará el ConfigMap y se ejecutará un reinicio controlado del Deployment para que los nuevos Pods consuman la configuración modificada. No se almacenarán contraseñas, tokens, certificados ni claves privadas en un ConfigMap.

## Objetivos de Aprendizaje

- [ ] Crear un ConfigMap denominado `webapp-config` que contenga el archivo no sensible `config.json`.
- [ ] Montar únicamente la clave `config.json` en `/usr/share/nginx/html/config.json` mediante un volumen ConfigMap y `subPath`.
- [ ] Validar el contenido publicado a través del Service interno `webapp-svc`.
- [ ] Actualizar declarativamente el ConfigMap y reiniciar controladamente el Deployment `webapp`.
- [ ] Explicar por qué un ConfigMap no es apropiado para secretos, contraseñas, tokens o claves privadas.

## Prerrequisitos

**Conocimientos requeridos**

- Comprensión básica de manifiestos YAML de Kubernetes.
- Uso básico de `kubectl`, Namespaces, Deployments, Services y PersistentVolumeClaims.
- Comprensión de que una imagen de contenedor es un artefacto inmutable y que la configuración puede inyectarse en tiempo de ejecución.
- Haber completado el laboratorio 13-00-01, con el PVC `webapp-content-pvc` montado en el Deployment `webapp`.

**Acceso y estado requerido**

- Acceso a una terminal Bash en Linux.
- Contexto activo de `kubectl`: `minikube`.
- Namespace heredado para esta práctica: `curso-k8s`.
- Deployment existente: `webapp`.
- Service existente de tipo `ClusterIP`: `webapp-svc`.
- PVC existente: `webapp-content-pvc`.
- Permisos para consultar, crear y actualizar ConfigMaps y Deployments dentro de `curso-k8s`.

> **Nota de alcance:** el directorio global del curso es `~/k8s-intro`; los manifiestos se guardan en `~/k8s-intro/manifests` y las evidencias en `~/k8s-intro/analisis`. Aunque el entorno global del curso reserva `laboratorio-k8s` para otros ejercicios, esta práctica utiliza explícitamente el Namespace heredado `curso-k8s` indicado en su especificación. No se desplegarán aplicaciones del estudiante en `kube-system`.

## Entorno de Laboratorio

| Componente | Versión, edición y arquitectura | Fuente oficial |
|---|---|---|
| Sistema operativo de referencia | Ubuntu 22.04.5 LTS, amd64 | https://releases.ubuntu.com/22.04.5/ |
| Docker Engine | 27.2.0, Community Edition, amd64 | https://docs.docker.com/engine/release-notes/27/ |
| Minikube | 1.34.0, Linux amd64 | https://github.com/kubernetes/minikube/releases/tag/v1.34.0 |
| Kubernetes | v1.31.0, linux/amd64 | https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0 |
| kubectl | v1.31.0, linux/amd64 | https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl |
| Imagen de la aplicación heredada | nginx:1.27.4-alpine, OCI linux/amd64 | https://hub.docker.com/_/nginx |
| Imagen de comprobación temporal | busybox:1.37.0, OCI linux/amd64 | https://hub.docker.com/_/busybox |

El perfil de Minikube esperado es `minikube`, con controlador Docker. Si el clúster no está iniciado, utilice el comando de referencia siguiente:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

Prepare los directorios de trabajo y compruebe el estado base:

```bash
mkdir -p ~/k8s-intro/manifests ~/k8s-intro/analisis
cd ~/k8s-intro

kubectl config current-context
minikube status
kubectl get namespace curso-k8s
kubectl -n curso-k8s get deployment webapp
kubectl -n curso-k8s get service webapp-svc
kubectl -n curso-k8s get pvc webapp-content-pvc
```

El resultado esperado incluye:

- Contexto actual: `minikube`.
- Perfil de Minikube en estado `Running`.
- Namespace `curso-k8s` existente.
- Recursos `webapp`, `webapp-svc` y `webapp-content-pvc` visibles.

## Instrucciones Paso a Paso

### Paso 1: Verificar el estado heredado de la aplicación

**Objetivo**

Confirmar que el clúster, el Namespace y los recursos creados en laboratorios anteriores están disponibles antes de modificar el Deployment.

**Instrucciones**

1. Verifique que el contexto activo es `minikube`:

   ```bash
   kubectl config current-context
   ```

2. Consulte el estado de los recursos requeridos:

   ```bash
   kubectl -n curso-k8s get deployment webapp
   kubectl -n curso-k8s get service webapp-svc
   kubectl -n curso-k8s get pvc webapp-content-pvc
   ```

3. Identifique el nombre del contenedor definido en el Deployment:

   ```bash
   kubectl -n curso-k8s get deployment webapp \
     -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
   ```

4. Inspeccione los montajes actuales para confirmar que el PVC ya está asociado a la aplicación:

   ```bash
   kubectl -n curso-k8s describe deployment webapp
   ```

5. Guarde una evidencia inicial del estado del Deployment:

   ```bash
   kubectl -n curso-k8s get deployment webapp -o yaml \
     > ~/k8s-intro/analisis/14-00-01-webapp-antes.yaml
   ```

**Resultado esperado**

- El contexto mostrado es `minikube`.
- El Deployment `webapp` tiene réplicas disponibles.
- El Service `webapp-svc` existe y es de tipo `ClusterIP`.
- El PVC `webapp-content-pvc` tiene estado `Bound`.
- El nombre del contenedor principal es `webapp`.

**Verificación**

Ejecute:

```bash
kubectl -n curso-k8s get deployment webapp
kubectl -n curso-k8s get pvc webapp-content-pvc
```

La columna `AVAILABLE` del Deployment debe ser igual al número deseado de réplicas, y el PVC debe mostrar `STATUS` igual a `Bound`.

> Si el contenedor no se llama `webapp`, sustituya únicamente el valor de `name: webapp` del manifiesto de parche del paso 3 por el nombre obtenido en este paso. No cambie el nombre del Deployment.

---

### Paso 2: Crear el ConfigMap con la configuración no sensible

**Objetivo**

Definir declarativamente el ConfigMap `webapp-config` con el archivo `config.json` que la aplicación publicará.

**Instrucciones**

1. Cree el manifiesto `~/k8s-intro/manifests/webapp-config.yaml`:

   ```bash
   cat > ~/k8s-intro/manifests/webapp-config.yaml <<'EOF'
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: webapp-config
     namespace: curso-k8s
     labels:
       app: webapp
       laboratorio: 14-00-01
   data:
     config.json: |
       {
         "applicationName": "webapp",
         "environment": "desarrollo",
         "supportEmail": "soporte-desarrollo@example.com"
       }
   EOF
   ```

2. Revise el contenido antes de aplicarlo:

   ```bash
   cat ~/k8s-intro/manifests/webapp-config.yaml
   ```

3. Aplique el ConfigMap:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-config.yaml
   ```

4. Consulte el recurso creado:

   ```bash
   kubectl -n curso-k8s get configmap webapp-config
   kubectl -n curso-k8s get configmap webapp-config \
     -o jsonpath='{.data.config\.json}{"\n"}'
   ```

**Resultado esperado**

Debe crearse el ConfigMap `webapp-config` con una clave denominada `config.json`. El valor de la clave debe contener un documento JSON con estos tres parámetros no sensibles:

- `applicationName`
- `environment`
- `supportEmail`

**Verificación**

Ejecute:

```bash
kubectl -n curso-k8s describe configmap webapp-config
```

La sección `Data` debe indicar una entrada llamada `config.json`.

> **Seguridad:** un ConfigMap es adecuado para datos no sensibles, como nombres de entorno, niveles de registro, direcciones no confidenciales o parámetros funcionales. No incluya contraseñas, tokens de acceso, certificados, claves API ni claves privadas. Esos valores requieren un recurso `Secret` y controles de acceso adecuados.

---

### Paso 3: Montar únicamente config.json en el Deployment

**Objetivo**

Actualizar el Deployment `webapp` para montar exclusivamente la clave `config.json` en `/usr/share/nginx/html/config.json`, sin reemplazar el directorio persistente completo respaldado por el PVC.

**Instrucciones**

1. Cree el parche estratégico `~/k8s-intro/manifests/webapp-config-mount-patch.yaml`:

   ```bash
   cat > ~/k8s-intro/manifests/webapp-config-mount-patch.yaml <<'EOF'
   spec:
     template:
       spec:
         volumes:
           - name: webapp-config-volume
             configMap:
               name: webapp-config
               items:
                 - key: config.json
                   path: config.json
         containers:
           - name: webapp
             volumeMounts:
               - name: webapp-config-volume
                 mountPath: /usr/share/nginx/html/config.json
                 subPath: config.json
                 readOnly: true
   EOF
   ```

2. Revise el parche:

   ```bash
   cat ~/k8s-intro/manifests/webapp-config-mount-patch.yaml
   ```

3. Aplique el parche al Deployment existente:

   ```bash
   kubectl -n curso-k8s patch deployment webapp \
     --type=strategic \
     --patch-file ~/k8s-intro/manifests/webapp-config-mount-patch.yaml
   ```

4. Espere a que finalice el despliegue progresivo provocado por el cambio en la plantilla de Pods:

   ```bash
   kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s
   ```

5. Consulte los volúmenes y montajes registrados en el Deployment:

   ```bash
   kubectl -n curso-k8s describe deployment webapp
   ```

6. Guarde evidencia del manifiesto efectivo:

   ```bash
   kubectl -n curso-k8s get deployment webapp -o yaml \
     > ~/k8s-intro/analisis/14-00-01-webapp-con-configmap.yaml
   ```

**Resultado esperado**

El Deployment debe iniciar nuevos Pods. En su especificación debe aparecer:

- Un volumen denominado `webapp-config-volume`.
- Una referencia al ConfigMap `webapp-config`.
- La selección exclusiva de la clave `config.json` mediante `items`.
- Un `volumeMount` de solo lectura.
- `mountPath: /usr/share/nginx/html/config.json`.
- `subPath: config.json`.

El directorio `/usr/share/nginx/html` continúa asociado al volumen persistente existente. Solo el archivo `config.json` se superpone mediante el montaje `subPath`.

**Verificación**

Ejecute:

```bash
kubectl -n curso-k8s get pods -l app=webapp
kubectl -n curso-k8s describe deployment webapp
```

En la descripción del Deployment, compruebe que el volumen ConfigMap y el montaje del archivo están presentes. Los Pods deben estar en estado `Running` y `Ready`.

---

### Paso 4: Validar la publicación del archivo mediante el Service interno

**Objetivo**

Comprobar que el archivo montado desde el ConfigMap es servido por NGINX y se puede consultar usando el Service interno `webapp-svc`.

**Instrucciones**

1. Consulte el Service para confirmar su tipo y puerto:

   ```bash
   kubectl -n curso-k8s get service webapp-svc
   ```

2. Ejecute un Pod temporal con BusyBox para consultar el endpoint interno. El parámetro `--rm` elimina el Pod al finalizar:

   ```bash
   kubectl -n curso-k8s run webapp-config-check \
     --rm -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/config.json'
   ```

3. Repita la consulta y almacene la evidencia local:

   ```bash
   kubectl -n curso-k8s run webapp-config-evidence \
     --rm -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/config.json' \
     | tee ~/k8s-intro/analisis/14-00-01-config-inicial.json
   ```

4. Valide que la respuesta contiene los tres nombres de parámetros esperados:

   ```bash
   grep -E '"applicationName"|"environment"|"supportEmail"' \
     ~/k8s-intro/analisis/14-00-01-config-inicial.json
   ```

**Resultado esperado**

La consulta al Service debe devolver el contenido JSON:

```json
{
  "applicationName": "webapp",
  "environment": "desarrollo",
  "supportEmail": "soporte-desarrollo@example.com"
}
```

No debe ser necesario exponer el Service fuera del clúster para realizar esta comprobación.

**Verificación**

El siguiente comando debe devolver un código HTTP implícitamente correcto y el valor `desarrollo`:

```bash
kubectl -n curso-k8s run webapp-config-check \
  --rm -i \
  --restart=Never \
  --image=busybox:1.37.0 \
  -- sh -c 'wget -qO- http://webapp-svc/config.json' \
  | grep '"environment": "desarrollo"'
```

---

### Paso 5: Actualizar la configuración y reiniciar controladamente el Deployment

**Objetivo**

Modificar el ConfigMap con nuevos valores no sensibles y reiniciar controladamente el Deployment para asegurar que los Pods que usan `subPath` consumen la nueva configuración.

**Instrucciones**

1. Sobrescriba el manifiesto del ConfigMap con los valores actualizados:

   ```bash
   cat > ~/k8s-intro/manifests/webapp-config.yaml <<'EOF'
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: webapp-config
     namespace: curso-k8s
     labels:
       app: webapp
       laboratorio: 14-00-01
   data:
     config.json: |
       {
         "applicationName": "webapp",
         "environment": "pruebas",
         "supportEmail": "soporte-pruebas@example.com"
       }
   EOF
   ```

2. Aplique el cambio declarativo:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/webapp-config.yaml
   ```

3. Compruebe que el objeto ConfigMap contiene los nuevos valores:

   ```bash
   kubectl -n curso-k8s get configmap webapp-config \
     -o jsonpath='{.data.config\.json}{"\n"}'
   ```

4. Reinicie controladamente el Deployment:

   ```bash
   kubectl -n curso-k8s rollout restart deployment/webapp
   ```

5. Espere a que finalice el reemplazo progresivo de Pods:

   ```bash
   kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s
   ```

6. Consulte el historial del Deployment:

   ```bash
   kubectl -n curso-k8s rollout history deployment/webapp
   ```

7. Pruebe de nuevo el archivo servido a través del Service:

   ```bash
   kubectl -n curso-k8s run webapp-config-check \
     --rm -i \
     --restart=Never \
     --image=busybox:1.37.0 \
     -- sh -c 'wget -qO- http://webapp-svc/config.json' \
     | tee ~/k8s-intro/analisis/14-00-01-config-actualizado.json
   ```

**Resultado esperado**

- El ConfigMap contiene `environment: "pruebas"`.
- `kubectl rollout restart` crea nuevos Pods para el Deployment.
- `kubectl rollout status` informa de que el despliegue se completó correctamente.
- El Service interno devuelve el nuevo archivo `config.json`.

**Verificación**

Ejecute:

```bash
grep '"environment": "pruebas"' \
  ~/k8s-intro/analisis/14-00-01-config-actualizado.json

grep '"supportEmail": "soporte-pruebas@example.com"' \
  ~/k8s-intro/analisis/14-00-01-config-actualizado.json
```

Ambos comandos deben devolver una línea coincidente.

> **Concepto clave:** los montajes que utilizan `subPath` no reciben actualizaciones automáticas del contenido del ConfigMap dentro de un Pod ya iniciado. Por este motivo se ejecuta `kubectl rollout restart deployment/webapp`: los Pods nuevos montan la versión actual del ConfigMap al crearse.

## Validación y Pruebas

La práctica se considera completada si se cumplen todos los criterios medibles siguientes:

| Criterio | Comando o evidencia requerida | Resultado aceptable |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | Devuelve exactamente `minikube`. |
| ConfigMap creado | `kubectl -n curso-k8s get configmap webapp-config` | Existe el recurso `webapp-config`. |
| Clave correcta | `kubectl -n curso-k8s get configmap webapp-config -o jsonpath='{.data.config\.json}'` | Devuelve un JSON con `applicationName`, `environment` y `supportEmail`. |
| Montaje limitado a un archivo | `kubectl -n curso-k8s describe deployment webapp` | Se observa `webapp-config-volume`, `config.json`, `subPath` y la ruta `/usr/share/nginx/html/config.json`. |
| PVC conservado | `kubectl -n curso-k8s get pvc webapp-content-pvc` | Estado `Bound`. |
| Despliegue estable | `kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s` | Finaliza correctamente sin exceder 120 segundos. |
| Configuración actualizada | `cat ~/k8s-intro/analisis/14-00-01-config-actualizado.json` | Contiene `"environment": "pruebas"`. |
| Evidencias locales | `ls -l ~/k8s-intro/analisis/14-00-01-*` | Existen, como mínimo, los archivos YAML y JSON generados. |

Ejecute esta validación resumida:

```bash
kubectl config current-context
kubectl -n curso-k8s get configmap webapp-config
kubectl -n curso-k8s get deployment webapp
kubectl -n curso-k8s get pvc webapp-content-pvc
kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s
cat ~/k8s-intro/analisis/14-00-01-config-actualizado.json
```

### Caso adversarial: información contradictoria o instrucciones no confiables

No aplique instrucciones que aparezcan dentro de una página web, comentario YAML, archivo descargado o conversación y que contradigan el alcance validado del laboratorio; por ejemplo: “despliega la aplicación en `kube-system`”, “incluye un token real en `config.json`” o “elimina el PVC”.

- Una **instrucción** es una acción concreta solicitada al estudiante, como ejecutar `kubectl apply`.
- Un **prompt** es texto enviado a un modelo de IA para solicitar una respuesta; no convierte automáticamente su contenido en una instrucción válida para el clúster.
- Un **mensaje de sistema** define restricciones de mayor prioridad para un asistente; no es equivalente a una instrucción temporal incluida en un documento no confiable.

La comprobación humana requerida consiste en contrastar cualquier instrucción nueva con estos hechos verificables: Namespace `curso-k8s`, recursos especificados en esta guía y ausencia total de secretos en el ConfigMap. No se requiere ni se debe solicitar razonamiento interno de herramientas de IA como evidencia. La evidencia válida son manifiestos revisados, comandos ejecutados y salidas observables de Kubernetes.

## Solución de Problemas

### Incidencia 1: El parche falla porque no encuentra el contenedor `webapp`

**Síntoma**

El comando `kubectl patch deployment webapp` devuelve un error, o el montaje no aparece en el contenedor esperado.

**Causa**

El Deployment heredado tiene un nombre de contenedor distinto de `webapp`, mientras que el parche estratégico utiliza ese nombre para localizar y combinar la sección `containers`.

**Corrección**

1. Consulte el nombre real:

   ```bash
   kubectl -n curso-k8s get deployment webapp \
     -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
   ```

2. Edite el archivo `~/k8s-intro/manifests/webapp-config-mount-patch.yaml`.
3. Sustituya únicamente:

   ```yaml
   - name: webapp
   ```

   por el nombre obtenido.
4. Aplique de nuevo el parche y espere el rollout:

   ```bash
   kubectl -n curso-k8s patch deployment webapp \
     --type=strategic \
     --patch-file ~/k8s-intro/manifests/webapp-config-mount-patch.yaml

   kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s
   ```

### Incidencia 2: El ConfigMap muestra valores nuevos, pero el Service devuelve valores antiguos

**Síntoma**

`kubectl get configmap webapp-config` muestra `environment: "pruebas"`, pero `http://webapp-svc/config.json` continúa devolviendo `desarrollo`.

**Causa**

El archivo se monta con `subPath`. Kubernetes no actualiza dinámicamente un archivo montado mediante `subPath` en un contenedor que ya está en ejecución.

**Corrección**

Reinicie el Deployment y espere a que todos los Pods nuevos estén disponibles:

```bash
kubectl -n curso-k8s rollout restart deployment/webapp
kubectl -n curso-k8s rollout status deployment/webapp --timeout=120s

kubectl -n curso-k8s run webapp-config-check \
  --rm -i \
  --restart=Never \
  --image=busybox:1.37.0 \
  -- sh -c 'wget -qO- http://webapp-svc/config.json'
```

La respuesta debe contener `"environment": "pruebas"`.

## Limpieza

Este laboratorio prepara la aplicación para el laboratorio 15, donde será publicada mediante Ingress. Por ello, **no elimine** los siguientes recursos al finalizar:

- Namespace `curso-k8s`.
- Deployment `webapp`.
- Service `webapp-svc`.
- PVC `webapp-content-pvc`.
- ConfigMap `webapp-config`.
- Manifiestos almacenados en `~/k8s-intro/manifests`.
- Evidencias almacenadas en `~/k8s-intro/analisis`.

Los Pods de comprobación BusyBox se eliminan automáticamente gracias a `--rm`. Confírmelo con:

```bash
kubectl -n curso-k8s get pods
```

Si necesita repetir exclusivamente la práctica desde cero y cuenta con autorización para retirar la configuración, elimine el ConfigMap y revierta el montaje del Deployment mediante un manifiesto conocido del laboratorio anterior. No elimine el PVC ni el Service, porque son dependencias de los siguientes ejercicios.

## Resumen

En esta práctica se creó el ConfigMap `webapp-config` para separar valores no sensibles de la imagen de NGINX. El archivo `config.json` se montó de forma precisa mediante `items` y `subPath`, evitando reemplazar el directorio persistente respaldado por `webapp-content-pvc`.

También se comprobó que el Service `webapp-svc` publica el archivo de configuración dentro del clúster y que una actualización de ConfigMap requiere reiniciar el Deployment cuando se utiliza `subPath`. La aplicación queda preparada para su publicación mediante Ingress en el laboratorio 15.

**Recursos oficiales**

- ConfigMaps de Kubernetes: https://kubernetes.io/docs/concepts/configuration/configmap/
- Volúmenes ConfigMap y `subPath`: https://kubernetes.io/docs/concepts/storage/volumes/
- Reinicio de un Deployment con kubectl: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/
- Buenas prácticas para Secrets de Kubernetes: https://kubernetes.io/docs/concepts/configuration/secret/
