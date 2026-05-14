# Relatório — Codificação de Fonte com Huffman e LZ78

Grupo 04 — TICF.

## 1. Resumo

Este relatório descreve a implementação e avaliação de dois codificadores de fonte sem perdas — Huffman e LZ78 — aplicados sobre o mesmo texto, e analisa o comportamento dos respectivos decoders quando os bitstreams codificados são submetidos a inserção externa de erros. O objetivo é caracterizar quantitativamente, sobre um exemplo concreto, o desempenho de cada esquema em compressão e em robustez a ruído de canal.

## 2. Fundamentação teórica

A teoria da informação de Shannon (1948) estabelece um limite inferior para o número médio de bits necessários para representar uma fonte discreta. Esse limite é a **entropia** da fonte:

$$H(X) = -\sum_{i} p_i \log_2 p_i$$

onde p_i é a probabilidade do símbolo i. Nenhum código sem perdas pode atingir comprimento médio inferior a H(X). Codificadores de fonte buscam comprimentos próximos desse limite.

Shannon também estabeleceu o princípio de **separação fonte-canal**: a compressão (remover redundância) e a proteção contra erros (adicionar redundância calibrada para o canal) podem ser tratadas como duas camadas independentes, sem perda de otimalidade assintótica. Este projeto cobre apenas a primeira camada.

## 3. Texto-fonte

```
Em 1948, Claude Shannon publicou um artigo que media a surpresa em bits.
Antes dele, a informacao era vaga; depois dele, virou grandeza. Provou que
toda mensagem tem um limite minimo de compressao, e que abaixo dele nada
sobrevive ao ruido.
```

- **Comprimento:** 241 caracteres
- **Alfabeto:** 36 símbolos distintos (letras minúsculas e maiúsculas, dígitos, pontuação, espaço, nova linha)
- **Codificação fixa de referência:** ASCII de 8 bits → 1.928 bits totais

A distribuição é dominada por espaço (`' '`), `e` e `o`, conforme esperado para texto em português. Símbolos raros incluem dígitos (`1948`), pontuação atípica (`;`) e letras maiúsculas (`E`, `C`, `S`, `A`, `P`), cada uma com frequência ≤ 0,01.

### 3.1 Entropia da fonte

A entropia empírica do texto, calculada a partir das frequências observadas, é:

$$H(X) = 4{,}3064 \text{ bits/símbolo}$$

Esse é o limite inferior teórico para qualquer código sem perdas operando símbolo a símbolo nesta fonte.

## 4. Codificação Huffman

### 4.1 Algoritmo

A construção da árvore segue o algoritmo clássico de Huffman:

1. Para cada símbolo distinto da fonte, criar um nó-folha com peso igual à sua frequência absoluta.
2. Inserir todos os nós-folha em uma fila de prioridade mínima (`heapq`).
3. Enquanto a fila tiver mais de um elemento: extrair os dois nós de menor peso, criar um nó interno cujo peso é a soma dos dois, e inseri-lo na fila.
4. O nó remanescente é a raiz da árvore.

A geração do codebook é uma travessia da árvore associando `'0'` à aresta esquerda e `'1'` à aresta direita; o código de um símbolo é a sequência de bits do caminho raiz → folha.

### 4.2 Codebook obtido

O codebook tem 36 entradas com comprimentos variando de 3 a 8 bits:

| Comprimento | Símbolos                                        |
|-------------|-------------------------------------------------|
| 3 bits      | `e`, espaço                                     |
| 4 bits      | `r`, `d`, `u`, `m`, `i`, `o`, `a`               |
| 5 bits      | `t`, `l`, `n`, `s`                              |
| 6 bits      | `q`, `c`, `.`, `,`, `p`, `g`, `b`, `v`          |
| 7 bits      | `z`                                             |
| 8 bits      | `f`, `S`, `9`, `\n`, `h`, `;`, `x`, `P`, `A`, `4`, `8`, `E`, `C`, `1` |

O comprimento médio de código é:

$$L_{\text{Huffman}} = \sum_i p_i \cdot |c_i| = 4{,}3444 \text{ bits/símbolo}$$

