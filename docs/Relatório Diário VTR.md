# 🚑 Relatório Diário VTR Form

> Fluxo n8n responsável por coletar, validar e consolidar os checklists diários das VTRs (veículos) e das USA/USB (unidades de suporte avançado), enviando um relatório automático via WhatsApp com o status de cada ambulância.

---

## 📌 Sobre o Projeto

Este fluxo monitora o preenchimento dos formulários de checklist das ambulâncias, detecta itens fora do padrão e envia relatórios consolidados para um grupo do WhatsApp. Ele opera de duas formas complementares:

- **Agendado:** dispara às 9h todos os dias, lendo as respostas dos Google Forms do dia
- **Via Webhook:** acionado em tempo real sempre que um novo formulário é submetido (VTR, USA ou USB), gerando um relatório parcial imediato

---

## 🗺️ Fluxo Geral

```
[Schedule Trigger 9h] ──────┐
                             ├──► Google Sheets (VTR + USA + USB)
[Webhook VTR] ──────────────┤       │
[Webhook USA] ──────────────┤       ▼
[Webhook USB] ──────────────┘   Filtra registros do dia
                                     │
                                     ▼
                             Verifica Campos Fora do Padrão
                                     │
                            ┌────────┴────────┐
                         Aprovado          Reprovado
                            │                  │
                            └────────┬─────────┘
                                     ▼
                             Organiza Informações
                                     │
                                     ▼
                             Junta e Mescla (por número da VTR)
                                     │
                                     ▼
                             Organiza Mensagem
                                     │
                                     ▼
                         Enviar Mensagem (WhatsApp)
```

---

## ⚙️ Triggers

### 1. Schedule Trigger (9h diário)
Dispara automaticamente às 9h, busca **todos** os registros do dia corrente nas três planilhas e gera um relatório completo das VTRs que preencheram o checklist.

### 2. Webhooks em Tempo Real
Três endpoints distintos recebem os dados assim que o formulário é enviado:

| Endpoint | Formulário | Responsável |
|----------|-----------|-------------|
| `/webhook/VTR` | Checklist do veículo | Condutor |
| `/webhook/USA` | Checklist médico USA | Enfermeiro |
| `/webhook/USB` | Checklist médico USB | Técnico de Enfermagem |

Ao receber um webhook, o fluxo lê também as outras planilhas do dia para montar um relatório parcial consolidado.

---

## ✅ Validações por Tipo de Checklist

### Checklist VTR (veículo)
Campos verificados: nível de óleo, água do radiador, combustível, pressão dos pneus, extintor, limpeza geral, amortecedores, freios, e outros 20 itens mecânicos.

Valores aceitos: `Ok`, `Bom`, `Normal`, `Cheio`, `Acima da metade`, `Sem Vazamento`, `Não tem ruídos`, `VTR Limpa`, `Dentro da Validade`, `Temperatura dentro da faixa normal`, `Sem fumaça anormal`, `Alto`, `Presente`

> Para VTRs USA 10, USA 11, USB 10 e USB 11: também verifica o campo **NIVEL DE ADBLUE**.

### Checklist USA (Unidade de Suporte Avançado)
Mais de 40 campos verificados, incluindo medicações, equipamentos de oxigênio, materiais de infusão, EPIs, mochilas de trauma, cardioversor, ventilador, bombas de infusão e limpeza concorrente.

Valores aceitos: `SIM`, `Acima de 100 libras`, `Acima de 50 libras`, `Todos PRESENTES`, `LACRADO`

### Checklist USB (Unidade de Suporte Básico)
Mais de 75 campos verificados, incluindo prancha adulto/infantil, oxigênio, DEA, kit PA, materiais descartáveis, EPIs, mochila de trauma, mochila via aérea, ambús e limpeza concorrente.

Valor aceito: `SIM`

---

## 🔧 Lógica de Normalização de Campos

Todos os nós de verificação utilizam uma função de normalização robusta que:
- Remove caracteres invisíveis (zero-width, BOM)
- Remove acentos (NFD)
- Substitui qualquer sequência não alfanumérica por espaço
- Colapsa espaços e converte para UPPERCASE

