# Runbook: CRC (OpenShift Local) + Argo CD (OpenShift GitOps) + Tekton (OpenShift Pipelines) en macOS

**Objetivo:** levantar un laboratorio local en macOS con **CRC/OpenShift Local** y desplegar **Argo CD** y **Tekton**, con verificación y troubleshooting basado en problemas comunes.

> Recomendación de recursos para lab con Argo+Tekton: **16 GB RAM** y **6 CPUs** asignadas a CRC.

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
9. [Instalar Tekton (OpenShift Pipelines) por CLI/OLM](#9-instalar-tekton-openshift-pipelines-por-cliolm)  
10. [Health checks](#10-health-checks)  
11. [Comandos operativos CRC](#11-comandos-operativos-crc)  
12. [Troubleshooting](#12-troubleshooting)  
13. [Checklist final](#13-checklist-final)

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
```bash
oc get secret -n openshift-gitops argocd-secret -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

- Usuario: `admin`
- URL: la `Route` `openshift-gitops-server` (o similar)

---

## 9) Instalar Tekton (OpenShift Pipelines) por CLI/OLM

### 9.1 Subscription (global en `openshift-operators`)
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

### 9.2 Verificar CSV + pods
```bash
oc get csv -n openshift-operators | grep -i pipelines
oc get pods -n openshift-pipelines
```

---

## 10) Health checks

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

## 11) Comandos operativos CRC

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

## 12) Troubleshooting

### 12.1 “No catalog items found” / confusión OperatorHub vs Software Catalog
- Si el GUI no muestra operadores, confirmar catálogo por CLI:
```bash
oc get packagemanifests | wc -l
oc get catalogsources -n openshift-marketplace
```
- Solución práctica: instalar por CLI con `OperatorGroup` + `Subscription`.

### 12.2 No se permite crear projects `openshift-*`
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

### 12.3 CSV FAILED: `UnsupportedOperatorGroup` / `OwnNamespace InstallModeType not supported`
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

### 12.4 Argo CD: Route muestra “Application is not available”
**Causa típica:** pods Pending por recursos insuficientes.

Diagnóstico:
```bash
oc get pods -n openshift-gitops
oc describe pod -n openshift-gitops <pod-pending> | tail -n 40
```

Si dice `Insufficient memory`, subir recursos y reiniciar CRC:
```bash
crc stop
crc config set memory 16384
crc config set cpus 6
crc start -p ~/Downloads/pull-secret.txt
```

### 12.5 Tekton Triggers Pending por taint `disk-pressure`
Diagnóstico:
```bash
oc describe node crc | egrep -i "Taints|DiskPressure"
```

Si hay:
`node.kubernetes.io/disk-pressure:NoSchedule`

Solución práctica (lab): liberar/limpiar cache local y reiniciar CRC:
```bash
crc stop
rm -rf ~/.crc/cache
crc start -p ~/Downloads/pull-secret.txt
```

Si queda un pod duplicado en `Error` y ya hay otro Running, borrar el pod fallido:
```bash
oc get pods -n openshift-pipelines | grep triggers
oc delete pod -n openshift-pipelines <pod-en-error>
```

### 12.6 Argo CD: “invalid username or password”
- Confirmar que estás en la **Route de ArgoCD** (no la consola OpenShift).
- Password correcto (OpenShift GitOps):
```bash
oc get secret -n openshift-gitops argocd-secret -o jsonpath='{.data.admin\.password}' | base64 -d; echo
```

Si reseteaste el password, reiniciar el server:
```bash
oc rollout restart deployment -n openshift-gitops openshift-gitops-server
oc rollout status deployment -n openshift-gitops openshift-gitops-server
```

---

## 13) Checklist final

- [ ] `crc status` → OpenShift Running  
- [ ] `oc get nodes` → nodo Ready  
- [ ] `oc get csv -n gitops-operator` → GitOps operator Succeeded  
- [ ] `oc get pods -n openshift-gitops` → todos Running 1/1  
- [ ] `oc get route -n openshift-gitops` → URL accesible y login ok  
- [ ] `oc get csv -n openshift-operators | grep pipelines` → Succeeded  
- [ ] `oc get pods -n openshift-pipelines` → pods Running  
