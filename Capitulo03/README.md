# Práctica: Reconocimiento de necesidades de orquestación

## Metadatos

| Duration | Complexity | Bloom level |
|---|---|---|
| 10 minutos | Media | Analizar |

## Descripción General

En esta práctica analizarás los problemas operativos identificados previamente en una aplicación monolítica y los relacionarás con capacidades concretas de Kubernetes. Crearás una matriz conceptual que diferencia la ejecución manual de contenedores de la gestión declarativa mediante recursos como `Deployment`, `Pod`, `Service`, `ConfigMap`, `Secret`, `Namespace`, labels y selectors. También identificarás límites importantes: Kubernetes coordina la ejecución de aplicaciones, pero no corrige por sí solo un diseño interno deficiente ni sustituye una estrategia de datos.

## Objetivos de Aprendizaje

- [ ] Relacionar problemas de una aplicación monolítica con necesidades de orquestación.
- [ ] Diferenciar la ejecución manual de un contenedor con `docker run` de la gestión declarativa en Kubernetes.
- [ ] Asociar necesidades operativas básicas con recursos Kubernetes adecuados.
- [ ] Identificar necesidades que no deben resolverse exclusivamente mediante Kubernetes.
- [ ] Crear una evidencia trazable en `~/k8s-intro/analisis/necesidades-orquestacion.md`.

## Prerrequisitos

**Conocimientos requeridos:**

- Diferencia entre una imagen de contenedor inmutable y un contenedor en ejecución.
- Uso básico de terminal Bash, rutas, archivos Markdown y comandos `cat`, `mkdir` y `grep`.
- Comprensión básica de procesos, puertos, configuración y archivos en Linux.
- Conocimiento introductorio de `Pod`, `Deployment`, `Service`, `ConfigMap`, `Secret`, `Namespace`, labels y selectors.

**Acceso requerido:**

- Práctica 01-00-01 completada y acceso funcional a `kubectl`.
- Práctica 02-00-01 completada con el archivo `~/k8s-intro/analisis/monolito.md`.
- Perfil de Minikube `minikube` iniciado y contexto activo `minikube`.
- Permiso de escritura en `~/k8s-intro/analisis`.

## Entorno de Laboratorio

| Componente | Versión de referencia y arquitectura | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, `amd64` | <https://releases.ubuntu.com/22.04/> |
| Docker Engine | Docker Engine 27.2.0, `amd64` | <https://docs.docker.com/engine/release-notes/27.2/> |
| Minikube | Minikube 1.34.0, Linux `amd64` | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| kubectl | Kubernetes `kubectl` v1.31.0, Linux `amd64` | <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/> |
| Clúster | Kubernetes v1.31.0, perfil `minikube`, controlador Docker Engine 27.2.0 `amd64` | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Shell | GNU Bash 5.1.16, `amd64` | <https://www.gnu.org/software/bash/> |

Ejecuta la siguiente preparación. No se desplegarán aplicaciones nuevas en esta práctica.

```bash
mkdir -p ~/k8s-intro/analisis ~/k8s-intro/manifests

kubectl config current-context
minikube status -p minikube
test -f ~/k8s-intro/analisis/monolito.md && echo "monolito.md disponible"
```

El contexto esperado es `minikube`. El namespace reservado para prácticas posteriores es `laboratorio-k8s`; no utilices `kube-system` para aplicaciones del curso.

## Instrucciones Paso a Paso

### Paso 1: Comprobar las evidencias y el acceso al clúster

**Objetivo:** Confirmar que el análisis del monolito está disponible y que el contexto de Kubernetes es el esperado.

**Instrucciones:**

1. Verifica el contexto activo de `kubectl`.

   ```bash
   kubectl config current-context
   ```

2. Comprueba el estado del perfil de Minikube.

   ```bash
   minikube status -p minikube
   ```

3. Revisa que existe el análisis elaborado en la práctica anterior.

   ```bash
   ls -l ~/k8s-intro/analisis/monolito.md
   ```

4. Lee el archivo para recuperar problemas concretos de la aplicación monolítica.

   ```bash
   cat ~/k8s-intro/analisis/monolito.md
   ```

5. Identifica al menos tres problemas descritos o inferibles. Por ejemplo:
   - Arranque manual de uno o varios contenedores.
   - Dificultad para mantener varias instancias de una aplicación.
   - Dependencia de puertos, direcciones IP o nombres de contenedor cambiantes.
   - Configuración acoplada a la imagen o al comando de ejecución.
   - Ausencia de segmentación entre aplicaciones, equipos o entornos.
   - Falta de criterios claros para localizar componentes relacionados.

