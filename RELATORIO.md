# Demanda, Provisionamento e Desempenho

**Grupo:** Daniel Anderson de Souza Silva · Ranya Duran Greco

| Nome | Matrícula |
| --- | --- |
| Daniel Anderson de Souza Silva | 0126015852-52M |
| Ranya Duran Greco | 0126015844-44M |

> **Conformidade com o template.** Este documento segue as seções do template
> disponibilizado pelo professor — *Aplicações desenvolvidas*; *Configuração da
> VM e Deploy da Aplicação*; *Teste de Stress da Aplicação Selecionada* — e
> cobre, dentro delas, os seis itens mínimos do enunciado: título e grupo (aqui),
> aplicações desenvolvidas (§1), configuração da VM e deploy (§2), teste de
> stress da aplicação selecionada (§3.1 a §3.3), resultados comparativos (§3.3 e
> §3.5) e interpretação e conclusões (§3.7). A §3.4 é um acréscimo nosso: uma
> segunda execução do experimento, feita no ambiente do LCC3, usada como
> replicação independente do achado principal.

---

## Conjuntos de dados deste documento

Este trabalho produziu **três conjuntos de dados distintos**, em momentos
diferentes e sobre configurações diferentes. Como as conclusões não valem
igualmente para todos, cada seção abaixo declara explicitamente de qual conjunto
está falando.

| Conjunto | Onde está | Origem | Quando (UTC) | O que contém | Onde é usado |
| --- | --- | --- | --- | --- | --- |
| **A** — matriz completa | `atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/` | VM `Po-2026.2`, instalada pelo grupo a partir do ISO | 2026-10-14, 19h07–19h23 | 12 pontos: C1–C4 × níveis 1, 2 e 4. Perfil **memory**, 256 MiB | §2 e §3.1 a §3.3 — é o conjunto principal |
| **B** — replicação no LCC3 | [`lcc3/dados/`](lcc3/dados/) | VM `Po_2_2026.2`, construída da imagem do professor | 2026-10-14, 22h08–22h15 | 6 pontos: C1 e C3 × níveis 1, 2 e 4. Perfil **memory**, 256 MiB | §3.4 e §3.5 — replicação independente |
| **Ensaio preliminar** | `atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/ensaio-2026-10-14/` | VM `Po-2026.2` | 2026-10-14, 16h17–16h23 | 7 pontos, perfil **cpu**, size 10, níveis 1 a 64 | **Não usado em nenhuma conclusão.** Sondagem para calibrar o gerador de carga e escolher o perfil |

**Por que o ensaio preliminar não entra nas conclusões.** Ele mediu o perfil
*CPU-bound* com `size=10` e níveis até 64 concorrentes — uma configuração que não
faz parte da matriz de provisionamento do enunciado. Ele serviu para verificar
que o gerador de carga, o monitor e a instrumentação funcionavam ponta a ponta e
para decidir qual dos três perfis seria o mais sensível à variação de recursos.
O resultado dessa decisão está na §2.3. O conjunto é preservado como dado bruto
por transparência, e não sustenta nenhuma afirmação deste relatório.

**Relação entre A e B.** Os dois medem a mesma aplicação (memory-bound), com a
mesma carga (256 MiB por requisição) e os mesmos níveis de demanda (1, 2 e 4),
nas mesmas configurações de recurso para os cenários que têm em comum. O que
muda é a máquina: A roda na VM que instalamos do ISO, com disco de 12 GB; B roda
na VM construída a partir da imagem `ubuntu_serve_2204_poi.vdi` do professor,
seguindo o roteiro do LCC3. B é, portanto, uma **replicação em ambiente
independente** — e é isso que lhe dá valor: ele testa se o achado de A é uma
propriedade do sistema medido ou um artefato daquela máquina específica.

---

## 1. Aplicações desenvolvidas

Desenvolvemos três aplicações web de backend em Python, uma para cada perfil
computacional pedido pelo enunciado. As três compartilham a mesma estrutura e o
mesmo contrato HTTP; o que muda entre elas é exclusivamente o trabalho que
executam. Isso é deliberado: mantendo o servidor, o contrato e a instrumentação
idênticos, qualquer diferença de desempenho observada entre elas pode ser
atribuída ao perfil de carga, e não a detalhes de implementação.

### 1.1 Estrutura

```
Atividade2-Demanda-Provisionamento-Desempenho/
├── README.md                          enunciado da atividade e decisões do grupo
├── RELATORIO.md                       este documento, o único relatório do projeto
├── figuras/                           figuras do conjunto A e da configuração da VM
├── lcc3/                              execução no ambiente do LCC3 (conjunto B)
│   ├── dados/                         dados brutos, summary.csv e graphs/
│   └── evidencias/                    capturas de tela do roteiro do LCC3
└── atividade2-demanda-desempenho/
    ├── evidencias/                    configuração da VM e consoles do conjunto A
    └── atividade2/atividade2_demanda/
        ├── apps/
        │   ├── server.py              implementação única dos três perfis
        │   ├── cpu/server.py          lançador do perfil CPU-bound
        │   ├── memory/server.py       lançador do perfil Memory-bound
        │   └── io/server.py           lançador do perfil IO-bound
        ├── scripts/
        │   ├── load_test.py           gerador de carga (executa FORA da VM)
        │   ├── monitor.py             coleta de métricas da VM (executa NA VM)
        │   ├── run_scenario.py        orquestra um cenário completo
        │   ├── plot_results.py        gera os gráficos e a tabela consolidada
        │   └── capture_evidence.py    captura a configuração do VirtualBox
        ├── tests/
        │   └── test_applications.py   9 testes funcionais
        ├── requirements.txt
        └── results/                   dados brutos do conjunto A, summary.csv e graphs/
```

### 1.2 Contrato HTTP comum

| Rota | Resposta |
| --- | --- |
| `GET /health` | `{"status": "ok", "profile": "<perfil>"}` — identifica qual aplicação respondeu |
| `GET /work?size=N` | `{"profile": ..., "duration_seconds": ..., "result": {...}}` |
| Rota desconhecida | HTTP 404 |
| `size` inválido | HTTP 400 |
| Todos os slots ocupados | HTTP 503 |
| Falha de armazenamento (perfil IO) | HTTP 500 |

Todas as aplicações validam `size` antes de executar qualquer trabalho,
aceitando apenas um inteiro decimal positivo, e recusam valores acima do teto do
perfil. Esse teto existe para que um teste de carga não consiga levar a máquina a
consumo de recurso descontrolado.

### 1.3 As três aplicações

#### CPU-bound — `apps/cpu/server.py`

Executa SHA-256 repetidamente sobre um bloco fixo de 1 MiB, escrevendo o resumo
de volta nos primeiros 32 bytes do bloco, de modo que cada iteração dependa da
anterior e a requisição permaneça determinística. `size` é o número de iterações
(padrão 10, máximo 300). Resultado: `{"sha256": ..., "iterations": N}`.