A **eficiência** do código em relação ao limite de Shannon é:

$$\eta = \frac{H(X)}{L} = \frac{4{,}3064}{4{,}3444} = 99{,}12\%$$

O código gerado está a menos de 1% do limite teórico — para esta fonte específica, Huffman é praticamente ótimo.

### 4.3 Bitstream produzido

Total de bits: 241 × 4,3444 = **1.047 bits**, o que corresponde a 130,875 bytes. Empacotando em bytes (com 1 bit de padding no último byte), obtemos **131 bytes**. Razão de compressão: 131 / 241 = **54,4%**.

## 5. Codificação LZ78

### 5.1 Algoritmo

LZ78 mantém um dicionário dinâmico de substrings já vistas, indexadas a partir de 1 (o índice 0 representa "sem prefixo"). O algoritmo de codificação:

1. Inicializar dicionário vazio e buffer `current = ""`.
2. Para cada caractere `ch` do texto:
   - Concatenar: `current = current + ch`.
   - Se `current` não está no dicionário:
     - Emitir token `(índice de current[:-1], ch)`, sendo o índice 0 se o prefixo for vazio.
     - Adicionar `current` ao dicionário com o próximo índice disponível.
     - Resetar `current = ""`.
3. Ao final, se `current` não estiver vazio, emitir um último token com o prefixo conhecido + caractere remanescente.

### 5.2 Estrutura do token e serialização

Cada token tem dois campos com largura fixa:

- **Índice** — 16 bits (suporta até 65.535 entradas no dicionário)
- **Caractere** — 8 bits (código ASCII/Latin-1)

Total por token: **24 bits**. A serialização é direta, em big-endian, concatenando os campos de todos os tokens em ordem.

### 5.3 Resultados

| Métrica                              | Valor       |
|--------------------------------------|-------------|
| Tokens gerados                       | 122         |
| Entradas no dicionário               | 122         |
| Tamanho médio das entradas           | 1,98 chars  |
| Maior entrada no dicionário          | 4 chars     |
| Bits totais                          | 2.928       |
| Bytes (sem padding, 24 = 3×8)        | 366         |

A razão de compressão é 366 / 241 = **151,9%** — o arquivo *expandiu* em ~52%. Esse resultado é consistente com o tamanho curto do texto: o dicionário não tem tempo de acumular entradas longas (a maior tem apenas 4 caracteres), então o overhead de 16 bits por token não é amortizado por referências a substrings longas.

## 6. Empacotamento em bytes

Os bitstreams produzidos pelos dois codificadores são, em memória, strings de `'0'` e `'1'`. Para armazenamento em disco como dado binário real, cada 8 bits é empacotado em um byte pela função:

```python
def bits_to_bytes(bitstream):
    pad = (8 - len(bitstream) % 8) % 8
    padded = bitstream + "0" * pad
    return bytes(int(padded[i:i+8], 2) for i in range(0, len(padded), 8))
```

O Huffman, por ter comprimento variável, normalmente termina com bits de padding sem significado no último byte. Para que o decoder saiba onde a mensagem útil termina, o valor de `bit_length` (1.047 neste caso) é salvo junto do codebook em `huffman_codebook.json`. O LZ78, por usar tokens de 24 bits — múltiplo de 8 — nunca requer padding.

## 7. Workflow do experimento

A inserção de erros é realizada externamente ao código do projeto, simulando a corrupção de bits durante a passagem por um canal ruidoso. Para facilitar essa operação, o projeto adota a seguinte cadeia de formatos:

```
enviar/*.bin                       saída da codificação (formato final)
       ↓ (conversão para edição)
*.txt (string de '0' e '1')        formato textual, 1 caractere por bit
       ↓ (inserção externa de erros)
recebido/*_com_erros.txt
       ↓ (reconversão para binário)
recebido/*_com_erros.bin
       ↓ (decodificação)
decodificado/*_decodificado.txt    texto reconstruído (com marcadores diagnósticos)
```

