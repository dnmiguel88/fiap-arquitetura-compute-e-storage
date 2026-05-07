# 02.2 - Network File System (Amazon EFS)

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Os comandos deste lab rodam em **dois ambientes distintos**: o provisionamento (Parte 1) no terminal do **Codespaces**; os testes de performance (Partes 2–4) dentro da **instância EC2** que você vai acessar via SSM. Cada passo sinaliza onde executar.

> [!WARNING]
> **Pré-requisitos — confira antes de começar:**
>
> - [ ] Codespace aberto e sincronizado com credenciais da AWS Academy (rodou o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md) na aula de hoje).
> - [ ] `aws sts get-caller-identity` retorna um `Account` e um `Arn` sem erro.
> - [ ] `aws s3 ls | grep base-config-` lista exatamente **um** bucket do seu RM.
> - [ ] `terraform -version` retorna >= 1.3.
> - [ ] Nenhuma VPC deste lab existe na conta (`aws ec2 describe-vpcs --filters "Name=tag:Name,Values=SID-Network" --query "Vpcs[].VpcId"` retorna `[]`).
>
> **O que você vai fazer:** provisionar VPC + subnets + EC2 + EFS com Terraform em 3 stacks, acessar a EC2 via SSM, rodar 10 cenários de benchmark de IOPS e throughput, e destruir tudo. **Tempo estimado: 60 minutos** (os benchmarks levam 20 min; o resto é provisionamento e setup).

Neste laboratório você coloca o **Amazon Elastic File System (EFS)** sob carga controlada e mede o impacto de três fatores chave: **quantidade de arquivos por operação** (IOPS), **tamanho do bloco de I/O** e **frequência de sincronização**, e **número de threads concorrentes**. EFS é um NFS gerenciado que escala throughput em função do tamanho do sistema de arquivos e do modo de performance — este lab te mostra exatamente como esses parâmetros viram segundos no cronômetro.

## Principais pontos de aprendizagem

- Por que IOPS **escala com paralelismo** (sequencial vs. `parallel -j 128` vs. diretórios separados).
- Diferença prática entre `conv=fsync` (sync no final) e `oflag=sync` (sync a cada bloco).
- Como o **tamanho do bloco** muda throughput: `bs=1M` vs. `bs=16M`.
- Por que EFS se beneficia de múltiplas threads enquanto EBS se beneficia de operações sequenciais.
- Como configurar logging de sessões SSM no CloudWatch (auditoria de acesso SSH).

## O que você terá ao final

Uma matriz mental EFS × carga, evidenciando quando EFS é a escolha certa (workloads paralelos multi-cliente, compartilhamento entre AZs) e quando não é (I/O sequencial de alta latência — aí prefira EBS).

> [!TIP]
> Ao longo do lab você vai encontrar blocos `<details><summary>💡 Clique para entender</summary>`. Eles aprofundam o "porquê". Se estiver com pressa, **pule**.

## Mapa do lab

