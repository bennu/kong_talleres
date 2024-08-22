# Taller 3 - Seguridad

**Objetivo:** Implementar modelo de API Security mediante herramientas de autenticación, autorización (OAuth 2.0) y prevención de amenazas (rate limiting, request size limiting y bot detection)

**Nota:** Como este levantamiento sucede a nivel local en el momento que se configure el plugin **`OAuth 2.0 Authentication`** será Kong API Gateway el que tomará la función de **`Authorization Server`**. En un ambiente productivo, es recomendado usar un servicio externo como Identity Provider (idP) como Okta ó MiniOrange

## I. Pre-requisitos:
Para poder iniciar el taller se necesita exponer los siguientes servicios de manera local:

**a) API Gateway**

```powershell
kubectl port-forward service/kong-kong-proxy 8000:80 8443:443
```

**b) Admin API**

```powershell
kubectl port-forward service/kong-kong-admin 8001:8001 
```

## II. Configuración de OAuth 2.0

1. Crearemos un services de nombre **`usuarios`**

```powershell
curl.exe -X POST `
--url 'http://localhost:8001/services' `
--data 'name=usuarios' `
--data 'url=https://dummyjson.com/user'
```

2. Posteriormente creamos un route y le definiremos un path de nombre **`/usuarios`**

```powershell
curl.exe -X POST `
--url 'http://localhost:8001/services/usuarios/routes' `
--data 'paths[]=/usuarios'
```

3. Se configura el plugin **`OAuth 2.0 Authentication`** con los siguientes parámetros

```powershell
curl.exe -X POST `
--url http://localhost:8001/services/usuarios/plugins/ `
--data 'name=oauth2' `
--data 'config.scopes[]=email' `
--data 'config.scopes[]=phone' `
--data 'config.scopes[]=address' `
--data 'config.mandatory_scope=true' `
--data 'config.provision_key=oauth2-demo-provision-key' `
--data 'config.enable_authorization_code=true' `
--data 'config.enable_client_credentials=true' `
--data 'config.enable_implicit_grant=true' `
--data 'config.enable_password_grant=true'
```

