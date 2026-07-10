# K8s Practice Lab — Runbooks de troubleshooting

Laboratorio de práctica en Kubernetes (clúster local con `kind`), gestionado
con GitOps vía ArgoCD, donde reproduzco errores reales de despliegue a
propósito, capturo la evidencia y documento el diagnóstico y la solución
como runbook.

El objetivo: no solo "saber" los errores comunes de K8s, sino haberlos
provocado, diagnosticado y resuelto con mis propias manos.

**Convención de rutas:**
- `manifest/` — estado deseado real, gestionado por ArgoCD. Nunca contiene
  manifiestos rotos.
- `incidents/` — manifiestos usados para reproducir errores a propósito,
  evidencia real de terminal, y notas de cada caso. Fuera del alcance de
  ArgoCD.

---

## 🧪 Caso 1 — ErrImagePull

**Archivo:** `incidents/deployment_malo.yaml`
**Namespace:** `staging`

### Qué hice
Desplegué `incidents/deployment_malo.yaml` apuntando a una imagen que no
existe en el registro, para reproducir el error de pull y documentar el
diagnóstico.

### Comando
```bash
kubectl apply -f incidents/deployment_malo.yaml -n staging
```

### Evidencia real (terminal)
```bash
➜  practicas kubectl apply -f incidents/deployment_malo.yaml -n staging
deployment.apps/app-1 configured
➜  practicas kubectl get pods -n staging
NAME                     READY   STATUS         RESTARTS   AGE
app-1-574666675-vjhhr    0/1     ErrImagePull   0          17s
```

### Diagnóstico
```bash
kubectl describe pod app-1-574666675-vjhhr -n staging
```
En la sección `Events` se confirma la causa: el kubelet no pudo descargar
la imagen especificada (nombre o tag inexistente en el registro).

### Causa raíz
El campo `image:` en `incidents/deployment_malo.yaml` apunta a una
imagen/tag que no existe en el registro configurado.

### Solución
Corregir el nombre/tag de la imagen por uno válido en `manifest/deployment.yaml`
(el estado deseado real, no el archivo de incidente) y dejar que ArgoCD
sincronice, o reaplicar manualmente en el lab:
```bash
kubectl apply -f manifest/deployment.yaml -n staging
kubectl rollout status deployment/app-1 -n staging
```

### Prevención
- Validar el nombre/tag de la imagen contra el registro antes de aplicar.
- En CI/CD, agregar un paso que verifique que la imagen existe (`docker
  manifest inspect` o equivalente) antes del deploy.
- Nunca corregir directo en el clúster si el recurso vive bajo ArgoCD:
  el fix va por commit a `manifest/`, para que ArgoCD lo sincronice.

### Evidencia
Salida completa guardada en `incidents/errImagePull.txt`.

---

## 🧪 Caso 2 — CreateContainerConfigError

**Archivos:** `manifest/deployment.yaml`, `manifest/configmap.yaml`, `manifest/secret.yaml`

### Diagnóstico rápido
1. **Síntoma:** `kubectl get pods` muestra `CreateContainerConfigError`.
2. **Diagnóstico:** `kubectl describe pod <nombre>` → revisar sección `Events`.
3. **Causas comunes:** ConfigMap/Secret referenciado no existe, o el
   Deployment se aplicó antes que sus dependencias.
4. **Verificación:**
   ```bash
   kubectl get configmaps
   kubectl get secrets
   ```
   Confirmar que el recurso referenciado en `envFrom` realmente existe.
5. **Resolución:** aplicar el recurso faltante (`configmap.yaml` / `secret.yaml`
   en `manifest/`), luego `kubectl delete pod <nombre>` para forzar la
   recreación con las dependencias ya presentes.
6. **Prevención:** validar con `kubectl diff -f manifest/` antes de aplicar
   en producción; aplicar siempre las dependencias antes que el recurso
   que las usa.
7. **Evidencia:** dejar el estado del pod descrito en un archivo `.txt`
   para poder hacer auditorías:
   ```bash
   kubectl describe pod <nombre-del-pod> > incidents/createcontainerconfigerror.txt
   ```
