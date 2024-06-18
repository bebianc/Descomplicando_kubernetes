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