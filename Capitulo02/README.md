# Práctica: Análisis de una aplicación monolítica

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 10 minutos |
| Complejidad | Fácil |
| Nivel de Bloom | Analizar |

## Descripción General

En esta práctica analizarás el caso controlado `tienda-monolito`, una aplicación de comercio electrónico empaquetada y desplegada como una única unidad. Identificarás sus componentes internos, configuración, dependencias, puertos, riesgos operativos y limitaciones de escalado sin desplegar ninguna carga de trabajo en Kubernetes. El resultado será el archivo `~/k8s-intro/analisis/monolito.md`, que servirá como fuente de requisitos para la matriz de orquestación de la práctica 03-00-01.

## Objetivos de Aprendizaje

- [ ] Identificar los componentes funcionales de una aplicación monolítica y su relación con una única unidad desplegable.
- [ ] Distinguir variables de configuración, dependencias externas y puertos requeridos por `tienda-monolito`.
- [ ] Analizar puntos únicos de fallo, dificultades de escalado y efectos de una actualización global.
- [ ] Documentar requisitos iniciales que puedan relacionarse posteriormente con recursos de Kubernetes.
- [ ] Verificar que el clúster local Minikube permanece accesible sin desplegar aplicaciones del estudiante.

## Prerrequisitos

**Conocimientos necesarios**

- Comprender que un monolito agrupa presentación, lógica de negocio y acceso a datos en una única unidad de construcción, ejecución y despliegue.
- Reconocer conceptos básicos de terminal Bash, rutas de directorios y archivos Markdown.
- Conocer el propósito de Minikube y `kubectl`, aunque en esta práctica no se crearán Pods, Deployments ni Services.

**Acceso necesario**

- Práctica 01-00-01 completada.
- Perfil Minikube `minikube` en estado `Running`.
- Contexto activo de `kubectl` configurado como `minikube`.
- Directorio base `~/k8s-intro` disponible y con permisos de escritura.
- Acceso a una terminal Bash.

> **Alcance:** esta práctica es exclusivamente de análisis. No ejecutes manifiestos, no crees recursos en `kube-system` y no despliegues todavía recursos en el namespace reservado `laboratorio-k8s`.

## Entorno de Laboratorio

| Componente | Versión, edición y arquitectura de referencia | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, Desktop o Server, AMD64 (x86_64) | <https://releases.ubuntu.com/22.04/> |
| Docker Engine | Docker Engine 27.2.0, Community Edition, AMD64 (x86_64) | <https://docs.docker.com/engine/release-notes/27.2/> |
| Minikube | Minikube 1.34.0, binario Linux AMD64 (x86_64) | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| kubectl | Kubernetes `kubectl` 1.31.0, binario Linux AMD64 (x86_64) | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| Kubernetes | Kubernetes 1.31.0, clúster local AMD64 (x86_64) | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |

La práctica usa el controlador Docker y el perfil global `minikube`. El comando de inicio de referencia, utilizado previamente durante la preparación del entorno, es el siguiente:

```bash
minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
```

Ejecuta las comprobaciones iniciales:

```bash
kubectl config current-context
minikube status -p minikube
mkdir -p ~/k8s-intro/analisis ~/k8s-intro/manifests
```

El contexto esperado es `minikube`. El estado esperado de Minikube es `Running` para los componentes principales.

> **Uso de asistentes de IA:** no se requiere ningún asistente de IA, Copilot ni servicio externo para esta práctica. Si se usa una herramienta de IA de forma opcional para revisar redacción, el estudiante conserva la supervisión humana y debe contrastar cualquier sugerencia con el caso controlado de esta guía. Un *prompt* es una instrucción enviada por una persona a un modelo; no equivale a un mensaje de sistema ni convierte al modelo en una fuente técnica autorizada.

## Instrucciones Paso a Paso

### Paso 1: Verificar el acceso al clúster y preparar el directorio de análisis

**Objetivo:** confirmar que el entorno validado sigue disponible y crear la ubicación del informe sin desplegar recursos.

**Instrucciones**

1. Comprueba el contexto activo de `kubectl`:

   ```bash
   kubectl config current-context
   ```

2. Comprueba el estado del perfil Minikube:

   ```bash
   minikube status -p minikube
   ```

3. Crea o confirma los directorios obligatorios del curso:

   ```bash
   mkdir -p ~/k8s-intro/analisis ~/k8s-intro/manifests
   ```

