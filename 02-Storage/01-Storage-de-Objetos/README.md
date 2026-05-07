# 02.1 - Storage de objetos (S3) e performance

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todos os comandos `bash` abaixo rodam no **terminal do GitHub Codespaces**. Os passos de console AWS estão sinalizados explicitamente.

> [!WARNING]
> **Pré-requisitos — confira antes de começar:**
>
> - [ ] Codespace aberto e sincronizado com credenciais da AWS Academy (rodou o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md) na aula de hoje).
> - [ ] `aws sts get-caller-identity` retorna um `Account` e um `Arn` sem erro.
> - [ ] `aws s3 ls | grep base-config-` lista exatamente **um** bucket do seu RM.
> - [ ] Pelo menos **15 GB livres** no Codespaces (`df -h /workspaces`) — o lab cria arquivos de 5 GB + 1 GB + 2000 arquivos de 1 MB.
>
> **O que você vai fazer:** subir arquivos de diferentes tamanhos ao S3 medindo o impacto de concorrência, multipart upload e `sync` no tempo de transferência. **Tempo estimado: 50 minutos** (uploads de 5 GB são o gargalo).

Neste laboratório você vai **sentir na mão** por que o S3 reage de forma diferente a arquivos grandes, médios e muito pequenos. A AWS documenta números (3.500 PUTs/s, 5.500 GETs/s por prefixo particionado), mas só quando você cronometra um upload real percebe que o **gargalo quase nunca é o S3** — é o cliente: quantas conexões TCP em paralelo, qual o `multipart_threshold`, qual `chunksize`. O lab te dá a evidência.

## Principais pontos de aprendizagem

- Como `aws configure set default.s3.max_concurrent_requests` muda o tempo de upload de um arquivo grande.
- O que é **multipart upload** e por que ele é automático acima de `multipart_threshold`.
- Diferença prática entre `aws s3 cp`, `aws s3 sync`, `aws s3api copy-object` e GNU `parallel`.
- Por que arquivos pequenos (1 KB) são dominados por overhead de conexão, não banda.
- Como comparar custos de cópia: download + reupload vs. server-side copy.

## O que você terá ao final

Uma tabela mental das operações S3 com medições reais de tempo para três faixas de tamanho (5 GB, 1 GB, 1 KB), e a intuição de quando usar cada ferramenta (`cp`, `sync`, `parallel`, server-side copy).

> [!TIP]
> Os blocos `<details><summary>💡 Clique para entender</summary>` aprofundam o "porquê". Se estiver com pressa, **pule**; se quiser absorver de verdade, **abra**.

## Mapa do lab

