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

1. Instalaremos el plugins Prometheus

Opción 1 - Configuración global mediante cURL

```powershell
curl.exe -i -X POST http://localhost:8001/plugins/ `
--data 'name=prometheus' `
--data 'config.per_consumer=true' 
```

Opción 2 - Instalación global  mediante interfaz Konga

Nos dirigimos a la sección plugins de Konga, luego seleccionamos la opción plugins, elegimos la categoría **Analytics & Monitoring** y seleccionamos Prometheus

![Untitled](images/Untitled.png)

**peer consumers**: recopila si debe recopilar metricas por consumidor

![Untitled](images/Untitled%201.png)

2. Se puede verificar las métricas en el path /metrics del Kong Gateway 

```powershell
curl.exe http://localhost:8001/metrics
```

Resultado

![Untitled](images/Untitled%202.png)

### b) Instalación de Prometheus con Grafana

1. Agregar repositorio Helm de **`Prometheus`**

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

2. Instalamos Prometheus con Grafana usando Helm chart de **`kube-prometheus-stack`**

```powershell
helm install monitoring prometheus-community/kube-prometheus-stack --create-namespace --namespace monitoring -f taller_2/prometheus_grafana/values.yaml
```

3. Crear **`ServiceMonitor`** actualizando los valores de Helm chart **`Kong API Gateway`**

```
helm upgrade kong kong/kong -f taller_2/prometheus_grafana/kong_values.yaml
```

4. Verificar si el **`ServiceMonitor`** se creó correctamente

```
kubectl get serviceMonitor -A
```

5. En la imagen se observa que ya está creado el **`ServiceMonitor`**, identificándose con el nombre **`kong-kong`**

![Captura de pantalla de 2024-06-28 12-37-45.png](images/Captura_de_pantalla_de_2024-06-28_12-37-45.png)

6. Luego expondremos Grafana de manera local en el puerto **`8081`**

```
kubectl -n monitoring port-forward service/monitoring-grafana 8081:80
```

7. Desde el navegador ingresamos a la siguiente URL [http://localhost:8081](http://localhost:8081) 

![Pasted image 20240628124704.png](images/Pasted_image_20240628124704.png)

Iniciar sesión con las siguiente credenciales:

- **usuario:** admin
- **password:** prom-operator

### c) Importación de dashboard Kong API Gateway

1. Desde [http://localhost:8081](http://localhost:8081) seleccionamos el apartado **`Dashboard`**. Nos dirigimos al extremo superior derecho, pulsamos sobre **`New`** y luego en **`Import`**

![Pasted image 20240628125431.png](images/Pasted_image_20240628125431.png)

2. Subimos el archivo JSON **`dashboard_kong.json`** que se encuentra en el repositorio del taller [promethus_grafana/dashboard_kong.json](https://github.com/bennu/kong_talleres/blob/main/taller_2/promethus_grafana/dashboard_kong.json)

![Pasted image 20240628125500.png](images/Pasted_image_20240628125500.png)

3. Seleccionamos Prometheus

![Pasted image 20240628125601.png](images/Pasted_image_20240628125601.png)

4. Visualizar gráficos

![Untitled](images/Untitled%203.png)

5. Dentro de este gráficos las métricas a destacar son:

   * Total request per seconds (RPS)
   * RPS/service
   * Total Bandwith
   * Egress per service
   * Ingress per service
   * Kong Proxy Latency all services // per services

## III. Logs (Loggly)

### a) Creación de cuenta en Solarwings Loggly

1. Crear una cuenta [https://www.loggly.com/signup/](https://www.loggly.com/signup/)

![Untitled](images/Untitled%204.png)

2. Ir al sección de LOGS/Source Setup 

![Untitled](images/Untitled%205.png)

3. Luego Customer Tokens y copiar o crear un token

![Untitled](images/Untitled%206.png)

### b) Implementar plugin Loggly 

1. Se debe seleccionar la categoría Logging   y  agregar Loggly  

![Untitled](images/Untitled%207.png)

2. En el campo key se debe agregar el consumer token vinculado a la cuenta de loggly, ademas podemos  dar un tags para identificar el trafico proveniente de Kong api gateway o entidad asociada al plugin . en este caso el Tags es “kong”. 

![Untitled](images/Untitled%208.png)

3. Para poder ver visualizar logs,  debemos generar trafico en el servicios creado en taller anterior, para ello podemos usar curl para enviar peticiones a nuestro servicio

```powershell
 for ($i=1; $i -le 60; $i++) {
     curl.exe -I http://localhost:8000/productos?apikey=<token>
     Start-Sleep -Seconds 1
 }
```

### c) Exploración de logging desde consola de administración loggly 

1. Nos dirigimos a la sección LOGS/Log explorer 

![Untitled](images/Untitled%209.png)

2. Se pueden visualizar los logs en formato JSON que tiene como origen Kong API Gateway

![Untitled](images/Untitled%2010.png)

3. Dentro de estos logs los parámetros a destacar son:

**a) request**

- method
- size
- request_URI
- headers

**b) response**

- status
- headers
- date 

**c) latencies**

- request (ms)
- proxy (ms)
- receive (ms)
  
**d) started at**

**e) client ip**


## IV. Tracing (Zipkin)

### a) Instalación de Zipkin

1. Las configuraciones necesario para el despligue del servidor zipkin se encuentra dentro del archivo deployment.yaml 

```powershell
kubectl apply -f sesion_3_monitoreo/zipkin/deployment.yaml
```

2. Se crea port-forward de Zipkin

```powershell
kubectl port-forward service/zipkin  9411:80
```

3. Una vez expuesto es posible acceder a GUI de Zipkin desde http:/localhost:9411

4. Instalación de plugins Zipkin

```powershell
curl.exe -i -X POST http://localhost:8001/plugins/ `
--data 'name=zipkin' `
--data 'config.http_endpoint=http://zipkin.default/api/v2/spans' `
--data 'config.sample_ratio=1' `
--data 'config.include_credential=true' 
```

4. Generar tráfico 

```powershell
 for ($i=1; $i -le 60; $i++) {
     curl.exe -I http://localhost:8000/productos?apikey=<token>
     Start-Sleep -Seconds 1
 }
```

Resultado

![Untitled](images/Untitled%2011.png)