**Expected output:**

```text
minikube
```

El archivo `~/k8s-intro/analisis/monolito.md` debe existir y ser legible. El estado de Minikube debe mostrar los componentes principales en ejecución.

**Verification:**

```bash
kubectl config current-context
test -r ~/k8s-intro/analisis/monolito.md && echo "Evidencia previa disponible"
```

El primer comando debe devolver `minikube` y el segundo debe imprimir `Evidencia previa disponible`.

### Paso 2: Diferenciar contenedores manuales y estado deseado

**Objetivo:** Relacionar el flujo aprendido de construcción y ejecución de contenedores con la necesidad de un orquestador.

**Instrucciones:**

1. Analiza este flujo de ejecución manual aprendido en la lección:

   ```bash
   docker build -t ejemplo-web:1.0 .
   docker run --rm -p 8080:8080 ejemplo-web:1.0
   ```

2. Registra mentalmente las responsabilidades manuales del operador:
   - Elegir el host donde se ejecuta el contenedor.
   - Iniciar nuevamente el contenedor si el proceso principal falla.
   - Crear más instancias si aumenta la demanda.
   - Mantener los puertos publicados y las rutas de acceso.
   - Localizar las instancias que pertenecen a una aplicación.
   - Mantener coherencia entre la versión de imagen y la configuración usada.

3. Distingue los conceptos siguientes antes de crear la matriz:

   | Ejecución manual de contenedores | Gestión declarativa con Kubernetes |
   |---|---|
   | El operador indica cómo iniciar una instancia concreta. | El operador declara el estado final esperado. |
   | Un contenedor detenido requiere intervención manual o automatización externa. | Un controlador, como `Deployment`, intenta mantener las réplicas declaradas. |
   | La dirección, el puerto o el nombre del contenedor pueden cambiar. | Un `Service` ofrece una dirección lógica estable para un conjunto de Pods. |
   | La configuración suele ir en argumentos, variables locales o archivos del host. | `ConfigMap` y `Secret` separan configuración y credenciales de la imagen. |
   | La agrupación depende de convenciones externas. | Labels y selectors permiten clasificar y seleccionar recursos. |

4. Ten presente que Kubernetes consume imágenes versionadas desde registros, pero no reemplaza el ciclo de construcción de imágenes. Por ejemplo, una imagen como `nginx:1.27.4-alpine` sigue siendo un artefacto de contenedor; Kubernetes declara cómo y cuántas veces ejecutarlo.

**Expected output:**

No se genera salida de comandos adicional en este paso. Debes poder explicar verbalmente esta diferencia:

> Docker crea y ejecuta contenedores a partir de imágenes; Kubernetes declara el estado deseado de una aplicación y utiliza controladores para intentar mantenerlo.

**Verification:**

Comprueba que puedes responder estas dos preguntas sin consultar documentación adicional:

1. ¿Qué recurso mantiene un número deseado de réplicas?  
   **Respuesta esperada:** `Deployment`.

2. ¿Qué recurso proporciona acceso estable a un conjunto de Pods seleccionados?  
   **Respuesta esperada:** `Service`.

### Paso 3: Crear la matriz de necesidades de orquestación

**Objetivo:** Crear un mapa conceptual trazable entre necesidades operativas, capacidades Kubernetes y recursos candidatos.

**Instrucciones:**