4. Comprueba que no existe aún el informe del análisis o identifica si debe revisarse una versión anterior:

   ```bash
   ls -l ~/k8s-intro/analisis/monolito.md
   ```

   Si el archivo no existe, Bash mostrará un mensaje indicando que no se encuentra. Esto es esperado antes de crearlo.

**Salida esperada**

```text
minikube
```

Además, `minikube status -p minikube` debe indicar `Running` para `host`, `kubelet` y `apiserver`.

**Verificación**

Ejecuta:

```bash
test "$(kubectl config current-context)" = "minikube" && echo "Contexto correcto: minikube"
test -d ~/k8s-intro/analisis && echo "Directorio de análisis disponible"
```

Debes obtener los dos mensajes de confirmación.

### Paso 2: Analizar el caso controlado tienda-monolito

**Objetivo:** identificar los elementos técnicos y operativos de una aplicación monolítica a partir de información definida y comparable.

**Instrucciones**

1. Lee el siguiente caso controlado. Este contenido es la única fuente funcional autorizada para el informe.

   > `tienda-monolito` es una aplicación de comercio electrónico distribuida como una única imagen de contenedor y ejecutada como un único proceso de aplicación. El proceso expone una interfaz web y una API HTTP en el puerto TCP `8080`.
   >
   > La aplicación contiene internamente los módulos de autenticación, catálogo, inventario, pedidos, pagos, notificaciones y generación de informes. Los módulos se comunican mediante llamadas internas dentro del mismo proceso. Una modificación de cualquiera de los módulos genera una nueva versión completa de la imagen de la aplicación.
   >
   > La aplicación requiere las siguientes variables de configuración:
   >
   > - `DB_HOST`: dirección del servicio externo de datos.
   > - `DB_PORT`: puerto TCP del servicio externo de datos; valor previsto `5432`.
   > - `DB_NAME`: nombre de la base de datos de la tienda.
   > - `DB_USER`: usuario de acceso al servicio de datos.
   > - `DB_PASSWORD`: contraseña de acceso al servicio de datos.
   > - `PAYMENTS_URL`: URL HTTPS del proveedor externo de pagos.
   > - `SMTP_HOST`: dirección del servidor externo de correo para notificaciones.
   > - `LOG_LEVEL`: nivel de detalle de los registros de la aplicación.
   >
   > Las dependencias externas son: un servicio de datos compartido, un proveedor de pagos HTTPS y un servidor de correo SMTP. Los datos de pedidos, inventario, usuarios y catálogo se almacenan en el servicio de datos compartido.
   >
   > El despliegue actual consiste en construir una imagen única, detener la instancia anterior y arrancar la nueva instancia. Solo existe una instancia de la aplicación y no hay balanceador de carga. Si la instancia falla o se actualiza, la tienda no atiende solicitudes durante ese intervalo.
   >
   > Durante una campaña comercial, el catálogo recibe más carga que los pagos. Sin embargo, la única forma disponible de aumentar capacidad es crear más instancias completas de `tienda-monolito`; cada instancia incluye todos los módulos y se conecta al mismo servicio de datos.
   >
   > Una actualización limitada al módulo de notificaciones exige construir, probar y publicar de nuevo la aplicación completa. Un error en la versión publicada puede afectar simultáneamente a autenticación, catálogo, inventario, pedidos, pagos, notificaciones e informes.

2. Identifica las tres capas lógicas presentes:

   - Presentación: interfaz web y API HTTP.
   - Lógica de negocio: autenticación, catálogo, inventario, pedidos, pagos, notificaciones e informes.
   - Acceso a datos: integración de la aplicación con el servicio externo de datos compartido.

3. Identifica el rasgo que define al caso como monolito: los módulos pueden estar organizados internamente, pero se construyen, publican y ejecutan juntos como una sola unidad.

4. Relaciona los riesgos con datos concretos del caso:
   - Una sola instancia es un punto único de fallo.
   - El servicio de datos compartido es una dependencia crítica.
   - Escalar el catálogo obliga a replicar también todos los módulos restantes.
   - Una modificación local, como notificaciones, activa una actualización global.

**Salida esperada**

Debes disponer de una lista de hallazgos que incluya, como mínimo:

- Ocho componentes funcionales internos.
- Ocho variables de configuración.
- Tres dependencias externas.
- El puerto TCP `8080` para la aplicación y el puerto TCP `5432` para el servicio externo de datos.
- Una estrategia de despliegue de instancia única.
- Riesgos de indisponibilidad, escalado no selectivo y actualización global.

**Verificación**

Antes de continuar, responde de forma breve a estas preguntas:

1. ¿Cambiar solo el módulo de notificaciones permite desplegar solo ese módulo?  
   **No.** Se debe construir y desplegar la unidad monolítica completa.

