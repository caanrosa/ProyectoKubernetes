# Proyecto Kubernetes 202630 — Jobs parametrizados desde línea de comandos

## 1. Descripción
Este programa es un gestor de tareas para Kubernetes, implementado en Python. Muestra un menú en la consola donde el usuario elige el lenguaje, la tarea y la complejidad, y crea un Job en el clúster. Permite:

- Crear un Job con los argumentos y recursos de la tarea elegida.
- Revisar el estado de todos los Jobs del namespace estudiantes-202630.
- Ver los logs de un Job.
- Limpiar los Jobs que ya terminaron.

El recorrido completo es:

`gestor_jobs.py -> Job de Kubernetes -> Pod -> contenedor con argumentos -> logs`

Video de demostración: (pendiente el link del video)

Integrantes: Carlos Rojas, Sebastián Mercado, Cesar Jarava

## 2. Requisitos

- Windows con PowerShell
- Docker Desktop
- Minikube y kubectl
- Python 3 y la librería kubernetes.

## 3. Preparación

Ejecutar desde la carpeta raíz del proyecto, con Docker Desktop abierto:

```powershell
minikube start --driver=docker --container-runtime=docker
kubectl apply -f .\k8s\00-namespace.yaml
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\preparar-imagenes.ps1
pip install -r .\entrega\requirements.txt
```

Si se cambia el programa de un contenedor, hay que volver a ejecutar "preparar-imagenes.ps1". 

## 4. Estructura del proyecto

- catalogo/tareas.json: lenguajes, imágenes y tareas disponibles.
- entrega/gestor_jobs.py: el código del menú.
- imagenes/: los códigos y Dockerfiles de las imágenes de Python, Java y C.
- k8s/00-namespace.yaml: el namespace donde se crean los Jobs.
- scripts/preparar-imagenes.ps1: creación de las imágenes dentro de Minikube.

## 5. Ejecución

1. Abrir Docker Desktop y comprobar que Minikube está activo con "minikube status".
2. Desde la raíz del proyecto, ejecutar:

```powershell
python .\entrega\gestor_jobs.py
```

3. Elegir una opción del menú:

```
=== Gestor de Jobs - namespace estudiantes-202630 ===
  1. Ejecutar una tarea nueva
  2. Revisar el estado de las tareas
  3. Ver los logs de una tarea
  4. Limpiar las tareas terminadas
  5. Salir
```

- **Opción 1:** pide lenguaje, tarea y complejidad, y crea el Job.
- **Opción 2:** lista los Jobs con su estado: en ejecucion, completado, fallido o pendiente.
- **Opción 3:** muestra la salida del contenedor del Job elegido.
- **Opción 4:** borra los Jobs terminados (completados o fallidos) tras pedir confirmación. No toca los que siguen corriendo.

Para comparar con el clúster:

```powershell
kubectl get jobs,pods -n estudiantes-202630
kubectl logs -n estudiantes-202630 job/<nombre-del-job>
```

## 6. Cómo funciona el gestor

El Job se crea con el cliente de Python. Este es contenido del archivo YAML:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: <nombre>
  namespace: estudiantes-202630
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: tarea
          image: <imagen>
          imagePullPolicy: Never
          args: [<tarea>, <N>]
          resources:
            requests: { cpu: <cpu_request>, memory: <mem_request> }
            limits:   { cpu: <cpu_limit>,   memory: <mem_limit> }
```

El YAML lo que dice realmente es:

- `image_pull_policy="Never"`: las imágenes son locales y no existen en ningún registro.
- `restart_policy="Never"` y `backoff_limit=0`: si el Pod falla, no se reintenta y el Job queda como fallido.
- **Estado:** se calcula con los contadores `active`, `succeeded` y `failed` del Job.
- **Logs:** pertenecen al Pod, no al Job. El Pod se encuentra con la etiqueta `job-name=<nombre del Job>`.
- **Limpieza:** se borra con `propagation_policy="Background"`. Sin esa opción, el Job desaparece pero su Pod queda huérfano.

## 7. Complejidad

La complejidad fija los recursos del Job y el tamaño del trabajo (el valor de N):

| Nivel | CPU (request - limit) | Memoria (request - limit) |
|---|---|---|
| baja | 100m - 500m | 64Mi - 128Mi |
| media | 250m - 1000m | 128Mi - 512Mi |
| alta | 500m - 2000m | 256Mi - 1Gi |

| Tarea | baja | media | alta |
|---|---|---|---|
| `hola` | 0 | 0 | 0 |
| `ordenar` | 100000 | 1000000 | 3000000 |
| `fib` | 25 | 30 | 35 |
| `matriz` | 60 | 120 | 250 |
| `primos` | 20000 | 200000 | 1000000 |

`requests` es lo que Kubernetes reserva y `limits` es el tope. Kubernetes exige que `request` sea menor o igual que `limit`; de lo contrario rechaza el Job con un error 422.

## 8. Tarea adicional: `primos`

Cuenta cuántos números primos hay hasta N, por división de prueba y sin criba, para saturar la CPU. Está implementada en los tres lenguajes (Python, Java y C) con el mismo algoritmo, así que se puede comparar el tiempo de la misma tarea entre ellos. Para agregarla se hizo lo siguiente:

1. Se añadió la función `primos` en el programa de cada contenedor: `imagenes/python/tareas.py` (registrada en `TAREAS`), `imagenes/java/Tareas.java` (nuevo `case "primos"`) e `imagenes/c/tareas.c` (nuevo bloque `else if`).
2. Se reconstruyeron las imágenes con `.\scripts\preparar-imagenes.ps1`.
3. Se añadió `"primos"` a las tareas de `python`, `java` y `c` en `catalogo/tareas.json`.
4. Se añadió la fila de `primos` en la tabla `TAMANOS` del gestor.

Las tres imágenes terminan con el mismo contrato: una última línea JSON con `n`, `cuenta` y `ultimo` (el mayor primo encontrado).

Resultados esperados (sabemos la cantidad de primos):

| Complejidad | N | `cuenta` |
|---|---|---|
| baja | 20000 | 2262 |
| media | 200000 | 17984 |
| alta | 1000000 | 78498 |

## 9. OOMKilled

Si un contenedor supera su límite de memoria, Kubernetes lo mata con `OOMKilled`. Usualmente, no sucedía, por lo que una manera de causarlo es bajar temporalmente la complejidad `alta` a `32Mi` - `64Mi` y se ejecutó `python / ordenar / alta`. El gestor lee el motivo de terminación del contenedor y lo muestra:

```
Tareas en estudiantes-202630:
  - python-ordenar-alta-1791659060: fallido (OOMKilled)
```

Se confirmó con `kubectl describe pod -n estudiantes-202630`, que muestra `Reason: OOMKilled`. Después se restauró el nivel `alta` a `256Mi` - `1Gi`. Pero es solo una manera de provocarlo, y comprobar que sí funciona.