O bloco ser grande é uma decisão de projeto, não um detalhe. O CPython libera a
GIL para entradas de `hashlib` com 2048 bytes ou mais. Uma cadeia de resumos
pequenos ficaria inteiramente dentro da GIL e a aplicação **não ganharia nada**
com vCPUs adicionais — exatamente a relação que o experimento precisa medir. Com
o bloco de 1 MiB, requisições concorrentes conseguem ocupar mais de um núcleo.

#### Memory-bound — `apps/memory/server.py`

Aloca `size` MiB, percorre o buffer escrevendo um byte a cada página de 4 KiB
(obrigando o sistema a materializar cada página em memória física) e devolve a
soma de uma amostra das páginas. `size` está em MiB (padrão 32, máximo 256).
Resultado: `{"allocated_mib": N, "checksum": ...}`.

O laço que percorre as páginas parece desperdício, mas é o que sustenta o
experimento. Ele devolve a GIL a cada ~5 ms, e é isso que permite que as
alocações de requisições concorrentes coexistam em memória ao mesmo tempo. Sem
esse laço — com o preenchimento feito em uma única chamada de C — o pico de
memória residente cai de 1033 MB para 265 MB e o cenário de 1 GB **deixa de
entrar em swap**. O detalhamento está na §3.3.8.

#### IO-bound — `apps/io/server.py`

Escreve `size` KiB em um arquivo temporário, força a descida para o disco com
`fsync`, e lê o arquivo de volta. `size` está em KiB (padrão 1024, máximo
32768). Resultado: `{"bytes_written": ..., "bytes_read": ...}`.

### 1.4 Execução

Cada aplicação é um processo independente:

```bash
python apps/cpu/server.py    --port 8000
python apps/memory/server.py --port 8000
python apps/io/server.py     --port 8000
```

Há também o ponto de entrada compartilhado, útil para testes:

```bash
python apps/server.py --profile memory --port 8000
```

Requisitos: Python 3.10 ou superior e `matplotlib` (apenas para os gráficos). O
servidor em si usa somente a biblioteca padrão. No interior da VM, o script de
monitoramento requer `python3-psutil`, instalado via `apt`.

### 1.5 Como a aplicação e a infraestrutura foram monitoradas

O monitoramento acontece em dois lugares, e essa separação é essencial ao
método:

- **Cliente (fora da VM)** — `scripts/load_test.py` mede a latência ponta a
  ponta que o cliente observa e a vazão efetiva. É dessa fonte que saem p50, p95,
  p99, vazão e contagem de falhas. Os percentis são calculados por interpolação
  linear sobre a amostra completa.
- **Servidor (dentro da VM)** — `scripts/monitor.py` amostra CPU, memória, swap e
  load average a cada ~0,5 s, usando `psutil`, e grava em `results/resources.csv`.

Medir a latência apenas dentro da aplicação omitiria o caminho cliente→servidor
e os efeitos da virtualização, que são justamente o objeto do estudo.

### 1.6 Testes

`tests/test_applications.py` contém 9 testes funcionais, cobrindo a identidade de
cada perfil, a forma do resultado de cada um, determinismo, tamanho padrão,
rejeição de tamanhos inválidos (`0`, `-1`, `1.5`, `not-a-number`, `1e3`,
`999999999`, vazio), HTTP 404, o ponto de entrada compartilhado, rejeição de
perfil desconhecido e o limite de admissão (HTTP 503).

```bash
python -m unittest discover -s tests -v
```

Nenhum teste verifica latência ou vazão: esses valores dependem da máquina e da
carga aplicada, portanto pertencem aos experimentos e ao relatório, nunca a um
critério de aprovação/reprovação.

---

## 2. Configuração da VM e Deploy da Aplicação

Foram usadas **duas máquinas virtuais**, uma para cada conjunto de dados. As duas
rodam a mesma aplicação e o mesmo método; o que difere é o disco de origem e o
ambiente hospedeiro. A Tabela abaixo é o mapa; as §2.2 e §2.3 detalham cada uma.

| | `Po-2026.2` (conjunto A) | `Po_2_2026.2` (conjunto B) |
| --- | --- | --- |
| Origem do disco | instalado pelo grupo a partir do ISO do Ubuntu Server 22.04 LTS | imagem `ubuntu_serve_2204_poi.vdi` do professor |
| Disco | 12 GB (VDI) | 500 GB (VDI dinâmico; ~5 GB ocupados) |
| Cenários medidos | C1, C2, C3, C4 | C1 e C3 |
| Redirecionamento de portas | `ssh` 2222→22, `app` 8080→8000 | `ssh` 2222→22, `app` 8080→8000, `web` 8090→80 |
| nginx | não instalado | instalado e verificado (§2.4) |
| Papel no trabalho | experimento principal | replicação independente |

### 2.1 Ambiente hospedeiro

| Item | Valor |
| --- | --- |
| Máquina | Notebook pessoal, Windows 10 Home Single Language (10.0.19045) |
| Processador | Intel Core i3-6006U @ 2,00 GHz — **2 núcleos físicos**, 4 lógicos |
| Virtualização | Oracle VirtualBox |
| Geração de carga | No próprio hospedeiro, via `scripts/load_test.py` |

O número de núcleos físicos é relevante e está registrado de propósito: ele
limita a precisão alcançável nos níveis de demanda mais altos, porque o
hospedeiro executa ao mesmo tempo a VM, o gerador de carga e o motor de NAT do
VirtualBox. Voltamos a esse ponto na §3.3.9.

### 2.2 VM do conjunto A — `Po-2026.2`

| Recurso | Valor |
| --- | --- |
| Nome | `Po-2026.2` |
| Sistema operacional | Ubuntu Server 22.04 LTS (64-bit) |
| Disco | 12 GB (VDI) |
| vCPUs | 1 ou 2, conforme o cenário (§3.1) |
| Memória | 1024 ou 2048 MB, conforme o cenário (§3.1) |
| Rede | NAT — endereço do guest `10.0.2.15` |
| Redirecionamento de portas | `ssh` 2222→22 e `app` 8080→8000 |
| Python | 3.10.12 |
| Monitoramento | `python3-psutil` 5.9.0 (via `apt`) |

### 2.3 A aplicação escolhida e suas características

O enunciado pede escolher **uma** das três aplicações para implantar na VM. A
escolhida foi a **Memory-bound**, e a mesma escolha vale para os dois conjuntos de
dados.

Suas características: recebe `size` em MiB, aloca um buffer desse tamanho,
percorre-o escrevendo um byte por página de 4 KiB — o que força o sistema
operacional a materializar cada página em memória física, em vez de deixá-la
como página zero preguiçosa — e devolve a soma de uma amostra das páginas como
verificação. O teto por requisição é de 256 MiB e o limite de admissão é de 4
requisições simultâneas, o que significa um pico teórico de 1 GiB de alocação
viva. É exatamente esse número que torna a aplicação sensível ao provisionamento
de memória da VM.

