# Taller 1b - Administración

**Objetivo:** Crear entidades como Routes, Services, Plugins y Consumers usando distintas estrategias de administración (Konga, cURL y deck)

Para poder iniciar el taller se necesita exponer los siguientes servicios de manera local, exponiendo API Gateway y Admin API de Kong

**a) API Gateway**

```bash
kubectl port-forward service/kong-kong-proxy 8000:80 
```

**b) Admin API**

```bash
kubectl port-forward service/kong-kong-admin 8001:8001 
```

**c) Konga**

```bash
kubectl port-forward service/konga 8080:80 
```

## I. Konga

En este ejemplo ocuparemos el API https://dummyjson.com/products, que es una API de desarrollo que muestra datos de prueba. Para esto se expondrá esta API desde la interfaz de Konga

### **A) Services**

1. Crearemos un services llamado **products-service**. Importante tener en cuenta los siguientes parámetros:

- **Name:  products-service**
- **URL:** Se ingresa el endpoint https://dummyjson.com/products para que se muestre desde Kong

![Untitled](images/Untitled.png)

### **B) Routes**

1. Una vez creado el services, podemos visualizar los detalles haciendo click sobre el nombre de este

![Untitled](images/Untitled%201.png)

2. Dentro del services **products-service**, nos dirigimos el apartado de **routes** y agregamos un nuevo **route**

![Untitled](images/Untitled%202.png)

Los valores que se deben considerar son los siguientes:

- **Name:** Corresponde al nombre que se le va a otorgar al route, en este caso los llamaremos **productos**
- **Paths:**  Aquí podemos definir distintos paths independiente de ello todos apuntan al mismo route. Importante que al momento de agregar nuevos paths se debe pulsar “ENTER” para agregarlos. En este caso agregaremos un solo path de nombre **“/productos”**

![Untitled](images/Untitled%203.png)

![Untitled](images/Untitled%204.png)



### **C) Plugins**

Una vez expuesto el API, es necesario definir como será consumido (tratamiento)

Lo primero que definiremos será el control del tráfico, limitando los orígenes de los que consumirán nuestro servicio ó API expuesto desde Kong y como estos se autenticarán. Para ello usaremos los plugins **rate-limit**, **Cors**, **key-auth** y **ACL**. Estos los agregaremos a nivel de la entidad **services** y se definirá un policy del tipo local para todos. 


### **- rate-limit:**

1. Nos dirigiremos al services **products-service**, luego seleccionamos la opción plugins, elegimos la categoría Traffic Control y seleccionamos el plugin **rate-limit**.  

![Untitled](images/Untitled%205.png)

Para efectos de prueba se deben configurar los siguientes parámetros:

- **minute:** 5 
- **limit by:** consumer
- **policy:** local

![Untitled](images/Untitled%206.png)

![Untitled](images/Untitled%207.png)

Podemos validar el comportamiento enviando 6 peticiones mediante cURL 

```bash
curl.exe -I http://localhost:8000/productos
#for _ in {1..6}; do {curl -I http://localhost:8000/productos; sleep 1;}  done
```

En la 6ª petición debería mostrarse el error 429 como código de estado, esto significa que llegó al límite definido de peticiones por minutos. 

### **- Cors**

1. Para agregarlo nos aseguramos de estar posicionados en el services **products-service**, seleccionamos la opción plugins y pulsamos el botón **ADD PLUGIN** 

![Untitled](images/Untitled%208.png)

2. Seleccionamos la categoría **Security** y el plugin llamado **Cors**

![Untitled](images/Untitled%209.png)

3. Los valores a considerar son los siguientes:

**origins:** Esta opción definirá los orígenes de las solicitudes entrantes que puede aceptar nuestro servicio ó API. Para esta prueba, definiremos asterisco como valor para que permita que puedan acceder desde cualquier origen. 

> Nota:  Para agregar valores al campo “origins” se debe presionar ENTER
> 

![Untitled](images/Untitled%2010.png)

### **- key-auth**

1. Para agregar el plugin **key-auth** nos aseguraremos de estar en la entidad services **products-service**, nos dirigimos a la opción plugins y elegimos la categoría **Authentication**, tal como lo hicimos con el plugins **Cors**

