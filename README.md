# Codificação de Fonte — Huffman e LZ78

Implementação completa de dois codificadores sem perdas (Huffman e LZ78) com simulação de canal ruidoso. O projeto cobre o pipeline de codificação, a inserção externa de erros sobre o bitstream codificado, e a decodificação com diagnóstico do impacto dos erros.

## Visão geral

A codificação de fonte busca remover redundância para reduzir o tamanho da mensagem. Este projeto compara dois esquemas clássicos:

- **Huffman** — código prefix-free de comprimento variável; símbolos frequentes recebem códigos curtos
- **LZ78** — dicionário dinâmico, tokens de tamanho fixo

A mensagem comprimida é então submetida a um canal ruidoso. A inserção de erros é feita externamente ao projeto, simulando a degradação real de uma transmissão. Os decoders recebem os arquivos corrompidos e tentam reconstruir o texto original; o resultado evidencia como cada esquema responde ao ruído.

## Estrutura

```
TICF/
├── huffman_lz78_codificacao.ipynb   pipeline completo (PARTE 1 + PARTE 2)
├── texto_original.txt                mensagem-fonte
├── enviar/                           saída da codificação
│   ├── huffman_codificado.bin
│   ├── huffman_codebook.json         codebook + bit_length original
│   └── lz78_codificado.bin
├── recebido/                         arquivos com erros (corrompidos externamente)
│   ├── *.txt                         formato textual usado para edição manual
│   └── *.bin                         reconvertido para decodificação
├── decodificado/                     textos reconstruídos
└── legado/                           arquivos da primeira versão, sem empacotamento
```

## Pipeline de codificação

### Huffman

1. Calcula a frequência absoluta de cada caractere do texto-fonte (`collections.Counter`)
2. Constrói a árvore de Huffman a partir de uma fila de prioridade (`heapq`), unindo iterativamente os dois nós de menor frequência
3. Percorre a árvore para atribuir um código binário a cada folha (esquerda = `0`, direita = `1`)
4. Substitui cada caractere do texto pelo código correspondente, produzindo um bitstream de comprimento variável

A árvore favorece símbolos frequentes com códigos curtos (3 bits para `e`, `o`, espaço) e penaliza símbolos raros (até 8 bits para dígitos e maiúsculas pouco usadas).

### LZ78

1. Percorre o texto símbolo a símbolo construindo um dicionário dinâmico
2. Cada vez que encontra uma sequência nova, emite um token `(índice_da_maior_substring_conhecida, próximo_caractere)`
3. Cada token é serializado como **16 bits de índice + 8 bits de caractere = 24 bits fixos**
4. O dicionário cresce até 65.535 entradas (limite dos 16 bits do campo de índice)

### Empacotamento em bytes

Os bitstreams das duas codificações são, em memória, strings de `'0'` e `'1'`. Para armazenamento real em disco como dado binário, cada 8 bits é empacotado em um byte:

```python
def bits_to_bytes(bitstream):
    pad = (8 - len(bitstream) % 8) % 8
    padded = bitstream + "0" * pad
    return bytes(int(padded[i:i+8], 2) for i in range(0, len(padded), 8))
```

Como o Huffman tem comprimento variável, o último byte pode conter bits de padding sem significado. Para que o decoder saiba onde termina a mensagem útil, o `bit_length` original é salvo junto do codebook em `huffman_codebook.json`. O LZ78, por usar tokens de 24 bits (múltiplo de 8), nunca precisa de padding.

## Inserção de erros

Em uma transmissão real, o canal corromperia bits arbitrários do arquivo binário. Aqui esse papel é desempenhado por uma entidade externa ao projeto, que modifica o conteúdo dos arquivos antes deles serem entregues ao decoder.

### Conversão `.bin` → `.txt` → `.bin`

