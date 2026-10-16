# Revisão para a prova prática — Processamento de Imagens

**Escopo:** `aula-01`, `aula-hist`, `exercicio-quantizacoes` e `atividade-pbl`, incluindo o enunciado da atividade. A pasta de seminário foi excluída. Todos os trechos de Python aqui usam somente `cv2`, `numpy` e `matplotlib`, presentes nos notebooks. `cv2_imshow` e `files.upload` aparecem nas aulas quando o ambiente é Google Colab.

**Como estudar comigo:** leia uma seção, resolva o exercício sem consultar o gabarito e envie sua resposta pelo número (por exemplo, `E1: ...`). Eu corrijo, explico o raciocínio e passo ao próximo. O arquivo `GABARITO_REVISAO.md` fica separado para autocorreção posterior. Se tiver pouco tempo, priorize E4, E6, E8, E10, E11 e E14.

## 1. Imagem digital e operações ponto a ponto

Uma imagem em cinza de 8 bits é uma matriz `altura × largura` de inteiros entre 0 (preto) e 255 (branco). Uma imagem colorida costuma ter formato `altura × largura × 3`. Em `cv2.imread`, a ordem dos canais é **B, G, R**, embora a descrição cotidiana diga “RGB”. `img[y, x]` acessa linha `y`, coluna `x`; numa imagem colorida, `img[y, x, 0]` é o azul.

Uma transformação ponto a ponto calcula cada pixel de saída a partir do pixel de entrada correspondente: `saida[y, x] = T(entrada[y, x])`. Negativo, limiarização, gamma, expansão, equalização e quantização aplicam um mapeamento por intensidade. Algumas delas **calculam esse mapeamento com dados da imagem inteira** (mínimo/máximo ou histograma), mas, depois de definido o mapeamento, aplicam-no pixel a pixel. Nenhuma dessas operações usa a posição dos vizinhos para calcular o valor daquele pixel.

Em imagens `uint8`, cuidado com contas intermediárias: multiplicar ou somar sem converter o tipo pode transbordar. Para cálculos contínuos, converta antes para `np.float32`; antes de voltar a `uint8`, limite o intervalo com `np.clip`. A conversão para `uint8` trunca a parte fracionária; se o enunciado exigir arredondamento, use `np.round` explicitamente.

```python
import cv2
import numpy as np
import matplotlib.pyplot as plt

img = cv2.imread('imagem.png', cv2.IMREAD_GRAYSCALE)
if img is None:
    raise FileNotFoundError('imagem.png')
print(img.shape, img.dtype, img.min(), img.max())
```

**E1 — responda.** Uma imagem colorida lida com `cv2.imread` tem `shape == (120, 200, 3)`. Quantas linhas e colunas há? O que representa `img[10, 20, 0]`? Por que `img[10, 20]` sozinho não é uma intensidade de cinza?

**E2 — encontre o erro.** O objetivo é obter uma imagem em cinza a partir de `img_bgr` lida com `cv2.imread`:

```python
cinza = cv2.cvtColor(img_bgr, cv2.COLOR_RGB2GRAY)
```

Indique a constante apropriada e explique por que a original pode mudar os tons.

## 2. Canais, brilho e negativo

`cv2.split(img_bgr)` separa os três planos na ordem `B, G, R`. Cada canal isolado é uma matriz 2D. Para mostrar só o azul em uma imagem colorida, junte azul e dois planos de zeros: `cv2.merge([B, zeros, zeros])`. Para exibir com Matplotlib em cores corretas, converta BGR para RGB; `cv2_imshow` do Colab aceita a imagem BGR diretamente.

Escurecer pela metade pode ser feito com `img.astype(np.float32) / 2`, seguido de `np.clip(...).astype(np.uint8)`. Isso desloca os tons para baixo e reduz a distância entre intensidades. O **negativo** faz `s = 255 - r`: preto vira branco, 50 vira 205. Ele espelha o histograma e troca a polaridade, mas não amplia a diferença entre dois tons. Aplicado duas vezes, recupera a imagem original.

