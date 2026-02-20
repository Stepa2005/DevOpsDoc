**Gateway API** состоит из 4 типов:
- **GatewayClass:** описывает сет gateway с похожими конфигурациями и управляемыми одном контроллером
- **Gateway:** правила приема входящего трафика и выбора маршрутов (`HTTPRoute`) для этого трафика.
- **HTTPRoute:** правила маршрутизации HTTP трафика по бэкендам или его перенаправления
- **GRPCRoute:** правила маршрутизации gRPC трафика по бэкендам или его перенаправления
![A figure illustrating the relationships of the three stable Gateway API kinds](https://kubernetes.io/docs/images/gateway-kind-relationships.svg)

**GatewayClass**
Gateway могут быть имплемментированны разными контроллерами. Сам gateway должен ссылаться на GatewayClass, в котором содержится имя контроллера, который отвечает за этот класс.

Минимальный пример:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```
В этом примере контроллер управляет GatewayClass с именем контроллера example.com/gateway-controller

**Gateway**
Gateway описывает единицу инфраструктуры, которая отвечает за трафик. Gateway определяет endpoint, который используется при обработке трафика, например фильтрация, балансировка для таких backend как Service. К примеру, Gateway может представлять облачный балансировщик или внутрикласстерный прокси сервер, который сконфигурирован так, чтобы принимать HTTP трафик.

Пример Gateway:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: example-namespace
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "www.example.com"
    allowedRoutes:
      namespaces:
        from: Same
```
В этом примере Gateway сделан так, чтобы слушать HTTP трафик на 80 порту. Поскольку поле addresses не указано, контроллер скажет этому Gateway с какого адреса обрабатывать трафик.

**HTTPRoute**
HTTPRoute описывает поведение HTTP реквестов от Gateway к backend network endpoints. 

Пример HTTPRoute:
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```
В этом примере HTTP трафик из Gateway example-gateway с Host. Запрос на www.example.com  путем /login будет обрабатываться Сервисом example-svc на порту 8080.


Вот пример обработки трафика с помощью K8s Gateway API:
![A diagram that provides an example of HTTP traffic being routed to a Service by using a Gateway and an HTTPRoute](https://kubernetes.io/docs/images/gateway-request-flow.svg)
В этом примере запрос к Gateway сделан как реверсивное прокси:
1. Клиент делает HTTP запрос к http://www.example.com
2. Его DNS Resolver подставляет IP адресс по этому доменному имени
3. Запрос отправляется на адрес Gateway API
4. Реверсивный прокси (например типа nginx) смотрит какой домен указан в заголовке и ищет правила маршрутизации, которые настроены через Gateway + HTTPRoute и перенаправляет трафик в нужное место
5. В конце, реверсивное прокси оправляет запрос на один или несколько backends.
