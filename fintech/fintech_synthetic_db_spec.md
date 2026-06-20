# BASE DE DADOS SINTÉTICA — FINTECH/BANCO DIGITAL BRASILEIRO
## Especificação Técnica Completa | 2018–2026

> **Equipe:** Data Engineering · Data Science · Analytics Engineering · Crédito & Cobrança · ML · BI  
> **Versão:** 1.0 | Junho 2026

---

# PARTE 1 — VISÃO GERAL E MODELO CONCEITUAL

## 1.1 Escopo e Contexto

| Dimensão | Valor |
|---|---|
| Clientes | ~2.000.000 |
| Contratos | ~15.000.000 |
| Eventos (logs, cobrança, interações) | ~500.000.000 |
| Período histórico | Jan/2018 – Jun/2026 (8,5 anos) |
| Granularidade mínima | Evento (timestamp) |
| Domínio | Fintech/Banco Digital Brasileiro |

**Cenários macro embutidos propositalmente:**
- Crise pré-eleições 2018 → alta do dólar, SELIC em queda
- 2019: desaceleração econômica, reforma da previdência
- 2020: pandemia COVID-19 → inadimplência, renegociações massivas
- 2021: recuperação assimétrica, Pix lançado, boom de fintechs
- 2022: inflação elevada, SELIC 13,75%, eleições
- 2023: Arcabouço fiscal, início de queda de juros
- 2024–2026: normalização gradual, open banking consolidado

---

## 1.2 Modelo Conceitual (Diagrama de Entidades)

```
┌─────────────────────────────────────────────────────────────────┐
│                    MACROECONÔMICO (dim_macro)                   │
│              Tabela temporal — 1 registro/mês                   │
└───────────────────────┬─────────────────────────────────────────┘
                        │ contexto temporal (join por mês/ano)
┌───────────────────────▼─────────────────────────────────────────┐
│                      CLIENTES (dim_cliente)                     │
│  2.000.000 registros | PK: id_cliente                           │
└──┬──────────┬──────────┬──────────┬──────────┬──────────────────┘
   │1:N       │1:N       │1:N       │1:N       │1:N
   ▼          ▼          ▼          ▼          ▼
CONTRATOS  BUREAU    RECLAMAÇÕES CAMPANHAS  INTERAÇÕES
(fato_     (fato_    (fato_       (fato_     DIGITAIS
contrato)  bureau)   reclamacao)  campanha)  (fato_
  │1:N                                       interacao)
  ▼
PARCELAS ──────── EVENTOS DE COBRANÇA
(fato_parcela)    (fato_cobrança)
                        │
                   CALL CENTER
                  (fato_callcenter)
```

---

## 1.3 Domínios de Produtos

| Código | Produto | Ticket Médio | Prazo Típico |
|--------|---------|-------------|--------------|
| EP | Empréstimo Pessoal | R$ 8.000 | 24 meses |
| CC | Cartão de Crédito | R$ 3.500 (limite) | Rotativo |
| FV | Financiamento Veículo | R$ 35.000 | 48 meses |
| CS | Crédito Consignado | R$ 12.000 | 60 meses |
| CG | Capital de Giro | R$ 50.000 | 12 meses |
| FGTS | Antecipação FGTS | R$ 4.000 | 12 meses |
| CD | Crédito Digital | R$ 1.500 | 6 meses |

---

# PARTE 2 — MODELO LÓGICO E DICIONÁRIO DE DADOS

## 2.1 Tabela: DIM_CLIENTE

**Descrição:** Cadastro único de clientes. Versioned (SCD Type 2 recomendado).  
**Volume:** ~2.000.000 linhas ativas + ~400.000 históricas (atualizações)  
**Granularidade:** 1 linha por versão de cliente  
**Particionamento:** `data_cadastro` (ano/mês)

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_cliente | BIGINT | N | PK | Sequencial | Identificador único do cliente |
| sk_cliente | BIGINT | N | — | Sequencial | Surrogate key (SCD2) |
| cpf | VARCHAR(11) | N | UK | — | CPF sem formatação. ~2% duplicados intencionais (fraude/erro) |
| nome_hash | VARCHAR(64) | N | — | — | SHA-256 do nome (LGPD) |
| sexo | CHAR(1) | Y | — | M:52%, F:47%, NB:1% | M=Masculino, F=Feminino, NB=Não-binário |
| data_nascimento | DATE | N | — | Normal μ=38, σ=12 anos | Faixa 18–80 anos |
| idade | INT | N | — | Derivado | Calculado em run-time |
| estado_civil | VARCHAR(20) | Y | — | Solteiro:40%, Casado:38%, Div:15%, Viúvo:5%, Outros:2% | Estado civil declarado |
| escolaridade | VARCHAR(30) | Y | — | Fund:15%, Médio:45%, Superior:30%, Pós:10% | Nível de escolaridade |
| profissao_cod | VARCHAR(10) | Y | FK | — | FK → dim_profissao |
| cidade | VARCHAR(80) | Y | — | Log-normal (concentração SP/RJ/MG) | Município de residência |
| estado | CHAR(2) | Y | — | SP:35%, RJ:13%, MG:10%, outros:42% | UF |
| regiao | VARCHAR(10) | N | — | Derivado de estado | SE/S/NE/CO/N |
| cep_prefixo | VARCHAR(5) | Y | — | — | Primeiros 5 dígitos do CEP |
| renda_mensal | DECIMAL(12,2) | Y | — | Log-normal μ=R$3.200, σ=R$2.800 | Renda declarada. Cauda longa. |
| renda_presumida | DECIMAL(12,2) | Y | — | — | Estimativa por modelo interno |
| patrimonio_estimado | DECIMAL(14,2) | Y | — | Log-normal | Estimativa bureau + declarado |
| score_bureau_entrada | SMALLINT | Y | — | Normal truncada [0,1000] μ=580, σ=140 | Score no momento do cadastro |
| data_cadastro | DATE | N | — | Poisson acelerado 2019–2021 | Data de abertura da conta |
| data_atualizacao | TIMESTAMP | Y | — | — | Última atualização cadastral |
| segmento | VARCHAR(20) | N | — | Standard:60%, Plus:25%, Premium:12%, Private:3% | Segmento de valor |
| canal_aquisicao | VARCHAR(30) | N | — | App:55%, Indicação:20%, Parceiro:15%, Web:8%, PDV:2% | Canal de captação |
| status_cliente | VARCHAR(15) | N | — | Ativo:72%, Inativo:18%, Bloqueado:6%, Encerrado:4% | Status atual |
| flag_pep | BOOLEAN | N | — | Bernoulli p=0.008 | Pessoa Politicamente Exposta |
| flag_fraude | BOOLEAN | N | — | Bernoulli p=0.012 | Marcado como fraude (descoberta posterior) |
| flag_obito | BOOLEAN | N | — | Bernoulli p=0.003 | Óbito registrado |
| consentimento_lgpd | BOOLEAN | N | — | Bernoulli p=0.97 | Aceite LGPD |
| dt_inicio_validade | DATE | N | — | — | SCD2: início da versão |
| dt_fim_validade | DATE | Y | — | — | SCD2: fim da versão (NULL=atual) |
| is_current | BOOLEAN | N | — | — | SCD2: flag versão atual |

**Regras de negócio:**
- CPF deve ter 11 dígitos; ~2% com CPF duplicado ou inválido (simular erro de digitação/fraude)
- `renda_mensal` correlacionada com `escolaridade`, `profissao_cod`, `segmento`
- Clientes `Premium/Private` têm `score_bureau_entrada` > 700 em 80% dos casos
- `canal_aquisicao = App` concentrado em idades 18–35
- `flag_fraude` correlacionado com CPF duplicado e score baixo

---

## 2.2 Tabela: DIM_PROFISSAO

| Campo | Tipo | Descrição |
|-------|------|-----------|
| cod_profissao | VARCHAR(10) | PK |
| descricao | VARCHAR(60) | Nome da profissão |
| categoria | VARCHAR(30) | CLT / Autônomo / Empresário / Servidor / Aposentado / Estudante |
| risco_renda | CHAR(1) | A/M/B — Risco de instabilidade de renda |
| renda_media_cbo | DECIMAL(10,2) | Renda média por CBO (referência IBGE) |

---

## 2.3 Tabela: FATO_CONTRATO

**Descrição:** Registro de cada contrato firmado.  
**Volume:** ~15.000.000 linhas  
**Granularidade:** 1 linha por contrato  
**Particionamento:** `data_contratacao` (ano/mês) + `tipo_produto`

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_contrato | BIGINT | N | PK | Sequencial | ID único do contrato |
| id_cliente | BIGINT | N | FK→dim_cliente | — | Dono do contrato |
| tipo_produto | VARCHAR(10) | N | FK→dim_produto | EP/CC/FV/CS/CG/FGTS/CD | Tipo de produto |
| valor_contratado | DECIMAL(14,2) | N | — | Log-normal por produto | Valor principal |
| prazo_meses | SMALLINT | N | — | Discreto por produto | Número de parcelas |
| taxa_juros_am | DECIMAL(6,4) | N | — | Normal por produto/segmento | Taxa mensal (ex: 0.0289 = 2,89% a.m.) |
| taxa_cet_am | DECIMAL(6,4) | N | — | taxa_juros_am + spread | Custo Efetivo Total mensal |
| valor_parcela | DECIMAL(12,2) | N | — | Derivado (Price/SAC) | Valor da parcela |
| sistema_amortizacao | CHAR(5) | N | — | Price:70%, SAC:30% | Price ou SAC |
| data_contratacao | DATE | N | — | Poisson acelerado 2020–2022 | Data de assinatura |
| data_primeiro_vencimento | DATE | N | — | data_contratacao + 30d | Vencimento da 1ª parcela |
| data_encerramento_prevista | DATE | N | — | Derivado | Previsão de quitação |
| data_encerramento_real | DATE | Y | — | — | Data real (NULL=em aberto) |
| status_contrato | VARCHAR(20) | N | — | Ver tabela abaixo | Status atual |
| canal_contratacao | VARCHAR(20) | N | — | App:65%, Web:15%, Agente:12%, Parceiro:8% | Canal de origem |
| score_decisao | SMALLINT | N | — | Normal [0,1000] | Score usado na aprovação |
| limite_aprovado | DECIMAL(12,2) | Y | — | — | Para CC: limite total aprovado |
| finalidade | VARCHAR(40) | Y | — | — | Declarada pelo cliente |
| flag_renegociado | BOOLEAN | N | — | Bernoulli p=0.18 | Originado de renegociação |
| id_contrato_origem | BIGINT | Y | FK→fato_contrato | — | Contrato que originou esta renegociação |
| flag_consignado_publico | BOOLEAN | N | — | — | Desconto em folha pública |
| orgao_pagador | VARCHAR(60) | Y | — | — | Para consignado |
| dt_cancelamento | DATE | Y | — | — | Data de cancelamento |
| motivo_cancelamento | VARCHAR(40) | Y | — | — | Razão do cancelamento |

