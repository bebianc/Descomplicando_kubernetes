# Volumes

Sempre que um Pod é criado é gerado algum tipo de dado, quando o Pod é morto os dados são perdidos.
Uma maneira para preservar esses dados é através de volumes.
Tipos de Volumes:
 - `Ephemeral volumes`: são volumes criados e destruídos junto com o Pod, por exemplo o `EmptyDir`.
 - `Persistent volumes`: são volumes criados e não são destruídos junto com o Pod. Exemplo, o `StorageClass` que provisiona os PVs (PersistentVolumes) e PVCs (PersistentVolumeClaims)
 

 **StorageClass** (classe de storage) é uma forma de categorizar ou descrever uma classe de disco/armazenamento, que pode ser criada pelos administradores do cluster, para permitir que os usuários escolham a classe adequada para sua necessidade sobre o tipo de armazenamento.
 
Cada StorageClass contém os campos `provisioner`, `parameters` e `reclaimPolicy`, que são usados para automatizar e gerenciar a criação dos PVs (PersistentVolume), através dos `provisioners`. Os PVs são provisionados dinamicamente de acordo com os requisitos dos PersistentVolumeClains (PVCs).

Quando um PVC não especifica uma classe de armazenamento, é usado uma StorageClass default.
 
 **Provisioner**: É como uma *engine* que provê um armazenamento. Cada StorageClass tem um provisionador que determina qual plugin de volume é usado para criar o PV. 
  - Na criação de um StorageClass se especificar  qualquer nome no campo `provisioner` será necessário criar um provisoner manualmente. Uma boa pátrica se caso não tenha um provisoner de um provedor de numve ou onde o kubernetes está sendo executado (Kind), uma opção é inserir o "kubernetes.io/no-provisioner". Esse nome indica que o provisoner será criado manual. Então de forma resumida, o provisioner muda com forme o local aonde será criado, se for usar um da AWS, deverá ter o nome do provisioner da AWS, se for na Azure, mesma coisa. Exemplos de provisionadores padrão que criam os PVs:

Container Storage Interface (CSI): interface que permite com que seja criado soluções de storage/volume para o uso no kubernetes:

- `kubernetes.io/aws-ebs`: AWS Elastic Block Store (EBS)
- `kubernetes.io/azure-disk`: Azure Disk
- `kubernetes.io/gce-pd`: Google Compute Engine (GCE) Persistent Disk
- `kubernetes.io/cinder`: OpenStack Cinder
- `kubernetes.io/vsphere-volume`: vSphere
- `kubernetes.io/no-provisioner`: Volumes locais
- `kubernetes.io/host-path`: Volumes locais (não é muito utilizado)
- `rancher.io/local-path`: Kind
- `kubernetes.io/external-nfs`: NFS server

Lista de provisioners:
[https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner).

Ver os StorageClass disponíveis:
``` 
 kubectl get storageclass
```
Mais detalhes:
``` 
 kubectl describe storageclass standard
 kubectl describe storageclasses.storage.k8s.io

```
Detalhes do retorno do comando.
Quando está com a opção *IsDefaultClass: Yes* significa que essa é a classe de armazenamento default do cluster e será usada pelos PVCs que não tenham um StorageClass definido.

```bash
Name:            standard
IsDefaultClass:  Yes
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"standard"},"provisioner":"rancher.io/local-path","reclaimPolicy":"Delete","volumeBindingMode":"WaitForFirstConsumer"}
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           rancher.io/local-path
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

Para criar uma StorageClass simples, usando o *provisioner* de volume local, pode verificar o exemplo do manisfesto *storageclass.yaml*.

**PersistentVolume** *PV*: É como se fosse o disco, um bloco para armazenamento. Representa um recurso de armazenamento físico em um cluster kubernetes.
Os dados não são perdidos se o container é reiniciado ou movido.

- Tipos de PV: 
  - armazenamento local: irá guardar o dado no próprio host (host-path). Não é sugerido para ambiente de produção, mas para ambiente de teste.
  - armazenamento em rede: NFS (*Network File System*), diretório compartilhado na rede para o uso do cluster. ISCI permite conexão de dispositivos de armazenamento via rede IP.
  - armazenamento em nuvem: AWS Elastic Block Store (EBS), Azure Disk, Google Compute Engine (GCE) Persistent Disk.

Lista de tipos de armazenamentos suportados:
[Kubernetes Docs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes).

Comandos:
``` 
 kubectl get persistentvolume
 kubectl get pv -A

```
Verificar exemplo do manifesto `persistentvolume.yaml`.


**Configurando um servidor NFS integrando com PV**

- Criar o diretório compartilhado:

```bash
mkdir /mnt/nfs
#mudar o dono do diretório
chown bebianc:bebianc /mnt/nfs
```
- Instalar o NFS server e o cliente para testar:

```bash
sudo apt-get install nfs-kernel-server nfs-common
``` 
- Editar o arquivo de configuração do NFS:
  
```bash
sudo nano /etc/exports
# Adicionar o diretório que será compartilhado
/mnt/nfs *(rw,sync,no_root_squash,no_subtree_check)

```
Onde:

- `/mnt/nfs`: é o diretório que você deseja compartilhar.

- `*`: permite que qualquer host acesse o diretório compartilhado. Para maior segurança, você pode substituir * por um intervalo de IPs ou por IPs específicos dos clientes que terão acesso ao diretório compartilhado. Por exemplo, 192.168.1.0/24 permitiria que todos os hosts na sub-rede 192.168.1.0/24 acessassem o diretório compartilhado.

- `rw`: concede permissões de leitura e gravação aos clientes.

- `sync`: garante que as solicitações de gravação sejam confirmadas somente quando as alterações tiverem sido realmente gravadas no disco.

- `no_root_squash`: permite que o usuário root em um cliente NFS acesse os arquivos como root. Caso contrário, o acesso seria limitado a um usuário não privilegiado.

- `no_subtree_check`: desativa a verificação de subárvore, o que pode melhorar a confiabilidade em alguns casos. A verificação de subárvore normalmente verifica se um arquivo faz parte do diretório exportado.

Falar para o NFS que o diretório `/mnt/nfs` está disponível

```bash
sudo exportfs -arv
```
Verificar se o NFS está funcionando
 ```bash
showmount -e
```
```bash
Export list for localhost:
/mnt/nfs *
```
- O Kubernetes não possui um provisioner NFS nativo, então é necessário criar um PV utilizando o NFS server.
Exemplo de um manifesto k8s PV com NFS acima.
- Na criação desse PV NFS, campo `storageClassName` deve associá-lo a um Storage Class, sugiro criar um Storage Class específico para esse tipo de armazenamento. Exemplo em anexo.


**PersistentVolumeClaim** *PVC*: É o recurso que solicitará para o StorageClass o pedaço de armazenamento (PV) para uso.
Permite que os usuários ou aplicação solicitem um volume específicom com base no tamanho, tipo de armazenamento...

O Kubernetes irá entregar o melhor disco disponível seguindo as configurações do PVC.

Exemplo de manifesto PVC em pvc.yaml.

Com o PVC pronto, como a regra definimos a regra `volumeBindingMode: WaitForFirstConsumer` no Storage Class "giro", temos que configurar uma API para usar esse PV.

Criando um Pod para solicitar um PVC.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
    - name: pvc #de acordo com o nome do pvc
      mountPath: /usr/share/nginx/html
  volumes:
  - name: pvc
    persistentVolumeClaim:
      claimName: pvc
```

