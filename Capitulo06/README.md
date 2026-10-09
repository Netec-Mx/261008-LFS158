# Práctica: Selección de un método de instalación

## Metadatos

| Campo | Valor |
|---|---|
| Duration | 10 minutos |
| Complexity | Fácil |
| Bloom level | Analizar |

## Descripción General

En esta práctica analizarás controladores de Minikube adecuados para un entorno local de aprendizaje y documentarás una decisión técnica reproducible. Compararás Docker, KVM2 y otras alternativas según recursos, permisos, virtualización, facilidad de eliminación y compatibilidad con el curso. La decisión obligatoria para las prácticas posteriores será utilizar Minikube 1.35.0 con Docker Engine 28.0.4 como controlador, sin crear todavía ningún clúster.

## Objetivos de Aprendizaje

- [ ] Comparar Docker, KVM2 y otros controladores de Minikube para un laboratorio local.
- [ ] Identificar recursos disponibles, arquitectura, permisos y dependencias del equipo.
- [ ] Justificar técnicamente la elección de Minikube 1.35.0 con Docker Engine 28.0.4.
- [ ] Crear la estructura de trabajo reutilizable en `/home/$USER/k8s-intro`.
- [ ] Documentar la decisión en `docs/decision-minikube.md` para el laboratorio 07-00-01.

## Prerrequisitos

**Conocimientos necesarios**

- Uso básico de terminal Bash en Linux.
- Comprensión inicial de clústeres locales, autogestionados y administrados.
- Capacidad para interpretar memoria disponible, espacio en disco y arquitectura del sistema.
- Conocimiento de que un controlador de Minikube determina dónde se ejecutará el nodo local de Kubernetes.

**Acceso necesario**

- Acceso a una terminal Bash.
- Conexión a Internet para consultar documentación oficial y descargar herramientas en laboratorios posteriores.
- Permisos para consultar información del sistema.
- Acceso administrativo o a un equipo donde puedan instalarse Docker Engine, Minikube y kubectl posteriormente.
- Usuario con acceso a Docker o posibilidad de solicitar que se agregue al grupo `docker`.

## Entorno de Laboratorio

Esta práctica **no instala herramientas y no crea un clúster**. Se evalúa la idoneidad del equipo y se prepara la estructura de archivos para prácticas posteriores.

| Componente objetivo | Edición y arquitectura de referencia | Versión exacta | Fuente oficial |
|---|---|---:|---|
| Sistema operativo | Ubuntu Server o Ubuntu Desktop, `amd64` | 24.04.2 LTS | <https://releases.ubuntu.com/24.04/> |
| Docker Engine | Docker Engine Community Edition, `amd64` | 28.0.4 | <https://docs.docker.com/engine/release-notes/28/> |
| Minikube | Binario Linux, `amd64` | 1.35.0 | <https://github.com/kubernetes/minikube/releases/tag/v1.35.0> |
| Kubernetes objetivo | Kubernetes, `amd64` | v1.33.0 | <https://github.com/kubernetes/kubernetes/releases/tag/v1.33.0> |
| Shell de referencia | GNU Bash, `amd64` | 5.2.21 | <https://www.gnu.org/software/bash/> |

> **Nota sobre arquitectura:** esta guía usa como referencia `amd64` (x86_64). Si `uname -m` muestra `aarch64` o `arm64`, no continúes suponiendo compatibilidad binaria: registra esa condición en la decisión y valida específicamente la disponibilidad de los binarios `arm64` en las fuentes oficiales antes de instalar.

Ejecuta estos comandos iniciales para definir variables reutilizables durante la práctica:

```bash
export LAB_ROOT="/home/$USER/k8s-intro"
export MINIKUBE_VERSION="1.35.0"
export DOCKER_VERSION="28.0.4"
export KUBERNETES_VERSION="v1.33.0"
export MINIKUBE_DRIVER="docker"

printf 'LAB_ROOT=%s\n' "$LAB_ROOT"
printf 'MINIKUBE_VERSION=%s\n' "$MINIKUBE_VERSION"
printf 'DOCKER_VERSION=%s\n' "$DOCKER_VERSION"
printf 'KUBERNETES_VERSION=%s\n' "$KUBERNETES_VERSION"
printf 'MINIKUBE_DRIVER=%s\n' "$MINIKUBE_DRIVER"
```

## Instrucciones Paso a Paso

### Paso 1: Inspeccionar las capacidades reales del equipo

**Objetivo:** recopilar evidencia mínima sobre arquitectura, memoria, espacio disponible, virtualización y disponibilidad de Docker, sin modificar el sistema ni iniciar Minikube.