**Status de contrato:**
| Status | Significado | % Aprox |
|--------|------------|---------|
| ATIVO | Contrato em dia | 52% |
| ATRASO_1_30 | Atraso 1–30 dias | 8% |
| ATRASO_31_60 | Atraso 31–60 dias | 4% |
| ATRASO_61_90 | Atraso 61–90 dias | 3% |
| ATRASO_91_180 | Atraso 91–180 dias | 5% |
| ATRASO_180_PLUS | Atraso >180 dias | 6% |
| QUITADO | Pago integralmente | 15% |
| RENEGOCIADO | Substituído por novo | 5% |
| CANCELADO | Cancelado antes de uso | 2% |

---

## 2.4 Tabela: FATO_PARCELA

**Descrição:** Cada parcela de cada contrato.  
**Volume:** ~150.000.000 linhas (média 10 parcelas/contrato)  
**Granularidade:** 1 linha por parcela  
**Particionamento:** `vencimento` (ano/mês)

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_parcela | BIGINT | N | PK | Sequencial | ID único |
| id_contrato | BIGINT | N | FK→fato_contrato | — | Contrato pai |
| id_cliente | BIGINT | N | FK→dim_cliente | — | Desnormalizado para performance |
| numero_parcela | SMALLINT | N | — | [1, prazo_meses] | Número sequencial |
| vencimento | DATE | N | — | — | Data de vencimento |
| valor_nominal | DECIMAL(12,2) | N | — | — | Valor original da parcela |
| valor_atualizado | DECIMAL(12,2) | Y | — | — | Com multa/juros de mora |
| valor_pago | DECIMAL(12,2) | Y | — | — | Valor efetivamente pago (NULL=não pago) |
| data_pagamento | DATE | Y | — | — | Data do pagamento (NULL=não pago) |
| dias_atraso | SMALLINT | N | — | Exp(λ=0.1) para inadimplentes | Dias em atraso no fechamento do dia |
| status_parcela | VARCHAR(20) | N | — | Ver abaixo | Status |
| canal_pagamento | VARCHAR(20) | Y | — | Pix:58%, Boleto:25%, Débito:12%, Outros:5% | Como foi pago |
| flag_acordo | BOOLEAN | N | — | Bernoulli p=0.08 | Pago via acordo de cobrança |
| valor_desconto_acordo | DECIMAL(10,2) | Y | — | — | Desconto dado em acordo |
| flag_fraude_pagamento | BOOLEAN | N | — | Bernoulli p=0.002 | Fraude no pagamento |
| competencia_mes | INT | N | — | — | Mês de referência (YYYYMM) |

**Status parcela:** `A_VENCER` / `PAGA_EM_DIA` / `PAGA_ATRASO` / `EM_ATRASO` / `PERDIDA` / `RENEGOCIADA`

**Correlações estatísticas críticas:**
- `dias_atraso` correlacionado negativamente com `score_bureau` do cliente
- Clientes com `flag_renegociado=TRUE` têm 3× mais probabilidade de novo atraso
- Parcelas próximas ao Natal/Janeiro têm 15% mais inadimplência (sazonalidade)
- Crise COVID (Abr-Jun/2020): multiplicador de inadimplência ×2,1

---

## 2.5 Tabela: FATO_EVENTO_COBRANCA

**Descrição:** Cada tentativa de contato/cobrança.  
**Volume:** ~200.000.000 linhas  
**Granularidade:** 1 evento por tentativa de contato  
**Particionamento:** `data_hora` (ano/mês) + `canal`

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_evento | BIGINT | N | PK | Sequencial | ID único |
| id_cliente | BIGINT | N | FK→dim_cliente | — | Cliente contactado |
| id_contrato | BIGINT | N | FK→fato_contrato | — | Contrato em cobrança |
| id_parcela | BIGINT | Y | FK→fato_parcela | — | Parcela específica (nullable para cobrança geral) |
| data_hora | TIMESTAMP | N | — | Concentrado seg-sex 8h–20h | Data e hora do evento |
| canal | VARCHAR(20) | N | — | Ver distribuição abaixo | Canal de contato |
| tipo_acao | VARCHAR(30) | N | — | — | notificacao_preventiva / aviso_vencimento / cobrança_ativa / negativacao / acordo |
| resultado_contato | VARCHAR(20) | N | — | Ver abaixo | Resultado |
| promessa_pagamento | BOOLEAN | N | — | — | Cliente prometeu pagar |
| valor_prometido | DECIMAL(12,2) | Y | — | — | Valor prometido |
| data_promessa | DATE | Y | — | — | Data prometida pelo cliente |
| promessa_cumprida | BOOLEAN | Y | — | — | NULL até a data, depois preenchido |
| custo_acao | DECIMAL(8,2) | Y | — | Por canal | Custo operacional do contato |
| id_agente | INTEGER | Y | FK→dim_agente | — | Agente (NULL=automático) |
| script_utilizado | VARCHAR(20) | Y | — | — | Versão do script |
| dias_atraso_momento | SMALLINT | N | — | — | Dias em atraso no momento do contato |
| faixa_cobranca | CHAR(2) | N | — | P1/P2/P3/P4/P5 | Faixa de cobrança (P1=1-30d, P5=180d+) |

**Distribuição de canais:**
| Canal | % | Taxa Contato | Taxa Recuperação | Custo |
|-------|---|-------------|-----------------|-------|
| sms | 30% | 85% (entrega) | 8% | R$0,08 |
| whatsapp | 25% | 72% (leitura) | 18% | R$0,12 |
| email | 15% | 60% (abertura) | 6% | R$0,02 |
| telefone | 12% | 45% (atendimento) | 28% | R$2,50 |
| chatbot | 10% | 90% (iniciado) | 12% | R$0,20 |
| agente_humano | 8% | 95% (contato) | 38% | R$18,00 |

**Resultados do contato:**
`sem_resposta` / `nao_localizado` / `numero_errado` / `promessa_pag` / `pagamento_efetuado` / `negou_divida` / `acionamento_juridico` / `acordo_fechado`

**Regras de negócio:**
- WhatsApp tem eficácia 2,3× maior em clientes com idade ≤ 35 anos
- Email tem eficácia 1,8× maior em segmento Plus/Premium e empresários
- Agente humano reservado para faixas P3+ (>60 dias atraso)
- Máximo 3 contatos/semana por cliente (lei anti-assédio)
- Horário de cobrança: seg-sex 8h-20h, sáb 8h-14h (Lei 14.181/2021)

---

## 2.6 Tabela: FATO_INTERACAO_DIGITAL

**Descrição:** Eventos de uso do aplicativo/web.  
**Volume:** ~150.000.000 linhas  
**Granularidade:** 1 evento por interação  
**Particionamento:** `data_hora` (ano/mês)

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_interacao | BIGINT | N | PK | Sequencial | ID único |
| id_cliente | BIGINT | N | FK→dim_cliente | — | Cliente |
| id_sessao | VARCHAR(36) | N | — | UUID | Sessão do usuário |
| data_hora | TIMESTAMP | N | — | — | Timestamp do evento |
| tipo_evento | VARCHAR(30) | N | — | Ver abaixo | Tipo de ação |
| plataforma | VARCHAR(10) | N | — | iOS:45%, Android:48%, Web:7% | Plataforma de acesso |
| versao_app | VARCHAR(10) | Y | — | — | Versão do app |
| duracao_seg | SMALLINT | Y | — | Exp por tipo | Duração do evento em segundos |
| id_contrato_ref | BIGINT | Y | FK→fato_contrato | — | Contrato referenciado (se aplicável) |
| valor_transacao | DECIMAL(12,2) | Y | — | — | Para transações financeiras |
| sucesso | BOOLEAN | N | — | p=0.94 | Ação concluída com sucesso |
| erro_codigo | VARCHAR(10) | Y | — | — | Código de erro (se falhou) |
| latitude | DECIMAL(9,6) | Y | — | — | Geolocalização (consentido) |
| longitude | DECIMAL(9,6) | Y | — | — | Geolocalização (consentido) |

**Tipos de evento:**
`login` / `logout` / `consulta_saldo` / `consulta_fatura` / `emissao_boleto` / `pagamento_pix` / `pagamento_boleto` / `renegociacao_inicio` / `renegociacao_conclusao` / `abertura_ticket` / `consulta_limite` / `aumento_limite_solicitacao` / `atualizacao_cadastro` / `simulacao_emprestimo` / `contratacao_produto` / `consulta_extrato` / `desbloqueio_cartao` / `bloqueio_cartao` / `alteracao_vencimento`

**Correlação com churn:** Clientes que não logar por 45+ dias têm probabilidade de churn 4,2× maior.

---

## 2.7 Tabela: FATO_RECLAMACAO

**Descrição:** Reclamações formais (app, Reclame Aqui, BACEN, Procon).  
**Volume:** ~3.000.000 linhas  
**Granularidade:** 1 por reclamação  
**Particionamento:** `data_abertura` (ano)

| Campo | Tipo | Nullable | PK/FK | Distribuição | Descrição |
|-------|------|----------|-------|-------------|-----------|
| id_reclamacao | BIGINT | N | PK | — | ID único |
| id_cliente | BIGINT | N | FK→dim_cliente | — | Cliente |
| canal_reclamacao | VARCHAR(20) | N | — | App:45%, ReclameAqui:25%, BACEN:15%, Procon:10%, Email:5% | Canal |
| assunto | VARCHAR(40) | N | — | Ver categorias | Motivo da reclamação |
| gravidade | CHAR(1) | N | — | A:30%, M:45%, B:25% | A=Alta, M=Média, B=Baixa |
| data_abertura | TIMESTAMP | N | — | — | Abertura |
| data_resolucao | TIMESTAMP | Y | — | — | Resolução (NULL=aberta) |
| tempo_resolucao_horas | DECIMAL(8,2) | Y | — | Log-normal | Horas até resolução |
| status | VARCHAR(15) | N | — | — | `aberta`/`em_andamento`/`resolvida`/`escalada`/`encerrada` |
| satisfacao_pos | TINYINT | Y | — | Normal [1,5] | NPS 1–5 pós-resolução |
| reincidente | BOOLEAN | N | — | p=0.22 | Mesmo assunto em <90 dias |
| id_contrato_ref | BIGINT | Y | FK→fato_contrato | — | Contrato relacionado |

