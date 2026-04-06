
# Manual para Restaurar Backup no Nexus via Kubernetes

Este manual descreve os passos para restaurar um backup do Nexus utilizando um pod temporário no Kubernetes, incluindo a transferência do arquivo de backup e a instalação do serviço Nexus em um pod.

## 1. Criar Pod Temporário para Buscar Arquivos

Primeiro, criamos um pod temporário no Kubernetes para acessar os volumes onde o backup está armazenado. 

## No cluster de serviços: 192.168.16.60

**Comando para criar o pod:**

```bash
kubectl run nexus-restore --rm -it --restart=Never --image=alpine:3.19 --overrides='
{
  "apiVersion": "v1",
  "kind": "Pod",
  "spec": {
    "containers": [
      {
        "name": "restore",
        "image": "alpine:3.19",
        "stdin": true,
        "tty": true,
        "command": ["sh"],
        "volumeMounts": [
          {
            "name": "nexus-backup",
            "mountPath": "/mnt"
          },
          {
            "name": "nexus-data",
            "mountPath": "/nexus-data"
          }
        ]
      }
    ],
    "volumes": [
      {
        "name": "nexus-backup",
        "persistentVolumeClaim": {
          "claimName": "nexus-backup"
        }
      },
      {
        "name": "nexus-data",
        "persistentVolumeClaim": {
          "claimName": "nexus-data"
        }
      }
    ]
  }
}' -n servicos
```

Este comando cria um pod temporário usando a imagem `alpine:3.19`, monta os volumes do `nexus-backup` e `nexus-data`, e abre um terminal interativo (`sh`) no pod.

## 2. Verificar os Arquivos no Volume de Backup

Após acessar o pod, vá até o diretório `/mnt/nexus-backup` e liste os arquivos de backup.

```bash
cd /mnt/nexus-backup
ls -lh
```

Aqui você pode verificar o tamanho e a data dos arquivos. Identifique o arquivo de backup desejado para transferir para o servidor de teste.

## 3. Transferir o Backup para o Servidor de Teste

## No Cluser de Teste: 192.168.17.175

Use o comando `scp` para copiar o arquivo de backup para o servidor de teste do Nexus:

```bash
scp nexus-data-2026-02-03-0200.tar.gz manager@192.168.17.175:/home/manager/
```

Certifique-se de substituir o caminho e o nome do arquivo conforme necessário.

## 4. Criar um Pod para Restaurar o Backup no Nexus de Teste

Crie um novo pod com o volume `nexus-data` montado para restaurar o backup.

**Comando para criar o pod:**

```bash
kubectl run nexus-restore --rm -it --restart=Never --image=alpine:3.19 --overrides='
{
  "apiVersion": "v1",
  "kind": "Pod",
  "spec": {
    "containers": [
      {
        "name": "restore",
        "image": "alpine:3.19",
        "stdin": true,
        "tty": true,
        "command": ["sh"],
        "volumeMounts": [
          {
            "name": "backup-volume",
            "mountPath": "/mnt"
          },
          {
            "name": "nexus-data",
            "mountPath": "/nexus-data"
          }
        ],
        "resources": {
          "requests": {
            "memory": "4Gi",
            "cpu": "4"
          },
          "limits": {
            "memory": "4Gi",
            "cpu": "4"
          }
        }
      }
    ],
    "volumes": [
      {
        "name": "backup-volume",
        "persistentVolumeClaim": {
          "claimName": "nexus-backup"
        }
      },
      {
        "name": "nexus-data",
        "persistentVolumeClaim": {
          "claimName": "nexus-data"
        }
      }
    ]
  }
}' -n nexus
```

Esse pod vai permitir acessar o volume `nexus-data` e restaurar o backup.

## 5. Instalar o SCP Dentro do Container

Dentro do pod criado, instale o cliente `scp` para transferir o arquivo de backup:

```bash
apk add openssh-client
```

## 6. Transferir o Arquivo de Backup para o Pod

Agora, dentro do pod, execute o comando `scp` para transferir o arquivo de backup do servidor de teste para o pod:

```bash
scp manager@192.168.17.175:/home/manager/nexus-data-2026-02-03-0200.tar.gz .
```

## 7. Desabilitar o Nexus Temporariamente

Antes de restaurar o backup, é necessário parar o serviço Nexus temporariamente. Para isso, escale o número de réplicas para 0:

```bash
kubectl scale deploy nexus --replicas=0 -n nexus
```

## 8. Restaurar o Backup

Agora, vá para o diretório onde o `nexus-data` está montado e extraia o backup:

```bash
cd /nexus-data
tar -xvf nome.do.tar
```

## 9. Reabilitar o Nexus

Depois de restaurar o backup, volte a escalar o serviço Nexus para 1 réplica para que o serviço volte a funcionar:

```bash
kubectl scale deploy nexus --replicas=1 -n nexus
```

---

## **Considerações Finais:**

- Certifique-se de que os volumes `nexus-backup` e `nexus-data` estejam configurados corretamente no seu cluster.
- A transferência de arquivos deve ser feita de maneira segura, utilizando SSH.
- Caso o serviço Nexus precise ser escalado novamente, faça isso com cuidado para garantir que a restauração do backup seja bem-sucedida.