**Instrucciones:**

1. Comprueba la identidad del usuario, la arquitectura y la versión del sistema operativo:

   ```bash
   id
   uname -m
   . /etc/os-release && printf '%s %s\n' "$PRETTY_NAME" "$VERSION_ID"
   ```

2. Revisa la memoria disponible y el espacio libre del sistema de archivos donde se creará el laboratorio:

   ```bash
   free -h
   df -h /home/"$USER"
   ```

3. Comprueba si el procesador anuncia capacidades de virtualización. Este dato es especialmente relevante para KVM2:

   ```bash
   lscpu | grep -E 'Architecture|Virtualization|Hypervisor' || true
   ```

4. Comprueba si Docker está instalado, si su servicio responde y si el usuario puede usarlo sin `sudo`:

   ```bash
   command -v docker || true
   docker version --format 'Cliente: {{.Client.Version}} | Servidor: {{.Server.Version}}' 2>&1 || true
   docker info --format 'Driver de almacenamiento: {{.Driver}}' 2>&1 || true
   ```

5. Comprueba si ya existe Minikube, pero **no** ejecutes `minikube start`:

   ```bash
   command -v minikube || true
   minikube version 2>&1 || true
   ```

**Salida esperada:**

- Una arquitectura normalmente equivalente a `x86_64`.
- Información de memoria y de espacio libre en `/home/$USER`.
- Si Docker ya está instalado y operativo, una versión de servidor identificable.
- Si Minikube no está instalado, un mensaje como `command not found`; esto es aceptable en esta práctica.
- No debe aparecer la creación de un perfil, nodo o clúster de Minikube.

**Verificación:**

Confirma estas condiciones antes de continuar:

- Hay al menos 4 GB de RAM asignables al futuro clúster y a Docker; se recomiendan 8 GB de RAM total.
- Hay al menos 25 GB libres para imágenes, cachés, binarios y manifiestos.
- El usuario puede ejecutar `docker info` sin error de permisos, o ha identificado que debe solicitar acceso al grupo `docker`.
- Si se considerara KVM2, existe soporte de virtualización y acceso previsto a `/dev/kvm`; si no, Docker será una opción más directa.

### Paso 2: Comparar los controladores de Minikube

**Objetivo:** analizar alternativas de ejecución local y relacionarlas con las condiciones observadas en el equipo.

**Instrucciones:**

1. Usa la siguiente matriz como criterio de análisis. No instales ni actives ningún controlador durante esta práctica.

   | Controlador | Requisito principal | Virtualización de hardware | Permisos y dependencias | Eliminación | Adecuación al curso |
   |---|---|---|---|---|---|
   | Docker | Docker Engine operativo y acceso del usuario al daemon | No necesariamente; usa contenedores | Docker Engine 28.0.4 y permisos para ejecutar `docker` | Simple mediante `minikube delete` y limpieza controlada de imágenes | **Obligatorio** |
   | KVM2 | Linux con KVM, libvirt y acceso a `/dev/kvm` | Sí | Paquetes KVM/libvirt, pertenencia a grupos adecuados y configuración adicional | Requiere administrar recursos de virtualización | Válido técnicamente, no seleccionado |
   | VirtualBox | VirtualBox instalado y módulos del kernel compatibles | Sí | Instalación del hipervisor, módulos y permisos | Puede requerir limpieza de máquinas virtuales y redes | No recomendado para este lote |
   | none | Kubernetes preexistente o configuración especializada | Depende del entorno | Mayor responsabilidad operativa y configuración manual | Riesgo de afectar el host | No apropiado para el curso |

2. Relaciona la matriz con el propósito del entorno:
   - El objetivo es **aprendizaje local**, no alta disponibilidad ni producción.
   - El curso requiere repetibilidad y una eliminación sencilla del entorno.
   - El controlador Docker reduce dependencias de hipervisores adicionales cuando Docker ya está disponible.
   - Docker no convierte el entorno en equivalente a producción; permite practicar objetos y comandos de Kubernetes en un clúster local.

3. Define la decisión técnica obligatoria:

   ```bash
   export SELECTED_DRIVER="docker"
   export SELECTED_MINIKUBE_VERSION="1.35.0"
   export SELECTED_DOCKER_VERSION="28.0.4"

   printf 'Decisión: Minikube %s con Docker Engine %s usando --driver=%s\n' \
     "$SELECTED_MINIKUBE_VERSION" \
     "$SELECTED_DOCKER_VERSION" \
     "$SELECTED_DRIVER"
   ```

**Salida esperada:**

