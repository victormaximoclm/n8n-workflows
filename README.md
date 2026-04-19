# N8n workflows

Repositório responsável por centralizar **workflows de automação (n8n)** e suas respectivas documentações.

---

## 📁 Estrutura do Projeto

```
.
├── workflows/   # Fluxos n8n (JSON exportado)
├── docs/        # Documentação detalhada de cada workflow
└── README.md    # Este arquivo
```

---

## ⚙️ Sobre o Projeto

Este repositório contém automações utilizadas nas operações de cliente reais, incluindo:

- Check-in de pacientes
- Controle de presença
- Geração de documentos (PDF)
- Integrações com ClickUp
- Integrações com Google Drive / Sheets
- Chatbots

Todos os fluxos são construídos utilizando **n8n** e seguem um padrão de organização para facilitar manutenção e escalabilidade.

---

## 🔄 Como funciona

Cada automação segue o padrão:

1. **Entrada (Webhook ou Trigger)**
2. **Processamento de dados**
3. **Integração com APIs externas**
4. **Persistência / Atualização**
5. **Resposta ou ação final**

---

## 📂 Workflows

Os arquivos dentro da pasta `/workflows` são exportações diretas do n8n.

### Padrão de nomenclatura

```
[contexto]-[ação].json
```

Exemplos:

- `checkin-hpsm.json`
- `impressao-presenca.json`

---

## 📄 Documentação

Cada workflow possui um arquivo correspondente dentro da pasta `/docs`.

### Estrutura esperada:

```
docs/
├── checkin-hpsm.md
├── impressao-presenca.md
```

### Conteúdo da documentação:

- Objetivo do fluxo
- Etapas detalhadas
- Estrutura dos dados
- Integrações utilizadas
- Pontos de atenção
- Possíveis melhorias

---

## 🚀 Como usar

### 1. Importar workflow no n8n

- Acesse o n8n
- Vá em **Import**
- Cole ou faça upload do JSON localizado em `/workflows`

---

### 2. Configurar credenciais

Os fluxos dependem de:

- ClickUp OAuth2
- Google Drive OAuth2
- Google Sheets OAuth2

Configure corretamente antes de executar.

---

### 3. Ativar workflow

Após importar:

- Ajuste variáveis se necessário
- Ative o fluxo

---

## 🧩 Integrações utilizadas

- ClickUp API
- Google Drive API
- Google Sheets API
- Webhooks (entrada de dados externos)

---

## 📌 Boas práticas adotadas

- Separação entre lógica e documentação
- Uso de Set Nodes para organização de dados
- Reutilização de templates (Google Sheets)
- Padronização de nomes de campos
- Limpeza de arquivos temporários