O motivo da escolha é metodológico. A matriz de cenários varia dois recursos:
vCPUs e memória. A aplicação Memory-bound é a única das três cujo comportamento é
diretamente controlado pelo provisionamento de memória — e o efeito é de ordem de
grandeza, não marginal: no cenário de 1 GB a VM troca páginas para o swap, no de
2 GB não troca nenhuma. As outras duas responderiam sobretudo a vCPUs, o que
produziria curvas muito mais próximas entre si. Essa previsão foi confirmada de
forma indireta pelo ensaio preliminar com o perfil CPU-bound, que atingiu 14 a 15
req/s estáveis em todos os níveis testados — uma curva essencialmente plana, sem
o fenômeno que este trabalho quer medir.

### 2.4 Etapas para a configuração das máquinas virtuais

**Conjunto A (`Po-2026.2`):**

1. Instalação do Ubuntu Server 22.04 LTS no VirtualBox, seguindo o roteiro do
   professor (Figuras 1 a 3). A instalação foi feita com 2048 MB de memória: com
   1024 MB — o mínimo aceito pelo instalador — o processo travava.
2. Configuração da rede em modo NAT com dois redirecionamentos de porta,
   `ssh` 2222→22 e `app` 8080→8000, para que o hospedeiro alcance o guest.
3. Instalação do `python3-psutil` no guest via `apt` (o guest não possui `pip`, e
   não é necessário).
4. Cópia do código para `~/atividade2` no guest.
5. Execução dos testes dentro do guest: 9/9 aprovados.
6. Início da aplicação: `python3 apps/memory/server.py --port 8000`.
7. Verificação a partir do hospedeiro: `curl http://127.0.0.1:8080/health`
   devolve `{"status": "ok", "profile": "memory"}`, confirmando que o caminho
   hospedeiro→NAT→guest está funcionando.

**Conjunto B (`Po_2_2026.2`)** — criada com o `VBoxManage`, a partir da imagem do
professor, e seguindo o roteiro do LCC3 passo a passo:

```bash
VBoxManage createvm --name "Po_2_2026.2" --ostype Ubuntu_64 \
    --basefolder "C:\Users\Daniel\VirtualBox VMs\Po-2026.2" --register
VBoxManage modifyvm "Po_2_2026.2" --memory 1024 --cpus 1 --firmware bios
VBoxManage modifyvm "Po_2_2026.2" \
    --nic1 nat --natpf1 "ssh,tcp,,2222,,22" \
    --natpf1 "app,tcp,,8080,,8000" --natpf1 "web,tcp,,8090,,80"
VBoxManage storagectl "Po_2_2026.2" --name SATA --add sata --controller IntelAhci
VBoxManage storageattach "Po_2_2026.2" --storagectl SATA --port 0 --device 0 \
    --type hdd --medium "...\ubuntu_serve_2204_poi.vdi"
```

O nome `Po_2_2026.2` segue o padrão pedido (`Po_2_<semestre>`). O login é o
documentado no roteiro do professor, `osboxes` / `osboxes.org` — o disco
entregue é a imagem dele, então a credencial é a dele.

**Decisão de origem do disco.** A VM do conjunto B poderia ter sido construída a
partir do ISO que já tínhamos. Optamos por partir da imagem do professor por
duas razões: o roteiro do LCC3 parte dela, e o enunciado pede o disco VDI
configurado como entrega. Construindo a partir da imagem dele, o disco entregue é
o ambiente que ele espera encontrar, com a credencial que ele mesmo documentou —
e o conjunto B ganha uma propriedade que o A não tem: ele é uma medição numa
máquina que **não** foi preparada por nós.

### 2.5 Requisitos e etapas do deploy

| Requisito | Situação |
| --- | --- |
| Ubuntu Server 22.04 LTS | instalado na VM `Po-2026.2`; presente na imagem do professor em `Po_2_2026.2` |
| Python 3.10+ | 3.10.12 em `Po-2026.2`; 3.10.4 em `Po_2_2026.2` |
| `python3-psutil` | instalado via `apt` para o monitor |
| Rede do guest alcançável pelo hospedeiro | NAT + redirecionamento 8080→8000 |
| Código no guest | copiado para `~/atividade2` |
| nginx (roteiro do LCC3) | instalado e verificado em `Po_2_2026.2` (§2.6) |

O deploy é direto porque a aplicação usa apenas a biblioteca padrão: não há
gerenciador de pacotes, build ou serviço a configurar. As etapas são a cópia do
código, a instalação do `psutil` e a execução do processo. Não há credenciais nem
segredos envolvidos na configuração da aplicação, e nenhuma senha é registrada
neste repositório.

### 2.6 Evidências da configuração

#### 2.6.1 Conjunto A

Cada cenário tem sua configuração registrada automaticamente pelo script
`scripts/capture_evidence.py`, que consulta o próprio VirtualBox e grava o
resultado. Optamos por capturar a configuração direto do VirtualBox, e não
redigitá-la, para que o registro não possa divergir do que de fato executou. As
capturas abaixo são as originais, copiadas para `figuras/`, ao lado deste
documento.

![Tela de configuração da placa-mãe da máquina virtual](figuras/vm-configuracao-placa-mae.png)

*Figura 1 — Configuração da placa-mãe no VirtualBox (`Po-2026.2`).*

![Tela de configuração do processador da máquina virtual](figuras/vm-configuracao-processador.png)

*Figura 2 — Configuração do processador no VirtualBox (`Po-2026.2`).*

![Ubuntu Server instalado e em execução](figuras/vm-ubuntu-instalada.png)

*Figura 3 — Ubuntu Server 22.04 LTS instalado e em execução (`Po-2026.2`).*

O registro completo de cada cenário está em
`atividade2-demanda-desempenho/evidencias/`:

| Arquivo | Conteúdo |
| --- | --- |
| `C1-config.txt` … `C4-config.txt` | Resumo do provisionamento e a descrição completa da máquina (`VBoxManage showvminfo --machinereadable`), lidos da máquina em execução |
| `C1-console.png` … `C4-console.png` | Console do guest logo após o boot, na mesma sessão em que a medição foi feita |

#### 2.6.2 Conjunto B

A configuração também foi lida do próprio VirtualBox e gravada em
[`lcc3/evidencias/config-C1.txt`](lcc3/evidencias/config-C1.txt) e
[`lcc3/evidencias/config-C3.txt`](lcc3/evidencias/config-C3.txt). As capturas a
seguir são o registro da VM e do roteiro do LCC3 cumprido.

![VM Po_2_2026.2 no gerenciador do VirtualBox](lcc3/evidencias/08-virtualbox-manager.png)