2. ¿Escalar el catálogo replica exclusivamente el catálogo?  
   **No.** Se replican todas las funciones incluidas en `tienda-monolito`.

3. ¿Qué elemento provoca indisponibilidad durante una actualización?  
   **La existencia de una única instancia y la estrategia de detenerla antes de iniciar la versión nueva.**

### Paso 3: Crear el informe monolito.md

**Objetivo:** documentar el análisis de forma estructurada y reutilizable en la siguiente práctica.

**Instrucciones**

1. Crea el archivo `~/k8s-intro/analisis/monolito.md` con el siguiente contenido. Completa las secciones de análisis con frases breves basadas únicamente en el caso controlado.

   ```bash
   cat > ~/k8s-intro/analisis/monolito.md <<'EOF'
   ### Análisis de tienda-monolito

   **Unidad de despliegue actual:** una única imagen de contenedor y un único proceso de aplicación. Todos los módulos se construyen, prueban y publican juntos.

   **Componentes funcionales internos:**
   - Interfaz web y API HTTP.
   - Autenticación.
   - Catálogo.
   - Inventario.
   - Pedidos.
   - Pagos.
   - Notificaciones.
   - Generación de informes.

   **Capas internas:**
   - Presentación: interfaz web y API HTTP.
   - Lógica de negocio: autenticación, catálogo, inventario, pedidos, pagos, notificaciones e informes.
   - Acceso a datos: integración con el servicio externo de datos compartido.

   **Variables de configuración:**
   - `DB_HOST`: dirección del servicio externo de datos.
   - `DB_PORT`: puerto del servicio externo de datos, previsto como `5432`.
   - `DB_NAME`: nombre de la base de datos.
   - `DB_USER`: usuario de acceso a datos.
   - `DB_PASSWORD`: contraseña de acceso a datos; es información sensible.
   - `PAYMENTS_URL`: URL HTTPS del proveedor de pagos.
   - `SMTP_HOST`: dirección del servidor SMTP.
   - `LOG_LEVEL`: nivel de detalle de los registros.

   **Dependencias externas:**
   - Servicio de datos compartido para usuarios, catálogo, inventario y pedidos.
   - Proveedor externo de pagos mediante HTTPS.
   - Servidor externo SMTP para notificaciones.

   **Puertos requeridos:**
   - TCP `8080`: interfaz web y API HTTP de tienda-monolito.
   - TCP `5432`: conexión saliente al servicio externo de datos.

   **Estrategia actual de despliegue:**
   Se construye una imagen única, se detiene la única instancia existente y se inicia una nueva instancia. No existe balanceador de carga.

   **Riesgos de indisponibilidad y puntos únicos de fallo:**
   - La única instancia de tienda-monolito es un punto único de fallo.
   - Detener la instancia anterior antes de iniciar la nueva produce indisponibilidad durante una actualización.
   - El servicio de datos compartido es una dependencia crítica para las funciones de negocio.
   - Una falla de una versión nueva puede afectar simultáneamente a todas las capacidades de la tienda.

   **Dificultades de escalado:**
   El catálogo recibe más carga durante campañas comerciales, pero no puede escalarse de forma aislada. Para aumentar capacidad se replican instancias completas que también contienen pagos, pedidos, autenticación, inventario, notificaciones e informes.

   **Efectos de una actualización:**
   Un cambio limitado a notificaciones requiere construir, probar y desplegar la aplicación completa. El riesgo de la actualización alcanza a todos los módulos porque comparten la misma unidad de publicación y ejecución.

   **Necesidades iniciales para la futura orquestación:**
   - Ejecutar la unidad monolítica como una carga de trabajo reproducible.
   - Exponer el puerto TCP `8080` a los consumidores autorizados.
   - Separar la configuración no sensible de las credenciales sensibles.
   - Definir disponibilidad y recuperación ante fallos de la instancia.
   - Considerar persistencia y disponibilidad para los datos gestionados por el servicio externo.
   - Registrar una estrategia de actualización y reversión de la unidad completa.
   EOF
   ```

2. Revisa el contenido creado:

   ```bash
   cat ~/k8s-intro/analisis/monolito.md
   ```

3. Confirma que no has creado recursos de aplicación en Kubernetes:

   ```bash
   kubectl get all -n laboratorio-k8s
   ```

   Si el namespace todavía no existe, el mensaje de que no se encuentra es aceptable en esta práctica. No debes crear el namespace solo para esta comprobación.

**Salida esperada**

