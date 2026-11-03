# Práctica 6: limpiar los logs con un CronJob

### Propósito

Ejecutar el job real de limpieza de logs de la calculadora en Kubernetes. El
backend y el job montan el mismo PVC `calculadora-logs` en `/app/logs`, por lo que
en cada ejecución el CronJob puede vaciar el archivo activo sin perder el contexto
entre ejecuciones del contenedor.

El CronJob demuestra programación, Jobs efímeros, reintentos, política de
concurrencia y almacenamiento compartido. Para la práctica se ejecuta cada dos
minutos; el repositorio usa una ejecución diaria a las 00:05 UTC.

## Preparación

Construye ambas imágenes y reemplaza el marcador `your-registry` si usas otro
registro:

```bash
cd ../calculadora-backend
docker build -t tu-registro/calculadora-backend:latest .
docker push tu-registro/calculadora-backend:latest
cd ../calculadora-job
docker build -t tu-registro/calculadora-job:latest .
docker push tu-registro/calculadora-job:latest
cd ../practice-6
```

## Crear y ejecutar

```bash
minikube addons enable csi-hostpath-driver
kubectl apply -f namespace.yaml
kubectl apply -f calculadora-pvc.yaml
kubectl apply -f calculadora-backend.yaml
kubectl rollout status deployment/calculadora-backend -n practice-6
kubectl apply -f calculadora-cronjob.yaml
kubectl get cronjobs,pvc -n practice-6
```

Genera registros y ejecuta una limpieza manual sin esperar el horario:

```bash
kubectl port-forward service/calculadora-backend 8080:8080 -n practice-6
```

En otra terminal:

```bash
curl "http://localhost:8080/api/v1/calculadora/sumar?numero1=3&numero2=4"
curl "http://localhost:8080/api/v1/calculadora/restar?numero1=9&numero2=1"
```

Comprueba el cron y crea un Job desde su plantilla:

```bash
kubectl create job --from=cronjob/calculadora-request-logs-cleanup limpieza-manual -n practice-6
kubectl wait --for=condition=complete job/limpieza-manual --timeout=120s -n practice-6
kubectl logs job/limpieza-manual -n practice-6
kubectl get jobs,pods -n practice-6
```

El job vacía `api-requests.txt` y crea en el PVC un archivo
`api-requests-cleanup-YYYY-MM-DD.txt` con la hora UTC del borrado.
El backend debe seguir escribiendo el archivo activo cuando recibe nuevas solicitudes.

## Limpieza

```bash
kubectl delete namespace practice-6
```
