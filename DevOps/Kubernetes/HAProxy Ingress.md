**Установка**
```bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts

helm repo update

helm install haproxy-ingress haproxytech/kubernetes-ingress \
      --create-namespace \
      --namespace haproxy-controller
```

Запуск ингресс:
``` haproxy-ingress-test.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-migrate-ingress
  namespace: test-migrate 
  annotations:
    haproxy.org/rate-limit-requests: "1000"
spec:
  ingressClassName: haproxy 
  rules:
  - host: test-migrate.ru
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: k8s-test-service
            port:
              number: 8000
```

```bash
kubectl apply -f haproxy-ingress-test.yaml
```

Если тип сервиса у контроллера - Loadbalancer, то теперь можно сделать запрос:
```bash
curl -v -H "Host: test-migrate.ru" http://IP
```
Или же, если он ClusterIP, сделать запрос по адресу ноды и указать правильный порт.

Теперь нужно добавить tls сертификаты, чтобы можно было делать https запрос.


Для того, чтобы провести миграцию с контроллера NGINX на HAPRoxy нужно перенести аннотации. Для этого есть сервис https://www.haproxy.com/ingress-nginx-migration-assistant. 

