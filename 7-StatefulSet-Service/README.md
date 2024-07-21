# StatefulSet

- Conceito de `stateless`: Aplicação que não precisam guardar o seu estado, com recursos isolados, não armazena dados e se uma operação for perdida, terá que ser refeita. Se o pod não precisa armanezar informação, iniciar de onde parou, não requer um identificador estável, é melhor utilizar Deployment ou ReplicaSet para criar suas replicas *stateless*.

- Conceito de `stateful`: Aplicação que guardam o seu estado atual e são executadas novamente com base no contexto das transações anteriores. Se uma transação stateful for interrompida, você conseguirá retomá-la praticamente de onde parou, pois os dados são armazenados.

- O `statefulSet` é o objeto API de *workload* usado para gerenciar aplicações *stateful*. Gerencia o deploy e escalonamento dos Pods e fornece garantias sobre a ordem e exclusividade desses Pods. É uma maneira de gerenciar um conjunto de Pods replicados.

- Usado em aplicações que requerem um ou mais desses:
  - Único identificador de rede estável.
  - Armazenamento persistido (PV) estável.
  - Deploy e escalonamento (*scaling*) elegantes (*graceful*) e ordenados
  - Updates contínuos ordenados e automatizados, seguindo uma sequência.

- Quando usado o `StatefulSet` sempre será criado um `PV` para cada Pod replicado. E se o Pod for recriado será utilizado o mesmo PV, garantindo que usará e os mesmos dados, mesmos requisitos citados acima. Aplicações como banco de dados, mensageria, monitoração são alguns exemplos.

**Limitações:**
- Necessita de um PV provisioner baseado no `StorageClass` solicitado.
- Deletar ou diminuir o número de réplicas não excluirá os volumes relacionados, para garantia na segurança dos dados.
- Quando usado o `StatefulSet` é necessário criar um `Service` do tipo *Headless* que é um tipo de serviço que não contém um IP próprio. Ele permite a comunicação de rede entre os Pods de uma aplicação *StatefulSet*. O *Headless* é usado para controlar o nome de host DNS (no formato:`<pod-name>.<service-name>.<namespace>.svc.cluster.local`) criado pelo StatefulSet*. Isso permite que aplicações `stateful`, tenham um identificador de rede único, estável e prevísivel, facilitando a comunicação entre diferentes instâncias da mesma api.


**Funcionamento**
- Cada réplica do Pod criada a partir do mesmo spec, mas diferenciada por um índice e hostname. Cada Pod em um *StatefulSet* tem um índice persitente e um hostname vinculado a sua identidade.

Por exemplo: Um StatefulSet com 3 réplicas, ele criará os Pods com hostname: meupod-0, meupod-1, meupod-2. E o meupod-1 não será iniciado até que o Pod meupod-0 está disponível e pronto. Isso garante a ordem no deploy, scaling e updates.

Exemplo de manifesto *StatefulSet* em statefulset.yaml.

Se der o apply no statefulset.yaml e executar os comandos get e describe e verá que os pods foram criados com nome nginx-0, nginx-1 e nginx-2, e foram criados em sequência, após o anterior ficar pronto. 

É possível ver também que os PVs e PVCs foram criados, um para cada Pod. Perceba que o CLAIM é de acordo com o nome de volume criado no "nginx-persistent-storage", no StatefulSet, usando o StorageClass default do cluster "standard".

```bash
ebianc@BerUbuntu22:~/Descomplicando_kubernetes/7-StatefulSet-Service$ kubectl  get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                      STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
primeiro-pv                                1Gi        RWO            Retain           Available                                              giro           <unset>                          46h
pvc-221fafac-cc6d-45f6-849f-96a9482b3218   1Gi        RWO            Delete           Bound       default/nginx-persistent-storage-nginx-1   standard       <unset>                          22h
pvc-338e19d9-e2d8-40da-a385-066749be6a93   1Gi        RWO            Delete           Bound       default/nginx-persistent-storage-nginx-2   standard       <unset>                          22h
pvc-dbd4f969-f504-4555-a04e-34a78c950baf   1Gi        RWO            Delete           Bound       default/nginx-persistent-storage-nginx-0   standard       <unset>                          22h
pvnfs                                      1Gi        RWO            Retain           Available                                              nfs            <unset>                          46h
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/7-StatefulSet-Service$ kubectl  get pvc
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nginx-persistent-storage-nginx-0   Bound    pvc-dbd4f969-f504-4555-a04e-34a78c950baf   1Gi        RWO            standard       <unset>                 22h
nginx-persistent-storage-nginx-1   Bound    pvc-221fafac-cc6d-45f6-849f-96a9482b3218   1Gi        RWO            standard       <unset>                 22h
nginx-persistent-storage-nginx-2   Bound    pvc-338e19d9-e2d8-40da-a385-066749be6a93   1Gi        RWO            standard       <unset>                 22h
```

