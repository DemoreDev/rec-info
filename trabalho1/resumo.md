# Trabalho Prático 1 — Recuperação da Informação

Modelo Vetorial e BM25 sobre a coleção **Cranfield**, implementados em Python puro
(bibliotecas apenas para pré-processamento e gráficos). Notebook: `main.ipynb`.

**Coleção:** 1400 documentos, 225 consultas, 1837 julgamentos. Graus `-1` e `1..4`,
onde **1 = *complete answer*** e 4 = *minimum interest* (a escala é decrescente). Relevante = grau ≥ 1;
`-1` e não julgados = não relevantes. Todas as consultas têm ≥ 1 relevante.

**Decisões globais:** indexa apenas o campo `text` (já contém o título em 1399/1400
documentos; `author` e `bib` são metadados). Tokenização por regex `[a-z0-9]+` com
minúsculas. Stopwords **antes** do stemming — o Porter transforma `does`→`doe`, que
escaparia da lista se a ordem fosse invertida.

---

## R1 · Pré-processamento

| config | vocabulário | termos totais | termos/doc |
|---|---|---|---|
| nada | 7.471 | 226.526 | 161,8 |
| stopwords | 7.352 | 132.721 | 94,8 |
| stemming | 4.820 | 226.526 | 161,8 |
| stopwords+stemming | 4.709 | 132.721 | 94,8 |

**As duas etapas agem em eixos ortogonais.** Stopwords cortam **41% dos tokens** mas só
119 termos do vocabulário → ganho de **eficiência**. Stemming não muda o total de tokens
mas corta **35% do vocabulário** → ganho de **recall**. Há pouca sobreposição entre o que
cada uma elimina, e por isso somam.

Termos mais frequentes passam de `the, of, and` (sem pré-processamento) para
`flow, pressur, number, boundari, layer` — o vocabulário técnico do domínio.

## R2 · Modelo Vetorial

Peso `ltc`: $w_{t,d} = (1 + \log_{10}\mathrm{tf}) \cdot \log_{10}(N/\mathrm{df})$, similaridade por cosseno.
Índice invertido termo → {doc: peso}; o score percorre só os termos da consulta.

**Verificação:** documento usado como consulta contra si mesmo dá cosseno **exatamente 1,0000**.

**Candidatos por consulta** (documentos com score não nulo): `nada` 1366,3 · `stopwords` 730,8
· `stemming` 1373,3 · `stop+stem` 904,6. Sem remover stopwords, 1366 dos 1400 documentos
entram no cálculo — basta um *the* em comum. O ranking quase não muda (idf ≈ 0), mas
percorre-se a coleção inteira: é a medida concreta do ganho de eficiência do R1.

## R3 · BM25

$$\mathrm{BM25} = \sum_{t \in q} \mathrm{qtf} \cdot \mathrm{idf}_t \cdot \frac{\mathrm{tf}(k_1+1)}{\mathrm{tf} + k_1(1 - b + b\,|d|/\mathrm{avgdl})}$$

- **idf com deslocamento** $\ln(1 + \frac{N-\mathrm{df}+0,5}{\mathrm{df}+0,5})$. Sem o `1+`, 16 termos
  (`of`, `the`, `and`…) teriam idf **negativo** na config `nada` — um documento perderia
  pontos por conter o termo da consulta.
- **$k_1$ e $b$ são parâmetros de busca, não do índice** — o R7 varia sem reconstruir nada.

**Saturação** (a diferença central em relação ao Vetorial): com $k_1=1,2$ a contribuição vai
de 1,000 (tf=1) a 2,148 (tf=50) e converge para $k_1+1=2,2$. O tf-idf logarítmico no mesmo
intervalo vai a 2,699 e **continua subindo**.

## R4 · Avaliação quantitativa

Métricas: P@10, R@10, MAP (exigidas) + NDCG@10 (ganho **5 − grau**, invertido porque o grau 1 é o
mais relevante; IDCG da própria consulta).
MAP sobre o ranking completo. 2 modelos × 4 configs × 225 consultas.

| modelo | config | P@10 | R@10 | MAP | NDCG@10 |
|---|---|---|---|---|---|
| bm25 | stopwords+stemming | 0,2356 | 0,3991 | **0,3050** | 0,3676 |
| bm25 | stemming | 0,2276 | 0,3882 | 0,2951 | 0,3599 |
| bm25 | stopwords | 0,2240 | 0,3801 | 0,2803 | 0,3439 |
| vetorial | stopwords+stemming | 0,2120 | 0,3681 | 0,2718 | 0,3285 |
| bm25 | nada | 0,2164 | 0,3670 | 0,2694 | 0,3344 |
| vetorial | stemming | 0,2027 | 0,3532 | 0,2589 | 0,3146 |
| vetorial | stopwords | 0,1987 | 0,3368 | 0,2475 | 0,3022 |
| vetorial | nada | 0,1933 | 0,3262 | 0,2396 | 0,2924 |

- **BM25 vence nas 4 métricas, em todas as 4 configs.** O pior BM25 (0,2694) empata com o
  melhor Vetorial (0,2718): o modelo pesa tanto quanto o pré-processamento.