A razão para o detour pelo `.txt`: editar bits em um arquivo binário exige um editor hexadecimal e atenção à estrutura de bytes. Editar uma sequência de `'0'` e `'1'` em um arquivo de texto é trivial em qualquer editor de texto, basta trocar caracteres. Após a corrupção, a célula `txt_para_bin` filtra qualquer caractere fora de `{'0', '1'}` (ignorando whitespace introduzido pela ferramenta de edição) e reempacota os bits em bytes para que o decoder possa processar.

## 8. Modelo de inserção de erros

O agente externo aplicou o seguinte protocolo para cada arquivo:

- **Huffman:** 15 trocas aleatórias de bits a partir da posição 500 do bitstream (aproximadamente a metade do arquivo, que tem 1.047 bits)
- **LZ78:** 15 trocas aleatórias de bits a partir da posição 1500 do bitstream (aproximadamente a metade do arquivo, que tem 2.928 bits)

A medição posterior (XOR entre o bitstream original e o corrompido) revelou:

| Esquema | Bits totais | Trocas aplicadas | Bits efetivamente diferentes |
|---------|-------------|------------------|------------------------------|
| Huffman | 1.047       | 15               | 10                           |
| LZ78    | 2.928       | 15               | 9                            |

A diferença entre trocas aplicadas e bits efetivamente diferentes é compatível com seleção aleatória **com possibilidade de repetição em uma janela curta**: quando o mesmo bit é selecionado mais de uma vez, as flips sucessivas se cancelam (`x ⊕ 1 ⊕ 1 = x`). As posições corrompidas observadas foram, de fato, concentradas em janelas estreitas:

- Huffman: posições 499–513 (janela de 15 posições, 10 efetivamente diferentes)
- LZ78: posições 1499–1512 (janela de 14 posições, 9 efetivamente diferentes)

Esse padrão de erro — **rajada concentrada** — é estrutural e importante para a análise, porque produz efeitos qualitativamente distintos dos esperados em flips bem dispersos.

## 9. Decodificação

### 9.1 Huffman

O decoder mantém um buffer que acumula bits até casar com algum código no codebook invertido. Quando há casamento, o caractere correspondente é emitido e o buffer é zerado. Se ao final do stream sobrarem bits que não formam nenhum código completo, é anexado o marcador `[BITS_RESID:N]` ao texto, onde N é o número de bits residuais.

Como os códigos de Huffman são prefix-free e a árvore é completa (todo nó interno tem dois filhos), qualquer sequência de bits — corrompida ou não — eventualmente casa com alguma folha. O decoder nunca trava; sempre produz saída.

### 9.2 LZ78

O decoder lê o bitstream em blocos de 24 bits e reconstrói o dicionário em paralelo à decodificação. Para cada token `(idx, ch)`:

- `idx == 0` → emite apenas `ch`
- `idx` válido (presente no dicionário) → emite `dicionário[idx] + ch`
- `idx` inválido (fora do dicionário) → emite o marcador `[ERR:idx]` seguido de `ch`

Cada token decodificado entra no dicionário do decoder com o próximo índice disponível. **Tokens corrompidos também entram no dicionário** — isto é, o decoder não tem como saber que a entrada está errada, e a corrupção pode propagar para referências futuras.

### 9.3 Marcadores diagnósticos

Dois marcadores são inseridos no texto reconstruído quando o estrago é estruturalmente detectável:

- `[ERR:idx]` — LZ78 recebeu um índice fora do dicionário
- `[BITS_RESID:N]` — sobraram N bits sem fechar um código (Huffman) ou um token (LZ78)

Estes marcadores não corrigem nada; apenas tornam o erro visível para análise.

## 10. Resultados

### 10.1 Tamanhos finais

| Arquivo                              | Tamanho      | Razão  |
|--------------------------------------|--------------|--------|
| `texto_original.txt`                 | 241 bytes    | 100,0% |
| `enviar/huffman_codificado.bin`      | 131 bytes    | 54,4%  |
| `enviar/lz78_codificado.bin`         | 366 bytes    | 151,9% |

### 10.2 Texto decodificado

**Huffman:**

