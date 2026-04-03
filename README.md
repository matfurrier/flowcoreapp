# FlowCore - Plataforma de Gestão de Processos e Workflows

O **FlowCore** é um **Motor de Workflows (Workflow Engine)** flexível e escalável, projetado para digitalizar e orquestrar qualquer processo de negócio da **De Sangosse**. 

A arquitetura é **agnóstica ao processo**: toda a lógica de etapas, formulários, regras de validação e transições é definida em schemas JSON/Python, permitindo que novos fluxos sejam implementados rapidamente sem alterar o código-fonte principal.

> **Status Atual**: A plataforma está em produção com o módulo de **Gestão da Qualidade (RNC)** como seu primeiro caso de uso. **Autenticação e controle de acesso** implementados (6 fases concluídas).

## 🚀 Diferenciais da Plataforma

### 1. Arquitetura Multi-Fluxo (Schema-Driven)
O coração do FlowCore é um interpretador de schemas que renderiza interfaces e valida regras dinamicamente.
- **Criação Rápida**: Novos processos (ex: Auditorias, Solicitação de Compras, Manutenção) são criados apenas definindo um arquivo de configuração (Seed).
- **Zero-Code UI**: O Frontend (`FormRenderer`) se adapta automaticamente aos campos, tipos de dados e layouts definidos no Backend.
- **Versionamento**: Suporte a múltiplas versões do mesmo processo rodando simultaneamente.

### 2. Motor de Regras Poderoso
- **Validação Condicional**: Regras como `required_if` (Obrigatório SE...) configuráveis via JSON.
- **Visibilidade Condicional**: Regras `visible_if` para exibir/ocultar widgets dinamicamente baseado em seleções do usuário.
- **Transições Complexas**: Gateways de aprovação, retorno para ajustes e bifurcações de fluxo.
- **Visibilidade Dinâmica**: Campos podem ser ocultos, somente-leitura ou editáveis dependendo do status atual ou do perfil do usuário.

### 3. Componentes Ricos (Widgets)
Agrega valor aos processos com componentes visuais avançados que vão além de formulários simples:
- **Seções Agrupadas (`section_group`)**: Organização visual de campos com propagação de visibilidade.
- **Análise de Causa Raiz**: Diagramas interativos (Ishikawa/Espinha de Peixe) com textos de ajuda em cada categoria 6M.
- **Planos de Ação**: Grids editáveis (5W2H) com rastreamento de status e prazos.
- **Gestão de Evidências**: Upload de múltiplos anexos integrado ao MinIO S3.
- **Assinatura e Aprovação**: Rastreabilidade completa de quem fez o quê e quando.

### 4. Gestão de SLA e Alertas
- **Alertas de Prazo**: Sistema de notificações para prazos expirados (`SLA_ALERT`).
- **Background Tasks**: Verificação periódica automática de deadlines.
- **Indicadores Visuais**: Badge "Expirado" em campos de data que ultrapassaram o prazo limite.

### 5. Autenticação e Controle de Acesso
- **Login obrigatório**: Autenticação via banco Security (Argon2/bcrypt), JWT com departamento e perfil admin.
- **Filtros por usuário**: Torre de Controle e listagem filtradas — usuário vê apenas instâncias onde é criador, responsável ou no payload.
- **Admins**: Perfil com acesso total; usuários do TI são admin automaticamente; menu para administrar usuários.
- **Links públicos**: Colaboradores externos acessam etapas específicas via link sem login (token com expiração).
- **Refinamentos**: Refresh token, auditoria de acesso público, notificações por email (estrutura), mapeamento grupo-departamento.

### 6. Integrações e Saída
- **Gerador de Relatórios Universal**: O serviço de PDF (`PdfService`) é capaz de gerar relatórios finais fidelizados para qualquer fluxo, utilizando templates customizáveis (Jinja2).
- **Audit Trail Imutável**: Todas as operações geram logs de auditoria para compliance.
- **API de Notificações**: Endpoint `/notifications` para consulta e gerenciamento de alertas.

---

## 🔐 Autenticação — 6 Fases Implementadas

| Fase | Descrição |
|------|-----------|
| **1** | Infraestrutura: login, tabela `flowcore_app_users`, middleware JWT, rotas protegidas, tela de login |
| **2** | Departamento: integração com Security, `department_id`/`department_name` no token e perfil |
| **3** | Admins: coluna `is_admin`, script `seed_admins.py`, badge Admin, menu "Administrar usuários" |
| **4** | Filtros: `created_by`, `assigned_to`, visibilidade por payload e admin; Torre de Controle e listagem filtradas |
| **5** | Links públicos: tabela `public_access_tokens`, GET/POST `/public/instance/{token}`, página `/public/[token]`, botão "Gerar link público" |
| **6** | Refinamentos: refresh token, auditoria de acesso público, notificações por email, mapeamento grupo-departamento |

---

## 📂 Módulos Implementados

### ✅ Qualidade: Relatório de Não Conformidade (RNC)
O primeiro processo digitalizado na plataforma, substituindo o fluxo manual.

### ✅ RH: Requisição de Aumento de Quadro
Solicitação ao RH para aumento de quadro de colaboradores. Baseado na planilha `legacy/Requisição de Aumento de Quadro.xlsx`. Fluxo de aprovação: Solicitante → RH (MVP para testes).
1.  **Abertura**: Registro detalhado com lote, produto e fornecedor (organizado em seções).
2.  **Triagem**: Validação inicial pela equipe da Qualidade.
3.  **Investigação (6Ms & 5 Porquês)**: Ferramentas de qualidade integradas com exibição condicional baseada na abordagem selecionada.
4.  **Plano de Ação (5W2H)**: Acompanhamento de eficácia com controle de SLA.
5.  **Relatório Final**: Geração automática de PDF oficial.

---

## 🛠️ Arquitetura Técnica

### Backend (`/backend`)
- **Framework**: FastAPI (Python 3.12+)
- **Banco de Dados**: PostgreSQL (SQLAlchemy AsyncIO)
- **Armazenamento**: MinIO (S3 Compatible) para arquivos e PDFs.
- **Relatórios**: Jinja2 + xhtml2pdf.
- **Tasks**: Background tasks para verificação de SLA.

### Frontend (`/frontend`)
- **Framework**: Next.js 14 (App Router)
- **UI Components**: TailwindCSS + Shadcn/UI.
- **Gerenciamento de Estado**: React Hook Form + Zod (Schema Validation).
- **Renderização Dinâmica**: `FormRenderer` com suporte a widgets condicionais e seções agrupadas.

---



## 🧪 Estrutura de Pastas Relevante

```
/backend
  /app
    /models      # Modelos do BD (SQLAlchemy) - inclui Notification
    /schemas     # Schemas Pydantic
    /services    # Lógica de Negócio (Instance, PDF, Workflow, SLA)
    /routers     # Endpoints API (inclui /notifications)
    /tasks       # Background tasks (deadline_checker)
    /templates   # Templates HTML para Emails/PDFs
  /scripts       # Scripts de Seed e Migração

/frontend
  /src
    /components
      /form      # FormRenderer (Renderizador de Workflow)
      /widgets   # Widgets Customizados (Ishikawa, Upload, AnalysisGrid)
    /lib         # Utilitários (Schema Builder c/ visible_if, API Client)
```
