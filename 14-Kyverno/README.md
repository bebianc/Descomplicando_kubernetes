# Kyverno

Kyverno é uma ferramenta de gerenciamento de políticas para Kubernetes, focada na automação de várias tarefas relacionadas à segurança e configuração dos 
clusters de Kubernetes. Ele permite que você defina, gerencie e aplique políticas de forma declarativa para garantir que os clusters e suas cargas de trabalho 
estejam em conformidade com as regras e normas definidas.

### Principais funções:

1. `Validação de Recursos`: Verifica se os recursos do Kubernetes estão em conformidade com as políticas definidas. Por exemplo, pode garantir que todos os
Pods tenham limites e CPU de memória definidos.
2. `Mutação de Recursos`: Modifica automaticamente os recursos do Kubernetes para atender às políticas definidas. Por exemplo, pode adicionar automaticamente 
labels específicos a todos os novos Pods.
3. `Geração de Recursos`: Cria recursos adicionais do Kubernetes com base nas políticas definidas. Por exemplo, pode gerar NetworkPolicies para cada novo
Namespace criado.

## Instalando através do Helm

Pode ser feita de várias maneiras, diretamente através de arquivos YAML ou pelo gerenciador de pacotes como o Helm.

### Utilizando o Helm 

O Helm é um gerenciador de pacotes para Kubernetes, que facilita a instalação e gerenciamento de aplicações.

1. Adicione o repositório do Kyverno para pegar todos os charts disponíveis:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm repo update
```
2. Instalar o Kyverno no namespace `kyverno`:

```bash
#kyverno é no nome do repo / kyverno que é o nome do chart
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
kubectl get pods -n kyverno
kubectl get crd | grep kyverno
```

`Documentação` oficial com informações para policies para AWS, CAST AI, Istio, Argo e outros: https://kyverno.io/

Na aba **Playground** é possível criar policies e simular.

## Criando uma Policy no Kyverno

As políticas do Kyverno podem ser aplicadas de duas maneiras: a nível de cluster (**ClusterPolicy**) ou a nível de namespace (**Policy**).

1. **ClusterPolicy**: Ela é aplicada a todos os namespaces do cluster. Ou seja, as regras definidas em uma ClusterPolicy são automaticamente aplicadas a
todos os recursos correspondentes em todos os namespaces, a menos que especificamente excluídos.
2. **Policy**: Se deseja aplicar apenas a um namespace, você usaria o tipo Policy.

**Importante**: se não especificar um namespace as políticas serão definidas globalmente, em todos os namespaces.

### Exempo de políticas

1. Política de Limite de Recursos: Garantir que todos os containers em um Pod tenham limites de CPU e memória definidos. Isso pode ser importante para evitar
uso excessivo de recursos em um cluster compartilhado.

### Criando ClusterPolicy usando Validate no Kyverno

Nessa regra criamos um manifesto com política ClusterPolicy, que não permite (Enforce) a criação de Pod em qualquer namespace, sem os limites de cpu e memória não definidos. 

```yaml
apiVersion: kyverno.io/v1 
kind: ClusterPolicy # regra para todo o cluster
metadata:
  name: require-resources-limits
  # aqui não precisa definir namespace já que é uma policy para global
spec:
  validationFailureAction: Enforce # aqui defino se será "audit" ou "enforce", nesse caso "enforce", pois nao deve permitir de forma alguma
  rules: 
  - name: validate-limits
    match:
      resources:
        kinds: # aqui defino quais serão os objetos que farão parte da "rule", Pod, Deployment, Sts...
        - Pod 
    validate: # regra que define que será apenas validado, não será alterado (mutate) a tentantiva de criação de um pod sem limits.
      message: "Necessário definir o limite de recursos" # mensagem que será mostrada se cair na regra da validação.
      pattern: # define o padrão da regra
        spec: # spec que declara como deve ser o padrão de criação de containers, nesse caso com cpu e memória definidos
          containers: 
          - name: "*" # aplica para todos os containers
            resources:
              limits:
                cpu: "?*" # qualquer valor precisa ser definido
                memory: "?*" # qualquer valor precisa ser definido
```                

```bash
kubectl apply -f require-resource-limits.yaml
kubectl get clusterpolicies.kyverno.io
kubectl describe clusterpolicies.kyverno.io require-resources-limits
```

#### Validando a regra:

Criar um Pod qualquer, nesse caso usamos o nginx:
```bash
kubectl run nginx --image nginx --dry-run=client -o yaml > podsemlimites.yaml
```
Ajustar o yaml `podsemlimites.yaml` deixando apenas o básico, sem os recursos de memória e CPU definidos e aplicar.
Verá que irá o Kyverno irá bloquear a criação do Pod, retornando a mensagem que definios na política.
```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes/14-Kyverno$ kubectl apply -f podsemlimites.yaml 
Error from server: error when creating "podsemlimites.yaml": admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/default/nginx was blocked due to the following policies 

require-resources-limits:
  validate-limits: 'validation error: Necessário definir o limite de recursos. rule
    validate-limits failed at path /spec/containers/0/resources/limits/'

```
A mesma mensagem será exibida se definir apenas um dos recursos, memória ou CPU.

### Adicionar Labem ao Namespace usando o Mutate no Kyverno

O `Mutate` do Kyverno é projetado para realizar alguma alteração seja no Pod, deployment, namespace e etc. Nesse caso, vamos utilizar uma política para 
adicionar uma label. 

A política `add-label-namespace` é projetada para automatizar a adição de um label específico a todos os Namespaces em um cluster. 

**Detalhes da política**: O label adicionado por esta política é `Gato: Bartolomeu`. A aplciação deste label a todos os namespaces facilita a identificação 
e a categorização dos mesmos, permitindo uma gestão mais eficiente e uma padronização no uso de labels.

**Utilização da política**: Garante que cada Namespace no cluster seja automaticamente etiquetado com `Gato: Bartolomeu`. Isso é útil para garantir conformidade 
e a uniformidade na atribuição de labels, facilitando operações como filtragem e busca de Namespace, com base em critérios específicos.

### Criando uma Policy do tipo Generate

A política `generate-configmap-for-namespace` é uma estratégia prática para automatizar a criação de ConfigMaps em Namespaces, simplificando a configuração e gestão de múltiplos ambientes em um cluster.

Exemplo da regra está no manifesto **generate-configmap-for-namespace.yaml**;

```bash
kubectl apply -f generate-configmap-for-namespace.yaml
kubectl get clusterpolicies.kyverno.io
kubectl describe clusterpolicies.kyverno.io generate-configmap-for-namespace
kubectl create ns bichos
# irá mostrar o CM criado automaticamente após a criação da namespace
kubectl get cm -n bichos
```

### Criar policy para proibir root nos containers

A política `disallow-root-user` é uma regra de segurança crítica no gerenciamento do Kubernetes. Ela proíbe a execução de containers com usuário root dentro dos Pods. Este controle ajuda a previnir possíveis vulnerabilidades de segurança e a reforçar as melhores práticas no ambiente de contêineres.

Aplicando a política, os pods que tentarem executar containers como usuário root serão impedidos, com a exibição de uma mensagem de erro, indicando que a execução como root não é permitida. Isso assegura uma camada adicional de segurança no ambiente Kubernetes, evitando práticas que possam comprometer a integridade e a segurança do cluster.

```bash
kubectl apply -f disallow-root-user.yaml
kubectl get clusterpolicies.kyverno.io
# Tentar criar um Pod com container com usuário root, irá retornar a mensagem.
kubectl apply -f podssemlimites.yaml
```