**Categorias de assunto:** `cobrança_indevida` / `fraude_conta` / `cancelamento_negado` / `taxa_abusiva` / `atendimento_ruim` / `portabilidade` / `limite_negado` / `negativacao_indevida` / `produto_nao_solicitado`

---

## 2.8 Tabela: FATO_CAMPANHA

**Descrição:** Registro de ações de marketing/CRM.  
**Volume:** ~50.000.000 linhas (1 linha por cliente-campanha)  
**Granularidade:** 1 por cliente por campanha  
**Particionamento:** `data_envio` (ano/mês)

| Campo | Tipo | Nullable | PK/FK | Descrição |
|-------|------|----------|-------|-----------|
| id_campanha_cliente | BIGINT | N | PK | ID da participação |
| id_campanha | INTEGER | N | FK→dim_campanha | Campanha |
| id_cliente | BIGINT | N | FK→dim_cliente | Cliente |
| data_envio | DATE | N | — | Data de envio |
| canal_envio | VARCHAR(20) | N | — | Canal utilizado |
| abriu | BOOLEAN | Y | — | Abriu a comunicação |
| clicou | BOOLEAN | Y | — | Clicou no CTA |
| converteu | BOOLEAN | N | — | Comprou/aderiu ao produto |
| data_conversao | DATE | Y | — | Data da conversão |
| receita_gerada | DECIMAL(12,2) | Y | — | Receita atribuída (last-touch) |
| custo_contato | DECIMAL(8,2) | N | — | Custo do disparo |

---

## 2.9 Tabela: DIM_CAMPANHA

| Campo | Tipo | Descrição |
|-------|------|-----------|
| id_campanha | INTEGER | PK |
| nome_campanha | VARCHAR(80) | Nome |
| tipo | VARCHAR(30) | retencao / aquisicao / cross_sell / up_sell / cobranca / reativacao |
| produto_alvo | VARCHAR(10) | Produto sendo ofertado |
| segmento_alvo | VARCHAR(20) | Segmento público-alvo |
| canal_principal | VARCHAR(20) | Canal principal |
| data_inicio | DATE | — |
| data_fim | DATE | — |
| orcamento_total | DECIMAL(14,2) | Orçamento |
| meta_conversao_pct | DECIMAL(5,2) | Meta de conversão % |
| conversao_real_pct | DECIMAL(5,2) | Resultado real |
| status | VARCHAR(15) | ativa / encerrada / pausada / cancelada |

---

## 2.10 Tabela: FATO_CALLCENTER

**Descrição:** Atendimentos de call center (receptivo + ativo).  
**Volume:** ~20.000.000 linhas  
**Granularidade:** 1 por ligação  
**Particionamento:** `data_hora` (ano/mês)

| Campo | Tipo | Nullable | Descrição |
|-------|------|----------|-----------|
| id_chamada | BIGINT | N | PK |
| id_cliente | BIGINT | Y | FK→dim_cliente (NULL se não identificado) |
| data_hora_entrada | TIMESTAMP | N | — |
| data_hora_atendimento | TIMESTAMP | Y | NULL=abandonou |
| data_hora_encerramento | TIMESTAMP | Y | — |
| tempo_espera_seg | SMALLINT | N | Exp(λ=0.02) |
| tempo_atendimento_seg | SMALLINT | Y | Normal μ=240, σ=120 |
| abandonou | BOOLEAN | N | Bernoulli p=0.18 |
| id_agente | INTEGER | Y | FK→dim_agente |
| motivo_contato | VARCHAR(40) | N | — |
| resolucao_primeiro_contato | BOOLEAN | Y | — |
| transferido | BOOLEAN | N | — |
| satisfacao | TINYINT | Y | CSAT 1–5 |
| tipo_atendimento | CHAR(1) | N | R=Receptivo, A=Ativo |

---

## 2.11 Tabela: FATO_BUREAU

**Descrição:** Snapshot mensal do bureau de crédito por cliente.  
**Volume:** ~96.000.000 linhas (2M clientes × ~48 meses médio)  
**Granularidade:** 1 por cliente por mês  
**Particionamento:** `competencia` (ano/mês)

| Campo | Tipo | Nullable | Distribuição | Descrição |
|-------|------|----------|-------------|-----------|
| id_bureau | BIGINT | N | PK | ID único |
| id_cliente | BIGINT | N | FK | Cliente |
| competencia | INT | N | — | YYYYMM |
| score_serasa | SMALLINT | Y | Normal [0,1000] μ=560 | Score Serasa |
| score_spc | SMALLINT | Y | Normal [0,1000] μ=545 | Score SPC |
| qtd_consultas_30d | TINYINT | Y | Poisson λ=1.2 | Consultas ao CPF últimos 30d |
| qtd_consultas_90d | TINYINT | Y | Poisson λ=2.8 | Consultas últimos 90d |
| qtd_restricoes_ativas | TINYINT | Y | Poisson λ=0.4 | Restrições ativas |
| valor_negativado | DECIMAL(12,2) | Y | Log-normal | Valor total negativado |
| qtd_operacoes_ativas | TINYINT | Y | Poisson λ=3.2 | Operações de crédito ativas |
| valor_total_dividas | DECIMAL(14,2) | Y | Log-normal | Total de dívidas |
| renda_presumida_bureau | DECIMAL(12,2) | Y | Log-normal | Renda estimada pelo bureau |
| flag_negativado | BOOLEAN | N | Bernoulli p=0.15 | Tem restrição ativa |
| tempo_negativado_meses | TINYINT | Y | — | Meses desde a negativação |
| data_consulta | DATE | N | — | Data da consulta |

---

## 2.12 Tabela: DIM_MACRO

**Descrição:** Indicadores macroeconômicos mensais.  
**Volume:** 102 linhas (Jan/2018 – Jun/2026)  
**Granularidade:** 1 por mês  
**Sem particionamento** (pequena)

| Campo | Tipo | Descrição | Fonte real (aproximada) |
|-------|------|-----------|------------------------|
| competencia | INT | PK — YYYYMM | — |
| ano | SMALLINT | Ano | — |
| mes | TINYINT | Mês 1–12 | — |
| ipca_mensal_pct | DECIMAL(5,3) | Inflação IPCA % no mês | IBGE |
| ipca_acumulado_12m_pct | DECIMAL(6,3) | IPCA acumulado 12 meses | IBGE |
| selic_meta_pct | DECIMAL(5,2) | Taxa SELIC meta (% a.a.) | BACEN |
| cdi_diario_pct | DECIMAL(7,5) | CDI diário | CETIP |
| desemprego_pct | DECIMAL(5,2) | Taxa de desemprego PNAD | IBGE |
| pib_crescimento_pct | DECIMAL(6,3) | Variação % do PIB trimestral (interpolado) | IBGE |
| confianca_consumidor | DECIMAL(6,2) | Índice FGV (base 100) | FGV |
| cambio_usd_brl | DECIMAL(6,3) | Câmbio USD/BRL médio | BACEN |
| ibc_br | DECIMAL(8,3) | IBC-Br (proxy PIB mensal) | BACEN |
| indice_atividade | DECIMAL(7,3) | Índice geral de atividade econômica | IBGE |
| massa_salarial_var_pct | DECIMAL(5,3) | Variação massa salarial real | IBGE |
| credito_pf_var_pct | DECIMAL(5,3) | Variação carteira de crédito PF | BACEN |
| inadimplencia_sistema_pct | DECIMAL(5,2) | Inadimplência do sistema financeiro | BACEN |
| flag_crise | BOOLEAN | Período de crise declarada | Manual |
| descricao_evento | VARCHAR(100) | Descrição do evento macro | Manual |

**Valores de referência críticos para o modelo:**
| Período | SELIC | Desemprego | Evento |
|---------|-------|-----------|--------|
| Jan/2018 | 6,75% | 12,2% | — |
| Mar/2020 | 3,75% | 12,5% | COVID começa |
| Abr/2020 | — | 14,7% | Lockdowns |
| Ago/2020 | 2,00% | 14,9% | SELIC mínima histórica |
| Mar/2021 | 2,75% | 14,7% | Início alta SELIC |
| Dez/2022 | 13,75% | 7,9% | SELIC máxima recente |
| Jun/2026 | 10,50% | 6,2% | Normalização |

---

# PARTE 3 — RELACIONAMENTOS, CARDINALIDADES E MODELO FÍSICO

## 3.1 Mapa de Relacionamentos

```
dim_cliente (2M)
│
├──[1:N]──► fato_contrato (15M)
│                │
│                ├──[1:N]──► fato_parcela (150M)
│                │
│                └──[1:N]──► fato_evento_cobranca (200M)
│                                    │
│                                    └──[N:1]──► dim_agente
│
├──[1:N]──► fato_bureau (96M)       [mensal]
│
├──[1:N]──► fato_interacao_digital (150M)
│
├──[1:N]──► fato_reclamacao (3M)
│
├──[1:N]──► fato_campanha (50M)
│                └──[N:1]──► dim_campanha
│
├──[1:N]──► fato_callcenter (20M)
│
└──[N:1]──► dim_profissao
             dim_macro [join por competencia/mês]
```

## 3.2 Estratégia de Particionamento (Delta Lake / Databricks)

| Tabela | Partition By | Z-Order By | Retention |
|--------|-------------|-----------|-----------|
| fato_contrato | ano_contratacao, tipo_produto | id_cliente, status_contrato | 10 anos |
| fato_parcela | ano_vencimento, mes_vencimento | id_contrato, status_parcela | 10 anos |
| fato_evento_cobranca | ano, mes, canal | id_cliente, dias_atraso | 5 anos |
| fato_interacao_digital | ano, mes | id_cliente, tipo_evento | 3 anos |
| fato_bureau | competencia | id_cliente | 7 anos |
| fato_reclamacao | ano_abertura | id_cliente, gravidade | 5 anos |
| fato_callcenter | ano, mes | id_cliente, motivo | 3 anos |
| dim_cliente | is_current | segmento, estado | Permanente |
| dim_macro | — | competencia | Permanente |

---

# PARTE 4 — FEATURE ENGINEERING (200+ FEATURES)

## 4.1 Features Financeiras (F)

