### Kubernetes

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

  - Ref.: 
    - [https://github.com/containerd/containerd/blob/main/docs/getting-started.md](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
    - [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
