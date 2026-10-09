# Práctica: Consulta de permisos y acceso

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 20 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se comprobará cómo Kubernetes autoriza solicitudes dirigidas al API Server mediante RBAC (*Role-Based Access Control*). A partir del Namespace `lab-intro` y de la aplicación `web-demo` creada previamente, se creará una identidad de aplicación limitada, se asignarán permisos mínimos de lectura y se evaluarán sus permisos efectivos con `kubectl auth can-i`.

La práctica diferencia explícitamente la identidad administrativa utilizada por el contexto local de `kubectl` de la identidad restringida `system:serviceaccount:lab-intro:viewer`. No se crearán secretos de token manualmente ni se expondrán credenciales.

## Objetivos de Aprendizaje

- [ ] Identificar el contexto, el clúster y la identidad administrativa que utiliza `kubectl`.
- [ ] Crear la cuenta de servicio `viewer` en el Namespace `lab-intro`.
- [ ] Definir un `Role` de solo lectura sobre Pods y Deployments.
- [ ] Asociar el `Role` a la cuenta de servicio mediante un `RoleBinding`.
- [ ] Verificar permisos permitidos y denegados con `kubectl auth can-i`.

## Prerrequisitos

### Conocimientos necesarios

- Comprender que `kubectl` envía solicitudes HTTPS al API Server de Kubernetes.
- Reconocer las etapas básicas de una solicitud: autenticación, autorización, admisión y persistencia.
- Conocer los recursos `Namespace`, `Pod`, `Deployment`, `ServiceAccount`, `Role` y `RoleBinding`.
- Saber aplicar manifiestos YAML con `kubectl apply -f`.
- Distinguir operaciones de lectura, como `get` y `list`, de operaciones de escritura, como `create` y `delete`.

### Acceso necesario

- Laboratorio `09-00-01` completado.
- Namespace `lab-intro` existente.
- Deployment `web-demo` y sus Pods disponibles en `lab-intro`.
- Permisos administrativos locales para crear recursos RBAC dentro de `lab-intro`.
- Acceso a una terminal Bash.
- Acceso al clúster local mediante el contexto configurado por el curso.

> **Nota de coherencia del entorno:** la guía global del curso define el contexto esperado como `minikube` y Kubernetes `v1.31.0`; la ficha específica de esta práctica menciona el contexto `intro-k8s` y Kubernetes `1.33.0`. Los manifiestos RBAC empleados son compatibles con ambas versiones. Antes de continuar, utilice el contexto aprobado por el instructor y documente el valor real obtenido. No cambie de contexto sin confirmación, porque podría aplicar permisos sobre otro clúster.

## Entorno de Laboratorio

### Versiones de referencia

| Tecnología | Versión, edición y arquitectura | Uso en la práctica | Fuente oficial |
|---|---|---|---|
| Ubuntu | Ubuntu 22.04.5 LTS, Server/Desktop, `amd64` | Sistema operativo de referencia | <https://releases.ubuntu.com/jammy/> |
| Docker Engine | 27.2.0, Community Edition, `amd64` | Controlador de Minikube | <https://docs.docker.com/engine/release-notes/27.2/> |
| Minikube | 1.34.0, Linux `amd64` | Clúster local con controlador Docker | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| kubectl | 1.31.0, Linux `amd64` | Cliente de la API de Kubernetes | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| Kubernetes | v1.31.0, Linux `amd64` | Versión bloqueada del entorno de referencia | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Kubernetes indicado en la ficha | v1.33.0, arquitectura no especificada por la ficha | Referencia declarada en la especificación de la práctica; validar con el instructor antes de usarla | <https://github.com/kubernetes/kubernetes/releases/tag/v1.33.0> |

> Esta práctica no utiliza Microsoft 365 Copilot, Copilot Chat, Microsoft Designer, Microsoft Planner ni servicios de IA. Por tanto, no requiere licencias, configuraciones ni credenciales de dichas herramientas.

