# ServiceMonitor

O ServiceMonitor é um recurso do Prometheus Operator que permite que você configure o Prometheus para monitorar um serviço. 

O ServiceMonitor define o conjunto de endpoints a serem monitorados pelo Prometheus e também fornece informações adicionais, como o caminho e a porta de um endpoint, além de permitir a definição de labels personalizados. Ele pode ser usado para monitorar os endpoints de um aplicativo ou de um serviço específico em um cluster Kubernetes.

É possível criar uma regra para monitorar apenas os pods que estão com o label `app=nginx` ou ainda capturar as métricas somente de endpoints que estejam com alguma label que você definiu.

ServiceMonitors configurados na instalação do Kube-Prometheus : API Server, do Node Exporter, do Blackbox Exporter, etc.

```bash
kubectl get servicemonitors -n monitoring
NAME                      AGE
alertmanager              17m
blackbox-exporter         17m
coredns                   17m
grafana                   17m
kube-apiserver            17m
kube-controller-manager   17m
kube-scheduler            17m
kube-state-metrics        17m
kubelet                   17m
node-exporter             17m
prometheus-adapter        17m
prometheus-k8s            17m
prometheus-operator       17m
```
Manifesto exemplo:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.41.0
  name: prometheus-k8s
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: web
  - interval: 30s
    port: reloader-web
  selector:
    matchLabels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
```

```yaml
apiVersion: Versão da API do Kubernetes que estamos utilizando.
kind: Tipo de objeto que estamos criando.
metadata: Informações sobre o objeto que estamos criando.
metadata.annotations: Anotações que podemos adicionar ao nosso objeto.
metadata.labels: Labels que podemos adicionar ao nosso objeto.
metadata.name: Nome do nosso objeto.
metadata.namespace: Namespace onde o nosso objeto será criado.
spec: Especificações do nosso objeto.
spec.endpoints: Endpoints que o nosso ServiceMonitor irá monitorar.
spec.endpoints.interval: Intervalo de tempo que o Prometheus irá fazer a coleta de métricas.
spec.endpoints.port: Porta que o Prometheus irá utilizar para coletar as métricas.
spec.selector: Selector que o ServiceMonitor irá utilizar para encontrar os serviços que ele irá monitorar.
```
Com isso sabemos que o ServiceMonitor do Prometheus irá monitorar os serviços que possuem os labels declarados no yaml.

Retorna todos os Custom Resources criados:
```bash
kubectl get customresourcedefinitions.apiextensions.k8s.io
NAME                                        CREATED AT
alertmanagerconfigs.monitoring.coreos.com   2024-08-20T09:53:15Z
alertmanagers.monitoring.coreos.com         2024-08-20T09:53:15Z
podmonitors.monitoring.coreos.com           2024-08-20T09:53:15Z
probes.monitoring.coreos.com                2024-08-20T09:53:15Z
prometheusagents.monitoring.coreos.com      2024-08-20T09:53:15Z
prometheuses.monitoring.coreos.com          2024-08-20T09:53:15Z
prometheusrules.monitoring.coreos.com       2024-08-20T09:53:15Z
scrapeconfigs.monitoring.coreos.com         2024-08-20T09:53:15Z
servicemonitors.monitoring.coreos.com       2024-08-20T09:53:16Z
thanosrulers.monitoring.coreos.com          2024-08-20T09:53:16Z
```
## Criando um ServiceMonitor

Para utilizar um ServiceMonitor é necessário criar uma aplicação como o Nginx e utilizar o exporter do Nginx para monitorarmos o serviço.
Depois vamos injetar carga nessa aplicação para ter dados a serem monitorados.

Será necessário criar um ConfigMap onde terá as configurações do Nginx. Nesse caso, vamo definir a rota `/nginx_status` para export as métricas do Nginx e expor a rota /metrics para export as métricas Nginx Exporter. Exemplo do `ConfigMap` em: ServiceMonitor/nginx-configmap.yaml.

Criando o `Deployment` da aplicação, exemplo em: ServiceMonitor/nginx-deployment.yaml.

Criando um `Service` para expor nosso deployment em: ServiceMonitor/nginx-service.yaml.

Depois de tudo criado, verificar se o Nginx está rodando:
```bash
curl http://<EXTERNAL-IP-DO-SERVICE>:80
```
Verificar se as métricas do Nginx estão expostas:
```bash
curl http://<EXTERNAL-IP-DO-SERVICE>:80/nginx_status
```
Verificar se as métricas do Nginx Exporter estão expostas:
```bash
curl http://<EXTERNAL-IP-DO-SERVICE>:9113/metrics
```
Finalizada a configuração e teste para criar um Service no Kubernetes e expor as `métricas do Nginx` e `Nginx Exporter`.

Criando um `ServiceMonitor` para que o Prometheus capture as métricas do Nginx Exporter em: ServiceMonitor/nginx-servicemonitor.yaml.

Para verificar o ServiceMonitor criado:
```bash
kubectl get servicemonitors.monitoring.coreos.com
```
Verificar se o Prometheus está capturando as métricas do Nginx e Nginx Exporter:
```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090
curl localhost:9090/api/v1/targets | grep nginx
```
Acessar a página do Prometheus: http://localhost:9090/

**Importante**: se o Pod cair o container do nginx-exporter também cai e com isso o serviço não é mais monitorado. Uma sugestão é criar um trigger de nodata, por exemplo, se está a x tempo sem receber dados, criar um alerta.


## Criando um PodMonitor

Temos situações que não temos um Service na frente dos Pods, quando temos CronJobs, Jobs, DaemonSets, etc. Em algumas situções o PodMonitor pode ser usado para monitorar Pods que não respondem por requisições HTTP, por exemplo, pods que expõem métricas do RabbitMQ, do Redis, Kafka, etc.

Criando um `Pod` PodMonitor/nginx-pod.yaml com os containers do Nginx e Nginx Exporter.

Criando um `PodMonitor` PodMonitor/nginx-podmonitor.yaml que irá monitorar o Pod.

Verificar o PodMonitor criado:
```bash
kubectl get podmonitors.monitoring.coreos.com
```

## Criando alertas no Prometheus

### PrometheusRule

Acesso ao AlertManager:
```bash
kubectl port-forward svc/alertmanager-main 9093:9093 -n monitoring
```
Boa parte da configurção do `Prometheus` está dentro de `configmaps`, que são recursos do Kubernetes que armazenam dados em formato de chave e valor e são muito usados para armazenar configurações de aplicações.

O ConfigMap `prometheus-k8s-rulefiles-0` é o que contém os alertas do Prometheus. Para visualizar:
```bash
kubectl get configmap prometheus-k8s-rulefiles-0 -n monitoring -o yaml
```

O `PrometheusRule` é um recurso do Kubernetes que permite a definição de alertas no Prometheus. Um exemplo de como criar está no manifesto: PrometheusRule/nginx-prometheusrule.yaml. Com o alerta configurado no Prometheus, quando for disparado, ele será enviado para o `AlertManager` e o AlertManager pode enviar uma notificação, por exemplo, via e-mail.

```bash
kubectl get prometheusrules.monitoring.coreos.com -n monitoring
```