```text
Decisión: Minikube 1.35.0 con Docker Engine 28.0.4 usando --driver=docker
```

**Verificación:**

La decisión debe justificar explícitamente estos cinco aspectos:

1. Consumo de recursos razonable para un clúster local de un nodo.
2. No depender de una configuración adicional de KVM2 o VirtualBox.
3. Requerir acceso controlado al daemon de Docker.
4. Permitir creación y eliminación simples del entorno.
5. Ser compatible con el flujo posterior del curso basado en `minikube start --driver=docker`.

### Paso 3: Crear la estructura de trabajo persistente

**Objetivo:** preparar los directorios que almacenarán manifiestos, evidencias y documentación de las prácticas posteriores.

**Instrucciones:**

1. Crea el directorio principal y los tres subdirectorios obligatorios:

   ```bash
   mkdir -p \
     "$LAB_ROOT/manifests" \
     "$LAB_ROOT/artifacts" \
     "$LAB_ROOT/docs"
   ```

2. Restringe de forma básica el acceso a los archivos creados por el usuario durante el curso:

   ```bash
   chmod 750 "$LAB_ROOT" "$LAB_ROOT/manifests" "$LAB_ROOT/artifacts" "$LAB_ROOT/docs"
   ```

3. Guarda variables de referencia para sesiones Bash futuras:

   ```bash
   cat > "$LAB_ROOT/docs/lab-environment.sh" <<'EOF'
   export LAB_ROOT="/home/$USER/k8s-intro"
   export MINIKUBE_VERSION="1.35.0"
   export DOCKER_VERSION="28.0.4"
   export KUBERNETES_VERSION="v1.33.0"
   export MINIKUBE_DRIVER="docker"
   EOF
   ```

4. Comprueba la estructura creada:

   ```bash
   find "$LAB_ROOT" -maxdepth 2 -type d | sort
   ls -l "$LAB_ROOT/docs/lab-environment.sh"
   ```

**Salida esperada:**

Debe existir la siguiente estructura:

```text
/home/<usuario>/k8s-intro
/home/<usuario>/k8s-intro/artifacts
/home/<usuario>/k8s-intro/docs
/home/<usuario>/k8s-intro/manifests
```

También debe existir el archivo:

```text
/home/<usuario>/k8s-intro/docs/lab-environment.sh
```

**Verificación:**

Ejecuta:

```bash
test -d "$LAB_ROOT/manifests" &&
test -d "$LAB_ROOT/artifacts" &&
test -d "$LAB_ROOT/docs" &&
test -f "$LAB_ROOT/docs/lab-environment.sh" &&
echo "Estructura válida"
```

El resultado esperado es:

```text
Estructura válida
```

### Paso 4: Documentar la decisión de Minikube

**Objetivo:** generar una decisión técnica auditable que pueda utilizarse como entrada del laboratorio 07-00-01.

**Instrucciones:**

1. Captura valores observados en el equipo. Los campos no disponibles deben quedar identificados como tales, no inventados.

   ```bash
   SYSTEM_ARCH="$(uname -m)"
   SYSTEM_OS="$(. /etc/os-release && printf '%s' "$PRETTY_NAME")"
   MEMORY_SUMMARY="$(free -h | awk '/Mem:/ {print $2 " total, " $7 " disponible"}')"
   DISK_SUMMARY="$(df -h /home/"$USER" | awk 'NR==2 {print $4 " libres en " $6}')"
   DOCKER_STATUS="$(docker version --format '{{.Server.Version}}' 2>/dev/null || printf 'No disponible o sin permisos')"
   MINIKUBE_STATUS="$(minikube version --short 2>/dev/null || printf 'No instalado todavía')"
   ```