> Em 1948, Claude Shannon publicou um artigo que media a surpresa em bits. Antes dele, a informacao era vaga; **dei.at** dele, virou grandeza. Provou que toda mensagem tem um limite minimo de compressao, e que abaixo dele nada sobrevive ao ruido.

A palavra `depois` foi substituída por `dei.at` — 4 caracteres alterados em posições contíguas (110–113), dano localizado em uma única região. O resto da mensagem foi reconstruído sem erro, com tamanho idêntico ao original (241 caracteres).

**LZ78:**

> Em 1948, Claude Shannon publicou um artigo que media a surpresa em bits. Antes dele, a informacao era v**p^[ERR:32779]**; depois dele, virou grandeza. Provou que toda mens**p^**em tem um limite minimo de compressao, e que abaixo dele nada sobrevive ao ruido.

O texto decodificado tem 251 caracteres (10 a mais que o original) por causa da inserção do marcador `[ERR:32779]`. Há **duas regiões corrompidas**, separadas por um bloco intacto de 51 caracteres no meio:

- **1ª corrupção (no ponto do erro):** `aga` virou `p^[ERR:32779]`. Aqui o token #63 teve seu índice flipado para 32.779 (fora do dicionário de 62 entradas àquela altura), e o decoder marcou explicitamente.
- **2ª corrupção (envenenamento de dicionário):** `ag` em "mensagem" virou `p^`. Esse trecho viria de uma entrada do dicionário que, na decodificação, ficou armazenada com a versão corrompida — a corrupção do token #63 propagou para uma referência posterior.

### 10.3 Métricas de impacto

| Esquema | Bits flipados | Chars do original modificados | Chars preservados | Caracteres adicionais |
|---------|---------------|-------------------------------|-------------------|------------------------|
| Huffman | 10            | 4 / 241 (1,7%)                | 237 / 241 (98,3%) | 0                      |
| LZ78    | 9             | 5 / 241 (2,1%)                | 236 / 241 (97,9%) | +10 (marcador `[ERR:idx]`) |

A métrica usada aqui é o número de chars do texto original que foram efetivamente modificados, alinhando o original e o decodificado por prefixo/sufixo comuns. Comparações posição a posição superestimam o dano quando há inserção (como o marcador do LZ78), porque todo o sufixo fica deslocado e contado como "diferente" mesmo sendo idêntico.

A integridade efetiva dos dois esquemas neste experimento foi **bastante próxima**: ambos preservaram acima de 97,9% do texto original. A diferença qualitativa é outra: o Huffman concentrou todo o seu dano em uma região contígua, enquanto o LZ78 espalhou em duas regiões separadas, evidenciando o efeito de envenenamento de dicionário (a segunda corrupção está a ~54 caracteres da primeira).

## 11. Análise técnica

Ambos os esquemas preservaram a quase totalidade do texto sob a rajada de erros aplicada — 98,3% no Huffman, 97,9% no LZ78. Apesar dos números próximos, o **padrão estrutural** do dano em cada esquema foi diferente, e essa diferença é a parte mais informativa do experimento.

### 11.1 Re-sincronização em códigos prefix-free

Códigos de Huffman têm a propriedade de **re-sincronização espontânea**: dado um bitstream arbitrário, o decoder eventualmente cai em uma folha da árvore e reinicia a leitura. Mesmo após um deslocamento de fase causado por bit flips, é estatisticamente comum que o decoder volte a alinhar-se com o bitstream original alguns símbolos adiante. Para esta fonte, isso aconteceu rapidamente: a rajada de 10 bits errados perturbou a decodificação de exatamente quatro caracteres consecutivos (a palavra `depois` virou `dei.at`), e o decoder retomou o alinhamento correto a partir do próximo símbolo. Não houve nenhum efeito além dessa janela.

Esta resiliência depende do **padrão temporal do erro**. Bit flips esparsos forçariam dessincronizações repetidas, e o decoder gastaria mais bits para re-sincronizar a cada flip, potencialmente corrompendo regiões maiores do texto. Rajadas concentradas são tratadas com mais facilidade, porque cada flip extra dentro da mesma janela tende a desperdiçar bits que já estavam corrompidos.

