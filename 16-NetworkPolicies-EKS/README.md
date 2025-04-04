# Network Policies

É uma ferramenta essencial para a segurança e o gerenciamento eficaz da comunicação entre os Pods em um cluster Kubernetes.

NetPol é um conjunto de regras que definem como os Pods podem se comunicar entre si e com os outros endpoints de rede. 
Por padrão, os pods podem se comunicar livremente entre si. O que pode não ser ideal para todos os cenários.
As Network Policies permitem que você restrinja esse acesso, garantindo que apenas tráfego permitido possa fluir entre os Pods ou para endereços IP externos.

## Para que servem?

Usadas para:
 - **Isolar** Pods de tráfego não autorizado.
 - **Controlar** o acesso à serviços específicos.
 - **Implementar** padrões de segurança e conformidade.

 ## Conceitos fundamentais: Ingress e Egress

 - **Ingress**: Regras de ingresso controlam o tráfego de entrada para um Pod.
 - **Egress**: Regras de saída controlam o tráfego de saída de um Pod.

 Nas Network Policies será necessário especificar se uma regra é aplicada ao tráfego de entrada ou de saída.

 ## Como funcionam as Network Policies

 Utilizam `SELECTORS` para identificar grupos de Pods e definir regra de tráfego para eles. A política pode especificar:

 - **Ingress (entrada)**: quais Pods ou endereços IP podem se conectar a Pods selecionados.
 - **Egress (saída)**: para quais Pods ou endereços IP os Pods selecionados podem se conectar.

 ### Ainda não é padrão

 As Network Policies ainda não são um recurso padrão em todos os clusters Kubernetes.
 
 A AWS possui suporte a Network Policies no EKS, mas ainda não é padrão, é necessário instalar o `CNI da AWS` e depois habilitar o `Network Policy` nas configurações avançadas do CNI.

 Para verificar se seu cluster suporta Network Policies:

 ```bash
kubectl api-versions | grep networking
 ```
Se retornar `networking.k8s.io/v1` significa que o cluster suporta.

Se retornar `networking.k8s.io/v1beta1` significa que o cluster não suporta.

Se o cluster suporta, para implementar Network Policies você pode utilizar o `Calico`, `Weave Net` e o `Cilium` que são ferramentas mais atualizadas. O Flannel atualmente, não tem suporte a Network Policy.

Referência Calico: https://docs.tigera.io/calico/latest/about

Uma outra de aplicar políticas é com um `Service Mesh` como Istio, por exemplo, inclusive um controle maior, pois o Network Policy não faz esse controle na camada de aplicação (layer 7), porém com um Service Mesh pode deixar muito mais pesado, diferente do Network Policy que é leve, mais simples. 
A recomendação é utilizar o Network Policy para possíveis regras, havendo necessidade de um controle maior, novas políticas podem ser criadas usando o `Istio`.

## Instalando o EKS com o EKSCTL

O EKS é o Kubernetes gerenciado na AWS, nesse caso não precisaremos nos preocupar com a instalação e configuração do Kubernetes.
Os Nodes do Control Plane são gerenciados pela AWS e Workers podemos ter de 3 maneiras:

- **Managed Node Groups**: Nesse tipo de cluster, os Nodes Workers são gerenciados pela AWS, ou seja, não precisaremos nos preocupar com eles. A AWS irá criar e 
gerenciar os Workers para nós. Esse tipo de cluster é ideal para quem não quer se preocupar com a administração dos Workers.

- **Self-Managed Node Groups**: Os Workers são gerenciados por nós, precisaremos criar e gerenciá-los. É ideal para quem quer ter o controle total do Workers.

- **Fargate**: Os Workers são gerenciados pela AWS. É ideal para quem não quer se preocupar com a criação, administração e gerenciamento dos Workers.

Cada tipo tem prós e contras, e precisa analisar o seu cenário para escolher o melhor tipo que se encaixa nas suas necessidades.