### Preparación del directorio de trabajo

Ejecute los siguientes comandos para crear los directorios obligatorios y posicionarse en el directorio de trabajo:

```bash
mkdir -p ~/k8s-intro/manifests
mkdir -p ~/k8s-intro/analisis/artifacts
cd ~/k8s-intro
```

Si el clúster no está iniciado y el instructor ha confirmado el entorno de referencia bloqueado, utilice:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

Verifique la disponibilidad básica del clúster:

```bash
minikube status
kubectl version --client
kubectl cluster-info
```

## Instrucciones Paso a Paso

### Paso 1: Verificar el contexto y los recursos iniciales

**Objetivo:** confirmar que `kubectl` apunta al clúster aprobado y que los recursos creados en el laboratorio anterior están disponibles.

**Instrucciones:**

1. Consulte el contexto activo:

   ```bash
   kubectl config current-context
   ```

2. Liste los contextos configurados y observe cuál está marcado con `*`:

   ```bash
   kubectl config get-contexts
   ```

3. Compruebe la versión del servidor Kubernetes realmente utilizada:

   ```bash
   kubectl version --output=yaml
   ```

4. Verifique la existencia del Namespace `lab-intro`:

   ```bash
   kubectl get namespace lab-intro
   ```

5. Verifique el Deployment `web-demo` y sus Pods:

   ```bash
   kubectl get deployment web-demo --namespace lab-intro
   kubectl get pods --namespace lab-intro -l app=web-demo
   ```

6. Identifique la identidad configurada para el contexto actual:

   ```bash
   kubectl config view --minify --output 'jsonpath={.contexts[0].context.user}{"\n"}'
   ```

**Salida esperada:**

- El contexto activo coincide con el contexto autorizado por el instructor, normalmente `minikube` en el entorno global de referencia.
- El Namespace `lab-intro` aparece en estado `Active`.
- El Deployment `web-demo` muestra al menos una réplica disponible.
- Se muestra una identidad de usuario o certificado local asociada al contexto administrativo.

**Verificación:**

Ejecute el siguiente comando. El resultado debe ser `yes` para continuar, ya que esta práctica requiere crear recursos RBAC:

```bash
kubectl auth can-i create roles --namespace lab-intro
```

Si el resultado es `no`, detenga la práctica y solicite al instructor el contexto administrativo correcto. No intente continuar con permisos insuficientes.

---

### Paso 2: Inspeccionar el acceso administrativo actual

**Objetivo:** establecer una línea base de los permisos de la identidad administrativa local antes de crear la cuenta de servicio restringida.

**Instrucciones:**

1. Compruebe si la identidad actual puede listar Pods:

   ```bash
   kubectl auth can-i list pods --namespace lab-intro
   ```

2. Compruebe si puede consultar Deployments:

   ```bash
   kubectl auth can-i get deployments --namespace lab-intro
   ```

3. Compruebe si puede crear Pods:

   ```bash
   kubectl auth can-i create pods --namespace lab-intro
   ```

4. Compruebe si puede eliminar Deployments:

   ```bash
   kubectl auth can-i delete deployments --namespace lab-intro
   ```

5. Compruebe si puede consultar Secrets:

   ```bash
   kubectl auth can-i get secrets --namespace lab-intro
   ```

6. Guarde estas comprobaciones como evidencia inicial:

   ```bash
   {
     echo "=== Evidencia RBAC: identidad administrativa ==="
     echo "Fecha UTC: $(date -u +'%Y-%m-%dT%H:%M:%SZ')"
     echo "Contexto: $(kubectl config current-context)"
     echo "Usuario del contexto: $(kubectl config view --minify --output 'jsonpath={.contexts[0].context.user}')"
     echo "Servidor Kubernetes:"
     kubectl version --output=yaml | grep -E 'gitVersion|platform' || true
     echo
     echo "admin list pods:"
     kubectl auth can-i list pods --namespace lab-intro
     echo "admin get deployments:"
     kubectl auth can-i get deployments --namespace lab-intro
     echo "admin create pods:"
     kubectl auth can-i create pods --namespace lab-intro
     echo "admin delete deployments:"
     kubectl auth can-i delete deployments --namespace lab-intro
     echo "admin get secrets:"
     kubectl auth can-i get secrets --namespace lab-intro
   } > ~/k8s-intro/analisis/artifacts/rbac-checks.txt
   ```