![Untitled](images/Untitled%2011.png)

2. Los valores a considerar son los siguientes:
 
- **key names:** corresponde al nombre de parámetro por el cual el cliente enviará un token asociado a uno de los consumers

**Ejemplo:** api-key = token asociado al consumers

Para este caso el campo key names, lo dejaremos vacío para que tome “**"apikey”** como nombre por defecto  

![Untitled](images/Untitled%2012.png)

### **- ACL**

El plugin ACL (autorización) forma parte de la gestión de identidad y de acceso a un sistema. El ACL define el tipo de acceso para que se pueda acceder a un determinado servicio ó API. Por ejemplo, un consumer que tenga creado un API Key, que no pertenezca a un grupo de consumers y que intente acceder al services **products-service**, no le va a ser posible, puesto que la regla ACL está configurada para que solo permita la autorización a un grupo de consumers.

En este caso configuraremos el plugin ACL a nivel de **products-service** 

![Untitled](images/Untitled%2013.png)

Importante tener en cuenta los siguientes parámetros:

- **allow:** admin (hace referencia a un grupo de consumers), en este caso se va a permitir solo al grupo admin 
- **deny:**  permite denegar el acceso a un grupo, tal como indica su nombre
- **hide groups header:**  determina si se envía el encabezado X-Consumer-Groups a servicio ascendente

![Untitled](images/Untitled%2014.png)

### D) Consumers

Como definimos la manera de como se autenticarán a nuestro servicio ó API mediante key-auth, es necesario crear un consumer al que asociaremos un api-key para que autorice su acceso al services **products-service**

1. Crear un consumer de nombre **app**

![Untitled](images/Untitled%2015.png)

![Untitled](images/Untitled%2016.png)

2. Dentro del consumer **app**, nos dirigiremos a la opción Credentials y desde ahí creamos un api-key

![Untitled](images/Untitled%2017.png)

3. Para este ejemplo dejaremos el campo key vacío para que Kong genere un valor aleatorio 

![Untitled](images/Untitled%2018.png)

4. Seleccionamos el valor de api-key generado para confirmar si el consumer puede acceder al servicio ó API

![Untitled](images/Untitled%2019.png)


> Nota: Para este ejemplo se agregará el consumer **app** al grupo consumers (**admin**) permitidos por el ACL
> 


5. Agregar consumer **app** a un grupo de consumers

![Untitled](images/Untitled%2020.png)


6. El grupo de consumers se llama `admin`

![Untitled](images/Untitled%2021.png)

7. Para confirmar que podemos acceder al servicio. Desde el navegador apuntamos la siguiente URL  **`http://localhost:8000/productos?apikey=<api key>`**, en ella se debiese mostrar un listado de productos en formato JSON tal como se observa en la imagen

![Untitled](images/Untitled%2022.png)

## II. cURL

cURL es una herramienta de línea de comandos que interactuar con la Admin API de Kong, permitiendo realizar solicitudes HTTP.

1. Crear un nuevo route con nombre “lista-productos” asociado al service products-service que sumará al route “productos” que creamos desde Konga

```powershell
$ curl -i -X POST http://localhost:8001/services/products-service/routes \
     --data 'paths[]=/lista-productos' \
     --data name=products_route
```

Los parámetros a destacar en este ejemplo son:

- **paths:**  Contiene la  ruta por donde se va a exponer el servicio al consumidor
- **name:**  Nombre para identificar el route que se va crear

2. Restringir los métodos configurados de CORS a service products-service, limitándolos solo a GET y POST. Para esto se debe buscar el ID del plugin Cors asociado al servicio

```bash
curl -X GET http://localhost:8001/services/products-service/plugins 
```

![Untitled](images/Untitled%2023.png)

```bash
curl -X PATCH  --url http://localhost:8001/services/products-service/plugins/<plugin id> \
   --header "accept: application/json" \
   --header "Content-Type: application/json" \
   --data '
 {
  "name": "cors",
  "config": {
    "origins": [
      "*"
    ],
    "methods": [
      "GET",
      "POST"
    ],
    "headers": [
      "Accept",
      "Accept-Version",
      "Content-Length",
      "Content-MD5",
      "Content-Type",
      "Date",
      "X-Auth-Token"
    ],
    "exposed_headers": [
      "X-Auth-Token"[https://konghq.com/blog/learning-center/what-is-api-security](https://konghq.com/blog/learning-center/what-is-api-security)
    ],
    "credentials": true,
    "max_age": 3600
  }
}
   '
```

