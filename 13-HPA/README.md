# Horizontal Pod Autoscaler (HPA)

Recurso que permite ajustar automaticamente o número de réplicas de um conjunto de pods, assegurando que nosso aplicativo tenha sempre recursos necessários para performar eficientemente, sem desperdiçar recursos.
O HPA monitora as métricas dos pods. Em um intervalo de tempo, ele avalia se os pods estão sobrecarregados ou estão com pouca carga. 
Com base nessa avaliação, ele toma a decisão de escalar mais pods ou reduzir para uma quantidade segura para diminuir os recursos do cluster.

O `Metric Server` é o componente que irá fornecer as métricas necessárias para que o HPA tome decições de escalonamento.

## Metrics Server

O Metric Server é um agregador de métricas de recursos de sistema, que coleta métricas como uso de CPU e memória dos `nós` e `pods` do cluster.

O HPA utiliza essas métricas de uso de recursos para tomar decisões sobre o escalonamento. 
Por exemplo, se o uso de CPU de um pod exceder um determinado limite, o HPA pode decidir aumentar o número de réplicas desse pod. 
Da mesma forma, se o uso de CPU for muito baixa, o HPA pode decidir reduzir o número de réplicas. 
Para fazer isso de forma eficaz, o HPA precisa ter acesso a métricas precisas e atualizadas que são fornecidas pelo Metrics Server.

Para pegar outros tipos de métricas é possível especificar métricas do Prometheus, no metrics server.

### Instalando o Metrics Server

Para instalação no `Kind` e no `EKS` é o mesmo comando:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get pods -n kube-system | grep metrics-server
```

Se o container demorar e não subir, ficando *0/1 Running* provavelmente o Readiness probe falhou.
Verificar os logs do pod para identificar o erro.

```bash
kubectl describe pods metrics-server-77fd554d9d-rdsv8 -n kube-system
kubectl logs -f metrics-server-77fd554d9d-rdsv8
```
Se o seu Kind não possui um certificado para o nodos, irá retornar um erro de certificado no log do metrics-server.

```bash
Failed to scrape node" err="Get \"https://172.18.0.3:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.18.0.3 because it doesn't contain any IP SANs" node="clusterk8s-control-plane"
```
Para contornar, será necessário instalar um certificado para o cluster ou ignorar a validação de certificado, no deployment do metrics-server.

```bash
kubectl edit deploy metrics-server -n kube-system
```
Para Ignorar adicione a chave `kubelet-insecure-tls`, nos argumentos do container.

```yaml
spec:
      containers:
      - args:
        - --kubelet-insecure-tls
```

Verificar se a instalação foi bem sucedida:
```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top pods 
kubectl top nodes
```

## Criar um HPA

```bash
# Criar um deployment do Nginx
kubectl create deployment nginx-hpa --image nginx --port 80 --dry-run=client -o yaml > nginx-hpa-deployment.yaml
# Editar as configurações conforme o manifesto nginx-hpa-deployment.yaml, adicionando a limits de CPU e memória
# Depois aplicar as configurações
kubectl apply -f nginx-hpa-deployment.yaml
kubectl expose deploy nginx-hpa
```
Criando um HPA para o deployment `nginx-hpa` para realizar scale up ou scale down, de acordo com a utilização de CPU em 50%.

```bash
kubectl apply -f hpa1.yaml
kubectl get hpa
kubectl describe hpa
# Verá novos pods sendo criado de acordo com o minimo configurado no HPA
kubectl get pods
```
## Instalando o Locust para Stress Test

Aplicar o service.yaml, confimap e Deployment do Locust, com a imagem da linuxtips.
Porém pode ser instalado de outras maneiras seguindo a documentação: https://locust.io/

**Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: locust
  name: locust
spec:
  replicas: 1
  selector:
    matchLabels:
      app: locust
  template:
    metadata:
      labels:
        app: locust
    spec:
      containers:
      - image: linuxtips/locust-giropops:1.0
        name: locust
        env:
          - name:  LOCUST_LOCUSTFILE
            value: "/usr/src/app/scripts/locustfile.py"
        ports:
        - containerPort: 8089
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: locust-scripts
          mountPath: /usr/src/app/scripts
      volumes:
      - name: locust-scripts
        configMap:
          name: locust-scripts
          optional: true
```
**Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: locust
spec:
  selector:
    app: locust
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8089
  type: ClusterIP
```

No **ConfigMap** declaramos a configuração das requisições de como iremos realizar o teste, dependendo do cenário da sua aplicação. 
Há várias maneiras e alguns exemplos também estão na documentação oficial.
```yaml
apiVersion: v1
data:
  locustfile.py: |-
    from locust import HttpUser, task, between
    from requests.auth import HTTPBasicAuth

    class Giropops(HttpUser):
        wait_time = between(1, 2)

        def on_start(self):
            self.auth = HTTPBasicAuth('username', 'senha') # usar se aplicação requer autenticacao


        @task(1)
        def listar_senha(self):
            self.client.get("/spsmb/azure/ia/doRetrieve?id=S3-dd8b9942-3f7f-1232-aa4b-7cd2ba675678", auth=self.auth) # contexto da requisição, metodo de autenticacao
kind: ConfigMap
metadata:
  name: locust-scripts
```

## Criando HPA baseado em CPU e memória

Basta adicionar no hpaCpuMemory.yaml esse trecho incluindo a utilização por recurso de memória, em 50%. Nesse caso quando atingir 50% de memória, irá escalar mais um pod.

```yaml
[...]
 - type: Resource # ira se basear no tipo de resources      
    resource:
      name: memory
      target:
        type: Utilization 
        averageUtilization: 50 
```        

## Utilizando configurações avançadas para o ScaleUP e ScaleDown usando HPA

Nesse exemplo, adicionamos o parâmetro `behavior`, no hpaCpuMemory.yaml e definimos as regras de scaleUp e scaleDown.

```yaml
[...]
behavior:
    scaleUp:
      stabilizationWindowSeconds: 5 # depois de 5s atingindo o 'averageUtilization' irá escalar o Pod
      policies:
      - type: Percent # Policie que indica que 100% das vezes essa regra tem que ser respeitada
        value: 100 
        periodSeconds: 10 # a cada 10s valida essas regras da politica  
    scaleDown:
      stabilizationWindowSeconds: 300 # 5min é o tempo que o HPA irá esperar para fazer o scaleDown
      policies:
      - type: Percent # Policie que indica que 100% das vezes essa regra tem que ser respeitada
        value: 100 
        periodSeconds: 10 # a cada 10s valida essas regras da politica          
``` 

No parâmetro "stabilizationWindowSeconds" geralmente é configurado 0 segundos para escalar imediatamente, porém dependendo da situação o HPA pode ficar intermitente, subindo e descendo pods a todo momento e isso pode comprometer o bom funcionamento da aplicação.

No teste de stress é possível acompanhar o comportamento do deployment fazendo o escalonamento, com o parâmetro `-w` para contar a quantidade de réplicas em execução.
```bash
kubectl get deployment giropops-senhas -w
```