Na maioria das vezes nos ambientes produtivos vamos optar pelo `Self-Managed Node Groups`, pois teremos o controle total sobre os Workers, podendo customizá-los
e gerenciá-los da forma que acharmos melhor. 

Quando optamos pelo `Fargate`, temos que levar em consideração que não teremos acesso aos Workers, isso significa menos liberdade e recursos, mas também menos 
preocupação e menos trabalho com a administração dos Workers.

Nesse exemplo, vamos utilizar o tipo `Managed Node Groups`, pois assim não precisaremos nos preocupar com a administração dos Workers.

### Instalando o EKSCTL

Para criar o cluster vamos utilizar o EKSCTL que é uma ferramenta de linha de comando que nos ajuda a criar e gerenciar o cluster EKS. 

Referência: https://eksctl.io/#

Ela se tornou uma das formas oficiais de criar e gerenciar clusters EKS. É uma das ferramentas mais usadas quando não estamos utilizando alguma ferramenta
de IaC, como Terraform, por exemplo.

Instalar no Linux:

 ```bash
 # instalando eksctl
 curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
 sudo mv /tmp/eksctl /usr/local/bin
 eksctl version
 ```

### Instalando o AWS CLI

O EKSCTL utiliza o AWS CLI para se comunicar com a AWS. 

Referência: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

É uma ferramenta que nos ajuda a interagir com os serviços da AWS. 

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```
Configurar o AWS CLI:
```bash
aws configure
```

As suas credencias da AWS, você pode encontrar em: https://console.aws.amazon.com/iam/home?#/security_credentials
As informações que irá precisar são: 

- AWS Acess Key
- AWS Secret Key
- Default region name
- Default output format

O `Access` e `Secret Key` podem ser encontrados no usuário do IAM, já a região fica a seu critério, nesse caso vamos utilizar a `região` us-east-1. 
E o formato de saída, vamos utilizar json, mas você pode utilizar tipo `text`, por exemplo.


### Criar um cluster do Modo Automático do EKS com eksctl

**Criar a política de confiança**

Criar política de confiança para permitir que o usuário IAM tenha permissão para criar o cluster. Não é a opção mais segura, mas funciona.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "eks:*",
                "iam:*",
                "cloudformation:*"
            ],
            "Resource": "*"
        }
    ]
}
```

**Comando para criar o cluster EKS**

```bash
eksctl create cluster --name=eks-pick1 --version=1.30 --region=us-east-1 --nodegroup-name=eks-pick1-nodegroup --node-type=t3.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed
```

Para deletar o cluster:
```bash
eksctl delete cluster --region=us-east-1 --name=eks-pick1
```

### Instalando o AWS VPC CNI Plugin

O AWS VPC Plugin é um plugin de rede que permite que os Pods se comuniquem com outros Pods e serviços dentro do cluster. Ele também permite que os Pods se
comuniquem com serviços fora do cluster, como Amazon S3, por exemplo.

Vamos utilizar o EKSCTL para instalar o AWS CLI Plugin, pois apesar de ser o CNI padrão do EKS, ele não vem instalado:
```bash
eksctl create addon --name vpc-cni --version v1.19.2-eksbuild.5 --cluster eks-pick1 --force
```

Link com a versão do CNI compatível com a versão do Kubernetes: https://docs.aws.amazon.com/pt_br/eks/latest/userguide/managing-vpc-cni.html

Para verificar os addons instalados no cluster:
```bash
eksctl get addon --cluster eks-pick1
```

### Habilitar o Network Policy nas configurações avançadas do CNI

Acessar o console e seguir os passos:
 - Acessar o serviço EKS.
 - Selecionar o seu cluster.
 - Selecionar a aba `Add-ons`.
 - Selecionar o edit do Addon `vpc-cni`.
 - Configuração avançada do CNI em `optional configuration settings`.
 - Habilitar o Network Policy adicionando no campo `Configuration values`: "enableNetworkPolicy": "true"

 Aguardar alguns minutos para validar se o Network Policy está atualizado e habilitado.

 ### Instalando um Nginx Ingress Controller