```python
B, G, R = cv2.split(img_bgr)
zeros = np.zeros_like(B)
so_azul = cv2.merge([B, zeros, zeros])
negativo = 255 - img_bgr
```

**E3 — complete e calcule.** Preencha `???` para que `escura` tenha valores inteiros de 0 a 255. Depois diga o resultado do negativo para os pixels `[0, 50, 255]`.

```python
escura = np.clip(img_bgr.astype(np.float32) / 2, 0, 255).astype(???)
```

## 3. Histograma: contar antes de escolher a correção

O histograma `h[r]` conta quantos pixels têm intensidade `r`. Em 8 bits, há 256 posições (`0` a `255`), e `h.sum()` deve ser `img.size`. Ele informa a **distribuição**, não onde os pixels ficam: duas imagens com os mesmos valores rearranjados têm o mesmo histograma. Uma concentração à esquerda sugere imagem escura; à direita, clara; uma faixa estreita sugere baixo contraste global; dois grupos separados podem sugerir um limiar entre fundo e objeto. Confira sempre a imagem, pois a forma do histograma sozinha não garante a interpretação da cena.

No notebook de histogramas, a contagem foi implementada manualmente:

```python
def calclist(img):
    hist = np.zeros(256, dtype=np.int64)
    for pixel in img.ravel():
        hist[pixel] += 1
    return hist
```

Para cores, conte cada canal separadamente. Os três histogramas de B, G e R mostram a distribuição por canal; não são um histograma único de luminosidade. Ao exibir cinza com Matplotlib, fixe `vmin=0, vmax=255` ao comparar antes/depois, para que cada painel use a mesma escala visual.

**E4 — complete e interprete.** Para `img = np.array([[0, 0, 2], [2, 2, 255]], dtype=np.uint8)`, complete as lacunas e informe `h[0]`, `h[2]`, `h[255]` e `h.sum()`.

```python
h = np.zeros(???, dtype=np.int64)
for pixel in img.???:
    h[pixel] += ???
```

**E5 — responda.** Se você embaralhar a posição dos pixels dessa matriz, o histograma muda? O histograma sozinho permite saber se os pixels claros formam um objeto conectado?

## 4. Limiarização

A limiarização produz dois valores: `s = 255` quando `r >= T`, e `s = 0` quando `r < T`. É útil para separar objeto e fundo quando suas intensidades são suficientemente distintas. A escolha de `T` é crítica: um limiar ruim pode apagar objetos ou incluir fundo. A saída só contém 0 e 255, logo perde detalhes internos em cinza. No PBL, a imagem 1 foi usada para separar grãos claros de fundo escuro com `T = 125`.

```python
def limiarizar(img, T):
    return np.where(img >= T, 255, 0).astype(np.uint8)
```

**E6 — calcule.** Aplique `T = 125` a `[0, 124, 125, 220]`. O pixel exatamente igual a `T` vira quanto? Que forma deve ter o histograma final?

**E7 — encontre o erro.** No PBL, o cálculo usa `limiar=125`, mas o título é construído assim:

```python
limiarizada1 = limiarizar(img=imagens[1], limiar=125)
mostrar_comparacao(imagens[1], limiarizada1,
                   'Imagem 1 — original', f'Imagem 1 — limiarizada (T = {T})')
```

O que pode ocorrer ao executar o notebook do início em uma sessão limpa? Corrija o trecho.

## 5. Correção gamma

Com `r` entre 0 e 255, normalize: `u = r/255`. Aplique `s = 255 × u**gamma`. Se `0 < gamma < 1`, os tons intermediários sobem e as sombras clareiam; se `gamma > 1`, os tons intermediários descem e a imagem escurece; `gamma = 1` mantém os valores. Preto e branco continuam nos extremos. Gamma não recupera detalhe já perdido em regiões saturadas e pode tornar ruído das sombras mais visível. No PBL, a imagem 2 foi tratada com `gamma = 0.30`.

```python
def correcao_gama(img, gamma):
    u = img.astype(np.float32) / 255.0
    saida = 255.0 * (u ** gamma)
    return np.clip(saida, 0, 255).astype(np.uint8)
```

