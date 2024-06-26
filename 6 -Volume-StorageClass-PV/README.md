# Volumes

Sempre que um Pod é criado é gerado algum tipo de dado, quando o Pod é morto os dados são perdidos.
Uma maneira para preservar esses dados é através de volumes.
Tipos de Volumes:
 - `Ephemeral volumes`: são volumes criados e destruídos junto com o Pod, por exemplo o `EmptyDir`.
 - `Persistent volumes`: são volumes criados e não são destruídos junto com o Pod. Exemplo `StorageClass` que provisiona os PVs (PersistentVolumes) e PVCs (PersistentVolumeClaims)
 