| # | Parte | O que acontece | Tempo |
|---|-------|---------------|-------|
| 1 | [Provisionar ambiente](#parte-1---provisionar-o-ambiente) | Terraform em 3 stacks (VPC, route tables, EFS+EC2), logging SSM no CloudWatch, entrar na EC2 via SSM. | ~20 min |
| 2 | [Teste de IOPS](#parte-2---teste-de-iops) | 1024 arquivos vazios: sequencial, paralelo, paralelo em múltiplos diretórios. | ~10 min |
| 3 | [Tamanho de I/O e frequência de sync](#parte-3---tamanho-de-io-e-frequência-de-sync) | Arquivo de 2 GB escrito 4 vezes, variando `bs` e modo de sync. | ~10 min |
| 4 | [Multi-threaded I/O](#parte-4---multi-threaded-io) | Mesmo volume (2 GB) escrito com 4 e 16 threads concorrentes. | ~5 min |
| 5 | [Limpeza](#parte-5---limpeza) | `terraform destroy` no stack do EFS+EC2. | ~5 min |

<details>
<summary><b>💡 O que é EFS em 3 parágrafos (abra se nunca viu em aula)</b></summary>
<blockquote>

**Amazon EFS** é um sistema de arquivos de rede (NFSv4.1) totalmente gerenciado. Diferente do EBS (que é um disco anexado a **uma** EC2), um único EFS pode ser montado por **múltiplas EC2s em múltiplas AZs simultaneamente**. Isso o torna a escolha certa quando você precisa de storage compartilhado entre instâncias — pipelines de processamento, servidores web com conteúdo comum, clusters de ML lendo datasets.

O EFS tem dois **modos de performance**: *General Purpose* (latência mínima, teto de 35k IOPS, default) e *Max I/O* (escala além de 35k IOPS à custa de latência maior). E dois **modos de throughput**: *Bursting* (throughput proporcional ao tamanho armazenado, com créditos de burst) e *Provisioned* (você paga fixo por MB/s independente do tamanho). Neste lab usamos o default (General Purpose + Bursting).

O truque central de performance do EFS é que ele foi projetado para **paralelismo distribuído**. Operações sequenciais de uma única thread ficam limitadas pela latência de rede (~1-3 ms por round-trip). Com 10, 100, 1000 operações em paralelo, o throughput agregado cresce quase linearmente — você verá isso saltar aos olhos na Parte 2.

</blockquote>
</details>

## Contexto

A aula introdutória comparou três famílias de storage AWS: **object** (S3), **block** (EBS) e **file** (EFS/FSx). Cada uma tem um perfil de acesso ótimo. Este lab faz o que o [02.1 de S3](../01-Storage-de-Objetos/README.md) fez para object store: medir, com cronômetro na mão, quando o EFS brilha e quando decepciona. Ao final, você terá dados empíricos para decidir em code review ou arquitetura de solução.

O trabalho final ([04-Trabalho-Final](../../04-Trabalho-Final)) retoma exatamente esses comandos para comparar transferência entre EFS e S3 — por isso a notação das variáveis e o ambiente foram pensados para ser reaproveitados.

---

## Parte 1 - Provisionar o ambiente

### Resultado esperado desta parte

VPC `SID-Network` com subnets públicas, instância EC2 `SID-performance-instance` rodando, EFS `SID-efs` com mount point ativo em `/efs` dentro da EC2, CloudWatch log group `/ssm/ssh` gravando sessões.

1. Entre na pasta do primeiro stack Terraform (VPC):

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/rede-vpc/vpc-call
```

2. Descubra o bucket de estado e substitua o placeholder em `state.tf`:

```bash
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}')
echo "Bucket detectado: $bucket"
sed -i "s/base-config-SEU_RM/$bucket/g" state.tf
```

Se `Bucket detectado:` veio vazio, **pare** e revise o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md).

<!-- PRINT SUGERIDO: img/t1.png
     Terminal mostrando o state.tf aberto (ou o sed aplicado) antes do terraform init. -->
![](img/t1.png)

3. Inicialize e aplique o Terraform da VPC:

```bash
terraform init
terraform apply -auto-approve
```

<!-- PRINT SUGERIDO: img/t2.png
     Saída do `terraform apply` da VPC com "Apply complete! Resources: N added". -->
![](img/t2.png)

4. Entre no segundo stack (route tables e subnets), repita o mesmo sed e aplique:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/rede-vpc/RT-call
sed -i "s/base-config-SEU_RM/$bucket/g" state.tf
terraform init
terraform apply -auto-approve
```

<!-- PRINT SUGERIDO: img/t3.png
     Saída do `terraform apply` das route tables. -->
![](img/t3.png)

5. Entre no terceiro stack (EC2 + EFS) e aplique:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/efs-instance
sed -i "s/base-config-SEU_RM/$bucket/g" state.tf
terraform init
terraform apply -auto-approve
```

<!-- PRINT SUGERIDO: img/t4.png
     Saída do `terraform apply` do stack EFS+EC2 com todos os recursos criados. -->
![](img/t4.png)

<details>
<summary><b>💡 Por que 3 stacks Terraform separados?</b></summary>
<blockquote>

Dividir em VPC → route tables → EC2+EFS é didático: cada stack mostra uma camada AWS (rede core, roteamento, compute+storage). Em produção, esse padrão também faz sentido quando **camadas têm ciclos de vida diferentes**: a VPC raramente muda, enquanto as EC2s podem subir/descer várias vezes por dia. Stacks separados deixam `terraform plan` mais rápido e blast radius menor.

O estado de cada stack é guardado no mesmo bucket S3 (`base-config-<RM>`) com chaves diferentes, evitando colisão.

</blockquote>
</details>

<details>
<summary><b>⚠ Se der erro: <code>Error: error creating VPC: InvalidParameterValue</code></b></summary>
<blockquote>
Provavelmente a VPC `SID-Network` já existe de uma execução anterior. Rode `terraform destroy -auto-approve` nos 3 stacks (ordem inversa: efs-instance → RT-call → vpc-call) e recomece do passo 1.
</blockquote>
</details>

6. Crie o log group do CloudWatch que vai receber os comandos da sessão SSM (auditoria):

   Acesse o [console do CloudWatch Logs](https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups) e clique em `Criar grupo de logs`.

<!-- PRINT SUGERIDO: img/t5.png
     Página "Criar grupo de logs" do CloudWatch vazia, antes de preencher. -->
![](img/t5.png)

7. Preencha e salve:

   - **Nome do grupo de logs**: `/ssm/ssh`
   - **Configuração de retenção**: `3 Dias`

<!-- PRINT SUGERIDO: img/t6.png
     Formulário preenchido com /ssm/ssh e 3 dias de retenção. -->
![](img/t6.png)

8. Agora configure o SSM para enviar as sessões a esse log group. Acesse o [Session Manager → Preferences](https://us-east-1.console.aws.amazon.com/systems-manager/session-manager/preferences?region=us-east-1) e clique em `Editar`.

<!-- PRINT SUGERIDO: img/t7.png
     Botão "Editar" das preferências do Session Manager. -->
![](img/t7.png)

9. Na seção `CloudWatch Logging`, preencha:

   - **CloudWatch logging**: `Marcado`
   - **Enforce encryption**: `Desmarcado`
   - **Grupo de logs do CloudWatch**: `Selecionar um grupo da lista`
   - **Grupo de logs**: `/ssm/ssh`

<!-- PRINT SUGERIDO: img/t8.png
     Seção CloudWatch Logging preenchida com /ssm/ssh selecionado. -->
![](img/t8.png)

10. Na seção `Linux shell profile` cole o texto abaixo e clique em `Salvar` no final da página:

```bash
bash
sudo su -
```

11. Abra o [console do EC2](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running) e localize a instância `SID-performance-instance`.

<!-- PRINT SUGERIDO: img/t9.png
     Lista de instâncias EC2 mostrando SID-performance-instance em estado running. -->
![](img/t9.png)

12. Selecione a instância e clique em `Conectar` no topo.

<!-- PRINT SUGERIDO: img/t10.png
     Painel de conectar da instância EC2. -->
![](img/t10.png)

13. Na aba `Gerenciador de sessões`, clique em `Conectar`. Uma nova aba vai abrir com o terminal da instância.

<!-- PRINT SUGERIDO: img/t11.png
     Aba "Gerenciador de sessões" selecionada, botão Conectar à vista. -->
![](img/t11.png)

<!-- PRINT SUGERIDO: img/t12.png
     Terminal do SSM já aberto dentro da EC2 como root. -->
![](img/t12.png)

14. **Dentro da sessão SSM da EC2**, configure o AWS CLI e valide que o EFS está montado:

```bash
mkdir -p ~/.aws
aws configure set region us-east-1
aws configure set output json
df -h
```

<!-- PRINT SUGERIDO: img/t14.png
     Saída do df -h mostrando /efs montado. -->
![](img/t14.png)

> [!NOTE]
> Se `df -h` **não** mostra `/efs`, o mount não subiu automaticamente. Use o bloco abaixo para montar manualmente antes de continuar.

<details>
<summary><b>⚠ Se der erro: <code>/efs</code> não aparece no <code>df -h</code></b></summary>
<blockquote>

Instale o client `amazon-efs-utils` e faça o mount manualmente:

```bash
sudo yum install -y amazon-efs-utils

FS_ID=$(aws efs describe-file-systems \
  --query "FileSystems[?Name=='SID-efs'].FileSystemId" \
  --output text)

sudo mkdir -p /efs
sudo mount -t efs ${FS_ID}:/ /efs
mount | grep /efs
df -h
```

Saída esperada: linha com `127.0.0.1:/` (o helper do EFS usa um stunnel local) e ponto de montagem `/efs`.

</blockquote>
</details>

### Checkpoint

- [x] 3 `terraform apply` concluídos sem erro.
- [x] Log group `/ssm/ssh` criado e Session Manager configurado para gravar nele.
- [x] Sessão SSM aberta na `SID-performance-instance`.
- [x] `/efs` aparece em `df -h` dentro da EC2.

---

## Parte 2 - Teste de IOPS

### Resultado esperado desta parte

Três execuções criando 1024 arquivos vazios no EFS — sequencial, paralela em um diretório, paralela em 32 diretórios — com o tempo da execução paralela **significativamente menor** que a sequencial (tipicamente 10-50×).

> Todos os comandos desta parte e das próximas rodam **dentro da sessão SSM** da EC2, não no Codespaces.

IOPS (*Input/Output Operations Per Second*) é a métrica que governa workloads com **muitos arquivos pequenos**: checkpoints de ML, fila de mensagens em disco, sistemas de cache. O EFS escala IOPS essencialmente pela **quantidade de conexões concorrentes** contra o file system.

15. Teste sequencial — 1024 arquivos vazios, um loop `for` serial:

```bash
directory=$(echo $(uuidgen)| grep -o ".\\{6\\}$")
mkdir -p /efs/tutorial/touch/${directory}
time for i in {1..1024}; do
  touch /efs/tutorial/touch/${directory}/test-1.3-$i;
done;
```

<!-- PRINT SUGERIDO: img/t16.png
     Saída do time mostrando o real do for sequencial (ordem de minutos). -->
![](img/t16.png)

<details>
<summary><b>💡 Clique para entender — por que esse teste é lento</b></summary>
<blockquote>

Cada `touch` vira uma chamada NFS separada contra o EFS, com round-trip completo: cliente → endpoint NFS → backend EFS → resposta. Com latência típica de 1-3 ms por operação, 1024 operações sequenciais = 1-3 segundos **no melhor caso**. Na prática o overhead do shell, do kernel e do contexto Linux inflaciona para ordem de minutos.

O loop `for ... do ... done` é sequencial por design: o próximo `touch` só começa quando o anterior volta. A capacidade do EFS está ociosa durante o round-trip.

</blockquote>
</details>

16. Teste paralelo — mesmos 1024 arquivos, mas com GNU `parallel -j 128`:

```bash
directory=$(echo $(uuidgen)| grep -o ".\\{6\\}$")
mkdir -p /efs/tutorial/touch/${directory}
time seq 1 1024 | parallel --will-cite -j 128 touch /efs/tutorial/touch/${directory}/test-1.4-{}
```

<!-- PRINT SUGERIDO: img/t17.png
     Saída do time mostrando real drasticamente menor que o teste sequencial. -->
![](img/t17.png)

Expectativa: **muito mais rápido** que o passo 15 (comum ver 10-50× de diferença).

17. Teste paralelo distribuído — 1024 arquivos, mas em **32 subdiretórios**:

```bash
directory=$(echo $(uuidgen)| grep -o ".\\{6\\}$")
mkdir -p /efs/tutorial/touch/${directory}/{1..32}
time seq 1 32 | parallel --will-cite -j 32 touch /efs/tutorial/touch/${directory}/{}/test1.5{1..32}
```

<!-- PRINT SUGERIDO: img/t18.png
     Saída do time do teste em diretórios separados. -->
![](img/t18.png)

<details>
<summary><b>💡 Clique para entender — distribuição em múltiplos diretórios</b></summary>
<blockquote>

Sistemas de arquivos tradicionais pagam custo quando muitos arquivos estão no mesmo diretório (bloqueios no inode do dir, locks de metadados). No EFS isso é atenuado mas não zerado. Distribuir arquivos em múltiplos diretórios **reduz contenção de metadata** e tende a escalar melhor.

Em produção, sistemas como Hadoop/Spark ou storages de training de ML já seguem esse padrão por default: hash dos arquivos em prefixos.

</blockquote>
</details>

### Checkpoint

- [x] Três tempos anotados (sequencial, paralelo, paralelo distribuído).
- [x] A execução paralela foi **ordens de grandeza** mais rápida que a sequencial.

---

## Parte 3 - Tamanho de I/O e frequência de sync

### Resultado esperado desta parte

Quatro escritas de 2 GB no EFS comparando **block size** (1 MB vs 16 MB) e **modo de sync** (`conv=fsync` vs `oflag=sync`). Expectativa: bloco maior + sync no final = muito mais rápido.

18. Crie a estrutura de diretórios para os testes:

```bash
sudo mkdir -p /efs/tutorial/{dd,touch,rsync,cp,parallelcp,parallelcpio}/
```

19. **Teste A** — 2 GB, bloco de 1 MB, sync no final (`conv=fsync` = fsync uma vez ao final do dd):

```bash
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) \
bs=1M count=2048 status=progress conv=fsync
```

<!-- PRINT SUGERIDO: img/t19.png
     Saída do dd com bs=1M count=2048 conv=fsync, taxa em MB/s. -->
![](img/t19.png)

20. **Teste B** — 2 GB, bloco de **16 MB**, sync no final:

```bash
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) \
bs=16M count=128 status=progress conv=fsync
```

<!-- PRINT SUGERIDO: img/t20.png
     Saída do dd com bs=16M, taxa tipicamente maior que o teste A. -->
![](img/t20.png)

Expectativa: **mais rápido que o teste A** — menos chamadas `write()` e menos overhead por bloco.

21. **Teste C** — 2 GB, bloco de 1 MB, **sync a cada bloco** (`oflag=sync`):

```bash
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) \
bs=1M count=2048 status=progress oflag=sync
```

<!-- PRINT SUGERIDO: img/t21.png
     Saída do dd com oflag=sync, taxa drasticamente menor que os testes A e B. -->
![](img/t21.png)

Expectativa: **muito mais lento** — cada bloco de 1 MB força round-trip completo até o storage antes do próximo `write`.

22. **Teste D** — 2 GB, bloco de 16 MB, sync a cada bloco:

```bash
time dd if=/dev/zero of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) \
bs=16M count=128 status=progress oflag=sync
```

<!-- PRINT SUGERIDO: img/t22.png
     Saída do dd com bs=16M oflag=sync. -->
![](img/t22.png)

Expectativa: mais lento que o B (sync caro), mas **muito mais rápido que o C** (poucos blocos grandes amortizam o custo do sync).

<details>
<summary><b>💡 Clique para entender — <code>conv=fsync</code> vs <code>oflag=sync</code></b></summary>
<blockquote>

| Opção | Quando o sync acontece | Impacto |
|-------|------------------------|---------|
| `conv=fsync` | **Uma única vez**, ao final de todo o dd | Throughput próximo do máximo, garante durabilidade só no final |
| `oflag=sync` | **A cada `write()`** (cada bloco) | Throughput despenca, garante durabilidade em todo ponto intermediário |

Na prática: use `conv=fsync` para **backups e grandes dumps** onde você só se importa com "chegou ou não". Use `oflag=sync` em **logs de banco de dados** onde perder o último segundo de escritas é inaceitável.

EBS vs EFS: EBS tolera melhor `oflag=sync` porque é um disco local anexado; EFS paga round-trip de rede em cada sync, por isso a diferença entre B e C neste lab é tão marcante.

</blockquote>
</details>

### Checkpoint

- [x] 4 tempos anotados (A/B/C/D).
- [x] Ordem esperada do mais rápido ao mais lento: B < A < D < C.

---

## Parte 4 - Multi-threaded I/O

### Resultado esperado desta parte

Comparar 2 GB escritos em 4 threads (512 MB cada) vs. 16 threads (128 MB cada). Expectativa: 16 threads saturam mais banda que 4 threads, mas com retorno decrescente por causa do overhead de abrir 16 conexões NFS paralelas.

O EFS foi projetado para **paralelismo distribuído** entre clientes/threads. Workloads mono-threaded batem rápido no teto. Este teste mostra o ganho de abrir múltiplos `dd` em paralelo.

23. **Teste E** — 2 GB total em 4 threads (4 × 512 MB):

```bash
time seq 0 3 | parallel --will-cite -j 4 dd if=/dev/zero \
of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{} bs=1M count=512 oflag=sync
```

<!-- PRINT SUGERIDO: img/t23.png
     Saída do time com 4 threads paralelas de dd. -->
![](img/t23.png)

24. **Teste F** — 2 GB total em 16 threads (16 × 128 MB):

```bash
time seq 0 15 | parallel --will-cite -j 16 dd if=/dev/zero \
of=/efs/tutorial/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{} bs=1M count=128 oflag=sync
```

<!-- PRINT SUGERIDO: img/t24.png
     Saída do time com 16 threads paralelas de dd. -->
![](img/t24.png)

<details>
<summary><b>💡 Por que 16 threads nem sempre é 4× mais rápido que 4 threads</b></summary>
<blockquote>

O ganho é sublinear por três razões: (1) o cliente EC2 tem limites de conexões NFS concorrentes; (2) o EFS em modo *Bursting* dá throughput proporcional ao tamanho do FS — se o FS é pequeno (caso deste lab), o teto é baixo; (3) o overhead de `parallel` (fork + setup) cresce com `-j`. Em produção, a lógica é medir e escolher o ponto de retorno decrescente.

Se o lab estiver em modo *Max I/O* e o FS tiver muito dado armazenado, 16 threads tende a ficar próximo de 4× mais rápido que 4 threads.

</blockquote>
</details>

### Checkpoint

- [x] Dois tempos anotados (E e F).
- [x] 16 threads foi mais rápido que 4 threads, mas **não** necessariamente 4× mais rápido.

---

## Parte 5 - Limpeza

### Resultado esperado desta parte

EC2, EFS, subnets e VPC destruídos. Bucket de estado preservado para próximos labs.

> [!CAUTION]
> EC2 `c5.large` custa ~$0.085/hora (~$2/dia se esquecer ligada); EFS armazena os arquivos criados no lab e cobra ~$0.30/GB/mês. Sem destruir, você acumula custo silencioso.

25. **No Codespaces**, destrua os stacks em ordem inversa (EFS+EC2 → route tables → VPC):

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/efs-instance
terraform destroy -auto-approve
```

