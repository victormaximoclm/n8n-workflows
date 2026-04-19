# 🤖 Chatbot de Orçamento — Cria Orçamento do Zero

> Sub-fluxo n8n responsável por coletar, validar, confirmar e processar todas as informações necessárias para gerar um orçamento completo via WhatsApp, criar uma tarefa no ClickUp e gerar o PDF final.

---

## 📌 Sobre o Projeto

Este fluxo é a **continuação direta** do fluxo principal **"Chatbot de Orçamento"**. Ele é acionado como sub-workflow (via `Execute Workflow`) e cuida de toda a jornada conversacional com o usuário no WhatsApp, desde a coleta dos dados do evento até a entrega do PDF de orçamento.

A comunicação é feita pela **Evolution API** (WhatsApp), os dados são persistidos no **PostgreSQL** e a tarefa final é criada no **ClickUp**.

---

## 🗺️ Fluxo Geral

```
Início (Inform.)
    │
    ▼
[Switch] — roteamento por etapa da conversa
    ├── Etapa < 9   → Coletando INFO. (perguntas sequenciais)
    ├── Etapa = 9   → Confirmação INFO. (resumo para o usuário confirmar)
    ├── Etapa = 10  → Editar Campo (usuário corrige um campo específico)
    └── Etapa = 11  → Processando INFO. (criação da tarefa e geração do PDF)
```

---

## 🔁 Etapas da Conversa

| Etapa | Campo coletado | Validação |
|-------|---------------|-----------|
| 1 | Opção inicial (1, 2 ou 3) | `options_1_2_3` |
| 2 | Nome do evento | `text` |
| 3 | Endereço | `text` |
| 4 | Data do evento | `date` (dd/MM/yyyy) |
| 5 | Horário (início - fim) | `time` (hh:mm - hh:mm) |
| 6 | Tipo de suporte | `enum` (Básico / Avançado / Ambos) |
| 7 | Validade da proposta | `prazo` (em dias) |
| 8 ou 12/13 | Valor(es) do suporte | `number` (moeda BRL) |
| 9 | Confirmação de dados | `yesno` |
| 10 | Edição de campo específico | formato `N - Novo valor` |
| 11 | Processamento final | — |

> Quando o tipo de suporte é **"Ambos"**, o fluxo coleta separadamente o valor do suporte básico (mensagem_12) e do suporte avançado (mensagem_13).

---

## ✅ Validações Implementadas

Cada resposta do usuário passa por um nó de código JavaScript antes de ser aceita:

- **`text`** — colapsa espaços, corrige plural/singular, capitaliza a primeira letra
- **`date`** — valida calendário real (aceita `dd/MM/yyyy` e `dd-MM-yyyy`), rejeita datas passadas quando aplicável
- **`time`** — valida formato `hh:mm - hh:mm`, rejeita horários iguais
- **`enum:basico,avancado`** — normaliza para `Básico`, `Avançado` ou `Ambos` com detecção inteligente de variações
- **`number`** — parseia valores monetários em vários formatos e normaliza para `R$ X.XXX,XX`
- **`prazo`** — extrai número de dias e formata (`1 dia` / `N dias`)
- **`yesno`** — aceita variações de sim/não em português
- **`options_1_7`** — valida edição de campo no formato `N - valor`

Se a resposta for inválida, o bot envia uma mensagem de erro e aguarda nova tentativa.

---

## 🏗️ Arquitetura dos Nós Principais

### Coleta de Dados
- **`Validador de resposta`** → prepara validator e resposta para verificação
- **`Verifica respostas`** → executa a lógica de validação por tipo
- **`Aceita?`** → IF: resposta válida segue; inválida volta ao usuário
- **`Envia Pergunta Atual`** → envia a pergunta da próxima etapa via Evolution API
- **`Salva estado da conversa`** → persiste a resposta e atualiza a etapa no PostgreSQL

### Confirmação & Edição
- **`Confirma Informações`** / **`Confirma Informações BA`** — envia resumo completo ao usuário
- **`Sim ou Nao`** — roteamento após confirmação
- **`Pergunta qual campo quer editar`** — solicita campo no formato `N - valor`
- **`Verifica Resposta 1/2 Sup`** — valida edições de campo com lógica específica por número

### Criação da Tarefa no ClickUp
- **`Pega todas as info.`** — carrega todos os dados do PostgreSQL
- **`Get many tasks`** → **`Filter`** → **`Gera ID novo`** — gera ID sequencial no formato `YYYYMM-NNN`
- **`Create a task1`** — cria a tarefa no ClickUp
- **`array de campos`** / **`array de campos 2 SUP`** — monta array de custom fields
- **`Loop Over Items`** → **`Update Custom Fields`** — atualiza campos customizados via API do ClickUp
- **`If3`** — roteamento entre suporte simples e "Ambos"

### Geração do PDF
- **`Avisa que Tarefa foi criada no Click Up`** — notifica o usuário
- **`Pega ID do PDF`** — captura o `task_id` da tarefa criada
- **`Gera PDF`** — chama sub-workflow de geração de PDF
- **`Avisa que Tarefa foi criada no Click Up1`** — envia o PDF via `sendMedia`
- **`Remove Histórico de conversa`** — limpa o registro no PostgreSQL ao finalizar

---

## 🗄️ Banco de Dados (PostgreSQL)

**Tabela:** `conversas_chatbot`

| Coluna | Descrição |
|--------|-----------|
| `conversa_id` | Identificador baseado no número do WhatsApp (12 dígitos) |
| `etapa` | Etapa atual da conversa (0–11) |
| `mensagem_1` a `mensagem_13` | Respostas coletadas em cada etapa |
| `data_modificacao` | Timestamp da última interação |

---

## 🔧 Integrações

| Serviço | Uso |
|---------|-----|
| **Evolution API** | Envio e recebimento de mensagens WhatsApp |
| **PostgreSQL** | Persistência do estado da conversa |
| **ClickUp** | Criação de tarefas e atualização de campos customizados |
| **Sub-workflow "Gera PDF"** | Geração do documento PDF do orçamento |

---

## ⚙️ Configuração

As seguintes variáveis são passadas pelo fluxo pai via `Inform.` (Execute Workflow Trigger):

```json
{
  "instance": "nome-da-instancia-evolution",
  "apikey": "sua-api-key-evolution",
  "remoteJid": "5511999999999@s.whatsapp.net",
  "wait": "3.2",
  "etapa": "1",
  "data": [ /* array com conteúdo e validators de cada etapa */ ]
}
```

---

## 📦 Dependências

- n8n (self-hosted ou cloud)
- Evolution API v2+
- PostgreSQL 13+
- Credenciais ClickUp (API Key)
- Sub-workflow: **Gera PDF** (`fq5ymcybLFH9BCOT`)

---

## 🔗 Fluxos Relacionados

- **Chatbot de Orçamento** *(fluxo pai)* — responsável por receber os webhooks do WhatsApp e acionar este sub-fluxo
- **Gera PDF** — sub-workflow chamado na etapa final para gerar o documento

---

## 📄 Licença

Projeto interno — Simplifica Gestão.
