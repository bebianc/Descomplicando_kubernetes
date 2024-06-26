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

```bash
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

Instalando pacotes adicionais e o Kubernetes versão 1.28.1:
```bash
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl -y

# Carregar a chave para instalação dos pacotes do k8s
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Adicionar o pacote do Kubernetes no arquivo "kubernetes.list"
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Atualizar
sudo apt-get update

# Instalar os pacotes
sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1

# Adicionar os pacotes para não atualizar automaticamente, evitar quebrar o cluster
sudo apt-mark hold kubelet kubeadm kubectl

```
**Nota**: Para instalar versão 1.29.1 executar os mesmos comandos mudando o versionamento.

**Nota**: Em versões anteriores ao Debian 12 e Ubuntu 22.04, o /etc/apt/keyrings não existe por padrão. Você pode criar este diretório se precisar, tornando-o visível para todos, mas com permissão de escrita apenas aos administradores.


### Instalando *container runtime* - *containerd*

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Copiar a chave do repositório do docker para acessar o repositório assinado
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Baixar o conteúdo
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#Instalar containerd
sudo apt-get update && sudo apt-get install -y containerd.io
```

#### Configurar o *containerd*

```bash
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

### Inicializando o cluster

Inicializar a partir do nodo 1 do controlplane:

```
# Init para iniciar, passar rede que vai ser usada e a interface de rede que o kubeadm vai usar para comunicar com demais nós
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=<O IP QUE VAI FALAR COM OS NODES>
```
É possível passar outros parâmetros no *kubeadm init* de acordo de como será a configuração do seu cluster, setando

&nbsp;

Após execução o próprio retorno do comando irá mostrar novos comando a serem executados, que são basicamente:
Como acessar o cluster com usuário regular ou root, como fazer um deploy de um POD e como fazer o deploy de um nodo worker, usando um usuário root:

```bash

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.57.89:6443 --token if9hn9.xhxo6s89byj9rsmd \
	--discovery-token-ca-cert-hash sha256:ad583497a4171d1fc7d21e2ca2ea7b32bdc8450a1a4ca4cfa2022748a99fa477 

```

Essa configuração é necessária para que o kubectl possa se comunicar com o cluster, pois quando estamos copiando o arquivo `admin.conf` para o diretório `.kube` do usuário, estamos copiando o arquivo com as permissões de root, esse é o motivo de executarmos o comando `sudo chown $(id -u):$(id -g) $HOME/.kube/config` para alterar as permissões do arquivo para o usuário que está executando o comando.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### Entendendo o arquivo admin.conf

- É um arquivo de configuração do kubectl, que é o cliente de linha de comando do Kubernetes. Ele é usado para se comunicar com o cluster Kubernetes.

```yaml
apiVersion: v1

#A seção clusters contém informações sobre os clusters Kubernetes que você deseja acessar, como o endereço do servidor API e o certificado de autoridade. Neste arquivo, há somente um cluster chamado kubernetes, que é o cluster que acabamos de criar.
clusters:
- cluster:
    certificate-authority-data: SEU_CERTIFICADO_AQUI
    server: https://172.31.57.89:6443
  name: kubernetes

#seção contexts define configurações específicas para cada combinação de cluster, usuário e namespace. Pode ter mais de um contexto dentro do `admin.conf`, por exemplo, um para o cluster de produção outro para homologação. Nesse caso, ele é chamado kubernetes-admin@kubernetes e combina o cluster kubernetes com o usuário kubernetes-admin.
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes

#A propriedade current-context indica o contexto atualmente ativo, ou seja, qual combinação de cluster, usuário e namespace será usada ao executar comandos kubectl. Neste arquivo, o contexto atual é o kubernetes-admin@kubernetes.
current-context: kubernetes-admin@kubernetes

kind: Config

#A seção preferences contém configurações globais que afetam o comportamento do kubectl. Aqui podemos definir o editor de texto padrão, por exemplo.
preferences: {}

#A seção users contém informações sobre os usuários e suas credenciais para acessar os clusters. Neste arquivo, há somente um usuário chamado kubernetes-admin. Ele contém os dados do certificado de cliente e da chave do cliente.
users:
- name: kubernetes-admin
  user:
    client-certificate-data: SUA_CHAVE_PUBLICA_AQUI
    client-key-data: SUA_CHAVE_PRIVADA_AQUI