8. Guardar el archivo en `incidents/` y adjuntarlo si se necesita escalar.

---

## 🧪 Caso 3 — NodePort inaccesible en kind + Windows

### Síntoma
`curl http://localhost:<nodeport>` fallaba con "Could not connect" para
todos los Services tipo `NodePort`.

### Diagnóstico
`kubectl get endpoints` mostró que los Services sí tenían pods sanos detrás.
`kubectl port-forward` sí funcionaba. El problema estaba en la capa entre
Docker Desktop y localhost, no en Kubernetes.

### Causa raíz
`kind` sobre Docker Desktop en Windows no reenvía automáticamente puertos
`NodePort` altos al host.

### Resolución
Cambiar el tipo del Service a `LoadBalancer` en `manifest/service.yaml`.
Docker Desktop sí expone estos directo a `localhost:<port>` sin
configuración extra.

### Prevención
En local, `LoadBalancer` o `Ingress` son las opciones prácticas —
`NodePort` queda solo para debugging interno.

---

## 🧪 Caso 4 — ImagePullBackOff post-sincronización de ArgoCD

### Síntoma
Tras cambiar el `Path` de la Application de ArgoCD a `manifest/`, se
sincronizó un `deployment.yaml` con una imagen inválida que no se había
corregido antes del commit.

### Diagnóstico
```bash
kubectl describe pod <nombre-del-pod-nuevo> -n default
```
Sección `Events` confirma `ImagePullBackOff` sobre el ReplicaSet recién
creado por la sincronización.

### Causa raíz
El commit a `manifest/deployment.yaml` incluía una imagen/tag inválido;
ArgoCD lo sincronizó automáticamente como parte de su Sync Policy.

### Resolución
Corrección aplicada **en el repositorio**, nunca con `kubectl` directo:
```bash
git add manifest/deployment.yaml
git commit -m "Fix: corregir tag de imagen inválido"
git push
```
ArgoCD detecta el nuevo commit y sincroniza la versión corregida
automáticamente.

### Verificación
```
Sync Status: Synced
Health: Healthy
```

### Prevención
Revisar el `DIFF` que muestra ArgoCD antes de confirmar cada sync manual
(mismo criterio que un `terraform plan`), y nunca aplicar cambios directos
al clúster cuando el recurso vive bajo GitOps — eso rompe la fuente única
de verdad y ArgoCD puede revertirlo en el próximo sync.

---

## 📂 Estructura del repo

```
k8s-practice-lab/
├── manifest/                  # Estado deseado real — sincronizado por ArgoCD
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── role.yaml               # RBAC: permisos de solo lectura sobre pods
│   └── rolebinding.yaml         # RBAC: asigna el Role a la ServiceAccount
├── incidents/                  # Evidencia de troubleshooting — fuera del alcance de ArgoCD
│   ├── deployment_malo.yaml     # Manifiesto usado para reproducir ErrImagePull
│   ├── errImagePull.txt         # Salida real de terminal del incidente
│   └── evidencia.txt
├── runbook.md                  # Este archivo — procedimientos operativos L2
└── README.md
```

## 🛠️ Cómo reproducirlo tú mismo

```bash
kind create cluster
kubectl create namespace argocd
kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Nota: usar 'create', no 'apply' — el CRD applicationsets.argoproj.io
# excede el límite de anotaciones que impone 'apply' (last-applied-configuration)

kubectl port-forward svc/argocd-server -n argocd 8081:443
# Conectar la Application vía UI apuntando a este repo, path: manifest/
```

Para reproducir el incidente de ErrImagePull a propósito:
```bash
kubectl apply -f incidents/deployment_malo.yaml
kubectl get pods
kubectl describe pod <nombre-del-pod-roto>
```

---

## 🎯 Próximos casos (en progreso)
- [ ] `CrashLoopBackOff` como quinto caso documentado
- [ ] Namespaces `staging`/`production` separados (hoy todo vive en `default`)
- [ ] Kustomize (base + overlays) para separar entornos
