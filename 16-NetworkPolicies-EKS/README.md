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

Se o cluster suporta, para implementar Network Policies você pode utilizar o `Calico`, `Weave Net` e o `Cilium` que são ferramentas mais atualizadas.

Referência Calico: https://docs.tigera.io/calico/latest/about

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
# para ARM system
ARCH=adm64 # verificar a arquitetura antes com 'uname -m'.
PLATAFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATAFORM.tar.gz"

#Verificação checksum - opcional
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATAFORM | sha256sum --check

tar -xzf eksctl_$PLATAFORM.tar.gz -C /tmp && rm eksctl_$PLATAFORM.tar.gz

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