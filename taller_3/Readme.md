# Taller 3 - Seguridad

**Objetivo:** Implementar modelo de API Security mediante herramientas de autenticación, autorización (OAuth 2.0) y prevención de amenazas (rate limiting y boot detection)

**Nota:** Como este levantamiento sucede a nivel local en el momento que se configure el plugin **`OAuth 2.0 Authentication`** será Kong API Gateway el que tomará la función de **`Authorization Server`**. En un ambiente productivo, es recomendado usar un servicio externo como Identity Provider (idP) como Okta ó MiniOrange

### I. Configuración de OAuth 2.0

1. Se crea servicio de nombre “**usuarios**”

```powershell
curl.exe -X POST `
  --url 'http://localhost:8001/services' `
  --data 'name=usuarios' `
  --data 'url=https://dummyjson.com/user'
```

1. Posteriormente se crea un route 

```powershell
curl.exe -X POST `
  --url 'http://localhost:8001/services/usuarios/routes' `
  --data 'paths[]=/usuarios'
```

1. Se configura el plugin OAuth 2.0 Authentication

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

1. Se valida API con curl [http://localhost:8000/usuarios](http://localhost:8000/usuarios). Debiésemos obtener un código 401 Unauthorized

```powershell
curl.exe localhost:8000/usuarios
```

Resultado

```powershell
{
    'error': 'invalid_request',
    'error_description': 'The access token is missing'
}
```

1. Se crea un consumer

```powershell
curl.exe -X POST `
  --url 'http://localhost:8001/consumers/' `
  --data 'username=app_tercero'
```

1. Se crean credenciales OAuth 2.0 para el consumer “**app_tercero**”

```powershell
curl.exe -X POST `
  --url 'http://localhost:8001/consumers/app_tercero/oauth2/' `
  --data 'name=Oauth2 App example' `
  --data 'client_id=oauth2-demo-client-id' `
  --data 'client_secret=oauth2-demo-client-secret' `
  --data 'redirect_uris[]=http://localhost:8000/usuarios' `
  --data 'hash_secret=true'
```

### II. Flujo Authorization Code

1. El consumer **app_tercero** envía petición de autorización  

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

1. Otorgamiento de autorización **app_tercero**. Este debiese devolver una URL con el Authorization code 

Ejemplo:

```powershell
{
  'redirect_uri': 'http://localhost:8000/?code=jvnD1XBgFqZuqT2OlbcXpDiOlFkx75bU'
}
```

1. Se realiza intercambio de Authorization code por Access Token

```powershell
curl.exe -X POST https://localhost:8443/usuarios/oauth2/token `
--data 'grant_type=authorization_code' `
--data 'client_id=oauth2-demo-client-id' `
--data 'client_secret=oauth2-demo-client-secret' `
--data 'code=jvnD1XBgFqZuqT2OlbcXpDiOlFkx75bU' `
--insecure
```

1. Respuesta  intercambio de  Authorization code por  Access Token

se obtiene un :   Access Token , refresh token y tiempo de expiracion del Acess Token

```powershell
{
'refresh_token': 'LqJW6mVH4XsNZnoQ5fYbjngBsbUJPVPh',
'token_type': 'bearer',
'access_token': 'BZiZzJVEuP2mgNvZZBr0mgbRtKsdqgZf',
'expires_in': 7200
}
```

Paso 5:  Validar  Access Token

```powershell
curl.exe -X GET `
--url '[http://localhost:8000/](http://localhost:8000/demo)usuarios' `
--header 'Authorization: Bearer?? <ACCESS_TOKEN>'
```

resultados

```powershell

```

Paso 6:  El Access Token suele tener un tiempo de expiración asociado y eso obliga a tener un flujo de renovación de Token, podemos probar este flujo de la siguiente manera 

```powershell
	curl.exe -X POST `
	  --url 'https://localhost:8443/usuarios/oauth2/token' `
	  --data 'grant_type=refresh_token' `
	  --data 'client_id=oauth2-demo-client-id' `
	  --data 'client_secret=oauth2-demo-client-secret' `
	  --data 'refresh_token=<REFRESH_TOKEN>' `
	  --insecure

```

### Protección de api gateway

rate limiting

```powershell
curl.exe -X POST http://localhost:8001/services/usuarios/plugins `
   --data 'name=rate-limiting' `
   --data 'config.minute=3' `
   --data 'config.hour=10000' `
   --data 'config.policy=local'
```

Prueba 

```powershell
for ($i=1; $i -le 10; $i++) {
     curl.exe -X GET --url 'http://localhost:8000/usuarios/auth/login' --header 'Authorization: Bearer token'
     Start-Sleep -Seconds 1
}
```

Resultado

```powershell
{
  'message':'API rate limit exceeded'
}
```

Podemos crear un dashboard y visualizar alertas

![Untitled](images/Untitled.png)

request size limiting

```powershell
curl.exe -X POST http://localhost:8001/services/usuarios/plugins `
    --data 'name=request-size-limiting' `
    --data 'config.allowed_payload_size=9' `
    --data 'config.size_unit=kilobytes' `
    --data 'config.require_content_length=false' 
```

prueba

```powershell
curl.exe -X POST --url 'http://localhost:8000/usuarios' --data '@payload.json' --header 'Authorization: Bearer YJ3NKhFGxbd1wbvul8oXfQO26xejffWw'
```

Resultado 

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

Bot detection

```powershell
curl.exe -X POST http://localhost:8001/services/oauth2-test/plugins `
   --data 'name=bot-detection' `
   --data 'config.deny=postman'
```

Prueba

```powershell
curl.exe -H  'user-agent: postman' localhost:8000/usuarios
```

Resultado

```powershell
{
  'message':'Forbidden'
}
```
