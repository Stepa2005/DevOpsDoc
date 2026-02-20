Поскольку для разворачивания настоящего кластера мне не хватит ресурсов, я буду использовать [[minikube]]. Это позволит мне на одном кластере (моем компьютере) запускать сразу и master processes и worker processes. Запускаться это все будет через virtualbox.

Теперь, когда у меня есть виртуальная машина мне надо как то с ней общаться и для этого я буду использовать [[kubectl]].

При запущеном gitlab CI пайплайне и удалении pod, в котором он выполняется автоматически создастся новый pod, который будет исполнять команды.

Для копирования в другой namespace: kubectl get deployment k8s-test-deployment -n default -o yaml | \
  sed 's/namespace: default/namespace: test-migrate/' | \
  kubectl apply -n test-migrate -f -

Для миграции сервиса в другой ns:
```
apiVersion: v1
kind: Service
metadata:
  name: k8s-test-service
  namespace: test-migrate
  labels:
    app: k8s-test
    version: v1
spec:
  selector:
    app: k8s-test  
  ports:
  - name: http
    port: 8000      
    targetPort: your-port  
    protocol: TCP
  type: LoadBalancer  
  externalTrafficPolicy: Cluster
  loadBalancerIP: "" 
```

