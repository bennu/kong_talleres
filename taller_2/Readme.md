# Taller 2 - Observabilidad

**Objetivo:** Levantar un flujo de observabilidad (métricas, logs y tracing) para visibilizar el tráfico y rendimiento de Kong API Gateway


## I. Pre-requisitos:
Para poder iniciar el taller se necesita exponer los siguientes servicios de manera local:

**a) API Gateway**

```powershell
kubectl port-forward service/kong-kong-proxy 8000:80 
```

**b) Admin API**

```powershell
kubectl port-forward service/kong-kong-admin 8001:8001 
```

**c) Konga**

```powershell
kubectl port-forward service/konga 8080:80 
```

## II. Métricas (Prometheus)

### a) Instalación y configuración de plugin Prometheus

1. Instalación de plugins Prometheus

Opción 1 - Configuración global mediante cURL

```powershell
curl.exe -i -X POST http://localhost:8001/plugins/ `
--data 'name=prometheus' `
--data 'config.per_consumer=true' 
```

Opción 2 - Instalación global  mediante interfaz Konga

Nos dirigimos a la sección plugins de Konga, luego seleccionamos la opción plugins, elegimos la categoría Análisis y Monitoreo y seleccionamos el plugin prometheus

![Untitled](images/Untitled.png)

**peer consumers**: recopila si debe recopilar metricas por consumidor

![Untitled](images/Untitled%201.png)

2. Se puede verificar las metricas en el path /metrics del Kong Gateway 

```powershell
curl.exe http://localhost:8001/metrics
```

Resultado

![Untitled](images/Untitled%202.png)

### b) Instalación de Prometheus con Grafana

1. Agregar helm chart

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

2. Instalación de Prometheus con Grafana usando kube-prometheus-stack

```powershell
helm install monitoring prometheus-community/kube-prometheus-stack --create-namespace --namespace monitoring -f taller_2/promethus_grafana/values.yaml
```

3. Creación de service monitor mediante helm Kong API Gateway

```
helm upgrade kong kong/kong -f taller_2/promethus_grafana/kong_values.yaml
```

4. Verificar si el serviceMonitor relacionado a kong se creo

```
kubectl get serviceMonitor -A
```

5. En la imagen se observa que ya está creado el service-monitor

![Captura de pantalla de 2024-06-28 12-37-45.png](images/Captura_de_pantalla_de_2024-06-28_12-37-45.png)

6. Exponer Grafana de forma local

```
kubectl -n monitoring port-forward service/monitoring-grafana 8081:80
```

7. Ingresar a [http://localhost:8081](http://localhost:8081) desde el navegador 

![Pasted image 20240628124704.png](images/Pasted_image_20240628124704.png)

Iniciar sesion con las siguiente credenciales:
**usuario: admin
password: prom-operator**

### c) Importación de dashboard Kong API Gateway

1. Seleccionar la  sección de dashboard para luego presionar import 

![Pasted image 20240628125431.png](images/Pasted_image_20240628125431.png)

2. Subir JSON que está en el repositorio

![Pasted image 20240628125500.png](images/Pasted_image_20240628125500.png)

3. Seleccionar Prometheus

![Pasted image 20240628125601.png](images/Pasted_image_20240628125601.png)

4. Visualizar gráficos

![Untitled](images/Untitled%203.png)

5. Dentro de este gráficos las métricas a destacar son:

   *
   *
   *

## III. Logs (Loggly)

Creacion de cuenta en solarwings loggly

crear una cuenta [https://www.loggly.com/signup/](https://www.loggly.com/signup/)

![Untitled](images/Untitled%204.png)

Ir al sección de LOGS/Source Setup 

![Untitled](images/Untitled%205.png)

Luego Customer Tokens y copiar o crear un token

![Untitled](images/Untitled%206.png)

Implementan plugin loggly 

Se debe seleccionar la categoría Logging   y  agregar Loggly  

![Untitled](images/Untitled%207.png)

En  el campo key se debe agregar el consumer token vinculado a la cuenta de loggly, ademas podemos  dar un tags para identificar el trafico proveniente de Kong api gateway o entidad asociada al plugin . en este caso el Tags es “kong”. 

![Untitled](images/Untitled%208.png)

Para poder ver visualizar logs,  debemos generar trafico en el servicios creado en taller anterior, para ello podemos usar curl para enviar peticiones a nuestro servicio
```powershell
 for ($i=1; $i -le 60; $i++) {
     curl.exe -I http://localhost:8000/productos?apikey=<token>
     Start-Sleep -Seconds 1
 }
```

Exploración de logging en consola de administración loggly 

Si vamos a la sección LOGS/Log explorer 

![Untitled](images/Untitled%209.png)

 

Se puede visualizar los logs en formato Json que tiene como origen el kong Api Gateway

![Untitled](images/Untitled%2010.png)

[https://konghq.com/blog/learning-center/what-is-api-security](https://konghq.com/blog/learning-center/what-is-api-security)

dashboard  

## IV. Tracing (Zipkin)

Instalación de zipkin

Las configuraciones necesario para el despligue del servidor zipkin se encuentra  dentro del archivo deployment.yaml 

```powershell
kubectl apply -f sesion_3_monitoreo/zipkin/deployment.yaml
```

exponer gui zipkin

```powershell
kubectl port-forward service/zipkin  9411:80 &
```

La interfaz de zipkin se puede acceder a traves de http:/localhost:9411

Instalación de plugins Zipkin

```powershell
curl -X POST http://localhost:8001/plugins/ \
   --header "accept: application/json" \https://konghq.com/blog/learning-center/what-is-api-security
   --header "Content-Type: application/json" \
   --data '
   {
 "name": "zipkin",
 "config": {
   "http_endpoint": "http://zipkin.default/api/v2/spans",
   "sample_ratio": 1,
   "include_credential": true
 }
}
   '
```

generar trafico 

```powershell
 for ($i=1; $i -le 60; $i++) {
     curl.exe -I http://localhost:8000/productos?apikey=<token>
     Start-Sleep -Seconds 1
 }
```

Resultado

![Untitled](images/Untitled%2011.png)
