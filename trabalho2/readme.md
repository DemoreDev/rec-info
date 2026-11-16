# Trabalho Prático 2

Sistema de Recuperação de Informação com busca lexical (BM25 sobre índice invertido próprio),
busca semântica (bi-encoder), busca híbrida (RRF), reranking (cross-encoder) e RAG, avaliado
sobre o **NFCorpus** (BEIR).

## Integrantes do grupo

- Leonardo D. Demore - 15674786
- Arthur Araujo - 14651458

## Componentes escolhidos

| componente | escolha | como foi escolhido |
|---|---|---|
| pré-processamento | minúsculas, remoção de acentos, regex `[a-z0-9]+`, stopwords do NLTK e stemming de Porter | maior NDCG@10 no dev entre 5 configurações |
| modelo lexical | BM25 (`k1 = 2,0`, `b = 0,5`) sobre índice invertido implementado do zero | modelo vencedor do Trabalho 1; `k1`/`b` por grid no dev |
| correção ortográfica | k-gramas (k = 2) → Jaccard (≥ 0,2) → distância de Damerau-Levenshtein (≤ 2) | acurácia em erros sintéticos no dev |
| modelo de embeddings | `BAAI/bge-small-en-v1.5` (Sentence Transformers, 384 dimensões, cosseno) | maior NDCG@10 no dev entre 4 bi-encoders |
| busca híbrida | Reciprocal Rank Fusion, `k = 60`, Top-100 de cada sistema | maior NDCG@10 no dev entre k ∈ {10, 30, 60, 100} |
| reranking | `cross-encoder/ms-marco-MiniLM-L6-v2` sobre o Top-50 do híbrido | maior NDCG@10 no dev entre 2 cross-encoders (L6 e L12) |
| LLM (RAG) | `qwen2.5:3b` (Qwen2.5-3B-Instruct, Q4_K_M) via Ollama, local, temperatura 0, seed 42 | maior modelo que cabe na RAM disponível em CPU |

## Hardware utilizado

Notebook sem GPU: Intel Core i5-1340P (12 núcleos / 16 threads), 7,4 GB de RAM, Linux.
Todas as etapas (embeddings, cross-encoder e LLM) rodam em CPU.

## Instalação

Pré-requisitos: Python 3.12 (usamos 3.12.16 via conda) e, para o RAG, o [Ollama](https://ollama.com).

```bash
conda create -n ri-t2 python=3.12
conda activate ri-t2
pip install torch==2.15.0 --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
python -c "import nltk; nltk.download('stopwords'); nltk.download('wordnet')"
```

Ollama e o modelo da LLM:

```bash
# instalação padrão (ou baixar o pacote standalone em https://ollama.com/download)
curl -fsSL https://ollama.com/install.sh | sh
ollama serve &          # servidor local em http://127.0.0.1:11434
ollama pull qwen2.5:3b
```

## Execução

```bash
jupyter notebook main.ipynb   # executar as células na ordem
```

Ordem recomendada: o notebook é único e deve ser executado de cima para baixo. O servidor do
Ollama precisa estar ativo antes da seção do requisito 11.

Na primeira execução o notebook:

1. baixa e extrai o NFCorpus oficial do BEIR em `data/nfcorpus/` (arquivo `nfcorpus.zip`);
2. baixa os modelos do Hugging Face (bi-encoders e cross-encoders);
3. calcula e guarda os embeddings dos documentos em `data/embeddings/`;
4. calcula e guarda os scores do cross-encoder em `data/rerank/`;
5. gera as respostas da LLM e as guarda em `results/rag_answers.json`.

Esses arquivos não fazem parte da entrega (podem ser regerados), exceto `results/rag_answers.json`,
que é mantido porque a avaliação humana se refere a essas respostas específicas. Com os caches
presentes, uma nova execução completa leva poucos minutos.

## Linguagem e principais bibliotecas

- Python 3.12.16
- `sentence-transformers` 6.3.0 / `transformers` 5.23.0 / `torch` 2.15.0 (CPU): bi-encoder e cross-encoder.
- `nltk` 3.10.4: stopwords, stemmer de Porter e lematizador WordNet.
- `ir_measures` 0.4.3: conferência das métricas implementadas.
- `pandas`, `numpy`, `matplotlib`, `scikit-learn` (kappa de Cohen), `requests` (API do Ollama).
- Ollama 0.40.16 com `qwen2.5:3b`.

Versões fixadas em `requirements.txt`. O índice invertido, o BM25, a correção ortográfica,
o RRF e todas as métricas são implementações próprias.

## Base de dados

NFCorpus na versão do BEIR (`corpus.jsonl`, `queries.jsonl`, `qrels/dev.tsv`, `qrels/test.tsv`),
obtido de <https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/nfcorpus.zip>.
O conjunto dev (324 consultas) foi usado para todas as escolhas; o test (323 consultas), só para a
avaliação final.

## Resultados

Em `results/`:

- `results_lexical.csv`, `results_semantic.csv`, `results_hybrid.csv`, `results_reranking.csv`:
  Top-100 de cada consulta de test (`query_id, doc_id, rank, score, relevance`). No reranking, a
  coluna `stage` indica se o score é do cross-encoder (posições 1–50) ou do RRF (51–100).
  O BM25 só pontua documentos com ao menos um termo da consulta: quando há menos de 100 desses
  documentos, o ranking lexical é mais curto (e uma consulta sem nenhum termo na coleção não tem linhas).
- `metrics_per_query.csv`: P@10, R@10, AP@10 e NDCG@10 por consulta e sistema; `metrics_aggregated.csv`: médias.
- `typo_experiment.csv`: experimento de correção ortográfica (10 consultas).
- `rag_experiment.csv`: experimento sem RAG × com RAG, com prompts, respostas e notas.
- `rag_answers.json`: cache das respostas da LLM.
- Gráficos: `comparacao_lexical_semantico.png`, `sistemas_test.png`.

Em `avaliacao/`: planilhas da avaliação humana às cegas (`avaliador_1.csv`, `avaliador_2.csv`),
respostas embaralhadas, documentos de contexto, instruções e a chave de desembaralhamento.
O notebook só cria esses arquivos se eles ainda não existirem e lê as notas preenchidas para
calcular médias, concordância e o `rag_experiment.csv`.