| # | Feature | Fórmula | Frequência | Explicação de Negócio |
|---|---------|---------|-----------|----------------------|
| F01 | ticket_medio_contratos | SUM(valor_contratado) / COUNT(id_contrato) | Mensal | Ticket médio do cliente |
| F02 | exposicao_total | SUM(valor_contratado - valor_amortizado) | Diária | Exposição atual da carteira |
| F03 | comprometimento_renda | SUM(parcela_mensal) / renda_mensal | Mensal | % da renda comprometida com dívidas |
| F04 | ltv_renda | exposicao_total / (renda_mensal * 12) | Mensal | Múltiplo de renda em dívida |
| F05 | taxa_media_ponderada | SUM(taxa_juros_am * saldo) / SUM(saldo) | Mensal | Custo médio da carteira ponderado |
| F06 | crescimento_divida_3m | (exposicao_total_t - exposicao_total_t3) / exposicao_total_t3 | Mensal | Crescimento da dívida em 3 meses |
| F07 | razao_cc_total | saldo_cartao / exposicao_total | Mensal | Proporção de cartão na dívida total |
| F08 | utilizacao_limite_cc | saldo_rotativo / limite_cc | Diária | % do limite de cartão utilizado |
| F09 | variacao_utilizacao_cc | utilizacao_limite_cc_t - utilizacao_limite_cc_t1 | Mensal | Variação mês a mês |
| F10 | qtd_contratos_ativos | COUNT(id_contrato WHERE status='ATIVO') | Diária | Quantidade de contratos ativos |
| F11 | qtd_contratos_atraso | COUNT(id_contrato WHERE dias_atraso > 0) | Diária | Contratos com atraso |
| F12 | max_dias_atraso_12m | MAX(dias_atraso) nos últimos 12m | Mensal | Pior atraso do último ano |
| F13 | soma_dias_atraso_12m | SUM(dias_atraso) nos últimos 12m | Mensal | Total de dias em atraso |
| F14 | valor_em_atraso | SUM(valor_atualizado WHERE status='EM_ATRASO') | Diária | R$ total em atraso |
| F15 | perc_carteira_atraso | valor_em_atraso / exposicao_total | Diária | % da carteira em atraso |
| F16 | receita_juros_estimada | SUM(taxa_juros_am * saldo_devedor) | Mensal | Receita projetada de juros |
| F17 | desconto_concedido_total | SUM(valor_desconto_acordo) | Histórico | Total de descontos em acordos |
| F18 | taxa_desconto_media | desconto_concedido_total / (desconto_concedido_total + recuperado) | Histórico | % médio de desconto |
| F19 | qtd_renegociacoes | COUNT(id_contrato WHERE flag_renegociado=TRUE) | Histórico | Quantas vezes renegociou |
| F20 | tempo_desde_ultima_renegociacao | DATEDIFF(today, MAX(data_contratacao WHERE flag_renegociado)) | Diária | Dias desde última renegociação |
| F21 | valor_pago_pontual_12m | SUM(valor_pago WHERE dias_atraso=0) últimos 12m | Mensal | Pagamentos em dia no período |
| F22 | taxa_pontualidade_12m | qtd_parc_pagas_dia / qtd_parc_vencidas_12m | Mensal | % de parcelas pagas em dia |
| F23 | antecipacoes_12m | COUNT(data_pagamento < vencimento) últimos 12m | Mensal | Pagamentos antecipados |
| F24 | saldo_devedor_normalizado | exposicao_total / patrimonio_estimado | Mensal | Alavancagem patrimonial |
| F25 | custo_efetivo_medio | SUM(taxa_cet_am * saldo) / SUM(saldo) | Mensal | CET médio ponderado |

## 4.2 Features Comportamentais (C)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| C01 | frequencia_login_30d | COUNT(tipo_evento='login') últimos 30d | Diária | Engajamento digital recente |
| C02 | frequencia_login_90d | COUNT(tipo_evento='login') últimos 90d | Semanal | Engajamento trimestral |
| C03 | dias_desde_ultimo_login | DATEDIFF(today, MAX(data_hora WHERE tipo='login')) | Diária | Inatividade digital |
| C04 | duracao_media_sessao | AVG(duracao_sessao) últimos 30d | Semanal | Profundidade de uso |
| C05 | qtd_emissoes_boleto_30d | COUNT(tipo='emissao_boleto') 30d | Mensal | Necessidade de boleto (vs débito) |
| C06 | qtd_consultas_limite_30d | COUNT(tipo='consulta_limite') 30d | Mensal | Pressão por crédito |
| C07 | qtd_simulacoes_emprestimo_90d | COUNT(tipo='simulacao_emprestimo') 90d | Mensal | Intenção de novo crédito |
| C08 | flag_renegociacao_iniciada | MAX(tipo='renegociacao_inicio') 30d | Diária | Iniciou renegociação recentemente |
| C09 | qtd_tickets_abertos_90d | COUNT(tipo='abertura_ticket') 90d | Mensal | Volume de problemas relatados |
| C10 | plataforma_predominante | MODE(plataforma) últimos 90d | Mensal | iOS / Android / Web |
| C11 | acessos_segunda_a_sexta | COUNT(login) dias úteis / total logins | Mensal | Padrão de acesso |
| C12 | acessos_fora_horario | COUNT(login WHERE hora NOT IN 8-18) / total | Mensal | Comportamento atípico |
| C13 | variacao_logins_mom | (logins_mes_atual - logins_mes_ant) / logins_mes_ant | Mensal | Tendência de engajamento |
| C14 | diversidade_produtos | COUNT(DISTINCT tipo_produto) | Histórico | Profundidade do relacionamento |
| C15 | tempo_relacionamento_dias | DATEDIFF(today, data_cadastro) | Diária | Antiguidade do cliente |
| C16 | qtd_atualizacoes_cadastro_12m | COUNT(tipo='atualizacao_cadastro') 12m | Anual | Manutenção do cadastro |
| C17 | flag_desbloqueio_recente | MAX(tipo='desbloqueio_cartao') 30d | Diária | Desbloqueio recente |
| C18 | score_engajamento | f(logins, transações, produtos, tempo) | Semanal | Score interno de engajamento |
| C19 | tendencia_uso_90d | slope(login_count, últimos 90d) | Mensal | Tendência de uso (linear) |
| C20 | qtd_pix_enviados_30d | COUNT(tipo='pagamento_pix') 30d | Diária | Uso do Pix |

## 4.3 Features Temporais (T)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| T01 | meses_desde_cadastro | DATEDIFF(today, data_cadastro)/30 | Diária | Maturidade do cliente |
| T02 | meses_desde_primeiro_contrato | DATEDIFF(today, MIN(data_contratacao))/30 | Diária | Início da relação de crédito |
| T03 | sazonalidade_mes | mes_referencia | Mensal | Mês como feature (1–12) |
| T04 | flag_dezembro | mes=12 | Mensal | Mês de maior consumo |
| T05 | flag_janeiro | mes=1 | Mensal | Mês de maior inadimplência |
| T06 | trimestre | CEIL(mes/3) | Mensal | Trimestre (1–4) |
| T07 | ciclo_vencimento | dia_vencimento_predominante | Mensal | Dia do mês de vencimento |
| T08 | distancia_vencimento | DATEDIFF(proximo_vencimento, today) | Diária | Dias para próximo vencimento |
| T09 | tempo_desde_negativacao | meses desde primeira negativação | Mensal | Histórico de negativação |
| T10 | tempo_ativo_sem_atraso | meses desde último atraso | Mensal | Período de "reabilitação" |
| T11 | idade_contrato_meses | DATEDIFF(today, data_contratacao)/30 | Mensal | Maturidade do contrato |
| T12 | perc_prazo_cumprido | numero_parcela / prazo_meses | Mensal | % do contrato pago |
| T13 | janela_temporal_covid | flag_período abr/2020-dez/2021 | Mensal | Cliente exposto à crise COVID |
| T14 | contratou_crise | flag_data_contratacao IN crise | Histórico | Contrato originado em crise |
| T15 | safra_cliente | YEAR(data_cadastro) * 100 + QUARTER | Histórico | Coorte de entrada |

## 4.4 Features de Cobrança (CB)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| CB01 | qtd_contatos_cobranca_30d | COUNT(id_evento) 30d | Diária | Volume de cobranças recentes |
| CB02 | taxa_resposta_cobranca | COUNT(resultado!='sem_resposta') / COUNT(*) | Mensal | % de vezes que responde |
| CB03 | melhor_canal_contato | Canal com maior taxa_resposta | Mensal | Canal mais eficaz |
| CB04 | qtd_promessas_nao_cumpridas | COUNT(promessa_cumprida=FALSE) histórico | Histórico | Promessas quebradas |
| CB05 | taxa_cumprimento_promessas | COUNT(cumprida) / COUNT(prometida) | Histórico | Confiabilidade de promessa |
| CB06 | valor_recuperado_12m | SUM(valor_pago WHERE flag_acordo) 12m | Anual | Valor recuperado por acordos |
| CB07 | faixa_cobranca_atual | MAX(faixa_cobranca) | Diária | Pior faixa de cobrança atual |
| CB08 | tempo_medio_resposta | AVG(horas entre contato e pagamento) | Histórico | Velocidade de resposta à cobrança |
| CB09 | sensibilidade_canal_whats | taxa_resposta_whatsapp - taxa_media | Mensal | Sobre/sub-responsividade ao WhatsApp |
| CB10 | sensibilidade_canal_tel | taxa_resposta_telefone - taxa_media | Mensal | Sobre/sub-responsividade ao telefone |
| CB11 | qtd_acordos_historico | COUNT(tipo_acao='acordo_fechado') | Histórico | Total de acordos fechados |
| CB12 | perc_acordos_cumpridos | acordos_cumpridos / acordos_fechados | Histórico | Taxa de cumprimento de acordos |
| CB13 | dias_atraso_maximo_historico | MAX(dias_atraso) lifetime | Histórico | Pior momento de inadimplência |
| CB14 | ciclo_inadimplencia | contagem de ciclos atraso>0 → 0 → atraso>0 | Histórico | Recorrência de inadimplência |
| CB15 | custo_cobranca_acumulado | SUM(custo_acao) para este cliente | Histórico | Custo total de cobrança |
| CB16 | roi_cobranca | valor_recuperado / custo_cobranca | Histórico | Retorno das ações de cobrança |
| CB17 | probabilidade_pag_7d | [output do modelo de collection score] | Diária | P(pagamento em 7 dias) |
| CB18 | probabilidade_pag_30d | [output do modelo] | Diária | P(pagamento em 30 dias) |
| CB19 | melhor_horario_contato | hora com maior taxa de resposta | Mensal | Horário ótimo de abordagem |
| CB20 | resistencia_cobranca | score interno de dificuldade de contato | Mensal | Dificuldade de localizar/cobrar |