- Ordenação idêntica nos dois modelos: `nada` < `stopwords` < `stemming` < `stop+stem`.
- **Stemming (+9,5% de MAP) contribui mais que stopwords (+4,0%)** — confirma o R1: o idf já
  neutraliza termos ubíquos, então stopwords rendem eficiência, não efetividade.
- R@10 > P@10 porque a maioria das consultas tem menos de dez relevantes: o teto de P@10 já é < 1 por construção.

## R5 · Comparação entre modelos

Config fixa `stopwords+stemming`, comparação pareada por consulta.

- **BM25 ganha em 144 consultas, perde em 74, empata em 7.** A média esconde a dispersão.
- **Sobreposição do Top-10: 6,48/10** (mediana 6, mínimo 1) — rankings genuinamente distintos.

**Hipótese testada — a diferença é a normalização por comprimento.** Das três características
candidatas, só uma correlaciona com o ΔAP:

| característica | r |
|---|---|
| comprimento médio dos relevantes | **0,374** |
| tamanho da consulta | −0,067 |
| número de relevantes | −0,069 |

Verificando nos 1612 pares (consulta, documento relevante), o ganho de posição do BM25 cresce
monotonicamente com o comprimento (avgdl = 94,8):

| \|d\| | <60 | 60–90 | 90–120 | 120–160 | >160 |
|---|---|---|---|---|---|
| posições ganhas pelo BM25 | −14,1 | +3,4 | +8,5 | +19,8 | **+29,4** |

Relevantes com >120 termos no Top-10: **BM25 157 × Vetorial 105**. Com <60 termos inverte: 128 × 158.

**Mecanismo:** o cosseno divide por $\lVert\vec d\rVert$, que cresce com *todos* os termos do
documento — penaliza o documento longo mesmo quando ele é longo por tratar o assunto a fundo.
O BM25 com $b=0,75$ desconta só 75% do desvio em relação ao `avgdl`, e sobre a saturação da tf.

## R6 · Análise por consulta

Seleção automática pelo ΔAP (não escolhida a dedo).

- **BM25 superior — q167, q36.** Relevantes de 142 e 195 termos; o Vetorial os enterra. Na q36 o
  Vetorial não traz *nenhum* relevante no Top-5.
- **Vetorial superior — q17, q190.** Inversão exata: o relevante da q17 tem 37 termos e o Vetorial
  o põe em 1º; o Top-5 do BM25 é feito de documentos longos e não tem relevante algum.
- **Ambos falham — q13, q22.** AP **exatamente 0,000**: os relevantes não compartilham **um único
  termo** com a consulta. 27 consultas têm P@10 = 0 nos dois modelos.

A **simetria entre (i) e (ii)** é a evidência mais forte de que a diferença é normalização por
comprimento, e não superioridade genérica de um modelo.

## R7 · Variação de $k_1$ e $b$

**MAP** (NDCG@10 produz a mesma ordenação, máximo na mesma célula):

| $k_1$ \ $b$ | 0,00 | 0,75 | 1,00 |
|---|---|---|---|
| 0,5 | 0,2668 | 0,2902 | 0,2925 |
| 1,2 | 0,2756 | 0,3050 | 0,3010 |
| 2,0 | 0,2798 | **0,3106** | 0,3066 |

- **$b$ domina** (+10,7% ao variar) sobre **$k_1$** (+7,0%), mas nenhum é desprezível.
- **$b=0$ é a única escolha claramente ruim.** De 0 → 0,75 o ganho é grande; de 0,75 → 1 há leve
  *perda*. Alguma normalização é essencial, normalização total é excessiva.
- $k_1=2,0$ vence nas três colunas: saturar devagar favorece resumos técnicos curtos, onde
  repetir um termo é sinal de assunto, não de enchimento.
- Melhor ponto rende **apenas +1,8%** sobre o padrão — mantivemos $k_1=1,2$, $b=0,75$.

**q161 — `b` reorganiza o ranking** (3/10 em comum entre $b=0$ e $b=1$): o comprimento médio do
Top-5 cai de **218 para 95 termos** (≈ avgdl), o relevante de 81 termos entra em 3º, AP@10 de
0,417 → 0,500. Confirma **manipulando o parâmetro** o que R5 e R6 observaram.

## R8 · Modificação de consultas

Cinco reformulações manuais, justificadas pelo texto da consulta e conhecimento de domínio —
**nunca pelos qrels**, cujo uso para ajustar rankings o enunciado proíbe.

**Reformular mexe muito no ranking, e as duas métricas discordam:** 4,4 de 10 documentos trocados,
P@10 médio **idêntico** (0,1100 → 0,1100) mas NDCG@10 médio **melhor** (0,2143 → 0,2450). Dos 10
casos: 4 melhoram, 3 pioram, 3 ficam iguais. Como o P@10 não se move, as reformulações não
recuperaram *mais* relevantes — recuperaram relevantes *melhores*, ou os colocaram acima. O ganho
é de qualidade do topo, não de cobertura, e só é visível porque o NDCG@10 usa os graus.

