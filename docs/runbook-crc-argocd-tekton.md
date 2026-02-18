# Runbook: CRC (OpenShift Local) + Argo CD (OpenShift GitOps) + Tekton (OpenShift Pipelines) en macOS

**Objetivo:** levantar un laboratorio local en macOS con **CRC/OpenShift Local** y desplegar **Argo CD** y **Tekton**, con verificación, operación diaria y troubleshooting basado en problemas reales del lab (DiskPressure/taints, pods Pending, rutas, credenciales).

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

> Recomendación: **no copies a mano** desde un output “sucio”. Mejor guardarlo en variable y parchar directo.

### 9.3 Parchar el Secret con el hash (sin copiar/pegar)
```bash
NEWPASS='Admin1234!'
HASH=$(htpasswd -nbBC 10 admin "${NEWPASS}" | cut -d: -f2)
TS=$(date -u +%FT%TZ)

oc -n openshift-gitops patch secret argocd-secret --type merge \
  -p '{"stringData":{"admin.password":"'"${HASH}"'","admin.passwordMtime":"'"${TS}"'"}}'
```

### 9.4 Reiniciar ArgoCD server para aplicar el cambio
Ver nombre del deployment (puede variar):
```bash
oc -n openshift-gitops get deploy | grep -i server
```

Reiniciar (si el deployment es `openshift-gitops-server`):
```bash
oc -n openshift-gitops rollout restart deploy/openshift-gitops-server
oc -n openshift-gitops rollout status deploy/openshift-gitops-server
```

### 9.5 Probar login sin depender de la Route (port-forward)
Si la Route/certificados molestan, probá local:
```bash
oc -n openshift-gitops port-forward svc/openshift-gitops-server 8080:443
```

Abrir:
- `https://localhost:8080`
- user: `admin`
- pass: el de `NEWPASS`

### 9.6 Verificación del hash
```bash
oc -n openshift-gitops get secret argocd-secret -o jsonpath='{.data.admin\.password}' | base64 -d; echo
oc -n openshift-gitops get secret argocd-secret -o jsonpath='{.data.admin\.passwordMtime}' | base64 -d; echo
```

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

### Diagnóstico general
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

### 13.2 Limpieza de pods terminados (suele ayudar y es segura)
Borra pods `Completed/Succeeded` y `Failed` que se acumulan:

```bash
# GitOps
oc delete pod -n openshift-gitops --field-selector=status.phase==Succeeded
oc delete pod -n openshift-gitops --field-selector=status.phase==Failed

# Pipelines
oc delete pod -n openshift-pipelines --field-selector=status.phase==Succeeded
oc delete pod -n openshift-pipelines --field-selector=status.phase==Failed
```

### 13.3 Limpieza de ejecuciones Tekton (PipelineRuns/TaskRuns)
Tekton puede acumular historial y objetos rápidamente (lab = se pueden borrar):

```bash
# Ver lo que hay
oc get pipelineruns -A
oc get taskruns -A

# Borrar en el namespace típico (ajusta si usas otro)
oc delete pipelineruns -n openshift-pipelines --all
oc delete taskruns -n openshift-pipelines --all
```

### 13.4 Limpieza de “basura” del cluster por label (opcional)
Si querés borrar pods viejos por label (ejemplo apps), ajusta labels a tu caso.

```bash
# Ejemplo: borrar pods de una app por label
oc delete pod -n <ns> -l app=<appname>
```

### 13.5 Reinicio “suave” de CRC (sin borrar cluster)
Esto NO borra tu cluster, solo reinicia VM/servicios:

```bash
crc stop
crc start -p ~/Downloads/pull-secret.txt
```

### 13.6 Limpiar cache de CRC en el host (cuando hay DiskPressure recurrente)
Esto NO es `crc delete`, pero ayuda cuando el cache se vuelve grande:

```bash
crc stop
rm -rf ~/.crc/cache
crc start -p ~/Downloads/pull-secret.txt
```