*Figura 4 — a VM `Po_2_2026.2` no gerenciador do VirtualBox: sistema Ubuntu
(64-bit), memória principal de 2048 MB (configuração C3) e o disco
`ubuntu_serve_2204_poi.vdi` anexado à porta SATA 0.*

![Primeiro boot da VM Po_2_2026.2](lcc3/evidencias/01-boot.png)

*Figura 5 — a VM `Po_2_2026.2` no primeiro boot, com o login `osboxes` já
efetuado. A tela mostra o sistema Ubuntu Server 22.04 LTS da imagem do
professor.*

![nginx instalado, ativo e respondendo](lcc3/evidencias/02-nginx.png)

*Figura 6 — nginx instalado e em execução, como o roteiro do LCC3 pede: a versão
instalada, o estado `active (running)` do serviço e o HTML devolvido por
`curl -s localhost:80`.*

![As três aplicações em execução dentro da VM](lcc3/evidencias/03-apps.png)

*Figura 7 — as três aplicações em execução simultânea dentro da VM
`Po_2_2026.2`: o comando que iniciou cada uma, a resposta de `/health`
identificando perfil e porta, e a resposta de `/work` de cada perfil.*

---

## 3. Teste de Stress da Aplicação Selecionada

### 3.1 Cenários de configuração de recursos

Foram definidos quatro cenários a partir da combinação de dois níveis de CPU (1 e 2 vCPUs) e dois níveis de memória (1024 MB e 2048 MB). Essa configuração permite analisar separadamente os efeitos do provisionamento de CPU e memória, comparando cenários que diferem em apenas um recurso.

| Cenário | vCPUs | Memória | O que isola | Medido em A | Medido em B |
| --- | --- | --- | --- | --- | --- |
| **C1** | 1 | 1024 MB | Linha de base | sim | sim |
| **C2** | 2 | 1024 MB | Efeito de dobrar a CPU, mantendo a memória | sim | não |
| **C3** | 1 | 2048 MB | Efeito de dobrar a memória, mantendo a CPU | sim | sim |
| **C4** | 2 | 2048 MB | Efeito de dobrar ambos | sim | não |

O conjunto B cobre apenas C1 e C3 — o par que isola a memória, que é a variável
cujo efeito se quer replicar. A ausência de C2 e C4 em B não é uma lacuna para as
conclusões: nenhuma delas depende do efeito de dobrar vCPUs isoladamente, e a
§3.3.7 mostra que esse efeito é nulo para esta aplicação.

### 3.2 Como foram realizados os testes de stress

**A carga é gerada fora da máquina virtual**, pelo hospedeiro, através do
redirecionamento de porta do NAT, usando `scripts/load_test.py` — um gerador
próprio, em Python, com um cliente por thread.

Escolhemos um gerador próprio em vez de uma ferramenta pronta (JMeter, `ab`,
`wrk`) por um motivo específico ao trabalho: precisávamos que cada requisição
fosse classificada individualmente por tipo de falha (timeout, conexão recusada,
HTTP 5xx) e que os percentis fossem calculados por interpolação linear sobre a
amostra completa, não estimados por histograma.

#### 3.2.1 Níveis de demanda

Três níveis por cenário, nos dois conjuntos: **1, 2 e 4 clientes concorrentes**.

O teto de 4 não é arbitrário. A aplicação Memory-bound tem um limite de admissão
de 4 requisições simultâneas (`DEFAULT_MAX_CONCURRENCY["memory"] = 4`), então o
nível 4 satura exatamente a capacidade configurada. Níveis acima disso apenas
exercitariam o caminho de rejeição por HTTP 503, que não é o que este
experimento quer medir.

#### 3.2.2 Tamanho da carga

Fixado em **256 MiB por requisição** (o máximo do perfil), nos dois conjuntos.

Esse valor foi escolhido por ser a única faixa em que o cenário de 1 GB e o de
2 GB se comportam de forma diferente. Com 4 clientes concorrentes, a carga
simultânea é de 4 × 256 MiB = 1024 MiB. Num guest de 1 GB, isso excede a memória
disponível e força o swap; num guest de 2 GB, cabe com folga. Tamanhos menores
não produziriam diferença alguma entre C1/C2 e C3/C4, e o experimento não teria
o que medir.

#### 3.2.3 Protocolo de medição

| Parâmetro | Valor |
| --- | --- |
| Aquecimento por nível | 5 s, não registrados |
| Medição por nível | 40 s |
| Timeout por requisição | 120 s |
| Repetições | 1 |
| Intervalo de amostragem da VM | ~0,5 s |
| Ordem | Níveis crescentes, dentro de um mesmo boot da VM |

O aquecimento existe para que o primeiro nível não seja penalizado por ser o
primeiro: ele aquece o cache de páginas do guest e leva o processo Python ao
regime estacionário.

#### 3.2.4 Coleta de dados

Três arquivos por conjunto, todos preservados como dados brutos:

| Arquivo | Conteúdo | Origem |
| --- | --- | --- |
| `measurements.csv` | Uma linha por nível: latências (p50/p95/p99), vazão, tentativas, sucessos, erros por tipo | Hospedeiro |
| `run_log.csv` | Janela UTC de início e fim de cada nível | Hospedeiro |
| `resources.csv` | ~0,5 s: CPU, memória, swap, load average | Guest |

No conjunto A os três estão em
`atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/`; no
conjunto B, em [`lcc3/dados/`](lcc3/dados/).

A atribuição das amostras de recurso ao nível de demanda correto é feita
cruzando-se as janelas UTC de `run_log.csv`. Sem isso, os gráficos de CPU e
memória por nível seriam médias de cenário inteiro e esconderiam justamente a
transição que interessa.

A partir desses três arquivos, `scripts/plot_results.py` gera a tabela
consolidada (`summary.csv`, uma linha por cenário e nível) e as figuras. Como o
script usa caminhos relativos ao diretório de trabalho, cada conjunto foi
processado no seu próprio diretório, de modo que um não sobrescrevesse o outro —
é por isso que os dados do conjunto B vivem em `lcc3/dados/` e não em
`results/`.

#### 3.2.5 Como reproduzir

```bash
# 1. Reconfigurar a VM para o cenário e iniciá-la
VBoxManage modifyvm "Po_2_2026.2" --cpus 1 --memory 1024
VBoxManage startvm  "Po_2_2026.2" --type headless

# 2. No guest: subir a aplicação e o monitor
python3 apps/memory/server.py --port 8000 &
python3 scripts/monitor.py --scenario C1 --seconds 185

# 3. No hospedeiro: varrer os níveis
python scripts/run_scenario.py --scenario C1 --size 256 \
    --levels 1,2,4 --duration 40 --warmup 5 --expect-profile memory

# 4. Gerar os gráficos (no diretório do conjunto, para não sobrescrever o outro)
python scripts/plot_results.py
```