| # | Parte | O que acontece | Tempo |
|---|-------|---------------|-------|
| 1 | [Primeiros passos no S3](#parte-1---primeiros-passos-no-s3) | Verificar bucket, baixar CSVs de exemplo, subir ao S3 com diferentes prefixos. | ~5 min |
| 2 | [Performance com arquivos grandes (5 GB)](#parte-2---performance-com-arquivos-grandes-5-gb) | Criar arquivo de 5 GB e comparar tempos com concorrência 1, 2 e 10. | ~15 min |
| 3 | [Performance com arquivos médios (1 GB em paralelo)](#parte-3---performance-com-arquivos-médios-1-gb) | 5 cópias paralelas de arquivo 1 GB via GNU `parallel`. | ~10 min |
| 4 | [Performance com arquivos pequenos (1 MB e 1 KB)](#parte-4---performance-com-arquivos-pequenos) | 2000 arquivos de 1 MB via `sync`, 500 arquivos de 1 KB via `parallel`. | ~10 min |
| 5 | [Comparando opções de cópia](#parte-5---comparando-opções-de-cópia-no-s3) | `cp` (2 etapas) vs. `s3api copy-object` vs. `cp` server-side. | ~5 min |
| 6 | [Limpeza](#parte-6---limpeza) | Apagar objetos do bucket e arquivos locais. | ~5 min |

<details>
<summary><b>💡 Por que a performance do S3 depende tanto do cliente (abra se nunca viu em aula)</b></summary>
<blockquote>

O S3 escala sozinho: ele aceita pelo menos **3.500 PUTs/COPY/POST/DELETE ou 5.500 GETs/HEAD por segundo por prefixo particionado** ([referência oficial](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)). O teto do S3 raramente é o problema em um lab — o problema é que o cliente padrão da AWS CLI abre **uma** conexão TCP e sobe chunks de 8 MB sequencialmente. Se você não ajustar `max_concurrent_requests`, você está subutilizando o S3 em 90% mesmo com banda sobrando.

Para arquivos grandes (> 64 MB por padrão), a CLI liga o **multipart upload**: divide o arquivo em chunks (`multipart_chunksize`), sobe cada chunk como uma operação `UploadPart` independente, e no final chama `CompleteMultipartUpload`. Isso destrava o paralelismo server-side; o paralelismo client-side ainda depende de `max_concurrent_requests`.

Para arquivos minúsculos (< 1 MB) o padrão inverte: cada arquivo vira uma conexão nova, o handshake TCP + TLS domina o tempo, e você não consegue subir nem 100 arquivos/s sem paralelizar no client. É aí que o GNU `parallel` entra — subir 500 arquivos de 1 KB com `-j 1` leva minutos; com `-j 10` leva segundos.

</blockquote>
</details>

## Contexto

A aula introdutória mostrou o S3 como "armazenamento infinito barato". Este lab mostra o outro lado: **o S3 só é rápido se você configurar o cliente certo**. Os três cenários (5 GB, 1 GB, 1 MB, 1 KB) cobrem quase todas as cargas reais. Ao final, você terá dados empíricos para defender escolhas em code review ou em apresentação de arquitetura.

---

## Parte 1 - Primeiros passos no S3

### Resultado esperado desta parte

Bucket listado, variável `$bucket` exportada, 4 objetos CSV no bucket em 4 prefixos diferentes (`car/`, `cereal/`, `factbook/`, `other/`).

1. Verifique que o bucket do setup existe:

```bash
aws s3 ls
```

Saída esperada: pelo menos uma linha com `base-config-<SEU_RM>`.

<!-- PRINT SUGERIDO: img/s3-1.png
     Terminal listando buckets, com base-config-<RM> destacado. -->
![](img/s3-1.png)

2. Exporte o nome do bucket para uma variável reutilizável (todos os próximos passos dependem disso):

```bash
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
```

> [!NOTE]
> Se `echo $bucket` imprimir vazio, **pare** — o setup de credenciais não foi executado. Volte ao [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md).

3. Entre na pasta do lab:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/02-Storage/01-Storage-de-Objetos
```

4. Baixe 3 arquivos CSV de exemplo:

```bash
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/cereal.csv -o cereal.csv
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/cars.csv -o car.csv
curl https://perso.telecom-paristech.fr/eagan/class/igr204/data/factbook.csv -o factbook.csv
```

5. Suba os arquivos para o bucket, cada um em um prefixo diferente:

```bash
aws s3 cp car.csv s3://$bucket/car/car.csv
aws s3 cp cereal.csv s3://$bucket/cereal/cereal.csv
aws s3 cp factbook.csv s3://$bucket/factbook/factbook.csv
aws s3 cp factbook.csv s3://$bucket/other/factbook.tst
```

<details>
<summary><b>💡 Clique para entender — anatomia do <code>aws s3 cp</code></b></summary>
<blockquote>

O comando `aws s3 cp` é a "faca suíça" da CLI do S3. A sintaxe é `aws s3 cp <origem> <destino> [opções]` e funciona em três modos:

| Modo | Exemplo |
|------|---------|
| Local → S3 | `aws s3 cp arquivo.txt s3://bucket/arquivo.txt` |
| S3 → Local | `aws s3 cp s3://bucket/arquivo.txt arquivo.txt` |
| S3 → S3 | `aws s3 cp s3://origem/arq s3://destino/arq` (server-side copy quando mesma região) |

Opções comuns: `--recursive` para pastas, `--exclude "*.jpg"` para filtrar, `--storage-class STANDARD_IA` para classe específica. Acima de `multipart_threshold` (padrão 8 MB), o multipart é automático.

Referência oficial: [AWS CLI S3 high-level commands](https://docs.aws.amazon.com/cli/latest/topic/s3-commands.html).

</blockquote>
</details>

6. Verifique no [console do S3](https://us-east-1.console.aws.amazon.com/s3/buckets?region=us-east-1&bucketType=general) que os 4 objetos apareceram. Clique no seu bucket e navegue pelos prefixos.

<!-- PRINT SUGERIDO: img/s3-2.png
     Console do S3 mostrando os prefixos car/, cereal/, factbook/, other/ dentro do bucket. -->
![](img/s3-2.png)

### Checkpoint

- [x] `aws s3 ls s3://$bucket/` lista os 4 prefixos (`car/`, `cereal/`, `factbook/`, `other/`).
- [x] Console do S3 mostra os mesmos objetos.

---

## Parte 2 - Performance com arquivos grandes (5 GB)

### Resultado esperado desta parte

Três uploads do mesmo arquivo de 5 GB com `max_concurrent_requests` 1, 2 e 10, comparando tempos. Expectativa: concorrência 2 aproximadamente corta o tempo pela metade; concorrência 10 continua caindo, mas com retorno decrescente.

7. Configure o cliente para comportamento previsível e crie pasta de trabalho:

```bash
aws configure set default.s3.max_concurrent_requests 1
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
cd /workspaces/
mkdir -p s3-performance && cd s3-performance
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
```

<details>
<summary><b>💡 Clique para entender — o que cada configuração faz</b></summary>
<blockquote>

| Configuração | Efeito |
|--------------|--------|
| `max_concurrent_requests = 1` | Desliga o paralelismo do client. Base de comparação "worst case". |
| `multipart_threshold = 64MB` | Arquivos > 64 MB ligam multipart automaticamente. Padrão é 8 MB. |
| `multipart_chunksize = 16MB` | Quando multipart liga, cada pedaço tem 16 MB. Padrão é 8 MB. |

Chunk maior = menos overhead por chunk + menos paralelismo efetivo. Chunk menor = mais paralelismo mas mais overhead (hash + HTTP por chunk). 16 MB é meio-termo razoável para banda > 100 Mbps.

Ver [AWS CLI S3 config](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html).

</blockquote>
</details>

8. Crie o arquivo de 5 GB com zeros (rápido — não gera dado aleatório, só aloca):

```bash
dd if=/dev/zero of=5GB.file count=5120 bs=1M
```

9. Upload com concorrência 1 (baseline):

```bash
time aws s3 cp 5GB.file s3://${bucket}/upload1.test
```

Anote o `real`. Exemplo de ordem de grandeza no Codespaces 2-core: ~2-4 min.

10. Upload com concorrência 2:

```bash
aws configure set default.s3.max_concurrent_requests 2
time aws s3 cp 5GB.file s3://${bucket}/upload2.test
```

Expectativa: **aproximadamente metade** do tempo do passo 9.

11. Upload com concorrência 10:

```bash
aws configure set default.s3.max_concurrent_requests 10
time aws s3 cp 5GB.file s3://${bucket}/upload3.test
```

Expectativa: **mais rápido que o passo 10, mas não 5× mais rápido**. Banda do Codespaces e teto de conexões simultâneas viram o gargalo.

> [!TIP]
> Anote os três tempos (`real`) em um papel ou doc. Na Parte 5 você vai referenciá-los quando avaliar a diferença entre `cp` e `copy-object`.

12. Volte à concorrência 1 para os próximos testes:

```bash
aws configure set default.s3.max_concurrent_requests 1
```

### Checkpoint

- [x] 3 objetos no bucket: `upload1.test`, `upload2.test`, `upload3.test`, cada um 5 GB.
- [x] Anotou os 3 tempos do `time`.
- [x] Concorrência 2 deu ~50% do tempo da concorrência 1.

<details>
<summary><b>⚠ Se der erro: <code>An error occurred (EntityTooLarge)</code> ou upload trava em 0%</b></summary>
<blockquote>

- `EntityTooLarge`: o `multipart_threshold` não foi aplicado. Confira com `aws configure get default.s3.multipart_threshold` (deve retornar `64MB`).
- Upload travado: credenciais da AWS Academy expiraram (duram 4 h). Refaça o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md) em outra aba e tente de novo.

</blockquote>
</details>

---

## Parte 3 - Performance com arquivos médios (1 GB)

### Resultado esperado desta parte

5 cópias paralelas de um arquivo de 1 GB subindo em menos de ~30% do tempo de um upload sequencial de 5 GB.

13. Crie o arquivo de 1 GB:

```bash
dd if=/dev/zero of=1GB.file count=1024 bs=1M
```

14. Instale o GNU `parallel` (não vem no Codespaces por padrão):

```bash
sudo apt update -y && sudo apt-get install parallel -y
```

15. Suba 5 cópias paralelas do mesmo arquivo, uma por prefixo distinto:

```bash
time seq 1 5 | parallel --will-cite -j 5 aws s3 cp 1GB.file s3://${bucket}/parallel/object{}.test
```

<details>
<summary><b>💡 Clique para entender — paralelismo em dois níveis</b></summary>
<blockquote>

Este comando paraleliza em **duas camadas**:

1. **GNU parallel (`-j 5`)**: 5 processos `aws s3 cp` independentes, cada um com sua conexão TCP própria.
2. **AWS CLI interno**: cada processo ainda divide seu arquivo em chunks de 16 MB e sobe com `max_concurrent_requests` threads por processo.

Resultado: 5 × 1 GB = 5 GB teoricamente, mas em tempo similar ou inferior a **um único upload de 5 GB**, porque a banda do Codespaces fica saturada com menos overhead fixo por byte. Para > 1000 arquivos pequenos, `aws s3 sync` costuma vencer `parallel` porque faz batching inteligente.

</blockquote>
</details>

16. Limpe o arquivo local de 1 GB:

```bash
rm 1GB.file
```

### Checkpoint

- [x] 5 objetos em `s3://$bucket/parallel/object1.test` até `object5.test`.
- [x] Tempo total comparável ou inferior ao upload sequencial de 5 GB.

---

## Parte 4 - Performance com arquivos pequenos

### Resultado esperado desta parte

2000 arquivos de 1 MB subindo **muito mais rápido** com `sync` concorrente 10 do que com `sync` concorrente 1. 500 arquivos de 1 KB enviados via `parallel -j 5`.

17. Crie 2000 arquivos de 1 MB em uma pasta `sync/`:

```bash
cd /workspaces/s3-performance
mkdir -p sync
seq -w 1 2000 | xargs -n1 -I% sh -c 'dd if=/dev/zero of=sync/file.% bs=1M count=1'
```

> [!NOTE]
> Esse comando cria os 2000 arquivos em sequência e leva 1-2 min. Aguarde terminar antes de seguir.

18. Sincronize para o S3 com concorrência 1 (baseline):

```bash
aws configure set default.s3.max_concurrent_requests 1
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}') && echo $bucket
time aws s3 sync sync/ s3://${bucket}/sync1/
```

Expectativa: **minutos** — o overhead de abrir conexão TCP/TLS domina.

<details>
<summary><b>💡 Clique para entender — por que <code>sync</code> e não <code>cp</code></b></summary>
<blockquote>

`aws s3 sync` compara origem e destino e **só copia o que mudou**. Para 2000 arquivos isso é crítico: se você rodar o sync duas vezes seguidas, a segunda execução é quase instantânea. O `cp --recursive` copia tudo sempre.

Opções úteis: `--delete` (apaga no destino o que não existe na origem), `--exclude` / `--include`, `--storage-class STANDARD_IA` (classe mais barata para dados pouco acessados).

Ver [aws s3 sync docs](https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html).

</blockquote>
</details>

19. Sincronize o **mesmo** diretório para um prefixo diferente com concorrência 10:

```bash
aws configure set default.s3.max_concurrent_requests 10
time aws s3 sync sync/ s3://${bucket}/sync2/
```

Expectativa: **significativamente mais rápido** que o passo 18 (muitas vezes 5-8× mais rápido).

20. Limpe os arquivos locais:

```bash
rm -rf sync
```

21. Teste agora com arquivos **ainda menores** (1 KB), 500 deles:

```bash
seq 1 500 > object_ids
dd if=/dev/zero of=1KB.file count=1 bs=1K
aws configure set default.s3.max_concurrent_requests 1
```

22. Suba os 500 arquivos via `parallel -j 5`:

```bash
time parallel --will-cite -a object_ids -j 5 aws s3 cp 1KB.file s3://${bucket}/run1/{}
```

<details>
<summary><b>💡 Clique para entender — por que <code>parallel</code> e não <code>sync</code> para arquivos minúsculos</b></summary>
<blockquote>

Para arquivos de 1 KB, o tempo é **100% dominado pelo overhead por requisição**: TCP handshake, TLS handshake, assinatura SigV4, parsing de resposta. `aws s3 cp` sequencial consegue uns 5-10 uploads/s. Com `parallel -j 5`, 5 conexões simultâneas dão 25-50 uploads/s.

Para volumes muito grandes (> 10 mil arquivos pequenos) ferramentas como [s5cmd](https://github.com/peak/s5cmd) ou usar o S3 Batch Operations pode ser 10× mais rápido que tudo isso.

</blockquote>
</details>

### Checkpoint

- [x] Prefixos `sync1/`, `sync2/`, `run1/` existem no bucket com a contagem correta de objetos.
- [x] `sync2/` terminou significativamente mais rápido que `sync1/`.

---

## Parte 5 - Comparando opções de cópia no S3

### Resultado esperado desta parte

Três cópias do mesmo objeto de 5 GB usando estratégias diferentes, com tempo medido. Server-side copy esmaga o download + reupload.

23. Configure a concorrência base e execute os três comandos **em sequência**:

```bash
aws configure set default.s3.max_concurrent_requests 1
# 1) cp "2 etapas": download para disco local, depois reupload
time (aws s3 cp s3://$bucket/upload1.test 5GB.file; aws s3 cp 5GB.file s3://$bucket/copy/5GB.file)
# 2) s3api copy-object: server-side, atômico, limite 5 GB
time aws s3api copy-object --copy-source $bucket/upload1.test --bucket $bucket --key copy/5GB-2.file
# 3) s3 cp S3 → S3: server-side com multipart automático
time aws s3 cp s3://$bucket/upload1.test s3://$bucket/copy/5GB-3.file
```

<details>
<summary><b>💡 Clique para entender — por que server-side copy é muito mais barato</b></summary>
<blockquote>

| Método | Onde roda | Tráfego de rede | Custo | Tamanho máx |
|--------|-----------|-----------------|-------|-------------|
| `cp` 2 etapas (download + reupload) | Passa pelo seu cliente | 2 × tamanho do objeto (egress + ingress) | Alto (egress cobrado) | 5 TB (com multipart) |
| `s3api copy-object` | Server-side na AWS | Zero | Zero (mesma região) | 5 GB |
| `s3 cp s3://... s3://...` | Server-side com multipart automático | Zero | Zero (mesma região) | 5 TB |

**Regra de ouro**: se origem e destino são S3 na mesma região, **nunca** baixe para copiar. Use `aws s3 cp s3://... s3://...` para qualquer tamanho (ele liga multipart sozinho) ou `s3api copy-object` para objetos ≤ 5 GB quando você quer a operação atômica.

</blockquote>
</details>

### Checkpoint

- [x] `copy/5GB.file`, `copy/5GB-2.file`, `copy/5GB-3.file` existem no bucket.
- [x] O método 1 (download + reupload) foi **ordens de grandeza mais lento** que os outros dois.

---

## Parte 6 - Limpeza

### Resultado esperado desta parte

Bucket vazio (ou só com os artefatos do setup) e diretório local limpo.

> [!CAUTION]
> Objetos de 5 GB no S3 custam ~$0.12/mês cada. Deixar o lab sem limpar acumula custo silencioso. Não pule este passo.

24. Apague todos os objetos do bucket:

```bash
aws s3 rm s3://${bucket}/ --recursive
```

25. Limpe a pasta local:

```bash
cd /workspaces
rm -rf s3-performance
```

### Checkpoint

- [x] `aws s3 ls s3://$bucket/ --recursive` retorna vazio (ou só arquivos de outros labs que você queira preservar).
- [x] `ls /workspaces/s3-performance` retorna "No such file or directory".

---

## Conclusão

Três lições centrais para levar deste lab:

1. **O gargalo do S3 quase nunca é o S3.** É o cliente — `max_concurrent_requests`, `multipart_threshold`, `multipart_chunksize`. Sem ajustar, você fica em 10-20% do que o S3 permite.
2. **Servidor-side copy é grátis.** Qualquer fluxo que baixa um objeto para reenviar na mesma região está queimando banda e dinheiro à toa. Use `aws s3 cp s3://... s3://...` ou `s3api copy-object`.
3. **Ferramenta certa para cada tamanho:** `cp` para um arquivo grande; `sync` para muitos arquivos médios com idempotência; `parallel` (ou `s5cmd`) para milhares de arquivos pequenos.

## Próximo passo

No [lab 02.2 de EFS](../02-Network-file-system/README.md) você repete o mesmo exercício de performance, mas em um **file system** ao invés de object store. A comparação mental entre S3 e EFS é direta: S3 escala em throughput agregado via múltiplas conexões; EFS escala via múltiplas threads contra um único mount point.

---

<details>
<summary><b>💡 Glossário rápido</b></summary>
<blockquote>

| Termo | O que é |
|-------|---------|
| S3 | Simple Storage Service — object store da AWS, escala por prefixo particionado. |
| Bucket | Container raiz de objetos em uma região. |
| Prefixo | "Pasta virtual" no S3; cada prefixo particionado escala independentemente. |
| Multipart upload | Protocolo do S3 que divide um arquivo em chunks para subida paralela. |
| `multipart_threshold` | Tamanho a partir do qual a CLI liga multipart automaticamente. |
| `multipart_chunksize` | Tamanho de cada pedaço em um multipart upload. |
| `max_concurrent_requests` | Nº de conexões HTTP paralelas que a CLI abre por operação. |
| Server-side copy | Cópia que acontece dentro da AWS, sem tráfego via cliente. |
| GNU parallel | Utilitário Linux que roda N comandos shell em paralelo. |
| `aws s3 sync` | Comando que copia só as diferenças entre origem e destino. |
| `aws s3api copy-object` | API de baixo nível para cópia atômica ≤ 5 GB. |

</blockquote>
</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>
<blockquote>

**Antes de abrir issue ou chamar o professor, colete:**

1. Em qual passo (número) travou.
2. A mensagem de erro **literal** (copie e cole).
3. O que `aws sts get-caller-identity` retorna agora.
4. O que `aws configure list` retorna (sem expor a secret).

**Canais, em ordem:**

1. [Issues deste repositório](https://github.com/rafaelmbarbosa/fiap-arquitetura-compute-e-storage/issues) — preferido, cria histórico pesquisável.
2. Email do professor com os 4 itens acima.
3. Na sala de aula, durante o laboratório.

</blockquote>
</details>
