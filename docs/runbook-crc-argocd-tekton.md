# Runbook: CRC (OpenShift Local) + Argo CD (OpenShift GitOps) + Tekton (OpenShift Pipelines) en macOS

**Objetivo:** levantar un laboratorio local en macOS con **CRC/OpenShift Local** y desplegar **Argo CD** y **Tekton**, con verificación, operación diaria y troubleshooting basado en problemas reales del lab (DiskPressure/taints, pods Pending, rutas, credenciales, GitOps sync, imágenes “short-name enforcing”, etc.).

> Recomendación de recursos para lab con Argo+Tekton: **16 GB RAM** y **6 CPUs** asignadas a CRC (mínimo razonable).

---

## Índice

1. [Descripción rápida: qué es cada componente](#1-descripción-rápida-qué-es-cada-componente)  
2. [Requisitos previos](#2-requisitos-previos)  
3. [Instalar CRC y herramientas](#3-instalar-crc-y-herramientas)  
4. [Pull Secret de Red Hat](#4-pull-secret-de-red-hat)  
5. [Configurar y arrancar CRC](#5-configurar-y-arrancar-crc)  
6. [Acceso a consola y oc](#6-acceso-a-consola-y-oc)  
7. [Instalar Argo CD (OpenShift GitOps) por CLI/OLM](#7-instalar-argo-cd-openshift-gitops-por-cliolm)  
8. [Crear instancia Argo CD + Route + credenciales](#8-crear-instancia-argo-cd--route--credenciales)  
9. [Cambiar contraseña de admin en Argo CD](#9-cambiar-contraseña-de-admin-en-argo-cd)  
10. [Instalar Tekton (OpenShift Pipelines) por CLI/OLM](#10-instalar-tekton-openshift-pipelines-por-cliolm)  
11. [Health checks](#11-health-checks)  
12. [Comandos operativos CRC](#12-comandos-operativos-crc)  
13. [Limpieza preventiva para evitar DiskPressure/taints](#13-limpieza-preventiva-para-evitar-diskpressuretaints)  
14. [Troubleshooting](#14-troubleshooting)  
15. [Si ejecutás `crc delete`: qué pasos repetir](#15-si-ejecutás-crc-delete-qué-pasos-repetir)  
16. [Checklist final](#16-checklist-final)  
17. [Mini caso real (historia paso a paso): hello-http con GitOps](#17-mini-caso-real-historia-paso-a-paso-hello-http-con-gitops)

---

## 1) Descripción rápida: qué es cada componente

### CRC (OpenShift Local)
- Cluster OpenShift local (single-node) en una VM.
- Útil para pruebas y labs sin depender de un cluster remoto.

### Argo CD (OpenShift GitOps)
- GitOps (CD): Git es la fuente de verdad de manifiestos YAML.
- Sincroniza cambios de Git al cluster.

### Tekton (OpenShift Pipelines)
- CI/CD nativo en Kubernetes: `Task` → `Pipeline` → `PipelineRun`.
- Comúnmente ejecuta build/test/push y actualiza Git (y Argo despliega).

---

## 2) Requisitos previos

- macOS con virtualización habilitada.
- Espacio en disco disponible (CRC puede consumir decenas de GB).
- Recomendado: **16GB RAM / 6 CPU** para CRC si vas a usar Argo+Tekton.

---

## 3) Instalar CRC y herramientas

### 3.1 Homebrew (si no lo tenés)
```bash
brew --version
```

Si no existe:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Apple Silicon (si hace falta PATH):
```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 3.2 Instalar CRC
```bash
brew install crc
```

### 3.3 Verificar `oc`
```bash
oc version
```

---

## 4) Pull Secret de Red Hat

CRC necesita **Pull Secret**:
1. Crear cuenta en Red Hat.
2. Descargar Pull Secret (archivo `.json` o `.txt`).
3. Guardarlo, por ejemplo: `~/Downloads/pull-secret.txt`

---

## 5) Configurar y arrancar CRC

### 5.1 Configurar recursos (recomendado)
```bash
crc config set memory 16384
crc config set cpus 6
```

Ver valores:
```bash
crc config get memory
crc config get cpus
```

### 5.2 Arrancar CRC
```bash
crc start -p ~/Downloads/pull-secret.txt
```

### 5.3 Estado
```bash
crc status
```

---

## 6) Acceso a consola y oc

### 6.1 URL de consola
```bash
crc console --url
```

### 6.2 Credenciales
```bash
crc console --credentials
```

### 6.3 Login `oc` (si hace falta)
```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443
```

Verificar:
```bash
oc whoami
oc get nodes
```

---

## 7) Instalar Argo CD (OpenShift GitOps) por CLI/OLM

> Importante: OpenShift bloquea crear `Project` con prefijo `openshift-` con `oc new-project`.  
> Para el operador usá un namespace no reservado: `gitops-operator`.

### 7.1 Crear namespace del operador
```bash
oc new-project gitops-operator
```

### 7.2 OperatorGroup (AllNamespaces)
Esto evita el error: `OwnNamespace InstallModeType not supported`.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: gitops-operator-group
  namespace: gitops-operator
spec: {}
EOF
```

### 7.3 Subscription
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: gitops-operator
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### 7.4 Verificar CSV (debe ser `Succeeded`)
```bash
oc get csv -n gitops-operator
```

---

## 8) Crear instancia Argo CD + Route + credenciales

### 8.1 Crear namespace runtime `openshift-gitops` (si no existe)
Como `new-project` está bloqueado para `openshift-*`, se crea como `Namespace`:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-gitops
EOF
```

### 8.2 Crear instancia ArgoCD (CR)
```bash
cat <<'EOF' | oc apply -f -
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec: {}
EOF
```

### 8.3 Ver pods y Route
```bash
oc get pods -n openshift-gitops
oc get route -n openshift-gitops
```

### 8.4 Password admin (OpenShift GitOps)
En OpenShift GitOps el password suele estar en `argocd-secret` (no en `argocd-initial-admin-secret`).

```bash
oc get secret -n openshift-gitops argocd-secret -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

- Usuario: `admin`
- URL: la `Route` `openshift-gitops-server` (o similar)

---

## 9) Cambiar contraseña de admin en Argo CD

### 9.1 Por qué no se guarda en texto plano
Argo CD guarda el password como **hash bcrypt** dentro del Secret `argocd-secret`:
- `admin.password` (bcrypt)
- `admin.passwordMtime` (timestamp para forzar reload)

### 9.2 Generar bcrypt en macOS (recomendado: `htpasswd`)
Instalar herramienta:
```bash
brew install httpd
```

Generar hash (ejemplo):
```bash
htpasswd -nbBC 10 admin 'Admin1234!' | cut -d: -f2
```

> Comentario: evitá copiar/pegar “a mano” si el terminal mete caracteres raros; mejor usar variables y patch directo.

### 9.3 Parchar el Secret con el hash (sin copiar/pegar)
```bash
NEWPASS='Admin1234!'
HASH=$(htpasswd -nbBC 10 admin "${NEWPASS}" | cut -d: -f2)
TS=$(date -u +%FT%TZ)

oc -n openshift-gitops patch secret argocd-secret --type merge \
  -p '{"stringData":{"admin.password":"'"${HASH}"'","admin.passwordMtime":"'"${TS}"'"}}'
```

### 9.4 Reiniciar ArgoCD server para aplicar el cambio
```bash
oc -n openshift-gitops get deploy | grep -i server
oc -n openshift-gitops rollout restart deploy/openshift-gitops-server
oc -n openshift-gitops rollout status deploy/openshift-gitops-server
```

### 9.5 Probar login sin depender de la Route (port-forward)
```bash
oc -n openshift-gitops port-forward svc/openshift-gitops-server 8080:443
```

Abrir:
- `https://localhost:8080`
- user: `admin`
- pass: `NEWPASS`

---

## 10) Instalar Tekton (OpenShift Pipelines) por CLI/OLM

### 10.1 Subscription (global en `openshift-operators`)
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### 10.2 Verificar CSV + pods
```bash
oc get csv -n openshift-operators | grep -i pipelines
oc get pods -n openshift-pipelines
```

---

## 11) Health checks

### CRC / Cluster
```bash
crc status
oc get nodes
oc get clusterversion
```

### Argo CD
```bash
oc get pods -n openshift-gitops
oc get route -n openshift-gitops
```

### Tekton
```bash
oc get pods -n openshift-pipelines
```

---

## 12) Comandos operativos CRC

### Start/Stop/Status
```bash
crc start -p ~/Downloads/pull-secret.txt
crc stop
crc status
```

### Consola
```bash
crc console --url
crc console --credentials
```

### Config recursos
```bash
crc config get memory
crc config get cpus
crc config set memory 16384
crc config set cpus 6
```

### Diagnóstico general (golden commands)
```bash
oc get events -A --sort-by='.lastTimestamp' | tail -n 50
oc describe node crc | egrep -i 'Taints|DiskPressure|MemoryPressure|Ready' -A2
```

---

## 13) Limpieza preventiva para evitar DiskPressure/taints

> Objetivo: evitar llegar a `node.kubernetes.io/disk-pressure:NoSchedule` y no tener que hacer `crc delete`.

### 13.1 Señales de alerta temprana
- Pods se quedan en `Pending`.
- `oc describe node crc` muestra `DiskPressure True` o aparece taint `disk-pressure:NoSchedule`.
- `crc status` muestra `Disk Usage` muy cerca del total dentro de la VM.

### 13.2 Limpieza de pods terminados (segura en lab)
```bash
# GitOps
oc delete pod -n openshift-gitops --field-selector=status.phase==Succeeded
oc delete pod -n openshift-gitops --field-selector=status.phase==Failed

# Pipelines
oc delete pod -n openshift-pipelines --field-selector=status.phase==Succeeded
oc delete pod -n openshift-pipelines --field-selector=status.phase==Failed
```

### 13.3 Limpieza de ejecuciones Tekton (PipelineRuns/TaskRuns)
```bash
oc get pipelineruns -A
oc get taskruns -A

oc delete pipelineruns -n openshift-pipelines --all
oc delete taskruns -n openshift-pipelines --all
```

### 13.4 Reinicio “suave” de CRC (sin borrar cluster)
```bash
crc stop
crc start -p ~/Downloads/pull-secret.txt
```

### 13.5 Workaround si el nodo quedó tainted (solo para destrabar)
```bash
oc adm taint nodes crc node.kubernetes.io/disk-pressure:NoSchedule-
```

> Comentario: si `DiskPressure=True` el taint puede volver. Lo correcto es liberar espacio.

---

## 14) Troubleshooting

### 14.1 “No catalog items found” / confusión OperatorHub vs Software Catalog
```bash
oc get packagemanifests | wc -l
oc get catalogsources -n openshift-marketplace
```
Solución práctica: instalar por CLI con `OperatorGroup` + `Subscription`.

### 14.2 No se permite crear projects `openshift-*`
**Error:** `cannot request a project starting with "openshift-"`
**Solución:** crear como `Namespace` YAML (ver sección 8.1).

### 14.3 CSV FAILED: `UnsupportedOperatorGroup` / `OwnNamespace InstallModeType not supported`
Causa: OperatorGroup en OwnNamespace.  
Solución: OperatorGroup AllNamespaces (`spec: {}`), luego reintentar.

### 14.4 Pods “Pending / ContainerCreating” por CNI (Multus/OVN) + eventos
**Señal típica:** el pod queda en `ContainerCreating` y en eventos ves errores de sandbox/network.

Comandos de diagnóstico:
```bash
oc -n <ns> get pods -o wide
oc -n <ns> describe pod <pod> | tail -n 80
oc -n <ns> get events --sort-by='.lastTimestamp' | tail -n 50
```

**Workaround que funcionó en lab (reiniciar componentes de red):**
- Verificar estado de pods OVN y Multus:
```bash
oc -n openshift-multus get pods
oc -n openshift-ovn-kubernetes get pods
```
- Si ves `ovnkube-node` no Ready (ej: 7/8), reiniciá el pod del daemonset (en CRC hay 1):
```bash
oc -n openshift-ovn-kubernetes delete pod -l app=ovnkube-node
oc -n openshift-ovn-kubernetes get pods -w
```
> Comentario: una vez que `ovnkube-node` quedó 8/8 Running, desaparecieron los errores de red en nuevos pods.

### 14.5 Error real: `ImageInspectError` por “short name mode is enforcing”
**Síntomo (real):**
- Pod queda `ImageInspectError`
- Mensaje tipo: “short name mode is enforcing … returns ambiguous list”

Diagnóstico:
```bash
oc -n <ns> describe pod <pod> | tail -n 80
oc -n <ns> get pod <pod> -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}{"\n"}{.status.containerStatuses[0].state.waiting.message}{"\n"}'
```

**Solución que funcionó:** **usar imagen fully qualified** (por ejemplo `docker.io/...`) en el YAML del Deployment y hacer sync con Argo.

Ejemplo:
- `nginxinc/nginx-unprivileged:stable`
- `docker.io/nginxinc/nginx-unprivileged:stable`

### 14.6 ArgoCD “Synced pero Suspended / App no aplica cambios”
Si `Application` dice `Suspended`, o no aplica cambios automáticamente:
- Hacer **sync manual** desde la UI de ArgoCD.
- O forzar refresh:
```bash
oc -n openshift-gitops annotate application hello-http argocd.argoproj.io/refresh=hard --overwrite
```

**Qué seleccionar en la pantalla de Sync (recomendación práctica para lab):**
- **Sync options**:
  - Prune (si querés que borre recursos que ya no están en Git)
  - Apply Out of Sync Only (opcional; más rápido y seguro)
  - Create Namespace (solo si tu app administra namespaces y querés que Argo cree el ns)
  - Force (solo si estás trabado por drift/estado raro; úsalo como “martillo”)
- **Strategy**:
  - Generalmente Apply (default) alcanza.
- **Dry run**:
  - desactivado cuando querés aplicar de verdad.

> Comentario: el sync manual desde la consola de ArgoCD fue lo que finalmente actualizó el Deployment al `docker.io/...`.

### 14.7 Rollout trabado: “1 old replicas are pending termination…”
Esto puede pasar si el rollout no puede crear el nuevo pod o el viejo no termina.

Comandos útiles:
```bash
oc -n <ns> rollout status deploy/<deploy>
oc -n <ns> get rs
oc -n <ns> get pods -o wide
```

Acciones típicas en lab:
```bash
oc -n <ns> delete pod -l app=<app> --force --grace-period=0
```

> Comentario: borrar pods “a la fuerza” sirve para destrabar, pero la causa real suele estar en CNI o en imagen.

---

## 15) Si ejecutás `crc delete`: qué pasos repetir

### 15.1 Qué hace `crc delete`
Borra **todo** el cluster local (VM + recursos). Se pierde:
- operadores (GitOps/Pipelines),
- namespaces creados,
- routes,
- secrets,
- CRDs y configuración del lab.

### 15.2 Qué repetir después de `crc delete` (resumen)
1. `crc start -p ~/Downloads/pull-secret.txt`  
2. Reconfigurar recursos (`crc config set memory/cpus`).  
3. Instalar GitOps operator (sección 7).  
4. Crear `openshift-gitops` + ArgoCD CR (sección 8).  
5. Instalar Pipelines operator (sección 10).  
6. Re-aplicar manifests GitOps desde GitHub (Git vuelve a ser fuente de verdad).

---

## 16) Checklist final

- [ ] `crc status` → OpenShift Running  
- [ ] `oc get nodes` → nodo Ready  
- [ ] `oc get csv -n gitops-operator` → GitOps operator Succeeded  
- [ ] `oc get pods -n openshift-gitops` → todos Running  
- [ ] `oc get route -n openshift-gitops` → URL accesible, login ok  
- [ ] `oc get csv -n openshift-operators | grep pipelines` → Succeeded  
- [ ] `oc get pods -n openshift-pipelines` → pods Running  
- [ ] ArgoCD `Application` → `Synced / Healthy`

---

## 17) Mini caso real (historia paso a paso): hello-http con GitOps

### 17.1 Objetivo del mini caso
Desplegar `hello-http` vía ArgoCD (GitOps), y resolver problemas reales hasta llegar a:
- `Application`: **Synced / Healthy**
- `Deployment rollout`: **successfully rolled out**
- `Pod`: **Running**

---

### 17.2 Manifiesto base del Deployment (Git)
Archivo:
```bash
apps/hello-http/manifests/deploy.yaml
```

Ejemplo (importante: imagen fully qualified cuando aplica):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-http
  namespace: hello-http
  labels:
    app: hello-http
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-http
  template:
    metadata:
      labels:
        app: hello-http
    spec:
      containers:
        - name: nginx
          image: docker.io/nginxinc/nginx-unprivileged:stable
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: web
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web
          configMap:
            name: hello-index
```

---

### 17.3 Diagnóstico real 1: pods en `ContainerCreating` (CNI / Multus / OVN)
**Síntomas que viste:**
- Pod del nuevo ReplicaSet queda `ContainerCreating`
- En `events` aparece error de sandbox/network con Multus y “Unauthorized”.

**Comandos que ayudaron:**
```bash
oc -n hello-http get pods -o wide
oc -n hello-http get events --sort-by='.lastTimestamp' | tail -n 20

oc -n openshift-multus get pods
oc -n openshift-ovn-kubernetes get pods
```

**Fix que funcionó (reiniciar OVN node):**
```bash
oc -n openshift-ovn-kubernetes delete pod -l app=ovnkube-node
oc -n openshift-ovn-kubernetes get pods
```

> Comentario: cuando `ovnkube-node` quedó 8/8 Running, el problema de CNI dejó de repetirse en nuevos pods.

---

### 17.4 Diagnóstico real 2: `ImageInspectError` por short-name enforcing
**Síntoma que viste:**
```bash
oc -n hello-http describe pod <pod> | tail -n 80
# ...
# Failed to inspect image "": short name mode is enforcing ... ambiguous list
```

**Causa práctica (lab):** imagen no fully qualified:
- `nginxinc/nginx-unprivileged:stable` → ambiguo en el runtime (short-name enforcing)
- Solución: `docker.io/nginxinc/nginx-unprivileged:stable`

---

### 17.5 Fix real aplicado en Git con `sed` + git push (lo que hiciste y funcionó)

**Editar YAML con `sed` (macOS):**
```bash
sed -i '' 's#image: nginxinc/nginx-unprivileged:stable#image: docker.io/nginxinc/nginx-unprivileged:stable#' apps/hello-http/manifests/deploy.yaml
grep -n "image:" apps/hello-http/manifests/deploy.yaml
```

**Commits y push:**
```bash
git add apps/hello-http/manifests/deploy.yaml
git commit -m "Fully qualify nginx unprivileged image"
git push
```

---

### 17.6 Sync real en ArgoCD (lo que destrabó)
Aunque intentaste refresh por CLI, lo que terminó de aplicar fue:
- Sync manual desde la consola web de ArgoCD (Application → Sync).

Verificación posterior (funcionó):
```bash
oc -n openshift-gitops get application hello-http \
  -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}{.status.sync.revision}{"\n"}'
# Synced
# Healthy
# <commit>
```

Y en el deploy:
```bash
oc -n hello-http get deploy hello-http -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# docker.io/nginxinc/nginx-unprivileged:stable
```

---

### 17.7 Rollout final: recrear pod y confirmar Running
**Borrar pod por label (lab-friendly) para que recree con la nueva imagen:**
```bash
oc -n hello-http delete pod -l app=hello-http --force --grace-period=0
```

**Verificar:**
```bash
oc -n hello-http get pods -o wide
oc -n hello-http rollout status deploy/hello-http
```

Esperado:
- Pod `Running`
- `deployment "hello-http" successfully rolled out`

---

### 17.8 Extra: dejar `replicas` “permanente” en el YAML con sed
Si querés fijar `replicas: 1` (o el valor que quieras) directamente en Git:

```bash
# Setear replicas a 1 en el YAML del deploy (asume que ya existe la línea "replicas:")
sed -i '' 's/^[[:space:]]*replicas:[[:space:]]*[0-9]\+/  replicas: 1/' apps/hello-http/manifests/deploy.yaml

git add apps/hello-http/manifests/deploy.yaml
git commit -m "Set hello-http replicas to 1"
git push
```

> Comentario: esto evita “parches manuales” en cluster que después Argo te pisa.

---

### 17.9 Revert real (cuando “apliqué el cambio pero no quería”)
Si querés revertir el último commit (recomendado en GitOps):

```bash
git revert HEAD
git push
```

Si querés revertir un commit específico:
```bash
git log --oneline --max-count=10
git revert <SHA_DEL_COMMIT>
git push
```

> Comentario: en GitOps, revert en Git es lo más limpio. Evitás drift con el cluster.

---

### 17.10 Higiene: limitar historial de ReplicaSets en el YAML (no con patch)
En vez de:
```bash
oc -n hello-http patch deploy hello-http -p '{"spec":{"revisionHistoryLimit":3}}'
```

Hacelo en Git (permanente):

```bash
# Inserta revisionHistoryLimit debajo de replicas (si tu archivo tiene replicas: en spec)
sed -i '' '/^[[:space:]]*replicas:[[:space:]]*[0-9]\+/a\
\  revisionHistoryLimit: 3
' apps/hello-http/manifests/deploy.yaml

git add apps/hello-http/manifests/deploy.yaml
git commit -m "Set revisionHistoryLimit to 3"
git push
```

> Comentario: si ya existe revisionHistoryLimit, mejor reemplazarla (evitás duplicados).
