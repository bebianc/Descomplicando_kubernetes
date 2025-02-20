# Taints

Quando estamos falando de Taints, estamos falando sobre os nós, se eu quero "manchar" um nodo, eu adiciono uma Taint nele.
No Control Plane temos um exemplo real de uma Taint, repare a Taint "NoSchedule" adicionada ao nodo. 

```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes$ kubectl describe nodes clusterk8s-control-plane
Name:               clusterk8s-control-plane
Roles:              control-plane
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```
## Tipos de Taints

- `NoSchedule`: Se o nodo está configurado com essa Taint, significa que nenhum Pod novo será criado e executado nesse nodo.
- `NoExecute`: Se eu quero garantir que um nodo não tenha nenhum Pod em execução, adiciono essa Taint, com isso todos os Pods são movidos para os outros nós que não tenham essa Taint. Usada por exemplo para retirar o nó do cluster para fazer uma manutenção.
- `PreferNoSchedule`: Adicionando essa Taint no nodo, o `kube-scheduler` não irá agendar a execução de novos Pods nesse nó, a não ser que não haja outra opção, por exemplo, os recursos dos outros nós já foram esgotados, já estão sobrecarregados de Pods, o agendador poderá provisionar novos Pods nesse nodo.

## Aplicando um Taint 

Aplicar uma Taint não remover os Pods de um nodo:
```bash
kubectl taint node clusterk8s-worker manutencao=true:NoExecute
kubectl taint node clusterk8s-worker manutencao=true:NoSchedule
```

Remover a Taint do nodo, basta adicionar um "-" no final:
```bash
kubectl taint node clusterk8s-worker manutencao=true:NoExecute-
kubectl taint node clusterk8s-worker manutencao=true:NoSchedule-
```
**Importante**: O kubernetes não vai redistribuir, ou seja, não vai recriar os Pods para o nodo que foi adicionado novamente no Pool, para evita ficar descendo e subindo pod a todo momento. Para isso, você pode reiniciar o Deployment manualmente com o `rollout`:

```bash
kubectl rollout restart deployment nginx
```

# Tolerations

Adicionando uma regra de Toleration no Deployment de uma aplicação, com os valores que referenciam uma Taint configurada em um nodo, o `kube-scheduler` irá agendar essa aplicação para executar no nodo, não importando o tipo de Taint. 
Esse cenário pode ser usado quando, por exemplo, uma determinada aplicação precisa executar em um nodo que tem Hardware diferente dos demais, mesmo que esse esteja com regra `NoSchedule`.

A configuração de Toleration no deployment da aplicação é a seguinte:
```yaml
      tolerations:
      - key: "gpu" #taint que foi criada no nodo
        operator: "Equal" #chave igual ("Equal") ao valor abaixo
        value: "true"  
        effect: "NoSchedule" # passar o efeito da taint
```

Aplicando a taint NoSchedule no worker "clusterk8s-worker" e aplicando o deployment com Toleration:
```bash
kubectl taint node clusterk8s-worker manutencao=true:NoSchedule
kubectl apply -f nginx-gpu-toleration.yaml
```

Temos o seguinte cenário:

```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/15-Taint-Tolerations-Affinity$ kubectl get pods -o wide
NAME                                    READY   STATUS    RESTARTS       AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
giropops-senhas-7bc7b65c4d-7cw4s        1/1     Running   11 (91m ago)   26d   10.244.1.3   clusterk8s-worker2   <none>           <none>
nginx-canary                            1/1     Running   9 (91m ago)    20d   10.244.2.3   clusterk8s-worker3   <none>           <none>
nginx-gpu-toleration-5bff49b5bf-6hmsn   1/1     Running   0              6s    10.244.1.7   clusterk8s-worker2   <none>           <none>
nginx-gpu-toleration-5bff49b5bf-tx7n5   1/1     Running   0              6s    10.244.3.7   clusterk8s-worker    <none>           <none>
```

