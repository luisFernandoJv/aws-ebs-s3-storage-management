# aws-ebs-s3-storage-management

> Laboratório prático de gerenciamento de armazenamento na AWS: snapshots automatizados de EBS com política de retenção via Python e sincronização inteligente com Amazon S3, incluindo versionamento e recuperação de arquivos.

---

## Visão geral

Este projeto documenta a implementação de uma estratégia completa de **proteção e recuperação de dados na AWS**, executada 100% via **AWS CLI** a partir de uma instância EC2 (Command Host).

O ambiente simula um cenário real de produção: um servidor ativo (Processor) com volume EBS crítico, onde é necessário garantir backups automáticos, sincronização com armazenamento durável e capacidade de restauração sem perda de dados.

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS VPC — us-west-2                      │
│                      Sub-rede Pública                           │
│                                                                 │
│   ┌──────────────────┐          ┌──────────────────────────┐   │
│   │   Command Host   │  AWS CLI │       Processor          │   │
│   │   (t3.medium)    │─────────▶│      (t3.micro)          │   │
│   │                  │          │                          │   │
│   └──────────────────┘          │  ┌────────────────────┐  │   │
│                                 │  │   Volume EBS (8GB) │  │   │
│                                 │  └─────────┬──────────┘  │   │
│                                 └────────────│─────────────┘   │
│                                              │                  │
│                    ┌─────────────────────────▼──────────┐      │
│                    │         Snapshots EBS               │      │
│                    │   (política: 2 mais recentes)       │      │
│                    └────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ aws s3 sync
                              ▼
              ┌───────────────────────────────┐
              │         Amazon S3             │
              │  s3://bucket/files/           │
              │  ├── file1.txt  (29.6 KB)     │
              │  ├── file2.txt  (42.8 KB)     │
              │  └── file3.txt  (94.4 KB)     │
              │                               │
              │  Versionamento: ATIVADO       │
              └───────────────────────────────┘
```

---

## O que foi implementado

### Parte 1 — Snapshots automatizados do EBS

| Etapa                | Comando                                       | Resultado                   |
| -------------------- | --------------------------------------------- | --------------------------- |
| Obter volume ID      | `describe-instances`                          | `vol-0aa2b0e8654ed5525`     |
| Parar instância      | `stop-instances` + `wait instance-stopped`    | Parada controlada           |
| Criar snapshot       | `create-snapshot` + `wait snapshot-completed` | `snap-09d794ea7b4203ccf`    |
| Automatizar          | `crontab` com job a cada minuto               | Múltiplos snapshots gerados |
| Política de retenção | `python3 snapshotter_v2.py`                   | Apenas 2 snapshots mantidos |

### Parte 2 — Sincronização EBS → S3 com versionamento

| Etapa                   | Comando                                   | Resultado                  |
| ----------------------- | ----------------------------------------- | -------------------------- |
| Habilitar versionamento | `s3api put-bucket-versioning`             | Histórico de versões ativo |
| Sincronizar arquivos    | `aws s3 sync files s3://bucket/files/`    | 3 arquivos enviados        |
| Simular exclusão        | `rm files/file1.txt` + `s3 sync --delete` | Arquivo removido do S3     |
| Listar versões          | `s3api list-object-versions`              | Version ID localizado      |
| Restaurar arquivo       | `s3api get-object --version-id`           | `file1.txt` recuperado     |
| Re-sincronizar          | `aws s3 sync`                             | Arquivo restaurado no S3   |

---

## Tecnologias utilizadas

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20EBS%20%7C%20S3%20%7C%20IAM-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.x-3776AB?style=flat&logo=python&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-cron%20%7C%20bash-FCC624?style=flat&logo=linux&logoColor=black)
![CLI](https://img.shields.io/badge/AWS%20CLI-v2-232F3E?style=flat&logo=amazonaws&logoColor=white)

---

## Evidências de execução

### 1. Parando a instância Processor e obtendo IDs

![Stop instance e obtenção de IDs](assets/01-stop-instance-get-ids.png)

Instância `i-021978cf2e0f48ebf` interrompida de forma controlada antes da criação do snapshot.

---

### 2. Snapshot criado — estado pending

![Snapshot pending](assets/02-create-snapshot-pending.png)

Snapshot `snap-09d794ea7b4203ccf` iniciado para o volume `vol-0aa2b0e8654ed5525`.

---

### 3. Snapshot concluído + cron job rodando

![Snapshot completed e cron job](assets/03-snapshot-completed-cronjob.png)

Snapshot com `"State": "completed"` e `"Progress": "100%"`. Cron job configurado para automação.

---

### 4. Instâncias EC2 em execução

![EC2 instances](assets/04-ec2-instances-running.png)

Command Host (`t3.medium`) e Processor (`t3.micro`) ambos com status `Executando` e verificações aprovadas.

---

### 5. Bucket S3 criado

![S3 bucket](assets/05-s3-bucket-created.png)

Bucket `s3-bucket-name-114197082420-us-west-2-an` criado na região `us-west-2` com versionamento ativado.

---

### 6. Arquivos sincronizados no S3

![S3 files synced](assets/06-s3-files-synced.png)

3 arquivos sincronizados com sucesso: `file1.txt` (29.6 KB), `file2.txt` (42.8 KB), `file3.txt` (94.4 KB).

---

### 7. file1.txt restaurado via versionamento

![S3 versioning restore](assets/07-s3-file1-restored-versioning.png)

Arquivo `file1.txt` recuperado após exclusão acidental usando `s3api get-object --version-id`. Zero perda de dados.

---

## Principais aprendizados

- **Snapshots sem política de retenção viram custo** — o script Python que mantém apenas os 2 mais recentes é essencial em produção
- **Versionamento no S3 é a diferença entre perda e recuperação** — um `--delete` sem versionamento ativo seria irreversível
- **Tudo via CLI garante reprodutibilidade** — cada etapa pode ser integrada a pipelines, scripts de IaC ou orquestração com Airflow
- **Parar a instância antes do snapshot** garante consistência dos dados no volume

---

## Contexto

Laboratório realizado como parte da formação prática em **AWS e Engenharia de Dados**, com foco em construir infraestrutura resiliente para pipelines de dados em produção.

Combina diretamente com o stack que uso em projetos reais: **Python · Airflow · PostgreSQL · Docker · AWS**.

---

## Autor

**Luis Fernando Alexandre dos Santos**  
Engenheiro de Dados | Mestrando em IA (UFERSA)  
[LinkedIn](https://www.linkedin.com/in/luisfernando-eng/) · [GitHub](https://github.com/luisFernandoJv)