Editar bits diretamente em um arquivo `.bin` exige um editor hexadecimal e cuidado com a estrutura de bytes. Para facilitar a inserção manual de erros, **o projeto converte cada `.bin` em um `.txt`** contendo a representação textual do bitstream — uma sequência de caracteres `'0'` e `'1'`, um por bit. Nesse formato, flipar um bit é simplesmente trocar um caractere em qualquer editor de texto.

Após a corrupção, **o `.txt` com erros é reconvertido para `.bin`** para que o decoder possa processá-lo no formato binário esperado pelo pipeline. Essa reconversão é feita pela função `txt_para_bin`, que filtra qualquer caractere fora de `{'0', '1'}` (whitespace e quebras de linha são ignorados) e empacota de volta em bytes.

O fluxo completo:

```
enviar/*.bin                       saída da codificação
      ↓ converte para edição
*.txt (string de '0'/'1')          formato textual, 1 char por bit
      ↓ inserção externa de erros
recebido/*_com_erros.txt
      ↓ reconversão para binário
recebido/*_com_erros.bin
      ↓ decodificação
decodificado/*_decodificado.txt    texto reconstruído
```

## Pipeline de decodificação

Os decoders operam sobre os arquivos `.bin` corrompidos, sem nenhuma informação sobre as posições onde os erros ocorreram.

### Huffman

Percorre o bitstream bit a bit, acumulando em um buffer até casar com algum código no codebook invertido. Quando há casamento, emite o caractere correspondente e zera o buffer. Se o stream termina com bits que não fecham nenhum código, o decoder anexa o marcador `[BITS_RESID:N]` ao texto reconstruído.

### LZ78

Lê o bitstream em janelas de 24 bits (16 + 8), reconstruindo o dicionário em paralelo à decodificação. Para cada token:

- `índice == 0` → emite apenas o caractere
- `índice ∈ dicionário` → emite `dicionário[índice] + caractere`
- `índice ∉ dicionário` → emite `[ERR:idx]` + caractere (sinal de corrupção do índice)

Tokens decodificados — mesmo os corrompidos — entram no dicionário do decoder, o que significa que um índice errado pode envenenar referências futuras.

## Como executar

1. Colocar o texto-fonte em `texto_original.txt`
2. Abrir `huffman_lz78_codificacao.ipynb` e rodar as células da PARTE 1
   → gera `enviar/huffman_codificado.bin`, `enviar/huffman_codebook.json`, `enviar/lz78_codificado.bin`
3. Converter os `.bin` em `.txt` para a inserção manual de erros
4. Aplicar a corrupção externa e salvar como `recebido/huffman_com_erros.txt` e `recebido/lz78_com_erros.txt`
5. Rodar a PARTE 2 do notebook
   → reconverte `.txt` em `.bin`, decodifica os arquivos corrompidos, salva os textos reconstruídos em `decodificado/`

## Observações sobre comportamento sob erro

Os decoders não fazem correção de erro. Codificação de fonte e codificação de canal são camadas independentes (princípio da separação fonte-canal de Shannon), e a primeira destrói toda a redundância que a segunda precisaria para detectar ou corrigir corrupções. Os marcadores `[ERR:idx]` e `[BITS_RESID:N]` aparecem apenas em situações em que o estrago é estruturalmente detectável.

O comportamento típico observado:

- **Huffman** dessincroniza após qualquer flip, mas re-sincroniza naturalmente alguns símbolos adiante por causa da propriedade prefix-free. Rajadas curtas tendem a produzir dano localizado; erros dispersos podem corromper grande parte do texto.
- **LZ78** mantém o dano contido ao token onde o erro ocorreu, exceto quando o bit cai no campo de índice. Nesse caso, se o índice corrompido apontar para uma entrada válida, o decoder emite silenciosamente uma substring errada e a corrupção propaga via dicionário para todos os tokens posteriores que a referenciarem.

A robustez efetiva depende mais de onde o erro cai do que da quantidade de bits flipados.