## 4.5 Features de CRM (CR)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| CR01 | nps_medio | AVG(satisfacao_pos) reclamações | Trimestral | Satisfação média |
| CR02 | qtd_reclamacoes_12m | COUNT(id_reclamacao) 12m | Mensal | Volume de reclamações |
| CR03 | gravidade_media_reclamacoes | AVG(gravidade_num) 12m | Mensal | Gravidade média dos problemas |
| CR04 | tempo_resolucao_medio | AVG(tempo_resolucao_horas) | Histórico | Eficiência de atendimento |
| CR05 | flag_reclamacao_bacen | MAX(canal='BACEN') | Histórico | Já reclamou no BACEN |
| CR06 | flag_reclamacao_reincidente | MAX(reincidente=TRUE) | Histórico | Reclamações recorrentes |
| CR07 | score_satisfacao_geral | f(nps, reclamações, resolução) | Mensal | Score sintético de satisfação |
| CR08 | conversao_campanhas | COUNT(converteu=TRUE) / COUNT(campanha) | Histórico | Taxa de resposta a campanhas |
| CR09 | canal_campanha_preferido | Canal com maior conversão pessoal | Histórico | Canal mais responsivo |
| CR10 | lifetime_campanhas | COUNT(participou) lifetime | Histórico | Total de campanhas participadas |
| CR11 | ultima_campanha_convertida_dias | DATEDIFF(today, MAX(data_conversao)) | Mensal | Recência de conversão |
| CR12 | segmento_atual | segmento do dim_cliente | Diária | Segmento de valor |
| CR13 | historico_upgrades_segmento | COUNT(mudanças para segmento superior) | Histórico | Evolução positiva |
| CR14 | historico_downgrades_segmento | COUNT(mudanças para segmento inferior) | Histórico | Deterioração |
| CR15 | clv_realizado | SUM(receita_atribuída) lifetime | Histórico | CLV realizado |

## 4.6 Features de Atendimento (AT)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| AT01 | qtd_ligacoes_receptivas_30d | COUNT(tipo='R') 30d | Mensal | Volume de ligações recebidas |
| AT02 | qtd_ligacoes_ativas_30d | COUNT(tipo='A') 30d | Mensal | Volume de ligações ativas |
| AT03 | taxa_resolucao_primeiro_contato | COUNT(resolucao_1c) / COUNT(*) | Histórico | Eficiência de atendimento |
| AT04 | tempo_espera_medio | AVG(tempo_espera_seg) | Histórico | Experiência de espera |
| AT05 | taxa_abandono | COUNT(abandonou) / COUNT(*) | Histórico | Frustração no call center |
| AT06 | motivo_contato_predominante | MODE(motivo_contato) | Histórico | Principal razão de contato |
| AT07 | flag_escalado_recente | MAX(transferido=TRUE) 30d | Mensal | Atendimento escalado recente |
| AT08 | csat_medio | AVG(satisfacao) 12m | Trimestral | Satisfação de atendimento |
| AT09 | qtd_transferencias_total | SUM(transferido=TRUE) | Histórico | Total de escalamentos |
| AT10 | complexidade_atendimento | AVG(tempo_atendimento_seg) | Histórico | Complexidade média dos atendimentos |

## 4.7 Features de Bureau (BU)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| BU01 | score_serasa_atual | score_serasa último mês | Mensal | Score bureau atual |
| BU02 | variacao_score_3m | score_t - score_t3 | Mensal | Tendência do score |
| BU03 | variacao_score_6m | score_t - score_t6 | Mensal | Tendência semestral |
| BU04 | score_minimo_12m | MIN(score_serasa) 12m | Anual | Pior score do ano |
| BU05 | qtd_consultas_90d | qtd_consultas_90d bureau | Mensal | Demanda por crédito no mercado |
| BU06 | spike_consultas | qtd_consultas_30d > 2× média | Mensal | Aumento atípico de consultas |
| BU07 | qtd_restricoes_ativas | qtd_restricoes_ativas bureau | Mensal | Restrições no bureau |
| BU08 | valor_total_negativado | valor_negativado bureau | Mensal | Exposição no bureau |
| BU09 | qtd_operacoes_mercado | qtd_operacoes_ativas bureau | Mensal | Endividamento total no SFN |
| BU10 | razao_divida_mercado_renda | valor_total_dividas / renda_presumida_bureau | Mensal | Alavancagem total |
| BU11 | flag_negativado_bureau | flag_negativado | Diária | Tem restrição ativa |
| BU12 | meses_negativado | tempo_negativado_meses | Mensal | Tempo de negativação |
| BU13 | novos_debitos_mercado | qtd_operacoes_t - qtd_operacoes_t3 | Mensal | Novas dívidas no período |
| BU14 | tendencia_divida_mercado | slope(valor_total_dividas, 6m) | Mensal | Tendência de endividamento |
| BU15 | risco_concentracao_bureau | exposicao_fintech / valor_total_dividas | Mensal | Concentração na fintech |

## 4.8 Features Macroeconômicas (ME)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| ME01 | selic_atual | dim_macro.selic_meta_pct | Mensal | Taxa de juros base |
| ME02 | delta_selic_3m | selic_t - selic_t3 | Mensal | Direção da política monetária |
| ME03 | desemprego_atual | dim_macro.desemprego_pct | Mensal | Pressão no mercado de trabalho |
| ME04 | desemprego_variacao_6m | desemprego_t - desemprego_t6 | Mensal | Tendência do desemprego |
| ME05 | ipca_acumulado_12m | dim_macro.ipca_acumulado_12m | Mensal | Inflação acumulada |
| ME06 | ipca_variacao_mom | ipca_t - ipca_t1 | Mensal | Aceleração da inflação |
| ME07 | confianca_consumidor | dim_macro.confianca_consumidor | Mensal | Expectativa de consumo |
| ME08 | cambio_atual | dim_macro.cambio_usd_brl | Mensal | Câmbio (impacto em importados) |
| ME09 | credito_pf_crescimento | dim_macro.credito_pf_var_pct | Mensal | Expansão de crédito no sistema |
| ME10 | inadimplencia_sistema | dim_macro.inadimplencia_sistema_pct | Mensal | Proxy de stress sistêmico |
| ME11 | risco_macro_score | f(selic, desemprego, ipca, confianca) | Mensal | Score sintético de risco macro |
| ME12 | flag_crise | dim_macro.flag_crise | Mensal | Período de crise econômica |
| ME13 | interacao_renda_desemprego | renda_mensal × (1 - desemprego_pct/100) | Mensal | Renda ajustada ao risco macro |
| ME14 | pressao_juros_real | taxa_contrato - ipca_acumulado_12m | Mensal | Spread real pago pelo cliente |
| ME15 | pib_crescimento | dim_macro.pib_crescimento_pct | Trimestral | Crescimento econômico |

## 4.9 Features Geográficas (GE)

| # | Feature | Fórmula | Frequência | Explicação |
|---|---------|---------|-----------|-----------|
| GE01 | regiao | dim_cliente.regiao | Estático | Região geográfica |
| GE02 | estado | dim_cliente.estado | Estático | UF |
| GE03 | pib_per_capita_estado | join dim_geo_estado | Anual | Riqueza regional |
| GE04 | desemprego_estado | join dim_geo_estado | Mensal | Desemprego regional |
| GE05 | inadimplencia_estado | join dim_geo_estado | Mensal | Inadimplência média da UF |
| GE06 | densidade_agencias | bancos_por_100k_hab (estado) | Anual | Bancarização regional |
| GE07 | flag_capital | cidade IN capitais | Estático | Município capital |
| GE08 | flag_municipio_grande | pop_municipio > 500k | Estático | Grande cidade |
| GE09 | indice_desenvolvimento_humano | IDH municipal | Anual | Desenvolvimento do município |
| GE10 | distancia_centro_financeiro | km até SP ou RJ | Estático | Proxy de acesso a serviços |
| GE11 | penetracao_fintech_regiao | clientes_fintech / pop_bancarizada | Anual | Competição regional |
| GE12 | concentracao_carteira_uf | exposicao_uf / exposicao_total_fintech | Mensal | Risco de concentração geográfica |

---

# PARTE 5 — MODELOS DE MACHINE LEARNING

## 5.1 Credit Score (Previsão de Default)

### 5.1.1 Default 90+ (PD90)

| Atributo | Detalhe |
|---------|---------|
| **Definição do target** | `flag_default_90` = 1 se cliente atrasar ≥90 dias nos próximos 6 meses |
| **Janela de observação** | 12 meses de histórico |
| **Janela de performance** | 6 meses futuros |
| **Features principais** | BU01–15, F01–25, T01–T15, ME01–ME15 |
| **Algoritmos** | XGBoost (primário), LightGBM (challenger), Regressão Logística (baseline) |
| **Métrica principal** | KS ≥ 45, AUC-ROC ≥ 0.80 |
| **Métricas secundárias** | Gini, PSI, Lift curva |
| **Frequência de retreinamento** | Trimestral + trigger por PSI > 0.25 |
| **Segmentação** | Por tipo de produto e segmento de cliente |
| **Dados de treinamento** | Safras 2018–2023 |
| **Dados de validação OOT** | Safras 2024–2025 |
| **Cutoff sugerido** | F-score ótimo por faixa de risco |

**Faixas de risco:**
| Faixa | Score | PD estimada | Ação |
|-------|-------|------------|------|
| Muito Baixo | 850–1000 | <1% | Aprovação automática, maior limite |
| Baixo | 700–849 | 1–3% | Aprovação, condições padrão |
| Médio | 550–699 | 3–8% | Análise adicional, limite conservador |
| Alto | 400–549 | 8–20% | Aprovação restrita |
| Muito Alto | <400 | >20% | Recusa ou garantia exigida |

### 5.1.2 Default 180+ (PD180)

Similar ao PD90, janela de performance = 12 meses, mais features macro.

---

## 5.2 Collection Score (Priorização de Cobrança)

### 5.2.1 Probabilidade de Pagamento em 7 dias

