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

A primeira coisa que você precisará configurar depois de instalar o cert-manager é um Issuer ou ClusterIssuer. Estes são recursos do Kubernetes que representam autoridades de certificação (CAs) capazes de assinar certificados em resposta a solicitações de assinatura de certificados.

Namespaces:
Um Issuer é um recurso com namespace e não é possível emitir certificados de um Issuer em um namespace diferente. Isso significa que você precisará criar um Issuer em cada namespace no qual deseja obter Certificados.
Se você quiser criar um único Issuer que possa ser consumido em vários namespaces, considere criar um recurso ClusterIssuer. Isso é quase idêntico ao recurso Issuer, porém não tem namespace, portanto pode ser usado para emitir certificados em todos os namespaces.

**ClusterIssuer**
O recurso ClusterIssuer tem escopo de cluster. Isso significa que ao fazer referência a um secret por meio do campo secretName, os segredos serão procurados no Namespace de Recursos de Cluster. Por padrão, este namespace é cert-manager, mas pode ser alterado por meio de uma flag no componente cert-manager-controller:

```yaml
--cluster-resource-namespace=minha-namespace
```
Referência: https://cert-manager.io/

**Utilizando Let's Encrypt como CA**

Importante saber que se o Let's Encrypt como Issuer no cert-manager, existem dois modos, o `letsencrypt-staging` recomendado para testes, pois não tem limite de requisições para a CA para validação do certificado, e o modo `letsencrypt-prod` recomendado para uso em produção, pois há um limite na quantidade de validações do certificado.

O Issuer do tipo ACME (Atomated Certificate Management Environment) é usado na configuração para emissão com Let's Encrypt. Quando criado um novo emissor ACME, o cert-manager irá gerar uma chave privada que é usada para identificá-lo no servidor ACME.
Para garantir que os clientes não possam solicitar certificados para domínios que não sejam de sua propriedade, há dois testes ou desafios:
 - HTTP01 que é feito através de um endpoint de URL HTTP.
 - DNS01 que pe feito a partir de um registro DNS TXT.