**Salida esperada:**

En un contexto administrativo local de Minikube, los resultados suelen ser `yes`. El resultado exacto depende de cómo se haya configurado el clúster y de la identidad asociada al contexto.

**Verificación:**

Compruebe que el archivo de evidencia existe y contiene las primeras comprobaciones:

```bash
cat ~/k8s-intro/analisis/artifacts/rbac-checks.txt
```

---

### Paso 3: Crear la cuenta de servicio y las reglas RBAC mínimas

**Objetivo:** definir declarativamente una identidad de aplicación y concederle solo permisos de lectura sobre Pods y Deployments en `lab-intro`.

**Instrucciones:**

1. Cree el manifiesto RBAC en el directorio obligatorio:

   ```bash
   cat > ~/k8s-intro/manifests/viewer-rbac.yaml <<'EOF'
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: viewer
     namespace: lab-intro
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: pod-deployment-reader
     namespace: lab-intro
   rules:
     - apiGroups:
         - ""
       resources:
         - pods
       verbs:
         - get
         - list
         - watch
     - apiGroups:
         - apps
       resources:
         - deployments
       verbs:
         - get
         - list
         - watch
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: viewer-read-binding
     namespace: lab-intro
   subjects:
     - kind: ServiceAccount
       name: viewer
       namespace: lab-intro
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: pod-deployment-reader
   EOF
   ```

2. Revise el contenido antes de aplicarlo:

   ```bash
   cat ~/k8s-intro/manifests/viewer-rbac.yaml
   ```