**E8 — complete e responda.** Para clarear uma imagem escura, escolha `gamma` menor ou maior que 1? Preencha as lacunas:

```python
u = img.astype(np.float32) / ???
saida = 255.0 * (u ** ???)
```

O que acontece com um pixel 0 e com um pixel 255 quando `gamma = 0.5`?

## 6. Expansão linear de contraste

Se os pixels ocupam só `[r_min, r_max]`, a expansão linear leva o menor valor a 0 e o maior a 255:

`s = (r - r_min) × 255 / (r_max - r_min)`.

A ordem dos tons é preservada e as diferenças são ampliadas por um fator constante. Se todos os pixels forem iguais, o denominador é zero: trate esse caso. Um único pixel extremo pode ditar os limites e reduzir o benefício para a maioria da imagem. No PBL, a imagem 5 tinha intervalo registrado de 75 a 140, motivando essa técnica.

```python
def expandir_contraste(img):
    minimo, maximo = int(img.min()), int(img.max())
    if minimo == maximo:
        return img.copy()
    saida = (img.astype(np.float32) - minimo) * 255.0 / (maximo - minimo)
    return np.clip(saida, 0, 255).astype(np.uint8)
```

**E9 — calcule.** Para uma imagem com mínimo 50 e máximo 150, onde os valores 50, 100 e 150 vão parar? Diga também o que faria se mínimo e máximo fossem ambos 80.

## 7. Equalização global de histograma

A equalização usa o histograma **acumulado**: `CDF[r] = h[0] + ... + h[r]`. Uma tabela `lut[r]` transforma cada tom. A implementação do PBL usa `cdf_min` para que o primeiro nível presente mapeie para 0:

`lut[r] = round((CDF[r] - cdf_min) × 255 / (N - cdf_min))`, limitada a `[0, 255]`, onde `N` é o número de pixels. A tabela é então aplicada como `lut[img]`. Diferentemente da expansão, a curva depende das frequências: tons comuns podem receber mais espaço. O histograma final tende a se espalhar, mas não fica obrigatoriamente plano por causa dos valores discretos e das frequências repetidas. Equalização global pode exagerar ruído. No PBL, a imagem 4 foi equalizada para destacar uma cena com tons concentrados.

```python
def equalizar_manual(img):
    h = calclist(img)
    cdf = np.cumsum(h)
    cdf_min = cdf[cdf > 0][0]
    if img.size == cdf_min:  # imagem de um único tom
        return img.copy()
    lut = (cdf - cdf_min) * 255.0 / (img.size - cdf_min)
    lut = np.clip(np.round(lut), 0, 255).astype(np.uint8)
    return lut[img]
```

**E10 — complete.** Considere uma imagem com quatro pixels `[0, 0, 128, 255]`. Escreva `h[0]`, `h[128]`, `h[255]`, depois `CDF[0]`, `CDF[128]`, `CDF[255]`. Com a fórmula acima, para quais valores aproximados irão 0, 128 e 255?

**E11 — encontre o erro.** A versão original da função no PBL não tem o `if img.size == cdf_min`. O que acontece se todos os pixels têm intensidade 80? Qual é uma resposta defensável para esse caso?

## 8. Quantização e profundidade de bits

Quantizar é reduzir a quantidade de intensidades representáveis. Com `b` bits, há `L = 2**b` níveis: 1 bit → 2, 2 bits → 4, 4 bits → 16, 8 bits → 256. Para níveis igualmente espaçados no intervalo de 8 bits, `delta = 256 // L`. A estratégia usada no PBL escolhe o **início de cada faixa** como representante: `s = (r // delta) × delta`. Assim, com 4 níveis (`delta = 64`), os valores possíveis são 0, 64, 128 e 192. Isso não obriga o último nível a ser 255. Menos bits significam menos detalhes tonais e transições em patamares; a estrutura espacial permanece, mas gradientes podem virar faixas visíveis.

