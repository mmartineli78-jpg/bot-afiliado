# Bot de Gestão de Afiliados

## Setup local

```bash
npm install
cp .env.example .env    # preencha DISCORD_TOKEN, DATABASE_URL, etc.
npm run prisma:migrate  # cria as tabelas no Postgres
npm run deploy:commands # registra os slash commands no seu servidor
npm run dev              # inicia o bot com auto-reload
```

## Setup com Docker

```bash
cp .env.example .env
docker compose up --build
```

## Estrutura

```
src/
  commands/     -> um arquivo por slash command (/cadastrar-afiliado, /registrar-venda, ...)
  events/       -> handlers de eventos do Discord
  config/roles.js -> hierarquia de cargos e permissões
  services/paymentGateway/ -> camada abstrata de pagamento (troque o provedor aqui)
prisma/schema.prisma -> modelo de dados completo
```

## Status (Fase 1 - em andamento)

- [x] Estrutura do projeto, Docker, Prisma
- [x] Modelo de dados completo (afiliados, vendas, cashback, saques, metas, auditoria)
- [x] Hierarquia de cargos com permissões configuráveis
- [x] Camada de gateway de pagamento desacoplada (mock até definir provedor)
- [x] Comando `/cadastrar-afiliado`
- [ ] Comando `/solicitar-qrcode`
- [ ] Comando `/registrar-venda` + fluxo de aprovação
- [ ] Sistema de cashback automático
- [ ] Sistema de saque
- [ ] Rankings (diário/semanal/mensal/anual)
- [ ] Fechamento diário automático
- [ ] Painel do afiliado / painel administrativo
# bot-afiliado
bot para loja que contem afiliado














# Bot de Gestão de Afiliados para Discord — Especificação Técnica

## 1. Visão Geral

Bot para Discord voltado à gestão completa de uma equipe de afiliados: vendas, recrutamento, cashback, metas e acompanhamento de desempenho em tempo real, com controle hierárquico (afiliado → líder → gestor/admin).

## 2. Stack Tecnológica Recomendada

| Camada | Tecnologia |
|---|---|
| Runtime | Node.js (LTS 20+) |
| Bot Discord | discord.js v14 |
| API/Backend | Express.js (para painéis web, webhooks de pagamento, geração de QR Code) |
| Banco de dados | PostgreSQL (recomendado pela integridade relacional exigida por saldo/cashback/auditoria) — alternativa: MongoDB se preferir schema flexível |
| ORM | Prisma (facilita migrations e é compatível com Postgres/Mongo) |
| QR Code | biblioteca `qrcode` (geração) + payload apontando para link de checkout/pagamento |
| Hospedagem | VPS Linux + Docker + Docker Compose + PM2 (ou PM2 dentro do container) |
| Cache/filas (opcional, recomendado) | Redis — para rankings em tempo real e filas de aprovação |

## 3. Módulos Funcionais

### 3.1 Solicitação de QR Code (fechamento de venda)
Canal/painel onde o afiliado solicita um QR Code para fechar uma venda.
- **Campos obrigatórios:** nome do cliente, CPF, e-mail, telefone (DDD), valor da compra
- Ao gerar, o sistema cria um **registro de solicitação** para auditoria (quem pediu, quando, dados do cliente, valor)

### 3.2 Registro de Venda
- **Campos:** cliente vinculado, produto/serviço, valor da venda, observação (opcional)
- **Fluxo:** afiliado registra → venda entra em "análise" → líder/gestor aprova ou rejeita → se aprovada, entra no ranking e gera cashback

### 3.3 Sistema de Cashback
- Ao confirmar a venda, cashback é creditado automaticamente conforme percentual definido pela empresa
- Afiliado acompanha: saldo disponível, histórico de ganhos, total acumulado, extrato detalhado

### 3.4 Sistema de Saque
- **Campos:** valor desejado, chave Pix, nome do titular
- **Fluxo:** afiliado solicita → vai para análise financeira → admin aprova/recusa → registro salvo no histórico

### 3.5 Rankings
- **Vendas:** diário, semanal, mensal, anual — mostrando posição, nome, qtd. de vendas, valor total
- **Recrutamento:** mesmas periodicidades — mostrando qtd. recrutados, posição, nome do recrutador

