# 03.1 - Compute: x86 vs Graviton

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Os comandos deste lab rodam em **três ambientes distintos**: o provisionamento (Parte 1) no terminal do **Codespaces**; os benchmarks (Partes 2–6) dentro de **duas sessões SSM simultâneas** — uma na instância x86 e outra na Graviton. Cada passo sinaliza onde executar.

> [!WARNING]
> **Pré-requisitos — confira antes de começar:**
>
> - [ ] Codespace aberto e sincronizado com credenciais da AWS Academy (rodou o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md) na aula de hoje).
> - [ ] `aws sts get-caller-identity` retorna um `Account` e um `Arn` sem erro.
> - [ ] `aws s3 ls | grep base-config-` lista exatamente **um** bucket do seu RM.
> - [ ] `terraform -version` retorna >= 1.3.
> - [ ] Nenhuma EC2 deste lab já existe (`aws ec2 describe-instances --filters "Name=tag:Name,Values=x86,graviton" --query "Reservations[].Instances[?State.Name=='running'].InstanceId"` retorna `[]`).
>
> **O que você vai fazer:** provisionar duas EC2s idênticas (mesmo vCPU e RAM) mas com arquiteturas diferentes — Intel x86_64 (`t3.large`) e AWS Graviton ARM64 (`t4g.large`) — e rodar 5 benchmarks lado a lado. **Tempo estimado: 50 minutos.**

Este laboratório responde uma pergunta concreta de arquitetura: **vale a pena migrar workloads para Graviton?** A resposta técnica depende do tipo de workload — e este lab te dá os dados para defender sua posição em code review ou apresentação. Você vai cronometrar CPU single-thread, CPU multi-thread, banda de memória, compressão, Python recursivo e SHA-256 em Node.js. Ao final, os números falam.

## Principais pontos de aprendizagem

- Diferença entre **x86_64** (Intel/AMD, CISC) e **ARM64** (Graviton, RISC) no mundo AWS.
- Quando Graviton **ganha** (CPU-bound com boa otimização ARM) e quando **empata ou perde** (memória sequencial, runtime menos otimizado para ARM).
- Por que Graviton ainda é **mais barato** mesmo quando perde em benchmark individual — ratio custo/performance.
- Como usar `sysbench` como ferramenta de benchmark portável para avaliar instâncias.
- Como acessar EC2s via **SSM** sem expor porta 22 (auditoria e segurança).

## O que você terá ao final

Uma matriz de resultados empíricos (CPU, memória, compressão, Python, Node.js) comparando x86 e Graviton em instâncias equivalentes, e a intuição de qual arquitetura escolher para cada classe de workload.

> [!TIP]
> Os blocos `<details><summary>💡 Clique para entender</summary>` aprofundam cada benchmark. Se estiver com pressa, **pule**.

## Recursos úteis

