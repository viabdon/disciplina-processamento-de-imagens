# Gabarito — Revisão para a prova prática

Use este arquivo depois de tentar os exercícios em `REVISAO_PROVA_PRATICA.md`. Há mais de uma justificativa aceitável quando a decisão depende da finalidade da imagem.

## E1

Há 120 linhas (altura) e 200 colunas (largura). `img[10, 20, 0]` é o valor do **canal azul** do pixel na linha 10, coluna 20, porque OpenCV lê BGR. `img[10, 20]` contém três componentes, não um único nível de cinza.

## E2

```python
cinza = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2GRAY)
```

`COLOR_RGB2GRAY` interpreta o primeiro componente como vermelho, mas `cv2.imread` colocou ali o azul. A ponderação dos canais na conversão pode, portanto, produzir cinzas diferentes.

## E3

```python
escura = np.clip(img_bgr.astype(np.float32) / 2, 0, 255).astype(np.uint8)
```

O negativo de `[0, 50, 255]` é `[255, 205, 0]`. A conversão para `uint8` explicita a saída de 8 bits; `astype` trunca valores fracionários.

## E4

```python
h = np.zeros(256, dtype=np.int64)
for pixel in img.ravel():
    h[pixel] += 1
```

`h[0] = 2`, `h[2] = 3`, `h[255] = 1`; `h.sum() = 6`.

## E5

Não muda: o histograma conta valores, independentemente de suas posições. Ele não revela se os pixels claros estão conectados; para isso seria preciso informação espacial.

## E6

`[0, 0, 255, 255]`. Como o teste é `>=`, 125 vira 255. O histograma final tem contagens somente em 0 e 255 (neste exemplo, duas em cada posição).

## E7

`T` não foi definido nesse trecho. Em uma execução limpa, a expressão do título pode causar `NameError`, embora o cálculo da imagem já tenha sido feito. Uma correção é:

```python
T = 125
limiarizada1 = limiarizar(img=imagens[1], limiar=T)
mostrar_comparacao(imagens[1], limiarizada1,
                   'Imagem 1 — original', f'Imagem 1 — limiarizada (T = {T})')
```

## E8

Escolha `0 < gamma < 1`, por exemplo `0.5`.

```python
u = img.astype(np.float32) / 255.0
saida = 255.0 * (u ** gamma)
```

Para `gamma = 0.5`, 0 continua 0 e 255 continua 255. Os níveis intermediários aumentam.

## E9

Pela fórmula, `50 → 0`, `100 → 127,5` antes de converter para inteiro, `150 → 255`. Com o `astype(np.uint8)` mostrado na revisão, 100 vira 127 por truncamento. Se mínimo e máximo forem 80, há divisão por zero; a função da revisão retorna uma cópia da imagem sem alteração.

## E10

`h[0] = 2`, `h[128] = 1`, `h[255] = 1`. A CDF nos níveis pedidos é 2, 3 e 4. `cdf_min = 2` e `N = 4`; logo `0 → 0`, `128 → round(127,5) ≈ 128` e `255 → 255`. O arredondamento segue a regra de `np.round`; neste caso, 127,5 vai para 128.

## E11

Com todos os pixels iguais a 80, `cdf_min == total`, então `total - cdf_min` é zero. A expressão da LUT divide por zero; o resultado pode gerar avisos e valores inválidos na conversão, em vez de uma equalização útil. Uma solução defensável é detectar a imagem constante e devolver `img.copy()`. Não há contraste tonal a redistribuir.

## E12

| Bits | Níveis | `delta` | Resultado de 200 |
|---:|---:|---:|---:|
| 1 | 2 | 128 | 128 |
| 2 | 4 | 64 | 192 |
| 4 | 16 | 16 | 192 |
| 8 | 256 | 1 | 200 |

A regra escolhe o início da faixa; por isso 255 não precisa virar 255 quando há menos de 256 níveis.

## E13

Três problemas centrais:

1. O laço `for img in (B, G, R)` sobrescreve a variável `img` original; ao terminar, `img` é apenas o canal R, e a quantização não parte da imagem em tons de cinza pedida.
2. O passo está errado: deve depender do intervalo de 256 valores, `delta = 256 // niveis`, não de `niveis // bits`.
3. `img // fator` só calcula o índice da faixa e não o valor representativo na escala de 0 a 255; falta multiplicar por `delta`. A comparação final deve usar a imagem cinza e as quatro versões, identificando os bits.

Uma correção compacta é:

```python
img_cinza = cv2.imread('imagem.png', cv2.IMREAD_GRAYSCALE)
if img_cinza is None:
    raise FileNotFoundError('imagem.png')

quantizacoes = {}
for bits in (1, 2, 4, 8):
    niveis = 2 ** bits
    delta = 256 // niveis
    quantizacoes[bits] = ((img_cinza.astype(np.uint16) // delta) * delta).astype(np.uint8)
```

## E14

Escolheria **expansão linear de contraste**: o intervalo `[75, 140]` ocupa uma parte estreita de `[0, 255]`, e a expansão leva os extremos a 0 e 255, ampliando proporcionalmente as diferenças. Ela preserva a ordem tonal sem binarizar. Limitação: um valor extremo isolado pode dominar o intervalo e também se pode acentuar ruído.

## E15

Começaria com **limiarização**. Escolheria `T` na região de baixa frequência entre os dois grupos e verificaria visualmente o resultado; objeto claro seria 255 e fundo escuro, 0. A saída perde texturas e variações de cinza internas dos objetos. Contá-los de fato exigiria uma etapa posterior de análise espacial, que não faz parte dessas transformações ponto a ponto.

## E16

(a) Cada pixel contribui exatamente uma contagem para o histograma, então a soma das contagens é o número de pixels. (b) `cv2.imread` usa o diretório de execução atual: no PBL, as imagens estão em `imagens/`, logo só o nome do arquivo pode apontar para o lugar errado. (c) Matplotlib pode ajustar automaticamente a escala de cada painel, fazendo intensidades diferentes parecerem visualmente semelhantes; `vmin=0, vmax=255` fixa uma referência comum.

## Pontos para memorizar antes de entrar na prova

`cv2.imread` → BGR; `cv2.COLOR_BGR2GRAY` para cinza. `h.sum() == img.size`. Limiar `>= T` inclui a igualdade. Gamma menor que 1 clareia. Negativo `255 - r`. Expansão usa mínimo/máximo e trata a imagem constante. Equalização usa CDF e precisa tratar `N == cdf_min`. Quantização usa `2**bits` níveis e `delta = 256 // niveis`. Confira caminhos dos arquivos, `dtype` e comparação antes/depois.
