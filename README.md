# k8s-practice-lab

Laboratorio de práctica en Kubernetes ejecutado sobre un clúster local con `kind`, cubriendo objetos base (Deployment, Service, ConfigMap, Secret, Ingress), operación L2 (monitoreo con metrics-server, rollback real) y troubleshooting de incidentes reales vividos durante la práctica.

Repo de aprendizaje deliberado, no un proyecto de producción. La estructura está pensada para servir como referencia rápida cuando revise un incidente similar en un rol real.

---

## Objetivos

- Practicar los conceptos que aparecen como excluyentes en ofertas de DevOps junior con foco en Kubernetes: Pods, Deployments, Services, ConfigMaps, Secrets, Ingress, RBAC básico, rollout/rollback.
- Documentar cada troubleshooting real como runbook reproducible, no como blog post.
- Construir muscle memory con `kubectl` sobre un clúster real (kind + Docker Desktop), no simuladores.

---

## Stack

| Componente | Versión |
|---|---|
| Kubernetes (kind) | v1.34.3 |
| Docker Desktop | Kubernetes integrado |
| Ingress Controller | NGINX Ingress Controller (deploy para kind) |
| Metrics | metrics-server v0.7+ |

---

## Estructura

```
k8s-practice-lab/
├── deployment.yaml      # Deployment con 2 réplicas, resource limits, envFrom
├── service.yaml         # Service LoadBalancer (expuesto a localhost por Docker Desktop)
├── configmap.yaml       # Variables no sensibles
├── secret.yaml.example  # Plantilla — NO subir valores reales
├── ingress.yaml         # Ingress apuntando al Service
├── runbook.md               # Procedimientos operativos L2
└── README.md
```

---

## Decisiones técnicas

| Decisión | Razón |
|---|---|
| `kind` en vez de minikube | Ya tenía Docker Desktop; kind es más liviano y crea el clúster dentro de un contenedor Docker |
| `LoadBalancer` en el Service | Docker Desktop lo expone directo a localhost, mientras que `NodePort` en kind falla por limitación de puertos altos (documentado como incidente #1) |
| `envFrom` en vez de `env` individual | Todas las variables del ConfigMap/Secret se necesitan iguales dentro del contenedor; sin renombrado |
| `resource.limits` de 200m CPU / 128Mi memoria | Suficientes para nginx sin tráfico real; `kubectl top pods` confirma consumo real de ~0m CPU / ~12Mi |
| RollingUpdate por defecto (max surge 25%, max unavailable 25%) | Verificado en un rollback real: los pods viejos siguieron sirviendo mientras el nuevo fallaba |
| Ingress con NGINX Controller (variante para kind) | El manifiesto oficial de kind es distinto al de clústeres cloud — un solo comando lo instala |

---

## Cómo lo probé

### 1. Recreación de pod desde ReplicaSet

Borré manualmente uno de los pods del Deployment:

```bash
kubectl delete pod app-1-684d4b6b56-47ftd
kubectl get pods
```

Kubernetes creó un pod nuevo con nombre distinto (`app-1-684d4b6b56-rfklc`) en menos de 15 segundos. Repetí el mismo experimento con un Pod suelto (no gestionado por Deployment) y el pod NO se recreó. Prueba de que Deployment/ReplicaSet garantiza persistencia declarativa.

### 2. Rollback tras despliegue con imagen inexistente

Cambié la imagen del Deployment a `nginx:no-existo` y apliqué:

```bash
kubectl get pods
NAME                     READY   STATUS         AGE
app-1-574666675-2kqtt    0/1     ErrImagePull   11s     ← nuevo, roto
app-1-684d4b6b56-245nb   1/1     Running        20h     ← viejo, sano
app-1-684d4b6b56-2zk8k   1/1     Running        20h     ← viejo, sano
```

Los pods viejos siguieron sirviendo tráfico (verificado con `curl http://localhost/`). Ejecuté `kubectl rollout undo deployment/app-1` y el estado volvió a la última revisión buena sin downtime medible.

### 3. Ingress funcional en localhost

Instalé el NGINX Ingress Controller para kind y apliqué el manifiesto de Ingress. `curl http://localhost/` responde con la página de nginx a través del Ingress Controller → Service → Pod.

### 4. Consumo de recursos vs limits

```bash
kubectl top pods
NAME                     CPU(cores)   MEMORY(bytes)
app-1-684d4b6b56-245nb   0m           12Mi
app-1-684d4b6b56-2zk8k   0m           11Mi
```

Contra los limits de 200m CPU / 128Mi memoria. Margen de sobra — sin riesgo de OOMKilled con la carga actual (nginx sin tráfico real).

---

## Incidentes reales resueltos durante la práctica

### Incidente #1 — NodePort no expuesto a `localhost` con kind + Windows

**Síntoma:** `curl http://localhost:<nodeport>` fallaba con "Could not connect" para todos los Services tipo NodePort.

**Diagnóstico:** `kubectl get endpoints` mostró que los Services sí tenían pods sanos detrás. `kubectl port-forward` sí funcionaba. El problema estaba en la capa entre Docker Desktop y localhost, no en Kubernetes.

**Causa raíz:** kind sobre Docker Desktop en Windows no reenvía automáticamente puertos NodePort altos al host.

**Resolución:** cambié el tipo del Service a `LoadBalancer`. Docker Desktop sí expone estos directo a `localhost:<port>` sin configuración extra.

**Prevención:** en local, `LoadBalancer` o `Ingress` son las opciones prácticas — NodePort queda solo para debugging interno.

### Incidente #2 — `CreateContainerConfigError` en pods nuevos

**Síntoma:** tras aplicar el Deployment con `envFrom`, los pods quedaron en `CreateContainerConfigError` sin arrancar.

**Diagnóstico:**

```bash
kubectl describe pod app-1-684d4b6b56-98krf
```

Sección `Events`:
```
Error: configmap "app-1-config" not found
```

Verificación:
```bash
kubectl get configmaps  # solo aparecía kube-root-ca.crt, no app-1-config
kubectl get secrets     # vacío
```

**Causa raíz:** apliqué solo el `deployment.yaml`, sin aplicar antes los archivos `configmap.yaml` y `secret.yaml`. El Deployment referenciaba recursos que no existían.

**Resolución:**

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl delete pod <pods-en-error>  # ReplicaSet los recrea con las deps ya presentes
```

**Prevención:** aplicar dependencias antes que el recurso que las usa. Considerar Kustomize para ordenar esto automáticamente en el futuro.

---

## Runbook operativo

Ver `runbook.md` para procedimientos L2:

- Monitoreo diario de contenedores
- Triage de pipeline fallido (rollback ejecutado)
- Namespaces y RBAC básico
- Recolección de evidencia para incidentes
- Runbooks de incidentes documentados

---

## Qué NO está aquí (por ahora)

- ArgoCD/GitOps — pendiente (parte siguiente del runbook)
- Kustomize base + overlays — diseñado, no aplicado
- Cluster real en EKS/GKE — este repo es 100% local con kind
- Métricas persistentes (Prometheus/Grafana) — fuera del alcance de esta práctica inicial

---

## Referencias

- [Kubernetes Docs — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Docs — Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [NGINX Ingress Controller para kind](https://kind.sigs.k8s.io/docs/user/ingress/)
