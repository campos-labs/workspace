# Workspace & Monorepo

Repositório central de projetos em Web, Dados e IA.

## Estrutura do Repositório

```text
/
├── docs/
│   └── rfcs/                           # Propostas antes do início da execução
│       ├── _template/        
│       │   └── DESIGN.md
│       ├── 003-projeto-c/          
│       │   └── DESIGN.md
│       └── 004-projeto-d/           
│           └── DESIGN.md           
├── projetos/
│   ├── 001-projeto-a/    
│   │   ├── src/                        # Código-fonte
│   │   ├── DESIGN.md                   # Documento vivo (estado atual, arquitetura, requisitos, premissas)
│   │   └── README.md                   # Instruções de uso locais
│   └── 002-projeto-b/      
│       ├── src/                
│       ├── ARCHITECTURE.md             # O autor escolhe a nomenclatura ideal
│       └── README.md           
├── README.md                           # Visão geral do monorepo
└── CONTRIBUTING.md                     # Fluxo de trabalho e regras operacionais

```
