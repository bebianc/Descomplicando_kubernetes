# Cert-Manager

cert-manager é um X.509 certificate controller poderoso e extensível para workloads do Kubernetes e OpenShift. Ele obterá certificados de vários Issuer (emissores), tanto Issuer públicos populares quanto Issuer privados, e garantirá que os certificados sejam válidos e atualizados, e tentará renovar os certificados em um horário configurado antes de expirarem.

**Recursos**:
 - Emissão e renovação automatizadas de certificados para proteger o Ingress com TLS
 - Issuers totalmente integrados com autoridades de certificação públicas e privadas reconhecidas
 - Comunicação segura entre pods com mTLS usando Issuers de PKI privados
 - Suporta certificados para Workloads internos e para a Web
 
O cert-manager pode obter certificados de uma variedade de autoridades de certificadoras, incluindo: Let's Encrypt, HashiCorp Vault, Venafi e PKI privada.

Com o `Certificate resource` do cert-manager, a chave privada e o certificado são armazenados em um `Secret` do Kubernetes que é montado por um Pod da aplicação ou usado por um `Ingress Controller`. Já com `csi-driver`, `csi-driver-spiffe` ou `istio-csr`, a chave privada é gerada sob demanda, antes da inicialização da aplicação; a chave privada nunca sai do nodo e não é armazenada em um Secret do Kubernetes.

**Issuer**

A primeira coisa que você precisará configurar depois de instalar o cert-manager é um Issuer ou ClusterIssuer. Estes são recursos do Kubernetes que representam autoridades de certificação (CAs) capazes de assinar certificados em resposta a solicitações de assinatura de certificados, ou seja, é o componente que irá fazer a requisição para as CAs para assinatura do certificado.

Namespaces:
Um Issuer é um recurso com namespace e não é possível emitir certificados de um Issuer em um namespace diferente. Isso significa que você precisará criar um Issuer em cada namespace no qual deseja obter Certificados.
Se você quiser criar um único Issuer que possa ser consumido em vários namespaces, considere criar um recurso ClusterIssuer. Isso é quase idêntico ao recurso Issuer, porém não tem namespace, portanto pode ser usado para emitir certificados em todos os namespaces.

**ClusterIssuer**

O recurso ClusterIssuer tem escopo de cluster. Isso significa que ao fazer referência a um secret por meio do campo secretName, os segredos serão procurados no Namespace de Recursos de Cluster. Por padrão, este namespace é cert-manager, mas pode ser alterado por meio de uma flag no componente cert-manager-controller:

```yaml
--cluster-resource-namespace=minha-namespace
```
Referência: https://cert-manager.io/

## Utilizando Let's Encrypt como CA

Importante saber que se o Let's Encrypt como Issuer no cert-manager, existem dois modos, o `letsencrypt-staging` recomendado para testes, pois não tem limite de requisições para a CA para validação do certificado, e o modo `letsencrypt-prod` recomendado para uso em produção, pois há um limite na quantidade de validações do certificado.


O Issuer do tipo `ACME` (Atomated Certificate Management Environment) é usado na configuração para emissão com Let's Encrypt. Quando criado um novo emissor ACME, o cert-manager irá gerar uma chave privada que é usada para identificá-lo no servidor ACME.

**Teste para validar se o certificado é real**

Para garantir que o certificado é válido quando realizado uma requisição a um endereço HTTPs, há dois testes ou desafios:
    
    - HTTP01 que é feito através de um endpoint de URL HTTP, aonde possui um arquivo específico no endpoint que o cliente vai acessar para validar a autenticidade. Utilizando o cert-manager o arquivo é gerado de forma automática. Esse formato é o que vamos utilizar nessa documentação.

    - DNS01 que é feito a partir de um registro DNS TXT, aonde é necessário adicionar uma entrada de um arquivo TXT no DNS.

- Se utilizando o Let's Encrypt, os certificados gerados são armazenados automaticamente no Secrets do Kubernetes, pelo cert-manager.
- Se já temos outros certificados, é possível substituí-los no Secrets ou armazenar por exemplo, no Vault.

## Instalação e Configuração do cert-manager

Instalação simples, conforme documentação:

*Lembre-se de olhar os pré-requisitos.

```yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.yaml
kubectl -n cert-manager get all 
```

### Criar um Issuer ACME básico 

Essa configuração irá criar um `ClusterIssuer` que será aplicação para todas as namespaces do cluster, usando o modo `staging`, sem limites de requisições para a CA. Como solver para testar se o certificado é válido utilizamos o modo HTTP01.

Para aplicar a configuração apenas para uma namespace específica deve mudar o **kind** para **Issuer**.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-staging
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

Essa configuração irá criar o Issuer no modo prod, com limitação de requisições para a CA.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-prod
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

Aplicar e validar configuração do Issuer:

```bash
kubectl apply -f ingress-clusterIssuer.yaml
kubectl get clusterissuers
kubectl get issuers
kubectl describe clusterissuers letsencrypt-staging
# secret criado automaticamente pelo Issuer
kubectl -n cert-manager get secret 
```

