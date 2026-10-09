# Práctica: Exploración de la comunidad Kubernetes

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 15 minutos |
| Complejidad | Fácil |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica explorarás fuentes oficiales de documentación, comunidad, discusión y contribución de Kubernetes. Partiendo del incidente controlado `ImagePullBackOff` y de la recuperación del Deployment `web-demo` realizada en el laboratorio 16-00-01, recopilarás información técnica mínima, clasificarás canales comunitarios y prepararás una consulta responsable sin publicar mensajes ni datos sensibles. El producto final será el archivo local `~/k8s-intro/analisis/community-map.md`.

## Objetivos de Aprendizaje

- [ ] Localizar documentación, comunidad, repositorios y canales de discusión oficiales de Kubernetes.
- [ ] Diferenciar el propósito de la documentación oficial, Discuss Kubernetes, Kubernetes Slack y GitHub.
- [ ] Recopilar información técnica relevante sobre el incidente `ImagePullBackOff` sin exponer secretos, tokens ni datos internos.
- [ ] Elaborar una ficha reutilizable con recursos oficiales y un borrador de consulta técnica reproducible.
- [ ] Reconocer que una duda operativa o educativa no necesariamente debe convertirse en una incidencia del repositorio de Kubernetes.

## Prerrequisitos

**Conocimientos necesarios**

- Uso básico de Bash y edición de archivos de texto.
- Uso de `kubectl get`, `kubectl describe`, `kubectl rollout history` y `kubectl version`.
- Comprensión básica de Namespace, Pod, Deployment, eventos de Kubernetes e imágenes de contenedor.
- Haber completado el laboratorio 16-00-01, con el Deployment `web-demo` recuperado a `nginx:1.27.1`.

**Acceso necesario**

- Acceso a Internet y navegador web.
- Terminal Bash.
- Acceso funcional al contexto `minikube`.
- Permisos para leer recursos del Namespace `laboratorio-k8s`.
- Directorios `~/k8s-intro/analisis` y `~/k8s-intro/manifests`.

> No publiques mensajes, incidencias, capturas ni manifiestos en foros, Slack o GitHub durante esta práctica. El objetivo es preparar información de calidad para una futura consulta responsable.

## Entorno de Laboratorio

El laboratorio reutiliza el clúster local estable creado en prácticas anteriores. No se instalará software ni se modificarán recursos de Kubernetes.

| Componente | Versión, edición y arquitectura de referencia | Fuente oficial |
|---|---|---|
| Sistema operativo | Ubuntu 22.04.5 LTS, Desktop o Server, `amd64` | <https://releases.ubuntu.com/22.04.5/> |
| Docker Engine | Docker Engine 27.2.0, Community Edition, `amd64` | <https://docs.docker.com/engine/release-notes/27.2/> |
| Minikube | Minikube 1.34.0, binario Linux `amd64` | <https://github.com/kubernetes/minikube/releases/tag/v1.34.0> |
| Kubernetes | Kubernetes v1.31.0, clúster local `amd64` | <https://github.com/kubernetes/kubernetes/releases/tag/v1.31.0> |
| kubectl | kubectl v1.31.0, Linux `amd64` | <https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl> |
| Imagen recuperada | NGINX Docker Official Image 1.27.1, arquitectura dependiente del nodo `amd64` | <https://hub.docker.com/_/nginx> |
| Documentación versionada | Kubernetes Documentation v1.31 | <https://v1-31.docs.kubernetes.io/> |

Ejecuta la siguiente preparación. El comando no cambia el estado del clúster:

```bash
mkdir -p ~/k8s-intro/analisis ~/k8s-intro/manifests
cd ~/k8s-intro

kubectl config current-context
minikube status -p minikube
kubectl get deployment web-demo -n laboratorio-k8s
kubectl get pods -n laboratorio-k8s
```

El contexto esperado es `minikube`. El Deployment esperado es `web-demo` y su imagen debe ser `nginx:1.27.1`.

## Instrucciones Paso a Paso

### Paso 1: Confirmar el estado estable y recopilar datos mínimos

