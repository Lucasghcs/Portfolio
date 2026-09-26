# Manual Operacional e Técnico de Implantação: Agente IA Eurovasos

**Projeto:** Operação Digital D2C & Agente Autônomo Multicanal  
**Empresa:** Eurovasos Indústria e Comércio de Vasos (Caruaru - PE)  
**Canais:** Mercado Livre, Amazon Brasil, Instagram (@eurovasos2019)  
**Versão:** 1.0 (Plano Piloto e Escala)

---

## 1. Visão Geral da Arquitetura

O sistema opera com nós de automação integrados via n8n (self-hosted), utilizando a API da LLM (Google Gemini / OpenAI) ancorada em uma base de conhecimento vetorial e relacional (RAG) contendo as fichas técnicas exatas dos moldes de polietileno da Eurovasos.

```
                    ┌───────────────────────────────┐
                    │      ERP Fábrica / Base       │
                    │   (Bling/Tiny ou JSON SKUs)   │
                    └──────────────┬────────────────┘
                                   │ Catálogo / Estoque
                                   ▼
┌──────────────────────┐    ┌──────────────┐    ┌──────────────────────┐
│ Mercado Livre API    │◄──►│ Orquestrador │◄──►│ LLM API              │
│ (Perguntas/Webhooks) │    │     n8n      │    │ (RAG Técnico)        │
└──────────────────────┘    └──────┬───────┘    └──────────────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────────┐
                    │ Meta Graph API (Instagram)    │
                    │ Agendamento de Conteúdo D2C   │
                    └───────────────────────────────┘
```

---

## 2. Passo a Passo Detalhado de Implantação

### Fase 1: Base de Conhecimento e Estruturação de SKUs (Dias 1 a 3)

#### Passo 1.1: Consolidação do Banco de Dados Técnico
Criação do arquivo estruturado `catalogo_eurovasos.json` com os 5 SKUs selecionados para o piloto:
1. Linha Ancara 90 cm (Cone Alto)
2. Linha Ancara 65 cm (Cone Médio)
3. Linha Namur 50 cm (Bojo Redondo)
4. Linha Namur 38 cm (Bojo Pequeno)
5. Bacia Roma / Jardineira Baixa

#### Passo 1.2: Estrutura do Schema JSON (`catalogo_eurovasos.json`)
```json
[
  {
    "sku": "EV-ANC-90-PRETO",
    "linha": "Ancara",
    "formato": "Cônico Alto",
    "material": "Polietileno 100% virgem rotomoldado",
    "aditivo_uv": true,
    "dimensoes": {
      "altura_cm": 90,
      "diametro_boca_cm": 42,
      "diametro_base_cm": 28,
      "volume_litros": 85,
      "peso_vazio_kg": 3.4
    },
    "cores_disponiveis": ["Preto", "Marmorizado Granito", "Tabaco", "Bege Areia"],
    "caracteristicas_tecnicas": {
      "furos_drenagem": "Sem furos de fábrica (possui marcações circulares na base para furar com facilidade)",
      "prato_compativel": "Prato Eurovasos R-30",
      "resistencia": "Suporta sol pleno, chuva, maresia; não descasca nem desbota"
    },
    "plantas_recomendadas": [
      "Ficus Lyrata",
      "Palmeira Raphis",
      "Pleomele",
      "Pata de Elefante",
      "Espada de São Jorge Grande"
    ],
    "cubagem_logistica": {
      "comprimento_cm": 45,
      "largura_cm": 45,
      "altura_cm": 93,
      "peso_cubado_kg": 7.5
    }
  }
]
```

---

### Fase 2: Configuração do Agente no n8n e Webhooks (Dias 4 a 6)

#### Passo 2.1: Infraestrutura de Execução
1. Subir uma VPS Linux (Ubuntu 22.04 LTS) com Docker e Docker Compose.
2. Subir container n8n com certificado SSL via Traefik ou Nginx Reverse Proxy.
3. Configurar persistência em volume seguro para manter histórico de perguntas e respostas.