Outro detalhe é que é importante especificar no MountPath do Volume o caminho correto que a aplicação irá executar. 
Por exemplo: nginx - /usr/share/nginx/html ; apache - /var/www/html

**Criando um Service Headless**

Necessário criar um service headless para vincular ao StatefulSet, para a comunicação entre os Pods, usando o DNS local de cada Pod, ao invés de IP.

Exemplo de manifesto em `nginx-headless-svc.yaml`.

Após dar um apply no manifesto, podemos validar a comunicação entre os Pods, atráves do DNS. No Exemplo abaixo foi conectado em cada Pod, criado um index.html no Nginx, somente para facilitar o entendimento no retorno do comando curl nginx-0.nginx.default.svc.cluster.local. 
```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/7-StatefulSet-Service$ kubectl exec -it nginx-2 bash 
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@nginx-2:/# echo "Pod 2" > /usr/share/nginx/html/index.html
root@nginx-2:/# curl nginx-2.nginx.default.svc.cluster.local
Pod 2
root@nginx-2:/# curl nginx-1.nginx.default.svc.cluster.local
Pod 1
root@nginx-2:/# curl nginx-0.nginx.default.svc.cluster.local
Pod 0
```
Para deletar os recursos criados, basta dar um delete nos STS, SVC e nos três PVCs. Assim que os PVCs forem deletados, os PVs serão deletados automaticamente.

Deletar o StatefulSet sem deletar os Pods:
```bash
kubectl delete statefulset meu-statefulset --cascade=false
```

# Services

- Permite que os Pods sejam acessados de fora do cluster ou de fora da rede.
- Para cada replica de Pod criada, o Service irá distribuir as requisições para cada uma.
- O Service fornece uma abstração que define um conjunto lógico de Pods e uma política para a cessá-los. O conjunto de Pods são determinados por meio de Label Selectors. Importante as Labels de cada recurso do Kubernetes estarem bem definidas, para que o Service possa saber qual é o Pod que ele precisa direcionar a requisição. Se por exemplo, duas APIs diferentes estiverem com a mesma Label acidentalmente, o Service vai direcionar as requisições para as duas APIs e podemos ter um problema.
- Quando criado um Service, é criado um **ENDPOINT**, que é o endereço IP para comunicação com o Pod. Esse objeto Endpoint rastreia os IPs e as portas dos Pods que correspondem aos critérios de seleção do Service. Eles são responsáveis por manter o mapeamento entre o Service e os Pods que ele está expondo.
Eles mantêm um endereço IP estável e uma porta de serviço que permanecem constantes ao longo do tempo, mesmo que os Pods subjacentes sejam substituídos.

