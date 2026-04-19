# 📌 Fluxo n8n — Check-In HPSM (Documentação)

## 🎯 Objetivo
Este fluxo realiza o **check-in de pacientes** no sistema, garantindo:

- Criação de paciente (task) caso não exista
- Criação de presença (subtask por data)
- Atualização da presença se já existir
- Controle de múltiplos serviços no mesmo dia

---

## 🔄 Visão Geral do Fluxo

```
Webhook → Edit Fields → Get User → Name Field → Get Tasks
→ Filter (paciente existe?)
  → IF:
    ✔ SIM:
        → Buscar presença
        → Verificar se já existe subtask (data)
            ✔ SIM:
                → Atualiza serviços da presença
            ❌ NÃO:
                → Cria nova presença
    ❌ NÃO:
        → Cria paciente
        → Cria presença (subtask)
```

---

## 📥 Entrada (Webhook)

### Endpoint:
```
POST /checkin-hpsm
```

### Body esperado:
```json
{
  "paciente": {
    "id": "868gc8paf",
    "name": "Nome do Paciente",
    "service": [
      {
        "id": "uuid-servico",
        "label": "HOSPITAL DIA"
      }
    ]
  },
  "tipoAtendimento": {
    "id": "uuid-servico",
    "label": "HOSPITAL DIA"
  },
  "data": "2026-03-18",
  "authorization": "Bearer TOKEN"
}
```

---

## 🧠 Lógica do Fluxo

### 1. 🔧 Edit Fields
Normaliza os dados:
- identificador → ID do paciente
- nome → Nome do paciente
- data → Data do atendimento
- serviços → lista de serviços
- usuário → token
- serviço escolhido

---

### 2. 👤 Get User ClickUp
Busca o nome do usuário autenticado via API do ClickUp.

---

### 3. 🏷️ Name Field
Extrai:
```
user.username
```

---

### 4. 📋 Get Many Tasks
Busca todas as tasks da lista (pacientes).

---

### 5. 🔍 Filter (Paciente existe?)
Compara:
```
custom_fields[identificador] == paciente.id
```

---

### 6. ⚖️ IF

#### ✔ Se paciente EXISTE:

---

### 6.1 📂 Buscar Presença
Busca task com subtasks incluídas.

---

### 6.2 ❓ Já existe presença?
Verifica se já existe subtask com nome:

```
dd/MM/yyyy
```

---

### ✔ Se JÁ EXISTE:

#### Fluxo:
```
Split subtasks → Filtra pela data → Detalha task
→ Extrai serviços → Adiciona novo serviço → Atualiza
```

#### 🧠 Regra importante:
Evita duplicação de serviços:

```js
Array.from(new Set([...existentes, novo]))
```

---

### ❌ Se NÃO EXISTE:

→ Cria nova subtask (presença)

---

#### ❌ Se paciente NÃO EXISTE:

---

### 7. 🆕 Criar Paciente

Cria task com:
- Nome
- Identificador
- Outros campos

---

### 8. ➕ Criar Presença

Cria subtask com:
- Nome = data formatada
- Serviço selecionado
- Usuário

---

## 🧩 Estrutura no ClickUp

### Task (Paciente)
- Nome = Nome do paciente
- Campo custom: Identificador

---

### Subtask (Presença)
- Nome = Data (dd/MM/yyyy)
- Campo custom: Serviços (array)
- Campo custom: Usuário responsável

---

## 💡 Possíveis Evoluções

- Cache de pacientes
- Integração direta com Spincare
- Dashboard de presença em tempo real
- Logs estruturados
- Retry automático
