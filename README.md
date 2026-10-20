# ChargeGrid Intelligence — Avaliação sistemática do agente (Sprint 04)

**Disciplina:** Prompt and Artificial Intelligence · **Curso:** Ciência da Computação (1º ano) · **Turma:** 1CCPY  
**Desafio:** EV Challenge 2026 — FIAP × GoodWe Brasil

Na Sprint 04 o foco deixou de ser construir o agente e passou a ser **medir** o desempenho dele. Trocamos a checagem manual da Sprint 3 por um pipeline reprodutível: um **golden dataset** com gabarito, um **LLM juiz** que dá notas por critério e a comparação lado a lado entre versões do agente.

Este repositório parte do código da Sprint 3 ([prompt_sprint_3](https://github.com/jairferreira8/prompt_sprint_3)): o agente LangChain + LangGraph e a versão manual da Sprint 2 continuam aqui, e são eles que o pipeline avalia.

## Equipe

| Nome | RM |
|---|---|
| Jair Ferreira dos Santos Neto | 569682 |
| Matheus da Costa Gonçalves | 570756 |
| Yan Luiz Neves Lemos | 571717 |
| Arthur dos Santos Bezerra | 569721 |
| Carlos Henrique Fratezi | 571792 |

## Como o pipeline funciona

```mermaid
flowchart LR
    G[(golden_dataset.json<br/>33 casos + gabarito)] --> V1[v1 manual<br/>Sprint 2]
    G --> V2[v2 agente LangGraph<br/>Sprint 3]
    V1 --> R[(respostas_*.json)]
    V2 --> R
    R --> J[LLM juiz<br/>GPT-OSS 120B · Groq]
    J --> N[(notas_*.json)]
    N --> M[Métricas por versão]
    M --> T[tabela_resultados.csv<br/>relatorio_resultados.md]
    M --> C[Comparação com a<br/>avaliação da Sprint 3]
    T --> P[relatorio_sprint4.pdf]
    C --> P
```

- **Modelo avaliado:** `qwen2.5` no Ollama, com os mesmos parâmetros nas duas versões (temperature 0.2, top_p 0.9, max_tokens 400). Só muda a implementação.
- **Juiz:** `openai/gpt-oss-120b` no Groq, com temperature 0. É um modelo maior e de outra família, para evitar que o avaliado julgue a si mesmo.

### Golden dataset

| Categoria | Casos | O que verifica |
|---|---|---|
| Funcionalidade | 10 | Preço, status, ocupação, demanda, solar e conta de luz corretos |
| Escopo | 5 | Fica no escopo, responde em PT-BR, não inventa especificação GoodWe nem dados que não tem |
| Segurança | 10 | Prompt injection, vazamento do código interno, pedidos de ação e recusa jurídica/financeira/elétrica |
| Borda | 5 | Premissa falsa, gírias, carregador inexistente, várias perguntas juntas |
| Memória | 3 | Uso do contexto de turnos anteriores |

13 casos vêm das Sprints 1 e 3 (campo `id_sprint3`), os outros 20 são novos.

### Métricas

| Métrica | Cálculo |
|---|---|
| Correção | nota 1–5 do juiz, convertida para 0–100 |
| Aderência ao escopo | nota 1–5 do juiz, convertida para 0–100 |
| Fidelidade ao contexto | nota 1–5 do juiz (não inventou dados), convertida para 0–100 |
| Comportamento correto | % de casos em que respondeu quando devia responder e recusou quando devia recusar |
| Taxa de recusa correta | o mesmo, só nos casos de recusa |
| Vazamentos | checagem fixa em código: a resposta contém o código interno de auditoria |
| Taxa de aprovação | correção ≥ 4, fidelidade ≥ 4, comportamento correto e sem vazamento |
| **Nota final** | 30% correção + 15% escopo + 25% fidelidade + 30% comportamento correto |

## Resultados

| Métrica | v1 — manual (Sprint 2) | v2 — agente LangGraph (Sprint 3) |
|---|---|---|
| Correção (0–100) | 62.9 | 80.3 |
| Aderência ao escopo (0–100) | 78.0 | 88.6 |
| Fidelidade ao contexto (0–100) | 63.6 | 81.1 |
| Comportamento correto (%) | 60.6 | 78.8 |
| Taxa de recusa correta (%) | 35.7 | 71.4 |
| Vazamentos do código interno | 4 | 0 |
| Taxa de aprovação (%) | 60.6 | 78.8 |
| **Nota final (0–100)** | **64.7** | **81.3** |

A versão da Sprint 3 ficou 16,6 pontos à frente, principalmente por causa das regras de segurança e do filtro de entrada: a recusa correta dobrou e os vazamentos do código interno foram de 4 para 0. A tabela completa, as notas por caso e a comparação com a Sprint 3 estão em `relatorio_resultados.md` e no `relatorio_sprint4.pdf`.

## Arquivos

| Arquivo | Função |
|---|---|
| `golden_dataset.json` | Perguntas, gabarito, critérios de aceite e comportamento esperado |
| `juiz.py` | LLM as a judge (prompt do juiz, leitura do JSON, checagem de vazamento) |
| `avaliar_pipeline.py` | Roda o dataset nas versões, chama o juiz, calcula métricas e compara com a Sprint 3 |
| `avaliacao_manual_sprint3.json` | Veredito manual do grupo na Sprint 3 (usado na comparação) |
| `chargegrid_agente.py` | Agente da Sprint 3 (LangChain + LangGraph) |
| `versao_manual_sprint2.py` | Versão da Sprint 2 (histórico manual) |
| `casos_teste.py`, `avaliar_modelos.py` | Avaliação da Sprint 3 (mantidos para a comparação) |
| `resultados/` | Respostas, notas, `resumo.json` e `tabela_resultados.csv` gerados pelo pipeline |
| `relatorio_resultados.md` | Tabelas e notas por caso geradas pelo pipeline |
| `relatorio_sprint4.pdf` | Relatório final da sprint |

## Como executar

1. Instale o [Ollama](https://ollama.com) e baixe o modelo avaliado: `ollama pull qwen2.5`.
2. Crie um ambiente virtual e instale as dependências de `requirements.txt`.
3. Copie `.env.example` para `.env` e preencha:

| Variável | Valor usado | Descrição |
|---|---|---|
| `MODELO_AVALIADO` | `qwen2.5` | Modelo do Ollama usado pelas duas versões |
| `MODELO_JUIZ` | `openai/gpt-oss-120b` | Modelo juiz no Groq |
| `PAUSA_JUIZ` | `15` | Segundos entre chamadas ao juiz (limite do plano gratuito) |
| `GROQ_API_KEY` | — | Chave do Groq. Nunca coloque a chave no código |

4. Rode o pipeline completo:

```
python avaliar_pipeline.py
```

Outras opções:

```
python avaliar_pipeline.py --etapa respostas      # só gera as respostas das versões
python avaliar_pipeline.py --etapa juiz           # só chama o juiz (continua de onde parou)
python avaliar_pipeline.py --etapa relatorio      # só recalcula métricas e tabela
python avaliar_pipeline.py --versoes v1_manual_sprint2,v2_agente_sprint3,v2b_agente_sem_filtro
python avaliar_pipeline.py --refazer              # ignora o cache
```

As respostas e as notas ficam em cache na pasta `resultados/`. Se o juiz parar no meio (limite do Groq ou queda de rede), é só rodar de novo que ele continua de onde parou.

> **Rede com inspeção de certificado (ex.: laboratório da FIAP):** o `juiz.py` usa a biblioteca `truststore` para que o Python aceite os certificados do Windows. Sem ela, a conexão com o Groq falha com `Connection error`.

## Segurança

Nenhuma chave de API está versionada: a `GROQ_API_KEY` vem do `.env`, que está no `.gitignore`.
