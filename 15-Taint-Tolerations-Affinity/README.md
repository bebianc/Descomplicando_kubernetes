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

## Affinity

Para seja possível que um Pod execute em um determinado nodo, não em qualquer um, é necessário ter Labels e regra de `Affinity` definidas.