| Atributo | Detalhe |
|---------|---------|
| **Target** | `pag_7d` = 1 se pagamento realizado em ≤7 dias após a data de referência |
| **Universo** | Contratos com atraso > 0 dias |
| **Features** | CB01–20, C01–C20, F11–F20, BU01–BU15 |
| **Algoritmo** | LightGBM (rápido, atualização diária) |
| **Métrica** | AUC ≥ 0.75, KS ≥ 35 |
| **Output** | Probabilidade + faixa (Alta/Média/Baixa/Muito Baixa) |
| **Uso** | Priorização da régua de cobrança diária |
| **Retreinamento** | Semanal (dados fluem diariamente) |

### 5.2.2 Probabilidade de Pagamento em 30 dias

Similar, target = pagamento em ≤30 dias, mais features de comportamento.

### 5.2.3 Score de Canal Ótimo

| Atributo | Detalhe |
|---------|---------|
| **Target** | Canal que maximiza P(pagamento | contato via canal X) |
| **Abordagem** | Modelo multi-class ou 1 modelo por canal |
| **Features** | CB01–20, C10, C11, idade, segmento, GE01–02 |
| **Saída** | Ranking de canais por cliente |

---

## 5.3 Churn Prediction

### 5.3.1 Churn de Relacionamento (encerramento de conta)

| Atributo | Detalhe |
|---------|---------|
| **Target** | `flag_churn` = 1 se cliente encerrar conta nos próximos 90 dias |
| **Features principais** | C01–C20, CR01–CR15, AT01–AT10, F01–F10 |
| **Algoritmo** | XGBoost + SHAP para explicabilidade |
| **Métrica** | AUC ≥ 0.82, Precision@K (top 20%) |
| **Frequência** | Semanal |
| **Output** | Score 0–100 + motivo principal (SHAP) |
| **Ação** | Campanha de retenção segmentada |

### 5.3.2 Abandono de Produto (cartão de crédito)

Target: sem transações no cartão por 90 dias consecutivos. Feature: utilização_limite_cc (F08), variacao_utilizacao (F09), qtd_pix (C20).

---

## 5.4 Customer Lifetime Value (CLV)

| Atributo | Detalhe |
|---------|---------|
| **Modelo** | BG/NBD (frequência/recência) + Gamma-Gamma (monetary) |
| **Alternativa DL** | LSTM para séries de receita por cliente |
| **Horizonte** | 12, 24 e 36 meses |
| **Features** | Histórico de transações, contratos, renegociações, cross-sell |
| **Output** | CLV esperado em R$ + segmento de valor futuro |
| **Uso** | Alocação de orçamento de CRM, pricing de aquisição |

---

## 5.5 Detecção de Anomalias / Fraude

| Caso | Algoritmo | Features | Threshold |
|------|-----------|---------|-----------|
| Fraude em pagamento | Isolation Forest + AutoEncoder | valor, horário, localização, dispositivo | Recall ≥ 95% |
| CPF duplicado/fraude cadastral | Gradient Boosting em pares | distância de nomes, endereços, dispositivos | Precision ≥ 80% |
| Comportamento atípico de uso | LSTM Autoencoder | sequência de eventos digitais | Percentil 99 |
| Transação fora do padrão | One-Class SVM | ticket médio, localização, horário | F1 ≥ 0.75 |

---

## 5.6 Recomendação de Campanhas / Next Best Offer

| Atributo | Detalhe |
|---------|---------|
| **Abordagem** | Collaborative Filtering + Content-Based Hybrid |
| **Features** | Histórico de produtos, segmento, perfil, resposta a campanhas |
| **Algoritmos** | Matrix Factorization (ALS), LightFM, Two-Tower Neural Network |
| **Métrica** | MAP@K, NDCG@K |
| **Output** | Top-3 produtos/campanhas recomendados por cliente |
| **Uso** | Personalização do app, email marketing, call center |

---

## 5.7 Forecasting de Inadimplência

| Atributo | Detalhe |
|---------|---------|
| **Granularidade** | Produto × Safra × Mês |
| **Horizonte** | 12 meses à frente |
| **Modelos** | Prophet (baseline), ARIMA-X, LSTM, Temporal Fusion Transformer |
| **Features exógenas** | ME01–ME15 (macro), sazonalidade, crises |
| **Métrica** | MAPE ≤ 8%, WAPE ≤ 6% |
| **Output** | Curva de inadimplência projetada por safra |

---

# PARTE 6 — DISTRIBUIÇÕES ESTATÍSTICAS SUGERIDAS

## 6.1 Distribuições por variável

| Variável | Distribuição | Parâmetros | Justificativa |
|---------|-------------|-----------|--------------|
| renda_mensal | Log-Normal | μ=8.0, σ=0.9 (em log) | Renda tem cauda longa à direita |
| valor_contratado (EP) | Log-Normal | μ=8.9, σ=0.7 | Empréstimos com cauda |
| taxa_juros_am | Normal truncada | μ=0.028, σ=0.008, [0.01, 0.08] | Dispersão por risco e produto |
| dias_atraso | Exponencial (inadimplentes) | λ=0.08 | Maioria atrasa pouco, alguns muito |
| score_bureau | Normal truncada | μ=560, σ=140, [0, 1000] | Distribuição típica de bureaus |
| tempo_espera_callcenter | Exponencial | λ=0.02 | Fila M/M/c |
| qtd_consultas_bureau | Poisson | λ=1.5 | Eventos discretos raros |
| qtd_eventos_cobranca | Poisson | λ=3.2 por mês em atraso | Contatos periódicos |
| duracao_sessao_app | Log-Normal | μ=4.5, σ=1.1 (em log, segundos) | Sessões curtas predominam |
| valor_negativado | Log-Normal | μ=7.5, σ=1.3 | Dívidas negativadas com cauda |
| tempo_resolucao_reclamacao | Log-Normal | μ=4.8, σ=1.5 (horas) | Maioria rápida, alguns casos complexos |

## 6.2 Correlações Estatísticas Obrigatórias

```
score_bureau ←→ dias_atraso:           ρ ≈ -0.55
renda_mensal ←→ comprometimento_renda: ρ ≈ -0.32 (ricos se endividam menos %)
tempo_relacionamento ←→ default:        ρ ≈ -0.28 (clientes antigos inadimplem menos)
qtd_renegociacoes ←→ prob_novo_default: ρ ≈ +0.61
desemprego_macro ←→ inadimplencia:      ρ ≈ +0.73
confianca_consumidor ←→ conversao_camp: ρ ≈ +0.44
qtd_reclamacoes ←→ churn_prob:          ρ ≈ +0.58
logins_30d ←→ churn_prob:               ρ ≈ -0.67
utilizacao_limite_cc ←→ default_90:     ρ ≈ +0.49
```

---

# PARTE 7 — CENÁRIOS REALISTAS E ANOMALIAS INTENCIONAIS

## 7.1 Problemas de Qualidade de Dados (Intencionais)

| Problema | Como Simular | % Afetados | Aprendizado |
|---------|-------------|-----------|------------|
| CPF duplicado | 2% dos CPFs aparecem em 2+ clientes | 40.000 clientes | Deduplicação, entity resolution |
| Cadastro desatualizado | renda_mensal nula em 8% dos clientes cadastrados pre-2019 | 60.000 clientes | Imputation, MCAR/MAR |
| Data de nascimento futura | 0.1% com data_nascimento > data_cadastro | 2.000 clientes | Validação de regras |
| Renegociação circular | Contrato A renegociado em B, B em A | 500 casos | Detecção de ciclos |
| Valor de parcela negativo | 0.05% das parcelas | 7.500 parcelas | Sanitização |
| Score bureau = 0 | 3% sem histórico | 60.000 | Cold-start, thin file |
| Mudança de produto mid-stream | Produto CC virou EP no mesmo contrato | 1.000 contratos | Change data capture |
| Timestamp fora do horário comercial | 0.3% de eventos de cobrança | Detecção de fraude interna |

## 7.2 Sazonalidade Embutida

| Período | Fenômeno | Impacto |
|---------|---------|---------|
| Janeiro | Inadimplência +20%, IPVA, escola | Pico de atraso |
| Março | Imposto de Renda, antecipação restituição | +15% simulações de empréstimo |
| Junho/Julho | Férias, 13º antecipado | +8% pagamento de dívidas |
| Outubro | Black Friday preparação | +25% consultas de limite |
| Novembro | Black Friday | +40% contratações, +12% fraude |
| Dezembro | Natal, 13º salário | -30% inadimplência, +60% contratações |

## 7.3 Efeito COVID-19 (Abr/2020–Dez/2021)

```python
# Multiplicadores a aplicar no período
inadimplencia_mult = 2.1  # Multiplicador de atraso
renegociacao_mult = 3.5   # Explosão de renegociações
digital_acesso_mult = 1.8 # Maior uso do app
contratos_novos_mult = 0.6 # Queda na concessão
score_bureau_reducao = -45 # Queda média do score
campanha_eficiencia_mult = 0.7 # Campanhas menos eficazes
```

---

# PARTE 8 — DASHBOARDS EXECUTIVOS

## 8.1 Dashboard: Diretoria Executiva

**Atualização:** Diária (D-1)  
**Ferramenta sugerida:** Power BI / Looker / Databricks SQL

### KPIs Principais (Big Numbers)
| KPI | Fórmula | Benchmark | Alerta |
|-----|---------|-----------|--------|
| Carteira Total Ativa | SUM(exposicao_total) | — | Crescimento < 5% a.m. |
| Inadimplência 90+ (NPL) | valor_atraso_90+ / carteira_total | < 4.5% | > 6% |
| Taxa de Recuperação | recuperado_30d / em_atraso_inicio | > 35% | < 25% |
| Receita de Juros (MR) | SUM(receita_juros_estimada) | — | Margem < 15% |
| Churn Mensal | clientes_encerrados / clientes_ativos | < 1.5% | > 2.5% |
| NPS | (promotores - detratores) / total × 100 | > 40 | < 20 |
| CAC | custo_aquisicao / novos_clientes | < R$120 | > R$200 |
| LTV/CAC | clv_12m / cac | > 4× | < 2× |

### Visualizações
- Gráfico de estoque da carteira por produto (bar stacked)
- Curva de vintage de inadimplência por safra
- Mapa de calor por estado (inadimplência)
- Tendência de receita vs custo de crédito (12 meses)
- Funil de conversão de campanhas

---

## 8.2 Dashboard: Operação de Cobrança

**Atualização:** Horária  
**Usuário:** Gerentes e supervisores de cobrança

| KPI | Fórmula | Meta |
|-----|---------|------|
| Taxa de Contato | contatos_realizados / tentativas | > 60% |
| Taxa de Promessa | promessas / contatos | > 30% |
| Taxa de Cumprimento de Promessa | cumpridas / prometidas | > 70% |
| Taxa de Recuperação D+7 | recuperado_7d / carteira_acionada | > 18% |
| Custo por Real Recuperado | custo_cobrança / recuperado | < R$0,15 |
| Produtividade por Agente | recuperado / agente | > R$15.000/dia |