- **q13 sai do zero** (acréscimo de termos do domínio: *shock wave*, *boundary layer*,
  *interaction*). O doc 265 entra no Top-10 nos dois modelos. É a confirmação prática do R6:
  quando a falha é de vocabulário, só mudar a representação resolve. Mas a expansão trouxe
  junto dois vizinhos temáticos não relevantes — ela alarga de forma indiscriminada.
- **q22 continua em zero.** Limpar termos conversacionais eliminou ruído mas não *acrescentou*
  nada: sem interseção, não há o que ponderar. Remoção de ruído ≠ expansão de vocabulário.
- **q82 é o fracasso instrutivo.** Remover nomes de autores derrubou o NDCG@10 do BM25 de 0,540
  para **0,275**: `kuchemann` tem **df=1, idf=6,84** e apontava
  exatamente para o artigo relevante. (`multhopp` sequer existe na coleção.)

## R9 · Análise de erros

**Dois não relevantes no topo:**

- **q7, doc 492, score 64,08** — o maior entre os erros desse tipo. Título quase idêntico à consulta.
  Causa: `ogiv` (idf 4,57) e `forebodi` (idf 5,10) — raríssimos — **somados** a tf=4 em
  `pressur`, `angl` e `attack`. Alto idf e alta tf ao mesmo tempo. Julgado **−1**.
- **q13, doc 496, 1º nos dois modelos.** `buzz` tem **df=1, idf=6,84**: existe em um único
  documento da coleção, justamente esse, e sozinho garante a 1ª posição. Julgado **−1**.

**O padrão por trás deles** — cada consulta tem exatamente um documento de grau −1, e ele é:

| | 1º lugar | Top-3 | Top-10 | mediana |
|---|---|---|---|---|
| BM25 | 96/225 (**43%**) | 137 (61%) | 163 (72%) | 2 |
| Vetorial | 81/225 (36%) | 122 (54%) | 156 (69%) | 3 |

Somados dois indícios estruturais — o id do documento −1 correlaciona **0,662** com o número da
consulta, e consultas consecutivas compartilham o mesmo (q1/q2→486, q7/q8→492, q13/q14→496) —
a explicação mais provável é que **o documento −1 seja o artigo a partir do qual a consulta foi
formulada**: casamento lexical quase perfeito, posição de topo sistemática, numeração
correlacionada, e o julgamento "*references of no interest*" para quem já conhece o artigo.

> **Consequência:** em 43% das consultas o melhor resultado do sistema é contabilizado como erro
> por construção da coleção. Há um teto na P@10 que não vem de falha de modelagem — advertência
> necessária ao ler os números absolutos de R4 a R7.

**Um relevante de grau 1 fora do Top-10:** q176, doc 583, **posição 939 de ~1400**. Compartilha
**um** termo com a consulta: `use`, um verbo genérico de idf 1,00 — e ainda assim é grau 1,
*complete answer to the question*. É o oposto exato dos dois primeiros: casamento lexical quase
nulo com relevância máxima.

---

## Conclusões transversais

**1. A normalização por comprimento é o eixo que separa os dois modelos.** R5 correlaciona
(r = 0,374) e mede no nível do documento (+29,4 posições para relevantes longos), R6 exibe a
simetria em casos individuais, R7 confirma manipulando `b` diretamente. O Modelo Vetorial é um
extremo rígido dessa escala: normaliza sempre pelo módulo completo, sem parâmetro que afrouxe.

**2. O descasamento de vocabulário é o teto de ambos os modelos.** Os dois pontuam por
sobreposição de termos ponderada; onde a interseção é vazia o score é **zero por construção**, e
nenhum ajuste de ponderação, `k1` ou `b` chega lá (R6). Só mudar a representação da consulta
funciona (R8), e mesmo assim de forma imprecisa — o caminho robusto seria representação semântica.

**3. Termos de df muito baixo são apostas de alto risco.** `kuchemann` (df=1) era o sinal mais
forte da consulta e removê-lo custou metade do NDCG (R8); `buzz` (df=1) sozinho colocou um
documento não relevante em 1º (R9). O idf os torna decisivos nos dois sentidos.

**4. Hierarquia dos ganhos** — onde vale investir esforço:

| intervenção | ganho de MAP |
|---|---|
| pré-processamento (`nada` → `stop+stem`) | **+13,2%** |
| modelo (Vetorial → BM25) | **+12,2%** |
| parametrização do BM25 (padrão → ótimo) | +1,8% |

Pré-processamento e escolha do modelo valem uma ordem de grandeza mais que ajuste fino de
parâmetros. Os valores convencionais $k_1=1,2$ e $b=0,75$ já estão bem calibrados para esta coleção.

---

## Reprodução

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python -c "import nltk; nltk.download('stopwords')"
jupyter notebook   # executar main.ipynb na ordem
```

Saídas: `results_req4/` (métricas por consulta e agregadas), `results_req5/` e `results_req7/`
(gráficos). Python 3.12.3; `ir_datasets` para a coleção, `nltk` para stopwords e stemming,
`pandas`/`matplotlib` para tabelas e gráficos. Os algoritmos de indexação, ponderação, busca e
todas as métricas são implementações próprias.