## Configurando o Ingress para usar o Cert-Manager e ter o HTTPS

Editando o ingress-dnslocal-giropops.yaml que já existe, adicionando um Annotation definindo que será utilizado o Cert-Manager para gerenciamento do certificado e o host que irá receber as configurações de TLS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: giropops-senhas
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: / 
    cert-manager.io/cluster-issuer: "letsencrypt-staging" #definir no ingress para usar o cert-manager com cluster-issuer
spec:
  ingressClassName: nginx
  tls:
  - hosts: # definindo qual é o host que irá receber as configurações de TLS
    - giropops-senhas.local
    secretName: giropops-senhas-tls # secret que será criado automatico com as informacoes de TLS
  rules:
  - host: giropops-senhas.local
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: giropops-senhas
            port: 
              number: 5000

```
```bash
kubectl delete ingress giropops-senhas
kubectl apply -f ingress ingress-dnslocal-giropops.yaml
kubectl get certificate 
kubectl get orders
kubectl get certificaterequests 
```
Nos comandos o `certificate`, `orders` e `certificaterequests` o campo READY precisa está `True`, caso contrário o certificado não será aplicado no DNS, é possível validar o motivo nos eventos do describe.
Nesse caso, como o domínio que definimos não está publicado, o Issuer não consegue validar autenticidade e falha na requisição ao certificado. Por isso, o recomendado é criar um cluster na AWS com ingress público.

# Annotations e Labels

É uma forma de passar informações para o objeto possa utilizar outro recurso específico. A diferença para o Label é que no Label você utliza para identificação e categorização dos Pods, linkando com os objetos internos do cluster como os services, endpoints, já o Annotations é usado para linkar com os CRDs (Custom Recources Definition). 
Cada recurso como o Ingress, Cert-Manager, Vault... possui annotations específicos para ser adicionado no manifesto da aplicação.

Alguns comandos com Label:

```bash
# filtrar todos os objetos que tenham uma label específica
kubectl get all --selector=giropops
# adicionar um label com chave valor em um pod especifico
kubectl label pods nomeCompletoDoPod app2=nova
# Sobreescrever uma chave específica do label do Pod
kubectl label pods nomeCompletoDoPod app2=velha --overwrite
# filtrar todos os pods que possuem a Label 'app'
kubectl get pods -L app
# Apagar o label app2
kubectl label pods nomeCompletoDoPod app2-
```
Exemplo de um manifesto explicando as **LABELs** do Deployment e do Pod.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: giropops-senhas  ## LABEL do Deployment
  name: giropops-senhas
spec:
  replicas: 2
  selector:
    matchLabels:
      app: giropops-senhas ## A mesma LABEL do Deployment precisa existir dando match na definição do POD.
  template:
    metadata:
      labels:
        app: giropops-senhas ## LABEL do POD, diferente da label do Deployment.
        app2: velha  # repare que essa LABEL não existe no Deployment, então se eu filtrar por 'app2' apenas o POD irá retornar.
    spec:
      containers:
      - image: bebianc/linuxtips-giropops-senhas_semvelnerabi:7.0
```

Alguns comandos com **Annotations**:
```bash
#Criar um anotation
kubectl annotate pods redis-deployment-785855684c-vmn89 description="Pod do redis para ser usado com o giropops-senhas"
kubectl describe pod redis-deployment-785855684c-vmn89 | grep "Annotation"
#Apagar um annotation
kubectl annotate pods redis-deployment-785855684c-vmn89 description-
```

Criar filtros mais específicos com Json:
```bash
# Filtrar no Pod do Redis o Metadata annotation
kubectl get pods redis-deployment-785855684c-vmn89 -o jsonpath='{.metadata.annotations}'
kubectl get pods redis-deployment-785855684c-vmn89 -o jsonpath='{.metadata.annotations.description}'
```

# Adicionando autenticação no Ingress

Trecho do ingress-dnslocal.giropops.yaml que adicionamos os Annotations para que o usuário se autentique no Nginx antes de acessar a aplicação.

```yaml
kind: Ingress
metadata:
  name: giropops-senhas
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/auth-type: "basic" # configura o nginx para autenticacao do tipo basic
    nginx.ingress.kubernetes.io/auth-secret: "giropops-senhas-users" # secret para armazenar as credenciais
    nginx.ingress.kubernetes.io/auth-realm: "Autenticação necessária" # mensagem de autenticação que irá aparecer em tela 
[...]    
```

Para criptografar um usuário e senha pode ser usado um recurso do Apache que é o `htpasswd`.
```bash
# Instalar
sudo apt install apache2-utils
# Gerar um arquivo 'auth' com a senha criptografada
htpasswd -c auth bebianc
cat auth
# Criar um secret do tipo genérico, com o arquivo 'auth' gerado
kubectl create secret generic giropops-senhas-users --from-file=auth
# Acompanhar o log do ingress para verificar o login
kubectl logs -f -n ingress-nginx ingress-nginx-controller-*
```