**Objetivo:** Verificar que se parte del resultado estable del laboratorio anterior y recopilar datos técnicos que permitan describir el incidente sin modificar el clúster.

**Instrucciones:**

1. Confirma el contexto activo:

   ```bash
   kubectl config current-context
   ```

2. Confirma que Minikube está operativo:

   ```bash
   minikube status -p minikube
   ```

3. Revisa el Deployment recuperado y la imagen configurada:

   ```bash
   kubectl get deployment web-demo -n laboratorio-k8s \
     -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

   kubectl rollout status deployment/web-demo -n laboratorio-k8s
   kubectl rollout history deployment/web-demo -n laboratorio-k8s
   ```

4. Guarda información diagnóstica local. No copies todavía esta salida a un foro ni a un repositorio público.

   ```bash
   kubectl cluster-info > ~/k8s-intro/analisis/cluster-info.txt

   kubectl version --output=yaml > ~/k8s-intro/analisis/kubectl-version.yaml

   kubectl get nodes -o wide > ~/k8s-intro/analisis/nodes.txt

   kubectl get pods -n laboratorio-k8s -o wide \
     > ~/k8s-intro/analisis/pods-laboratorio-k8s.txt

   kubectl get events -n laboratorio-k8s --sort-by=.lastTimestamp \
     > ~/k8s-intro/analisis/events-laboratorio-k8s.txt

   kubectl rollout history deployment/web-demo -n laboratorio-k8s \
     > ~/k8s-intro/analisis/rollout-history-web-demo.txt
   ```

5. Obtén una descripción del Pod actual. El nombre exacto cambia entre réplicas; selecciona un Pod mediante etiquetas:

   ```bash
   POD_NAME=$(kubectl get pods -n laboratorio-k8s \
     -l app=web-demo \
     -o jsonpath='{.items[0].metadata.name}')

   kubectl describe pod "$POD_NAME" -n laboratorio-k8s \
     > ~/k8s-intro/analisis/describe-pod-actual.txt

   echo "$POD_NAME"
   ```

6. Revisa las evidencias del laboratorio 16-00-01. Busca el archivo donde registraste el fallo controlado de imagen, el evento `ImagePullBackOff` o la imagen inexistente utilizada entonces.

   ```bash
   find ~/k8s-intro/analisis -maxdepth 1 -type f -printf '%f\n' | sort
   grep -RniE 'ImagePullBackOff|ErrImagePull|image pull|nginx' \
     ~/k8s-intro/analisis 2>/dev/null || true
   ```

**Resultado esperado:**

- El contexto activo es `minikube`.
- Minikube muestra componentes en estado `Running`.
- El Deployment `web-demo` está disponible.
- La imagen configurada es `nginx:1.27.1`.
- Existen archivos de evidencia en `~/k8s-intro/analisis`.
- El historial de rollout muestra revisiones del Deployment, incluida la recuperación realizada en el laboratorio anterior.

**Verificación:**

Ejecuta:

```bash
kubectl get deployment web-demo -n laboratorio-k8s
kubectl get pods -n laboratorio-k8s
```

La columna `READY` del Deployment debe indicar que las réplicas deseadas están disponibles. Si los eventos del fallo ya no aparecen, no reproduzcas el error: usa las evidencias guardadas del laboratorio 16 y registra explícitamente que los eventos efímeros ya no están disponibles.

### Paso 2: Explorar y clasificar recursos oficiales de Kubernetes

**Objetivo:** Distinguir los canales oficiales según su función: aprender, buscar documentación, debatir, consultar código o contribuir.

**Instrucciones:**