### 11.2 Envenenamento do dicionário em LZ78

LZ78 tem uma estrutura de tokens com largura fixa, o que em princípio limita o dano de um bit flip ao token onde ele cai. Esse é o argumento usual a favor de sua robustez. No entanto, **se o erro atingir o campo de índice (16 bits) e o índice resultante for diferente do original**, dois casos se apresentam:

1. O novo índice está fora do dicionário (acima do número de entradas atuais). O decoder detecta a anomalia e marca `[ERR:idx]`. Esse foi o caso do token #63 neste experimento: o índice corrompido tornou-se 32.779, muito acima das 62 entradas existentes àquela altura.
2. O novo índice está dentro do dicionário, mas referencia uma entrada diferente da original. O decoder emite silenciosamente uma substring errada, sem nenhuma indicação.

Em ambos os casos, há um efeito secundário: a entrada corrompida é inserida no dicionário do decoder com o próximo índice disponível. Qualquer token posterior que referencie essa entrada propagará a corrupção. Foi o que aconteceu com `mensagem → mensp^em`: o sufixo `ag` viria de uma entrada do dicionário envenenada lá atrás, e o decoder a copiou como `p^`.

A propagação, neste experimento, foi limitada — apenas 2 caracteres adicionais foram corrompidos por causa do envenenamento. Mas o fenômeno é qualitativamente diferente do Huffman: enquanto o dano no Huffman fica geograficamente contido na vizinhança do bit flipado, no LZ78 ele pode **reaparecer em pontos distantes** do texto, dependendo de quais entradas envenenadas são referenciadas pelos tokens subsequentes.

### 11.3 Comparação dos padrões de dano

| Aspecto                         | Huffman                       | LZ78                                       |
|----------------------------------|--------------------------------|---------------------------------------------|
| Chars do original modificados   | 4                              | 5 (3 no ponto + 2 por propagação)           |
| Localização do dano             | 1 região contígua              | 2 regiões separadas por 54 chars intactos   |
| Mecanismo de propagação         | Nenhum (re-sincroniza)         | Envenenamento do dicionário                  |
| Detectabilidade do erro         | Indireta (texto sem sentido)   | Direta (marcador `[ERR:idx]` no 1º ponto)   |

Não há um "vencedor" claro neste experimento — ambos os esquemas se mostraram resilientes a uma rajada concentrada de erros na metade do bitstream. O que o experimento mostra é que **a estrutura do dano reflete a estrutura do codificador**: Huffman produz dano local e visualmente identificável (palavra sem sentido); LZ78 produz dano potencialmente disperso, com pontos de corrupção remota via dicionário, mas com a vantagem de oferecer um sinal de detecção quando o índice corrompido cai fora do dicionário.

## 12. Conclusões

1. **Eficiência de compressão.** Huffman atingiu 99,12% do limite de Shannon para esta fonte, comprimindo o arquivo em 45,6% do tamanho original. LZ78 expandiu o arquivo em 52% — comportamento esperado para textos curtos, onde o overhead de 16 bits por token não é compensado pela construção lenta do dicionário. Um experimento complementar com texto longo e redundante (Seção 13) confirma essa dependência: nele, LZ78 passa a comprimir (razão 85,7%), embora Huffman continue mais eficiente (~52%).

2. **Robustez a erros.** Ambos os esquemas preservaram acima de 97,9% do texto original sob a rajada de erros aplicada (Huffman 98,3%, LZ78 97,9%). A diferença é qualitativa, não quantitativa: o Huffman concentrou todo o seu dano em uma única região (4 chars contíguos), enquanto o LZ78 produziu duas regiões separadas (3 chars no ponto do erro + 2 chars depois, por envenenamento do dicionário). Em padrões de erro diferentes — bit flips dispersos, ou flips no campo de caractere do LZ78 — os resultados poderiam ser bem distintos.

3. **Implicação prática.** Codificação de fonte, isoladamente, não é robusta a canal ruidoso. Sistemas reais combinam codificação de fonte com codificação de canal (Hamming, Reed-Solomon, LDPC, etc.) que adiciona redundância calibrada para detecção e correção de erros. É o princípio da separação fonte-canal de Shannon: comprimir e proteger são duas camadas independentes, cada uma com seu próprio papel.

