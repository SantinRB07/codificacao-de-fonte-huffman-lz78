# Codificação de Fonte — Huffman e LZ78

Grupo 04. Projeto da disciplina de TICF.

Dois codificadores sem perdas aplicados ao mesmo texto. Os arquivos binários gerados vão para o professor, que insere erros simulando um canal ruidoso. Depois decodificamos os arquivos corrompidos para medir o impacto dos erros em cada esquema.

## Estrutura

- `texto_original.txt` — texto-fonte
- `huffman_lz78_codificacao.ipynb` — pipeline completo
- `enviar/` — saída do encoding, vai para o professor
  - `huffman_codificado.bin`, `huffman_codebook.json`, `lz78_codificado.bin`
- `recebido/` — arquivos com erros devolvidos pelo professor (`.txt` ou `.bin`)
- `decodificado/` — texto reconstruído após a decodificação
- `legado/` — `.txt` antigos da primeira versão, antes do empacotamento em bytes

## Como usar

1. Colocar o texto-fonte em `texto_original.txt`
2. Rodar as células da PARTE 1 do notebook
3. Enviar os arquivos de `enviar/` ao professor
4. Salvar a resposta com erros em `recebido/` (qualquer nome, `.txt` ou `.bin`)
5. Rodar a PARTE 2 — o resultado decodificado aparece em `decodificado/`

Se o professor devolver em formato `.txt` (string de `'0'`/`'1'`), a primeira célula da PARTE 2 detecta e converte para `.bin` automaticamente.

## Codificações

**Huffman.** Comprimento variável. Cada caractere recebe um código binário cujo tamanho é inversamente proporcional à sua frequência. O bitstream é empacotado em bytes; o último byte pode ter padding, então o `bit_length` original fica salvo em `huffman_codebook.json`.

**LZ78.** Dicionário dinâmico. Cada token tem 24 bits (16 de índice + 8 de caractere), sempre múltiplo de 8 — sem padding. Dicionário cresce até 65.535 entradas.

Para textos curtos o LZ78 tende a expandir o arquivo (o overhead de 16 bits por token não é amortizado pela repetição); para textos longos compensa.

## Comportamento sob erro de canal

Os decoders não fazem correção, só decodificam cegamente o que receberem. Dois marcadores tornam o estrago auditável no texto reconstruído:

- `[ERR:idx]` — LZ78 recebeu um índice fora do dicionário
- `[BITS_RESID:N]` — sobraram `N` bits que não fecham um código (Huffman) ou um token (LZ78)

Huffman tende a propagar erros catastroficamente: um flip dessincroniza toda a leitura seguinte. LZ78 contém o erro no token onde ele caiu, mas se o bit atingido for de índice e apontar para outra entrada válida, propaga parcialmente via dicionário.