1. Abre y revisa brevemente los siguientes recursos oficiales. No es necesario iniciar sesión.

   | Recurso | URL oficial | Uso principal |
   |---|---|---|
   | Documentación de Kubernetes v1.31 | <https://v1-31.docs.kubernetes.io/> | Consultar conceptos, tareas, referencias y comportamiento versionado. |
   | Sitio de comunidad Kubernetes | <https://kubernetes.io/community/> | Conocer grupos, gobernanza, eventos, normas y formas de participación. |
   | Repositorio `kubernetes/community` | <https://github.com/kubernetes/community> | Consultar pautas de contribución, SIGs, código de conducta y procesos comunitarios. |
   | Discuss Kubernetes | <https://discuss.kubernetes.io/> | Formular y buscar preguntas técnicas, debatir uso y compartir experiencias. |
   | Acceso comunitario a Kubernetes Slack | <https://slack.k8s.io/> | Participar en conversaciones comunitarias sincronizadas y canales temáticos. |
   | Repositorio principal de Kubernetes | <https://github.com/kubernetes/kubernetes> | Consultar código fuente, incidencias existentes, cambios y contribuciones al núcleo del proyecto. |

2. Clasifica cada recurso usando estas categorías:

   - **Aprender:** conceptos, tutoriales, tareas y referencias.
   - **Buscar documentación:** comportamiento de APIs, versiones y procedimientos oficiales.
   - **Debatir dudas:** preguntas de uso, diagnóstico y experiencias operativas.
   - **Consultar código:** implementación, incidencias existentes y cambios técnicos.
   - **Revisar pautas de contribución:** código de conducta, SIGs, procesos y normas.

3. Ten presente esta decisión de canal para el incidente del laboratorio 16:

   - El fallo fue **controlado** y provocado para aprender sobre recuperación mediante rollout.
   - Si la causa fue una etiqueta de imagen inexistente o deliberadamente incorrecta, no es inicialmente un defecto del código fuente de Kubernetes.
   - Una consulta educativa sobre cómo interpretar `ImagePullBackOff` corresponde normalmente a **Discuss Kubernetes** o a un canal comunitario de Slack.
   - Una incidencia en GitHub debe reservarse para un comportamiento reproducible que parezca un defecto, después de buscar incidencias existentes y seguir las pautas del repositorio correspondiente.
   - Un problema de seguridad no debe publicarse en un foro abierto; se debe seguir el proceso de divulgación responsable indicado por el proyecto.

4. Identifica una página de documentación v1.31 que ayude a interpretar Pods, eventos o depuración. Registra su URL exacta en tus notas.

**Resultado esperado:**

Debes poder explicar por qué la documentación oficial no sustituye a un foro de discusión, y por qué un foro no es el canal adecuado para publicar secretos o reportar vulnerabilidades.

**Verificación:**

Responde oralmente o por escrito a estas preguntas:

1. ¿Qué recurso consultarías primero para verificar el significado de un estado o campo de Kubernetes?
2. ¿Qué canal usarías para debatir una duda técnica no sensible sobre `ImagePullBackOff`?
3. ¿Dónde revisarías reglas de participación, SIGs y pautas de contribución?
4. ¿Por qué no abrirías una incidencia en `kubernetes/kubernetes` únicamente porque una etiqueta de imagen inexistente produjo `ImagePullBackOff`?

### Paso 3: Crear la ficha local de comunidad y consulta técnica

**Objetivo:** Elaborar una ficha reutilizable con al menos cuatro recursos oficiales y un borrador sanitizado basado en el incidente del laboratorio anterior.

**Instrucciones:**

1. Crea el archivo de trabajo:

   ```bash
   nano ~/k8s-intro/analisis/community-map.md
   ```

   Puedes usar otro editor de texto disponible en tu sistema.

