Это API object, который управляет внешним подключением к сервисам в кластере (обычно HTTP). 

Ingress предосталяет доступ к сервисам внутри кластера, например он может пересылать весь трафик на один сервис.
![ingress-diagram](https://kubernetes.io/docs/images/ingress.svg)
Просто при создании Ingress ничего не произойдет, надо иметь [[Ingress Controller]].

Вот минимальный пример Ingress
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80

```
Внутри spec мы описываем все что нужно для конфигурирования распределителя нагрузки (load balancer) или прокси сервер. **Внутри spec находится набор правил для всех HTTP(S) запросов и только их.**

Изначальный **ingressClassName** должен быть определен.

Для обработки запросов, которые не соответствуют правилам используют **defaultBackend**. Это сервисный под (сервис), который отвечает за возврат ошибки (например 404).