### Visualizações
- Ranking de eficiência por canal (tabela + bar chart)
- Evolução da régua de cobrança por faixa
- Heatmap de melhor horário por canal
- Funil de cobrança: tentativas → contato → promessa → pagamento
- Alertas de promessas vencendo hoje

---

## 8.3 Dashboard: Data Science / MLOps

**Atualização:** Diária  
**Usuário:** Times de Data Science e Analytics

| Métrica | Modelo | Alerta |
|---------|--------|--------|
| AUC-ROC | PD90, Collection, Churn | Queda > 2pp vs baseline |
| KS | PD90 | < 35 |
| PSI (Population Stability) | Todos | > 0.25 = retreinar |
| Precision@20% | Churn, Collection | Queda > 3pp |
| MAPE Forecasting | Inadimplência | > 10% |
| Feature Drift | Top-20 features | > 2σ da distribuição de treino |
| Latência de Predição | Todos | > 200ms p99 |
| Cobertura de Score | % clientes com score válido | < 95% |

---

# PARTE 9 — ARQUITETURA MLOPS (DATABRICKS + AWS/GCP)

## 9.1 Arquitetura de Dados (Medallion)

```
┌─────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                            │
│  CRM · Core Bancário · Bureau · App Events · Call Center · Macro│
└────────────────────────────┬────────────────────────────────────┘
                             │ Kafka / Kinesis (streaming)
                             │ S3 / GCS (batch)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      BRONZE LAYER (Raw)                         │
│  Delta Lake · S3/GCS · Sem transformação · Compactação diária   │
│  Retenção: 5 anos · Formato: Parquet + Delta                    │
└────────────────────────────┬────────────────────────────────────┘
                             │ Databricks Auto Loader (incremental)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SILVER LAYER (Clean)                       │
│  Deduplicação · Validação · Tipagem · SCD2 para dimensões       │
│  Join com dimensões · Particionamento por data + entidade       │
└────────────────────────────┬────────────────────────────────────┘
                             │ Spark Structured Streaming + Batch
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      GOLD LAYER (Business)                      │
│  Aggregações · Feature Store · Marts analíticos · KPIs          │
│  Tabelas prontas para BI e ML                                   │
└─────────────┬──────────────────────────┬────────────────────────┘
              │                          │
              ▼                          ▼
     FEATURE STORE              DATABRICKS SQL
     (Tecton / Feast /          (Dashboards, Ad-hoc,
      Databricks FS)             Relatórios regulatórios)
              │
              ▼
     MODEL TRAINING
     (MLflow + Databricks ML)
              │
              ▼
     MODEL REGISTRY
     (MLflow Registry · Staging → Production → Archived)
              │
              ▼
     SERVING / SCORING
     (Batch: Databricks Jobs diário
      Real-time: MLflow Serving / SageMaker Endpoint)
              │
              ▼
     MONITORING
     (Evidently AI · Grafana · PagerDuty alertas)
```

## 9.2 Feature Store — Estrutura

```
feature_store/
├── client_financial_features/
│   ├── version: v3.2
│   ├── update_freq: daily
│   ├── features: F01–F25
│   └── entity_key: id_cliente
│
├── client_behavioral_features/
│   ├── version: v2.1
│   ├── update_freq: daily
│   ├── features: C01–C20
│   └── entity_key: id_cliente
│
├── collection_features/
│   ├── version: v4.0
│   ├── update_freq: daily (streaming for CB17/CB18)
│   ├── features: CB01–CB20
│   └── entity_key: id_cliente, id_contrato
│
├── bureau_features/
│   ├── version: v1.8
│   ├── update_freq: monthly
│   ├── features: BU01–BU15
│   └── entity_key: id_cliente
│
└── macro_features/
    ├── version: v1.0
    ├── update_freq: monthly
    ├── features: ME01–ME15
    └── entity_key: competencia
```

## 9.3 Pipeline de Retreinamento (MLflow + Databricks)

```python
# Pseudocódigo do pipeline de retreinamento
class RetrainingPipeline:
    triggers = [
        PSITrigger(threshold=0.25, features=TOP_20),
        ScheduledTrigger(cron="0 2 1 */3 *"),  # Trimestral
        ManualTrigger()
    ]
    
    steps = [
        DataValidation(),          # Great Expectations
        FeatureGeneration(),       # Silver → Gold → Feature Store
        TrainTestSplit(           # Temporal split
            train_end="T-6M",
            validation="T-6M to T-3M",
            oot_test="T-3M to T"
        ),
        ModelTraining([            # Challenger models
            XGBoostTrainer(hyperopt_trials=200),
            LightGBMTrainer(hyperopt_trials=150),
        ]),
        ModelEvaluation(           # Gates de qualidade
            min_auc=0.78,
            min_ks=38,
            max_psi=0.15
        ),
        ChampionChallengerTest(    # A/B em produção
            traffic_split=0.10,    # 10% no challenger
            duration_days=14
        ),
        ModelPromotion(),          # Se challenger ganhar
        ModelDocumentation()       # Automática via MLflow
    ]
```

## 9.4 CI/CD para ML

```
Git Push (feature branch)
    │
    ▼
GitHub Actions / GitLab CI
    ├── Unit tests (pytest)
    ├── Data validation (Great Expectations)
    ├── Feature parity check
    └── Shadow scoring (novo modelo vs produção)
    │
    ▼ (se aprovado)
Staging Environment
    ├── Backtesting em dados históricos
    ├── Business rules validation
    └── Performance gate check
    │
    ▼ (sign-off humano)
Production Deployment
    ├── Blue/Green deployment
    ├── Canary (5% → 25% → 100%)
    └── Rollback automático se métricas caírem
```

---

# PARTE 10 — ESTRATÉGIA DE GERAÇÃO DOS DADOS

## 10.1 Ordem de Geração (Dependências)

```
1. dim_macro         → base temporal (102 meses)
2. dim_profissao     → dimensão de suporte
3. dim_cliente       → 2M clientes com correlações
4. fato_bureau       → histórico mensal por cliente
5. fato_contrato     → 15M contratos (dependem do cliente + macro)
6. fato_parcela      → ~150M parcelas (dependem do contrato)
7. fato_evento_cobr  → ~200M eventos (dependem de parcelas atrasadas)
8. fato_interacao    → ~150M eventos (dependem do cliente)
9. fato_reclamacao   → ~3M reclamações (correlacionadas com atraso)
10. fato_campanha    → ~50M participações (segmentação por perfil)
11. fato_callcenter  → ~20M chamadas (correlacionadas com eventos)
```

## 10.2 Correlações a Implementar na Geração

```python
# Exemplo de lógica de geração correlacionada

def gerar_dias_atraso(cliente, contrato, macro_mes):
    # Base: distribuição exponencial
    base_atraso = np.random.exponential(scale=12)
    
    # Ajustes por fatores de risco
    mult_score = map_score_to_multiplier(cliente.score_bureau)  # score baixo → mais atraso
    mult_renegociacao = 2.1 if contrato.flag_renegociado else 1.0
    mult_macro = 1 + (macro_mes.desemprego_pct - 12) * 0.08
    mult_covid = 2.1 if macro_mes.flag_crise else 1.0
    mult_sazonalidade = get_sazonalidade(macro_mes.mes)
    mult_comprometimento = 1 + max(0, cliente.comprometimento_renda - 0.30) * 1.5
    
    atraso_final = base_atraso * mult_score * mult_renegociacao * \
                   mult_macro * mult_covid * mult_sazonalidade * mult_comprometimento
    
    return max(0, int(atraso_final))
```

## 10.3 Tecnologia de Geração Recomendada

| Ferramenta | Uso | Volume |
|-----------|-----|--------|
| Python + Faker | Dados cadastrais | dim_cliente |
| NumPy/SciPy | Distribuições estatísticas | Todas |
| PySpark | Geração em escala | fato_* |
| SDV (Synthetic Data Vault) | Estrutura de correlações | Validação |
| Mimesis | Dados brasileiros (CPF, CEP) | dim_cliente |
| Databricks Notebooks | Orquestração e armazenamento | Pipeline completo |

## 10.4 Armazenamento (Databricks + AWS S3)

```
s3://fintech-synthetic-data/
├── bronze/
│   ├── clientes/year=2018/month=01/...
│   ├── contratos/year=2018/...
│   └── ...
├── silver/
│   ├── dim_cliente/           (Delta, Z-ordered by segmento)
│   ├── fato_contrato/         (Delta, partitioned by ano/produto)
│   └── ...
├── gold/
│   ├── feature_store/
│   ├── marts/
│   │   ├── mart_cobranca/
│   │   ├── mart_credito/
│   │   └── mart_clientes/
│   └── kpis/
└── ml/
    ├── training_datasets/
    ├── models/               (MLflow artifacts)
    └── predictions/
```

---

# PARTE 11 — ROADMAP DE ESTUDOS

## Nível de Chegada: Cientista de Dados Sênior / Analytics Engineer Sênior em Fintech

## 11.1 Fase 1 — Fundamentos de Dados (Meses 1–2)

### Módulo 1.1: SQL Avançado
**Exercícios práticos com esta base:**

```sql
-- Exercício 1: Vintage de inadimplência por safra
SELECT
    DATE_FORMAT(data_contratacao, '%Y-%m') AS safra,
    tipo_produto,
    COUNT(*) AS contratos,
    SUM(CASE WHEN dias_atraso >= 30 THEN 1 ELSE 0 END) AS em_atraso_30,
    SUM(CASE WHEN dias_atraso >= 90 THEN 1 ELSE 0 END) AS em_atraso_90,
    AVG(dias_atraso) AS media_atraso,
    SUM(CASE WHEN dias_atraso >= 90 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS npL_90
FROM fato_contrato c
JOIN fato_parcela p ON c.id_contrato = p.id_contrato
WHERE p.vencimento BETWEEN data_contratacao AND DATE_ADD(data_contratacao, 180)
GROUP BY 1, 2
ORDER BY 1, 2;

-- Exercício 2: Eficiência de canal de cobrança
WITH canal_stats AS (
    SELECT
        canal,
        faixa_cobranca,
        COUNT(*) AS tentativas,
        SUM(promessa_pagamento) AS promessas,
        SUM(promessa_cumprida) AS cumpridas,
        SUM(custo_acao) AS custo_total,
        SUM(CASE WHEN promessa_cumprida THEN valor_prometido ELSE 0 END) AS recuperado
    FROM fato_evento_cobranca
    WHERE data_hora >= DATEADD(month, -3, CURRENT_DATE)
    GROUP BY canal, faixa_cobranca
)
SELECT
    canal,
    faixa_cobranca,
    tentativas,
    ROUND(promessas * 1.0 / tentativas, 3) AS taxa_promessa,
    ROUND(cumpridas * 1.0 / NULLIF(promessas,0), 3) AS taxa_cumprimento,
    ROUND(recuperado / NULLIF(custo_total, 0), 2) AS roi,
    ROUND(custo_total / NULLIF(recuperado, 0), 3) AS custo_por_real_recuperado
FROM canal_stats
ORDER BY roi DESC;
```