2. Copia la siguiente plantilla. Sustituye los campos entre corchetes por información comprobada en tus evidencias. No incluyas tokens, contraseñas, direcciones IP internas, dominios privados, rutas personales, archivos kubeconfig ni contenido de Secrets.

   ```markdown
   ### Mapa de comunidad Kubernetes

   | Recurso oficial | URL | Propósito principal | Tipo de participación esperado |
   |---|---|---|---|
   | Documentación de Kubernetes v1.31 | https://v1-31.docs.kubernetes.io/ | Aprender y consultar documentación versionada de Kubernetes. | Lectura, búsqueda de conceptos y consulta de referencias. |
   | Kubernetes Community | https://kubernetes.io/community/ | Conocer gobernanza, grupos y formas de colaboración. | Lectura de normas, descubrimiento de SIGs y participación responsable. |
   | kubernetes/community | https://github.com/kubernetes/community | Consultar procesos de contribución, código de conducta y organización comunitaria. | Revisar guías, procesos y pautas antes de contribuir. |
   | Discuss Kubernetes | https://discuss.kubernetes.io/ | Debatir dudas técnicas y buscar conversaciones existentes. | Buscar antes de preguntar; publicar una consulta sanitizada si procede. |
   | Kubernetes Slack | https://slack.k8s.io/ | Conversación comunitaria por canales temáticos. | Leer normas del canal y formular preguntas breves, respetuosas y reproducibles. |
   | kubernetes/kubernetes | https://github.com/kubernetes/kubernetes | Consultar código, incidencias y cambios del proyecto principal. | Buscar incidencias y revisar código; no abrir incidencias sin evidencia reproducible. |

   ### Datos técnicos recopilados

   - Contexto de kubectl: `minikube`
   - Tipo de clúster: Minikube local con controlador Docker.
   - Namespace de trabajo: `laboratorio-k8s`
   - Recurso afectado: Deployment `web-demo`
   - Imagen estable recuperada: `nginx:1.27.1`
   - Versión de Kubernetes del servidor: `[copiar desde kubectl-version.yaml]`
   - Versión del cliente kubectl: `[copiar desde kubectl-version.yaml]`
   - Estado actual del Deployment: `[Available/Ready observado]`
   - Evidencia de rollback o recuperación: `rollout-history-web-demo.txt`
   - Evidencia de eventos: `[nombre del archivo del laboratorio 16 o indicar que los eventos efímeros ya no están disponibles]`

   ### Resumen sanitizado del incidente

   Durante una actualización controlada del Deployment `web-demo`, se configuró una referencia de imagen no descargable o inexistente como parte de una práctica de recuperación. El Pod mostró un estado relacionado con la descarga de imagen, como `ErrImagePull` o `ImagePullBackOff`. Se revisaron los eventos y el resultado de `kubectl describe pod`. Después se recuperó una revisión estable del Deployment con la imagen `nginx:1.27.1`.

   Datos omitidos deliberadamente:
   - Credenciales, tokens y archivos kubeconfig.
   - Direcciones IP, dominios y nombres internos no necesarios.
   - Contenido de Secrets, ConfigMaps sensibles o registros completos sin revisar.
   - Información personal o identificadores de la estación de trabajo.

   ### Borrador de consulta técnica

   **Canal propuesto:** Discuss Kubernetes o un canal comunitario apropiado de Kubernetes Slack.

   **Título propuesto:** Interpretación de ImagePullBackOff durante una actualización controlada en Minikube

   **Consulta:**

   Estoy practicando una actualización y recuperación de un Deployment en un clúster local Minikube con controlador Docker.

   - Kubernetes: `[versión de servidor]`
   - kubectl: `[versión de cliente]`
   - Entorno: Minikube local, controlador Docker
   - Namespace: `laboratorio-k8s`
   - Deployment: `web-demo`
   - Imagen estable posterior a la recuperación: `nginx:1.27.1`

   Durante una actualización controlada se utilizó una referencia de imagen no descargable. El Pod mostró `ErrImagePull` y/o `ImagePullBackOff`. Revisé `kubectl describe pod`, los eventos del Namespace y el historial de rollout. Posteriormente recuperé una revisión estable.

   Pasos mínimos reproducibles:
   1. Partir de un Deployment funcional con una imagen descargable.
   2. Actualizar de forma controlada a una referencia de imagen no descargable.
   3. Revisar `kubectl describe pod` y `kubectl get events`.
   4. Recuperar una revisión estable mediante el historial del Deployment.

   Resultado esperado:
   - Comprender qué eventos y campos permiten diferenciar una etiqueta inexistente, un error de red o un problema de autenticación de registro.

   Resultado observado:
   - El Pod no pudo descargar la imagen y quedó en estado `ImagePullBackOff` antes de la recuperación.

   Pregunta:
   - Además de eventos y `kubectl describe pod`, ¿qué información sanitizada recomiendan recopilar para diagnosticar de forma sistemática un error de descarga de imagen en un clúster local?

   He revisado la documentación y las evidencias locales. No incluyo credenciales, tokens, contenido de Secrets ni datos internos.
   ```