---

### 3.3 Conjunto A — resultados comparativos entre os cenários

> **Conjunto de dados: A.** Doze pontos, C1–C4 × níveis 1, 2 e 4.
> Fonte: `atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/`.

#### 3.3.1 Tabela consolidada

Latência em milissegundos; vazão em requisições por segundo. A coluna de CPU é a
utilização **total da VM** (100% = todos os vCPUs provisionados ocupados), e por
isso não é comparável diretamente entre cenários de 1 e 2 vCPUs.

| Cenário | vCPU | RAM | Nível | Vazão | p50 | p95 | p99 | CPU | Memória | Swap |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C1 | 1 | 1024 | 1 | 3,72 | 266 | 293 | 341 | 96,1% | 268 MB | 62 MB |
| C1 | 1 | 1024 | 2 | 3,59 | 528 | 734 | 849 | 99,1% | 382 MB | 62 MB |
| C1 | 1 | 1024 | 4 | 1,98 | 659 | 5128 | 35405 | 73,3% | 638 MB | 337 MB |
| C2 | 2 | 1024 | 1 | 3,39 | 278 | 384 | 470 | 48,3% | 317 MB | 0 MB |
| C2 | 2 | 1024 | 2 | 3,47 | 537 | 844 | 891 | 50,6% | 494 MB | 0 MB |
| C2 | 2 | 1024 | 4 | 1,19 | 706 | 11616 | 40803 | 37,8% | 697 MB | 314 MB |
| C3 | 1 | 2048 | 1 | 3,44 | 270 | 411 | 579 | 96,2% | 322 MB | 0 MB |
| C3 | 1 | 2048 | 2 | 2,12 | 673 | 2171 | 3695 | 99,8% | 504 MB | 0 MB |
| C3 | 1 | 2048 | 4 | 2,05 | 1363 | 5633 | 8281 | 99,0% | 837 MB | 0 MB |
| C4 | 2 | 2048 | 1 | 3,68 | 265 | 308 | 331 | 48,0% | 328 MB | 0 MB |
| C4 | 2 | 2048 | 2 | 1,65 | 883 | 2247 | 4636 | 43,4% | 456 MB | 0 MB |
| C4 | 2 | 2048 | 4 | 2,07 | 1662 | 3545 | 4136 | 48,4% | 810 MB | 0 MB |

Em nenhum dos 12 níveis houve falha, erro ou timeout: 100% das requisições
respondidas com HTTP 200 (Figura 10).

#### 3.3.2 Picos por cenário

| Cenário | vCPUs | RAM | Pico de CPU | Pico de memória | **Pico de swap** |
| --- | --- | --- | --- | --- | --- |
| C1 | 1 | 1024 MB | 100% | 879 MB | **507 MB** |
| C2 | 2 | 1024 MB | 98% | 877 MB | **545 MB** |
| C3 | 1 | 2048 MB | 100% | 1202 MB | **0 MB** |
| C4 | 2 | 2048 MB | 53% | 1204 MB | **0 MB** |

#### 3.3.3 Gráficos

As figuras a seguir relacionam a demanda (número de clientes simultâneos) com o
desempenho observado e com a utilização dos recursos, em cada cenário. Elas são
geradas por `scripts/plot_results.py` em `results/graphs/` e copiadas para
`figuras/`, ao lado deste documento.

![Latência por demanda e provisionamento](figuras/latencia_vs_demanda.png)

*Figura 8 — Latência por demanda e provisionamento: p50, p95 e p99 em painéis
separados, um por percentil.*

![Vazão por demanda e provisionamento](figuras/vazao_vs_demanda.png)

*Figura 9 — Vazão por demanda e provisionamento: requisições bem-sucedidas por
segundo.*

![Taxa de falha por demanda e provisionamento](figuras/erros_vs_demanda.png)

*Figura 10 — Taxa de falha por demanda e provisionamento: nula em todos os doze
pontos medidos.*

![Utilização de CPU da VM por demanda e provisionamento](figuras/cpu_vs_demanda.png)

*Figura 11 — Utilização de CPU da VM por demanda e provisionamento. 100%
corresponde a todos os vCPUs provisionados ocupados, de modo que a altura não é
comparável entre cenários de 1 e de 2 vCPUs.*

![Memória utilizada na VM por demanda e provisionamento](figuras/memoria_vs_demanda.png)

*Figura 12 — Memória utilizada na VM por demanda e provisionamento.*

![Uso de swap na VM por demanda e provisionamento](figuras/swap_vs_demanda.png)

*Figura 13 — Uso de swap na VM por demanda e provisionamento — a evidência direta
do achado principal.*

#### 3.3.4 A assinatura de saturação

No cenário C1, a vazão permaneceu relativamente estável entre os níveis 1 e 2, passando de 3,72 para 3,59 requisições por segundo, e caiu para 1,98 requisições por segundo no nível 4. Ao mesmo tempo, o percentil 95 do tempo de resposta aumentou de 293 ms para 5128 ms, aproximadamente 17,5 vezes.

Esse comportamento é compatível com uma situação de saturação: o aumento da concorrência deixa de proporcionar ganhos de vazão e passa a provocar elevação acentuada da latência. Dessa forma, o aumento da demanda não se traduz necessariamente em maior capacidade de processamento, podendo resultar em maiores tempos de espera.

#### 3.3.5 O provisionamento de memória é o fator decisivo

O resultado mais limpo do experimento está na Figura 13 e na coluna de swap da
tabela §3.3.1. Os dois cenários com 1 GB entraram em swap sob carga — 507 MB em
C1 e 545 MB em C2. Os dois cenários com 2 GB registraram **zero** swap, em todos
os níveis.

Isso não é coincidência numérica: é a fronteira de memória sendo cruzada. Com 4
clientes de 256 MiB, a carga simultânea é de 1024 MiB. Somada ao consumo de base
do sistema (~230 MB), ela excede a memória utilizável de um guest de 1 GB — cujo
total já fica abaixo dos 1024 MB nominais por causa da reserva do hipervisor — e
cabe com folga nos 2 GB. Os picos medidos (Figura 12) confirmam a leitura: os dois
cenários de 1 GB chegaram a 879 MB e 877 MB de memória residente e ainda assim
precisaram recorrer ao swap, enquanto os de 2 GB chegaram a 1202 MB e 1204 MB sem
usá-lo.

O efeito sobre o desempenho é direto: no nível 4, o p95 de C4 (3545 ms) é
**3,3 vezes menor** que o de C2 (11616 ms), com a mesma CPU e o dobro de RAM. A
memória deixou de ser o gargalo.

#### 3.3.6 O gargalo migra quando se adiciona memória

Em C3 (2 GB, sem swap), o nível 4 ainda degrada: a vazão fica em 2,05 rps e o p50
sobe de 270 ms para 1363 ms. Se não há swap, o que está saturando?

