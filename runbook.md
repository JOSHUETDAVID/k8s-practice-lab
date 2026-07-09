1. **Síntoma:** `kubectl get pods` muestra `CreateContainerConfigError`
2. **Diagnóstico:** `kubectl describe pod <nombre>` → revisar sección `Events`
3. **Causas comunes:** ConfigMap/Secret referenciado no existe, o nombre mal escrito
4. **Verificación:** `kubectl get configmaps` y `kubectl get secrets` — confirmar existencia real
5. **Resolución:** aplicar el recurso faltante, luego `kubectl delete pod` para forzar recreación
6. **Prevención:** validar con `kubectl diff` antes de aplicar en producción
7. **Evidencia:** dejar el estado del pod descrito en un archivo .txt para poder hacer auditorias con `kubectl describe pod <nombre-del-pod-id> > evidencia.txt`
8.  Guardar el archivo y enviar si es el caso