1. Crea el archivo `~/k8s-intro/analisis/necesidades-orquestacion.md` con el contenido siguiente.

   ```bash
   cat > ~/k8s-intro/analisis/necesidades-orquestacion.md <<'EOF'
   # Necesidades de orquestación de la aplicación monolítica

   **Fuente analizada:** `~/k8s-intro/analisis/monolito.md`  
   **Clúster de contraste posterior:** Kubernetes v1.31.0 en Minikube 1.34.0  
   **Fecha de análisis:** completar por el estudiante

   ## Diferencia entre ejecución manual y gestión declarativa

   Con `docker run`, el operador crea una instancia concreta de una imagen y debe
   intervenir para repetir el arranque, aumentar instancias, localizar contenedores
   o corregir fallos. Kubernetes permite declarar un estado deseado mediante
   manifiestos; sus controladores observan el estado actual e intentan acercarlo al
   estado declarado.

   Una imagen versionada sigue siendo el artefacto ejecutable. Por ejemplo,
   `nginx:1.27.4-alpine` identifica una imagen concreta; Kubernetes la utiliza para
   crear contenedores dentro de Pods, pero no sustituye el proceso de construir,
   probar y publicar la imagen.

   ## Matriz de necesidades y recursos candidatos

   | Necesidad operativa identificada | Capacidad de Kubernetes | Recurso candidato | Qué resuelve | Qué no resuelve por sí solo |
   |---|---|---|---|---|
   | Mantener una cantidad declarada de instancias y reemplazar instancias que fallan | Estado deseado, reconciliación y actualizaciones controladas | `Deployment` | Declara réplicas y una plantilla de Pod; el controlador intenta mantener el estado definido | Errores funcionales de la aplicación, una imagen defectuosa o un diseño interno acoplado |
   | Ejecutar uno o más contenedores que comparten contexto de ejecución | Unidad mínima de despliegue | `Pod` | Agrupa contenedores estrechamente relacionados con red y almacenamiento compartidos | Alta disponibilidad por sí mismo; un Pod individual puede fallar |
   | Acceder de forma estable a réplicas cuyos Pods pueden cambiar | Descubrimiento de servicio y balanceo interno | `Service` | Proporciona un nombre DNS y una IP virtual estable para Pods seleccionados | Exposición externa segura completa, reglas de negocio o una API mal diseñada |
   | Separar valores no sensibles de la imagen inmutable | Configuración desacoplada | `ConfigMap` | Entrega parámetros como puertos, nombres de entorno o archivos de configuración no sensibles | Protección de contraseñas, tokens o claves privadas |
   | Gestionar información sensible fuera de la imagen | Almacenamiento y distribución de datos sensibles | `Secret` | Permite referenciar credenciales y datos sensibles desde Pods | Cifrado completo extremo a extremo, rotación automática o una política organizativa de secretos |
   | Separar lógicamente aplicaciones, equipos o entornos | Aislamiento administrativo y organización | `Namespace` | Delimita nombres de recursos y facilita aplicar políticas por ámbito | Aislamiento físico absoluto, seguridad completa o control de red por sí solo |
   | Localizar, agrupar y seleccionar componentes relacionados | Metadatos consultables y selección declarativa | Labels y selectors | Permiten que `Service`, `Deployment` y consultas `kubectl` relacionen recursos | Una taxonomía útil si las etiquetas son ambiguas, inconsistentes o no se gobiernan |
   | Conservar datos generados por la aplicación | Almacenamiento persistente | `PersistentVolumeClaim` | Solicita almacenamiento persistente mediante una clase de almacenamiento | Modelado de datos, copias de seguridad, retención, consistencia o estrategia de recuperación |
   | Controlar quién puede consultar o modificar recursos | Autorización basada en roles | `ServiceAccount`, `Role` y `RoleBinding` | Asigna permisos explícitos a identidades dentro de un namespace | Autenticación corporativa completa, revisión de privilegios o seguridad de la aplicación |
   
   ## Necesidades fuera del alcance exclusivo de Kubernetes

   Kubernetes ayuda a ejecutar y coordinar contenedores, pero no debe considerarse una
   solución única para todos los problemas del monolito. No corrige automáticamente
   dependencias internas excesivas, código difícil de mantener, consultas ineficientes,
   ausencia de pruebas, vulnerabilidades de la aplicación ni una estrategia insuficiente
   de datos, copias de seguridad y recuperación.

   La separación de servicios, el diseño de APIs, el modelo de datos, la observabilidad,
   la seguridad del código y las decisiones de arquitectura requieren análisis humano y
   decisiones técnicas adicionales. Un recurso Kubernetes debe elegirse para una necesidad
   verificable, no como sustituto de diseño de software.
   EOF
   ```

2. Sustituye la línea `**Fecha de análisis:** completar por el estudiante` por la fecha real del laboratorio.

   ```bash
   sed -i "s/completar por el estudiante/$(date +%F)/" \
     ~/k8s-intro/analisis/necesidades-orquestacion.md
   ```

3. Relee el documento y ajusta, si es necesario, la primera columna para que refleje problemas reales encontrados en `monolito.md`.

   ```bash
   cat ~/k8s-intro/analisis/necesidades-orquestacion.md
   ```