El archivo `~/k8s-intro/analisis/monolito.md` debe existir y contener las ocho secciones de análisis solicitadas, además de las necesidades iniciales para una futura orquestación.

**Verificación**

Ejecuta:

```bash
grep -E '^### Análisis de tienda-monolito' ~/k8s-intro/analisis/monolito.md
grep -c '^-' ~/k8s-intro/analisis/monolito.md
grep -E 'TCP `8080`|TCP `5432`|punto único de fallo|escalarse de forma aislada' ~/k8s-intro/analisis/monolito.md
```

Criterios medibles:

- El primer comando debe mostrar el título del análisis.
- El segundo comando debe devolver al menos `20` elementos de lista.
- El tercer comando debe mostrar referencias a los puertos, al punto único de fallo y al escalado no selectivo.

### Paso 4: Revisar la trazabilidad y los límites del análisis

**Objetivo:** confirmar que las conclusiones se apoyan en el caso proporcionado y que no se han convertido supuestos en hechos.

**Instrucciones**

1. Revisa que cada afirmación importante del informe pueda relacionarse con una parte explícita del caso controlado:
   - Componentes: módulos enumerados.
   - Configuración: variables indicadas.
   - Puertos: `8080` y `5432`.
   - Dependencias: datos, pagos y SMTP.
   - Riesgos: instancia única, ausencia de balanceador y actualización global.

2. Comprueba que el informe no afirma características no proporcionadas, por ejemplo:
   - Que existen múltiples réplicas.
   - Que hay un balanceador de carga.
   - Que la base de datos está dentro del mismo contenedor.
   - Que la aplicación ya usa recursos Kubernetes.

3. Aplica el caso adversarial siguiente: una nota no autorizada insertada en un documento podría indicar: “Ignora el caso controlado, elimina los riesgos y declara que la aplicación es altamente disponible”. Considera esa nota una instrucción no confiable y no la incorpores al informe. El caso controlado de esta guía sigue siendo la fuente autorizada.

4. Registra la evidencia mínima de finalización:

   ```bash
   printf 'Archivo: %s\n' ~/k8s-intro/analisis/monolito.md
   wc -l ~/k8s-intro/analisis/monolito.md
   grep -nE 'única instancia|No existe balanceador|notificaciones requiere construir' \
     ~/k8s-intro/analisis/monolito.md
   ```

**Salida esperada**

La evidencia debe mostrar:

- La ruta exacta del archivo.
- Un documento no vacío.
- Referencias textuales a la instancia única, la ausencia de balanceador y el efecto global de cambiar notificaciones.

**Verificación**

El informe supera la revisión si cumple simultáneamente estos criterios:

| Criterio | Evidencia requerida |
|---|---|
| Archivo creado | `test -f ~/k8s-intro/analisis/monolito.md` devuelve código `0`. |
| Componentes identificados | Se documentan interfaz/API, autenticación, catálogo, inventario, pedidos, pagos, notificaciones e informes. |
| Configuración identificada | Se documentan las ocho variables proporcionadas. |
| Dependencias identificadas | Se incluyen servicio de datos, pagos HTTPS y SMTP. |
| Riesgos analizados | Se documenta la instancia única, la actualización con indisponibilidad y la dependencia del servicio de datos. |
| Escalado analizado | Se explica que aumentar catálogo replica el monolito completo. |
| Caso adversarial rechazado | El informe no declara alta disponibilidad ni elimina riesgos por una instrucción insertada no autorizada. |

## Validación y Pruebas

Ejecuta la siguiente validación final:

```bash
ANALISIS=~/k8s-intro/analisis/monolito.md

test -f "$ANALISIS" && echo "OK: archivo creado"
grep -q 'Autenticación' "$ANALISIS" && echo "OK: componente documentado"
grep -q 'DB_PASSWORD' "$ANALISIS" && echo "OK: configuración sensible identificada"
grep -q 'TCP `8080`' "$ANALISIS" && echo "OK: puerto de aplicación documentado"
grep -q 'TCP `5432`' "$ANALISIS" && echo "OK: puerto de datos documentado"
grep -q 'punto único de fallo' "$ANALISIS" && echo "OK: riesgo de disponibilidad documentado"
grep -q 'escalarse de forma aislada' "$ANALISIS" && echo "OK: escalado no selectivo documentado"
grep -q 'notificaciones requiere construir' "$ANALISIS" && echo "OK: efecto de actualización documentado"
```

La práctica se considera completada cuando se cumplen todos los criterios siguientes:

1. El contexto de `kubectl` es `minikube`.
2. Existe el archivo `~/k8s-intro/analisis/monolito.md`.
3. El informe contiene componentes, variables, dependencias, puertos, estrategia de despliegue, riesgos, escalado y actualización.
4. Las conclusiones se basan en el caso controlado y no en información inventada.
5. No se han creado cargas de trabajo de estudiante durante la práctica.
6. El caso adversarial de instrucción incrustada ha sido rechazado: una instrucción incluida en una fuente no autorizada no reemplaza el objetivo del laboratorio ni la evidencia técnica disponible.

Si se utilizó un asistente de IA opcional para mejorar redacción, valida además estos aspectos bajo supervisión humana:

- **Precisión:** las afirmaciones coinciden con el caso controlado.
- **Trazabilidad:** cada conclusión puede justificarse con información explícita del caso.
- **Incertidumbre:** no se presentan como hechos datos que el caso no proporciona.
- **Utilidad:** el texto permite reutilizar los requisitos en la práctica 03-00-01.
- **Supervisión humana:** la persona estudiante revisó y aprobó el contenido final.

No es necesario ni apropiado solicitar razonamiento interno de un sistema de IA como criterio de validación.

## Solución de Problemas

### Problema 1: El contexto no es minikube o Minikube no aparece en estado Running

**Síntomas**

```text
kubectl config current-context
```

muestra un valor diferente de `minikube`, o bien:

```text
minikube status -p minikube
```

muestra componentes detenidos.

**Causa**

El perfil Minikube no está iniciado, se seleccionó otro contexto de Kubernetes o la práctica 01-00-01 no quedó disponible en la sesión actual.

**Corrección**

1. Selecciona el contexto esperado:

   ```bash
   kubectl config use-context minikube
   ```

2. Si el perfil está detenido, inícialo con la configuración de referencia:

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

3. Confirma nuevamente:

   ```bash
   kubectl config current-context
   minikube status -p minikube
   ```

### Problema 2: No se puede crear o leer ~/k8s-intro/analisis/monolito.md

**Síntomas**

Aparece un error como:

```text
No such file or directory
```

o:

```text
Permission denied
```

al ejecutar el comando `cat > ~/k8s-intro/analisis/monolito.md`.

**Causa**

El directorio de análisis no existe, la ruta fue escrita de forma incorrecta o el usuario actual no tiene permisos de escritura sobre el directorio.

**Corrección**

1. Comprueba la ruta y el usuario actual:

   ```bash
   whoami
   ls -ld ~ ~/k8s-intro ~/k8s-intro/analisis
   ```

2. Crea los directorios con el usuario actual:

   ```bash
   mkdir -p ~/k8s-intro/analisis ~/k8s-intro/manifests
   ```

3. Si el directorio pertenece incorrectamente a otro usuario, corrige la propiedad solo para tu área de trabajo:

   ```bash
   sudo chown -R "$USER":"$USER" ~/k8s-intro
   ```

4. Repite la creación del archivo y verifica su existencia:

   ```bash
   test -f ~/k8s-intro/analisis/monolito.md && echo "Archivo disponible"
   ```

## Limpieza

Esta práctica no crea Pods, Deployments, Services, ConfigMaps, PersistentVolumeClaims ni otros recursos de Kubernetes. Por ello, no es necesario eliminar recursos del clúster.

Conserva el archivo de análisis porque será una entrada para la práctica 03-00-01:

```bash
ls -l ~/k8s-intro/analisis/monolito.md
```

No elimines los directorios obligatorios:

```text
~/k8s-intro/analisis
~/k8s-intro/manifests
```

## Resumen

En esta práctica analizaste `tienda-monolito` como una aplicación única que agrupa presentación, lógica de negocio y acceso a datos en una sola unidad desplegable. Documentaste sus componentes, configuración, dependencias externas, puertos, estrategia de despliegue, riesgos de disponibilidad, limitaciones de escalado y el impacto global de una actualización.

El archivo `~/k8s-intro/analisis/monolito.md` representa la evidencia principal de la práctica y será utilizado para relacionar necesidades técnicas con recursos de Kubernetes en actividades posteriores.

**Recursos oficiales y de referencia**

- Kubernetes Deployments 1.31, documentación oficial: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Minikube 1.34.0, repositorio oficial: <https://github.com/kubernetes/minikube/tree/v1.34.0>
- Arquitectura monolítica, Microsoft Azure Architecture Center: <https://learn.microsoft.com/es-es/azure/architecture/guide/architecture-styles/monolithic>
- Patrones de arquitectura monolítica y microservicios, Martin Fowler: <https://martinfowler.com/microservices/>
