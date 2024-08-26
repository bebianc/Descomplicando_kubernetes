# Ingress

É um recurso do Kubernetes que vai gerenciar o acesso externo aos Services dentro do cluster.
Funciona como uma camada de roteamento HTTP/HTTPs.

**Tipos de Ingress:**
- Istio
- HAProxy
- Ingress Nginx Controller
- Traefik

**Funcionalidades**
- `Controlador de Ingress`: É a implementação real que satifaz um recurso ingres.
- `Regras de roteamento`: Definidas em um objeto YAML, essas regras determinam como as requisições externas devem ser encaminhadas aos Services internos. 
- `Backend padrão`: Um serviço de fallback (mesmo diante de uma falha ou limitação, o recurso seja capaz de oferecer uma alternativa viável para a requsição) para onde as requisições são encaminhadas se nenhuma regra de roteamento for correspondida.
- `Balanceamento de carga`: Distribuição automática de tráfego entre múltiplos pods de um serviço.
- `Terminação SSL/TLS`: Permite a configuração de certificado SSL/TLS para a terminação de criptografia no ponto de entrada do cluster.
- `Anexos de recursos`: Possibilidade de anexar recursos adicionais como ConfigMaps ou Secrets, que podem ser utilizados para configurar comportamentos adicionais como autenticação básica, lista de controle de acesso, etc.

## Configurando o Kind para suportar o Ingress

Ao criar um cluster Kind, podemos especificar várias configurações que incluem mapeamentos de portas e rótulos para os nodos.
Um exemplo da configuração do Kind habilitando o uso do Ingress está em: 1-Arquiteturak8s-Instalações/`kind-cluster-ingress.yaml`.
Para criar o cluster: 
```bash
# Antes de criar, é necessário adicionar a chave "net.ipv4.ip_unprivileged_port_start=80" no arquivo /etc/sysctl.conf
sudo vi /etc/sysctl.conf
# Reiniciar a máquina
init 0
sudo kind create cluster --config kind-cluster-ingress.yaml
sudo kubectl get nodes
```

## Instalando o Ingress Nginx Controller

Projeto do Ingress Nginx:
https://github.com/kubernetes/ingress-nginx


Instalar no Kind, conforme a documentação do Kind:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

Utilizar o comando `wait` do kubectl para liberar o shell quando os pods estiverem prontos:
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

Instalando em outros provides:
https://kubernetes.github.io/ingress-nginx/deploy/

## Criando uma regra de Ingress

Antes de criar as regras, é necessário fazer o deploy de uma aplicação para aplicar às regras:

Fazer o deploy das aplicações no cluster Kind para fazer os testes:

```bash
kubectl apply -f giropops-deployment.yaml
kubectl apply -f giropops-service.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
```

Criar um recurso de Ingress para o serviço giropops-senhas, exemplo no yaml: ingress.yaml

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress giropops-senhas
```
Na saída do comando get e describe, você dever ver o endereço IP do seu Ingress no campo Address.

```bash
kubectl get ingress giropops-senhas -o jsonpath='{.status.loadlBalancer.ingress[0].hostname}'
# para testar o acesso
curl ENDERECO_INGRESS/giropops-senhas
# Como nao definimos um endereço de HOST, chamamos com localhost
curl localhost/giropops-senhas
kube
```
**Trobleshooting**

Fazendo um teste, acessando no navegador com o endereço `localhost/giropops-senhas` definido no Ingress `ingress.yaml`, vimos que a aplicação quebra para carregar o layout e outras funções. Isso acontece porque o Ingress está redirecionando as requisições do / para o contexto `/giropops-senhas` e as funções que quebram não estão em outros paths. Você pode verificar quais funções estão quebrando e o erro fazendo um `inspect` na página.
Por isso a aplicação precisa estar corretamente implementada, para que todas as funções sejam carregadas corretamente.

No caso do Ingress, também conseguimos contornar essa situação, criando um novo YAML `ingress-correto.yaml`, que vai direcionas as chamadas do "/" para o path "/", assim irá encontrar o path de todas as funções. Chamando no navegador dessa forma: `localhost`, verá que a página irá responder corretamente.

## Criando múltiplas regras de ingress para o mesmo Ingress Controller

Para acesso a mais de uma aplicação que tem como path "/", é necessário adicionar diferentes `host` em cada ingress de cada aplicação (cada aplicação vai ter o seu ingress).
Nesse exemplo criamos um Pod e um Service Nginx, na porta 80:

```bash
kubectl run nginx --image nginx --port 80
kubectl expose pod nginx
kubectl get svc
```

Criamos um ingress adicionando um host com um DNS local fictício e adicionamos o DNS no /etc/hosts para conseguir acessar no navegador:

```bash
kubectl apply -f ingress-nginx.yaml
# também criamos um Ingress com um DNS para a aplicação giropops: ingress-dnslocal-giropops.yaml
kubectl apply -f ingress-dnslocal-giropops.yaml
```
No /etc/hosts adicionamos a seguinte entrada. Detalhe é que como não temos um certificado SSL no DNS e para evitar que o navegador redirecione para HTTPS, temos que criar o DNS como *.local:

```bash
127.0.0.1 ingress.nginx.local
127.0.0.1 giropops-senhas.local
```
## Instalando um cluster EKS para teste com Ingress

No Kind não passamos a classe do Ingress que estamos usando, pois o default do Kind é o Ingress Nginx.
Porém em Cloud Providers é necessário definir a classe que estamos usando. Então cada regra de Ingress que será criada, deverá especificar qual é a classe, pois poderá ter mais de um Ingress Controller por cluster.

```bash
eksctl create cluster --name=eks-cluster --version=1.30 --region=us-east-1 --nodegroup-name=eks-cluster-nodegroup --node-type=t2.medium --nodes=2 --nodes-min=1 --nodes-max=3 --managed
```

**Contextos no Kubernetes**

Diferentes acesso à clusters configurado na sua máquina, são chamados de `Contextos`. Com o comando `config` podemos ver qual é o contexto que está configurado no momento.

```bash
# Verificar em qual contexto estou trabalhando no momento
kubectl config current-context
# Verificar contextos disponíveis
kubectl config get-contexts
# Mudar para outro contexto
kubectl config use-context <nome-do-contexto>
```

### Instalar Ingress Nginx Controller na AWS

Comando da AWS da documentação do Ingress Nginx:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/aws/deploy.yaml
```
Ref.: https://kubernetes.github.io/ingress-nginx/deploy/

Se acessar no navegador com HTTP, pois essa instalação não está com certificado SSL, verá a página do Nginx com 404 not found. Isso quer dizer que o Ingress Nginx foi instalado com sucesso.

Fazer o deploy das aplicações no cluster da AWS para fazer os testes:

```bash
kubectl apply -f giropops-deployment.yaml
kubectl apply -f giropops-service.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
```

**Ingress Class**
Na definição das regras de Ingress, é sempre necessário informar o Ingress Class do Ingress que determinada aplicação irá utilizar.
```bash
IngressClassName: nginx # informar a classe do Ingress
kubectl apply -f ingress-giropops-aws.yaml
```
#### Configurando um domínio válido para o Ingress no EKS

```bash
# deletar o ingress atual sem o dominio valido
kubectl delete -f ingress-giropops-aws.yaml
# Aplicar o ingress com dominio válido
kubectl apply -f ingress-giropops-aws-dominio.yaml
```