## 13. Experimento complementar — texto longo e redundante

O experimento principal foi conduzido sobre um texto de 241 caracteres, no qual o LZ78 expandiu o arquivo em 52%. Esse resultado é consequência direta do tamanho insuficiente para amortizar o overhead de 16 bits do campo de índice. Para avaliar o LZ78 em um regime onde o dicionário tenha espaço para crescer, repetimos a etapa de codificação sobre um texto significativamente maior e com forte redundância semântica.

### 13.1 Texto-fonte

O novo texto trata do princípio da separação fonte-canal de Shannon, com repetições deliberadas de termos como `codificacao de fonte`, `codificacao de canal`, `teoria da informacao`, `Claude Shannon`, `redundancia`, `ruido`, `canal`, `mensagem`.

| Métrica | Valor |
|---|---|
| Caracteres | 3.761 (15,6× o texto do experimento principal) |
| Símbolos distintos | 49 |
| Entropia H(X) | 4,1035 bits/símbolo |

### 13.2 Compressão Huffman

| Métrica | Valor |
|---|---|
| Comprimento médio do código | 4,1425 bits/símbolo |
| Eficiência (H/L) | 99,06% |
| Bits úteis | 15.580 |
| Bytes | 1.948 |
| Razão de compressão | **51,8%** |
| Comprimentos de código | 3 a 12 bits |

O comportamento é praticamente idêntico ao texto curto: razão ~52%, eficiência ~99% em relação ao limite de Shannon. Esperado: Huffman opera símbolo a símbolo, e o desempenho depende apenas da distribuição de frequências, não do tamanho do texto.

### 13.3 Compressão LZ78

| Métrica | Valor |
|---|---|
| Tokens gerados | 1.075 |
| Entradas no dicionário | 1.074 |
| Tamanho médio das entradas | 3,50 chars |
| Maior entrada | 10 chars |
| Bits totais | 25.800 |
| Bytes | 3.225 |
| Razão de compressão | **85,7%** |

Desta vez o LZ78 **comprime**. O motivo está na composição do dicionário: as entradas têm em média 3,5 caracteres (vs 1,98 no texto curto) e a maior tem 10 caracteres. As entradas mais longas capturam fragmentos repetidos do texto:

```
(10) 'odificacao'      radical de 'codificacao'
(10) 'icacao de '      sufixo compartilhado por codificacao de fonte/canal
 (9) 'acao de c'       variante
 (9) ' A codifi'       inicio de frases repetidas
 (9) 'cacao de '       variante
```

Cada referência a uma dessas entradas substitui 3-10 caracteres (24-80 bits no ASCII original) por um único token de 24 bits, gerando compressão real.

### 13.4 Comparação direta

| Texto                | Chars | Huffman | LZ78 |
|----------------------|-------|---------|------|
| Curto (Shannon)      | 241   | 54,4%   | 151,9% (expandiu) |
| Longo (redundante)   | 3.761 | 51,8%   | **85,7%** (comprime) |

O Huffman é praticamente insensível ao tamanho do texto — sua eficiência depende só da distribuição estatística dos símbolos. O LZ78, ao contrário, é fortemente dependente do volume e da redundância: precisa de tempo para o dicionário acumular entradas longas que justifiquem o overhead de 16 bits do índice. Esta é a razão pela qual LZ78 e seus descendentes (LZW, gzip, deflate, zstd) brilham em arquivos grandes e penalizam arquivos pequenos.

## 14. Arquivos

- [`huffman_lz78_codificacao.ipynb`](huffman_lz78_codificacao.ipynb) — implementação completa
- [`texto_original.txt`](texto_original.txt) — texto-fonte
- [`enviar/`](enviar/) — bitstreams codificados
- [`recebido/`](recebido/) — bitstreams com erros inseridos externamente
- [`decodificado/`](decodificado/) — textos reconstruídos
- [`README.md`](README.md) — documentação técnica do projeto
