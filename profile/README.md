# Arquitetura da solução DogMode

Componentes e ligações da solução de cadastro e saneamento de materiais da org Dog-Mode
(Hackathon VX), levantados nos repositórios em 2026-11-08.

<img width="1600" height="1172" alt="Diagrama da solução DogMode" src="https://github.com/user-attachments/assets/8d1723df-a421-4662-91b2-8cdd281b25dd" />

Versão editável: [`arquitetura.drawio`](arquitetura.drawio). Ao mudar o diagrama, edite o `.drawio`,
exporte de novo o `.svg` pelo draw.io (Arquivo › Exportar como › SVG) e atualize este arquivo.

## Fluxos

- **Cadastro.** O usuário usa o frontend, que chama o gateway pelo proxy `/api/gw`. O gateway valida
  o token do Cognito e encaminha a chamada ao cadastro-service. O cadastro-service usa o Claude para
  classificar e extrair, grava no PostgreSQL e, quando o worker-service existir, pede a ele as
  buscas e o enriquecimento.
- **Saneamento.** O frontend chama a API do saneamento-service pelo proxy `/api/saneamento`, sem
  passar pelo gateway. No modo agente, o agente LLM chama as ferramentas do servidor MCP, que rodam o
  pipeline. No modo direto, a API chama o pipeline sem LLM. O storage lê a base e a PDM e grava a
  saída e a auditoria no disco local, no S3 ou no Athena.
- **Carga de dados.** As planilhas do desafio passam pelo etl-materiais e entram uma vez no
  PostgreSQL (`cadastro-importar`). No plano de armazenamento do [`CLAUDE.md`](CLAUDE.md), elas
  também sobem para o bucket S3 por carga manual.

## Componentes

| Componente | Tipo | O que faz |
|---|---|---|
| Usuário (cadastrista) | pessoa | Usa o sistema pelo navegador. Pertence ao grupo `User` do Cognito |
| frontend | serviço do time | Next.js 16 e React 19. Tem dois proxies de servidor: `/api/gw` (gateway) e `/api/saneamento` (saneamento-service) |
| Amazon Cognito | serviço externo (AWS) | User pool com o grupo `User` |
| gateway-service · M1 | serviço do time | Rust e Actix Web 4. Valida o token do Cognito e encaminha `/v1/*` ao cadastro-service. Roda no ECS Fargate (`us-east-1`) |
| cadastro-service · M3 | serviço do time | Python e FastAPI. Pipeline de cadastro em 8 etapas, materiais, filas, propostas, saneamento por lotes e outbox |
| Rag-Tools | biblioteca | Biblioteca Python que o cadastro-service importa para buscar a classe PDM (BM25) |
| worker-service | planejado | Busca de similares e de exemplos, enriquecimento e leitura do outbox. O repositório ainda não existe: o cadastro-service usa um stub local |
| PostgreSQL 17 + pgvector | dados | Schemas `cadastro` e `busca` |
| LLM · Claude API | serviço externo | Usado pelo cadastro-service e pelo agente do saneamento. No saneamento, a OpenAI é opcional |
| etl-materiais | ferramenta | Lê as planilhas e gera um SQLite para a carga inicial. Também existe em `DogMode/etl` |
| Planilhas do desafio | dados | Base de materiais, pedidos e PDM |
| saneamento-service | serviço do time (este repositório) | Python, FastAPI e MCP. Partes internas na tabela abaixo |
| Disco local | dados | `data/` e `runs/` |
| Bucket S3 | planejado | Pastas `arquivos_originais/`, `arquivos_sanitizados/` e `agent_logs/` |
| Athena + Glue | serviço externo (AWS), opcional | Leitura da base e da PDM como tabelas e publicação do resultado |

### Partes do saneamento-service

| Parte | O que faz |
|---|---|
| API HTTP · FastAPI | Recebe os pedidos em `/api/saneamentos`. A CLI `saneamento` faz o mesmo pela linha de comando |
| Agente LLM | No máximo 12 passos, justificativa em toda chamada e auditoria gravada antes de agir |
| Servidor MCP (stdio) | Ferramentas para sanear, verificar e listar |
| Pipeline sem LLM | Decide cada par pela regra 10 da PDM e une os cadastros só dentro dos guardrails da união |
| Storage | Local, S3 ou Athena. Lê a base e a PDM, grava a saída e a auditoria |

## Ligações

| De | Para | Rótulo | Linha |
|---|---|---|---|
| Usuário | frontend | (sem rótulo) | contínua |
| Usuário | Amazon Cognito | login no navegador (F7, pendente) | tracejada |
| frontend | gateway-service | `/api/gw/v1/*` com Bearer (Cognito) | contínua |
| gateway-service | Amazon Cognito | valida o JWT (JWKS) | contínua |
| gateway-service | cadastro-service | `/v1/*` com `X-Internal-Token` e `X-User-*` | contínua |
| cadastro-service | Rag-Tools | importa | pontilhada |
| cadastro-service | PostgreSQL | schema `cadastro` | contínua |
| cadastro-service | worker-service | REST | tracejada |
| worker-service | PostgreSQL | schema `busca` | tracejada |
| cadastro-service | Claude API | Classifica, Extrai, pergunta | contínua |
| frontend | API HTTP do saneamento | `/api/saneamento/*`, direto, fora do gateway | contínua |
| API HTTP | Agente LLM | modo agente | contínua |
| Agente LLM | Servidor MCP | ferramentas | contínua |
| Servidor MCP | Pipeline | sanear / verificar | contínua |
| API HTTP | Pipeline | modo direto (sem LLM) | contínua |
| Agente LLM | Storage | auditoria | contínua |
| Pipeline | Storage | lê e grava | contínua |
| Agente LLM | Claude API | modelo | contínua |
| Storage | Disco local | caminho local | contínua |
| Storage | Bucket S3 | `s3://` | contínua |
| Storage | Athena + Glue | `athena://` | contínua |
| Planilhas do desafio | etl-materiais | lê | pontilhada |
| etl-materiais | PostgreSQL | `cadastro-importar` (carga inicial) | pontilhada |
| Planilhas do desafio | Bucket S3 | carga manual (Fase 0 do `CLAUDE.md`) | tracejada |

## Legenda

| No diagrama | Significado |
|---|---|
| Caixa azul | Serviço do time |
| Caixa verde | Biblioteca ou ferramenta |
| Caixa ou cilindro laranja | Dados |
| Caixa cinza | Serviço externo (AWS, LLM) |
| Borda tracejada | Planejado: ainda não existe |
| Linha contínua | Chamada (HTTP, MCP ou função) |
| Linha tracejada | Planejada ou pendente |
| Linha pontilhada | Import ou carga de dados |