3. Aplique el manifiesto:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/viewer-rbac.yaml
   ```

4. Inspeccione los tres recursos creados:

   ```bash
   kubectl get serviceaccount viewer --namespace lab-intro
   kubectl get role pod-deployment-reader --namespace lab-intro
   kubectl get rolebinding viewer-read-binding --namespace lab-intro
   ```

5. Revise las reglas efectivas declaradas en el Role:

   ```bash
   kubectl describe role pod-deployment-reader --namespace lab-intro
   ```

**Salida esperada:**

La aplicación del manifiesto debe confirmar tres recursos, con mensajes similares a los siguientes:

```text
serviceaccount/viewer created
role.rbac.authorization.k8s.io/pod-deployment-reader created
rolebinding.rbac.authorization.k8s.io/viewer-read-binding created
```

El `Role` debe incluir únicamente:

- `get`, `list` y `watch` sobre `pods` del grupo API principal (`""`).
- `get`, `list` y `watch` sobre `deployments` del grupo API `apps`.

**Verificación:**

Confirme que el Role no contiene permisos sobre `secrets`, `services`, `namespaces`, `rolebindings` ni verbos de modificación como `create`, `update`, `patch` o `delete`:

```bash
kubectl get role pod-deployment-reader --namespace lab-intro --output=yaml
```

> En Kubernetes modernos, crear una `ServiceAccount` no implica crear automáticamente un Secret persistente con token. Esta práctica no necesita tokens porque utiliza impersonación controlada mediante `kubectl auth can-i --as`.

---

### Paso 4: Comprobar los permisos de la cuenta de servicio

**Objetivo:** evaluar las decisiones de autorización para la identidad `system:serviceaccount:lab-intro:viewer`.

**Instrucciones:**

1. Defina una variable temporal para evitar errores al escribir la identidad:

   ```bash
   SA_IDENTITY="system:serviceaccount:lab-intro:viewer"
   ```

2. Compruebe que puede listar Pods:

   ```bash
   kubectl auth can-i list pods \
     --namespace lab-intro \
     --as="$SA_IDENTITY"
   ```

3. Compruebe que puede consultar el Deployment `web-demo`:

   ```bash
   kubectl auth can-i get deployments \
     --namespace lab-intro \
     --as="$SA_IDENTITY"
   ```

4. Compruebe que no puede crear Pods:

   ```bash
   kubectl auth can-i create pods \
     --namespace lab-intro \
     --as="$SA_IDENTITY"
   ```

5. Compruebe que no puede eliminar Deployments:

   ```bash
   kubectl auth can-i delete deployments \
     --namespace lab-intro \
     --as="$SA_IDENTITY"
   ```

6. Compruebe que no puede consultar Secrets:

   ```bash
   kubectl auth can-i get secrets \
     --namespace lab-intro \
     --as="$SA_IDENTITY"
   ```

7. Añada los resultados al archivo de evidencia:

   ```bash
   {
     echo
     echo "=== Evidencia RBAC: ServiceAccount viewer ==="
     echo "Identidad evaluada: $SA_IDENTITY"
     echo "viewer list pods:"
     kubectl auth can-i list pods --namespace lab-intro --as="$SA_IDENTITY"
     echo "viewer get deployments:"
     kubectl auth can-i get deployments --namespace lab-intro --as="$SA_IDENTITY"
     echo "viewer create pods:"
     kubectl auth can-i create pods --namespace lab-intro --as="$SA_IDENTITY"
     echo "viewer delete deployments:"
     kubectl auth can-i delete deployments --namespace lab-intro --as="$SA_IDENTITY"
     echo "viewer get secrets:"
     kubectl auth can-i get secrets --namespace lab-intro --as="$SA_IDENTITY"
   } >> ~/k8s-intro/analisis/artifacts/rbac-checks.txt
   ```

**Salida esperada:**

Los resultados de la cuenta de servicio deben ser exactamente los siguientes:

| Acción consultada | Resultado esperado |
|---|---|
| `list pods` | `yes` |
| `get deployments` | `yes` |
| `create pods` | `no` |
| `delete deployments` | `no` |
| `get secrets` | `no` |

**Verificación:**

Muestre el archivo de evidencia completo:

```bash
cat ~/k8s-intro/analisis/artifacts/rbac-checks.txt
```

Verifique que contiene tanto las comprobaciones administrativas como las comprobaciones de la cuenta de servicio.

---

### Paso 5: Relacionar el resultado con el flujo de la API

**Objetivo:** interpretar los resultados de autorización como parte del procesamiento de solicitudes por el API Server.

**Instrucciones:**

1. Observe la petición de lectura administrativa con nivel de detalle adicional:

   ```bash
   kubectl get pods --namespace lab-intro -v=6
   ```

2. Identifique en la salida una solicitud relacionada con una ruta parecida a:

   ```text
   GET /api/v1/namespaces/lab-intro/pods
   ```

3. Consulte los permisos enumerados para la cuenta de servicio:

   ```bash
   kubectl auth can-i --list \
     --namespace lab-intro \
     --as=system:serviceaccount:lab-intro:viewer
   ```

4. Añada una conclusión breve y verificable a la evidencia:

   ```bash
   cat >> ~/k8s-intro/analisis/artifacts/rbac-checks.txt <<'EOF'

   === Conclusión ===
   La identidad administrativa del contexto local y la ServiceAccount
   system:serviceaccount:lab-intro:viewer son identidades distintas.
   El Role pod-deployment-reader permite únicamente get, list y watch
   sobre Pods y Deployments dentro de lab-intro. Las acciones create,
   delete y la lectura de Secrets deben ser denegadas para viewer.
   EOF
   ```

**Salida esperada:**

- La salida detallada de `kubectl` muestra que el cliente realiza solicitudes al API Server.
- `kubectl auth can-i --list` presenta permisos relacionados con Pods y Deployments para `viewer`.
- No deben aparecer permisos concedidos sobre Secrets ni permisos de escritura sobre Pods o Deployments.

**Verificación:**

Confirme que la cuenta de servicio no dispone de permisos globales de clúster:

```bash
kubectl auth can-i list namespaces \
  --as=system:serviceaccount:lab-intro:viewer