```bash
kubectl get endpoints meu-service
```
Tipos de Service:
 - **ClusterIP**: Será atribuído ao Service um IP local para comunicação interna do cluster.
 - **NodePort**: Reserva uma porta no nodo e expõe o Service na mesma porta. A requisição externa que foi feita para essa porta reservada, será direcionada para acesso interno ao cluster. Isso é feito através do protocolo NAT. Portas disponíves para o NodePort 30000 a 32767. Importante se atentar a liberação no firewall.
 - **LoadBalancer**: Cria um balanceador de carga externo no ambiente e atribui um IP fixo, externo ao cluster, ao Service. Interessante criar um Service LoadBalancer quando estiver usando um Cloud Provider, um EKS, por exemplo, onde o Control Plane é executado a partir da AWS, assim o service se comunica com a AWS e é criado um LB automaticamente para ser acessado.
   - Caso não esteja utilizando um Cloud Provider, é possível integrar o Service LoadBalancer com uma ferramenta de LB local instalada no Cluster kubernetes, por exemplo `MetalLB`, aonde ele vai entregar para o Service LoadBalancer um range de IP definido.
 - **ExternalName**: Mapeia o Service para o conteúdo do campo externalName (por exemplo, foo.bar.example.com), retornando um registro CNAME com seu valor. Um Service ExternalName é um serviço para um recurso externo, por exemplo um banco de dados que está fora do cluster kubernetes. Nesse caso o nome do serviço será o endereço externo do banco, e os Pods que estiverem com esse nome de serviço vinculados, conseguirão acessar o banco externo atráves desse ALIAS vinculado ao endereço do banco.
 Outro uso comum para ExternalName é quando você tem ambientes diferentes, como produção e desenvolvimento, que possuem serviços externos diferentes. Você pode usar o mesmo nome de serviço em todos os ambientes, mas apontar para diferentes endereços externos.

Exemplo de como criar um Service por comando. Lembrando que o recomendado é via manifesto.

**ClusterIP**
Criar um deploy para vincular a um Service do tipo `ClusterIP`:
```bash
kubectl create deployment nginx-service --image=nginx --port=80
```

Expor como Service o deployment criado:
```bash
kubectl expose deployment nginx-service
kubectl get svc
kubectl get endpoints #Se escalar mais de uma replica, o Service saberá para qual encaminhar a requisição de acordo com os endereços dos ENDPOINTS.
```
Exemplo de manifesto ClusterIP em *nginx-clusterIP-svc.yaml*

**NodePort**:
Vinculando o Deployment a um Service tipo `NodePort`:
```bash
kubectl expose deployment nginx-service --type NodePort
kubectl get svc
kubectl get endpoints # Se escalar mais de uma replica, o Service saberá para qual encaminhar a requisição de acordo com os endereços dos ENDPOINTS.
```

Interessante o retorno do "get svc", pois mesmo sendo um NodePort, o ClusterIP é preenchido. 
Campo EXTERNAL-IP, preenchido quando Service LoadBalancer.
Campo Ports, mostra a porta do Service (80) e a porta que Porta do Node (30234), que será mapeada para a porta 80 do Service.
Para acessar o Service adicione o IP do Nodo: 192.168.0.10:30234, por exemplo.
```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/7-StatefulSet-Service$ kubectl get svc -o wide
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        42d   <none>
nginx-service   NodePort    10.96.114.171   <none>        80:30234/TCP   98s   app=nginx-service
```
Exemplo de manifesto NodePort em *nginx-nodePort-svc.yaml*

**LoadBalancer**

Para testar o tipo LoadBalancer utilizando um cloud provider, pode subir um AWS -EKS, Google Cloud - GKE, Azure - AKS. Nesse caso vamos subir um cluster EKS na AWS.

Comando exemplo para criar um cluster EKS:
```bash
eksctl create cluster --name=eks-cluster-1 --version=1.23 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t3.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed
```
Quando criado o cluster e um deployment no cluster, basta expor a o deployment com tipo LoadBalancer, e acompanhar com "get svc", verá que será atribuido ao Service um endereço do LB na AWS, no campo EXTERNAL-IP:
```bash
kubectl expode deployment nginx-service --type=LoadBalancer --port=80 --target-port=8080
kubectl get svc
```
Se acessar o endereço do LB no browser, aguardando o tempo de propagação do DNS, conseguirá acessar a página do Nginx.

Exemplo de manifesto LoadBalancer em *nginx-loadBalancer-svc.yaml*

Esse tipo de Service pode ser considerado quando deseja expor um serviço a ser acessado externamente por usuários ou sistemas que não estão dentro do seu cluster Kubernetes. 

**MetalLB**

MetalLB se conecta ao seu cluster Kubernetes e fornece uma implementação de balanceador de carga de rede. Ele permite criar serviços Kubernetes do tipo LoadBalancer em clusters que não possuem um balanceador de carga de um cloud provider.

Possui dois recursos que funcionam juntos para fornecer este serviço: address allocation e external announcement.