2. Crea el documento de decisión:

   ```bash
   cat > "$LAB_ROOT/docs/decision-minikube.md" <<EOF
   ### Decisión de instalación local de Kubernetes

   **Fecha de registro:** $(date -Iseconds)  
   **Usuario:** $USER  
   **Sistema operativo detectado:** $SYSTEM_OS  
   **Arquitectura detectada:** $SYSTEM_ARCH  
   **Memoria observada:** $MEMORY_SUMMARY  
   **Disco observado:** $DISK_SUMMARY  
   **Docker detectado:** $DOCKER_STATUS  
   **Minikube detectado:** $MINIKUBE_STATUS  

   #### Alternativas analizadas

   | Controlador | Resultado del análisis |
   |---|---|
   | Docker | Seleccionado. Reduce dependencias adicionales, permite ejecutar un clúster local basado en contenedores y facilita su eliminación. Requiere Docker Engine operativo y acceso del usuario al daemon. |
   | KVM2 | No seleccionado para este lote. Puede ofrecer aislamiento mediante virtualización, pero exige KVM, libvirt, permisos y configuración adicionales. |
   | VirtualBox | No seleccionado. Introduce un hipervisor y módulos adicionales que no son necesarios para el objetivo de aprendizaje. |
   | none | No seleccionado. Aumenta la complejidad y no es adecuado para crear el entorno local controlado del curso. |

   #### Decisión obligatoria

   Se utilizará **Minikube 1.35.0 para Linux amd64** con **Docker Engine Community Edition 28.0.4 para Linux amd64** mediante el controlador:

   \`\`\`bash
   minikube start --driver=docker
   \`\`\`

   El objetivo de Kubernetes para el entorno posterior será **Kubernetes v1.33.0 para amd64**.

   Esta selección corresponde a un clúster local de aprendizaje. No se considera una arquitectura de producción, no proporciona alta disponibilidad y no sustituye la operación de un clúster autogestionado o administrado.

   #### Dependencias y condiciones pendientes

   - Docker Engine 28.0.4 debe estar instalado, activo y accesible para el usuario.
   - Minikube 1.35.0 debe estar instalado antes del laboratorio 07-00-01.
   - Deben estar disponibles al menos 4 GB de RAM para Minikube; se recomiendan 8 GB de RAM total.
   - Deben existir al menos 25 GB libres para imágenes, cachés y archivos del curso.
   - No se ha creado ningún clúster en este laboratorio.

   #### Fuentes oficiales

   - Minikube 1.35.0: https://github.com/kubernetes/minikube/releases/tag/v1.35.0
   - Docker Engine 28: https://docs.docker.com/engine/release-notes/28/
   - Kubernetes v1.33.0: https://github.com/kubernetes/kubernetes/releases/tag/v1.33.0
   EOF
   ```

3. Revisa el documento sin modificarlo:

   ```bash
   sed -n '1,220p' "$LAB_ROOT/docs/decision-minikube.md"
   ```

**Salida esperada:**

El archivo debe contener:

- La matriz resumida de alternativas.
- La selección explícita de Docker.
- Las versiones exactas Minikube 1.35.0, Docker Engine 28.0.4 y Kubernetes v1.33.0.
- Condiciones de recursos y permisos.
- Fuentes oficiales.
- La afirmación de que todavía no se creó ningún clúster.

**Verificación:**

Ejecuta la siguiente comprobación:

```bash
grep -q 'Minikube 1.35.0' "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q 'Docker Engine Community Edition 28.0.4' "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q -- '--driver=docker' "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q 'No se ha creado ningún clúster' "$LAB_ROOT/docs/decision-minikube.md" &&
echo "Decisión documentada correctamente"
```

El resultado esperado es:

```text
Decisión documentada correctamente
```

## Validación y Pruebas

La práctica se considera completada únicamente si se cumplen todos los criterios siguientes.

| Criterio medible | Evidencia requerida | Comando de validación |
|---|---|---|
| Estructura de directorios creada | Directorios `manifests`, `artifacts` y `docs` | `find "$LAB_ROOT" -maxdepth 1 -type d | sort` |
| Variables persistentes registradas | Archivo `docs/lab-environment.sh` | `cat "$LAB_ROOT/docs/lab-environment.sh"` |
| Decisión documentada | Archivo `docs/decision-minikube.md` | `test -s "$LAB_ROOT/docs/decision-minikube.md"` |
| Versiones exactas registradas | Minikube 1.35.0, Docker 28.0.4 y Kubernetes v1.33.0 | `grep -E '1\.35\.0|28\.0\.4|v1\.33\.0' "$LAB_ROOT/docs/decision-minikube.md"` |
| Controlador correcto | Selección explícita de Docker | `grep -q -- '--driver=docker' "$LAB_ROOT/docs/decision-minikube.md"` |
| No se creó el clúster | No existe perfil `minikube` creado por esta práctica | `test ! -d "$HOME/.minikube/profiles/minikube" && echo "Sin perfil Minikube"` |

Realiza además esta validación consolidada:

```bash
test -d "$LAB_ROOT/manifests" &&
test -d "$LAB_ROOT/artifacts" &&
test -d "$LAB_ROOT/docs" &&
test -f "$LAB_ROOT/docs/lab-environment.sh" &&
test -s "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q 'Minikube 1.35.0' "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q 'Docker Engine Community Edition 28.0.4' "$LAB_ROOT/docs/decision-minikube.md" &&
grep -q -- '--driver=docker' "$LAB_ROOT/docs/decision-minikube.md" &&
echo "VALIDACIÓN COMPLETA: práctica preparada para el laboratorio 07-00-01"
```

