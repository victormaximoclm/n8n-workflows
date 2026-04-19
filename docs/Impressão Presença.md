# 📄 Documentação - Fluxo "Impressão Presença"

## 🎯 Objetivo
Este fluxo gera automaticamente um PDF de frequência do paciente a partir de uma tarefa do ClickUp e anexa o arquivo gerado na própria tarefa.

---

## 🔁 Visão Geral do Fluxo

1. Recebe webhook do ClickUp
2. Busca dados da tarefa
3. Organiza informações do paciente
4. Duplica planilha modelo no Google Drive
5. Preenche dados na planilha
6. Converte para PDF
7. Anexa PDF na tarefa
8. Exclui arquivo temporário

---

## ⚙️ Etapas Detalhadas

### 1. Webhook
- Endpoint: `/imprimeCheckin`
- Método: `POST`
- Recebe payload do ClickUp contendo o ID da tarefa

---

### 2. Pega Tarefa (ClickUp)
- Busca detalhes completos da tarefa usando:
```
$json.body.payload.id
```

---

### 3. Organiza Info (Set Node)

Extrai e organiza os seguintes dados:

- **Nome do paciente**
- **Diárias autorizadas**
- **Número da guia**
- **Convênio (label)**
- **Serviços (IDs)**
- **Serviços formatados (string)**

#### Lógica importante:
Transforma IDs dos serviços em nomes:

```javascript
return field.value
  .map(val => field.type_config.options.find(opt => opt.id === val)?.label)
  .filter(Boolean)
  .join(", ");
```

---

### 4. Copy File (Google Drive)

- Duplica planilha modelo:
```
MODELOS DE FREQUENCIA2026
```

- Nome do novo arquivo:
```
Frequência - {{ primeiro_nome }}
```

---

### 5. Atualiza File Presenças (HTTP Request)

Atualiza células da planilha:

| Campo | Valor |
|------|------|
| B3 | Nome do paciente |
| B4 | Serviços |
| B5 | Convênio |
| B6 | Número da guia |
| F6 | Diárias |
| B7 | Mês atual |
| E7 | Ano atual |

---

### 6. Download File

- Converte planilha para **PDF**

---

### 7. Anexa a tarefa (ClickUp)

- Endpoint:
```
POST /task/{task_id}/attachment
```

- Envia o PDF como arquivo binário

---

### 8. Delete a file

- Remove arquivo temporário do Google Drive
- Evita acúmulo de arquivos

---

### 9. Resposta Final

Retorna:

```json
{
  "Response": "Success"
}
```

---

## 🧠 Pontos Inteligentes do Fluxo

### ✔️ Conversão dinâmica de serviços
Transforma múltiplos IDs em texto legível automaticamente.

### ✔️ Uso de template
Evita recriar estrutura de documento toda vez.

### ✔️ Limpeza automática
Arquivo temporário é apagado após uso.

---

## 🧩 Dependências

- Google Drive API
- Google Sheets API
- ClickUp API
- Credenciais OAuth2 configuradas no n8n

Se quiser, posso te devolver isso em versão **mais enxuta (pra documentação interna)** ou **mais técnica (pra devs)**.
