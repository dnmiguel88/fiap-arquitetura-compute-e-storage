# 04 - Trabalho Final: Performance de transferência EFS → S3

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../01-create-codespaces/Inicio-de-aula.md)**

Este é o **trabalho avaliativo** da disciplina. Diferente dos laboratórios anteriores (que são guiados passo a passo), aqui o enunciado descreve **o que** testar e **o que entregar** — você decide como executar, com base nos comandos já vistos nos labs de [S3](../02-Storage/01-Storage-de-Objetos/README.md) e [EFS](../02-Storage/02-Network-file-system/README.md).

> [!WARNING]
> **Pré-requisitos — confira antes de começar:**
>
> - [ ] Setup inicial concluído ([01 - Setup](../01-create-codespaces/README.md)) e bucket `base-config-<SEU_RM>` existindo.
> - [ ] Credenciais AWS Academy ativas (bolinha verde) — `aws sts get-caller-identity` funciona.
> - [ ] Labs 02.1 (S3) e 02.2 (EFS) executados pelo menos uma vez — você vai reusar os mesmos comandos.
> - [ ] Ferramenta de screenshot no seu OS que você saiba usar: `Cmd+Shift+4` no macOS, `Print Screen`/`Win+Shift+S` no Windows, `gnome-screenshot` no Linux.
>
> **O que você vai fazer:** subir o ambiente (EC2 `c5.large` + EFS + bucket) via Terraform, executar 3 cenários de teste de transferência entre EFS e S3 variando threads e paralelismo, coletar prints de cada resultado e submeter um ZIP no portal da FIAP. **Tempo estimado: 2-3 horas** (grande parte é espera de uploads).

## O que será avaliado

Este trabalho mede sua capacidade de **reproduzir medições de performance controladas** e **interpretar os resultados** à luz do que os labs anteriores ensinaram. A banca vai olhar três dimensões:

1. **Completude** — todos os prints solicitados estão no ZIP.
2. **Rastreabilidade** — cada print mostra claramente **qual cenário** foi executado (tempo, tamanho de arquivo, número de threads visíveis no terminal).
3. **Evidência de boas práticas** — execução feita dentro da pasta do EFS (prints mostram `pwd` em `/efs`), ambiente destruído ao final (print do `terraform destroy` limpo).

> [!TIP]
> **Prepare o caderno antes de começar.** Para cada teste, anote em uma tabela: tamanho do arquivo, threads, tempo real (do `time`). Isso te salva de ter que voltar nos prints durante a análise.

## Mapa do trabalho