Um pod "nginx-gpu-toleration-5bff49b5bf-tx7n5" foi provisionado no "clusterk8s-worker", mesmo com a Taint `NoSchedule` e outro pod "nginx-gpu-toleration-5bff49b5bf-6hmsn" foi criado no worker "clusterk8s-worker2". Isso quer dizer que mesmo com o Toleration, a regra não impede que a aplicação seja criada em outros nodos.

# Labels, Affinity e AntiAffinity

Para seja possível que um Pod execute em um determinado nodo, não em qualquer um, é necessário ter Labels e regra de `Affinity` definidas.
E o `AntiAffinity`, quando determinado, o Pod não poderá executar em um ou mais nodos.

**Exemplo**: vamos separar o cluster simulando o conceito de região e mult AZ da AWS.

Separando os nós por Região da AWS:
Nó `clusterk8s-worker` está na região `sa-east-1`.
```bash
# Adicionando Label para separar por regiao
kubectl label node clusterk8s-worker region=sa-east-1
```
Nós `clusterk8s-worker2` e `clusterk8s-worker3` estão na região `us-east-1`.
```bash
# Adicionando Label para separar por regiao
kubectl label node clusterk8s-worker2 region=us-east-1
kubectl label node clusterk8s-worker3 region=us-east-1
```
Separando os nós por Zonas de disponibilidade:

Nó `clusterk8s-worker` está no AZ 'A' da região `sa-east-1`, então adicionamos mais uma Label:
```bash
# Adicionando Label para separar por AZ
kubectl label node clusterk8s-worker az=sa-east-1a
```
Nó `clusterk8s-worker2` está no AZ 'A' da região `us-east-1`:
```bash
# Adicionando Label para separar por AZ
kubectl label node clusterk8s-worker2 az=us-east-1a
```
Nó `clusterk8s-worker3` está no AZ 'B' da região `us-east-1`:
```bash
# Adicionando Label para separar por AZ
kubectl label node clusterk8s-worker3 az=us-east-1b
```
Validando as labels:
```bash
kubectl get nodes --show-labels
kubectl get nodes -L az,region
```

Com isso temos um cluster de alta disponibilidade, em diferentes regiões e diferentes AZs.

## Criando Affinity no Pod

Com as Labels criadas nos nós, precisamos criar as regras de Affinity para garantir que o Pod seja provisionado no nó específico que queremos.
A ideia é garantir que o Pod caia sempre no conjunto de nós que ele está mais preparado para executar com qualidade, por exemplo se uma aplicação
recursos de Hardware específicos para executar, podemos utilizar o Affinity para forçar que rode em nós específicos.

No deployment nginx-affinity.yaml, declaramos qual é a regra para provisionar os Pods em nodos especifícos:

```yaml
[...]
      affinity:
        nodeAffinity: # regra para informar qual será o nó correto
          requiredDuringSchedulingIgnoredDuringExecution: # Significa que é requirido (required) q a regra seja aplicada no momento de Schedule do Pod ignorando a execução
          #preferDuringSchedulingIgnoredDuringExecution: # Significa que de "preferencia" (prefer) a regra sera aplicada no momento de Schedule do Pod ignorando a execução
            nodeSelectorTerms: # define qual vai ser o termo selecionado no nodo
            - matchExpressions:
                - key: "region" # vai fazer match com a label regiao do nodo
                  operator: "In" # dentro do valor da chave
                  values:
                  - "us-east-1" 
```

Repare que mesmo tendo 3 nós, somente o 2 receberam os pods, pois estão vinculados pela Label `region=us-east-1`.

```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/15-Taint-Tolerations-Affinity$ kubectl get pods -o wide | grep "region"
nginx-affinity-region-84988bf8c8-8fb4r   1/1     Running   0              35s    10.244.1.12   clusterk8s-worker2   <none>           <none>
nginx-affinity-region-84988bf8c8-dzpps   1/1     Running   0              35s    10.244.2.13   clusterk8s-worker3   <none>           <none>
nginx-affinity-region-84988bf8c8-m7wxm   1/1     Running   0              35s    10.244.1.13   clusterk8s-worker2   <none>           <none>
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/15-Taint-Tolerations-Affinity$ kubectl get nodes -L region
NAME                       STATUS   ROLES           AGE   VERSION   REGION
clusterk8s-control-plane   Ready    control-plane   27d   v1.30.0   
clusterk8s-worker          Ready    <none>          27d   v1.30.0   sa-east-1
clusterk8s-worker2         Ready    <none>          27d   v1.30.0   us-east-1
clusterk8s-worker3         Ready    <none>          27d   v1.30.0   us-east-1
```

