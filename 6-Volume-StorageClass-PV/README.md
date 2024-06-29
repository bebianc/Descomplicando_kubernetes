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

  - `kubernetes.io/aws-ebs`: AWS Elastic Block Store (EBS)
- `kubernetes.io/azure-disk`: Azure Disk
- `kubernetes.io/gce-pd`: Google Compute Engine (GCE) Persistent Disk
- `kubernetes.io/cinder`: OpenStack Cinder
- `kubernetes.io/vsphere-volume`: vSphere
- `kubernetes.io/no-provisioner`: Volumes locais
- `kubernetes.io/host-path`: Volumes locais
- `rancher.io/local-path`: Kind
- `kubernetes.io/external-nfs`: NFS server

Ver os StorageClass disponíveis:
``` 
 kubectl get storageclass
```
Mais detalhes:
``` 
 kubectl describe storageclass standard
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

Para criar uma StorageClass simples, usando o *provisioner* de volume local, pode verificar o exemplo do manisfesto *storageClass.yaml*.
