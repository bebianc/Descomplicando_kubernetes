# StatefulSet

- Conceito de `stateless`: Aplicação que não precisam guardar o seu estado, com recursos isolados, não armazena dados e se uma operação for perdida, terá que ser refeita. Se o pod não precisa armanezar informação, iniciar de onde parou, não requer um identificador estável, é melhor utilizar Deployment ou ReplicaSet para criar suas replicas *stateless*.

- Conceito de `stateful`: Aplicação que guardam o seu estado atual e são executadas novamente com base no contexto das transações anteriores. Se uma transação stateful for interrompida, você conseguirá retomá-la praticamente de onde parou, pois os dados são armazenados.

- O `statefulSet` é o objeto API de *workload* usado para gerenciar aplicações *stateful*. Gerencia o deploy e escalonamento dos Pods e fornece garantias sobre a ordem e exclusividade desses Pods.

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

# Services

- Permite que os Pods sejam acessados de fora do cluster ou de fora da rede.
- Para cada replica de Pod criada, o Service irá distribuir as requisições para cada uma.
- O Service fornece uma abstração que define um conjunto lógico de Pods e uma política para a cessá-los. O conjunto de Pods são determinados por meio de Label Selectors. Importante as Labels de cada recurso do Kubernetes estarem bem definidas, para que o Service possa saber qual é o Pod que ele precisa direcionar a requisição. Se por exemplo, duas APIs diferentes estiverem com a mesma Label acidentalmente, o Service vai direcionar as requisições para as duas APIs e podemos ter um problema.
- Quando criado um Service, é criado um ENDPOINT, que é o endereço IP para comunicação com o Pod. Esse objeto Endpoint rastreia os IPs e as portas dos Pods que correspondem aos critérios de seleção do Service.
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

*Testar também com MetalLB*

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

