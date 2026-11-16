# Instruções da avaliação

1. Cada integrante preenche **só a sua** planilha (`avaliador_1.csv` ou `avaliador_2.csv`), sem ver a do outro.
2. Leia a resposta em `respostas.md` e os documentos da pergunta em `contextos.md`.
3. Dê nota **0, 1 ou 2** em cada critério e escreva uma justificativa curta para cada nota.
4. **Não abra `chave.csv`** antes de terminar.
5. Avalie a rastreabilidade de todas as respostas. Para respostas sem nenhuma fonte, a nota é 0 (o notebook converte para N/A na condição sem RAG).

| Critério | 0 | 1 | 2 |
|---|---|---|---|
| Fidelidade | Afirmações importantes não são sustentadas pelos documentos ou os contradizem. | Resposta parcialmente sustentada. | As afirmações principais são sustentadas pelos documentos. |
| Completude | Não responde aos aspectos principais. | Responde parcialmente. | Cobre adequadamente os aspectos que podem ser respondidos. |
| Utilidade | Não responde à necessidade da consulta. | Resposta parcialmente útil. | Responde diretamente à necessidade apresentada. |
| Rastreabilidade | Não é possível identificar as fontes. | Apenas parte das afirmações pode ser associada às fontes. | As afirmações principais podem ser associadas aos doc_ids apresentados. |

A fidelidade de **todas** as respostas é julgada contra os documentos de `contextos.md`, inclusive as respostas que não citam nenhum documento.