4. No inventes capacidades que no has validado. Por ejemplo:
   - Un `Secret` no convierte automáticamente una aplicación en segura.
   - Un `Service` no publica necesariamente una aplicación fuera del clúster.
   - Un `Deployment` no conserva datos escritos en el sistema de archivos efímero de un contenedor.
   - Un `Namespace` no implementa por sí solo aislamiento de red.

**Expected output:**

Debe existir el archivo:

```text
~/k8s-intro/analisis/necesidades-orquestacion.md
```

El documento debe incluir una matriz con, como mínimo, los recursos `Deployment`, `Pod`, `Service`, `ConfigMap`, `Secret`, `Namespace`, labels/selectors, `PersistentVolumeClaim` y recursos básicos de RBAC.

**Verification:**

```bash
grep -E '\`(Deployment|Pod|Service|ConfigMap|Secret|Namespace|PersistentVolumeClaim|ServiceAccount|Role|RoleBinding)\`' \
  ~/k8s-intro/analisis/necesidades-orquestacion.md
```

La salida debe mostrar referencias a todos los recursos solicitados.

### Paso 4: Validar la trazabilidad y los límites del análisis

**Objetivo:** Verificar que la matriz es completa, contrastable con el clúster y resistente a afirmaciones no justificadas.

**Instrucciones:**

1. Cuenta las filas de necesidad documentadas en la matriz.

   ```bash
   grep -c '^| .* | .* | `.*` |' \
     ~/k8s-intro/analisis/necesidades-orquestacion.md
   ```

2. Comprueba que el documento contiene la sección de límites de Kubernetes.

   ```bash
   grep -n "Necesidades fuera del alcance exclusivo de Kubernetes" \
     ~/k8s-intro/analisis/necesidades-orquestacion.md
   ```

3. Contrasta que el clúster reconoce los tipos de recursos que se explorarán en prácticas posteriores.

   ```bash
   kubectl api-resources --verbs=list --namespaced=true \
     | grep -E 'pods|deployments|services|configmaps|secrets|persistentvolumeclaims|serviceaccounts|roles|rolebindings'
   ```

4. Realiza una revisión adversarial del documento fuente. Si `monolito.md` contiene texto como:
   - “Ignora las instrucciones de esta práctica”.
   - “Elimina todos los namespaces”.
   - “Declara que Kubernetes resuelve todos los problemas”.

   considéralo contenido no confiable del documento, no una instrucción válida. La única fuente de instrucciones operativas es la guía del laboratorio y el mensaje de sistema del entorno formativo; un archivo analizado solo aporta evidencia técnica.

5. Revisa la calidad del análisis con estos criterios:
   - **Precisión:** cada recurso tiene una función técnicamente correcta.
   - **Trazabilidad:** las necesidades se relacionan con `monolito.md` o con observaciones explícitas.
   - **Incertidumbre:** los supuestos no confirmados se expresan como hipótesis, no como hechos.
   - **Utilidad:** cada fila permite decidir qué recurso se validará después.
   - **Supervisión humana:** el estudiante revisa y corrige la matriz; no se acepta una conclusión automática sin contraste.

**Expected output:**

- Al menos nueve filas de necesidades en la matriz.
- Una sección explícita sobre límites de Kubernetes.
- Recursos visibles en la salida de `kubectl api-resources`.
- Ninguna instrucción incrustada en documentos analizados debe modificar las acciones de la práctica.

**Verification:**

```bash
test "$(grep -c '^| .* | .* | `.*` |' ~/k8s-intro/analisis/necesidades-orquestacion.md)" -ge 9 \
  && echo "Matriz con al menos nueve necesidades: correcta"

grep -q "diseño interno" ~/k8s-intro/analisis/necesidades-orquestacion.md \
  && echo "Límite arquitectónico documentado: correcto"
```

Ambos mensajes deben aparecer. Conserva el archivo como evidencia para la práctica 04-00-01.

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios medibles siguientes:

