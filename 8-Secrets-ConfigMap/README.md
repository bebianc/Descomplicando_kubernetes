# Secrets

**O que são os Secrets**
- Uma forma de guardar informações sensíveis do cluster Kubernetes, tokens, certificados, senhas.
- Armazenados no etcd, que é o responsável por armazenar chave-valor do cluster. Por default, essas informações não são criptografadas no `etcd`.
- OS ``secrets`` não são criptografados por default, apenas embaralhados em base64, então vai do administrador criar estratégicas manter essas informações seguras, no cluster.

**Uso seguro dos Secrets**

1. Habilitar [Criptografia em Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) para Secrets.

**Tipos de Secrets**

Detalhes na documentação: 