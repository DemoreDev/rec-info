# Trabalho Prático 2 — Recuperação da Informação

Busca lexical, semântica, híbrida, reranking e RAG sobre o **NFCorpus** (BEIR). Notebook: `main.ipynb`.

**Coleção:** 3.633 documentos (título + resumo de artigos médicos), consultas de 2 a 3 palavras
(títulos de artigos e vídeos do NutritionFacts.org). Dev: 324 consultas, test: 323. Graus 1 e 2, com
**35–38 relevantes por consulta** em média (até 919). Os qrels vêm de citações (grau 2 = citação direta,
1 = indireta), não de leitura dos documentos: 86% da coleção é relevante para alguma consulta do dev.

**Protocolo:** todas as escolhas foram feitas no dev pelo NDCG@10 médio; o test foi usado só na
avaliação final.

---

## R1 · Pré-processamento

Minúsculas, remoção de acentos (NFKD), regex `[a-z0-9]+`, stopwords do NLTK e Porter. Escolha no dev
entre 5 configurações:

| config | vocabulário | NDCG@10 dev |
|---|---|---|
| **stopwords+stemming** | 19.398 | **0,2957** |
| stemming | 19.496 | 0,2897 |
| stopwords+lematização | 24.388 | 0,2884 |
| stopwords | 26.740 | 0,2747 |
| nada | 26.869 | 0,2678 |

Mesma ordem do T1. A lematização (WordNet) recupera 65% do ganho do stemming com um vocabulário 26% maior.

## R2 · Índice invertido (do zero)

`dict` termo → lista de `(doc_idx, tf)` + comprimentos + N + avgdl. **19.398 termos, 359.738 postings**
(mediana 2 por termo; 47% dos termos com df = 1). Construção em 0,13 s; **0,26 ms por consulta**.
Busca term-at-a-time: uma consulta toca em média **494 dos 3.633** documentos (13,6%).

## R3 · BM25

Grid 5 × 5 no dev: **k1 = 2,0, b = 0,5** (0,2977), só +0,7% sobre os valores usuais. O grid é plano
(2,7% de amplitude), e os relevantes têm o mesmo comprimento médio da coleção, então o comprimento
não explica a preferência por b menor.

## R4 · Correção ortográfica

k-gramas (k = 2) → Jaccard ≥ 0,2 → Damerau-Levenshtein (OSA, ≤ 2). Ajuste no dev com 2.079 erros
sintéticos: **87,7%** de acerto (inserção 99%, transposição 78%). Todos os erros são empates na
distância mínima: o limite está no desempate, não na geração de candidatos. Termos cujo radical já
está no índice são protegidos (evita `walter` → `walters`).

| | original | com erro | corrigida |
|---|---|---|---|
| BM25 (10 consultas) | 0,3421 | 0,0836 (−76%) | 0,3399 |
| semântico (opcional) | 0,4185 | 0,2035 (−51%) | 0,4127 |

9 de 10 correções exatas; **100% do NDCG perdido foi recuperado** nas 8 consultas afetadas.
Casos à parte: em *Sexually Transmitted Fish Toxin* o erro *melhorou* a busca (ambiguidade de
`transmit`), e em *low-carb diets* a correção errou (`varb` → `var`, empate com `carb`).

## R5 · Bi-encoder

**`BAAI/bge-small-en-v1.5`** (384 dimensões, cosseno, 512 tokens, 9,1% dos documentos truncados).
Escolhido no dev entre 4 modelos: bge 0,3281 · e5 0,3215 · all-MiniLM 0,3028 · multi-qa-MiniLM
0,2734 (abaixo do BM25). Embeddings dos documentos em 539 s; 122 ms por consulta, quase tudo na
codificação da consulta (a busca exaustiva leva 6,5 ms).

## R6 · Avaliação (test)

| sistema | P@10 | R@10 | MAP@10 | NDCG@10 |
|---|---|---|---|---|
| lexical | 0,2402 | 0,1548 | 0,1231 | 0,3300 |
| semântico | 0,2567 | 0,1620 | 0,1230 | 0,3433 |

