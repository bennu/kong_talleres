# Taller 1 - Instalación

**Objetivo:** Disponibilizar a nivel local Kong API Gateway con su respectiva interfaz de administración Konga

**Nota:** Independiente del sistema operativo recomendamos usar Rancher Desktop por sobre Docker Desktop. Los pasos del taller 1 están pensados para Windows. 

Clonar repo

```powershell
git clone https://github.com/bennu/kong_talleres.git
```

## **Levantamiento de clúster  y herramientas necesarias para el taller**

Para este taller utilizaremos Rancher Desktop. Para su instalación lo podemos hacer por medio de gestores de paquetes como [chocolatey](https://chocolatey.org/install#individual)

1. Descargar e instalar gestor de paquetes chocolatey para Windows. Para más información ver documentación [chocolatey](https://chocolatey.org/install#individual) 
    1. Abrir PowerShell en modo administrador
    2. Ejecutar el siguiente comando:
        
        ```powershell
        Set-ExecutionPolicy Bypass -Scope Process
        ```
        
    3.  Ejecutar el siguiente comando para instalar Chocolatey
        
        ```powershell
        Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
        ```
        
    4. Una vez completada la instalación,  podemos verificar ejecutado
        
        ```powershell
        choco -?
        ```
        
        Resultado
        
        ```powershell
           .....
             --skipcompatibilitychecks, --skip-compatibility-checks
             SkipCompatibilityChecks - Prevent warnings being shown before and after
               command execution when a runtime compatibility problem is found between
               the version of Chocolatey and the Chocolatey Licensed Extension.
        
             --ignore-http-cache
             IgnoreHttpCache - Ignore any HTTP caches that have previously been
               created when querying sources, and create new caches. Available in 2.1.0+
        Chocolatey v2.3.0
        ```
        

1. Instalar Rancher Desktop usando gesto de paquete chocolatey en Windows: 

```powershell
choco install rancher-desktop -y
```

Nota: Rancher Desktop viene con las siguientes herramientas encapsula dentro de la misma solución:

- nerdctl
- kubectl
- Helm
- Docker cli

Esto podría afectar si tiene ya instalada las herramientas mencionadas en la lista anterior

1. Instalación de deck

```powershell
curl.exe -sL https://github.com/kong/deck/releases/download/v1.38.1/deck_1.38.1_windows_amd64.tar.gz -o deck.tar.gz
mkdir deck
tar -xf deck.tar.gz -C deck
powershell -command "[Environment]::SetEnvironmentVariable('Path', [Environment]::GetEnvironmentVariable('Path', 'User') + [IO.Path]::PathSeparator + [System.IO.Directory]::GetCurrentDirectory() + '\deck', 'User')"
```

## **Instalación de Kong 2.8  y Konga sobre Kubernetes**

Agregar repo Helm chart de kong

```powershell
helm repo update
helm repo add kong https://charts.konghq.com
```

Instalar Kong versión 2.8 a través de helm chart

```powershell

helm install kong kong/kong -f taller_1a/kong_2.8/values.yaml
```

Verificar que se haya desplegado correctamente Kong  

```powershell
kubectl get pod --selector=app.kubernetes.io/instance=kong -w
```

```powershell

NAME                                  READY   STATUS      RESTARTS   AGE
pod/kong-postgresql-0                 1/1     Running     0          2m11s
pod/kong-kong-init-migrations-hsjhd   0/1     Completed   0          2m11s
pod/kong-kong-6547687c46-8cj4r        1/1     Running     0          2m11s
```

Konga

Instalación Konga

```powershell
kubectl apply -f taller_1a/konga/deployment.yaml
```

Verificar que se esté ejecutando el pod de konga

```powershell
kubectl get pods --selector=app=konga -w
```

Se debe acceder al interfaz generando un port-forward del servicio llamado Konga

exponer interfaz de administración - Konga

```powershell
kubectl port-forward service/konga 8080:80
```

Acceder a la siguiente URL  desde el navegador

```powershell
 http://localhost:8080
```

Se debe crear una cuenta de administración para utilizar Konga

![Untitled](./images/0.png)

Una vez creada la cuenta se debe iniciar sesión 

![Untitled](./images/1.png)

Conectar Kong Admin API con interfaz Konga

Se puede utilizar el DNS de Kubernetes si está en el mismo clúster más el puerto que se expone el servicio, en este caso:

```powershell
http://kong-kong-admin.default:8001
```

![Untitled](./images/2.png)

Podemos validar si se conectó correctamente a la Admin API de Kong viendo el número de conexiones activa y la versión de Kong 

![Untitled](./images/3.png)