### 3.6 Fechamento Diário Automático
Executado automaticamente ao fim do dia, com resumo contendo:
- Meta diária x valor vendido
- Qtd. de vendas
- Qtd. de afiliados ativos
- Top 5 vendedores e top 5 recrutadores
- % de meta atingida

### 3.7 Painel do Afiliado
Total vendido, vendas aprovadas/pendentes, cashback acumulado/disponível, recrutamentos, posição no ranking, histórico completo.

### 3.8 Painel Administrativo (líderes/gestores)
Aprovar/rejeitar vendas, aprovar/recusar saques, ajustar cashback, definir metas, editar registros, aplicar bonificações, gerar relatórios.

### 3.9 Sistema de Metas
- Individuais e coletivas (diária, semanal, mensal)
- Recompensas: cargo temporário, bônus, cashback extra, premiações especiais

### 3.10 Logs de Auditoria
Registro automático de: vendas, aprovações/rejeições, saques, alterações administrativas, recrutamentos — garantindo transparência total.

## 4. Modelo de Dados (proposta inicial — PostgreSQL)

```
affiliates (id, discord_id, nome, cpf, pix_key, upline_id, cargo, ativo, criado_em)
customers (id, nome, cpf, email, telefone)
qr_requests (id, affiliate_id, customer_id, valor, status, criado_em)
sales (id, affiliate_id, customer_id, produto, valor, observacao, status[pendente/aprovada/rejeitada], aprovado_por, criado_em, aprovado_em)
cashback_transactions (id, affiliate_id, sale_id, valor, tipo[credito/ajuste], criado_em)
withdrawals (id, affiliate_id, valor, pix_key, titular, status[pendente/aprovado/recusado], analisado_por, criado_em, analisado_em)
goals (id, tipo[individual/coletiva], periodo[diaria/semanal/mensal], meta_valor, afiliado_id ou equipe_id, criado_em)
recruitments (id, recrutador_id, recrutado_id, criado_em)
audit_logs (id, ator_id, acao, entidade, entidade_id, detalhes_json, criado_em)
```

## 5. Roadmap de Desenvolvimento Sugerido

| Fase | Escopo |
|---|---|
| **1 — Base** | Setup do projeto (Node.js + discord.js + Postgres + Prisma), cadastro de afiliados, estrutura de permissões (afiliado/líder/admin) |
| **2 — Vendas** | QR Code, registro de venda, fluxo de aprovação/rejeição |
| **3 — Financeiro** | Cashback automático, extrato, sistema de saque com aprovação |
| **4 — Engajamento** | Rankings (vendas e recrutamento), painel do afiliado |
| **5 — Gestão** | Painel administrativo, sistema de metas e bonificações |
| **6 — Automação e Transparência** | Fechamento diário automático, logs de auditoria completos |
| **7 — Deploy** | Dockerização, deploy em VPS com PM2/Docker Compose, monitoramento |

## 6. Definições Confirmadas

- **Banco de dados:** PostgreSQL
- **QR Code:** gera **link de pagamento real** (Pix / Mercado Pago) — precisa de integração com gateway de pagamento
- **Cashback:** percentual **configurável** por afiliado/produto (não é fixo)
- **Hierarquia de cargos** (do menor para o maior nível de acesso):

| Nível | Cargo |
|---|---|
| 1 | Membro AFL |
| 2 | Sublíder AFL |
| 3 | Líder AFL |
| 4 | Auxiliar AFL |
| 5 | Responsável AFL |
| 6 | ADM AFL |
| 7 | Master AFL |
| 8 | SG — Responsável Vendas Comercial |

> Isso vira uma tabela `roles` com `nivel` (inteiro) e permissões associadas, em vez de cargos fixos no código — assim dá pra ajustar sem redeploy.

## 7. Pendente

- **Gateway de pagamento:** a definir (Mercado Pago foi cogitado, mas ainda não fechado). O sistema será construído com uma camada de abstração (`PaymentGatewayService`), permitindo trocar o provedor sem alterar o restante do código.

