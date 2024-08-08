# Prometheus

## Prometheus Operator

É um recurso personalizado do Kubernetes para simplificar o deploy e configuração do Prometheus e componentes relacionados ao monitoramento do cluster. [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)

## kube-prometheus

Fornece uma stack de monitoramento completa do cluster baseado no Prometheus e Prometheus Operator. Inclui exportador de métricas como node_exporter para coletar métricas dos nós, e outros endpoints de métricas. É recomendado usá-lo ao invés do Prometheus Operator. [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus).

# helm chart

O [prometheus-community/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart 
fornece feature similares ao kube-prometheus. É recomendado usá-lo ao invés do Prometheus Operator.