A CPU. Em C3 a utilização fica em 99,0% do único núcleo durante todo o nível 4, e
a memória utilizada nesse nível (837 MB em média) está bem abaixo dos 2048 MB
provisionados. O gargalo **migrou** de memória para CPU.

Esse é o ponto central da interpretação: adicionar um recurso não elimina a
degradação, apenas desloca o limitante. Uma análise que olhasse só a vazão
concluiria que "C3 é igual a C1"; olhando swap e CPU separadamente, vê-se que são
situações fisicamente diferentes.

#### 3.3.7 O provisionamento de CPU não ajuda esta aplicação

Dobrar os vCPUs não trouxe ganho em nenhum nível. Comparando os pares que
diferem só na CPU:

| Par | Vazão no nível 4 | Variação |
| --- | --- | --- |
| C1 (1 vCPU) → C2 (2 vCPU), ambos 1 GB | 1,98 → 1,19 rps | piorou |
| C3 (1 vCPU) → C4 (2 vCPU), ambos 2 GB | 2,05 → 2,07 rps | empatou |

A explicação está na coluna de CPU da tabela §3.3.1 e na Figura 11. Nos cenários
de 2 vCPUs a utilização total fica em torno de 48%, ou seja, **cerca de um
núcleo** — a VM nunca ocupa os dois. O recurso adicional simplesmente não é
utilizado.

A causa é a GIL do CPython. O trabalho da aplicação Memory-bound é Python puro,
e a GIL serializa a execução de bytecode: apenas uma thread executa por vez,
independentemente de quantos núcleos existam. Adicionar vCPUs a um workload
preso à GIL é adicionar capacidade que a aplicação não consegue usar.

Vale registrar que isso **não é um defeito da aplicação**. É uma propriedade da
combinação entre o perfil escolhido e a linguagem. A aplicação CPU-bound, por
exemplo, não sofre disso: ela usa `hashlib` sobre blocos de 1 MiB, e o CPython
libera a GIL nesse caso, permitindo paralelismo real. O contraste entre as duas
é, em si, um resultado interessante sobre desempenho em Python.

#### 3.3.8 O laço de páginas é o que produz a pressão de memória

O laço que percorre as páginas (`for index in range(0, SIZE, 4096)`) parece
trabalho desperdiçado: o `bytearray(256 MiB)` já materializa todas as páginas
numa única operação de C. Para medir o que ele de fato faz, comparamos as duas
variantes no guest, com 4 requisições concorrentes de 256 MiB:

| Variante | Tempo por requisição | Pico de RSS do processo |
| --- | --- | --- |
| `bytearray()` + checksum, sem laço | 288 ms | **265 MB** |
| Implementação atual, com o laço | 298 ms | **1033 MB** |

O resultado é o oposto do que a aparência sugere:

1. **O laço custa 10 ms de 298 ms — cerca de 3%.** Quem domina o tempo é o
   próprio `bytearray(256 MiB)`, que já materializa todas as páginas numa única
   operação de C. Otimizar o laço não aceleraria praticamente nada.
2. **O laço é o que produz o swap.** Ele devolve a GIL a cada ~5 ms, e é
   justamente isso que permite que as quatro alocações coexistam — daí o pico de
   1033 MB. Sem o laço, a operação de C segura a GIL do início ao fim, as
   alocações não se sobrepõem, o pico cai para 265 MB e **o cenário de 1 GB
   deixaria de entrar em swap**. Removê-lo destruiria o achado principal, não o
   melhoraria.

O laço é, portanto, mantido de forma deliberada: ele não é um custo a eliminar, e
sim o mecanismo que dá ao perfil Memory-bound a sua característica distintiva.

#### 3.3.9 Limitações do conjunto A

Declaramos estas limitações explicitamente. Elas restringem a **precisão** dos
números, mas não sustentam nem derrubam as conclusões.

**Contenção no hospedeiro.** O hospedeiro tem 2 núcleos físicos e executa
simultaneamente a VM, o gerador de carga e o motor de NAT do VirtualBox. Isso
produz ruído em todos os níveis. A evidência mais clara são duas anomalias sem
explicação física: C3 no nível 2 (2,12 rps) e C4 no nível 2 (1,65 rps) saíram
**piores** que C1 no nível 2 (3,59 rps), embora ambos tenham mais memória que C1.
Não há mecanismo que explique isso; é ruído de medição. Medições mais finas
exigiriam gerar a carga a partir de uma segunda máquina — e é exatamente isso que
o conjunto B permite testar, porque foi executado em outra sessão, com outra
máquina, e o resultado está na §3.4.4.

**Sem repetições.** Cada nível foi medido uma única vez. Não temos, portanto,
desvio padrão nem intervalo de confiança, e não é possível separar o efeito real
do ruído ponto a ponto. Como consequência, damos peso às **tendências** entre
cenários, não a diferenças isoladas entre níveis vizinhos. É também por isso que
a replicação do conjunto B tem valor: ela não elimina a limitação, mas permite
comparar dois conjuntos independentes.

**Amostras pequenas.** Cada requisição de 256 MiB custa cerca de 288 ms de
preenchimento serializado pela GIL (§3.3.8), o que fixa o teto em ~3,5 rps e
produz apenas 57 a 149 requisições por nível. O p50 é razoavelmente estável, mas
o p95 e principalmente o p99 repousam sobre poucas observações — o p99 de
35405 ms em C1 nível 4, por exemplo, corresponde a um único evento.

**Swap residual em C1.** O guest não foi reiniciado entre as sondas da §3.3.8 e a
medição de C1: a primeira amostra de recursos de C1, coletada ainda durante a
espera inicial e portanto antes de o nível 1 começar, já registrava 62,98 MB de
swap em uso. É esse valor que aparece nos níveis 1 e 2 de C1. O resíduo afeta a
linha de base absoluta de C1, mas não seu pico de 507 MB nem qualquer conclusão.

**HTTP 503 nunca observado.** O caminho de rejeição por limite de admissão existe
e é testado unitariamente, mas não foi exercitado sob carga. A serialização pela
GIL faz com que poucas requisições estejam em voo simultaneamente, então o limite
de 4 slots nunca é atingido. Torná-lo observável exigiria baixar o limite entre
cenários, o que alteraria a configuração da aplicação no meio da comparação.

**Uma aplicação.** O teste de stress cobre apenas a aplicação Memory-bound, como
o enunciado permite. As três aplicações são exercitadas localmente pela suíte de
testes descrita na §1.6, que sobe cada perfil e verifica o seu comportamento, mas
as conclusões quantitativas desta seção valem apenas para o perfil Memory-bound.

---

### 3.4 Conjunto B — replicação no ambiente do LCC3