3. Modificar los parámetros del plugins rate-limit cambiando la frecuencia de consulta al servicio de 5 a 10 min 

```bash
curl -X PATCH --url http://localhost:8001/services/products-service/plugins/<plugin id> \
   --header "accept: application/json" \
   --header "Content-Type: application/json" \
   --data '
   {
 "name": "rate-limiting",
 "config": {
   "minute": 10,
   "policy": "local"
 }
}
   '
```

4. Podemos observar que la última petición arroja un código de estado **“429 too many request”**

```bash
for _ in {1..11}; do {curl -I http://localhost:8000/productos\?apikey\<token>; sleep 1;}  done
```

5. Configurar nuevo consumer con el nombre **dev** 

```bash
curl -i -X POST http://localhost:8001/consumers/ \
  --data username=dev
```

**Recordar que ya tenemos otro consumer de nombre **app**

6. Si creamos un API key sin ningún parámetro, este generará un valor aleatorio como token

```bash
curl -i -X POST http://localhost:8001/consumers/app/key-auth
```

7. Otra opción, sería setear un token en caso de ser necesario

```bash
curl -i -X POST http://localhost:8001/consumers/app/key-auth \
  --data key=top-secret-key
```

8. Agregar **consumer** a un grupo de consumers

```bash
 curl -X POST http://localhost:8001/consumers/{CONSUMER}/acls \
    --data "group=dev"
```

9. En este caso, es importante considerar que el consumer dev no podrá acceder con su apikey para consumir el servicio **service-products** dado la regla ACL que solo otorga la autorización al grupo de consumers **admin**. Para esto agregaremos el consumer **dev** al grupo consumers **dev** al listado de consumers permitidos

```bash
curl -X PATCH --url http://localhost:8001/services/products-service/plugins/<id plugin> \
   --header "accept: application/json" \
   --header "Content-Type: application/json" \
    --data '
    {
  "name": "acl",
  "config": {
    "allow": [
      "admin",
      "dev"
    ],                        
    "hide_groups_header": false
  }
}    
    '
```

10. Podemos validar si podemos acceder ingresando de la siguiente forma  

```
curl http://localhost:8000/productos?apikey=<apikey generado anteriormente>
```

## III. deck

1. Se expone Admin API

```powershell
kubectl port-forward service/kong-kong-admin 8001:8001 &
```

2. Generar un dump de las configuraciones de Kong , se va crear un archivos llamado “kong.yaml”

```powershell
λ ~/ deck dump         
Info: 'deck dump' functionality has moved to 'deck gateway dump' and will be removed
in a future MAJOR version of deck. Migration to 'deck gateway dump' is recommended.
   Note: - see 'deck gateway dump --help' for changes to the command
         - the default changed from 'kong.yaml' to '-' (stdin/stdout)

```

3. Abrir archivos generado por dump y buscar el route productos

![Untitled](images/Untitled%2024.png)

4. Cambiar path /productos por /products

![Untitled](images/Untitled%2025.png)

5. Para aplicar los cambios, ejecutar el siguiente comando

```powershell
deck sync
```

**Resultado**

```powershell
updating route productos  {
   "https_redirect_status_code": 426,
   "id": "373ca058-e0c4-4d3d-9380-97e65b605686",
   "name": "productos",
   "path_handling": "v1",
   "paths": [
-    "/productos"
+    "/products"
   ],
   "preserve_host": false,
   "protocols": [
     "http",
     "https"
   ],
   "regex_priority": 0,
   "request_buffering": true,
   "response_buffering": true,
   "service": {
     "id": "508d08e1-adf1-4e6a-8e07-0bcb18da5829",
     "name": "products-service"
   },
   "strip_path": true
 }

Summary:
  Created: 0
  Updated: 1
  Deleted: 0

```

6. Podemos validar el cambio ingresando a path /products

```bash
curl http://localhost:8000/products\?apikey=<Token>
```
