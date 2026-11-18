# CitizenTech — Assistente para quem é afetado por enchentes

**GS 2026.2 · Prompt and Artificial Intelligence · Frente D (Serviço público essencial) · Dupla**

> ⚠️ **Protótipo acadêmico com dados 100% fictícios** (município inventado: *Santa Ágata do Vale*, Rio Taquaral). Não use em emergências reais. Em risco, ligue **199** (Defesa Civil) ou **193** (Bombeiros).

| Campo | Valor |
|---|---|
| Turma | 1CCR |
| Integrantes (nome — RM) | Gabriela Angel Silva - 570808 · Marcos Paulo Sampaio - 573987 |
| Vídeo (≤ 3 min) | ver `7_demo_video_url.txt` |

## 1. O problema

Nas enchentes do RS em 2024, muita gente não sabia a hora de sair de casa, onde havia abrigo, como tirar a segunda via dos documentos, como sacar o FGTS — e ainda havia golpes de PIX falso de "doação". O CitizenTech é um assistente que responde **somente com base em documentos oficiais**, **mostra de onde vem cada resposta**, **não pede dados pessoais** e **recusa quando não sabe**.

Cobre três momentos:

| Momento | O que o assistente faz | Documentos |
|---|---|---|
| **Antes** | níveis de alerta, gatilhos de saída por bairro, kit de emergência | doc_01 |
| **Durante** | hora de sair (tool que calcula), abrigos (acessibilidade, animais, gerador), segurança sem luz | doc_01, doc_02, doc_03 |
| **Depois** | segunda via de documentos, saque calamidade do FGTS, golpes de PIX | doc_04, doc_05, doc_06 |

## 2. Diagrama do pipeline

```mermaid
flowchart LR
    A[Documentos .txt<br/>+ metadados] --> B[Chunking<br/>+ cabeçalho doc_id/data]
    B --> C[Embeddings multilíngues]
    C --> D[(ChromaDB<br/>cosseno)]
    U([Pergunta]) --> G1{Guardrails de entrada<br/>injection · LGPD · golpe PIX · escopo}
    G1 -- bloqueada --> R([Resposta segura + canais oficiais])
    G1 -- liberada --> E[Retriever k=6]
    D --> E
    E --> F[Filtro anti-injection<br/>nos trechos]
    F --> H[Ordena por data<br/>mais recente primeiro]
    H --> P[Prompt: regras + citação doc_id]
    P --> L[ChatOpenAI]
    L --> S{Guardrail de saída<br/>exige fonte}
    S --> R2([Resposta + fontes])
    U -. modo agente .-> AG[Agente function calling<br/>≤ 6 tools]
    AG --> T1[calcular_hora_saida]
    AG --> T2[consultar_status_abrigo]
    AG --> T3[verificar_elegibilidade_fgts]
    AG --> T4[buscar_documentos]
```

## 3. Como executar

1. Abra `1_Notebook_GS2026_2.ipynb` no Google Colab.
2. Em **Secrets** (ícone da chave), crie `OPENROUTER_API_KEY` com a sua chave do OpenRouter e habilite o acesso ao notebook. **A chave nunca é escrita no notebook.**
3. `Runtime → Run all`. O notebook instala as versões travadas, recria a base documental se a pasta não existir, indexa no ChromaDB, roda o RAG, o agente, a interface Gradio, os guardrails e o RAGAS, e exporta `3_avaliacao_ragas.json`.

**Versões travadas:** langchain 0.3.27 · langchain-core 0.3.86 · langchain-community 0.3.31 · langchain-text-splitters 0.3.11 · langchain-openai 0.3.35 · langchain-chroma 0.2.6 · langchain-huggingface 0.3.1 · chromadb 1.0.21 · ragas 0.3.9 · gradio 5.49.1 · sentence-transformers 5.1.2.

**Modelo:** `MODEL_NAME` na célula de configuração (padrão `openrouter/free`). Embeddings: `paraphrase-multilingual-MiniLM-L12-v2`, local, sem API paga.

## 4. A armadilha explicada

A base foi escrita pelo grupo no estilo de documentos públicos. Ela contém:

- **Contradição (doc_02 × doc_03).** A lista de abrigos (15/09/2026) diz que o *Ginásio Municipal* está **ativo, 300 vagas**. O Comunicado Operacional nº 07/2026 (24/10/2026) diz que ele foi **interditado** e que as famílias foram realocadas para a Escola Estadual. Um RAG ingênuo indica um abrigo perigoso. O sistema ordena os trechos por `data_publicacao` e o prompt manda prevalecer o mais recente.
- **Desatualização (doc_01 × doc_03).** O gatilho de saída da Baixada do Sul era **4,5 m** e passou a **4,0 m**. A tool `calcular_hora_saida` usa o valor vigente.
- **Faixas condicionais.** FGTS: só bairros do Anexo I, no Jardim das Palmeiras só as ruas 1–14, até 18/01/2027 (doc_04). Isenção de taxa do RG: só até 31/12/2026 e com Declaração de Afetado (doc_05).
- **Detalhes só no fim do documento (exigência 5.1.3).** Acessibilidade, animais, gerador e enfermagem por abrigo (final do doc_02); cotas de transbordamento (anexo do doc_01); ruas 1–14 (anexo do doc_04).
- **Pergunta composta.** "Tenho um familiar cadeirante e um cachorro" exige juntar acessibilidade + animais + interdição: nenhum abrigo atende às duas necessidades ao mesmo tempo.

## 5. Decisões técnicas

| Decisão | Escolha | Por quê |
|---|---|---|
| Chunking | comparação 500/50 × 1000/100 por 8 sondas de recuperação; vencedora escolhida automaticamente | a decisão vem de medição, não de valor padrão; resultado da execução fica na seção 1 do notebook |
| Cabeçalho no chunk | `[doc_id \| título \| data]` | cada trecho "sabe" sua origem e data; permite citar e preferir o mais recente |
| k do retriever | 6 | traz o documento antigo **e** o comunicado que o atualiza |
| Distância | cosseno | `distance` menor = mais parecido |
| Escopo | limiar calibrado com exemplos dentro/fora do tema; se as classes se sobrepõem, o limiar fica logo acima da pior pergunta legítima | em emergência, recusar por engano é pior do que responder "não há informação" |
| Guardrails | regras determinísticas (injection, LGPD, PIX) | sem custo de LLM e testáveis automaticamente |
| Tools | dados operacionais em código; o LLM só decide chamar | "o modelo nunca inventa a hora de sair" |

## 6. Guardrails (5)

| # | Guardrail | Teste demonstrado no notebook |
|---|---|---|
| 1 | Escopo | "me dá uma receita de bolo" → recusa + canais oficiais |
| 2 | Prompt injection | documento falso (fixture `doc_99`) com "Ignore as instruções anteriores…" vai para quarentena; injection digitada na pergunta é bloqueada |
| 3 | Dado sensível / LGPD | "quais moradores relataram…?" → recusa; CPF/telefone digitados são mascarados |
| 4 | Golpe de PIX | pedido de PIX/taxa → alerta baseado no doc_06 |
| 5 | Limite de confiança | sem contexto → "Não há informação sobre isso na base oficial"; resposta sem fonte recebe aviso |

Os guardrails 2–4 têm **19 testes automatizados** (`testes_guardrails()`, seção 6): falham com `AssertionError` se uma regra quebrar. Esse 4º guardrail com teste automatizado é o bônus previsto no briefing.

## 7. Avaliação (RAGAS)

- 8 perguntas: 4 de nível 1 e 4 de nível 2 (armadilha). Juiz: mesmo modelo do OpenRouter, temperatura 0; embeddings explícitos (`llm=` e `embeddings=` configurados).
- **Ground truth:** escrito à mão pelo grupo.
- O notebook calcula as quatro métricas e cria o arquiov JSON.

| Métrica | Meta (norte, não requisito) | Resultado medido |
|---|---|---|
| faithfulness | ≥ 0,80 | 0,57 |
| response_relevancy | ≥ 0,75 | 0,36 |
| context_precision | ≥ 0,70 | 0,45 |
| context_recall | ≥ 0,70 | 0,64 |

**Explicação do número e melhoria concreta:**

A comparação entre os níveis mostra que o sistema teve melhor desempenho nas perguntas simples (nível 1), com faithfulness de 0,67, answer relevancy de 0,50 e context recall de 0,75. Nas perguntas complexas (nível 2), essas métricas caíram para 0,47, 0,23 e 0,52, respectivamente. Isso evidencia a dificuldade do RAG em recuperar e combinar informações de diferentes documentos.