| # | Parte | O que acontece | Tempo |
|---|-------|---------------|-------|
| 1 | [Subir o ambiente](#parte-1---subir-o-ambiente) | Terraform cria EC2 `c5.large`, EFS montado em `/efs`, bucket `trabalho-fiap-<RM>`. | ~10 min |
| 2 | [Cenário 1 — arquivo de 5 GB](#cenário-1---arquivo-de-5-gb-no-efs) | Upload EFS → S3 de 5 GB com 1 e 5 threads no cliente. | ~30 min |
| 3 | [Cenário 2 — arquivo de 1 GB com `parallel`](#cenário-2---arquivo-de-1-gb-com-parallel) | 5 execuções paralelas de transferência, com 1 e 15 threads. | ~30 min |
| 4 | [Cenário 3 — 2000 arquivos de 1 MB com `sync`](#cenário-3---2000-arquivos-de-1-mb-com-sync) | `aws s3 sync` da pasta `/efs/sync/` com 1 e 10 threads. | ~30 min |
| 5 | [Evidências adicionais e entrega](#entrega-final) | Print do uso de storage no EFS, listagem do S3, `terraform destroy`, montagem do ZIP. | ~15 min |

---

## Parte 1 - Subir o ambiente

### Resultado esperado desta parte

EC2 `c5.large` rodando, EFS montado em `/efs` dentro dela, bucket `trabalho-fiap-<ACCOUNT_ID>` criado vazio, `parallel` instalado na EC2.

1. No Codespaces, entre na pasta de infraestrutura:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/04-Trabalho-Final/infraestrutura
```

2. Descubra o bucket de estado e aplique o placeholder:

```bash
export bucketState=$(aws s3 ls | awk '/base-config-/ {print $3; exit}')
echo "Bucket de estado: $bucketState"
sed -i "s/base-config-SEU_RM/$bucketState/g" state.tf
```

3. Inicialize e aplique o Terraform:

```bash
terraform init
terraform apply -auto-approve
```

4. Acesse o [console do EC2](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running), localize a instância criada pelo Terraform e conecte via **Session Manager** (mesmo fluxo do [Lab 02.2 de EFS](../02-Storage/02-Network-file-system/README.md#parte-1---provisionar-o-ambiente)).

5. **Dentro da sessão SSM da EC2**, valide que o EFS está montado:

```bash
df -h
```

Saída esperada: linha com ponto de montagem `/efs`.

<details>
<summary><b>⚠ Se der erro: <code>/efs</code> não aparece no <code>df -h</code></b></summary>
<blockquote>

Siga o procedimento do [Lab 02.2, passo 14 troubleshooting](../02-Storage/02-Network-file-system/README.md#parte-1---provisionar-o-ambiente) para instalar `amazon-efs-utils` e montar manualmente.

</blockquote>
</details>

6. Ainda na EC2, descubra o bucket do trabalho e instale o `parallel`:

```bash
export bucket=$(aws s3 ls | awk '/trabalho-fiap-/ {print $3; exit}')
echo "Bucket do trabalho: $bucket"
sudo yum update -y
sudo yum install -y parallel
```

### Checkpoint

- [x] `terraform apply` terminou com `Apply complete!`.
- [x] Sessão SSM aberta na EC2 criada pelo Terraform.
- [x] `/efs` aparece em `df -h`.
- [x] `$bucket` exibe o nome do bucket do trabalho (formato `trabalho-fiap-<ACCOUNT_ID>`).
- [x] `parallel --version` funciona.

---

## Cenário 1 - Arquivo de 5 GB no EFS

### O que testar

Criar um arquivo de **5 GB** dentro do EFS e transferir para o S3 com duas configurações de cliente:

| Config | `max_concurrent_requests` | `multipart_threshold` | `multipart_chunksize` |
|--------|--------------------------|----------------------|----------------------|
| **A** | 1 | 64 MB | 16 MB |
| **B** | 5 | 64 MB | 16 MB |

Para cada uma:

- Execute `aws s3 cp <arquivo-no-efs> s3://$bucket/...` usando `time` para medir.
- **Tire print** do terminal mostrando o comando, o progresso do upload e o resultado do `time`.
- Garanta que o print inclui `pwd` ou o path absoluto do arquivo (`/efs/...`) visível.

### Prints obrigatórios

- [ ] Execução com **1 thread** — comando + `time` + pasta `/efs/`.
- [ ] Execução com **5 threads** — comando + `time` + pasta `/efs/`.

### O que responder (mentalmente — não precisa enviar, mas pode aparecer na banca)

- A diferença de tempo entre 1 e 5 threads foi proporcional? Por quê?
- Onde está o gargalo — disco (EFS), CPU da EC2, banda da rede?

> [!TIP]
> Use `aws configure set default.s3.max_concurrent_requests N` antes de cada execução. Confira com `aws configure list`.

---

## Cenário 2 - Arquivo de 1 GB com `parallel`

### O que testar

Criar um arquivo de **1 GB** dentro do EFS e transferir **5 cópias paralelas** para o S3 usando GNU `parallel`, com duas configurações de cliente:

| Config | `max_concurrent_requests` | Execuções paralelas (`parallel -j`) |
|--------|--------------------------|---------|
| **A** | 1 | 5 |
| **B** | 15 | 5 |

Mesmos pré-requisitos do Cenário 1 para `multipart_threshold` e `multipart_chunksize` (64 MB e 16 MB).

### Prints obrigatórios

- [ ] Execução com **1 thread + `parallel -j 5`** — comando, `time`, saída do parallel.
- [ ] Execução com **15 threads + `parallel -j 5`** — mesmas evidências.

### Observações úteis

- Paralelismo aqui é em **dois níveis**: 5 processos `aws s3 cp` em paralelo, cada um com 1 ou 15 threads internas.
- Compare os tempos com o Cenário 1 — a diferença mostra o impacto de distribuir o trabalho entre múltiplos processos vs. escalar threads de um único processo.

---

## Cenário 3 - 2000 arquivos de 1 MB com `sync`

### O que testar

Criar uma pasta `/efs/sync/` com **2000 arquivos de 1 MB cada** e sincronizar para o S3 com duas configurações:

| Config | `max_concurrent_requests` |
|--------|--------------------------|
| **A** | 1 |
| **B** | 10 |

### Prints obrigatórios

- [ ] Execução `aws s3 sync /efs/sync/ s3://$bucket/...` com **1 thread** — comando + `time`.
- [ ] Execução `aws s3 sync` com **10 threads** — comando + `time`.

### Dica de como gerar os 2000 arquivos

O comando do [Lab 02.1 S3, passo 17](../02-Storage/01-Storage-de-Objetos/README.md#parte-4---performance-com-arquivos-pequenos) funciona dentro do EFS: basta ajustar o `mkdir` e o path para `/efs/sync/`.

> [!NOTE]
> A criação dos 2000 arquivos pode demorar **vários minutos** no EFS (IOPS é o gargalo — exatamente o que o Lab 02.2 mostrou). É esperado.

---

## Entrega final

### Prints adicionais além dos cenários

- [ ] **Uso de storage no EFS** — print do console AWS na página do EFS mostrando o `Size in bytes` ou `Metered size` depois dos testes.
- [ ] **Conteúdo do S3** — print do console AWS no bucket `trabalho-fiap-<RM>` mostrando os objetos criados pelos 3 cenários.
- [ ] **Destruição do ambiente** — print do `terraform destroy -auto-approve` com `Destroy complete!` no final.

### Como montar o ZIP

1. Organize os prints em subpastas claras:

   ```
   entrega-<SEU_RM>.zip
   ├── cenario-1/
   │   ├── 1-thread.png
   │   └── 5-threads.png
   ├── cenario-2/
   │   ├── 1-thread-parallel.png
   │   └── 15-threads-parallel.png
   ├── cenario-3/
   │   ├── 1-thread-sync.png
   │   └── 10-threads-sync.png
   └── evidencias/
       ├── efs-storage.png
       ├── s3-conteudo.png
       └── terraform-destroy.png
   ```

2. Confira que cada print:
   - Tem o comando executado visível.
   - Mostra o `real` do `time` (ou a barra de progresso concluída do `aws s3 cp`).
   - Para os cenários dentro do EFS, mostra que o comando foi executado com `pwd` em `/efs/...` ou com path absoluto.

3. Submeta o ZIP no portal da FIAP seguindo a instrução do professor.

### Critério de aceitação

| Critério | Peso |
|----------|------|
| 6 prints dos cenários (2 por cenário × 3 cenários) | 60% |
| Evidência de execução dentro do EFS (pwd ou path absoluto visível) | 15% |
| Print do uso de storage do EFS e conteúdo do S3 | 15% |
| Print do `terraform destroy` concluído | 10% |

> [!CAUTION]
> **`terraform destroy` é parte da nota.** EC2 `c5.large` + EFS com dados custa aproximadamente $3/dia; deixar ligado após entregar o trabalho queima crédito do Learner Lab e prejudica quem compartilha a conta. Destruir é obrigatório.

---

## Destruir o ambiente

**No Codespaces** (não dentro da sessão SSM, que morre junto com a EC2):

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/04-Trabalho-Final/infraestrutura
terraform destroy -auto-approve
```

Confirme que sumiu:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=trabalho-fiap*" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId"
```

Saída esperada: `[]`.

---

## Dicas para um bom trabalho

- **Execute cada teste uma vez sem cronometrar** para validar que o comando funciona antes de tirar o print oficial. Print com erro é print rejeitado.
- **Os comandos estão prontos** nos labs [02.1 S3](../02-Storage/01-Storage-de-Objetos/README.md) e [02.2 EFS](../02-Storage/02-Network-file-system/README.md) — **reuse**, não reinvente. Mudar os paths para `/efs/...` e o destino para `s3://$bucket/...` do trabalho é suficiente.
- **Não feche o terminal entre testes** — mudar `max_concurrent_requests` no meio altera só a execução seguinte, não as anteriores. Confira com `aws configure list` antes de cada teste.
- **Se uma transferência travar** por mais de 5 minutos sem progresso, a sessão SSM pode ter perdido conexão — reabra e refaça. Credenciais da AWS Academy também podem ter expirado (4 horas).

---

<details>
<summary><b>💡 Glossário rápido</b></summary>
<blockquote>

| Termo | O que é |
|-------|---------|
| EFS | Elastic File System — NFSv4.1 gerenciado da AWS, montável em `/efs`. |
| S3 | Simple Storage Service — object store, destino das transferências deste trabalho. |
| `aws s3 cp` | Comando da CLI para copiar um arquivo para o S3, com multipart automático acima de `multipart_threshold`. |
| `aws s3 sync` | Comando que copia só as diferenças entre origem e destino; ideal para muitos arquivos. |
| GNU parallel | Utilitário Linux que roda N comandos shell concorrentemente. |
| `max_concurrent_requests` | Nº de threads que a CLI abre por operação S3. |
| `multipart_threshold` | Tamanho a partir do qual o multipart upload é automático (default 8 MB). |
| `multipart_chunksize` | Tamanho de cada chunk no multipart (default 8 MB). |
| `time` | Built-in shell que reporta `real`, `user` e `sys` de um comando. |

</blockquote>
</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>
<blockquote>

**Antes de abrir issue ou chamar o professor, colete:**

1. Em qual cenário (1, 2 ou 3) travou.
2. A mensagem de erro **literal** (copie e cole).
3. Em qual ambiente o erro apareceu (Codespaces ou sessão SSM da EC2).
4. O que `aws sts get-caller-identity` retorna agora.

**Canais, em ordem:**

1. [Issues deste repositório](https://github.com/vamperst/fiap-arquitetura-compute-e-storage/issues) — preferido, cria histórico pesquisável.
2. Email do professor com os 4 itens acima.
3. Na sala de aula, durante o laboratório.

</blockquote>
</details>