```

Credenciais usadas para autenticar o usuário:

- **Token de autenticação**:Token de acesso que é usado para autenticar o usuário que está executando o comando kubectl. Gerado automaticamente quando o cluster é inicializado.

- **certificate-authority-data**: Certificado em base64 da autoridade de certificação (CA) do cluster. Responsável por assinar e emitir certificados para o cluster. Usado para verificar a autenticidade dos certificados apresentados pelo servidor de API e pelos clientes, garantindo que a comunicação entre eles seja segura e confiável. Pode ser encontrado em: /etc/kubernetes/pki/ca.crt.

- **client-certificate-data**: Certificado do cliente. Usado para autenticar o usuário ao se comunicar com o servidor de API do Kubernetes. Contém informações do usuário e a chave pública. Pode ser encontrado em: /etc/kubernetes/pki/apiserver-kubelet-client.crt.

- **client-key-data**: Chave privada do cliente. A chave privada é usada para assinar as solicitações enviadas ao servidor de API do Kubernetes, permitindo que o servidor verifique a autenticidade da solicitação. Pode ser encontrado em: /etc/kubernetes/pki/apiserver-kubelet-client.key.

Outra forma de acessar o `admin.conf`:
```bash
kubectl config view
```
#### Adicionar 2 workers no cluster

Executar o comando que retornou na inicialização do cluster, com o kubeadm.

Executar em cada nodo:
```bash
sudo kubeadm join 172.31.57.89:6443 --token if9hn9.xhxo6s89byj9rsmd \
	--discovery-token-ca-cert-hash sha256:ad583497a4171d1fc7d21e2ca2ea7b32bdc8450a1a4ca4cfa2022748a99fa477 
```

- **kubeadm join**: Adicionar um novo nó ao cluster.
 **172.31.57.89:6443**: Endereço IP e porta do servidor de API do control plane
 - **--token if9hn9.xhxo6s89byj9rsmd**: O token para o worker no CP durante o processo de adesão. Dão gerados pelo CP e têm uma validade limitada (por padrão, 24 horas).
 - **--discovery-token-ca-cert-hash sha256:ad583497a4171d1fc7d21e2ca2ea7b32bdc8450a1a4ca4cfa2022748a99fa477**: Hash criptografado do CA do control plane. Ele é usado para garantir que o nó worker esteja se comunicando com o control plane correto e autêntico.

Ver novos nodos. Perceber se estão com status `Ready`:
 ```bash
kubectl get nodes
```
##### Instalando o Weave Net

Instalar o plugin de rede que vai criar a rede de comunicação entre os pods.
Por padrão, o k8s não resolve a rede automaticamente, por isso precisa ser instalado um plugin adicional.

**CNI** - conjunto de bibliotecas e especificações para interface de rede em containers, para criar plugins para soluções de rede integradas com k8s.

Exemplos de plugins de mercado:

- **Calico** um dos mais populares e utilizado no k8s. [https://github.com/projectcalico/calico](https://github.com/projectcalico/calico)

- **Flannel** simples e fácil de configurar, projetado para o Kubernetes. [https://github.com/flannel-io/flannel](https://github.com/flannel-io/flannel)

- **Weave** é outra solução popular de rede para Kubernetes.[https://github.com/weaveworks/weave]()

- **Cilium** é um plugin de rede novo focado em segurança e desempenho. oferece recursos avançados, como balanceamento de carga, monitoramento e solução de problemas de rede. [https://cilium.io/](https://cilium.io/)

- **Kube-router** é uma solução de rede leve para Kubernetes.

Nesse caso vamos testar com `Weave Net`. Executar somente no Control Plane:

```
$ kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
**Não esquecer de liberar as portas do Weave Net**, nesse caso temos que fazer a liberação na AWS.

&nbsp;
Portas: 6783 TCP e 6784 UDP

&nbsp;
Se não houver a liberação os pods não irão subir.

Verificar a os pods do `Weave Net` após a instalação:
```
kubectl get pods -n kube-system
```

Para verificar se os pods de aplicações estão sendo criados corretamente após a instalação do cluster e plugin de rede, pode instalar uma API e validar:

```
kubectl create deployment nginx --image=nginx --replicas=3
kubectl get pods -o wide
```
Ver detalhes dos nodos criados:

```
kubectl get nodes -o wide
kubectl describe nodes k8s-01 -o wide 
```