#### Passo 2.2: Integração com o Mercado Livre Developers
1. Criar aplicação no portal [Mercado Livre Developers](https://developers.mercadolivre.com.br).
2. Obter `App ID` e `Secret Key`.
3. Configurar endpoint de notificação: `https://n8n.eurovasos.com.br/webhook/meli-questions`.
4. Habilitar tópicos: `questions` e `orders_v2`.

#### Passo 2.3: Fluxo Lógico no n8n para SAC Instantâneo
1. **Node 1 (Webhook Trigger):** Recebe o POST do Mercado Livre com `resource: "/questions/{id}"`.
2. **Node 2 (HTTP Request - Get Question):** Consulta os detalhes da pergunta:
   `GET https://api.mercadolibre.com/questions/{id}`.
3. **Node 3 (Item Lookup):** Consulta o `item_id` para obter o SKU e dimensões na base JSON.
4. **Node 4 (LLM Agent Node):** Executa o prompt com o contexto do SKU e a pergunta do cliente.
5. **Node 5 (Filtro de Segurança):** Verifica se a resposta não contém telefones, links externos ou termos proibidos pelas políticas do Mercado Livre.
6. **Node 6 (HTTP Request - Post Answer):** Envia a resposta:
   `POST https://api.mercadolibre.com/answers` com o payload `{"question_id": 1234567, "text": "..."}`.
7. **Node 7 (Alerta Telegram/WhatsApp):** Envia log de auditoria interna para a equipe.

---

### Fase 3: Prompts Mestres do Agente

#### Prompt do SAC no Mercado Livre (Injetado no n8n)
```text
Você é o Assistente Técnico Oficial de Vendas da Eurovasos, fábrica brasileira de vasos ornamentais em polietileno.

SUA MISSÃO:
Responder a dúvidas de compradores no Mercado Livre de forma rápida, educada e tecnicamente precisa, conduzindo para o fechamento imediato da compra.

DADOS TÉCNICOS DO PRODUTO:
{{ $json.produto_contexto }}

DIRETRIZES E REGRAS INEGOCIÁVEIS:
1. Limite-se a no máximo 3 ou 4 frases curtas e objetivas.
2. Destaque as qualidades industriais do polietileno: ultra-resistente, proteção anti-UV contra sol e chuva, leve para transportar e não quebra como concreto ou cerâmica.
3. Sempre informe a litragem exata e se o vaso suporta a planta perguntada pelo comprador.
4. Explique que o vaso não vem furado de fábrica para manter a opção de uso em ambientes internos, mas possui pontos demarcados na base para furação rápida caso seja usado em jardim.
5. JAMAIS forneça números de telefone, endereços, links externos ou redes sociais (risco de banimento da conta no Mercado Livre).
6. Encerre com cordialidade e disponibilidade imediata para despacho.
```

#### Prompt do Gerador de Conteúdo para Instagram (@eurovasos2019)
```text
Você é o Especialista em Paisagismo, Arquitetura e Biofilia da Eurovasos (@eurovasos2019).

SUA MISSÃO:
Gerar roteiros de postagens e Reels com foco em designers de interiores, paisagistas e proprietários de residências com varanda gourmet, sala de estar e área com piscina.

ESTRUTURA DE CADA POST:
1. Título do Gancho (Hook): Frase de alto impacto visual nos primeiros 3 segundos.
2. Conceito de Ambientação: Instruções exatas para a foto/vídeo (tipo de planta, iluminação natural, piso, composição com mais de um tamanho).
3. Texto da Legenda (Copy): Narrativa conectando estética refinada à praticidade do polietileno (fácil manutenção, não desbota no sol).
4. Chamada para Ação (CTA): Convidar para direct de especificadores ou link dos marketplaces na bio.
5. Bloco de Tags: 12 hashtags segmentadas em arquitetura, decoração e biofilia.
```

---

## 3. Matriz de Manutenção e Governança Semanal

| Dia da Semana | Responsável | Ação Operacional |
| :--- | :--- | :--- |
| **Segunda-feira** | Consultor IA | Auditoria de logs de respostas automáticas da semana anterior e refino de prompts. |
| **Quarta-feira** | Fábrica / Expedição | Checagem de saldo de estoque dos SKUs ativos para evitar ruptura nos marketplaces. |
| **Sexta-feira** | Consultor IA | Agendamento do lote de 3 conteúdos semanais validados para o perfil do Instagram. |
| **Quinzenal** | Diretoria & Consultor | Apresentação do dashboard com: Faturamento D2C, Tempo Médio de SAC e Margem Líquida. |

---

## 4. Checklist de Validação Final do Piloto (Dia 15)

- [ ] Mais de 90% das perguntas de pré-venda no Mercado Livre respondidas em $\le$ 3 minutos.
- [ ] Zero advertências ou notificações de infração de políticas nos marketplaces.
- [ ] Registro do faturamento bruto gerado pelos 5 SKUs iniciais.
- [ ] Confirmação da margem líquida D2C apurada em comparação à margem do atacado tradicional.
- [ ] Assinatura do termo de continuidade e início da vigência comercial com variável progressivo.