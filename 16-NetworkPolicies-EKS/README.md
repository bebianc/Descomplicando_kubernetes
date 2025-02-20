# Network Policies

É uma ferramenta essencial para a segurança e o gerenciamento eficaz da comunicação entre os Pods em um cluster Kubernetes.

NetPol é um conjunto de regras que definem como os Pods podem se comunicar entre si e com os outros endpoints de rede. 
Por padrão, os pods podem se comunicar livremente entre si. O que pode não ser ideal para todos os cenários.
As Network Policies permitem que você restrinja esse acesso, garantindo que apenas tráfego permitido possa fluir entre os Pods ou para endereços IP externos.

## Para que servem?

Usadas para:
 - **Isolar** Pods de tráfego não autorizado.
 - **Controlar** o acesso à serviços específicos.
 - **Implementar** padrões de segurança e conformidade.

 ## Conceitos fundamentais: Ingress e Egress

 - **Ingress**: Regras de ingresso controlam o tráfego de entrada para um Pod.
 - **Egress**: Regras de saída controlam o tráfego de saída de um Pod.

 Nas Network Policies será necessário especificar se uma regra é aplicada ao tráfego de entrada ou de saída.

 ## Como funcionam as Network Policies

 Utilizam `SELECTORS` para identificar grupos de Pods e definir regra de tráfego para eles. A política pode especificar:

 - **Ingress (entrada)**: quais Pods ou endereços IP podem se conectar a Pods selecionados.
 - **Egress (saída)**: para quais Pods ou endereços IP os Pods selecionados podem se conectar.

 ### Ainda não é padrão

 As Network Policies ainda não são um recurso padrão em todos os clusters Kubernetes.
 
 A AWS possui suporte a Network Policies no EKS, mas ainda não é padrão, é necessário instalar o `CNI da AWS` e depois habilitar o `Network Policy` nas configurações avançadas do CNI.

 Para verificar se seu cluster suporta Network Policies:

 ```bash
kubectl api-versions | grep networking
 ```
Se retornar `networking.k8s.io/v1` significa que o cluster suporta.

Se retornar `networking.k8s.io/v1beta1` significa que o cluster não suporta.

Se o cluster suporta, para implementar Network Policies você pode utilizar o `Calico`, `Weave Net` e o `Cilium` que são ferramentas mais atualizadas.

Referência Calico: https://docs.tigera.io/calico/latest/about