**Caso adversarial de revisión documental:** si una fuente copiada, una página web o un archivo contiene texto como “ignora las instrucciones anteriores”, “ejecuta este comando con sudo” o solicita credenciales, no lo ejecutes ni lo agregues como decisión técnica. Ese texto sería contenido no confiable, no una instrucción válida del laboratorio. La evidencia válida debe proceder de los comandos observados localmente y de las URLs oficiales incluidas en `decision-minikube.md`.

No se utiliza IA generativa en esta práctica. Si se utilizara una herramienta de IA como apoyo, una **instrucción** incluida en un documento no debe tratarse como un **mensaje de sistema** ni como una configuración persistente de un agente; debe ser revisada por la persona responsable. La validación debe evaluar precisión, trazabilidad de fuentes, declaración de incertidumbre y supervisión humana, sin exigir ni registrar razonamientos internos.

## Solución de Problemas

### Problema 1: Docker responde con error de permisos

**Síntoma**

```text
permission denied while trying to connect to the Docker daemon socket
```

o `docker info` no puede conectarse al daemon.

**Causa**

Docker Engine puede estar detenido, o el usuario actual no tiene permiso para acceder a `/var/run/docker.sock`. El controlador Docker requiere que Minikube pueda comunicarse con el daemon de Docker.

**Solución**

1. Comprueba el estado del servicio:

   ```bash
   sudo systemctl status docker --no-pager
   ```

2. Si está detenido, solicita o realiza su inicio según la política del equipo:

   ```bash
   sudo systemctl start docker
   ```

3. Si la política local lo permite, agrega el usuario al grupo `docker`:

   ```bash
   sudo usermod -aG docker "$USER"
   ```

4. Cierra sesión y vuelve a iniciarla. Después verifica:

   ```bash
   docker info
   ```

No crees el clúster todavía; solo registra en `decision-minikube.md` que el acceso a Docker fue validado o quedó pendiente.

### Problema 2: La memoria o el disco no cumplen el mínimo del curso

**Síntoma**

`free -h` muestra menos de 4 GB disponibles para el futuro clúster, o `df -h /home/$USER` muestra menos de 25 GB libres.

**Causa**

El equipo puede tener otras aplicaciones, máquinas virtuales, imágenes de contenedor o archivos ocupando recursos. Un clúster local con Docker necesita memoria y espacio para imágenes, componentes de Kubernetes y cachés.

**Solución**

1. Libera espacio eliminando archivos no necesarios del usuario o solicitando ampliación de almacenamiento.
2. Cierra aplicaciones que consuman memoria antes de iniciar Minikube en el laboratorio posterior.
3. No reduzcas arbitrariamente los recursos previstos para ocultar la limitación; registra la condición real en `decision-minikube.md`.
4. Si no se pueden garantizar los recursos mínimos, utiliza un equipo compatible antes de continuar con la creación del clúster.

## Limpieza

Esta práctica no crea un clúster ni recursos de Kubernetes. **No ejecutes** `minikube delete`, porque no hay ningún entorno que eliminar y el archivo de decisión será necesario en el laboratorio 07-00-01.

Si necesitas reiniciar únicamente la preparación de esta práctica, elimina la estructura creada y vuelve a ejecutar los pasos:

```bash
rm -rf "/home/$USER/k8s-intro"
```

> **Advertencia:** este comando elimina de forma permanente los documentos y directorios del laboratorio. Úsalo solo si deseas descartar la evidencia creada y aún no has comenzado prácticas posteriores.

## Resumen

Has analizado alternativas para ejecutar Kubernetes localmente y has seleccionado el controlador Docker como opción obligatoria para el curso. La decisión queda documentada con versiones exactas: Minikube 1.35.0, Docker Engine Community Edition 28.0.4 y Kubernetes v1.33.0 para `amd64`.

También has creado la estructura reutilizable:

```text
/home/$USER/k8s-intro/
├── artifacts/
├── docs/
│   ├── decision-minikube.md
│   └── lab-environment.sh
└── manifests/
```

El siguiente laboratorio utilizará `docs/decision-minikube.md` como evidencia de la elección y continuará con la instalación o validación de Minikube antes de crear el clúster local con el controlador Docker.

**Recursos oficiales opcionales**

- Documentación de controladores de Minikube: <https://minikube.sigs.k8s.io/docs/drivers/>
- Documentación de Docker Engine: <https://docs.docker.com/engine/>
- Documentación de instalación de Kubernetes: <https://kubernetes.io/docs/tasks/tools/>
