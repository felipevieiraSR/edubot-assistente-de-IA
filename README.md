# 🤖 EduBot — Autonomous Educational AI Assistant

[![Live Demo](https://img.shields.io/badge/Demo-Vercel%20Live-brightgreen?style=for-the-badge&logo=vercel)](https://edubot-phi.vercel.app)
[![OpenAI](https://img.shields.io/badge/LLM-GPT--4o--mini%20%2B%20Embeddings-412991?style=for-the-badge&logo=openai&logoColor=white)](https://openai.com/)
[![Supabase pgvector](https://img.shields.io/badge/Vector%20DB-Supabase%20pgvector%20(HNSW)-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com/)
[![n8n](https://img.shields.io/badge/Orchestrator-n8n%20Workflows-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENSE)

> **Solução integrada de Agente de IA com RAG (Retrieval-Augmented Generation), orquestração orientada a eventos via Webhooks no n8n e interface web interativa em produção.**

---

## 📌 Visão Geral

O **EduBot** é um assistente virtual inteligente projetado para atuar como tutor e suporte acadêmico na plataforma educacional **DevAcademy** (abrangendo bootcamps de Front-End React e Back-End Node.js). 

A solução resolve o problema de sobrecarga de atendimento em plataformas educacionais, combinando:
1. **Recuperação de Informação com Grounding (RAG)** sobre documentos regulatórios e ementas oficiais, eliminando alucinações.
2. **Arquitetura de Microsserviços Desacoplada** via Webhook HTTP no n8n.
3. **Guardrails Pedagógicos** no System Prompt que orientam o estudante sem resolver códigos diretamente (incentivo ao aprendizado ativo).
4. **Interface Web Intuitiva** inspirada no robô WALL-E, com suporte a sessões contextuais e testes em tempo real.

🔗 **Aplicação em Produção (Vercel):** [https://edubot-phi.vercel.app](https://edubot-phi.vercel.app)

---

## 🏗️ Arquitetura da Solução

O sistema adota uma arquitetura orientada a microsserviços, desacoplando completamente a camada de apresentação do motor de inferência e da base de dados vetorial:

```mermaid
flowchart TD
    subgraph CLIENTE["1. Camada de Apresentação (Frontend)"]
        UI["Interface Web EduBot<br>(index.html + WALL-E UI)"] -->|POST JSON<br>{ message, session_id }| WH
    end

    subgraph ORQUESTRACAO["2. Microsserviço de Orquestração (n8n Cloud)"]
        WH["Webhook POST /webhook/chat<br>(CORS Enabled: *)"] --> EF["Edit Fields<br>(Sanitização e normalização de chaves)"]
        EF --> AGENT["AI Agent Core<br>(LangChain + OpenAI gpt-4o-mini)"]
        AGENT --> RESP["Respond to Webhook<br>(HTTP 200 + { output: string })"]
        RESP -->|Resposta JSON| UI
        MEM[("Buffer Window Memory<br>(Contexto indexado por session_id)")] <--> AGENT
    end

    subgraph RETRIEVAL["3. Base Vetorial e Mecanismo RAG (Supabase)"]
        AGENT <-->|"Tool: Base_de_Conhecimento<br>(RPC match_documents)"| VSTORE[("Supabase PostgreSQL<br>Extension pgvector + HNSW")]
        EMBED["OpenAI Embeddings<br>(text-embedding-3-small / 1536d)"] -.-> VSTORE
    end

    subgraph ETL["4. Pipeline Automatizado de Ingestão (ETL)"]
        TRIGGER["Trigger de Ingestão"] --> GH["Download do Catálogo de Documentos"]
        GH --> HTTP["Download de Arquivos Regulatórios (PDF)"]
        HTTP --> EXTRACT["Extração de Texto"]
        EXTRACT --> CHUNK["Text Splitting (70 chunks estruturados)"]
        CHUNK --> VSTORE
    end
```

---

## ⚡ Destaques de Engenharia

### 1. RAG com Índice HNSW no Supabase (pgvector)
- **Embeddings:** `text-embedding-3-small` (1536 dimensões) da OpenAI.
- **Indexação de Alta Performance:** Utilização do algoritmo **HNSW (Hierarchical Navigable Small World)** configurado com `vector_cosine_ops`, garantindo latência de consulta sub-milissegundo para busca por similaridade de cosseno.
- **Função RPC (`match_documents`):** Procedimento PL/pgSQL encapsulado no banco de dados para filtrar metadados e retornar os top-$K$ fragmentos contextuais mais relevantes.

### 2. Guardrails e Diretrizes Pedagógicas
O agente possui políticas de resposta estritas injetadas via System Prompt:
- **Proteção Didática:** O EduBot é terminantemente proibido de fornecer soluções prontas ou corrigir trechos inteiros de código. Em vez disso, formula perguntas socráticas e direciona o estudante para mentorias 1:1.
- **Isonomia Acadêmica:** Bloqueio consistente de pedidos individuais de prorrogação de prazo, citando o edital institucional.
- **Triagem Ativa:** Identificação de dúvidas ambíguas (ex.: solicita confirmação da trilha: Front-End ou Back-End).
- **Encaminhamento de Incidentes:** Coleta padronizada de e-mail e abertura de chamados técnicos para bugs ou erros da plataforma (ex.: erro 500 no upload).

### 3. Memória Conversacional por Sessão
- Histórico mantido via **Buffer Window Memory** indexado pelo `session_id`.
- Permite diálogos contínuos com referências contextuais ("E quanto ao outro bootcamp?", "Quando encerra o projeto dele?") sem perda de contexto entre requisições HTTP desacopladas.

---

## 📁 Estrutura do Repositório

```text
├── .github/                       # Configurações de CI/CD (opcional)
├── data/
│   └── knowledge_base_catalog.json# Catálogo com a relação de documentos indexados
├── docs/
│   └── academic/                  # Documentos históricos da concepção acadêmica
├── sql/
│   └── supabase_setup.sql         # DDL: extensão pgvector, tabela documents, HNSW e RPC
├── tests/
│   ├── test_webhook.py            # Suite de testes de integração em Python (requests)
│   └── test_webhook.ps1           # Script de teste em PowerShell para automação Windows
├── workflows/
│   ├── workflow_edubot.json       # Workflow principal exportado do n8n (Agent + RAG + Webhook)
│   └── workflow_edubot_rag_webhook.json # Versão base do pipeline
├── index.html                     # Frontend moderno da aplicação consumidora (Single-Page)
├── avatar_walle.svg               # Assets vetoriais do avatar do mascote
├── mascot_walle.svg               # Ilustração completa do WALL-E
├── vercel.json                    # Configuração de rotas e clean URLs para Vercel
├── .env.example                   # Modelo de variáveis de ambiente
├── .gitignore                     # Filtros de arquivos ignorados
├── LICENSE                        # Licença MIT
└── README.md                      # Documentação técnica do projeto
```

---

## 🚀 Como Executar Localmente

### Pré-requisitos
- Conta no [Supabase](https://supabase.com/) com PostgreSQL habilitado.
- Instância do [n8n](https://n8n.io/) (Cloud ou Self-hosted via Docker).
- Chave de API da [OpenAI](https://platform.openai.com/).
- Python 3.9+ (opcional, para execução da suíte de testes).

---

### Passo 1: Configurar o Banco Vetorial (Supabase)
1. Acesse o **SQL Editor** no painel do Supabase.
2. Execute o script contido em [`sql/supabase_setup.sql`](sql/supabase_setup.sql).
3. O script irá:
   - Ativar a extensão `vector`.
   - Criar a tabela `documents` com suporte a metadados e vetor dimensional de 1536 posições.
   - Criar o índice `HNSW` com `vector_cosine_ops`.
   - Definir a stored procedure `match_documents` para consultas semânticas.

---

### Passo 2: Importar e Ativar o Workflow no n8n
1. No painel do seu n8n, navegue até **Workflows** > **Import from File**.
2. Selecione o arquivo [`workflows/workflow_edubot.json`](workflows/workflow_edubot.json).
3. Configure as credenciais dos nós:
   - **OpenAI:** Inserir `OPENAI_API_KEY`.
   - **Supabase:** Inserir a URL do projeto e a Service Role Key.
4. Alterne o status do workflow para **Active (ON)**.
5. Copie a URL do nó de **Webhook** (`/webhook/chat`).

---

### Passo 3: Executar a Interface Web
Você pode abrir o arquivo `index.html` diretamente em qualquer navegador moderno:
1. Abra o arquivo `index.html` no seu navegador favorito.
2. Na interface do chat, clique no ícone de **Configurações/Webhook** no canto superior direito para apontar para a sua instância do n8n (ou use o endpoint já configurado por padrão).
3. Opcionalmente, faça o deploy com um clique na **Vercel**:
   ```bash
   npx vercel
   ```

---

### Passo 4: Executar os Testes Automatizados via Terminal

Para validar a integridade da comunicação HTTP e a resposta do agente com Python:

```bash
# Instalar dependências de teste
pip install requests

# Executar a bateria de testes apontando para o webhook de produção
python tests/test_webhook.py

# Ou passando uma URL customizada:
python tests/test_webhook.py "http://localhost:5678/webhook/chat"
```

Alternativamente, no Windows via PowerShell:
```powershell
.\tests\test_webhook.ps1
```

---

## 🧪 Casos de Uso e Validação

| Cenário de Teste | Entrada do Aluno | Comportamento Observado do EduBot |
| :--- | :--- | :--- |
| **1. Prazo Oficial** | *"Qual a data limite de entrega do Bootcamp React?"* | Consulta a base vetorial via RAG e retorna: **05/12/2026 às 23:59**. |
| **2. Pergunta Incompleta** | *"Qual o prazo de entrega do projeto?"* | Triagem ativa: questiona se o aluno está no Bootcamp Front-End ou Back-End. |
| **3. Solicitação de Solução** | *"Corrige esse bug de sintaxe no meu código?"* | **Bloqueio didático:** recusa com empatia pedagógica e orienta agendamento de mentoria 1:1. |
| **4. Pedido de Prorrogação** | *"Pode adiar meu prazo por 3 dias?"* | Negação institucional amigável, garantindo a isonomia entre os alunos. |
| **5. Incidente Técnico** | *"Erro 500 ao enviar o arquivo na plataforma."* | Coleta o e-mail do aluno e abre protocolo exclusivo para a equipe de TI. |
| **6. Memória de Conversa** | Perguntas sequenciais utilizando pronomes | Responde mantendo o encadeamento histórico via `session_id`. |

---

## 🛠️ Tecnologias Utilizadas

- **Inteligência Artificial:** [OpenAI](https://openai.com/) (GPT-4o-mini & Text-Embedding-3-small), LangChain / AI Agent v2.
- **Banco de Dados Vetorial:** [Supabase](https://supabase.com/) (PostgreSQL 15+, extensão `pgvector`, índice HNSW).
- **Orquestração & Automação:** [n8n](https://n8n.io/) (Workflows, Webhooks, Buffer Memory).
- **Frontend:** HTML5 semântico, CSS3 (variáveis, Glassmorphism, animações responsivas), JavaScript Moderno (Fetch API, Session Storage).
- **Hospedagem & Deploy:** [Vercel](https://vercel.com/) (Edge Network com Clean URLs).
- **Testes & Scripts:** Python 3, PowerShell.

---

## 👨‍💻 Autor

**Felipe Vieira Saliba Rebouças**
- 💼 LinkedIn: [linkedin.com/in/felipesaliba](https://linkedin.com/in/felipesaliba) *(atualize com seu link)*
- 🐙 GitHub: [@felipesaliba5-design](https://github.com/felipesaliba5-design)
- 🌐 Portfólio: [edubot-phi.vercel.app](https://edubot-phi.vercel.app)

---

## 📄 Licença

Este projeto está sob a licença [MIT](LICENSE). Consulte o arquivo para obter mais informações.
