# Prometheus

## Prometheus Operator

É um recurso personalizado do Kubernetes para simplificar o deploy e configuração do Prometheus e componentes relacionados ao monitoramento do cluster. [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator)

## kube-prometheus

Fornece uma stack de monitoramento completa do cluster baseado no Prometheus e Prometheus Operator. Inclui exportador de métricas como node_exporter para coletar métricas dos nós, e outros endpoints de métricas. É recomendado usá-lo ao invés de somente Prometheus Operator para monitoramento do cluster Kubernetes. [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus).

# helm chart

O [prometheus-community/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart 
fornece feature similares ao kube-prometheus. É recomendado usá-lo ao invés do Prometheus Operator.


# Instalando cluster Kubernetes da AWS - EKS

## Instalando a ferramenta eksctl para instalar o EKS

 ```bash
 # instalando eksctl
 curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
 sudo mv /tmp/eksctl /usr/local/bin
 ```

 Instalar o CLI da AWS
 ```bash
 curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
 unzip awscliv2.zip
 sudo ./aws/install
 ```

Criar usuário Identity and Access Management (IAM) que é um serviço web que ajuda você a controlar com segurança o acesso aos recursos da AWS. Para isso:

- Acesso o console da AWS
- Acesse Identity and Access Management (IAM)
- Usuário e criar usuário
- Depois de criado, clique no usuário e criar chave de acesso.
- É necessário também atrelar as políticas para o usuário ter as permissões para criar o cluster.

 Configurar sua conta na AWS na máquina local
 ```bash
 aws configure
 ```

 Informar no comando acima a `AWS Access key ID`, sua `AWS Secret Access Key`, sua `Default region name` e seu `Default output format`.
 Para saber mais acessar a [Documentação oficila da AWS](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-chap-configure.html).

 ### Criar o cluster EKS com eksctl

```bash
 eksctl create cluster --name=eks-cluster --version=1.30 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t2.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed
 ```
Além do nome, versão, região... o comando cria um nodegroup chamado `eks-cluster-nodegroup`. O `eksctl` irá cuidar de toda a infraestrutura necessária para o funcionamento do nosso cluster EKS.

Instalar o kubectl:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

O comando abaixo é necessário para que o kubectl saiba qual cluster ele deve utilizar, ele irá pegar as credenciais do nosso cluster EKS e armazenar no arquivo ~/.kube/config. Aonde us-east-1 é a região do nosso cluster EKS, e eks-cluster é o nome do nosso cluster EKS. 
```bash
aws eks --region us-east-1 update-kubeconfig --name eks-cluster
```

Para listar os clusters EKS que temos em nossa conta, basta executar o seguinte comando:
```bash
eksctl get cluster -r us-east-1
```

Para aumentar o número de nós do nosso cluster EKS, basta executar o seguinte comando:
```bash
eksctl scale nodegroup --cluster=eks-cluster --nodes=3 --nodes-min=1 --nodes-max=3 --name=eks-cluster-nodegroup -r us-east-1
```

Para diminuir o número de nós do nosso cluster EKS, basta executar o seguinte comando:
```bash
eksctl scale nodegroup --cluster=eks-cluster --nodes=1 --nodes-min=1 --nodes-max=3 --name=eks-cluster-nodegroup -r us-east-1
```

Para deletar o nosso cluster EKS, basta executar o seguinte comando:
```bash
eksctl delete cluster --name=eks-cluster -r us-east-1
```

# Instalando o Kube-Prometheus

```bash
git clone https://github.com/prometheus-operator/kube-prometheus
cd kube-prometheus
kubectl create -f manifests/setup
```

No comando `create` instalamos alguns CRDs (Custom Resource Definitions) que são como extensões do Kubernetes, e que são utilizados pelo Kube-Prometheus e com isso o Kubernetes irá reconhecer esses novos recursos, como por exemplo o `PrometheusRule` e o `ServiceMonitor`

Instalar o Prometheus e o Alertmanager:

```bash
kubectl apply -f manifests/
kubectl get pods -n monitoring
```
Com isso fizemos a instalação da Stack do nosso Kube-Prometheus, que é composta pelo Prometheus, pelo Alertmanager, Blackbox Exporter e Grafana.

Acessar o Grafana após a instalação usando recurso `port-forward` do Kubernetes:
```bash
kubectl port-forward -n monitoring svc/grafana 33000:3000
```
Abrir no navegador:
```bash
http://localhost:33000
#login: admin / admin
```

Acessar o Prometheus:
```bash
kubectl port-forward -n monitoring svc/prometheus-k8s 8181:9090
http://localhost:8181
```