- [Documentação oficial EC2 Graviton](https://docs.aws.amazon.com/ec2/latest/userguide/graviton.html)
- [Por que Graviton é mais barato — página de produto AWS](https://aws.amazon.com/ec2/graviton/)
- [Sysbench no GitHub](https://github.com/akopytov/sysbench)
- [Comparação de instâncias EC2 (preço/performance)](https://instances.vantage.sh/)

## Mapa do lab

| # | Parte | O que acontece | Tempo |
|---|-------|---------------|-------|
| 1 | [Provisionar as duas EC2s](#parte-1---provisionar-as-duas-ec2s) | Terraform sobe `t3.large` (x86) e `t4g.large` (Graviton) com Ubuntu 22.04. Acessar ambas via SSM. | ~10 min |
| 2 | [Benchmark de CPU single-thread](#parte-2---benchmark-de-cpu-single-thread) | `sysbench cpu` com 1 thread, cálculo de primos. Esperado: Graviton ~3× mais rápido. | ~5 min |
| 3 | [Benchmark de CPU multi-thread](#parte-3---benchmark-de-cpu-multi-thread) | Mesmo teste com 2 threads (2 vCPUs). Confirma escala. | ~5 min |
| 4 | [Benchmark de memória](#parte-4---benchmark-de-memória) | Banda de escrita em RAM. Esperado: x86 ~10% à frente. | ~5 min |
| 5 | [Benchmark de compressão](#parte-5---benchmark-de-compressão-gzip) | `gzip` em 1 GB aleatório. Esperado: Graviton ~17% mais rápido. | ~10 min |
| 6 | [Workloads reais: Python e Node.js](#parte-6---workloads-reais-python-e-nodejs) | Fibonacci recursivo em Python e SHA-256 em Node.js. | ~10 min |
| 7 | [Limpeza](#parte-7---limpeza) | `terraform destroy` para zerar o custo. | ~5 min |

<details>
<summary><b>💡 x86 vs Graviton em 3 parágrafos (abra se nunca viu em aula)</b></summary>
<blockquote>

**x86_64** é a arquitetura dominante em servidores há décadas. CPUs da Intel (Xeon, Ice Lake) e AMD (EPYC) usam um conjunto de instruções **CISC** — rico, complexo, retrocompatível desde os anos 80. Muito software foi compilado e otimizado especificamente para x86, o que é uma vantagem herdada, não técnica.

**AWS Graviton** é a linha de processadores ARM64 (RISC) desenhada pela própria AWS. Graviton 3 (em `c7g`/`m7g`/`r7g`/`t4g`) entrega, segundo a AWS, **até 40% melhor relação preço/performance** vs. instâncias x86 equivalentes. A fonte do ganho é tripla: ISA mais simples (menos transistores → mais cores por die), design sem precisar suportar retrocompatibilidade com instruções legadas, e integração vertical com a nuvem (AWS desenha para AWS).

A contrapartida: **compatibilidade binária**. Código compilado para x86 não roda em ARM sem recompilar. Linguagens interpretadas ou com JIT (Python, Node.js, Java, Go) recompilam transparentemente. Código nativo (C/C++, Rust) precisa ser recompilado — 99% dos pacotes Linux já têm builds ARM64 pré-construídos. Bibliotecas proprietárias são o ponto de atenção real.

</blockquote>
</details>

## Contexto

A AWS está empurrando Graviton há anos porque é a arquitetura mais barata para operar dentro dos data centers dela. A pergunta prática para quem decide arquitetura é: **meu workload ganha ou perde ao migrar?** Este lab te dá 6 benchmarks como amostra — você extrapola para seu caso real. O próximo lab ([03.2 ECS+Fargate](../02-ECS-Fargate/README.md)) mostra como escolher a arquitetura de um container ECS entre x86 e Graviton no nível da task definition.

---

## Parte 1 - Provisionar as duas EC2s

### Resultado esperado desta parte

Duas EC2s em estado `running`, ambas com `3/3 verificações aprovadas` no console, acessíveis via SSM em duas abas do navegador nomeadas `x86` e `graviton`.

1. No Codespaces, entre na pasta do Terraform:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/03-Compute/01-X86-Graviton/terraform
```

2. Descubra o bucket de estado e substitua o placeholder em `state.tf`:

```bash
export bucket=$(aws s3 ls | awk '/base-config-/ {print $3; exit}')
echo "Bucket detectado: $bucket"
sed -i "s/base-config-SEU_RM/$bucket/g" state.tf
```

Se `Bucket detectado:` veio vazio, **pare** — revise o [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md).

3. Inicialize e aplique o Terraform:

```bash
terraform init
terraform apply -auto-approve
```

<details>
<summary><b>💡 Clique para entender — o que o Terraform provisiona</b></summary>
<blockquote>

Duas EC2s gêmeas em tudo exceto arquitetura:

| Campo | x86 | Graviton |
|-------|-----|----------|
| Tipo | `t3.large` | `t4g.large` |
| Arquitetura | `x86_64` (Intel) | `arm64` (Graviton 2) |
| vCPU | 2 | 2 |
| RAM | 8 GB | 8 GB |
| Preço on-demand (us-east-1) | ~$0.0832/h | ~$0.0672/h |

Ambas rodam **Ubuntu 22.04** e executam o mesmo [`install.sh`](terraform/install.sh) via user-data. O script instala `sysbench`, `gzip`, `python3`, `nodejs` e outras dependências. Como o script é idêntico para as duas arquiteturas, o `apt` resolve os pacotes ARM64 ou x86 automaticamente — **zero código específico de arquitetura**.

Acesso é via **SSM Session Manager**, não SSH. Não precisa de chave, não abre porta 22, e o comando fica registrado se você configurar logging.

</blockquote>
</details>

4. Acesse o [console do EC2](https://us-east-1.console.aws.amazon.com/ec2/home?region=us-east-1#Instances:instanceState=running) e aguarde as duas instâncias atingirem `running` + `3/3 verificações aprovadas` (5-10 min para o user-data terminar de rodar).

<!-- PRINT SUGERIDO: img/ec2-instances.png
     Console EC2 mostrando as duas instâncias (x86 e graviton) em running com 3/3 checks OK. -->
![](img/ec2-instances.png)

> [!NOTE]
> **Não siga antes dos 3/3 verificações** — se o user-data ainda estiver rodando, `sysbench` e outros utilitários vão faltar quando você abrir a sessão. Espere até os checks ficarem verdes.

5. Selecione **as duas instâncias** e clique em `Conectar` na barra superior. Isso vai abrir duas abas de conexão, uma para cada instância.

<!-- PRINT SUGERIDO: img/1.png
     Tela de conectar com ambas as instâncias selecionadas. -->
![](img/1.png)

6. Em cada aba, selecione `Gerenciador de sessões` e clique em `Conectar`.

<!-- PRINT SUGERIDO: img/2.png
     Aba "Gerenciador de sessões" selecionada com o botão Conectar à vista. -->
![](img/2.png)

7. Você terá agora **duas abas de terminal abertas**, uma para cada EC2. **Confira o nome de cada aba** — uma deve ser `x86`, a outra `graviton`. Não confunda durante os benchmarks.

<!-- PRINT SUGERIDO: img/3.png
     Terminal do SSM da instância x86 conectado. -->
![](img/3.png)

<!-- PRINT SUGERIDO: img/4.png
     Terminal do SSM da instância graviton conectado. -->
![](img/4.png)

> [!WARNING]
> **Daqui em diante todos os comandos das Partes 2-6 rodam nas DUAS sessões SSM simultaneamente.** Execute cada comando em `x86` e `graviton` antes de passar ao próximo passo. Isso permite comparação lado a lado.

8. **Em ambas as sessões SSM**, instale as dependências (caso o user-data ainda não tenha terminado, reinstalar é idempotente):

```bash
curl -Ssl https://raw.githubusercontent.com/vamperst/fiap-arquitetura-compute-e-storage/refs/heads/master/03-Compute/01-X86-Graviton/terraform/install.sh | bash
```

> [!TIP]
> Quando o terminal ficar poluído de saída de comando, pressione **`Ctrl + L`** para limpar (equivalente a `clear`). Facilita focar na saída do benchmark.

### Checkpoint

- [x] `terraform apply` terminou com `Apply complete!`.
- [x] Console EC2 mostra `x86` e `graviton` em `running` com `3/3 verificações aprovadas`.
- [x] Duas sessões SSM abertas, claramente identificadas.
- [x] `sysbench --version` funciona em ambas as sessões.

<details>
<summary><b>⚠ Se der erro: <code>sysbench: command not found</code></b></summary>
<blockquote>
O user-data ainda estava rodando quando você conectou. Re-execute o `curl ... | bash` do passo 8 para forçar a instalação.
</blockquote>
</details>

---

## Parte 2 - Benchmark de CPU single-thread

### Resultado esperado desta parte

Graviton processa aproximadamente **3× mais eventos por segundo** que x86 neste teste — a vantagem ARM para cálculo matemático pesado fica evidente.

9. **Em ambas as sessões** (x86 e graviton), execute:

```bash
sysbench cpu --cpu-max-prime=20000 --time=10 run
```

<!-- PRINT SUGERIDO: img/5.png
     Saída do sysbench na instância x86: events per second ~317. -->
![](img/5.png)

<!-- PRINT SUGERIDO: img/6.png
     Saída do sysbench na instância graviton: events per second ~1070. -->
![](img/6.png)

<details>
<summary><b>💡 Clique para entender — o que <code>sysbench cpu</code> mede</b></summary>
<blockquote>

O módulo `cpu` do sysbench calcula **números primos até N** repetidamente pelo tempo especificado. Por que primos? Porque é um cálculo que:

- Usa **apenas CPU** (não disco, rede ou RAM intensiva).
- Não é vetorizável trivialmente (evita que SIMD distorça a comparação).
- Repetitivo e determinístico (variação entre execuções é baixa).

Flags usados:

- `--cpu-max-prime=20000` → calcula primos até 20k a cada iteração.
- `--time=10` → roda por 10 segundos.
- Default: **1 thread** (single-core).

Métricas que importam:

- **Events per second** → quantos cálculos completos em 1 segundo. Quanto maior, melhor.
- **Avg latency (ms)** → tempo médio por cálculo. Quanto menor, melhor.

</blockquote>
</details>

Ordem de grandeza esperada (varia com carga do hypervisor):

| Métrica | x86 (`t3.large`) | Graviton (`t4g.large`) |
|---------|------------------|-------------------------|
| Events/s | ~317 | ~1070 |
| Total events (10s) | ~3178 | ~10704 |
| Avg latency (ms) | ~3.14 | ~0.93 |

A vantagem Graviton em single-thread vem em boa parte da frequência sustentada mais alta e da ISA mais direta para cálculos aritméticos.

### Checkpoint

- [x] Valores de `events per second` anotados para as duas máquinas.
- [x] Graviton apresentou valor **significativamente maior** (esperado ~3×).

---

## Parte 3 - Benchmark de CPU multi-thread

### Resultado esperado desta parte

Confirma a escala com 2 threads (ambas as máquinas têm 2 vCPUs). A proporção entre x86 e Graviton se mantém similar à Parte 2.

10. **Em ambas as sessões**, execute:

```bash
sysbench cpu --cpu-max-prime=30000 --time=40 --threads=2 run
```

<!-- PRINT SUGERIDO: img/Chart1.png
     Gráfico/saída comparativa de events/s entre x86 e graviton com 2 threads. -->
![](img/Chart1.png)

<!-- PRINT SUGERIDO: img/Chart2.png
     Gráfico de total de eventos em 40s. -->
![](img/Chart2.png)

<!-- PRINT SUGERIDO: img/Chart3.png
     Gráfico de latência média. -->
![](img/Chart3.png)

Ordem de grandeza esperada:

| Métrica | x86 (`t3.large`) | Graviton (`t4g.large`) |
|---------|------------------|-------------------------|
| Events/s | ~369 | ~1217 |
| Total events (40s) | ~14.779 | ~48.687 |
| Avg latency (ms) | ~5.41 | ~1.64 |

A diferença de latência (**~70% menor no Graviton**) reforça a intuição: para APIs que respondem por evento, Graviton devolve mais rápido por request.

### Checkpoint

- [x] Mesmo padrão da Parte 2 se manteve — Graviton ~3× mais rápido em events/s.
- [x] Latência Graviton ~70% menor.

---

## Parte 4 - Benchmark de memória

### Resultado esperado desta parte

x86 ganha esse teste **por ~10%**. Ao contrário da CPU, banda de memória é uma área onde x86 ainda tem vantagem marginal em instâncias comparáveis.

11. **Em ambas as sessões**, execute:

```bash
sysbench memory --memory-block-size=1M --memory-total-size=10G run
```

<details>
<summary><b>💡 Clique para entender — <code>sysbench memory</code> em 3 parágrafos</b></summary>
<blockquote>

O módulo `memory` faz **leituras e escritas sequenciais na RAM** sem tocar disco ou rede. O objetivo é medir a **banda pura de memória** do par CPU+RAM.

Flags:

- `--memory-block-size=1M` → cada operação manipula 1 MB.
- `--memory-total-size=10G` → teste processa 10 GB no total.

Métricas: velocidade em **MiB/s** e latência média.

Use este benchmark para estimar performance de: caches em RAM (Redis local, memcached), bancos in-memory (DuckDB, Polars), processamento de arquivos grandes em buffer. Se o workload é **memory-bound**, esse número é mais relevante que o `cpu`.

</blockquote>
</details>

Ordem de grandeza esperada:

| Métrica | x86 (`t3.large`) | Graviton (`t4g.large`) |
|---------|------------------|-------------------------|
| Velocidade (MiB/s) | ~14021 | ~12637 |
| Latência (ms) | ~0.07 | ~0.08 |
| Tempo total (s) | ~0.73 | ~0.81 |

### Checkpoint

- [x] x86 ~10% mais rápido em banda de memória.
- [x] Ambos com latência sub-milissegundo (excelente para workloads in-memory).

---

## Parte 5 - Benchmark de compressão (gzip)

### Resultado esperado desta parte

Graviton ~17% mais rápido a comprimir 1 GB de dados aleatórios. Compressão é um dos casos onde a ISA mais direta do ARM casa bem com algoritmos bit-twiddling.

12. **Em ambas as sessões**, execute:

```bash
mkdir -p teste-arquivo && cd teste-arquivo
dd if=/dev/urandom of=testfile.bin bs=1M count=1024
time gzip testfile.bin
```

<!-- PRINT SUGERIDO: img/Chart4.png
     Gráfico de tempo total de compressão: x86 ~48s, graviton ~40s. -->
![](img/Chart4.png)

<!-- PRINT SUGERIDO: img/Chart5.png
     Gráfico de throughput na criação do dd. -->
![](img/Chart5.png)

<!-- PRINT SUGERIDO: img/Chart6.png
     Gráfico de tempo de CPU user+sys. -->
![](img/Chart6.png)

<details>
<summary><b>💡 Clique para entender — por que <code>/dev/urandom</code></b></summary>
<blockquote>

Usamos dados aleatórios (`/dev/urandom`) no lugar de zeros porque zeros comprimem muito bem e o benchmark vira medida de memória, não de CPU. Dados aleatórios **não comprimem** — o gzip trabalha sem achar padrão algum, maximizando uso de CPU.

A geração do arquivo pelo `dd if=/dev/urandom` também é custosa: é o kernel rodando CSPRNG. O Graviton tende a gerar urandom **mais rápido** que x86, o que aparece no throughput do `dd` também.

</blockquote>
</details>

Ordem de grandeza esperada:

| Métrica | x86 (`t3.large`) | Graviton (`t4g.large`) |
|---------|------------------|-------------------------|
| Tempo total de `gzip` (real) | ~48s | ~40s |
| Throughput do `dd urandom` | ~227 MB/s | ~273 MB/s |
| CPU user+sys | ~48s | ~39s |

Graviton ganha em **três frentes** simultâneas: menos tempo total, maior throughput de urandom, menos CPU gasto.

### Checkpoint

- [x] Tempos anotados para os dois.
- [x] Graviton foi ~15-20% mais rápido em compressão.

---

## Parte 6 - Workloads reais: Python e Node.js

### Resultado esperado desta parte

Python recursivo: x86 levemente à frente (~3-8%). Node.js hashing: praticamente empate. Graviton não vence sempre — runtime e bibliotecas C subjacentes importam.

13. **Teste Python** — cálculo recursivo de Fibonacci(40). **Em ambas as sessões**:

```bash
echo "🐍 Teste com Python - cálculo de Fibonacci"
cat << 'EOF' > cpustress.py
import time

def fib(n):
    if n <= 1:
        return n
    else:
        return fib(n-1) + fib(n-2)

start = time.time()
print(f"Fibonacci(40): {fib(40)}")
print(f"Execution Time: {time.time() - start} seconds")
EOF
python3 cpustress.py
```

Ordem de grandeza esperada:

| Arquitetura | Tempo de execução |
|-------------|-------------------|
| x86 (`t3.large`) | ~37.9s |
| Graviton (`t4g.large`) | ~41.0s |

<details>
<summary><b>💡 Por que x86 ganha esse aqui e Graviton perde</b></summary>
<blockquote>

Python CPython é um **interpretador**, não JIT — cada chamada de função recursiva passa por bytecode + dispatch de opcode em software. Esse dispatch foi **historicamente mais otimizado para x86**, especialmente em micro-benchmarks de recursão pesada.

Alternativas que mudam o resultado: **PyPy** (JIT) costuma ser mais rápido em ambas e muitas vezes inverte o resultado. Bibliotecas nativas (NumPy, pandas) usam BLAS/LAPACK otimizados por arquitetura — nesses casos, Graviton volta a ganhar.

Lição: **não extrapole** o resultado de um benchmark para todo o ecossistema da linguagem. Mede o seu workload real.

</blockquote>
</details>

14. **Teste Node.js** — SHA-256 × 10 milhões de iterações. **Em ambas as sessões**:

```bash
echo "🟦 Teste com Node.js - hash SHA256"
cat << 'EOF' > hash.js
const crypto = require('crypto');

console.time('hash');
for (let i = 0; i < 1e7; i++) {
  crypto.createHash('sha256').update('AWS Academy').digest('hex');
}
console.timeEnd('hash');
EOF
node hash.js
```

Ordem de grandeza esperada:

| Arquitetura | Tempo de execução |
|-------------|-------------------|
| x86 (`t3.large`) | ~28.6s |
| Graviton (`t4g.large`) | ~29.1s |

**Diferença < 2%** — praticamente empate. Quando o runtime e a biblioteca subjacente (OpenSSL, neste caso) têm builds bem otimizados para as duas ISAs, a arquitetura **deixa de ser o fator dominante**.

### Checkpoint

- [x] Python: x86 levemente mais rápido em recursão pesada.
- [x] Node.js: empate técnico.
- [x] Lição absorvida: Graviton não vence sempre, mas perde por pouco — e é mais barato.

---

## Parte 7 - Limpeza

### Resultado esperado desta parte

Ambas as EC2s destruídas, conta AWS sem cobrança residual.

> [!CAUTION]
> Duas EC2s `.large` custam ~$0.15/hora somadas (~$3.60/dia). Em uma semana esquecida você esgota boa parte da cota da AWS Academy. **Destrua agora**.

15. **No Codespaces** (não na sessão SSM — a sessão SSM morre quando a EC2 morre), destrua o ambiente:

```bash
cd /workspaces/fiap-arquitetura-compute-e-storage/03-Compute/01-X86-Graviton/terraform
terraform destroy -auto-approve
```

16. Confirme que as instâncias sumiram:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=x86,graviton" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId"
```

Saída esperada: `[]`.

### Checkpoint

- [x] `terraform destroy` terminou com `Destroy complete!`.
- [x] Nenhuma EC2 `x86` ou `graviton` aparece em estado `running`.

---

## Conclusão

Três lições do lab, destiladas dos 6 benchmarks:

1. **Graviton vence em CPU e compressão — onde a ISA importa.** 3× em cálculo aritmético, ~17% em gzip. Para workloads CPU-bound com software bem portado para ARM, Graviton entrega mais performance por core.
2. **Graviton empata ou perde em memória e runtime interpretado.** Banda de RAM, Python recursivo. Não é regra — tende a ser função de otimização de biblioteca subjacente.
3. **Graviton é ~20% mais barato** em todas as classes (`t4g` vs `t3`, `c7g` vs `c6i`, etc). Mesmo **empatando** em performance, o custo por hora fica menor — daí o 40% preço/performance anunciado pela AWS em workloads bem portados.

A decisão de migrar não é "Graviton sempre" nem "x86 sempre" — é **"meça o seu workload real"**. Este lab te deu a ferramenta (sysbench, time, scripts) para fazer essa medição.

## Próximo passo

No [lab 03.2 de ECS+Fargate](../02-ECS-Fargate/README.md) você aplica essa decisão em um container ECS. A arquitetura da task definition (`runtime_platform`) é onde você marca x86 ou ARM — exatamente o ponto de decisão que este lab te preparou para tomar.

---

<details>
<summary><b>💡 Glossário rápido</b></summary>
<blockquote>

| Termo | O que é |
|-------|---------|
| x86_64 | ISA CISC dominante em servidores (Intel, AMD). Retrocompatível desde os 80. |
| ARM64 | ISA RISC moderna. Usada em Graviton (AWS), Ampere, Apple Silicon. |
| Graviton | Linha de CPUs ARM desenhada pela AWS. Graviton 2/3/4 disponíveis. |
| `t3.large` | EC2 x86 burstable de 2 vCPU + 8 GB RAM. |
| `t4g.large` | EC2 Graviton 2 equivalente a `t3.large`, ~20% mais barato. |
| `sysbench` | Ferramenta de benchmark modular portável (CPU, memória, I/O, MySQL). |
| SSM Session Manager | Serviço AWS que dá shell em EC2 sem expor porta 22. |
| SIMD | Single Instruction Multiple Data — instruções vetoriais (AVX no x86, NEON no ARM). |
| CISC / RISC | Complex vs Reduced Instruction Set Computing — filosofias de design de ISA. |
| `/dev/urandom` | Dispositivo Linux que entrega bytes criptograficamente aleatórios. |
| `time` | Built-in shell que mede o tempo de execução de um comando (real, user, sys). |

</blockquote>
</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>
<blockquote>

**Antes de abrir issue ou chamar o professor, colete:**

1. Em qual passo (número) travou.
2. A mensagem de erro **literal** (copie e cole).
3. Em qual máquina o erro apareceu (`x86`, `graviton`, Codespaces).
4. O que `uname -m` retorna dentro da EC2 (deve ser `x86_64` ou `aarch64`).

**Canais, em ordem:**

1. [Issues deste repositório](https://github.com/rafaelmbarbosa/fiap-arquitetura-compute-e-storage/issues) — preferido, cria histórico pesquisável.
2. Email do professor com os 4 itens acima.
3. Na sala de aula, durante o laboratório.

</blockquote>
</details>
