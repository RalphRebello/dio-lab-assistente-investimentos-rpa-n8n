# Documentação Técnica — Assistente de Investimentos (RPA + n8n)

Este documento descreve o que **está efetivamente implementado** neste repositório. Para o enunciado original do desafio, veja [README.md](README.md).

## 1. Visão Geral

O projeto automatiza a geração de mensagens de recomendação de investimento para clientes, combinando:

1. **RPA em Python** — extrai dados de clientes de uma página HTML via web scraping.
2. **n8n** — recebe os dados via webhook, cruza com uma tabela de produtos de investimento e gera uma mensagem personalizada por cliente.
3. **GitHub Pages** — hospeda os dois arquivos de dados consumidos pelo pipeline (`docs/index.html` e `docs/data.csv`).

Não há, até o momento, integração com um agente de IA generativa (LLM) no n8n — a geração de mensagem é feita por template em JavaScript (nó `Code`), não pela IA. Essa etapa é a extensão opcional sugerida no README.

## 2. Arquitetura Implementada

```
docs/index.html (clientes, GitHub Pages)
        │  scraping (requests + BeautifulSoup)
        ▼
rpa/extrair_clientes.ipynb
        │  POST JSON { clientes: [...] }
        ▼
n8n: Webhook (Info Clientes)  ──┬──► Merge (Combinar dados de cliente e investimentos)
                                  │              ▲
docs/data.csv (investimentos) ───┘              │
        │  GET (raw.githubusercontent.com)       │
        ▼                                        │
HTTP Request (Investiment type) ──► Code (Normalizar dados CSV) ──┘
                                                  │
                                                  ▼
                              Code "(Respostas para clientes)"
                              (cruza perfil × produto, monta mensagem)
                                                  │
                                                  ▼
                              Code "Normalizar Dados finais"
                              (limpeza, dedupe por e-mail, HTML da mensagem)
                                                  │
                                                  ▼
                              Respond to Webhook (JSON)
```

## 3. Componentes

### 3.1 Dados de clientes — `docs/index.html`

Página estática hospedada no GitHub Pages com uma tabela `#clientes` (`nome`, `email`, `saldo`, `perfil`). Contém 10 clientes fictícios com perfis `Conservador`, `Moderado` ou `Arrojado`.

### 3.2 Dados de investimentos — `docs/data.csv`

CSV com colunas `perfil,produto,minimo,rentabilidade`, 3 produtos por perfil (9 linhas de dados). Define o valor mínimo de aplicação e a rentabilidade de cada produto.

### 3.3 Script de RPA — `rpa/extrair_clientes.ipynb`

- Usa `requests` para baixar `docs/index.html` e `BeautifulSoup` para extrair as linhas de `#clientes tbody tr`.
- Monta uma lista de dicionários `clientes` e imprime o resultado.
- Envia a lista via `POST` para o webhook do n8n:
  ```python
  N8N_WEBHOOK = "http://homeserver.local:5678/webhook-test/clientes"
  requests.post(N8N_WEBHOOK, json={"clientes": clientes})
  ```
  > **Atenção:** a URL atual (`homeserver.local`) é um endpoint local/de teste. Para outro ambiente (n8n Cloud ou outra instância local), substitua pela URL real do nó *Webhook*.

### 3.4 Workflow n8n — `n8n/lab-rpa-n8n.json`

| Nó | Tipo | Função |
|----|------|--------|
| **Webhook (Info Clientes)** | `webhook` (POST `/clientes`) | Recebe `{ clientes: [...] }` do script Python. |
| **HTTP Request (Investiment type)** | `httpRequest` | Baixa `docs/data.csv` direto do GitHub (raw.githubusercontent.com). |
| **Code in JavaScript (Normalizar dados)** | `code` | Faz o parse manual do CSV (sem biblioteca), converte `minimo` para número. |
| **Merge (Combinar dados de cliente e investimentos)** | `merge` (`combineByPosition`) | Junta, no mesmo item, o payload do webhook com a lista de investimentos normalizada. |
| **(Respostas para clientes)** | `code` | Para cada cliente: converte saldo em número, filtra produtos elegíveis (perfil compatível + saldo ≥ mínimo) e monta a mensagem de recomendação com template por perfil. |
| **Normalizar Dados finais** | `code` | Limpa nomes/e-mails, remove duplicados por e-mail, valida formato de e-mail, converte a mensagem em HTML (`<p>`/`<br>`) e formata o saldo em `R$`. |
| **Respond to Webhook** | `respondToWebhook` | Retorna o array final de clientes processados como JSON (HTTP 200). |

**Regras de negócio implementadas no nó "(Respostas para clientes)":**
- Um produto só é recomendado se `produto.perfil === cliente.perfil` **e** `cliente.saldo >= produto.minimo`.
- Produtos do mesmo perfil que o cliente ainda não atinge o mínimo aparecem como "disponível ao atingir o valor mínimo de X".
- Se nenhum produto for elegível, é enviada uma mensagem incentivando o cliente a começar a poupar.
- Mensagens têm textos de abertura/fechamento diferentes por perfil (Conservador / Moderado / Arrojado).

### 3.5 Estrutura real do repositório

```
dio-lab-assistente-investimentos-rpa-n8n/
├── README.md                     # Enunciado do desafio (DIO)
├── DOCUMENTACAO.md                # Este documento
├── docs/
│   ├── index.html                 # Clientes fictícios (fonte do scraping)
│   └── data.csv                   # Produtos de investimento por perfil
├── n8n/
│   └── lab-rpa-n8n.json           # Workflow n8n exportado
└── rpa/
    └── extrair_clientes.ipynb     # Notebook de extração (RPA) + envio ao webhook
```

> Nota: o README descreve a estrutura esperada com `src/` e `n8n/workflow.json`; o repositório atual usa `rpa/extrair_clientes.ipynb` e `n8n/lab-rpa-n8n.json`.

## 4. Como Executar

1. **Publicar os dados** (já feito via GitHub Pages, se o fork mantiver `docs/`): confirme que `index.html` e `data.csv` estão acessíveis publicamente.
2. **Importar o workflow no n8n**: `n8n/lab-rpa-n8n.json` → ative o workflow e copie a URL gerada pelo nó *Webhook (Info Clientes)*.
3. **Ajustar o notebook**: em `rpa/extrair_clientes.ipynb`, defina `N8N_WEBHOOK` com a URL copiada no passo anterior.
4. **Ajustar a URL do CSV** no nó *HTTP Request (Investiment type)* caso o fork tenha outro usuário/branch (atualmente aponta para `RalphRebello/dio-lab-assistente-investimentos-rpa-n8n`, branch `main`).
5. **Executar o notebook** (Jupyter local ou Google Colab) — ele extrai os clientes e dispara o POST para o n8n.
6. O n8n processa e retorna o JSON com as mensagens personalizadas (visível na execução do workflow ou na resposta HTTP do webhook).

## 5. Limitações e Próximos Passos

- **Sem IA generativa**: a geração de mensagem é 100% baseada em templates fixos em JavaScript — a etapa "Desafio Completo" do README (conectar um Agente de IA/LLM no n8n) ainda não foi implementada.
- **Webhook apontando para endpoint local** (`homeserver.local`): precisa ser trocado antes de rodar em outro ambiente.
- **Parsing de CSV manual**: o nó de normalização faz split por vírgula sem suportar campos com vírgula/aspas — adequado apenas para o CSV simples atual.
- **Sem testes automatizados** cobrindo o workflow ou o script de extração.
