
Пробуем задеплоить большой проект с микросервисами
https://github.com/GoogleCloudPlatform/microservices-demo

Перед деплоем надо определить:

1) Какие микросервисы мы деплоим
2) Как они связаны друг с другом
3) Есть ли базы данных или 3rd party services
4) Какой сервис будет доступен снаружи

Деплой в кластер (Linode) -> получаем kubeconfig.yaml

`chmod 400 kubeconfig.yaml`
`export KUBECONFIG=kubeconfig.yaml`
`kubectl get node`

и видим наш кластер в облаке

`kubectl create ns microservices`
`kubectl apply -f config.yaml -n microservices`