```python
def quantizar(img_cinza, bits):
    niveis = 2 ** bits
    delta = 256 // niveis
    saida = (img_cinza.astype(np.uint16) // delta) * delta
    return saida.astype(np.uint8)
```

Use `bits` de 1 a 8 nessa função. Para comparar imagens com Matplotlib, use sempre `cmap='gray', vmin=0, vmax=255`. O exercício de quantização pedia cinza e 1, 2, 4 e 8 bits; o PBL pedia 16, 8 e 4 níveis após escolher uma das imagens.

**E12 — calcule.** Quantize o pixel 200 com 1, 2, 4 e 8 bits usando a regra do início da faixa acima. Diga quantos níveis existem em cada caso.

**E13 — encontre os erros.** No notebook de quantização há um laço que exibe cada canal e reaproveita o nome `img`; depois, a quantização é calculada assim:

```python
for img in list((B, G, R)):
    cv2_imshow(img)

for bits, niveis in {'1': 2, '2': 4, '4': 16, '8': 256}.items():
    fator = niveis // int(bits)
    resultado = img // fator
```

Indique **três problemas** que impedem a comparação pedida no enunciado e escreva a ideia da correção.

## 9. Diagnóstico e escolha da técnica na prova

Observe a imagem, confirme o padrão no histograma, escolha a operação conforme o objetivo e compare antes/depois. Um bom diagnóstico separa **o problema**, **a transformação**, **o parâmetro** e **a perda possível**. O PBL usa cinco situações: grãos claros sobre fundo escuro → limiarização; imagem muito escura → gamma menor que 1; troca de polaridade → negativo; tons comprimidos na região intermediária → equalização; faixa dinâmica estreita → expansão. A imagem 5 também foi usada para comparar equalização seguida de quantização.

| Sinal observado | Escolha provável | Atenção |
|---|---|---|
| Objeto e fundo separados por intensidade | Limiarização | Perde tons internos; depende de `T` |
| Sombras escuras com detalhes | Gamma menor que 1 | Pode realçar ruído |
| Polaridade dificulta a inspeção | Negativo | Não amplia contraste |
| Tons concentrados e distribuição desigual | Equalização | Pode exagerar textura/ruído |
| Mínimo e máximo próximos | Expansão linear | Extremos isolados podem dominar |
| Precisa reduzir profundidade tonal | Quantização | Gera patamares e perda de detalhes |

**E14 — caso prático.** Uma imagem em cinza tem mínimo 75, máximo 140 e desvio padrão 10. Você precisa evidenciar a geometria sem descartar tons. Qual transformação escolheria primeiro? Justifique em duas frases e diga uma limitação.

**E15 — caso prático.** Há dois grupos bem separados no histograma, um de fundo escuro e outro de objetos claros. O objetivo é **contar objetos**. Qual transformação vem primeiro, como escolheria o parâmetro e que informação a saída deixa de guardar?

## 10. Roteiro rápido de execução

1. Leia a imagem; confira `img is not None`, `shape`, `dtype`, mínimo e máximo. No PBL, os arquivos estão em `atividade-pbl/imagens`; se o notebook estiver rodando de dentro de `atividade-pbl`, o caminho é `imagens/imagem_01.png`.
2. Se o trabalho for em tons de cinza, carregue com `cv2.IMREAD_GRAYSCALE` ou converta uma imagem BGR com `cv2.COLOR_BGR2GRAY`.
3. Conte o histograma e verifique se `hist.sum() == img.size`. Observe a imagem e defina o objetivo.
4. Aplique a transformação com cuidado com `dtype`, divisão por zero e faixa `[0, 255]`.
5. Compare imagem e histograma antes/depois. Anote em poucas linhas por que escolheu a operação, qual parâmetro usou e o que pode ter sido perdido.

**E16 — revisão final.** Explique, em uma frase cada: (a) por que `h.sum()` deve ser `img.size`; (b) por que `cv2.imread('imagem_01.png')` pode falhar no notebook do PBL mesmo que o arquivo exista; (c) por que dois painéis `plt.imshow` sem `vmin`/`vmax` podem dificultar a comparação de brilho.
