# Cluster Kubernetes

Conjunto de nodes que trabalham juntos para executar todos os pods, sendo composto por nodes control plane quando workers.

## Instalação do cluster Kubernetes

Ferramentas para provisionar um cluster Kubernetes:
 - *kubeadm*: ferramenta para criar e gerenciar um cluster k8s em ambientes Bare metal, máquinas físicas, VMs.
 - *kubespray*: ferramenta que usa Ansible para implantar um cluster k8s.
 - *Cloud Providers*: provedores de nuvem: AWS, Google Cloud Platform e Microsoft Azure oferecem modelos predefinidos para implantar um cluster k8s. Modelos gerenciáveis e não gerenciáveis.
 - *Kubernetes Gerenciados*: serviços oferecidos por Cloud Providers. AWS - EKS, Google Cloud - GKE, Azure - AKS. São serviços gerenciados do cluster que configuram, atualizam e dão manutenção no control plane do cluster. Apenas nos preocupamos com as aplicações.
 - *Kops*: Cria um cluster k8s personalizado na nuvem. É uma opção manual e não autogerenciável, mas talvez de menor custo para implantação na nuvem. Vantagens são as personalização, escalabilidade e segurança, porém pode ser mais complexo se não estiver familiarizado com a nuvem.
 - *Minikube* e *kind*: Ferramentas para fins de teste e aprendizado, criando um cluster local em um único nó.

 Outras formas de implantação de um cluster Kubernetes consultar a doc oficial.

 ### Recomendações para criar um cluster com kubeadm
 
 Principais pré-requisitos mínimos de hardware:
  - Sistema Operacional Linux
  - 2 GB de memória RAM (1 control plane)
  - 2 core de CPU
  - Conexão de rede entre todos os nodes
  - Liberação de portas:
    - 6443 - API Server (Servidor da API do Kubernetes), utilizado por todos.
    - 2379 a 2380 - API servidor-cliente do etcd, utilizado por kube-apiserver, etcd
    - 10250 - API do kubelet, utilizado por 	kubeadm, Camada de gerenciamento
    - 10259 - kube-scheduler - utilizado por kubeadm
    - 10257 - kube-controller-manager - utilizado por kubeadm
    - 30000 a 32767 - Serviços NodePort, utilizado por todos.


 ### Criando instâncias na AWS para o cluster com kubeadm

 Criei uma conta na AWS para subir 3 instâncias EC2 para testes no k8s:
  - Ao acessar a conta, selecionei a região "N. Virginia", pois como é uma das mais baratas para subir instâncias EC2 Linux. Na opção gratuita é possível subir instânicias free, porém não suporta com os pré-requisitos mínimos para o funcionamento do cluster com kubeadm.
  - Criei um nome para o cluster e adicionei a quantidade de instâncias, nesse caso foram 3, sendo 1 control plane e 2 workers.
  - Selecionei a opção Ubuntu 24.04 TLS 64 bits
  - Tipo da instâncias: t2.medium 2vCPU / 4GiB RAM (custo 0.0464 USD por hora)
  - Criei um *key pair* para conectar nas instâncias e Criei as instâncias.
  - Como foi a primeiro vez que criei instâncias nessa região, tive que aguardar uns minutos (pode demorar até 4 horas) para receber um e-mail de uma validação adicional que a AWS exige.
  - Após validado, pude confirmar que as instâncias foram criadas em "Instances" e editar o nome de cada uma (k8s-controlPlane, k8s-worker1 e k8s-worker2).
  - Como as 3 foram criadas simultaneamente, fazem parte do mesmo *Security group* da AWS, que é basicamente o *firewall* onde é possível adicionar novas regras de entrada e saída. Adicionando uma regra em uma das instâncias será liberada para as demais.
  - Para o funcionamento do cluster k8s é necessário liberação de portas das APIs do k8s, com isso, em em, *Security* e clicando no *security group*, no meu caso "sg-09ef8ae06f73e860a - launch-wizard-4" adicionei mais uma regra (*Add rule*) e liberei a porta 6443 TCP. E novamente em *Add rule* criei outra regra liberando o range 10250 - 10259 TCP.

Após a inicialização das instâncias é necessário realizar a conexão em cada uma delas para fazer a instalação das ferramentas do k8s.
Para conectar e executar comandos simultaneamente, sugiro a instalação do terminal *Terminator*, ativando o modo *broadcast all*.

Instalando o *Terminator*:

```
sudo add-apt-repository ppa:gnome-terminator
sudo apt-get update
sudo apt-get install terminator
```
No meu caso, quando ativei o *broadcast all* caracteres eram inseridos duplicados quando digitava. Para solucionar tive que matar o processo *ibus* e desinstalar o pacote:

```
pskill -9 ibus
sudo apt remove ibus
```
Para conectar nas instâncias, selecione uma por vez e clique em "Conectar", na aba "Cliente SSH", copie o comando *ssh -i*... e cole em cada um dos terminais.
Uma vez conectado mude o hostname de cada máquina para facilitar a identificação.

```
sudo su
hostnamectl hostname k8s-controlplane #maquina1
hostnamectl hostname k8s-worker1 #maquina2
hostnamectl hostname k8s-worker2 #maquina3
exit
bash
#verá que o hostname foi alterado
```
### Preparando o sistema e instalando do kubeadm

Primeira coisa a se fazer em uma instalação do cluster k8s é desativar a *swap* das máquinas:

```
sudo swap off -a
```
Remova sua entrada do arquivo /etc/fstab.
Regenere unidades de montagem para que o sistema registre a nova configuração:

```
systemctl daemon-reload
```

Criar arquivo para definições para habilitar os módulos do Kernel que serão inicializados com a máquina (boot):

```
sudo nano /etc/modules-load.d/k8s.conf
```
Adicionar os 2 módulos no arquivo:
- overlay
- br_netfilter 

Recarregar os módulos:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
Parametrização do Kernel:

```
sudo nano /etc/sysctl.d/k8s.conf
```
Adicionar os parâmetros:
- net.ipv4.ip_forward = 1# habilitar encaminhamento de pacotes
- net.bridge.bridge-nf-call-iptables = 1 # habilitar modo brigde ipv4
- net.bridge.bridge-nf-call-ip6tables = 1

Aplicar as mudanças:
```
sudo sysctl --system
```

Instalando pacotes adicionais e o Kubernetes:
```
sudo apt-get update
sudo apt-get install apt-transport-https curl -y

# Carregar a chave para instalação dos pacotes do k8s
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Adicionar o pacote do Kubernetes no arquivo "kubernetes.list"
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Atualizar
sudo apt-get update

# Instalar os pacotes
sudo apt-get install -y kubelet kubeadm kubectl

# Adicionar os pacotes para não atualizar automaticamente, evitar quebrar o cluster
sudo apt-mark hold kubelet kubeadm kubectl

```
Instalando *container runtime* - *containerd*

```
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Copiar a chave do repositório do docker para acessar o repositório assinado
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Baixar o conteúdo
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#Instalar containerd
sudo apt-get update && sudo apt-get install -y containerd.io
```

Configurar o *containerd*

```
sudo containerd config default | sudo tee /etc/containerd/config.toml

#Alterando a configuração do SystemdCgroup para 'true'
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl status containerd
sudo systemctl enable containerd
```

Habilitar o kubelet

```
# Início automático com o sistema
sudo systemctl enable --now kubelet
```