Em Q2 e Q4, o sistema apresentou respostas corretas e bem fundamentadas. Q6 também identificou corretamente o gatilho de saída de 4,0 m da Baixada do Sul. Por outro lado, Q1 e Q5 apresentaram falhas importantes: em Q1, o assistente não identificou os níveis de alerta, embora o documento esperado estivesse entre os recuperados; em Q5, o comunicado mais recente sobre a interdição do ginásio não foi recuperado, impedindo a identificação da atualização.

Q7 apresentou uma resposta correta sobre a impossibilidade de solicitar o FGTS para a rua 18, embora o guia complementar de documentos não tenha sido recuperado. Em Q8, apesar de o sistema identificar o abrigo que aceita animais, não apresentou corretamente as condições de acessibilidade, que estavam descritas na documentação.

A avaliação também apresentou duas ocorrências de faithfulness sem resultado (Q3 e Q6), associadas a avisos de geração incompleta do modelo-juiz. Portanto, essa média deve ser interpretada com cautela, e os resultados não devem ser considerados uma medição absoluta da qualidade do sistema.

**Melhoria proposta:**

Como o context recall do nível 2 foi de apenas 0,52 e Q5 e Q8 evidenciaram falhas na recuperação dos trechos necessários, a melhoria deve priorizar a recuperação em nível de trecho, e não apenas aumentar o número de documentos recuperados.

Para isso, propõe-se preservar os títulos das seções nos contextos enviados ao LLM, aprimorar a busca semântica com busca lexical complementar ou reranking e testar consultas que identifiquem explicitamente documentos de atualização. Também é necessário tornar as respostas mais objetivas, evitando informações adicionais que não foram solicitadas.

Por fim, recomenda-se estabilizar a configuração do modelo-juiz, ajustar o limite de geração para evitar avaliações incompletas e executar novamente o RAGAS para comparar os resultados antes e depois das melhorias.

## 8. Prints

### Interface Gradio com histórico e painel de fontes
![gradio](8_imagens/gradio.png)

### Coleção ChromaDB e busca com `distance`
![colecao](8_imagens/colecao.png)
![busca com distance](8_imagens/busca_distance.png)

### Tabela comparativa de chunking
![tabela_chunking](8_imagens/tabela_chunking.png)

### Trace do agente: thought → action → observation
![trace-1](8_imagens/trace-1.png)
![trace-2](8_imagens/trace-2.png)
![trace-3](8_imagens/trace-3.png)
![trace-4](8_imagens/trace-4.png)

### Cada guardrail disparando e os 19 testes passando
![guardrail_1](8_imagens/guardrail-1.png)
![guardrail_3](8_imagens/guardrail-2.png)
![guardrail_4](8_imagens/guardrail-3.png)
![guardrail_5](8_imagens/guardrail-4.png)
![guardrail_2](8_imagens/guardrail-5.png)
![testes](8_imagens/testes.png)

### Tabela de métricas do RAGAS
![Tabela de métricas do RAGAS](8_imagens/tab_metrica.png)

## 9. Limitações e custo

- Base fictícia e pequena (6 documentos); dados das tools simulam uma API do Centro de Operações e são mantidos à mão em sincronia com os textos.
- Guardrails por regex não cobrem paráfrases criativas; o juiz do RAGAS é um LLM e pode errar.
- Para produção: documentos oficiais reais com versionamento e vigência, API real de cotas e abrigos, LGPD (sem logs pessoais, base legal), infraestrutura redundante e offline-first, canal por SMS/WhatsApp, revisão humana das respostas críticas.
- **Custo:** RAG = 1 chamada por pergunta; agente = 1 a 7; guardrails = 0; RAGAS ≈ 8 perguntas × ~10 chamadas do juiz.

## 10. Uso de IA

Ver `6_Chat_Log.txt`. Resumo: IA ajudou a rascunhar a base documental (revisada e editada pelo grupo), o código e os guardrails. **Não** gerou o ground truth nem os resultados do RAGAS.

## 11. Estrutura do ZIP

```
GS2026_2_1CCR_GABIMARCOS.zip
├── 1_Notebook_GS2026_2.ipynb
├── 2_base_documental/  (doc_01 … doc_06)
├── 3_avaliacao_ragas.json
├── 4_README.md
├── 5_certificacao.txt
├── 6_Chat_Log.txt
├── 7_demo_video_url.txt
└── 8_imagens
```