<!-- PRINT SUGERIDO: img/t25.png
     Saída do terraform destroy do stack EFS+EC2 com "Destroy complete!". -->
![](img/t25.png)

26. (Opcional, mas recomendado) Destrua também os stacks de rede:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/rede-vpc/RT-call
terraform destroy -auto-approve

cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/02-Network-file-system/rede-vpc/vpc-call
terraform destroy -auto-approve
```

> [!NOTE]
> O [trabalho final](../../04-Trabalho-Final) reutiliza a mesma VPC. Se você vai fazer o trabalho final em seguida, **pule o passo 26** e deixe a VPC e route tables vivas — mas ainda assim destrua o stack do EFS+EC2 (passo 25).

### Checkpoint

- [x] `aws efs describe-file-systems --query "FileSystems[?Name=='SID-efs']"` retorna `[]`.
- [x] `aws ec2 describe-instances --filters "Name=tag:Name,Values=SID-performance-instance" --query "Reservations[].Instances[?State.Name=='running']"` retorna `[]`.

---

## Conclusão

Três lições centrais do lab:

1. **IOPS no EFS = paralelismo.** Operações sequenciais deixam 90% da capacidade ociosa. `parallel -j 128` contra `for` serial muda a ordem de grandeza.
2. **Sync frequente e bloco pequeno é o pior dos mundos.** `oflag=sync` com `bs=1M` é tipicamente 10× mais lento que `conv=fsync` com `bs=16M`. Só vale quando durabilidade intermediária é requisito real.
3. **Threads paralelas têm retorno decrescente.** Dobrar de 4 para 16 nem sempre é 4× mais rápido. Medir é melhor que assumir.

## Próximo passo

No [trabalho final](../../04-Trabalho-Final) você combina esses comandos com os do lab de S3 para medir **transferência EFS → S3** sob diferentes configurações. Se você não destruiu a VPC no passo 26, está pronto para começar; se destruiu, basta rodar o Terraform do trabalho final que sobe uma VPC nova.

---

<details>
<summary><b>💡 Glossário rápido</b></summary>
<blockquote>

| Termo | O que é |
|-------|---------|
| EFS | Elastic File System — NFSv4.1 gerenciado, multi-AZ, multi-cliente. |
| NFS | Network File System — protocolo padrão Unix para compartilhamento de arquivos via rede. |
| IOPS | I/O Operations Per Second — métrica para cargas com muitos arquivos pequenos. |
| General Purpose | Modo de performance default do EFS, latência baixa, teto de ~35k IOPS. |
| Max I/O | Modo de performance que escala além de 35k IOPS, com latência maior. |
| Bursting | Modo de throughput default, proporcional ao tamanho armazenado, com créditos. |
| `dd` | Utilitário Unix para I/O byte a byte, usado como gerador de carga controlada. |
| `conv=fsync` | Flag do dd que força sync **uma vez** no final da execução. |
| `oflag=sync` | Flag do dd que força sync **a cada bloco** escrito. |
| `bs` | Block size — tamanho de cada chunk em cada chamada `write()` do dd. |
| SSM Session Manager | Serviço AWS que dá shell em EC2 sem expor porta 22, com auditoria no CloudWatch. |
| GNU parallel | Utilitário que roda N comandos shell concorrentemente. |

</blockquote>
</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>
<blockquote>

**Antes de abrir issue ou chamar o professor, colete:**

1. Em qual passo (número) travou.
2. A mensagem de erro **literal** (copie e cole).
3. O que `aws sts get-caller-identity` retorna agora.
4. Em qual ambiente o erro apareceu (Codespaces ou sessão SSM da EC2).

**Canais, em ordem:**

1. [Issues deste repositório](https://github.com/rafaelmbarbosa/fiap-arquitetura-compute-e-storage/issues) — preferido, cria histórico pesquisável.
2. Email do professor com os 4 itens acima.
3. Na sala de aula, durante o laboratório.

</blockquote>
</details>