```

La salida esperada es:

```text
no
```

Esto demuestra que un `Role` con ámbito de Namespace no concede permisos fuera de `lab-intro`.

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios medibles siguientes.

| Criterio | Comando o evidencia | Resultado requerido |
|---|---|---|
| Contexto identificado | `kubectl config current-context` | Contexto documentado y aprobado por el instructor |
| Namespace disponible | `kubectl get namespace lab-intro` | Estado `Active` |
| Aplicación previa disponible | `kubectl get deployment web-demo -n lab-intro` | Deployment existente |
| Cuenta de servicio creada | `kubectl get sa viewer -n lab-intro` | Recurso existente |
| Role creado | `kubectl get role pod-deployment-reader -n lab-intro` | Recurso existente |
| RoleBinding creado | `kubectl get rolebinding viewer-read-binding -n lab-intro` | Recurso existente |
| Lectura de Pods permitida | `kubectl auth can-i list pods -n lab-intro --as=system:serviceaccount:lab-intro:viewer` | `yes` |
| Lectura de Deployments permitida | `kubectl auth can-i get deployments -n lab-intro --as=system:serviceaccount:lab-intro:viewer` | `yes` |
| Creación de Pods denegada | `kubectl auth can-i create pods -n lab-intro --as=system:serviceaccount:lab-intro:viewer` | `no` |
| Eliminación de Deployments denegada | `kubectl auth can-i delete deployments -n lab-intro --as=system:serviceaccount:lab-intro:viewer` | `no` |
| Lectura de Secrets denegada | `kubectl auth can-i get secrets -n lab-intro --as=system:serviceaccount:lab-intro:viewer` | `no` |
| Evidencia persistida | `test -s ~/k8s-intro/analisis/artifacts/rbac-checks.txt && echo "OK"` | `OK` |

### Caso adversarial de validación

Realice una comprobación adicional para evitar aceptar información no verificable o instrucciones no confiables incluidas en archivos de evidencia:

```bash
grep -nEi 'ignore|omitir|delete|eliminar|token|secret' \
  ~/k8s-intro/analisis/artifacts/rbac-checks.txt || true