**Tópicos:** Window functions, CTEs, lateral joins, pivôs, calendar tables, gap analysis

### Módulo 1.2: Modelagem de Dados
- Conceitual → Lógico → Físico
- Normalização vs desnormalização (OLAP vs OLTP)
- SCD Type 1, 2 e 4
- Modelagem dimensional (Kimball)

---

## 11.2 Fase 2 — Engenharia de Dados (Meses 3–4)

### Módulo 2.1: Apache Spark
```python
# Exercício: Calcular features de cobrança em escala
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Window por cliente, ordenado por data
w_cliente = Window.partitionBy("id_cliente").orderBy("data_hora")
w_cliente_30d = Window.partitionBy("id_cliente")\
    .orderBy(F.col("data_hora").cast("long"))\
    .rangeBetween(-30*86400, 0)

df_features_cobranca = df_eventos\
    .withColumn("qtd_contatos_30d",
        F.count("id_evento").over(w_cliente_30d))\
    .withColumn("taxa_promessa_historica",
        F.avg(F.col("promessa_pagamento").cast("int")).over(w_cliente))\
    .withColumn("ultima_promessa_cumprida",
        F.last("promessa_cumprida", ignorenulls=True).over(w_cliente))
```

**Tópicos:** DataFrames, Spark SQL, otimização de jobs, broadcast joins, shuffle partitions, Adaptive Query Execution

### Módulo 2.2: Databricks
- Delta Lake (ACID, time travel, schema evolution)
- Auto Loader (ingestão incremental)
- Databricks Workflows (orquestração)
- Unity Catalog (governança)
- Databricks SQL (BI)

---

## 11.3 Fase 3 — Analytics Engineering (Meses 5–6)

### Módulo 3.1: dbt
```yaml
# models/gold/mart_cobranca.yml
version: 2
models:
  - name: mart_cobranca_diario
    description: "KPIs diários de cobrança por faixa e canal"
    columns:
      - name: dt_referencia
        tests: [not_null, unique]
      - name: roi_canal
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 100
```

**Tópicos:** Modelos, testes, documentação, lineage, seeds, snapshots, macros

### Módulo 3.2: Feature Engineering
- 200+ features desta especificação
- Tratamento de missing values (MCAR, MAR, MNAR)
- Encoding (target encoding, WOE)
- Feature selection (IV, SHAP, permutation importance)

---

## 11.4 Fase 4 — Machine Learning (Meses 7–9)

### Módulo 4.1: Credit Score (PD90)
- Construção do dataset temporal (evitar data leakage)
- Fine-classing e coarse-classing (scorecard)
- XGBoost com Optuna
- Validação: KS, Gini, AUC, PSI, Curva ROC
- Scorecard calibrado (binning, pontuação)

### Módulo 4.2: Collection Score
- Multi-target (7d, 30d, 60d)
- Atualização diária com Spark
- Output para régua de cobrança

### Módulo 4.3: Churn
- Definição correta do target (evite vazamento)
- SHAP values para explicabilidade
- Threshold optimization (custo-benefício)

### Módulo 4.4: Séries Temporais
- Prophet para forecasting de inadimplência
- LSTM para CLV
- Temporal Fusion Transformer

---

## 11.5 Fase 5 — MLOps (Meses 10–12)

### Módulo 5.1: MLflow
- Tracking (experimentos, métricas, parâmetros)
- Model Registry (staging → production)
- Serving (batch + real-time)

### Módulo 5.2: Monitoramento
- Data drift (PSI, KS test)
- Model drift (performance degradation)
- Feature monitoring (Evidently AI)
- Alertas automáticos

### Módulo 5.3: CI/CD para ML
- Versionamento de código e dados
- Testes automatizados para modelos
- Deployment automatizado

---

## 11.6 Fase 6 — Especialização em Crédito (Meses 11–12, paralelo)

| Tema | Conteúdo |
|------|---------|
| Regulação | Resolução 4.966 BACEN (IFRS 9), Basilea III, LGPD |
| Provisão | PD × LGD × EAD (Expected Credit Loss) |
| RAROC | Risk-Adjusted Return on Capital |
| Open Banking | API BACEN, compartilhamento de dados |
| Pix Fraud | Detecção de fraude em tempo real |

---

## 11.7 Mapa de Competências por Trilha

### Trilha A: Analytics Engineer Sênior
```
SQL Avançado ████████████ 100%
dbt          ████████████ 100%  
Spark        ████████░░░░  75%
Python       ████████░░░░  75%
BI/Dashboards████████████ 100%
ML (consumo) ██████░░░░░░  50%
Crédito      ████████░░░░  75%
```

### Trilha B: Data Scientist Sênior em Fintech
```
Python/Sklearn████████████ 100%
ML Avançado  ████████████ 100%
Feature Eng  ████████████ 100%
MLOps        ████████░░░░  75%
SQL          ████████░░░░  75%
Spark        ██████░░░░░░  50%
Crédito/Cobr ████████████ 100%
```

---

# PARTE 12 — EXEMPLOS DE REGISTROS

## 12.1 dim_cliente

```json
{
  "id_cliente": 1847293,
  "sk_cliente": 3291847,
  "cpf": "35219847263",
  "sexo": "F",
  "data_nascimento": "1988-07-14",
  "estado_civil": "Casado",
  "escolaridade": "Superior",
  "profissao_cod": "CLT_PRIV_ADM",
  "cidade": "São Paulo",
  "estado": "SP",
  "regiao": "SE",
  "renda_mensal": 6800.00,
  "patrimonio_estimado": 85000.00,
  "score_bureau_entrada": 698,
  "data_cadastro": "2020-03-15",
  "segmento": "Plus",
  "canal_aquisicao": "App",
  "status_cliente": "Ativo",
  "flag_pep": false,
  "flag_fraude": false,
  "is_current": true
}
```

## 12.2 fato_contrato

```json
{
  "id_contrato": 9283741,
  "id_cliente": 1847293,
  "tipo_produto": "EP",
  "valor_contratado": 12000.00,
  "prazo_meses": 24,
  "taxa_juros_am": 0.0289,
  "taxa_cet_am": 0.0312,
  "valor_parcela": 620.45,
  "sistema_amortizacao": "Price",
  "data_contratacao": "2021-06-10",
  "data_primeiro_vencimento": "2021-07-10",
  "status_contrato": "QUITADO",
  "canal_contratacao": "App",
  "score_decisao": 712,
  "flag_renegociado": false
}
```

## 12.3 fato_evento_cobranca

```json
{
  "id_evento": 47829301,
  "id_cliente": 2918374,
  "id_contrato": 11293847,
  "data_hora": "2023-03-14T14:32:00",
  "canal": "whatsapp",
  "tipo_acao": "cobranca_ativa",
  "resultado_contato": "promessa_pag",
  "promessa_pagamento": true,
  "valor_prometido": 850.00,
  "data_promessa": "2023-03-17",
  "promessa_cumprida": true,
  "custo_acao": 0.12,
  "dias_atraso_momento": 22,
  "faixa_cobranca": "P1"
}
```

## 12.4 dim_macro

```json
{
  "competencia": 202004,
  "ano": 2020,
  "mes": 4,
  "ipca_mensal_pct": -0.31,
  "ipca_acumulado_12m_pct": 2.40,
  "selic_meta_pct": 3.75,
  "desemprego_pct": 14.7,
  "pib_crescimento_pct": -1.50,
  "confianca_consumidor": 78.2,
  "cambio_usd_brl": 5.38,
  "inadimplencia_sistema_pct": 3.10,
  "flag_crise": true,
  "descricao_evento": "COVID-19: lockdowns, auxílio emergencial, moratória"
}
```

---

# APÊNDICE — GLOSSÁRIO E REFERÊNCIAS

## Termos Técnicos

| Termo | Definição |
|-------|-----------|
| PD | Probability of Default — probabilidade de inadimplência |
| LGD | Loss Given Default — perda dado o default (1 - recuperação) |
| EAD | Exposure at Default — exposição no momento do default |
| ECL | Expected Credit Loss = PD × LGD × EAD |
| NPL | Non-Performing Loan — crédito com atraso ≥90 dias |
| PSI | Population Stability Index — estabilidade da população |
| KS | Kolmogorov-Smirnov — discriminância do modelo |
| WOE | Weight of Evidence — transformação de variáveis categóricas |
| IV | Information Value — poder preditivo de uma variável |
| SCD2 | Slowly Changing Dimension Type 2 — versionamento de histórico |
| OOT | Out-of-Time — validação fora do período de treino |
| CET | Custo Efetivo Total — custo real do crédito ao consumidor |
| CLV | Customer Lifetime Value — valor do cliente ao longo do tempo |
| CAC | Custo de Aquisição de Cliente |
| RAROC | Risk-Adjusted Return on Capital |
| Vintage | Coorte de contratos originados no mesmo período |
| Safra | Sinônimo de vintage na terminologia brasileira |
| Régua | Sequência automatizada de ações de cobrança |

## Regulação Relevante

| Norma | Tema |
|-------|------|
| Res. BACEN 4.966/2021 | IFRS 9 — Provisão para devedores duvidosos |
| Res. BACEN 4.282/2013 | Requerimentos de capital (Basileia III) |
| Lei 14.181/2021 | Superendividamento — limita práticas abusivas |
| LGPD (Lei 13.709/2018) | Proteção de dados pessoais |
| Res. BACEN 1/2020 | Open Banking / Open Finance |
| Circular BACEN 3.978/2020 | Prevenção à lavagem de dinheiro (PLD) |

---

*Documento gerado por equipe multidisciplinar de especialistas em Data Engineering, Data Science, Analytics Engineering, Crédito & Cobrança e ML. Versão 1.0 — Junho 2026.*
