
#### kong

helm repo update kong

helm repo add kong https://charts.konghq.com

helm install kong kong/kong -f values.yaml


#### instalacion  konga
kubectl apply -f deployment.yaml



#### Uso
kubectl port-forward service/konga  8080:80 &

kubectl port-forward service/kong-kong-proxy  8000:80 &

nota: presionar enter en el campo para guardar los cambios