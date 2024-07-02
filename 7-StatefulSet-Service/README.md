# StatefulSet

- Conceito de `stateless`: Aplicação que não precisam guardar o seu estado, com recursos isolados, não armazena dados e se uma operação for perdida, terá que ser refeita. Se o pod não precisa armanezar informação, iniciar de onde parou, não requer um identificador estável, é melhor utilizar Deployment ou ReplicaSet para criar suas replicas *stateless*.

- Conceito de `stateful`: Aplicação que guardam o seu estado atual e são executadas novamente com base no contexto das transações anteriores. Se uma transação stateful for interrompida, você conseguirá retomá-la praticamente de onde parou, pois os dados são armazenados.

- O `statefulSet` é o objeto API de *workload* usado para gerenciar aplicações *stateful*. Gerencia o deploy e escalonamento dos Pods e fornece garantias sobre a ordem e exclusividade desses Pods.

- Usado em aplicações que requerem um ou mais desses:
  - Único identificador de rede estável.
  - Armazenamento persistido (PV) estável.
  - Deploy e escalonamento (*scaling*) elegantes (*graceful*) e ordenados
  - Updates contínuos ordenados e automatizados, seguindo uma sequência.

- Quando usado o `StatefulSet` sempre será criado um `PV` para cada Pod replicado. E se o Pod for recriado será utilizado o mesmo PV, garantindo que usará e os mesmos dados, mesmos requisitos citados acima. Aplicações como banco de dados, mensageria, monitoração são alguns exemplos.

**Limitações:**
- Necessita de um PV provisioner baseado no `StorageClass` solicitado.
- Deletar ou diminuir o número de réplicas não excluirá os volumes relacionados, para garantia na segurança dos dados.
- Quando usado o `StatefulSet` é necessário criar um `Service` do tipo *Headless* que é um tipo de serviço que não contém um IP próprio. Ele permite a comunicação de rede entre os Pods de uma aplicação *StatefulSet*. O *Headless* é usado para controlar o nome de host DNS (no formato:`<pod-name>.<service-name>.<namespace>.svc.cluster.local`) criado pelo StatefulSet*. Isso permite que aplicações `stateful`, tenham um identificador de rede único, estável e prevísivel, facilitando a comunicação entre diferentes instâncias da mesma api.


**Funcionamento**
- Cada réplica do Pod criada a partir do mesmo spec, mas diferenciada por um índice e hostname. Cada Pod em um *StatefulSet* tem um índice persitente e um hostname vinculado a sua identidade.

Por exemplo: Um StatefulSet com 3 réplicas, ele criará os Pods com hostname: meupod-0, meupod-1, meupod-2. E o meupod-1 não será iniciado até que o Pod meupod-0 está disponível e pronto. Isso garante a ordem no deploy, scaling e updates.

