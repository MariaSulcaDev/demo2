# 🚀 GUÍA COMPLETA - DESPLIEGUE EN KUBERNETES CON ARGO CD

**Alumna:** María Sulca
**Fecha:** 25 de Noviembre 2025
**Proyecto:** Microservicio de Usuarios + Frontend React

---

## 📋 TABLA DE CONTENIDO

1. [Arquitectura del Proyecto](#arquitectura)
2. [Explicación de Archivos Kubernetes](#archivos-kubernetes)
3. [Comandos de Despliegue](#comandos-despliegue)
4. [Validaciones](#validaciones)
5. [RETO: Argo CD + ConfigMap/Secrets + Probes](#reto)

---

## 🏗️ ARQUITECTURA DEL PROYECTO <a name="arquitectura"></a>

### Componentes Desplegados

```
┌─────────────────────────────────────────────────────┐
│                   KUBERNETES CLUSTER                 │
│                                                      │
│  ┌──────────────────┐       ┌──────────────────┐   │
│  │   FRONTEND       │       │   BACKEND        │   │
│  │   (React+Nginx)  │──────▶│   (Spring Boot)  │   │
│  │   Port: 80       │       │   Port: 9083     │   │
│  │   Replicas: 2    │       │   Replicas: 2    │   │
│  └──────────────────┘       └──────────────────┘   │
│         │                           │               │
│         │                           │               │
│  ┌──────▼───────┐           ┌──────▼───────┐       │
│  │  NodePort    │           │  NodePort    │       │
│  │  30081       │           │  30083       │       │
│  └──────────────┘           └──────────────┘       │
│         │                           │               │
│         └───────────┬───────────────┘               │
│                     │                               │
│              ┌──────▼──────┐                        │
│              │  ConfigMap  │                        │
│              │  + Secret   │                        │
│              └─────────────┘                        │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
              MongoDB Atlas (Externo)
```

### Tecnologías Utilizadas

- **Backend:** Spring Boot 3.5.6 (WebFlux) + MongoDB
- **Frontend:** React 19 + Vite + Nginx
- **Orquestación:** Kubernetes (Minikube)
- **Registry:** Docker Hub
- **CI/CD:** Argo CD (RETO)

---

## 📁 EXPLICACIÓN DE ARCHIVOS KUBERNETES <a name="archivos-kubernetes"></a>

### 1. ConfigMap (`12-maria-sulca-configmap.yml`)

**¿Qué es?**
Un ConfigMap almacena datos de configuración en formato clave-valor que pueden ser inyectados a los pods como variables de entorno.

**¿Para qué sirve?**
Separar la configuración del código, permitiendo cambiar parámetros sin reconstruir las imágenes Docker.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
     name: 12-maria-sulca-config
data:
     SPRING_PROFILES_ACTIVE: "dev"              # Perfil de Spring Boot
     SPRING_APPLICATION_NAME: "vg-ms-users-management"  # Nombre de la app
     SERVER_PORT: "9083"                        # Puerto del servidor
```

**Conceptos clave:**

- `apiVersion: v1` - Versión de la API de Kubernetes para ConfigMaps
- `kind: ConfigMap` - Tipo de recurso
- `metadata.name` - Nombre único del ConfigMap
- `data` - Pares clave-valor de configuración

---

### 2. Secret (`12-maria-sulca-secret.yml`)

**¿Qué es?**
Similar a ConfigMap pero diseñado para datos sensibles (contraseñas, tokens, claves API). Los valores están codificados en base64.

**¿Para qué sirve?**
Almacenar credenciales de forma segura sin exponerlas en código plano.

```yaml
apiVersion: v1
kind: Secret
metadata:
     name: 12-maria-sulca-secret
type: Opaque
data:
     # Valores en base64 (admin -> YWRtaW4=)
     BASIC_AUTH_USER: YWRtaW4=
     BASIC_AUTH_PASSWORD: cGFzc3dvcmQ=
```

**Conceptos clave:**

- `type: Opaque` - Tipo de Secret (genérico)
- `data` - Valores codificados en base64
- Se inyectan como variables de entorno o archivos montados

**¿Cómo generar base64?**

```bash
echo -n "admin" | base64
# Resultado: YWRtaW4=
```

---

### 3. Deployment Backend (`12-maria-sulca-users-deployment.yml`)

**¿Qué es?**
Define cómo desplegar y gestionar las réplicas de una aplicación.

**¿Para qué sirve?**
Garantizar alta disponibilidad, auto-recuperación y actualizaciones sin downtime.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
     name: 12-maria-sulca-users-deployment
     labels:
          app: 12-maria-sulca-users
spec:
     replicas: 2                    # 2 instancias del pod
     selector:
          matchLabels:
               app: 12-maria-sulca-users
     template:
          metadata:
               labels:
                    app: 12-maria-sulca-users
          spec:
               containers:
                    - name: 12-maria-sulca-users
                      image: mariasb/12-maria-sulca-users:latest
                      imagePullPolicy: IfNotPresent
                      ports:
                           - containerPort: 9083

                      # INYECCIÓN DE CONFIGMAP Y SECRET
                      envFrom:
                           - configMapRef:
                                  name: 12-maria-sulca-config
                           - secretRef:
                                  name: 12-maria-sulca-secret

                      env:
                           - name: SERVER_PORT
                             value: "9083"
                           - name: SPRING_APPLICATION_NAME
                             value: "vg-ms-users-management"

                      # LIVENESS PROBE (¿Está vivo el contenedor?)
                      livenessProbe:
                           tcpSocket:
                                port: 9083
                           initialDelaySeconds: 60    # Espera 60s antes de la 1ra prueba
                           periodSeconds: 20          # Prueba cada 20s
                           failureThreshold: 3        # Reinicia tras 3 fallos

                      # READINESS PROBE (¿Está listo para recibir tráfico?)
                      readinessProbe:
                           tcpSocket:
                                port: 9083
                           initialDelaySeconds: 45
                           periodSeconds: 10
                           failureThreshold: 3
```

**Conceptos clave:**

#### **Replicas**

- `replicas: 2` - Kubernetes mantiene siempre 2 pods corriendo
- Si un pod falla, Kubernetes crea uno nuevo automáticamente

#### **ImagePullPolicy**

- `IfNotPresent` - Usa imagen local si existe, si no la descarga
- `Always` - Siempre descarga la última versión

#### **EnvFrom vs Env**

- `envFrom` - Inyecta **todas** las variables del ConfigMap/Secret
- `env` - Inyecta variables individuales

#### **Liveness Probe (Prueba de Vida)**

- **¿Qué hace?** Verifica si el contenedor está "vivo"
- **¿Cuándo reinicia?** Si falla 3 veces consecutivas
- **Tipos:**
  - `httpGet` - Hace petición HTTP (ej: `/actuator/health`)
  - `tcpSocket` - Verifica si el puerto acepta conexiones
  - `exec` - Ejecuta un comando

#### **Readiness Probe (Prueba de Preparación)**

- **¿Qué hace?** Verifica si el contenedor está listo para recibir tráfico
- **¿Cuándo se usa?** Durante el inicio o actualizaciones
- **Diferencia con Liveness:**
  - Liveness → Reinicia el pod
  - Readiness → Detiene el tráfico temporalmente

---

### 4. Service Backend (`12-maria-sulca-users-service.yml`)

**¿Qué es?**
Expone los pods de un Deployment para que sean accesibles (dentro o fuera del cluster).

**¿Para qué sirve?**
Proporciona una IP estable y balanceo de carga entre las réplicas.

```yaml
apiVersion: v1
kind: Service
metadata:
     name: maria-sulca-users-service
     labels:
          app: 12-maria-sulca-users
spec:
     type: NodePort                # Accesible desde fuera del cluster
     selector:
          app: 12-maria-sulca-users  # Selecciona pods con este label
     ports:
          - protocol: TCP
            port: 9083              # Puerto interno del Service
            targetPort: 9083        # Puerto del contenedor
            nodePort: 30083         # Puerto externo (30000-32767)
```

**Conceptos clave:**

#### **Tipos de Service**

1. **ClusterIP** (default) - Solo accesible dentro del cluster
2. **NodePort** - Accesible desde fuera en `<NodeIP>:<NodePort>`
3. **LoadBalancer** - Crea un balanceador en cloud (AWS ELB, GCP LB)

#### **Ports**

- `port` - Puerto del Service (usado por otros pods)
- `targetPort` - Puerto del contenedor
- `nodePort` - Puerto expuesto en el nodo (30000-32767)

#### **Selector**

- Conecta el Service con los pods que tengan el label `app: 12-maria-sulca-users`

---

### 5. Deployment Frontend (`12-maria-sulca-front-deployment.yml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
     name: 12-maria-sulca-front-deployment
     labels:
          app: 12-maria-sulca-front
spec:
     replicas: 2
     selector:
          matchLabels:
               app: 12-maria-sulca-front
     template:
          metadata:
               labels:
                    app: 12-maria-sulca-front
          spec:
               containers:
                    - name: 12-maria-sulca-front
                      image: mariasb/12-maria-sulca-front:latest
                      imagePullPolicy: Always
                      ports:
                           - containerPort: 80

                      # Variables para conexión al backend
                      env:
                           - name: VITE_API_USERS_URL
                             value: "http://maria-sulca-users-service:9083"
                           - name: VITE_API_BASE_URL
                             value: "http://maria-sulca-users-service:9083/api/v1"

                      livenessProbe:
                           httpGet:
                                path: /          # Nginx raíz
                                port: 80
                           initialDelaySeconds: 15
                           periodSeconds: 10
                           failureThreshold: 3

                      readinessProbe:
                           httpGet:
                                path: /
                                port: 80
                           initialDelaySeconds: 10
                           periodSeconds: 5
                           failureThreshold: 3
```

**Diferencias con Backend:**

- Usa `httpGet` porque Nginx responde HTTP
- `imagePullPolicy: Always` para forzar actualización de imagen

---

### 6. Service Frontend (`12-maria-sulca-front-service.yml`)

```yaml
apiVersion: v1
kind: Service
metadata:
     name: maria-sulca-front-service
     labels:
          app: 12-maria-sulca-front
spec:
     type: NodePort
     selector:
          app: 12-maria-sulca-front
     ports:
          - protocol: TCP
            port: 80
            targetPort: 80
            nodePort: 30081
```

---

## 🚀 COMANDOS DE DESPLIEGUE <a name="comandos-despliegue"></a>

### PASO 1: Construir Imágenes Docker

```bash
# 1. Backend (Spring Boot)
cd /home/javidev/Escritorio/Demos/vg-ms-users-management-develop
docker build -t mariasb/12-maria-sulca-users:latest .
docker push mariasb/12-maria-sulca-users:latest

# 2. Frontend (React + Nginx)
cd /home/javidev/Escritorio/Demos/vg-web-sigei-feature-ms-teacher-assignment
docker build -t mariasb/12-maria-sulca-front:latest .
docker push mariasb/12-maria-sulca-front:latest

# 3. Verificar imágenes
docker images | grep 12-maria-sulca
```

**Explicación:**

- `docker build -t` → Crea imagen con tag
- `docker push` → Sube imagen a Docker Hub
- Requiere `docker login` previo

---

### PASO 2: Aplicar Manifiestos en Kubernetes

```bash
cd /home/javidev/Escritorio/Demos/k8s

# Orden correcto (ConfigMap/Secret primero)
kubectl apply -f 12-maria-sulca-configmap.yml
kubectl apply -f 12-maria-sulca-secret.yml
kubectl apply -f 12-maria-sulca-users-deployment.yml
kubectl apply -f 12-maria-sulca-users-service.yml
kubectl apply -f 12-maria-sulca-front-deployment.yml
kubectl apply -f 12-maria-sulca-front-service.yml
```

**¿Por qué este orden?**

- ConfigMap y Secret deben existir antes que los Deployments los referencien

---

### PASO 3: Validaciones <a name="validaciones"></a>

```bash
# 1. Ver PODS (deben estar Running y READY 1/1)
kubectl get pods
# Salida esperada:
# 12-maria-sulca-front-deployment-xxx   1/1     Running   0          2m
# 12-maria-sulca-users-deployment-xxx   1/1     Running   0          2m

# 2. Ver DEPLOYMENTS (READY debe ser 2/2)
kubectl get deployments
# Salida esperada:
# NAME                              READY   UP-TO-DATE   AVAILABLE
# 12-maria-sulca-front-deployment   2/2     2            2
# 12-maria-sulca-users-deployment   2/2     2            2

# 3. Ver SERVICES (verificar puertos)
kubectl get services
# Salida esperada:
# maria-sulca-front-service   NodePort   10.x.x.x   <none>   80:30081/TCP
# maria-sulca-users-service   NodePort   10.x.x.x   <none>   9083:30083/TCP

# 4. Ver PODS en detalle (revisar RESTARTS)
kubectl get pods -o wide

# 5. Ver logs de un pod
kubectl logs <nombre-pod>

# 6. Describir pod (ver eventos y errores)
kubectl describe pod <nombre-pod>
```

---

### PASO 4: Acceder desde Navegador

```bash
# Obtener IP del nodo
kubectl get nodes -o wide
# IP de minikube: 192.168.49.2

# O usar:
minikube ip
```

**URLs de acceso:**

- **Frontend:** `http://192.168.49.2:30081`
- **Backend (Swagger):** `http://192.168.49.2:30083/swagger-ui/index.html`
- **Backend (API):** `http://192.168.49.2:30083/api/v1/users`

---

## 🏆 RETO: ARGO CD + CONFIGMAP/SECRETS + PROBES <a name="reto"></a>

### PARTE 1: Instalar Argo CD

#### 1.1 Crear namespace e instalar

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que esté listo (2-3 minutos)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Verificar pods de Argo CD
kubectl get pods -n argocd
```

**¿Qué es Argo CD?**

- Herramienta de CI/CD declarativa para Kubernetes
- Sincroniza automáticamente manifiestos de Git con el cluster
- Interfaz visual para gestionar despliegues

---

#### 1.2 Acceder a la UI de Argo CD

```bash
# Opción A: Port-forward (acceso local)
kubectl -n argocd port-forward svc/argocd-server 8080:443 &

# Opción B: Exponer con NodePort (para Minikube)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Ver puerto asignado
kubectl get svc argocd-server -n argocd
```

**Acceso:**

- Port-forward: `https://localhost:8080`
- NodePort: `https://<MINIKUBE_IP>:<NODE_PORT>`

---

#### 1.3 Obtener credenciales

```bash
# Usuario: admin
# Password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

**Login:**

1. Abre el navegador en la URL de Argo CD
2. Usuario: `admin`
3. Contraseña: (output del comando anterior)
4. Acepta el certificado autofirmado

---

### PARTE 2: Configurar Aplicación en Argo CD

#### 2.1 Crear Application manifest

```bash
# Crear archivo para Argo CD Application
cat > /home/javidev/Escritorio/Demos/k8s/argocd-application.yml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: 12-maria-sulca-app
  namespace: argocd
spec:
  project: default

  # Fuente: carpeta con manifiestos k8s
  source:
    repoURL: https://github.com/TU-USUARIO/TU-REPO.git  # ⚠️ CAMBIAR
    targetRevision: main
    path: k8s

  # Destino: namespace default
  destination:
    server: https://kubernetes.default.svc
    namespace: default

  # Sincronización automática
  syncPolicy:
    automated:
      prune: true      # Elimina recursos que ya no están en Git
      selfHeal: true   # Auto-corrige si hay cambios manuales
    syncOptions:
      - CreateNamespace=true
EOF
```

**Conceptos:**

- `source.repoURL` - Repositorio Git con los manifiestos
- `source.path` - Carpeta dentro del repo
- `syncPolicy.automated` - Sincronización automática
- `prune` - Elimina recursos huérfanos
- `selfHeal` - Revierte cambios manuales

---

#### 2.2 Subir manifiestos a Git (IMPORTANTE)

```bash
cd /home/javidev/Escritorio/Demos

# Inicializar Git (si no existe)
git init

# Agregar archivos
git add k8s/

# Commit
git commit -m "feat: Add Kubernetes manifests for users microservice"

# Crear repo en GitHub y pushear
git remote add origin https://github.com/TU-USUARIO/TU-REPO.git
git branch -M main
git push -u origin main
```

---

#### 2.3 Aplicar Application en Argo CD

```bash
# Editar argocd-application.yml con tu repo
nano /home/javidev/Escritorio/Demos/k8s/argocd-application.yml

# Aplicar
kubectl apply -f /home/javidev/Escritorio/Demos/k8s/argocd-application.yml

# Verificar
kubectl get applications -n argocd
```

---

#### 2.4 Sincronizar desde la UI

1. Abre Argo CD UI
2. Verás la aplicación `12-maria-sulca-app`
3. Clic en **SYNC** → **SYNCHRONIZE**
4. Espera a que todos los recursos estén en estado **Healthy**

**Vista en Argo CD:**

```
┌──────────────────────────────────┐
│   12-maria-sulca-app             │
│   Status: Synced, Healthy        │
├──────────────────────────────────┤
│   ├─ ConfigMap                   │
│   ├─ Secret                      │
│   ├─ Deployment (users) [2/2]    │
│   ├─ Service (users)             │
│   ├─ Deployment (front) [2/2]    │
│   └─ Service (front)             │
└──────────────────────────────────┘
```

---

### PARTE 3: Demostrar ConfigMap y Secrets

#### 3.1 Ver ConfigMap en un Pod

```bash
# Entrar a un pod
kubectl exec -it <nombre-pod-users> -- sh

# Ver variables de entorno del ConfigMap
env | grep SPRING
# Salida:
# SPRING_PROFILES_ACTIVE=dev
# SPRING_APPLICATION_NAME=vg-ms-users-management

# Ver variables del Secret
env | grep BASIC
# Salida:
# BASIC_AUTH_USER=admin
# BASIC_AUTH_PASSWORD=password

exit
```

---

#### 3.2 Modificar ConfigMap y ver auto-sync

```bash
# Editar ConfigMap
kubectl edit configmap 12-maria-sulca-config

# Cambiar valor (ej: SERVER_PORT: "9084")
# Guardar y salir

# Reiniciar deployment para que tome cambios
kubectl rollout restart deployment/12-maria-sulca-users-deployment

# Verificar en Argo CD: detectará el cambio y lo revertirá (selfHeal)
```

**Esto demuestra:**

- ✅ GitOps: Git es la única fuente de verdad
- ✅ Self-healing: Argo CD revierte cambios manuales

---

### PARTE 4: Demostrar Liveness y Readiness Probes

#### 4.1 Ver estado de probes

```bash
# Ver detalles del pod
kubectl describe pod <nombre-pod> | grep -A 10 "Liveness\|Readiness"

# Salida:
#   Liveness:   tcp-socket :9083 delay=60s timeout=1s period=20s #success=1 #failure=3
#   Readiness:  tcp-socket :9083 delay=45s timeout=1s period=10s #success=1 #failure=3
```

---

#### 4.2 Simular fallo de Liveness Probe

```bash
# Opción 1: Detener el proceso dentro del contenedor
kubectl exec -it <nombre-pod-users> -- sh
kill 1  # Mata el proceso principal (PID 1)
exit

# Opción 2: Bloquear el puerto
kubectl exec -it <nombre-pod-users> -- sh
# (Si tuvieras iptables, bloquear puerto 9083)

# Observar reinicio automático
kubectl get pods -w
# Verás RESTARTS incrementarse
```

**Qué sucede:**

1. Liveness probe falla 3 veces consecutivas
2. Kubernetes reinicia el contenedor
3. El pod vuelve a estado Running

---

#### 4.3 Simular fallo de Readiness Probe

```bash
# Durante actualización rolling
kubectl set image deployment/12-maria-sulca-users-deployment \
  12-maria-sulca-users=mariasb/12-maria-sulca-users:v2

# Observar (pods nuevos esperan readiness antes de recibir tráfico)
kubectl get pods -w
```

**Qué sucede:**

1. Se crean nuevos pods
2. Readiness probe verifica si están listos
3. Solo cuando pasan readiness, el Service envía tráfico
4. Pods antiguos se terminan gradualmente (zero-downtime)

---

### PARTE 5: Comandos de Validación Final

```bash
# 1. Verificar todo está sincronizado en Argo CD
kubectl get applications -n argocd

# 2. Ver recursos gestionados por Argo CD
kubectl get all -l app.kubernetes.io/instance=12-maria-sulca-app

# 3. Ver ConfigMaps
kubectl get configmaps
kubectl describe configmap 12-maria-sulca-config

# 4. Ver Secrets
kubectl get secrets
kubectl describe secret 12-maria-sulca-secret

# 5. Ver eventos de probes
kubectl get events --sort-by=.metadata.creationTimestamp | grep probe

# 6. Logs de Argo CD
kubectl logs -n argocd deployment/argocd-application-controller
```

---

## 📊 CHECKLIST PARA VIDEO

### ✅ Preparación

- [ ] Cluster Kubernetes corriendo (minikube status)
- [ ] Docker Hub login (docker login)
- [ ] Imágenes pusheadas a Docker Hub
- [ ] Git repo con manifiestos k8s

### ✅ Demostración Básica

- [ ] Mostrar archivos YAML y explicar cada sección
- [ ] Ejecutar `kubectl apply` en orden
- [ ] Mostrar `kubectl get pods` (2/2 READY)
- [ ] Mostrar `kubectl get deployments` (READY 2/2)
- [ ] Mostrar `kubectl get services` (NodePort)
- [ ] Abrir navegador y acceder a frontend (30081)
- [ ] Abrir navegador y acceder a backend Swagger (30083)
- [ ] Mostrar DevTools → Network → petición front→back exitosa

### ✅ RETO: Argo CD

- [ ] Instalar Argo CD (`kubectl apply -f`)
- [ ] Acceder a UI de Argo CD (login exitoso)
- [ ] Crear Application apuntando a Git repo
- [ ] Sincronizar aplicación (SYNC button)
- [ ] Mostrar vista de recursos en Argo CD

### ✅ RETO: ConfigMap y Secrets

- [ ] Mostrar ConfigMap en UI de Kubernetes Dashboard o CLI
- [ ] `kubectl exec` en pod → ver variables de entorno
- [ ] Modificar ConfigMap manualmente
- [ ] Argo CD detecta drift y revierte (selfHeal)

### ✅ RETO: Liveness y Readiness

- [ ] `kubectl describe pod` → mostrar configuración de probes
- [ ] Simular fallo → mostrar reinicio automático
- [ ] `kubectl get pods -w` → observar RESTARTS incrementarse
- [ ] Explicar diferencia Liveness vs Readiness

---

## 🎬 GUIÓN SUGERIDO PARA VIDEO

### Introducción (1 min)

```
"Hola, soy María Sulca. Hoy voy a desplegar un microservicio completo en
Kubernetes con configuración profesional usando ConfigMaps, Secrets, probes
de salud y Argo CD para GitOps."
```

### Parte 1: Arquitectura (2 min)

```
"Tengo dos componentes:
1. Backend en Spring Boot (puerto 9083) con 2 réplicas
2. Frontend en React con Nginx (puerto 80) con 2 réplicas

Ambos usan ConfigMap para configuración y Secret para credenciales.
Los servicios se exponen con NodePort para acceso externo."
```

### Parte 2: Archivos YAML (5 min)

```
"Voy a explicar cada archivo:

ConfigMap: almacena variables como SPRING_PROFILES_ACTIVE...
Secret: credenciales en base64...
Deployment: define réplicas, probes de salud...
Service: expone los pods con balanceo de carga..."

[Mostrar cada archivo en editor]
```

### Parte 3: Despliegue (3 min)

```
"Primero aplico ConfigMap y Secret, luego los Deployments..."

[Ejecutar comandos kubectl apply]
[Mostrar kubectl get pods, deployments, services]
```

### Parte 4: Validación (2 min)

```
"Accedo al frontend en puerto 30081... funciona.
Swagger del backend en puerto 30083... funciona.
En DevTools veo que el front llama al back correctamente."

[Mostrar navegador]
```

### Parte 5: Argo CD (4 min)

```
"Ahora el reto: instalo Argo CD...
[kubectl apply]

Accedo a la UI, creo una Application apuntando a mi repo Git...
Sincronizo... todos los recursos están Healthy.

Si modifico algo manualmente, Argo CD lo detecta y revierte."

[Mostrar UI de Argo CD]
```

### Parte 6: Probes (3 min)

```
"Las probes garantizan alta disponibilidad:

Liveness: si falla, reinicia el pod automáticamente.
Readiness: controla cuándo un pod recibe tráfico.

Voy a simular un fallo..."

[kubectl exec, kill 1, kubectl get pods -w]
```

### Cierre (1 min)

```
"Hemos desplegado una arquitectura profesional con:
✅ Alta disponibilidad (2 réplicas)
✅ Gestión de configuración (ConfigMap/Secret)
✅ Auto-recuperación (probes)
✅ GitOps con Argo CD

Gracias por ver."
```

---

## 🔗 RECURSOS ADICIONALES

- [Documentación Kubernetes](https://kubernetes.io/docs/)
- [Argo CD Docs](https://argo-cd.readthedocs.io/)
- [Docker Hub - mariasb](https://hub.docker.com/u/mariasb)
- [Kubernetes Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## 📝 NOTAS FINALES

- **NodePort range:** 30000-32767
- **ConfigMap vs Secret:** ConfigMap para config, Secret para credenciales
- **Probe types:** httpGet, tcpSocket, exec
- **Argo CD sync:** Manual o automático (automated.prune, automated.selfHeal)
- **Service names:** No pueden empezar con número (DNS-1035 label)

**¡Éxito en tu presentación! 🚀**