```Address Allocation```: Em um cluster Kubernetes em um provedor de nuvem, você solicita um balanceador de carga e sua plataforma de nuvem atribui um endereço IP a você. Em um cluster bare-metal, o MetalLB é responsável por essa alocação. O MetalLB não pode criar endereços IP do nada, então você precisa fornecer conjuntos de endereços IP que ele possa usar. O MetalLB se encarregará de atribuir e cancelar a atribuição de endereços individuais à medida que os serviços vão e vêm, mas só distribuirá IPs que façam parte de seus pools configurados. A forma como você obtém pools de endereços IP para MetalLB depende do seu ambiente. Seu provedor de hospedagem provavelmente oferece endereços IP para locação. Nesse caso, você alugaria, digamos, um /26 de espaço IP (64 endereços) e forneceria esse intervalo ao MetalLB para serviços de cluster. Alternativamente, seu cluster pode ser privado, fornecendo serviços para uma LAN próxima, mas não exposto à Internet. Nesse caso, você poderia escolher um intervalo de IPs de um dos espaços de endereço privados (os chamados endereços RFC1918) e atribuí-los ao MetalLB. Esses endereços são gratuitos e funcionam bem, desde que você forneça apenas serviços de cluster para sua LAN. Ou você pode fazer as duas coisas! MetalLB permite definir quantos pools de endereços você quiser e não se importa com o “tipo” de endereços que você fornece.

```External announcement```: Depois que o MetalLB atribui um endereço IP externo a um serviço, ele precisa conscientizar a rede além do cluster de que o IP “vive” no cluster. MetalLB usa rede padrão ou protocolos de roteamento para conseguir isso, dependendo do modo usado: ARP, NDP ou BGP.

  ```Modo camada 2```: No modo de camada 2, uma máquina no cluster assume a propriedade do serviço e usa protocolos padrão de descoberta de endereços (ARP para IPv4, NDP para IPv6) para tornar esses IPs acessíveis na rede local. Do ponto de vista da LAN, a máquina anunciadora simplesmente possui vários endereços IP.