> **Conjunto de dados: B.** Seis pontos, C1 e C3 × níveis 1, 2 e 4.
> Fonte: [`lcc3/dados/`](lcc3/dados/). VM `Po_2_2026.2`, construída da imagem
> `ubuntu_serve_2204_poi.vdi` do professor.

O conjunto B repete C1 e C3 — o par que isola a memória — numa máquina diferente,
preparada de outra forma e medida em outra sessão. Se o achado da §3.3.5 for uma
propriedade real do sistema, ele deve reaparecer aqui. Se for um artefato da
máquina do conjunto A, não deve.

#### 3.4.1 Tabela consolidada

| Cenário | vCPU | RAM | Nível | Concluídas | Erros | Vazão | p50 | p95 | CPU | Memória | Swap |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C1 | 1 | 1024 | 1 | 139/139 | 0 | 3,46 | 276 | 345 | 96,0% | 314 MB | 1 MB |
| C1 | 1 | 1024 | 2 | 146/146 | 0 | 3,62 | 538 | 703 | 98,7% | 430 MB | 13 MB |
| C1 | 1 | 1024 | 4 | 112/112 | 0 | 2,57 | 908 | 2086 | 91,1% | 606 MB | **318 MB** |
| C3 | 1 | 2048 | 1 | 147/147 | 0 | 3,65 | 266 | 322 | 96,2% | 289 MB | 0 MB |
| C3 | 1 | 2048 | 2 | 147/147 | 0 | 3,63 | 526 | 681 | 99,3% | 439 MB | 0 MB |
| C3 | 1 | 2048 | 4 | 152/152 | 0 | 3,79 | 1037 | 1444 | 100,0% | 748 MB | **0 MB** |

Também aqui não houve um único erro: as 843 requisições disparadas foram todas
atendidas. O sistema não falha sob esta carga — ele fica **lento**.

#### 3.4.2 Evidência direta do swap, capturada durante a execução

As duas figuras abaixo foram capturadas do console da VM **durante** a execução,
não depois. Elas mostram `free -m`, as últimas amostras do monitor e o `uptime`,
e são a evidência mais direta de que a diferença entre C1 e C3 é o swap.

![C1 em execução](lcc3/evidencias/04-stress-C1.png)

*Figura 14 — cenário C1 durante a execução: coluna `Swap` já em uso e CPU
saturada nas amostras do monitor.*

![C1 ao final da execução](lcc3/evidencias/05-stress-C1-fim.png)

*Figura 15 — cenário C1 ao final da varredura. O swap usado chegou a 327 MiB
contra 971 MiB de RAM visível: a VM passou boa parte da execução paginando.*

![C3 em execução](lcc3/evidencias/06-stress-C3.png)

*Figura 16 — cenário C3 durante a execução, com a mesma carga e o mesmo número de
vCPUs de C1. A diferença é a RAM: 1977 MiB em vez de 971 MiB.*

![C3 ao final da execução](lcc3/evidencias/07-stress-C3-fim.png)

*Figura 17 — cenário C3 ao final da varredura. O swap usado é **0**: com o dobro
de RAM, o mesmo trabalho coube inteiramente na memória física e a VM nunca
precisou paginar. A CPU continua saturada — o recurso escasso deixou de ser a
memória.*

#### 3.4.3 Gráficos

![Latência por demanda no conjunto B](lcc3/dados/graphs/latencia_vs_demanda.png)

*Figura 18 — latência mediana e cauda em função da demanda, no conjunto B.*

![Vazão por demanda no conjunto B](lcc3/dados/graphs/vazao_vs_demanda.png)

*Figura 19 — vazão em função da demanda, no conjunto B.*

![Swap por demanda no conjunto B](lcc3/dados/graphs/swap_vs_demanda.png)

*Figura 20 — swap em função da demanda, no conjunto B. É a figura que reproduz o
achado principal em ambiente independente.*

![CPU por demanda no conjunto B](lcc3/dados/graphs/cpu_vs_demanda.png)

*Figura 21 — uso de CPU em função da demanda, no conjunto B.*

Os gráficos de memória e de taxa de erro do conjunto B estão em
[`lcc3/dados/graphs/`](lcc3/dados/graphs/) — o de erro é nulo em todos os pontos,
como no conjunto A.

#### 3.4.4 O que a replicação mostra

**O achado principal se reproduz.** No nível 4, C1 vai a 318 MB de swap enquanto
C3 fica em zero. O conjunto A havia medido 337 MB e 0 MB para o mesmo par. A
magnitude praticamente idêntica em duas máquinas diferentes, com discos de origem
diferentes e medidos em sessões diferentes, é evidência forte de que o fenômeno é
real e não um artefato da instrumentação ou daquela VM específica. Compare as
Figuras 13 e 20: as duas curvas de swap têm a mesma forma.

**A anomalia do conjunto A não se repetiu.** No conjunto A, o cenário C3 apresentou vazão de 2,12 requisições por segundo e p95 de 2171 ms no nível 2, valores inferiores aos observados em C1 no mesmo nível de demanda. No conjunto B, C3 apresentou vazão de 3,63 requisições por segundo e p95 de 681 ms, mostrando um comportamento diferente daquele observado anteriormente.

Essa diferença pode estar associada à contenção de recursos no hospedeiro e à variabilidade das medições. Entretanto, como cada configuração foi medida apenas uma vez por ambiente, não é possível determinar com precisão a causa da diferença observada.

**O conjunto B é internamente mais coerente.** Em A, a vazão de C3 cai de 3,44
para 2,05 rps entre os níveis 1 e 4. Em B, ela *sobe* de 3,65 para 3,79. A série
de B é monotônica e descreve o comportamento esperado de um recurso saturado: a
CPU vai a 100% e sustenta a vazão máxima com latência proporcional à demanda,
sem o colapso causado por paginação. A explicação mais provável é que o conjunto
B rodou com menos contenção no hospedeiro, e não que a máquina do professor seja
intrinsecamente melhor — o conjunto A tinha 2 vCPUs em metade dos cenários
disputando os mesmos 2 núcleos físicos do hospedeiro.

**O que a replicação não cobre.** O conjunto B mediu apenas configurações de 1
vCPU. Ele portanto **não testa** a conclusão da §3.3.7, de que dobrar vCPUs não
ajuda esta aplicação. Essa conclusão continua apoiada apenas no conjunto A, e
essa é a principal lacuna que uma replicação futura deveria fechar.

---

### 3.5 Os dois conjuntos lado a lado

Comparação ponto a ponto dos seis resultados em comum entre os conjuntos A e B, correspondentes aos cenários C1 e C3, cada um avaliado com 1, 2 e 4 clientes concorrentes. A latência é apresentada em milissegundos e a vazão em requisições por segundo.

