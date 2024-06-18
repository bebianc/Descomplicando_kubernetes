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

 - **Control Plane**: Controlar e garantir a disponibilidade, saúde e armazenamento do estado do cluster. É o orquestrador do cluster. Não é recomendado ter APIs no Control Plane. Cria e gerencia os workers e a rede do cluster, como: namespaces, deployments, services, configmaps, secrets...
  
 *Componentes do Control Plane*: 
    - ETCD (portas TCP 2379 e 2380): banco do cluster que guarda o estado real e atual do cluster, armazena as informações de configuração de todo o control plane. Pode usar criptografia TLS. Só comunica com o Kube ApiServer. Plano de Backup para os dados: [https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
    - kube-scheduler (porta 10251): Reponsável por agendar aonde irá executar os novos Pods, selecionando um node para executá-los de acordo com os requisitos e os recursos disponíveis. Monitora a situação do cluster podendo ajustar a distribuição dos pods para garantir melhor utilização dos recursos.
    - kube-controller-manager (porta 10252): Controlador do cluster. É o core do cluster que vai gerenciar os diferentes controladores que regulam o estado do cluster (controllers: deployment, replicaset...). Monitora o estado atual dos recursos comparando com o estado desejado. 
    -  kube-apiserver (porta TCP 6443): Comunica com todos os componentes do Control Plane e com os Workers, bem como ferramentas externas, se comuniquem com o cluster, sendo a principal interface de comunicação do Kubernetes. Centraliza as informações para guardar no ETCD. É o front da camada de gerenciamento.
    - cloud-controller-manager: permite vincular o cluster API da nuvem para interação entre componentes.
   
 - **Workers**: Nodos aonde estão rodando as aplicações. Principal função é executar os pods.
 *Componentes*:
    - Kubelet (porta TCP 10250): agent do kubernetes em cada node, garantindo que os containers estejam funcionando corretamente. Monitora o estado atual comparando com o estado desejado. Comunica com o ApiServer do Control Plane passando informações.
    - Kube Proxy: Proxy de rede que executa em cada node, permite a comunicação de rede entre pods e services dentro ou fora do cluster. Observa o control plane para identificar mudanças, atualizando as regras de encaminhamento de tráfego.