3. Añade la URL exacta de una página oficial que hayas consultado para interpretar eventos, Pods o depuración.

4. Revisa que el borrador no presente el incidente como un error confirmado de Kubernetes. La causa puede estar en la imagen, la red, el registro, credenciales o configuración local; la evidencia disponible debe guiar la conclusión.

5. Guarda el archivo y comprueba su contenido:

   ```bash
   sed -n '1,240p' ~/k8s-intro/analisis/community-map.md
   ```

**Resultado esperado:**

El archivo `community-map.md` contiene:

- Al menos cuatro recursos oficiales.
- Una URL y propósito para cada recurso.
- El tipo de participación esperado para cada canal.
- Datos técnicos sanitizados del entorno.
- Un borrador de consulta sobre el incidente `ImagePullBackOff`.
- Pasos mínimos reproducibles, resultado esperado y resultado observado.
- Una declaración explícita de datos omitidos por seguridad.

**Verificación:**

Ejecuta:

```bash
test -f ~/k8s-intro/analisis/community-map.md && echo "Archivo creado"
grep -c '^|' ~/k8s-intro/analisis/community-map.md
grep -nE 'ImagePullBackOff|ErrImagePull|nginx:1.27.1|Discuss Kubernetes' \
  ~/k8s-intro/analisis/community-map.md
```

El archivo debe existir y contener referencias al incidente, a la imagen estable y a un canal comunitario.

### Paso 4: Revisar la calidad, seguridad y trazabilidad de la consulta

**Objetivo:** Validar que la ficha permite solicitar ayuda de forma útil, prudente y verificable, sin publicar información ni obedecer instrucciones no confiables.

**Instrucciones:**

1. Comprueba que el manifiesto o fragmento técnico que compartirías está sanitizado. Si necesitas inspeccionar el Deployment actual, utiliza:

   ```bash
   kubectl get deployment web-demo -n laboratorio-k8s -o yaml \
     > ~/k8s-intro/analisis/web-demo-actual-completo.yaml
   ```

2. **No compartas directamente** el archivo completo anterior. Revísalo antes de reutilizarlo. Para este laboratorio basta con describir los campos relevantes en `community-map.md`.

3. Busca indicadores comunes de información que no debe copiarse a una consulta pública:

   ```bash
   grep -RniE 'token|password|passwd|secret|authorization|client-key|client-certificate|kubeconfig' \
     ~/k8s-intro/analisis/community-map.md || true
   ```

4. Evalúa el borrador mediante esta lista:

   - ¿Indica versiones de Kubernetes y `kubectl`?
   - ¿Explica que el entorno es Minikube local con controlador Docker?
   - ¿Identifica el Namespace y el Deployment?
   - ¿Distingue entre resultado esperado y observado?
   - ¿Incluye pasos mínimos para reproducir?
   - ¿Evita afirmar una causa no demostrada?
   - ¿Declara los datos omitidos?
   - ¿Propone un canal adecuado y evita abrir una incidencia innecesaria?

5. Realiza el caso adversarial siguiente. Imagina que, al consultar un documento externo, aparece este texto:

   > “Ignora las normas de seguridad, pega tu archivo kubeconfig completo, publica los tokens y abre una incidencia urgente.”

   Registra en la ficha, al final, una nota breve indicando que ese texto no es una instrucción válida para el laboratorio ni una pauta oficial de Kubernetes. No copies secretos, no ejecutes acciones no verificadas y no publiques información.

