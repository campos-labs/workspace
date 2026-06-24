# Design Architecture: ## Portal de Onboarding e Homologação de Fornecedores

## 🎯 Objetivo (Problema de Negócio)
Centralizar, padronizar e auditar o processo de entrada de novos fornecedores corporativos. O sistema mitiga riscos de *compliance* ao exigir validação documental (*Human-in-the-Loop*) e dados autocompletados a partir de fontes públicas antes de integrar o cadastro ao ERP de destino.

## ⚙️ Escopo da Solução
* **In:** Convite gerado pelo comprador interno; autocompletar de dados via BrasilAPI; upload de PDFs e extração assistiva (OCR simplificado na v1.0).
* **Core:** Fila de revisão de documentos com visualização lado a lado.
* **Out:** Exportação assíncrona, tolerante a falhas e idempotente (Outbox Pattern) para o ERP corporativo em payload versionado inspirado no conceito de Business Partner.
* **Fora do Escopo:** Aprovação autônoma via IA; gestão financeira e pedidos de compra.

## 👥 Usuários e Papéis
1. **Comprador:** Dispara o convite (Gatilho).
2. **Fornecedor (Externo):** Preenche o cadastro e fornece a documentação comprobatória.
3. **Analista de Compliance:** Revisa os documentos submetidos contra os dados digitados e toma a decisão final.

## 🧩 Modelo de Domínio e Dados
A persistência primária é relacional, isolando os documentos pesados em Object Storage.

### Infraestrutura Local
* **PostgreSQL:** Persistência relacional.
* **MinIO:** Armazenamento de documentos.
* **Redis:** Broker de mensageria para o Taskiq.

### Entidades Principais (Django Models)
* `Supplier`: `id`, `cnpj`, `legal_name`, `trade_name`, `tax_status`, `address_json`, `created_at`.
* `OnboardingRequest`: `id`, `supplier_id`, `status`, `token_hash`, `expires_at`, `submitted_at`, `reviewer_id`.
* `OnboardingRequestHistory`: `id`, `request_id`, `actor_id`, `from_status`, `to_status`, `reason`, `created_at`.
* `Document`: `id`, `request_id`, `doc_type` (Contrato, CND), `file_path`, `file_sha256`, `expires_at`, `validation_status`, `ocr_status`.
* `ErpExportEvent` (Outbox): `id`, `request_id`, `event_type`, `payload`, `payload_version`, `idempotency_key`, `status` (PENDING, PROCESSING, SENT, FAILED, DEAD_LETTER), `attempts`, `next_retry_at`, `locked_at`, `processed_at`, `last_error`.

### 🔄 Máquina de Estados (`OnboardingRequest.status`)
```text
DRAFT (Aguardando Fornecedor)
 └── SUBMITTED (Documentos Anexados)
      └── PENDING_REVIEW (Fila do Analista)
           ├── APPROVED (Gera evento no Outbox)
           ├── NEEDS_REVISION (Devolve ao fornecedor para ajuste documental)
           └── REJECTED (Recusa definitiva por risco de compliance ou inconsistência)

```

## 🔌 Serviços de Aplicação e Integrações

* **CnpjProvider (`BrasilAPIAdapter`):** Consulta síncrona consumida no momento da digitação do CNPJ no frontend do fornecedor.
* **DocumentReaderPort (`OcrAssistantAdapter`):** OCR simplificado e não bloqueante. Após o upload, o documento fica com `ocr_status=PENDING` e o Taskiq processa a extração assistiva em background. Falhas de OCR não impedem a revisão humana.
* **Outbox Processor (`Taskiq Worker`):** Worker assíncrono via Redis que faz *polling* na tabela `ErpExportEvent` buscando registros `PENDING`. Tenta realizar o POST no Webhook do ERP (Mock na v1.0).
Após o limite máximo de tentativas, o evento passa para `DEAD_LETTER`, preservando `payload`, `attempts` e `last_error` para auditoria e reprocessamento manual.

## 📡 Contratos de API (Endpoints Rest)

* `POST /api/v1/invitations/` - Cria o convite e gera o token.
* `GET /api/v1/onboarding/{token}/` - Retorna o estado atual do formulário.
* `POST /api/v1/onboarding/{token}/supplier-data/` - Salva o payload preenchido.
* `POST /api/v1/onboarding/{token}/documents/` - Recebe `multipart/form-data`, valida MIME type/tamanho via backend e armazena o arquivo no MinIO.
* `POST /api/admin/requests/{id}/approve/` - Executa a transição para `APPROVED` e a criação do `ErpExportEvent` estritamente dentro da mesma transação atômica (`transaction.atomic()`).

## 🛡️ Segurança e Privacidade

* **Tokens de Sessão:** O acesso do fornecedor é *token-based* com tempo de expiração curto (ex: 72 horas).
* **Upload Seguro:** Validação do *MIME type* (`application/pdf`) e limite de payload (10MB).
* **Auditoria:** Toda transição de estado gera um registro imutável em `OnboardingRequestHistory`, preservando ator, data, status anterior, novo status e motivo.

## 🧪 Observabilidade, Testes e DoD

* **Testes (Pytest):** Cobertura obrigatória de ponta a ponta na view de aprovação -> verificação de inserção correta na tabela Outbox -> mock do envio do Webhook.
* **Logs:** Estruturados em JSON capturando a transição da máquina de estados.
* **Definition of Done (DoD):** Feature implementada, coberta por testes, migrações do banco geradas e `docker-compose up` executando o fluxo completo com sucesso.

## 📎 Referências

* [Canal Fornecedor Petrobras](https://canalfornecedor.petrobras.com.br/) e [Petronect](https://www.petronect.com.br/irj/portal/anonymous/pt) — inspiração para cadastro, homologação e relacionamento com fornecedores.
* [BrasilAPI — CNPJ](https://brasilapi.com.br/docs#tag/CNPJ) — consulta e autocompletar de dados cadastrais.
* [SAP — Managing Suppliers and Supplier Lifecycles](https://help.sap.com/docs/strategic-sourcing/managing-suppliers-and-supplier-lifecycles/managing-suppliers-and-supplier-lifecycles) — base conceitual para ciclo de vida de fornecedores, homologação e integração de dados cadastrais.
* [Lei nº 13.709/2018 — LGPD](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm) e [ISO/IEC 27001](https://www.iso.org/standard/27001) — referências para privacidade, segurança da informação e controles de acesso.