Conferido com o `ir_measures` (diferença máxima de 3×10⁻¹⁶). O MAP@10 empata porque o lexical vence
nas consultas com 1–5 relevantes, onde cada acerto vale muito no AP@10.

## R7 · Lexical × semântico

Sobreposição do Top-10 de só **3,14/10**. 61% dos relevantes não têm nenhum termo da consulta.

- **Semântico vence:** *myelopathy* (termo ausente da coleção; o relevante fala em "degeneration of
  the spinal cord"), *Vitamin D: Shedding some light…* (idiomatismo; e o `d` é stopword no NLTK),
  *low-carb diets* (a coleção escreve *low-carbohydrate*).
- **Lexical vence:** *kohlrabi* (df = 1; o WordPiece quebra em `ko ##hl ##ra ##bi` e o semântico traz
  *Kombucha*), *Phytates for the Treatment of Cancer* (o semântico generaliza para fitoquímicos),
  *shelf life*.

## R8 · RRF

k = 60 no dev (curva plana). **Híbrido: NDCG@10 0,3618** (+9,6% sobre o lexical, +5,4% sobre o
semântico), melhor nas quatro métricas. Fica igual ou abaixo do pior dos dois sistemas em só 39 de 323 consultas.

## R9 · Reranking

`cross-encoder/ms-marco-MiniLM-L6-v2` (melhor que o L12 no dev). **Piora o híbrido:** dev
0,3401 → 0,3341, test 0,3618 → 0,3564. Custa 4,1 s por consulta (97% do pipeline).

## R10 · Por que o reranking piora

Ele sobe os relevantes de grau 2 (+0,56 posição) e resgata os que só o BM25 achou (+10,4), mas
rebaixa os de grau 1 (−1,15), que são a maioria. No Top-10, os de grau 1 caem de 662 para 611.
Hipótese: o MS MARCO ensina "responde à pergunta", e os qrels do NFCorpus marcam "citado pelo artigo".
Casos: *Harvard Physicians' Study II* (+0,37; reconhece o ensaio clínico) e *myelopathy* (−1,0;
confunde com *multiple myeloma*).

## R11–R14 · RAG

`qwen2.5:3b` (Q4_K_M) via Ollama, local, CPU, temperatura 0, seed 42, contexto = Top-5 após o
reranking, respostas em inglês. Prompt máximo de 2.943 tokens (sem truncamento).

- **Recuperação:** P@5 0,44 · R@5 0,074 · **Hit@5 0,9**. Só *More Antibiotics In White Meat or Dark
  Meat?* não leva nenhum relevante ao contexto: o relevante ideal fala em *muscle tissues*, fica em
  174º no BM25 e 33º no semântico, cai para 76º no RRF e o cross-encoder nunca o vê (falha
  lexical + fusão).
- **Respostas B:** 2,5 fontes citadas em média, **nenhum id fora do contexto**.
- **Avaliação humana:** *pendente* (planilhas cegas em `avaliacao/`).

---

## Conclusões transversais

1. **Lexical e semântico são complementares, não concorrentes.** Rankings com 3 documentos em comum
   em 10; o lexical ganha em termos raros e exatos, o semântico em paráfrase e variação de
   vocabulário. A fusão por RRF é a única intervenção que melhora tudo.
2. **Um modelo "mais preciso" em geral não é melhor aqui.** O cross-encoder e o modelo multi-qa,
   treinados em pares pergunta-resposta, pioram numa coleção cuja relevância é temática (citação).
   Só a validação no dev revelou isso.
3. **O gargalo do RAG é a primeira etapa da recuperação**: 65% dos relevantes das dez consultas não
   aparecem em nenhum dos rankings iniciais.
4. **Hierarquia dos ganhos (NDCG@10, test):** fusão (+9,6% sobre o BM25) > modelo semântico (+4,0%) >
   pré-processamento (dev: +10,4%, sobre `nada`) > parâmetros do BM25 (dev: +0,7%) > reranking (−1,5%).
