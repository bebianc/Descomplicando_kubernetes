# Secrets

**O que são os Secrets**
- Uma forma de guardar informações sensíveis do cluster Kubernetes, tokens OAuth, certificados, senhas.
- Armazenados no etcd, que é o responsável por armazenar chave-valor do cluster. Por default, essas informações não são criptografadas no `etcd`.
- OS ``secrets`` não são criptografados por default, apenas embaralhados em base64, então vai do administrador criar estratégicas manter essas informações seguras, no cluster.

**Uso seguro dos Secrets**

1. Habilitar [Criptografia em Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) para Secrets.
2. Habilitar e configurar [regras RBAC](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) com menos privilégios no acesso dos usuários e Pods aos Secrets.
3. Restringir acesso aos Secrets de containers específicos.
4. Considerar usar um [provedor de armazenamento de Secrets externo](https://secrets-store-csi-driver.sigs.k8s.io/concepts.html#provider-for-the-secrets-store-csi-driver).

*[Boas práticas para Secrets em Kubernetes](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

**Uso para Secrets**

- [Setar variável de ambiente para um container](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#define-container-environment-variables-using-secret-data).
- [Prover credenciais como chave SSH ou senhas para os Pods](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#provide-prod-test-creds).
- [Permitir o `kubelet` fazer pull de imagens de containers com privilégios restritos](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).
- Control Plane também usa Secrets, por exemplo o Secret do token de inicialização ajuda a automatizar o registro do nodo.

**Tipos de Secrets**

- `Opaque`: Mais simples, dados arbitrários e genéricos definidos pelo usuário, como senhas, tokens, certificado TLS, chaves, tudo em base64.

```bash
kubectl create secret generic empty-secret
kubectl get secret empty-secret
```

- `kubernetes.io/service-account-token`: Armazena credencial de token que identifica um ServiceAccount.
- `kubernetes.io/dockercfg`: Guardar as configurações dos arquivos ocultos .dockercfg, usado para salvar as credenciais de acesso ao DockerHub, quando é feito o login. No Kubernetes são utilizados pelo Pods para se autenticar no registry DockerHub ou em um registry privado.
- `kubernetes.io/dockerconfigjson`: basicamente a mesma funcão do dockercfg.
- `kubernetes.io/basic-auth`: Bastante utilizado para credenciais que precisam de autenticações básicas. Deve conter pelo menos uma dessas chaves: *username* ou *password*.
- `kubernetes.io/ssh-auth`: Armazena credenciais para autenticação SSH entre cliente e servidor.
- `kubernetes.io/tls`: Armazena o certificado TLS e associado a chave usada para TLS. 
- `bootstrap.kubernetes.io/token`: armazena o token de inicialização do cluster para autenticar os nodos no Control Plane.

**Importante lembrar**
 - Definir qual Secret usar em determinada situação.
 - Os Secrets não são criptografados, são armazenados em base64, pois é muito fácil transformar base64 para texto.
 
 Exemplo:
 ```bash
 # Dar um get nas informações do Secret
 kubectl get secrets -n kube-system bootstrap-token-abcdef -o yaml
 # Pegar uma chave do yaml e tirar o base64
 echo -n "YWJjZGVmZyU=" | base64 -d
 ```

**Criando um Secret tipo Opaque**
- Exemplo de manifesto de `Secret tipo Opaque` em secrets-opaque.yaml.
- `Opaque`: Mais simples, dados arbitrários e genéricos definidos pelo usuário, como senhas, tokens, certificado TLS, chaves, tudo em base64.

- Para aplicar em uma namespace específica (recomendado): 
```bash
 kubectl apply -f secrets.opaque-yaml -n namespace
 ```

- Para criar esse mesmo Secret via comando:
```bash
 kubectl create secret generic secret-opaque --from-literal=username=<SEGREDO> --from-literal=password=<SEGREDO>
 ```
 O parâmetro `--from-literal` é usado para definir os dados (segredo, por exemplo) Secret.
 Parâmetro `--from-file` que define o Secret a partir de um arquivo.
 Parâmetro `--from-env-file` que defini o Secret a partir de uma variável de ambiente.
 O Comando `create` codifica os dados em Base64 automaticamente.

 - **Adicionar o Secret em um Pod**, via variável de ambiente, exemplo de manifest em secrets-pod.yaml.
 - Após adicionar é possível acessar o Pod e verificar que foi adicionado na variável de ambiente os dados USERNAME e PASSWORD:
```bash
bebianc@BerUbuntu22:~/Descomplicando_kubernetes$ kubectl exec -it myapp -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=myapp
NGINX_VERSION=1.27.0
NJS_VERSION=0.8.4
NJS_RELEASE=2~bookworm
PKG_RELEASE=2~bookworm
PASSWORD=senhasecretopaque
USERNAME=opaque
```

**Criando um Secret *dockerconfigjson* para autenticar no Docker Hub e usar imagens privadas**

- `kubernetes.io/dockerconfigjson`: basicamente a mesma funcão do dockercfg.

- o Docker Hub ajustou as suas políticas de download (*limit rate*), com isso há uma quantidade específica de pull de imagens:
  - Usuários não autenticados - 100 pulls por 6 horas por endereço IP
  - Usuários auteticados- 200 pulls por 6 horas
  - Usuários com assinatura paga na Docker - 5000 pulls por dia

Nesse caso vamos criar um Secret para realizar a autenticação no Docker Hub e garantir os 200 pull de imagem por 6 horas.

Se executar o comando *docker login* veremos que a autentição é automática. Isso ocorre devido o token de autenticação já está configurado.
Você pode encontrar as configurações de autentição em:
```
cat $HOME/.docker/config.json
```
Vamos usar o conteúdo desse json para criar o nosso Secret. Primeiro vamos transformar em base64:
```
base64 $HOME/.docker/config.json
```
Criar um Secret conforme exemplo em *secrets-dockerhub.yaml*. Colar o conteúdo do config.json no parâmetro *data*, nesse manifesto.

Criar um Pod para usar esse Secret para se autenticar no Docker Hub e fazer o pull da imagem privada, exemplo em *secret-pod-dockerhub.yaml*.

**Criando um Secret TLS**

- `kubernetes.io/tls`: Armazena o certificado TLS e associado a chave usada para TLS. 

Necessitamos então de uma chave privada e um certificado TLS, para modificarmos para base64 e criar o Secret.
Para isso seguir o comando `openssl`:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout chave-privada.key -out certicado.crt
```
Parâmetro: `-nodes` para que a chave privada não seja criptografada com uma senha
Parâmetro: `-days` para validade do certificado.
Parâmetro: `-newkey` definir o algoritmo de criptografia da chave privada 

Com isso gerou um certificado TLS (Transport Layer Security) auto-assinado que será usado para autenticar e estabelecer uma conexão segura entre cliente e servidor. E uma chave privada, que será usada para descriptografar a informação da comunicação entre as duas partes, que foi criptografada com a chave pública.

Criando o Secret com certificado e chave privada:
```bash
kubectl create secret tls app-web-tls-secret --cert=certificado.crt --key=chave-privada.key
```

Agora para usar esse Secret TLS, podemos criar um webservice como NGINX rodando com HTTPS.

