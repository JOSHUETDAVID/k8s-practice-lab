# K8s Practice Lab — Runbooks de troubleshooting

Laboratorio de práctica en Kubernetes (cluster local con `kind`) donde reproduzco
errores reales de despliegue a propósito, capturo la evidencia y documento el
diagnóstico y la solución como runbook.

El objetivo: no solo "saber" los errores comunes de K8s, sino haberlos
provocado, diagnosticado y resuelto con mis propias manos.

---

## 🧪 Caso 1 — ErrImagePull

**Archivo:** `deployment_malo.yaml`
**Namespace:** `staging`

### Qué hice
Desplegué `deployment_malo.yaml` apuntando a una imagen que no existe en el
registro, para reproducir el error de pull y documentar el diagnóstico.

### Comando
```bash
kubectl apply -f deployment_malo.yaml -n staging
```

### Evidencia real (terminal)
```bash
➜  practicas kubectl apply -f deployment_malo.yaml -n staging
deployment_malo.apps/app-1 configured
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
El campo `image:` en `deployment_malo.yaml` apunta a una imagen/tag que no
existe en el registro configurado.

### Solución
Corregir el nombre/tag de la imagen en `deployment_malo.yaml` por uno válido y
reaplicar:
```bash
kubectl apply -f deployment_malo.yaml -n staging
kubectl rollout status deployment_malo/app-1 -n staging
```

### Prevención
- Validar el nombre/tag de la imagen contra el registro antes de aplicar.
- En CI/CD, agregar un paso que verifique que la imagen existe (`docker
  manifest inspect` o equivalente) antes del deploy.

---

## 📂 Estructura del repo
```
k8s-practice-lab/
├── deployment.yaml
├── deployment_malo.yaml
├── service.yaml
├── runbook.md
└── evidencia/
└── errimagepull.txt
```

## 🛠️ Cómo reproducirlo tú mismo
1. Cluster local con `kind`: `kind create cluster`
2. `kubectl apply -f deployment_malo.yaml -n staging`
3. `kubectl get pods -n staging` → verás `ErrImagePull`
4. `kubectl describe pod <nombre> -n staging` → revisa `Events`

---

## Diagnostico rapido más el estado del pod descrito

1. **Síntoma:** `kubectl get pods` muestra `CreateContainerConfigError` o `errImagePull`
2. **Diagnóstico:** `kubectl describe pod <nombre>` → revisar sección `Events`
3. **Causas comunes:** ConfigMap/Secret referenciado no existe, o nombre mal escrito
4. **Verificación:** `kubectl get configmaps` y `kubectl get secrets` — confirmar existencia real
5. **Resolución:** aplicar el recurso faltante, luego `kubectl delete pod` para forzar recreación
6. **Prevención:** validar con `kubectl diff` antes de aplicar en producción
7. **Evidencia:** dejar el estado del pod descrito en un archivo .txt para poder hacer auditorias con `kubectl describe pod <nombre-del-pod-id> > evidencia.txt`
8.  Guardar el archivo y enviar si es el caso