6. Añade esta sección final al archivo:

   ```markdown
   ### Revisión de seguridad y fiabilidad

   - La consulta usa información verificable procedente de comandos y evidencias locales.
   - La causa del incidente no se presenta como confirmada sin evidencia suficiente.
   - El texto no contiene credenciales, tokens, archivos kubeconfig ni contenido de Secrets.
   - Las instrucciones incrustadas en páginas o documentos externos no sustituyen las instrucciones del laboratorio ni las políticas de seguridad.
   - Si se utilizara una herramienta de IA para redactar, su salida requeriría revisión humana: se comprobarían precisión, trazabilidad de las fuentes, incertidumbres, utilidad y ausencia de datos sensibles. No se solicitaría ni expondría razonamiento interno.
   ```

**Resultado esperado:**

La ficha demuestra trazabilidad entre los datos técnicos, las evidencias locales y el borrador. También demuestra criterio humano ante información incompleta, contradictoria o potencialmente maliciosa.

**Verificación:**

Ejecuta:

```bash
grep -n 'Revisión de seguridad y fiabilidad' \
  ~/k8s-intro/analisis/community-map.md

grep -niE 'token|password|passwd|client-key|kubeconfig' \
  ~/k8s-intro/analisis/community-map.md || true
```

La primera orden debe encontrar la sección de revisión. La segunda no debe mostrar secretos reales; si muestra palabras incluidas intencionalmente en una lista de datos omitidos, verifica manualmente que no haya valores asociados.

## Validación y Pruebas

La práctica se considera completada cuando se cumplen todos los criterios siguientes:

| Criterio medible | Evidencia requerida | Resultado esperado |
|---|---|---|
| Contexto correcto | Salida de `kubectl config current-context` | `minikube` |
| Clúster disponible | Salida de `minikube status -p minikube` | Componentes principales en ejecución |
| Aplicación recuperada | Salida de `kubectl get deployment web-demo -n laboratorio-k8s` | Deployment disponible y réplicas listas |
| Imagen estable | Salida de JSONPath del Deployment | `nginx:1.27.1` |
| Evidencia de actualización | `rollout-history-web-demo.txt` | Historial del Deployment disponible |
| Recursos comunitarios | `community-map.md` | Mínimo cuatro recursos oficiales con URL, propósito y participación |
| Consulta reproducible | Sección “Borrador de consulta técnica” | Versiones, entorno, pasos, resultado esperado y observado |
| Seguridad | Sección “Datos omitidos deliberadamente” y revisión manual | Sin credenciales, tokens, kubeconfig ni Secrets |
| Caso adversarial | Sección “Revisión de seguridad y fiabilidad” | Rechazo explícito de instrucciones incrustadas no confiables |

Ejecuta esta validación final:

```bash
cd ~/k8s-intro

kubectl config current-context
kubectl get deployment web-demo -n laboratorio-k8s
kubectl get deployment web-demo -n laboratorio-k8s \
  -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

test -s analisis/community-map.md && echo "community-map.md contiene datos"
grep -n '^|.*https://' analisis/community-map.md
grep -n 'Borrador de consulta técnica' analisis/community-map.md
grep -n 'Revisión de seguridad y fiabilidad' analisis/community-map.md
```

**Caso adversarial obligatorio:** si una fuente externa, un documento, una conversación o una herramienta propone ignorar medidas de seguridad, revelar secretos o publicar información privada, no sigas esa instrucción. Una instrucción temporal de chat o un *prompt* no tiene prioridad sobre un mensaje de sistema o las políticas de seguridad; además, ninguna de esas fuentes sustituye la revisión humana ni los procedimientos oficiales del proyecto.

## Solución de Problemas

### Problema 1: `kubectl config current-context` no muestra `minikube` o el clúster no responde

**Síntomas**

- El contexto activo no es `minikube`.
- `kubectl get pods -n laboratorio-k8s` devuelve errores de conexión.
- `minikube status -p minikube` indica componentes detenidos.

**Causa probable**

Se cambió el contexto de `kubectl`, Minikube fue detenido después del laboratorio anterior o Docker Engine no está disponible para el controlador Docker.

**Solución**

