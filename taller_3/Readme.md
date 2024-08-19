# Taller 3 - Seguridad

**Objetivo:** Implementar modelo de API Security mediante herramientas de autenticación, autorización (OAuth 2.0) y prevención de amenazas (rate limiting y boot detection)

**Nota:** Como este levantamiento sucede a nivel local en el momento que se configure el plugin **`OAuth 2.0 Authentication`** Kong API Gateway tomará la función de **`Authorization Server`**. En un ambiente productivo, es recomendado usar un servicio externo como Identity Provider (idP) como Okta, MiniOrange

### I. Configuración de OAuth 2.0

1. Se crea servicio de nombre “**usuarios**”

```jsx
curl -X POST \
  --url "http://localhost:8001/services" \
  --data "name=usuarios" \
  --data "url=https://dummyjson.com/user"
```

1. Posteriormente se crea un route 

```jsx
curl -X POST \
  --url "http://localhost:8001/services/usuarios/routes" \
  --data 'paths[]=/usuarios'
```

1. Se configura el plugin OAuth 2.0 Authentication

```jsx
curl -X POST \
  --url http://localhost:8001/services/usuarios/plugins/ \
  --data "name=oauth2" \
  --data "config.scopes[]=email" \
  --data "config.scopes[]=phone" \
  --data "config.scopes[]=address" \
  --data "config.mandatory_scope=true" \
  --data "config.provision_key=oauth2-demo-provision-key" \
  --data "config.enable_authorization_code=true" \
  --data "config.enable_client_credentials=true" \
  --data "config.enable_implicit_grant=true" \
  --data "config.enable_password_grant=true"
```

1. Se valida API con curl [http://localhost:8000/usuarios](http://localhost:8000/usuarios). Debiésemos obtener un código 401 Unauthorized

```bash
curl localhost:8000/usuarios
```

Resultado

```jsx
{
    "error": "invalid_request",
    "error_description": "The access token is missing"
}
```

1. Se crea un consumer

```jsx
curl -X POST \
  --url "http://localhost:8001/consumers/" \
  --data "username=app_tercero"
```

1. Se crean credenciales OAuth 2.0 para el consumer “**app_tercero**”

```jsx
curl -X POST \
  --url "http://localhost:8001/consumers/app_tercero/oauth2/" \
  --data "name=Oauth2 App example" \
  --data "client_id=oauth2-demo-client-id" \
  --data "client_secret=oauth2-demo-client-secret" \
  --data "redirect_uris[]=http://localhost:8000/usuarios" \
  --data "hash_secret=true"
```

### II. Flujo Authorization Code

1. El consumer **app_tercero** envía petición de autorización  

```jsx
curl -X POST \
  --url "https://localhost:8443/usuarios/oauth2/authorize" \
  --data "response_type=code" \
  --data "scope=email address" \
  --data "client_id=oauth2-demo-client-id" \
  --data "provision_key=oauth2-demo-provision-key" \
  --data "authenticated_userid=authenticated_tester" \
  --insecure
```

1. Otorgamiento de autorización **app_tercero**. Este debiese devolver una URL con el Authorization code 

Ejemplo:

```jsx
{
  "redirect_uri": "http://localhost:8000/?code=jvnD1XBgFqZuqT2OlbcXpDiOlFkx75bU"
}
```

1. Se realiza intercambio de Authorization code por Access Token

```jsx
curl -X POST \
--url "https://localhost:8443/usuarios/oauth2/token" \
--data "grant_type=authorization_code" \
--data "client_id=oauth2-demo-client-id" \
--data "client_secret=oauth2-demo-client-secret" \
--data "code=jvnD1XBgFqZuqT2OlbcXpDiOlFkx75bU" \
--insecure
```

1. Respuesta  intercambio de  Authorization code por  Access Token

se obtiene un :   Access Token , refresh token y tiempo de expiracion del Acess Token

```jsx
{
"refresh_token": "LqJW6mVH4XsNZnoQ5fYbjngBsbUJPVPh",
"token_type": "bearer",
"access_token": "BZiZzJVEuP2mgNvZZBr0mgbRtKsdqgZf",
"expires_in": 7200
}
```

Paso 5:  Validar  Access Token

```jsx
curl -X GET \
--url "[http://localhost:8000/](http://localhost:8000/demo)usuarios" \
--header "Authorization: Bearer?? <ACCESS_TOKEN>"
```

resultados

```bash

```

Paso 6:  El Access Token suele tener un tiempo de expiración asociado y eso obliga a tener un flujo de renovación de Token, podemos probar este flujo de la siguiente manera 

```jsx
	curl -X POST \
	  --url "https://localhost:8443/usuarios/oauth2/token" \
	  --data "grant_type=refresh_token" \
	  --data "client_id=oauth2-demo-client-id" \
	  --data "client_secret=oauth2-demo-client-secret" \
	  --data "refresh_token=<REFRESH_TOKEN>" \
	  --insecure

```

### Protección de api gateway

rate limiting

```bash
curl -X POST http://localhost:8001/services/usuarios/plugins \
   --header "accept: application/json" \
   --header "Content-Type: application/json" \
   --data '
   {
 "name": "rate-limiting",
 "config": {
   "minute": 3,
   "hour": 10000,
   "policy": "local"
 }
}
```

Prueba 

```bash
for _ in {1..10}; do {curl -X GET --url "http://localhost:8000/usuarios/auth/login" --header "Authorization: Bearer YJ3NKhFGxbd1wbvul8oXfQO26xejffWw"; sleep 1;}  done
```

Resultado

```bash
{
  "message":"API rate limit exceeded"
}
```

Podemos crear un dashboard y visualizar alertas

![Untitled](images/Untitled.png)

request size limiting

```bash
curl -X POST http://localhost:8001/services/usuarios/plugins \
    --header "accept: application/json" \
    --header "Content-Type: application/json" \
    --data '
    {
  "name": "request-size-limiting",
  "config": {
    "allowed_payload_size": 9,
    "size_unit":"kilobytes",
    "require_content_length": false
  }
}
    '
```

prueba

```bash
curl -X POST --url "http://localhost:8000/usuarios" --data "@payload.json" --header "Authorization: Bearer YJ3NKhFGxbd1wbvul8oXfQO26xejffWw"
```

Resultado 

```bash
HTTP/1.1 413 Request Entity Too Large
Date: Mon, 29 Jul 2024 18:41:59 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 45
X-Kong-Response-Latency: 0
Server: kong/2.8.5

{
  "message":"Request size limit exceeded"
}
```

Bot detection

```bash
curl -X POST http://localhost:8001/services/oauth2-test/plugins \
   --header "accept: application/json" \
   --header "Content-Type: application/json" \
   --data '
   {
 "name": "bot-detection",
 "config": {
   "deny": [
     "postman"
   ]
 }
}
   '
```

Prueba

```bash
curl -H  "user-agent: postman" localhost:8000/usuarios
```

Resultado

```bash
{
  "message":"Forbidden"
}
```