Se atentar a versão do Ingress Controller que será instalada, de acordo com a versão do Kubernetes:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

Guia de instalação: https://kubernetes.github.io/ingress-nginx/deploy/

Ver se o controller foi instalado:
```bash
kubectl wait --namespace=ingress-nginx --for=condition=ready pod --selector=app.kubernees.io/component=controller --timeout=90s
```

### Instalando uma aplicação e o Ingress da app

```bash
kubectl create ns giropops
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
kubectl apply -f giropops-deployment.yaml
kubectl apply -f giropops-service.yaml
kubectl port-forward -n giropops svc/giropops-senhas 5000:5000
# aplicar um ingress para aplicação
kubectl apply -f ingress-giropops-aws-dominio.yaml
kubectl get ingress -n giropops
```

Se o Dominio não estiver na AWS, criar um CNAME nas configurações do domínio apontando para o Load Balancer da AWS. 
Se o domínio estiver na AWS, basta deixar comentado a regra "host" do domínio para usar a URL da AWS que gera automático.
Aguardar a propagação do DNS realizar requisições na aplicação e acompanhar os logs:

```bash
kubectl logs -f -n giropops nomedocontainer
```
## Criando uma Network Policy

Criar uma política para que um pode não tenha permissão para se comunicar com outro, mesmo se dividir o cluster por namespace, no Kubernetes os Pods podem se comunicar mesmo rodando em namespaces diferentes.

Verificar se há uma política criada:
```bash
kubectl get netpol
kubectl get networkpolicies
```

Criar uma aplicação de teste para validar a comunicação antes de criar a regra:
```bash
kubectl run -ti alpine --image alpine -- sh
# acessando o container, verificar os processos
ps ef
#instalar o curl
apk add curl
#testar comunicao com o giropops
curl giropops-senhas.giropops.svc:5000
curl giropops-senhas.giropops.svc.cluster.local:5000
curl giropops-senhas.giropops.svc.cluster.local:5000
curl redis-service.giropops.svc:6379
#instalar o redis no container para testar comunicação com o redis do giropops
apk add redis
redis-cli -h redis-service.giropops.svc ping # deve retornar um "PONG" do redis do giropops
```
**Criando uma regra para que Pods de outra namespace não se comuniquem com redis**

```bash
kubectl apply -f netpol/primera-netpol.yaml
kubectl get netpol -n giropops
kubectl describe -n giropops primeira-netpol
#verificar se o alpine esta rodando
kubectl get pods 
kubectl exec -ti alpine -- sh
#instalar o curl
apk add curl
curl giropops-senhas.giropops.svc:5000 # aqui irá funcionar, pois a regra se aplica só ao redis
apk add redis
redis-cli -h redis-service.giropops.svc ping # nao irá retornar o PONG como antes
```

Testando a comunicação a partir do alpine na mesma namespace do redis do giropops:
```bash
kubectl run -ti alpine --image alpine -n giropops -- sh
apk add curl 
curl giropops-senhas.giropops.svc:5000 
apk add redis
redis-cli -h redis-service.giropops.svc ping # aqui irá retornar o PONG, pq a regra se aplica apenas para requisições de fora da namespace do redis.
```
Nessa regra, a comunicação com o redis, só será permitida vindo de uma aplicação que está rodando na mesma namespace que ele.

 **Aplicativo STRESS no container: usado para estressar a aplicação**
$ apt-get install stress
$ stress --cpu 1 --vm-bytes 32M --vm 1 

**recursos consumidos pelo container**
$ docker container stats --no-stream # --no-stream para só imprimir uma vez
# gerar trafego para consumir recurso dentro do container
$ dd if=/dev/zero of=tatu.img bs=8k count=2560k