| Ponto | Vazão A | Vazão B | p95 A | p95 B | Swap A | Swap B | CPU A | CPU B |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C1 · nível 1 | 3,72 | 3,46 | 293 | 345 | 62 MB | 1 MB | 96,1% | 96,0% |
| C1 · nível 2 | 3,59 | 3,62 | 734 | 703 | 62 MB | 13 MB | 99,1% | 98,7% |
| **C1 · nível 4** | **1,98** | **2,57** | **5128** | **2086** | **337 MB** | **318 MB** | **73,3%** | **91,1%** |
| C3 · nível 1 | 3,44 | 3,65 | 411 | 322 | 0 MB | 0 MB | 96,2% | 96,2% |
| C3 · nível 2 | 2,12 | 3,63 | 2171 | 681 | 0 MB | 0 MB | 99,8% | 99,3% |
| C3 · nível 4 | 2,05 | 3,79 | 5633 | 1444 | 0 MB | 0 MB | 99,0% | 100,0% |

**O que é estável entre os dois conjuntos:**

| Grandeza | A | B | Leitura |
| --- | --- | --- | --- |
| Vazão nos níveis leves (1 e 2) | 3,4–3,7 rps | 3,5–3,6 rps | o teto da aplicação é o mesmo nas duas máquinas |
| Swap no nível 4 de C1 | 337 MB | 318 MB | o fenômeno tem a mesma magnitude |
| Swap no nível 4 de C3 | 0 MB | 0 MB | e a sua ausência também |
| CPU nos cenários de 1 vCPU | 96–100% | 96–100% | saturado nos dois |

**O que varia:** a latência de cauda no nível 4 e a vazão nos níveis pesados do
C3. Nos dois casos o conjunto B é melhor. Como a configuração da aplicação, o
tamanho da carga e os níveis de demanda são idênticos, a diferença só pode vir do
ambiente — contenção do hospedeiro, versão do kernel e do Python no disco de
origem, e a ordem em que as medições foram feitas. É precisamente por isso que
damos peso às grandezas estáveis da tabela acima, e não aos valores absolutos de
latência de um conjunto isolado.

---

### 3.6 Limitações gerais

Valem para os dois conjuntos, além das específicas da §3.3.9.

**Ausência de repetições por ponto.** Cada ponto foi medido uma única vez, nos
dois conjuntos. O conjunto B funciona como uma replicação *de cenário*, não como
repetição *de medição*: ele não permite calcular desvio padrão de um mesmo ponto.
A comparação entre A e B é entre duas medições independentes, não entre médias.

**Dois ambientes diferentes.** Como as duas VMs diferem no disco de origem (e
portanto na versão do kernel e do Python) e foram medidas em sessões distintas, a
diferença entre A e B confunde o efeito do provisionamento com o efeito do
ambiente. Isso não compromete a comparação **dentro** de cada conjunto, que é
onde estão todas as conclusões, mas impede tratar A e B como amostras de uma
mesma população.

**Cobertura desigual.** O conjunto A cobre os quatro cenários; o B, apenas dois.
Nenhuma conclusão deste relatório depende exclusivamente de um ponto que só
exista em um dos conjuntos, mas a §3.3.7 depende apenas de A.

---

### 3.7 Conclusões

1. **Provisionar memória resolveu o problema; provisionar CPU não.** Dobrar a RAM
   eliminou o swap (507/545 MB → 0 MB no conjunto A; 318 MB → 0 MB no conjunto B)
   e reduziu o p95 do nível mais exigente em 3,3 vezes. Dobrar os vCPUs não
   produziu ganho mensurável, porque a aplicação está presa à GIL e não consegue
   ocupar mais de um núcleo.

2. **O gargalo se desloca, não desaparece.** Em 1 GB o limitante é a memória; em
   2 GB, com o swap eliminado, o limitante passa a ser a CPU, que satura em 99% a
   100% de um núcleo nos dois conjuntos. Provisionar corretamente exige saber qual
   recurso é o próximo limitante, e isso só aparece separando as métricas de
   aplicação das de infraestrutura.

3. **A vazão desta aplicação não responde à demanda.** Ela é limitada pela
   linguagem, não pelo hardware: ~3,5 rps nos níveis leves, em todos os cenários
   e nos dois conjuntos. O que a demanda adicional produz é latência. Um sistema
   com esse perfil deve ser escalado em processos, não em threads.

4. **A pressão de memória depende de um detalhe de implementação.** O laço de
   páginas em Python, que parece desperdício, é o mecanismo que faz as alocações
   concorrentes se sobreporem e, portanto, o que produz o swap. É ele que torna o
   perfil Memory-bound sensível ao provisionamento de RAM.

5. **O achado se reproduz em ambiente independente.** O conjunto B, medido numa VM
   construída a partir da imagem do professor e não preparada por nós, reproduz a
   diferença de swap com magnitude praticamente idêntica (337 MB contra 318 MB) e
   a mesma ausência de swap em 2 GB. Ele também dissolve a anomalia que o conjunto
   A apresentava em C3 nível 2, confirmando-a como ruído de medição. As três
   primeiras conclusões não são, portanto, propriedade de uma máquina específica.

---

## 4. Artefatos e reprodutibilidade

| Caminho | Conteúdo |
| --- | --- |
| `README.md` | enunciado da atividade e decisões do grupo |
| `RELATORIO.md` | este documento — o único relatório do projeto |
| `figuras/` | figuras 1 a 3 e 8 a 13 (conjunto A) |
| [`lcc3/evidencias/`](lcc3/evidencias/) | figuras 4 a 7 e 14 a 17, e os `showvminfo` dos cenários de B |
| [`lcc3/dados/`](lcc3/dados/) | conjunto B: `measurements.csv`, `run_log.csv`, `resources.csv`, `summary.csv`, logs e `graphs/` |
| `atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/` | conjunto A: os mesmos três arquivos brutos, mais `summary.csv` e `graphs/` |
| `atividade2-demanda-desempenho/atividade2/atividade2_demanda/results/ensaio-2026-10-14/` | ensaio preliminar, preservado como dado bruto e não usado nas conclusões |
| `atividade2-demanda-desempenho/evidencias/` | configuração da VM `Po-2026.2` e consoles dos cenários de A |

**Como evitar que um conjunto sobrescreva o outro.** `scripts/plot_results.py`
resolve seus caminhos a partir do diretório de trabalho (`Path("results/...")`).
Por isso cada conjunto foi processado no seu próprio diretório: o conjunto A em
`atividade2_demanda/`, e o conjunto B num diretório de trabalho separado, cujos
resultados foram depois copiados para `lcc3/dados/`. Rodar o script na raiz do
repositório geraria um `results/` novo e não sobrescreveria nenhum dos dois — mas
o resultado não pertenceria a conjunto nenhum, e é por isso que não fazemos isso.

**Reprodutibilidade dos testes funcionais.** Independente dos dados:

```bash
python -m unittest discover -s tests -v
```