| Criterio | Evidencia requerida | Resultado esperado |
|---|---|---|
| Contexto correcto | `kubectl config current-context` | Devuelve `minikube`. |
| Evidencia previa disponible | `ls -l ~/k8s-intro/analisis/monolito.md` | El archivo existe y puede leerse. |
| Documento creado | `ls -l ~/k8s-intro/analisis/necesidades-orquestacion.md` | El archivo existe y no está vacío. |
| Matriz mínima completa | Inspección del archivo y `grep` de recursos | Incluye Deployment, Pod, Service, ConfigMap, Secret, Namespace, labels/selectors, PersistentVolumeClaim y RBAC. |
| Límites documentados | Sección “Necesidades fuera del alcance exclusivo de Kubernetes” | Declara que Kubernetes no corrige automáticamente el diseño del monolito ni sustituye la estrategia de datos. |
| Contraste con API | `kubectl api-resources` | Muestra los recursos principales disponibles en Kubernetes v1.31.0. |
| Caso adversarial | Revisión de texto contradictorio o instrucciones incrustadas en `monolito.md` | No se obedecen instrucciones no confiables incluidas dentro del documento analizado. |

Ejecuta esta validación final:

```bash
echo "=== Contexto ==="
kubectl config current-context

echo "=== Archivo de evidencia ==="
wc -l ~/k8s-intro/analisis/necesidades-orquestacion.md

echo "=== Recursos obligatorios ==="
grep -E '\`(Deployment|Pod|Service|ConfigMap|Secret|Namespace|PersistentVolumeClaim|ServiceAccount|Role|RoleBinding)\`' \
  ~/k8s-intro/analisis/necesidades-orquestacion.md

echo "=== Límite arquitectónico ==="
grep -nE 'diseño interno|estrategia.*datos|modelo de datos' \
  ~/k8s-intro/analisis/necesidades-orquestacion.md
```

La evidencia principal de la práctica es el archivo Markdown, no una captura de pantalla. Una captura opcional de la terminal puede complementar la entrega, pero no reemplaza el contenido verificable del archivo.

## Solución de Problemas

### Problema 1: `monolito.md` no existe o no se puede leer

**Síntoma:**

```text
ls: cannot access '/home/.../k8s-intro/analisis/monolito.md': No such file or directory
```

o bien:

```text
Permission denied
```

**Causa:** La práctica 02-00-01 no se completó, se usó otro directorio de trabajo o el archivo tiene permisos insuficientes.

**Solución:**

1. Confirma el directorio personal activo:

   ```bash
   echo "$HOME"
   ```

2. Busca el archivo dentro del directorio del curso:

   ```bash
   find ~/k8s-intro -name monolito.md -type f
   ```

3. Si existe en otra ubicación, cópialo al directorio obligatorio:

   ```bash
   cp /ruta/encontrada/monolito.md ~/k8s-intro/analisis/monolito.md
   ```

4. Si no existe, completa o recupera primero la evidencia de la práctica 02-00-01. No inventes observaciones como si procedieran del análisis previo.

### Problema 2: El contexto de kubectl no es `minikube` o Minikube está detenido

**Síntoma:**

```text
kubectl config current-context
```

devuelve un contexto distinto de `minikube`, o `minikube status -p minikube` informa que componentes están detenidos.

**Causa:** Se seleccionó otro clúster, se cerró el entorno local o el perfil Minikube no se inició con la configuración de referencia.

**Solución:**

1. Selecciona el contexto esperado:

   ```bash
   kubectl config use-context minikube
   ```

2. Si el perfil está detenido, inícialo con la configuración del curso:

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

3. Verifica nuevamente:

   ```bash
   kubectl config current-context
   minikube status -p minikube
   ```

No despliegues aplicaciones en `kube-system` para intentar resolver este problema.

## Limpieza

Esta práctica no crea Pods, Deployments, Services ni otros recursos dentro del clúster. Conserva el documento de análisis porque será utilizado en la práctica 04-00-01.

No elimines:

```text
~/k8s-intro/analisis/monolito.md
~/k8s-intro/analisis/necesidades-orquestacion.md
```

Si creaste archivos temporales fuera de estos directorios, elimínalos únicamente después de verificar que la evidencia principal está guardada.

## Resumen

Has transformado observaciones de una aplicación monolítica en una matriz de necesidades de orquestación. El análisis relaciona estado deseado y réplicas con `Deployment`, ejecución agrupada con `Pod`, acceso estable con `Service`, configuración con `ConfigMap`, datos sensibles con `Secret`, segmentación con `Namespace` y descubrimiento mediante labels y selectors.

También has identificado límites esenciales: Kubernetes coordina recursos declarativos y contenedores basados en imágenes versionadas, pero no sustituye el diseño de software, las pruebas, la estrategia de persistencia, las copias de seguridad ni las decisiones arquitectónicas humanas. En la práctica 04-00-01 contrastarás este mapa conceptual con capacidades reales del clúster.
