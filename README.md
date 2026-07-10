# k8s-practice-lab

Laboratorio de práctica en Kubernetes ejecutado sobre un clúster local con `kind`, gestionado con **GitOps vía ArgoCD**. Cubre objetos base (Deployment, Service, ConfigMap, Secret, Ingress), control de acceso (RBAC), operación L2 (monitoreo, rollback) y troubleshooting de incidentes reales — incluido uno detectado **después** de sincronizar con ArgoCD.

Repo de aprendizaje deliberado. La estructura separa el **estado deseado real** (`manifests/`, gestionado por ArgoCD) de la **evidencia histórica de incidentes** (`incidentes/`, que ArgoCD nunca toca).

---

## Estructura

```
k8s-practice-lab/
├── manifests/                # Estado deseado real — sincronizado por ArgoCD
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── role.yaml              # RBAC: permisos de solo lectura sobre pods
│   └── rolebinding.yaml        # RBAC: asigna el Role a la ServiceAccount
├── incidentes/                 # Evidencia de troubleshooting — fuera del alcance de ArgoCD
│   ├── deployment_malo.yaml     # Manifiesto usado para reproducir ErrImagePull a propósito
│   ├── errImagePull.txt         # Salida real de terminal del incidente
│   └── evidencia.txt
├── runbook.md                  # Procedimientos operativos L2
└── README.md
```

---

## Stack

| Componente | Detalle |
|---|---|
| Kubernetes (kind) | v1.34.3, clúster local vía Docker Desktop |
| GitOps | ArgoCD v3.4.5, instalado en el mismo clúster (namespace `argocd`) |
| Ingress Controller | NGINX Ingress Controller (variante oficial para kind) |
| Métricas | metrics-server |
| Control de acceso | RBAC nativo (Role + RoleBinding) |

---

## GitOps con ArgoCD

El repo se gestiona 100% declarativamente: **ArgoCD vigila `manifests/` y aplica los cambios mergeados a `main`**. No se corre `kubectl apply` a mano contra el clúster real desde que ArgoCD quedó configurado — el flujo es: cambio en código → commit → push → ArgoCD sincroniza (manual, revisado antes de aplicar).

**Application configurada:**
- Repository: `https://github.com/JOSHUETDAVID/k8s-practice-lab`
- Path: `manifests/`
- Destination: mismo clúster (`https://kubernetes.default.svc`), namespace `default`
- Sync Policy: Manual — cada sincronización se revisa (`DIFF`) antes de confirmar, mismo criterio que un `terraform plan`

**Estado actual:** `Synced` + `Healthy`, historial de 5 revisiones conservado (incluye el incidente descrito abajo y su corrección).

---

## RBAC — principio de mínimo privilegio, verificado

`manifests/role.yaml` define un `Role` de solo lectura sobre pods:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

`manifests/rolebinding.yaml` lo asigna a la ServiceAccount `default` del namespace.

**Verificación** (no solo aplicado — comprobado):

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:default -n default
# yes

kubectl auth can-i delete pods --as=system:serviceaccount:default:default -n default
# no
```

Confirma que el Role otorga exactamente los permisos declarados — ni más, ni menos.

---

## Decisiones técnicas

| Decisión | Razón |
|---|---|
| `manifests/` separado de `incidentes/` | Evita que ArgoCD reconcilie manifiestos rotos usados solo para reproducir errores |
| `LoadBalancer` en el Service | Docker Desktop lo expone directo a localhost; `NodePort` falla por limitación de puertos altos en kind + Windows |
| Ingress con NGINX Controller (variante kind) | El manifiesto oficial para kind difiere del usado en clústeres cloud |
| ArgoCD con Sync Policy Manual | Permite revisar el diff antes de aplicar, igual que `kubectl diff` o `terraform plan` |
| RBAC con Role acotado a namespace | Principio de mínimo privilegio — ClusterRole hubiera sido excesivo para este caso |
| Corrección de incidentes vía commit al repo, nunca `kubectl` directo | Respeta GitOps como fuente única de verdad; evita que ArgoCD revierta cambios manuales en el próximo sync |

---

## Incidentes documentados

Formato: síntoma → diagnóstico → causa raíz → resolución → prevención. Detalle completo en `runbook.md` y `incidentes/`.

### 1. ErrImagePull (reproducido a propósito)
Imagen con tag inexistente en `incidentes/deployment_malo.yaml`. Diagnosticado con `kubectl describe pod`, evidencia real en `incidentes/errImagePull.txt`.

### 2. CreateContainerConfigError
ConfigMap/Secret referenciado por el Deployment antes de existir en el clúster. Causa: se aplicó el Deployment sin aplicar antes sus dependencias.

### 3. NodePort inaccesible en kind + Windows
`curl` a puertos NodePort fallaba pese a que `kubectl get endpoints` mostraba pods sanos. Resuelto migrando a `LoadBalancer`, que Docker Desktop sí expone a localhost.

### 4. ImagePullBackOff post-sincronización de ArgoCD
Tras cambiar el Path de la Application a `manifests/`, ArgoCD sincronizó un `deployment.yaml` con una imagen inválida que no se había corregido antes del commit. Diagnosticado con `kubectl describe pod` sobre el nuevo ReplicaSet. **Corrección aplicada en el repositorio** (`git commit` con mensaje "Update deployment.yaml"), no en el clúster — ArgoCD detectó el nuevo commit y sincronizó la versión corregida automáticamente. 
Verificado: `Sync Status: Synced`, `Health: Healthy`.

---

## Cómo reproducirlo

```bash
kind create cluster
kubectl create namespace argocd
kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Nota: usar 'create', no 'apply', por que da error — el CRD applicationsets.argoproj.io
# excede el límite de anotaciones que impone 'apply' (last-applied-configuration)

kubectl port-forward svc/argocd-server -n argocd 8081:443
# Conectar la Application vía UI apuntando a este repo, path: manifests/
```

Para reproducir el incidente de ErrImagePull a propósito:
```bash
kubectl apply -f incidentes/deployment_malo.yaml
kubectl get pods
kubectl describe pod <nombre-del-pod-roto>
```

---

## Qué sigue

- [ ] `CrashLoopBackOff` como cuarto caso documentado
- [ ] Namespaces `staging`/`production` separados (hoy todo vive en `default`)
- [ ] Kustomize (base + overlays) para separar entornos
- [ ] Simulación de flujo de ticketing (formato ServiceNow/Jira) por incidente

---

## Referencias

- [Kubernetes Docs — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Docs — RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [ArgoCD — Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [kind — Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
