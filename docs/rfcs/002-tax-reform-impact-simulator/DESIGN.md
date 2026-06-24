# Design Architecture: Simulador Analítico da Reforma Tributária

## 🎯 Objetivo (Problema de Negócio)
Transformar obrigações acessórias complexas (EFD ICMS/IPI e ECD) em cenários projetados e determinísticos de impacto no fluxo de caixa frente à Reforma Tributária. O sistema substitui análises em planilhas por um motor de banco de dados e relatórios narrativos gerados por IA, garantindo escalabilidade e rastreabilidade (*Source Trace*).

## ⚙️ Escopo da Solução
* **In:** Upload de arquivos TXT brutos do SPED e configurações de cenários.
* **Core:** Processamento OLAP (DuckDB) em camadas (Staging, Canônica e Analítica). Aplicação da matemática de transição (ICMS "por dentro", IBS/CBS "por fora").
* **Out:** UI Triunvirato (Sliders, Waterfall, Data Grid) e Laudo Executivo em linguagem natural.
* **Fora do Escopo:** Entrega de obrigações ao fisco; RAG para cálculos matemáticos.

## 👥 Usuários e Papéis
1. **Analista Tributário / Consultor:** Faz o upload dos arquivos, parametriza as regras de transição e avalia a DRE projetada.
2. **Diretoria (C-Level):** Consome os relatórios narrativos executivos exportados pela plataforma.

## 🧩 Modelo de Domínio e Dados (Arquitetura Híbrida)
A v1.0 opera em modo single-tenant (isolamento multi-tenant previsto como evolução). O sistema utiliza **PostgreSQL** para transações (OLTP) e **DuckDB** para o motor analítico massivo (OLAP).

### Metadados e Parametrização (PostgreSQL)
* `Company`: `id`, `cnpj`, `name`.
* `TaxRuleSet`: `id`, `version`, `valid_from`, `valid_to`, `ibs_rate`, `cbs_rate`, `source_reference`, `notes`, `created_at`. (*Nota: `TaxRuleSet` é append-only; alterações criam nova versão, sem atualização destrutiva, permitindo reproduzir simulações anteriores com as mesmas premissas versionadas*).
* `RawFile`: `id`, `company_id`, `file_type` (EFD, ECD), `storage_path`, `checksum`.
* `Scenario`: `id`, `tax_rule_set_id`, `year`, `simples_supplier_share`.
* `SimulationRun`: `id`, `scenario_id`, `tax_rule_set_id`, `raw_file_ids`, `input_checksum`, `status`, `started_at`, `finished_at`. (Funciona como um *snapshot* imutável dos insumos utilizados na execução).

### Camadas de Processamento (DuckDB)
Cada `SimulationRun` gera um artefato DuckDB isolado, identificado pelo ID da simulação, evitando concorrência de escrita entre execuções paralelas.
1. **Staging:** `stg_efd_c170`, `stg_ecd_i200`.
2. **Canônica:** `dim_account_mapping`.
3. **Analítica (`fact_dre_simulation`):** Onde a regra determinística acontece: `(preco_liquido * cbs_rate)` + `(preco_liquido * ibs_rate)`.
   **Campos de Rastreabilidade (Source Trace):** `source_raw_file_id`, `source_line_number`, `source_record_type`, `calculation_rule_id`.

## 🔌 Serviços de Aplicação e Integrações
* **`ExecutiveReportGenerator` (LangGraph):** Recebe exclusivamente o JSON sumarizado da DRE. Uma *chain* de agentes analisa as variações percentuais e redige um sumário narrativo sem recalcular números, usando apenas os valores recebidos no JSON consolidado.
* **`BeneficioFiscalProvider`:** Motor matemático que aplica a redução escalonada (2029-2033) de incentivos regionais.

## 📡 Contratos de API (Endpoints FastAPI)
* `POST /api/v1/companies/{id}/files/` - Upload e registro do `RawFile` no Postgres.
* `POST /api/v1/scenarios/` - Cria ou clona um cenário associando um `TaxRuleSet`.
* `POST /api/v1/simulations/run/` - Dispara o DuckDB, processa os arquivos e gera as *views* analíticas.
* `GET /api/v1/simulations/{id}/report/` - Consulta o laudo executivo já processado de forma assíncrona. A geração pelo LangGraph ocorre em background a partir do JSON consolidado, evitando chamadas síncronas longas à LLM.

## 🛡️ Segurança e Rastreabilidade
* **Source Trace:** Cada linha gerada no Data Grid final carrega metadados apontando para o arquivo físico e a linha exata que originou o cálculo.

## 🧪 Observabilidade, Testes e DoD
* **Golden Files (Testes de Regressão):** Testes unitários focados na matemática. *Dado o arquivo sintético X e a regra Y, o lucro líquido projetado tem que ser exatamente Z.*
* **Definition of Done (DoD):** Simulação da v1.0 com dataset sintético de referência executando em até 10 segundos em ambiente local documentado; API retornando JSON consistente; relatório narrativo gerado exclusivamente a partir do JSON consolidado, sem recalcular números.

## 📎 Referências
* [LC nº 214/2025](https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp214.htm), [Decreto nº 12.955/2026](https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/decreto/d12955.htm) e [Resolução CGIBS nº 6/2026](https://www.cgibs.gov.br/resolucoes) — base legal da Reforma Tributária do Consumo, CBS e IBS.
* Leiautes SPED: [EFD-Contribuições](https://www.gov.br/sped/pt-br/assuntos/escrituracoes-digitais/efd-contribuicoes/manuais/guia_pratico_efd_contribuicoes_versao_1_35-18_06_2021.pdf/@@display-file/file), [EFD ICMS/IPI](https://sped.rfb.gov.br/item/show/8112) e [ECD](https://www.gov.br/sped/pt-br/assuntos/escrituracoes-digitais/ecd/manuais-e-documentos-tecnicos/manual_de_orientacao_da_ecd_leiaute_9_janeiro_2026.pdf/@@display-file/file).
* [Curso CFC/RFB — Reforma Tributária do Consumo](https://www.youtube.com/watch?v=94Y7E74L96g&t=12551s) e materiais de apoio: [IVA Dual](https://cfc.org.br/wp-content/uploads/2026/05/Modulo_01_Fernando_Mombelli_SLIDE-01.pdf) e [Normas Gerais](https://cfc.org.br/wp-content/uploads/2026/05/Modulo_01_Roni_Peterson_SLIDE-02.pdf).
* Premissas internas e sintéticas de modelagem contábil para cenários determinísticos de impacto.