Mais informações do [modo camada 2](https://metallb.universe.tf/concepts/layer2/)

  ```BGP```: No modo BGP, todas as máquinas no cluster estabelecem sessões de peering BGP com roteadores próximos que você controla e informam a esses roteadores como encaminhar o tráfego para os IPs de serviço. O uso do BGP permite um verdadeiro balanceamento de carga em vários nós e um controle de tráfego refinado graças aos mecanismos de política do BGP. Mais informações do modo [BGP](https://metallb.universe.tf/concepts/bgp/)

```Requisitos de instalação```:
 - Kubernetes versão 1.13 ou superior
 - Uma configuração de rede de cluster que pode coexistir com o MetalLB. Exemplo: Calico, Flannel, Cilium, Weave Net...
 - Alguns endereços IPv4 para MetalLB distribuir.
 - Ao usar o modo operacional BGP, você precisará de um ou mais roteadores capazes de falar BGP.
 - Ao usar o modo operacional L2, o tráfego na porta 7946 (TCP e UDP, outra porta pode ser configurada) deve ser permitido entre nós.

 ```Preparação```:
  - Se estiver usand kube-proxy, necessário habilitar o modo *strict ARP*: 
   ```bash
   kubectl edit configmap -n kube-system kube-proxy
   ``` 
   Setar: 

   ```yaml
        apiVersion: kubeproxy.config.k8s.io/v1alpha1
        kind: KubeProxyConfiguration
        mode: "ipvs"
        ipvs:
          strictARP: true
   ```
   Para automatizar essa mudança verificar a documentação: [Preparação para instalar MetalLB](https://metallb.universe.tf/installation/)

`Instalação por manifesto`:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml
```
Isso implantará o MetalLB em seu cluster, na namespace *metallb-system*. Os componentes do manifesto são:
 - O deployment metallb-system/controller. Este é o cluster-wide controller (controlador do cluster todo) que lida com atribuições de endereços IP.
- O metallb-system/speaker daemonset. Este é o componente que fala "speaker" os protocolos de sua escolha para tornar os serviços acessíveis.

O manifesto de instalação não inclui um arquivo de configuração. Os componentes do MetalLB ainda serão iniciados, mas permanecerão idle até você começar a fazer o deploy dos recursos.

Ao fazer o deploy do o Pod do *controller* retornar erro *crashloopbackoff*, conforme abaixo: 

```bash
{"caller":"main.go:257","error":"too many open files","level":"error","msg":"failed to run k8s client","op":"startup","ts":"2024-07-20T11:32:26Z"}
```

Validar aumentar os parâmetros do sistema relacionados ao inotify, que é uma ferramenta que monitora mudanças no sistema de arquivos.

Para saber os valores atuais:
```bash
sysctl fs.inotify.max_user_instances
sysctl fs.inotify.max_user_watches
```

Para aumentar os valores permanentemente:
```bash
sudo nano /etc/sysctl.conf
#Adicionar as linhas:
fs.inotify.max_user_instances=1280
fs.inotify.max_user_watches=655360
#Aplicar a mudança
sudo sysctl -p
```
Considerações: Verifique a quantidade ideal para o seu cluster, pois pode passar a consumir mais recurso.

[Integração com o Prometheus para monitoramento](https://metallb.universe.tf/installation/).

`Configuração da definição dos IPs`:
 - Definir os IPs a serem atribuídos aos services do Load Balancer, para isso o MetalLB deverá ser instruído a fazê-lo através do CR IPAddressPool.

 ```yaml
  apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    name: config-metallb
    namespace: metallb-system
  spec:
    addresses:
    - 192.168.10.0/24
    - 192.168.9.1-192.168.9.5
    - fc00:f853:0ccd:e799::/124
 ```

- Várias instâncias de IPAddressPools podem coexistir e os endereços podem ser definidos por CIDR, por um range, e endereços IPV4 e IPV6. 

`Configuração do modo camada 2`:
- O modo Camada 2 é o mais simples de configurar: em muitos casos, não é necessária nenhuma configuração específica de protocolo, apenas endereços IP.
- Não exige que os IPs estejam vinculados às interfaces de rede dos workers. Ele funciona respondendo diretamente às solicitações ARP na sua rede local, para fornecer o endereço MAC da máquina aos clientes.
- Para anunciar o IP proveniente de um IPAddressPool, uma instância L2Advertisement deve estar associada ao IPAddressPool, exemplo:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: config-metallb
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.250-192.168.0.250

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
```
- Se necessário definir um IP específico para o modo camada 2, pode ser usado *selector*.


**ExternalName**
```bash
#Quando não referenciado um deployment para o Service é usado o "create"
kubectl create service externalname bancodedados --external-name db.bancodedados.com.br
kubectl get svc
```
Irá retornar um serviço com tipo ExternalName e com EXTERNAL-IP o DNS do banco.

**Criando um Service apontando para outro Service**

Expor um Service já criado, a partir de um outro Service, pode ser uma estratégia para fazer um troubleshooting na sua estrutura, com algum problema relacionado com uma API que não está funcionando corretamente ou uma nova instalação que precisa ser validada. Nesse caso criamos um novo Service com outro tipo, para a API ser acessada de fora do cluster, sem mudar a configuração de Service original dela.
Você também pode ter mais de um Service de outro tipo para o seu Pod, Deployment, Sts, Ds, ReplicaSet...para que o recurso seja acessado de diferentes formas, ou seja, tendo um Service tipo ClusterIP e outro tipo NodePort, o seu Pod poderá ser acessado tanto pela rede interna da sua organização por você ou por outra API, quanto internamente, por outros Pods. Ou também um tipo LoadBalancer para acesso via internet.

Como funciona:
```bash
#Verificar quais services criados e são atrelados ao deployment da API
kubectl get svc -n namespace_app
#Digamos que a API está vinculada a Service do tipo ClusterIP e vc precisa expor esse Service para acesso fora do cluster
kubectl expose service --name nome_novo_service nome_service_exposto --type NodePort
#Se dar um get, verá que a porta do novo Service estará apontando para a porta do Service ClusterApi que queremos usar e as requisições são direcionadas para ela
kubectl get svc
```
**Dica**: para criar manifestos do Kubernetes com autocomplete, use o plugin do Kubernetes com Vs Code.

