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
- `scikit-learn`: [DESCREVER O USO, SE APLICÁVEL].
- `numpy`: [DESCREVER O USO, SE APLICÁVEL].
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

