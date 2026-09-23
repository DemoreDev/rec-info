# Trabalho Prático 1

## Integrantes do grupo

Leonardo D. Demore - 15674786
Arthur Araujo - 14651458

## Instalação das dependências

Pré-requisitos:

- Python 3.12.3
- Jupyter Notebook ou JupyterLab

Crie e ative um ambiente virtual:

```bash
python -m venv venv
source venv/bin/activate
```

Instale as dependências do projeto:

```bash
pip install -r requirements.txt
```

Baixe os recursos do NLTK utilizados no pré-processamento (lista de stopwords em
inglês):

```bash
python -c "import nltk; nltk.download('stopwords')"
```

## Execução

Inicie o Jupyter:

```bash
jupyter notebook
```

Abra o arquivo `main.ipynb` e execute as células na ordem apresentada. O notebook carrega a base Cranfield e, quando executado, gera os arquivos CSV dentro de `data/`, no mesmo diretório e nível que o arquivo `main.ipynb`.

## Linguagem e principais bibliotecas

- Linguagem: Python 3.12.3
- `pandas`: manipulação e exportação dos dados em formato tabular.
- `ir_datasets`: carregamento da base de recuperação de informação Cranfield.
- `nltk`: pré-processamento textual — lista de stopwords em inglês e stemmer de Porter.
- `numpy`: operações numéricas auxiliares no cálculo das métricas de avaliação.
- `matplotlib` e `seaborn`: produção de gráficos e tabelas de resultados.
- Jupyter/IPython: execução do notebook.

A lista completa e as versões fixadas estão em `requirements.txt`.

## Base de dados

A base utilizada é a **Cranfield**, um conjunto clássico para avaliação de sistemas de recuperação de informação. Ela é obtida por meio da biblioteca [`ir_datasets`](https://ir-datasets.com/) com o identificador `cranfield`:

```python
import ir_datasets

dataset = ir_datasets.load("cranfield")
```

O notebook transforma os dados em três arquivos CSV locais:

- `data/cranfield_docs.csv`: documentos, títulos, textos, autores e referências bibliográficas.
- `data/cranfield_queries.csv`: consultas da base.
- `data/cranfield_qrels.csv`: julgamentos de relevância das consultas em relação aos documentos.

## Resultados dos experimentos

A execução do notebook grava os resultados em um diretório por requisito:

- `results_req4/metricas_por_consulta.csv`: métricas por consulta (P@10, R@10, AP, NDCG@10) para cada combinação de modelo e configuração de pré-processamento — 1800 linhas.
- `results_req4/metricas_agregadas.csv`: médias sobre as 225 consultas.
- `results_req5/comparacao_modelos.png`: gráficos da comparação entre o Modelo Vetorial e o BM25.