### 13.7 Si el nodo quedó tainted y necesitas recuperar scheduling (workaround)
**No recomendado como solución permanente**, pero puede destrabar mientras limpias.

```bash
oc adm taint nodes crc node.kubernetes.io/disk-pressure:NoSchedule-
```

> Si `DiskPressure` sigue en True, el taint puede volver. Lo correcto es liberar espacio.

---

## 14) Troubleshooting

### 14.1 “No catalog items found” / confusión OperatorHub vs Software Catalog
- Si el GUI no muestra operadores, confirmar catálogo por CLI:
```bash
oc get packagemanifests | wc -l
oc get catalogsources -n openshift-marketplace
```
- Solución práctica: instalar por CLI con `OperatorGroup` + `Subscription`.

### 14.2 No se permite crear projects `openshift-*`
**Error:**
`cannot request a project starting with "openshift-"`

**Solución:** crear como `Namespace` YAML:
```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-gitops
EOF
```

### 14.3 CSV FAILED: `UnsupportedOperatorGroup` / `OwnNamespace InstallModeType not supported`
**Causa:** OperatorGroup en OwnNamespace.

**Solución:** OperatorGroup AllNamespaces (`spec: {}`), luego reinstalar/reintentar:
```bash
oc delete operatorgroup gitops-operator-group -n gitops-operator

cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: gitops-operator-group
  namespace: gitops-operator
spec: {}
EOF
```

Si el CSV quedó en Failed, borrar el CSV para que OLM lo recree:
```bash
oc get csv -n gitops-operator
oc delete csv -n gitops-operator <NOMBRE_DEL_CSV>
```

### 14.4 Argo CD: Route muestra “Application is not available”
**Causa típica:** pods Pending por recursos insuficientes o node tainted por DiskPressure.

Diagnóstico:
```bash
oc get pods -n openshift-gitops
oc describe node crc | egrep -i "Taints|DiskPressure|Ready"
```

Solución (lab):
- aplicar limpieza preventiva (sección 13),
- reiniciar CRC sin borrar cluster,
- evitar llenar el disco.

### 14.5 Argo CD: “invalid username or password”
- Confirmar que estás en la **Route de ArgoCD** (no la consola OpenShift).
- Confirmar `admin.enabled`:
```bash
oc -n openshift-gitops get cm argocd-cm -o jsonpath='{.data.admin\.enabled}{"\n"}'
```
- Reset de password con la sección 9 (bcrypt + patch + restart).
- Alternativa: probar por `port-forward` para evitar issues de Route/certs.

---

## 15) Si ejecutás `crc delete`: qué pasos repetir

### 15.1 Qué hace `crc delete`
`crc delete` borra **todo** el cluster local (VM + recursos). Se pierde:
- operadores (GitOps/Pipelines),
- namespaces creados,
- routes,
- secrets,
- CRDs y toda configuración del lab.

### 15.2 Qué repetir después de `crc delete` (resumen)
1. `crc start -p ~/Downloads/pull-secret.txt`  
2. Reconfigurar recursos si hace falta (`crc config set memory/cpus`).  
3. Instalar GitOps operator (sección 7).  
4. Crear namespace `openshift-gitops` + ArgoCD CR (sección 8).  
5. Instalar Pipelines operator (sección 10).  
6. Re-aplicar manifests de tu “hello world”/GitOps desde GitHub (tu repo vuelve a ser la fuente de verdad).

---

## 16) Checklist final

- [ ] `crc status` → OpenShift Running  
- [ ] `oc get nodes` → nodo Ready  
- [ ] `oc get csv -n gitops-operator` → GitOps operator Succeeded  
- [ ] `oc get pods -n openshift-gitops` → todos Running 1/1  
- [ ] `oc get route -n openshift-gitops` → URL accesible y login ok  
- [ ] `oc get csv -n openshift-operators | grep pipelines` → Succeeded  
- [ ] `oc get pods -n openshift-pipelines` → pods Running  
