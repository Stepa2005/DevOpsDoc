Тут я распишу как устанавливать [[Gateway API]] от HAProxy. Информация взята из статьи https://www.haproxy.com/documentation/kubernetes-ingress/gateway-api/enable-gateway-api/

```bash
kubectl scale deployment haproxy-ingress-kubernetes-ingress --replicas=1 -n haproxy-controller 

kubectl scale deployment k8s-test-deployment --replicas=1 -n test-migrate

kubectl edit svc haproxy-ingress-kubernetes-ingress -n haproxy-controller
```

Сделать сразу добавление gateway api, без промежуточного ingress!!!

При экспериментальной версии crd, helm зависал, поэтому рекомендуется установить без helm

***Установка с помощью kubectl***

```bash
helm template haproxy-ingress haproxytech/kubernetes-ingress \
>   --namespace haproxy-controller \
>   --set controller.gatewayAPI.enabled=true \
>   --set controller.service.type=NodePort > haproxy-ingress.yaml

kubectl create ns haproxy-controller

kubectl apply -f haproxy-ingress.yaml

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.5.1/experimental-install.yaml

kubectl apply -f https://raw.githubusercontent.com/haproxytech/kubernetes-ingress/master/deploy/tests/config/experimental/gwapi-rbac.yaml

kubectl delete validatingwebhookconfiguration gateway-api-admission

vim deployment-enable-gateway-api-patch.yaml
```

```deployment-enable-gateway-api-patch.yaml
spec:
  template:
    spec:
      containers:
      - name: kubernetes-ingress-controller
        args:
          - --configmap=haproxy-controller/haproxy-ingress-kubernetes-ingress
          - --gateway-controller-name=haproxy.org/gateway-controller
```

Убрать ошибку rate limit:
```bash
kubectl patch configmap haproxy-ingress-kubernetes-ingress -n haproxy-controller --type merge -p '{"data": {"rate-limit-requests": "1000"}}'

```


***Установка с Помощью Helm***

Для начала нужно создать HAProxy ingress controller, сделаем это через [[Helm Charts]]:
``` bash
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
helm install haproxy-ingress haproxytech/kubernetes-ingress   --create-namespace   -n haproxy-controller   --set controller.gatewayAPI.enabled=true --debug
```

```
helm install haproxy-ingress haproxytech/kubernetes-ingress \
      --create-namespace \
      --namespace haproxy-controller
```

Перед переходом к gateway api сделаем обычный ingress controller типа haproxy и переведем на него трафик. Создадим экземпляр контроллера:
```haproxy-ingress-test.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-migrate-ingress
  namespace: test-migrate 
  annotations:
    haproxy.org/rate-limit-requests: "100"
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

Проверить его работу можно командой 
```bash
curl -v -H "Host: test-migrate.ru" http://IP
```
Логи контроллера можно смотреть вот так:
```bash
kubectl logs -n haproxy-controller -l app.kubernetes.io/name=kubernetes-ingress
```

Далее надо выдать контроллеру доступ к ресурсам Gateway API, для этого создадим файл haproxy-values.yaml и обновим контроллер через helm:
```haproxy-values.yaml
controller:
  kubernetesGateway:
    enabled: true
    gatewayControllerName: haproxy.org/gateway-controller
```

``` bash
helm upgrade haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-controller \
  -f haproxy-values.yaml \
  --debug
```



**Теперь создадим GatewayClass:**
``` haproxy-gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: GatewayClass
metadata:
  name: haproxy-gatewayclass
spec:
  controllerName: haproxy.org/gateway-controller
```

 ```bash
 kubectl apply -f haproxy-gatewayclass.yaml
 gatewayclass.gateway.networking.k8s.io/haproxy-ingress-gatewayclass created
 ```

**Перейдем к созданию Gateway**. Я указываю hostname, чтобы он слушал трафик только для этого конкретного домена. Можно его не укаingress-зывать, тогда gateway будет слушать весь трафик и фильтрация будет происхожить в HTTPRoute
```haproxy-gateway-http.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: test-gateway
  namespace: test-migrate
spec:
  gatewayClassName: haproxy-gatewayclass
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "test-migrate.com"
    allowedRoutes:
      namespaces:
        from: Same
```

Если же мы хотим сделать  tcproute вместо httproute, то gateway должен выглядеть так:
```haproxy-gateway-tcp.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: test-gateway
  namespace: test-migrate
spec:
  gatewayClassName: haproxy-gatewayclass
  listeners:
    - name: test-tcp-listener
      protocol: TCP
      port: 80
      allowedRoutes:
        kinds:
        - kind: TCPRoute
        namespaces:
          from: All

```

```bash
kubectl apply -f haproxy-gateway.yaml
gateway.gateway.networking.k8s.io/test-gateway created
```
Можно проверить ее создание
```bash
kubectl get gateway test-gateway -n test-migrate
NAME           CLASS                          ADDRESS   PROGRAMMED   AGE
test-gateway   haproxy-ingress-gatewayclass             Unknown      26s
```

**Теперь надо установить HTTPRoute:**
``` haproxy-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: test-httproute
  namespace: test-migrate
spec:
  parentRefs:
  - name: test-gateway
    namespace: test-migrate
  hostnames:
  - "test-migrate.ru"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: k8s-test-service
      port: 8000
```

Или **TCPRoute**:
```haproxy-tcproute.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: test-tcproute
  namespace: test-migrate
spec:
  parentRefs:
    - name: test-gateway
      namespace: test-migrate
      sectionName: test-tcp-listener
  rules:
    - backendRefs:
        - name: k8s-test-service
          port: 8000
```

```bash
kubectl apply -f haproxy-httproute.yaml
httproute.gateway.networking.k8s.io/test-httproute created
```
