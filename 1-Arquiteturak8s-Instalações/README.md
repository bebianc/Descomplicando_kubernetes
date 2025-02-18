# Kubernetes

## Resumo do Kubernetes
  - Orquestrador de container. Software responsável por gerenciar os containers.
  - Nomad é outra opção de orquestrador.
  - Iniciou com o Projeto Borg da Google e parte do projeto foi disponibilizado para a comunidade que criou o k8s.
  
Alguns tópicos que são interessante conhecer antes de se aprofundar no universo k8s.

### Container
  - Isolamento de recursos da máquina, como CPU, memória, tabelas de usuários, ponto de montagem, interface de rede, processos, mas consumindo o kernel da máquina.

#### Módulos do Kernel do Linux:
  - cgroup: isola recursos de memória e CPU.
  - namespaces do kernel: isolamento de processos, ponto de montagem, de usuários.

#### Container engine
  - Engine é responsável por criar os container os módulos citados a cima, em execução, sem conversar diretamente com o Kernel do Linux.  
  - Kubernetes não é mais compatível com o docker.

#### Container runtime
  - Conversa com o Kernel do linux. Faz a comunicação do container engine com o Kernel e garante que o cotainer esteja em execução.
  - *Containerd*: é o runtime do docker. High-level: não conversa diretamente com o kernel e precisa de um container engine. É o mais utilizado.
  - *Runc*: runtime Low-level: comunicação com o kernel. Utilizado por diferentes container engines como o docker.
  - *Gvisor*: é uma outra opção de container runtime.

  - Refs.: 
    - [https://github.com/containerd/containerd/blob/main/docs/getting-started.md](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
    - [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

#### OCI - Open Container Initiative
  - Padrão OCI implementado para criação de containers para compatibilidade com outros opensources. Desenvolveram o *runc*.

### Arquitetura k8s

 - **Control Plane**: Controlar e garantir a disponibilidade, saúde e armazenamento do estado do cluster. É o orquestrador do cluster. Não é recomendado ter APIs no Control Plane. Cria e gerencia os workers e a rede do cluster, como: namespaces, deployments, services, configmaps, secrets... *Componentes do Control Plane*: 
  - `ETCD` (portas TCP 2379 e 2380): banco do cluster que guarda o estado real e atual do cluster, armazena as informações de configuração de todo o control plane. Pode usar criptografia TLS. Só comunica com o Kube ApiServer. Plano de Backup para os dados: [https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
  - `kube-scheduler` (porta 10251): Reponsável por agendar aonde irá executar os novos Pods, selecionando um node para executá-los de acordo com os requisitos e os recursos disponíveis. Monitora a situação do cluster podendo ajustar a distribuição dos pods para garantir melhor utilização dos recursos.
  - `kube-controller-manager` (porta 10252): Controlador do cluster. É o core do cluster que vai gerenciar os diferentes controladores que regulam o estado do cluster (controllers: deployment, replicaset...). Monitora o estado atual dos recursos comparando com o estado desejado. 
  -  `kube-apiserver` (porta TCP 6443): Comunica com todos os componentes do Control Plane e com os Workers, bem como ferramentas externas, se comuniquem com o cluster, sendo a principal interface de comunicação do Kubernetes. Centraliza as informações para guardar no ETCD. É o front da camada de gerenciamento.
  - `cloud-controller-manager`: permite vincular o cluster API da nuvem para interação entre componentes.
   
 - **Workers**: Nodos aonde estão rodando as aplicações. Principal função é executar os pods.
 *Componentes*:
    - `Kubelet` (porta TCP 10250): agent do kubernetes em cada node, garantindo que os containers estejam funcionando corretamente. Monitora o estado atual comparando com o estado desejado. Comunica com o ApiServer do Control Plane passando informações.
    - `Kube Proxy`: Proxy de rede que executa em cada node, permite a comunicação de rede entre pods e services dentro ou fora do cluster. Observa o control plane para identificar mudanças, atualizando as regras de encaminhamento de tráfego.


 **Instalar kubectl**:
 - https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

```bash
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x kubectl
  mv kubectl /usr/local/bin 
```
   
**Instalar o kind**:
 - https://kind.sigs.k8s.io/docs/user/quick-start/#installation

```bash
# For AMD64 / x86_64
$ [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
# For ARM64
$ [ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-arm64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

- Se Docker instalado com rootless, para criar um cluster com kind seguir os passos a seguir:
 https://kind.sigs.k8s.io/docs/user/rootless/

```bash 
export DOCKER_HOST=unix://${XDG_RUNTIME_DIR}/docker.sock
```

**Comandos kind**:
 - criar um cluster com mais de 1 node:

```bash 
nano kubernetes_pick/day1/kind/kind-cluster.yaml
kind create cluster --config kind-cluster.yaml --name clusterk8s
```

- Ĩnformações pós instalação:
  - Control plane em execução em: https://127.0.0.1:39503
  - CoreDNS em execução em: https://127.0.0.1:39503/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  - Para debugar o cluster em caso de problema:

```bash  
kubectl cluster-info dump
#Deletar o cluster
kind delete cluster --name clusterk8s
```