Isso garante que variações de digitação nos nomes dos campos do formulário não quebrem a verificação.

---

## 📊 Agrupamento e Consolidação

O nó **Junta e Mescla** agrupa os dados por número da VTR (extraído do nome, ex: `VTR 09` → número `9`) e combina:

- **Checklist VTR** (condutor) + **Checklist USA/USB** (enfermeiro/técnico) → mesmo grupo
- Pendências de cada parte são mantidas separadas (`pendencias_vtr` e `pendencias_usa_usb`)
- Campos fora do padrão também são separados por origem

O nó detecta ainda:
- **Checklist incompleto**: apenas VTR ou apenas USA/USB preenchido
- **Preenchimento duplicado**: mais de um checklist VTR ou mais de um USA/USB para o mesmo número

---

## 📱 Formato da Mensagem WhatsApp

A mensagem final enviada ao grupo segue este modelo por VTR:

```
🚑 *VTR:* 09

🛠️ *Checklist VTR (Condutor: Cleyton)*
✅ Dentro do padrão
📌 Observações: Para brisa trincado

🏥 *Checklist USA/USB (Enfermeiro(a): Thyago 336.739)*
🔎 2 item(s) fora do padrão:
• MOCHILA DE TRAUMA: NÃO
• EXTINTOR: Vencido
```

Alertas de duplicação ou preenchimento duplo também são incluídos ao final.

---

## 🏗️ Arquitetura dos Nós Principais

### Leitura e Filtragem
- **Get VTR / Get USA / Get USB** → leem as planilhas do Google Sheets
- **Filtra Data de Hoje** → mantém apenas registros com carimbo de hoje
- **Filtra 09h após** *(webhook VTR)* → garante que só exibe registros preenchidos a partir das 9h

### Verificação
- **Code / Code1 / Code2 / Code3 / Code4** → verifica campos fora do padrão para VTR via webhook/schedule
- **Verifica Campos Fora do Padrão USA / USA1 / USA2** → verifica USA via Google Sheets/webhook
- **Verifica Campos Fora do Padrão USB / USB1** → verifica USB via Google Sheets/webhook

### Organização
- **Switch** (Aprovados/Reprovados) → separa os itens em dois caminhos
- **Edit Fields / Organiza** → extrai campos padronizados para aprovados e reprovados
- **Organiza Informações** → monta estrutura com `VTR`, `condutor/enfermeiro`, `fora_do_padrao` e `pendencias`
- **Merge** → recombina aprovados e reprovados em um único fluxo

### Consolidação e Envio
- **Junta e Mescla / Junta e Mescla1 / Junta e Mescla2 / Junta e Mescla3** → agrupa por número da VTR
- **Filter / Filter1 / Filter2** *(webhook)* → filtra apenas a VTR correspondente ao webhook recebido
- **Organiza Mensagem / Organiza Mensagem (webhook) 1 / 2** → formata a mensagem final
- **Enviar Mensagem1** → envia via Evolution API para o grupo do WhatsApp

---

## 📋 Fontes de Dados (Google Sheets)

| Planilha | Conteúdo |
|----------|---------|
| `1o05f****************GvmBvQRg` | Checklist VTR (veículo) |
| `1oE*******************DUv5ILT1qWjHU` | Checklist USA |
| `1u_***********************fB5Q` | Checklist USB |

---

## 🔧 Integrações

| Serviço | Uso |
|---------|-----|
| **Google Sheets** | Leitura dos formulários preenchidos |
| **Evolution API** | Envio de mensagens no grupo do WhatsApp |
| **Google Apps Script** | Dispara os webhooks ao submeter o formulário |

---

## ⚙️ Configuração

- **Instância Evolution API:** `bem_cuidar_alerta`
- **Grupo WhatsApp:** `*********6@g.us`
- **Timezone:** `America/Sao_Paulo`
- **Agendamento:** diário às 9h

---

## 📄 Licença

Projeto interno — Simplifica Gestão / Bem Cuidar.