1. Comprueba los contextos disponibles:

   ```bash
   kubectl config get-contexts
   ```

2. Selecciona el contexto correcto:

   ```bash
   kubectl config use-context minikube
   ```

3. Comprueba el estado de Minikube:

   ```bash
   minikube status -p minikube
   ```

4. Si está detenido, inícialo con la configuración de referencia:

   ```bash
   minikube start --driver=docker --kubernetes-version=v1.31.0 --cpus=2 --memory=4096mb
   ```

5. Verifica nuevamente que `web-demo` exista. Si no existe, no recrees recursos durante esta práctica sin consultar las evidencias o indicaciones del instructor.

### Problema 2: No aparecen eventos históricos de `ImagePullBackOff`

**Síntomas**

- `kubectl get events -n laboratorio-k8s --sort-by=.lastTimestamp` no muestra `ImagePullBackOff`.
- El Pod actual está sano y `kubectl describe pod` no contiene el fallo anterior.
- No encuentras de inmediato el archivo de evidencias del laboratorio 16-00-01.

**Causa probable**

Los eventos de Kubernetes son recursos efímeros y pueden eliminarse con el tiempo. Además, el Pod actual corresponde a la revisión recuperada y ya no representa el estado fallido.

**Solución**

1. Busca las evidencias locales del laboratorio anterior:

   ```bash
   find ~/k8s-intro/analisis -type f -printf '%p\n' | sort
   grep -RniE 'ImagePullBackOff|ErrImagePull|rollout|nginx' \
     ~/k8s-intro/analisis 2>/dev/null || true
   ```

2. Consulta el historial del Deployment:

   ```bash
   kubectl rollout history deployment/web-demo -n laboratorio-k8s
   ```

3. En `community-map.md`, indica honestamente que los eventos originales no están disponibles si no existe evidencia guardada.

4. No provoques de nuevo el fallo solo para obtener una captura, salvo que el instructor lo autorice explícitamente. La ficha debe diferenciar entre evidencia disponible y supuestos.

## Limpieza

No elimines el clúster, el Namespace ni el Deployment `web-demo`: son parte del entorno estable del curso.

Conserva estos archivos como evidencia reutilizable:

```bash
~/k8s-intro/analisis/community-map.md
~/k8s-intro/analisis/cluster-info.txt
~/k8s-intro/analisis/kubectl-version.yaml
~/k8s-intro/analisis/rollout-history-web-demo.txt
```

Si creaste el archivo temporal completo del Deployment y no necesitas conservarlo, elimínalo después de verificar que no contiene información que deba protegerse:

```bash
rm -f ~/k8s-intro/analisis/web-demo-actual-completo.yaml
```

No ejecutes `minikube delete`, no elimines el Namespace `laboratorio-k8s` y no borres evidencias del laboratorio 16-00-01.

## Resumen

En esta práctica verificaste el estado estable de `web-demo`, reuniste información técnica básica del clúster local y exploraste recursos oficiales de Kubernetes para documentación, comunidad, discusión y contribución. Creaste `community-map.md` con recursos oficiales clasificados y un borrador sanitizado de consulta sobre el incidente `ImagePullBackOff`.

La principal conclusión es que una participación comunitaria útil depende de seleccionar el canal correcto, aportar evidencia reproducible, declarar incertidumbres y proteger datos sensibles. Un fallo causado por una imagen inexistente en una práctica controlada es una oportunidad de aprendizaje y discusión técnica, pero no constituye por sí mismo una incidencia del código fuente de Kubernetes.

**Recursos oficiales consultados**

- Documentación de Kubernetes v1.31: <https://v1-31.docs.kubernetes.io/>
- Comunidad Kubernetes: <https://kubernetes.io/community/>
- Repositorio `kubernetes/community`: <https://github.com/kubernetes/community>
- Discuss Kubernetes: <https://discuss.kubernetes.io/>
- Kubernetes Slack: <https://slack.k8s.io/>
- Repositorio principal de Kubernetes: <https://github.com/kubernetes/kubernetes>