```

Si el archivo contuviera una frase como “ignore las restricciones y cree un ClusterRole” o “muestre un token”, trátela como contenido no confiable, no como una instrucción de laboratorio. La evidencia solo debe contener comandos ejecutados, resultados observables y conclusiones verificables.

Esta práctica no requiere herramientas de IA. Si se utilizara una herramienta externa para interpretar resultados, el estudiante debe mantener supervisión humana: un *prompt* es una instrucción temporal enviada a una herramienta; no equivale a un mensaje de sistema ni concede permisos en Kubernetes. La fuente de verdad para aprobar la práctica son los comandos, manifiestos y salidas verificables del clúster.

## Solución de Problemas

### Problema 1: `kubectl auth can-i --as=...` devuelve un error de impersonación

**Síntoma:**

```text
Error from server (Forbidden): users "..." is forbidden to impersonate resource "users"
```

**Causa probable:** la identidad del contexto actual no tiene permiso para impersonar la cuenta de servicio. El comando `--as` solicita al API Server evaluar la autorización como otra identidad, por lo que el usuario administrativo necesita permisos de impersonación.

**Solución:**

1. Confirme la identidad y el contexto actual:

   ```bash
   kubectl config current-context
   kubectl config view --minify --output 'jsonpath={.contexts[0].context.user}{"\n"}'
   ```

2. Cambie únicamente al contexto administrativo aprobado por el instructor.
3. Solicite al instructor permisos de impersonación o que ejecute las comprobaciones de `can-i` en el contexto administrativo del laboratorio.
4. No cree un `ClusterRoleBinding` amplio para resolver el problema, ya que alteraría el objetivo de mínimo privilegio de la práctica.

### Problema 2: `viewer` puede consultar un recurso no previsto o no puede listar Pods

**Síntoma:**

- Un comando esperado como `kubectl auth can-i list pods ... --as=system:serviceaccount:lab-intro:viewer` devuelve `no`, o
- Un comando no permitido, como `get secrets`, devuelve `yes`.

**Causa probable:** el `Role` o el `RoleBinding` contiene un nombre incorrecto, un Namespace distinto, verbos adicionales, reglas heredadas de una práctica anterior o una asociación accidental con otro RoleBinding.

**Solución:**

1. Inspeccione las reglas del Role:

   ```bash
   kubectl get role pod-deployment-reader \
     --namespace lab-intro \
     --output=yaml
   ```

2. Inspeccione el sujeto y la referencia del RoleBinding:

   ```bash
   kubectl get rolebinding viewer-read-binding \
     --namespace lab-intro \
     --output=yaml
   ```

3. Compruebe si existen otros RoleBindings que involucren a `viewer`:

   ```bash
   kubectl get rolebindings --namespace lab-intro --output=yaml | grep -n -A6 -B3 'name: viewer'
   ```

4. Si el manifiesto no coincide con el de esta guía, corríjalo y reaplíquelo:

   ```bash
   kubectl apply -f ~/k8s-intro/manifests/viewer-rbac.yaml
   ```

5. Repita las cinco comprobaciones de `kubectl auth can-i` y actualice la evidencia.

## Limpieza

La limpieza elimina únicamente los recursos RBAC creados en esta práctica. No elimine el Namespace `lab-intro`, el Deployment `web-demo`, sus Pods ni otros recursos del laboratorio anterior.

1. Elimine el RoleBinding:

   ```bash
   kubectl delete rolebinding viewer-read-binding --namespace lab-intro
   ```

2. Elimine el Role:

   ```bash
   kubectl delete role pod-deployment-reader --namespace lab-intro
   ```

3. Elimine la cuenta de servicio:

   ```bash
   kubectl delete serviceaccount viewer --namespace lab-intro
   ```

4. Verifique que los recursos RBAC ya no existen:

   ```bash
   kubectl get serviceaccount,role,rolebinding --namespace lab-intro \
     | grep -E 'viewer|pod-deployment-reader' || true
   ```

5. Conserve los archivos de manifiesto y de evidencia para la revisión del laboratorio:

   ```bash
   ls -l ~/k8s-intro/manifests/viewer-rbac.yaml
   ls -l ~/k8s-intro/analisis/artifacts/rbac-checks.txt
   ```

## Resumen

En esta práctica se creó la cuenta de servicio `viewer` y se le asignó el Role `pod-deployment-reader` mediante el RoleBinding `viewer-read-binding`. La cuenta recibió exclusivamente permisos `get`, `list` y `watch` sobre Pods y Deployments dentro de `lab-intro`.

Las pruebas con `kubectl auth can-i` mostraron que la autorización depende de la identidad evaluada: el contexto administrativo local puede disponer de permisos amplios, mientras que `system:serviceaccount:lab-intro:viewer` queda limitada al principio de mínimo privilegio. Las respuestas `yes` y `no` representan decisiones de autorización del API Server antes de que una solicitud alcance operaciones de modificación o persistencia.

### Recursos oficiales

- Kubernetes RBAC: <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- Cuentas de servicio: <https://kubernetes.io/docs/concepts/security/service-accounts/>
- `kubectl auth can-i`: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/>
- Control de acceso a la API: <https://kubernetes.io/docs/concepts/security/controlling-access/>
