	Kubectl это один из способов общения с [[Api Server]]. И имеет больше возможностей чем общение через UI или API.

Kubectl позволяет общаться именно с master processes, но именно worker processes будут исполнять команды.

Перейдем к командам:
1. kubectl get nodes - получить статус нод
2. kubectl get pods - посмотреть поды
3. kubectl get services - посмотреть все сервисы в namespace
4. kubectl get replicaset - посмотреть все replicaset
5. kubectl create deployment ИМЯ --image=ОБРАЗ - создает Deployment, внутри которого находятся поды.
6. kubectl get deployment - получить информацию о Deployment
7. kubectl edit deployment ИМЯ - открывает [[yaml]] файл конфигурации, который был создан по умолчанию.
8. kubectl logs ИМЯ_POD - получить лог пода
9. kubectl describe pod ИМЯ_POD - получить информацию о жизни пода, можно применять, если логирование выдает ошибку
10. kubectl exec -it ИМЯ_POD -- bin/bash - открываем терминал контейнера
11. kubectl delete deployment ИМЯ_DEPLOYMENT
12. kubectl apply -f ИМЯ_ФАЙЛА.yaml - исполнение конфигурационного файла в формате [[yaml]]. Если Deployment был создан таким образом, то при изменении и повторном введении команды он будет изменяться
13. kubectl describe service ИМЯ - получить информацию о Service
14. kubectl describe ingress ИМЯ - получить информацию о Ingress
15. kubectl get endpoints - посмотреть все endpoints
16. kubectl describe endpoints ИМЯ - получить информацию о endpoints сервиса с именем
17. kubectl scale deployment ИМЯ --replicas=N - изменить количество подов в deployment
18. kubectl get sa -n NS - посмотреть список Service Accounts, которые существуют в конкретном NS
19. kubectl rollout restart ИМЯ - безопасный перезапуск, можно делать для Deployment, StatefullSet и Daemons