4. Se valida API mediante cURL [http://localhost:8000/usuarios](http://localhost:8000/usuarios). Debiésemos obtener un código del tipo **`401 Unauthorized`**

```powershell
curl.exe localhost:8000/usuarios
```

Resultado de la ejecución:
```powershell
{
    'error': 'invalid_request',
    'error_description': 'The access token is missing'
}
```

5. Crearemos un consumer de nombre **`app_tercero`**

```powershell
curl.exe -X POST `
--url 'http://localhost:8001/consumers/' `
--data 'username=app_tercero'
```

6. Se crean credenciales OAuth 2.0 para el consumer **`app_tercero`**

```powershell
curl.exe -X POST `
--url 'http://localhost:8001/consumers/app_tercero/oauth2/' `
--data 'name=Oauth2 App example' `
--data 'client_id=oauth2-demo-client-id' `
--data 'client_secret=oauth2-demo-client-secret' `
--data 'redirect_uris[]=http://localhost:8000/usuarios' `
--data 'hash_secret=true'
```

## III. Flujo Authorization Code

1. El consumer **`app_tercero`** enviará petición de autorización  

```powershell
curl.exe -X POST `
--url 'https://localhost:8443/usuarios/oauth2/authorize' `
--data 'response_type=code' `
--data 'scope=email address' `
--data 'client_id=oauth2-demo-client-id' `
--data 'provision_key=oauth2-demo-provision-key' `
--data 'authenticated_userid=authenticated_tester' `
--insecure
```

2. Luego realizamos el otorgamiento de autorización al consumer **`app_tercero`**. Este debiese devolver una URL con el **`Authorization code`**

 Ejemplo de la ejecucion del comando anterior:

```powershell
{
  'redirect_uri': 'http://localhost:8000/?code=<Authorization code>'
}
```

3. Se realiza intercambio de Authorization code por Access Token para que el consumer **`app_tercero`** pueda acceder a los recursos

```powershell
curl.exe -X POST https://localhost:8443/usuarios/oauth2/token `
--data 'grant_type=authorization_code' `
--data 'client_id=oauth2-demo-client-id' `
--data 'client_secret=oauth2-demo-client-secret' `
--data 'code=<Authorization Code devuelto en la URL>' `
--insecure
```

4. Realizado este intercambio obtendremos:

- Access Token
- Refresh Token
- Tiempo de expiración del Access Token

Ejemplo de respuesta de la ejecución del paso #3
```powershell
{
'refresh_token': '<Refresh Token>',
'token_type': 'bearer',
'access_token': '<Access Token>',
'expires_in': 7200
}
```

5. Para validar el **`Access Token`** debiésemos apuntar en el **`--header 'Authorization: Bearer`** el token generando en el paso #3

```powershell
curl.exe -X GET `
--url 'http://localhost:8000/usuarios' `
--header 'Authorization: Bearer <Access Token devuelto de la ejecución en el paso #3>'
```

6. El **`Access Token`** suele tener un tiempo de expiración asociado y eso obliga a tener un flujo de renovación del Token. Podemos probar este flujo de la siguiente manera 

> Nota: La función del **`Refresh Token`** es renovar el **`Access Token`** en el momento que este expire
> 

```powershell
curl.exe -X POST `
--url 'https://localhost:8443/usuarios/oauth2/token' `
--data 'grant_type=refresh_token' `
--data 'client_id=oauth2-demo-client-id' `
--data 'client_secret=oauth2-demo-client-secret' `
--data 'refresh_token=<Refresh Token devuelto de la ejecución en el paso #3>' `
--insecure
```

## IV. Protección de Kong API Gateway

#### a) Rate Limiting

1. Asignamos al plugin **`rate-limiting`** un número máximo de 5 peticiones por minutos

```powershell
curl.exe -X POST http://localhost:8001/services/usuarios/plugins `
--data 'name=rate-limiting' `
--data 'config.minute=5' `
--data 'config.policy=local'
```

2. Generamos tráfico repitiendo el comando por 6 veces

```powershell
for ($i=1; $i -le 10; $i++) {
     curl.exe -X GET --url 'http://localhost:8000/usuarios/auth/login' --header 'Authorization: Bearer token'
     Start-Sleep -Seconds 1
}
```

3. En la 6ta petición debería mostrarse **`429 Too Many Requests`** como código de estado

```powershell
HTTP/1.1 429 Too Many Requests
Date: Wed, 21 Aug 2024 17:04:14 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Retry-After: 46
X-RateLimit-Remaining-Minute: 0
X-RateLimit-Limit-Minute: 5
RateLimit-Remaining: 0
RateLimit-Limit: 5
RateLimit-Reset: 46
Content-Length: 41
vary: Origin
Access-Control-Allow-Origin: mockbin.com
Access-Control-Allow-Credentials: true
X-Kong-Response-Latency: 2
Server: kong/2.8.5

{
  "message":"API rate limit exceeded"
}
```

#### b) Request Size Limiting

1. Definimos en el plugin **`request-size-limiting`** una cantidad máxima de **`9 kilobytes`** como tamaño máximo de la solicitud

```powershell
curl.exe -X POST http://localhost:8001/services/usuarios/plugins `
--data 'name=request-size-limiting' `
--data 'config.allowed_payload_size=9' `
--data 'config.size_unit=kilobytes' `
--data 'config.require_content_length=false' 
```

2. Enviamos una solicitud enviando un archivo **`payload.json`** con un tamaño de **`282 kilobytes`** superando la regla previamente definida

```powershell
curl.exe -X POST --url 'http://localhost:8000/usuarios' --data '@taller_3/payload.json' --header 'Authorization: Bearer YJ3NKhFGxbd1wbvul8oXfQO26xejffWw'
```

3. Obtendremos un **`413 Request Entity Too Large`** como código de estado, mostrando que la solicitud supera la regla de los **`9 kilobytes`**

```powershell
HTTP/1.1 413 Request Entity Too Large
Date: Mon, 29 Jul 2024 18:41:59 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 45
X-Kong-Response-Latency: 0
Server: kong/2.8.5

{
  'message':'Request size limit exceeded'
}
```

#### c) Bot Detection

1. En el parámetro **`config.deny`** del plugin **`bot-detection`** apuntamos **`postman`** como un cliente que debiese clasificarse en la lista de clientes no admitidos

```powershell
curl.exe -X POST http://localhost:8001/services/usuarios/plugins `
--data 'name=bot-detection' `
--data 'config.deny=postman'
```

2. Se realiza una solicitud simulando ser el cliente **`postman`**

```powershell
curl.exe -H  'user-agent: postman' localhost:8000/usuarios
```

3. Se mostrará que la respuesta a la solicitud del cliente **`postman`** ha sido rechazada

```powershell
{
  'message':'Forbidden'
}
```
