texto
# 🏠 Assistente WhatsApp GPT v32.1 — Freelancer Imóveis Aracaju

Workflow N8N para automação completa de atendimento e qualificação de leads imobiliários via WhatsApp, integrado com GPT-4o-mini, Google Sheets (CRM) e Upstash Redis.

---

## 📋 Visão Geral

Este workflow automatiza o ciclo completo de captação e qualificação de leads para corretores autônomos de imóveis em Aracaju/SE. O bot atua como assistente virtual, qualifica o cliente com perguntas estratégicas, atribui um score de 0 a 100 e, quando o lead está "quente" (score ≥ 80), realiza o handover automático para o corretor via WhatsApp.

---

## ⚙️ Tecnologias

| Ferramenta | Função |
|---|---|
| [N8N](https://n8n.io) | Orquestração do workflow |
| WhatsApp Business API (Meta) | Canal de comunicação |
| OpenAI GPT-4o-mini | Qualificação inteligente e respostas |
| OpenAI Whisper | Transcrição de áudios |
| Google Sheets | CRM de leads |
| Upstash Redis | Deduplicação atômica de mensagens |

---

## 🔄 Fluxo Principal
Webhook WhatsApp POST
└─> Validar Ambiente (variáveis ​​obrigatórias)
└─> Extrair e Validar Payload Meta
└─> Dedup Redis SET NX (evita processamento duplicado)
└─> Switch por Tipo de Mensagem
├─ Áudio/Voz → Buscar URL Meta → Baixar OGG → Whisper → Consolidar
├─ Imagem → Solicitar descrição em texto
└─ Texto/Outros → Passe direto
└─> CRM: Buscar Lead no Google Sheets
└─> SE Status ENTREGUE?
├─ SIM → Verificar tentativas → Reabrir ou Avisar cliente
└─ NÃO → SE Levar Novo? → Cadastrar ou usar o contexto existente
└─> GPT-4o-mini: Qualificar e Gerar Resposta
└─> Parse Blindado da resposta JSON
└─> IF Lead QUENTE (pontuação ≥ 80)?
├─ SIM → Transferência: Avisar cliente + Notificar corretor + Marcar ENTREGUE
└─ NÃO → Resposta normal + Atualizar histórico CRM

texto

---

## 🔐 Variáveis de Ambiente Obrigatórias

Configure as seguintes variáveis no N8N antes de ativar o workflow:

| Variável | Descrição |
|---|---|
| `WHATSAPP_VERIFY_TOKEN` | Token de verificação do webhook Meta |
| `WHATSAPP_PHONE_NUMBER_ID` | ID do número de telefone WhatsApp Business |
| `GOOGLE_SHEET_ID` | ID da planilha Google Sheets (CRM) |
| `OPENAI_API_KEY` | Chave da API OpenAI (GPT-4o-mini + Whisper) |
| `CORRETOR_PHONE` | Número do corretor para handover (formato: `5579XXXXXXXX`) |
| `META_API_VERSION` | Versão da API Meta (ex: `v22.0`) |
| `UPSTASH_REDIS_URL` | URL do Upstash Redis (opcional, mas recomendado) |
| `UPSTASH_REDIS_TOKEN` | Token do Upstash Redis (opcional, mas recomendado) |
| `GPT_SYSTEM_PROMPT` | Prompt customizado do sistema (opcional — substitui o padrão) |

---

## 🏗️ Estrutura do CRM (Google Sheets)

### Aba Principal (gid=0) — Leads

| Coluna | Descrição |
|---|---|
| `Telefone` | Identificador do lead (chave primária) |
| `Status` | `NOVO` ou `ENTREGUE` |
| `Score` | Pontuação de 0–100 |
| `Resumo` | Perfil resumido do lead |
| `Historico` | Histórico completo da conversa (últimos 6000 chars) |
| `ULTIMA_MSG` | Timestamp da última mensagem |
| `Origem` | Canal de origem (`WhatsApp`) |
| `Data_Entrega` | Data/hora do handover ao corretor |
| `Tentativas_Pos_Entrega` | Contador de retornos pós-entrega |

### Aba de Erros (gid=2) — Log de Notificações

Registra falhas críticas na notificação ao corretor.

---

## 📊 Sistema de Scoring

O GPT classifica o lead incrementalmente. Os pontos **somam** ao score existente (nunca reduzem):

| Critério | Pontos |
|---|---|
| Orçamento > R$ 500k | +30 |
| Prazo de compra < 90 dias | +40 |
| Pediu visita ou proposta | +30 |
| Bairro nobre (Jardins, Atalaia, Ponta da Terra, 13 de Julho) | +20 |
| Compra à vista | +20 |
| Já visitou algum imóvel | +15 |

**Score ≥ 80 → Lead QUENTE → Handover automático ao corretor.**

---

## 🛡️ Tratamento de Erros e Resiliência

- **Dedup atômico via Redis** (`SET NX EX`): evita processamento duplicado mesmo sob race conditions. Se o Redis estiver offline, aceita a mensagem com log de warning.
- **Fallback de áudio**: cada etapa do pipeline de áudio (URL, download, Whisper) tem tratamento de erro individual com propagação de contexto via `_msgCtx`.
- **CRM inacessível**: se o Google Sheets falhar, o lead é tratado como novo e o flow continua sem interrupção.
- **GPT com retry**: até 3 tentativas com backoff exponencial para chamadas à API OpenAI.
- **Parse blindado**: o JSON retornado pelo GPT é validado e sanitizado antes de qualquer uso downstream.
- **Rate limit por telefone**: janela de 1s por número via Redis para evitar spam de respostas.

---

## 🚀 Como Importar no N8N

1. Acesse seu N8N e clique em **Import from file**.
2. Selecione o arquivo JSON deste repositório.
3. Configure todas as **variáveis de ambiente** listadas acima.
4. Configure as **credenciais** necessárias:
   - `httpHeaderAuth` com o token Bearer da Meta API
   - Credenciais do Google Sheets (OAuth2 ou Service Account)
   - Credenciais OpenAI
5. Configure o webhook na [Meta for Developers](https://developers.facebook.com) apontando para a URL do N8N.
6. Ative o workflow.

---

## 📌 Observações

- O workflow inicia **inativo** por padrão — ative manualmente após configurar todas as credenciais.
- O histórico de conversa é truncado nos últimos **6.000 caracteres** para otimizar o contexto enviado ao GPT.
- O `GPT_SYSTEM_PROMPT` permite customizar as regras de negócio sem editar o workflow diretamente.
- Leads com status `ENTREGUE` que retornam são reabertos automaticamente após **5 tentativas**.