## Criando regra de AntiAffinity

As regras de `AntiAffinity` ajudam a definir por exemplo, se mais de uma réplica de um Pod não pode executar em um único nó.
No kubernetes há um termo que é `topologyKey` onde é possível definir a Label do nodo de anti afinidade.

Nesse exemplo, definimos que não será possível provisionar mais de um Pod com label "app: nginx-antiaffinity-region", em um único nó que possui a Label "Region".

```yaml
[...]
affinity:
        podAntiAffinity: # regra para informar qual será os Pods que entrarao na regra de anti afinidade.
          requiredDuringSchedulingIgnoredDuringExecution: # Significa que é requirido (required) q a regra seja aplicada no momento de Schedule do Pod ignorando a execução
          - labelSelector: # define qual vai ser a label do Pod que irá ser aplicado a regra
            matchLabels:
              app: nginx-antiaffinity-region # tudo que fizer match com a "app: nginx-affinity-region" ira ser aplicado a regra de anti afinidade
            topologyKey: "region" # nao permitira provisionar mais de um Pod com label "app: nginx-affinity-region" no mesmo nodo que possui a label "region"
```

Ao aplicar o deployment `nginx-antiAffinity.yaml` um dos Pods ficaram com status `Pending`:

```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/15-Taint-Tolerations-Affinity$ kubectl get pods -o wide | grep "nginx-antiaffinity-region"
nginx-antiaffinity-region-7f5d74c55-g9j8p   1/1     Running   0               4m24s   10.244.3.4    clusterk8s-worker    <none>           <none>
nginx-antiaffinity-region-7f5d74c55-n2fwm   1/1     Running   0               4m24s   10.244.1.14   clusterk8s-worker2   <none>           <none>
nginx-antiaffinity-region-7f5d74c55-rpr5k   0/1     Pending   0               4m24s   <none>        <none>               <none>           <none>
```

Nos eventos do pod temos a seguinte mensagem:

```bash
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  2m26s  default-scheduler  0/4 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 3 node(s) didn't match pod anti-affinity rules. preemption: 0/4 nodes are available: 1 Preemption is not helpful for scheduling, 3 No preemption victims found for incoming pod.
```
Significa que há 4 nodos disponíveis no cluster, 1 possui Taint de NoSchedule por ser o Master e 3 nodos não deram match devido a regra de anti-affinity do Pod.
 Nesse caso a regra foi aplicada com sucesso, pois somente 1 Pod foi provisionado pelo kube-scheduler, no nodo com a Label region.

## Usando o preferredScheduling e o requiredScheduling

A regra de "preferredScheduling" abaixo significa que de preferência será escalado 1 Pod "nginx-preferredScheduling" no host que possui o label "az".
Porém se não houver outros nodos, vai provisionar no mesmo para manter o número de réplicas estipulado.

```yaml
affinity:
        podAntiAffinity: # regra para informar qual será os Pods que entrarao na regra de anti afinidade.
          preferredDuringSchedulingIgnoredDuringExecution: # Significa que de preferencia (preferred) a regra seja aplicada no momento de Schedule do Pod ignorando a execução
          - weight: 1 # aplicando um peso para a regra. Se tiver duas regras "preferred" será aplicada a regra que tiver o peso mais alto
            podAffinityTerm: # usado somente para o "preferred" para "required" é "nodeSelectorTerms"
              labelSelector:
                matchLabels:
                  app: nginx-preferredScheduling
              topologyKey: "az" # De preferencia será escalado 1 Pod "nginx-preferredScheduling" no host que possui o label "az"
                  
```