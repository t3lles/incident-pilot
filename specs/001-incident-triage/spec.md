# Feature Specification: IncidentPilot - API de Triagem de Incidentes

**Feature Branch**: `001-incident-triage`  
**Created**: 2026-10-28  
**Status**: Draft  
**Input**: User description: "Crie a especificação técnica para o IncidentPilot: uma API em .NET 8 (Minimal APIs) e SQL Server (via EF Core) para triagem de incidentes. O sistema deve ter endpoints para registrar chamados em texto livre, enviar à API do Gemini para categorização estruturada (Severidade, Componente, Diagnóstico, Ações) com fallback resiliente caso a IA falhe, e persistir o resultado no SQL Server. Detalhe os contratos dos endpoints REST, os schemas JSON de entrada/saída e o modelo relacional da tabela de incidentes."

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro e Triagem Automática de Incidente (Priority: P1)

Como engenheiro de suporte ou operador de sistemas, desejo registrar um chamado de incidente descrevendo o problema em texto livre para que o sistema analise e categorize automaticamente o incidente utilizando inteligência artificial, retornando diagnósticos e ações recomendadas imediatamente.

**Why this priority**: É a funcionalidade principal do produto (MVP). Permite automatizar o fluxo de atendimento de incidentes e acelerar a triagem de problemas críticos.

**Independent Test**: Pode ser testado enviando um payload contendo uma descrição de texto livre para o endpoint de criação de incidentes e verificando se o incidente retornado/salvo contém os campos de triagem da IA preenchidos (`Severidade`, `Componente`, `Diagnostico`, `Acoes`).

**Acceptance Scenarios**:

1. **Given** um texto livre descrevendo uma falha no sistema (ex: "Banco de dados fora do ar com erro de timeout no checkout"), **When** a solicitação é enviada via HTTP POST `/api/v1/incidents`, **Then** o sistema deve chamar a API do Gemini, receber a categorização estruturada e retornar HTTP 201 Created com o status `TriagemConcluida`, contendo os detalhes categorizados.
2. **Given** uma requisição de registro de incidente válida, **When** o processamento é concluído com sucesso, **Then** os dados originais e a triagem estruturada da IA devem ser persistidos no SQL Server.

---

### User Story 2 - Fallback Resiliente na Falha da IA (Priority: P2)

Como operador do sistema, desejo que o registro de incidentes permaneça funcional mesmo quando o serviço de IA estiver indisponível, sofrendo com restrições de cota ou timeouts, garantindo que nenhum chamado seja perdido.

**Why this priority**: Resiliência é um princípio arquitetural fundamental. O sistema não pode interromper a operação ou rejeitar incidentes por conta de instabilidade de serviços de terceiros.

**Independent Test**: Pode ser testado simulando um erro/timeout/falha na API do Gemini (ou chave inválida) ao registrar um incidente e verificando se o sistema persiste o chamado no banco com status `TriagemPendente` e retorna HTTP 201 Created.

**Acceptance Scenarios**:

1. **Given** que a API do Gemini está inacessível, retornando HTTP 5xx ou excedendo o tempo limite (timeout), **When** um usuário envia um novo incidente em texto livre, **Then** o sistema deve capturar a falha sem lançar exceção não tratada ao cliente, persistir o incidente no banco com o status `TriagemPendente` e retornar HTTP 201 Created informando que o registro foi salvo para triagem manual/reprocessamento.

---

### User Story 3 - Consulta e Auditoria de Incidentes (Priority: P3)

Como gestor de TI ou analista de suporte, desejo consultar os incidentes registrados por ID e listar chamados por status de triagem para acompanhar o volume de falhas e auditar o desempenho da triagem.

**Why this priority**: Permite visibilidade, acompanhamento do histórico e reconciliação de incidentes pendentes de triagem.

**Independent Test**: Pode ser testado executando requisições HTTP GET `/api/v1/incidents/{id}` e GET `/api/v1/incidents?status=TriagemPendente` e validando o retorno estruturado dos dados salvos no banco.

**Acceptance Scenarios**:

1. **Given** um ID de incidente válido existente no banco de dados, **When** é enviada uma requisição GET `/api/v1/incidents/{id}`, **Then** o sistema deve retornar HTTP 200 OK com os dados completos do incidente e métricas de telemetria da IA (se houver).
2. **Given** incidentes salvos com diferentes status (`TriagemConcluida` e `TriagemPendente`), **When** é enviada uma requisição GET `/api/v1/incidents?status=TriagemPendente`, **Then** o sistema deve retornar uma lista contendo apenas os incidentes que falharam na triagem automática.

---

### Edge Cases

- **Texto livre extremamente curto ou sem contexto**: O sistema deve aceitar e enviar à IA, que deve responder com a melhor interpretação possível ou categorizar severidade como `Low`/`Medium` e sugerir detalhamento manual.
- **Entrada vazia ou somente espaços em branco**: O endpoint deve rejeitar a requisição no nível de validação de entrada antes de chamar a IA, retornando HTTP 400 Bad Request.
- **Incompatibilidade ou quebra no JSON gerado pela IA**: O parser de validação estrita deve capturar a divergência de esquema, acionar o mecanismo de fallback e salvar o registro com status `TriagemPendente`.
- **Injeção de instrução/Prompt Injection no texto livre do chamado**: A camada de integração com a IA deve isolar o input do usuário dentro do parâmetro de contexto/conteúdo da mensagem, prevenindo alteração das instruções de sistema da IA.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST disponibilizar um endpoint REST HTTP POST `/api/v1/incidents` para receber chamados descritos em texto livre.
- **FR-002**: O sistema MUST validar que o campo de descrição em texto livre não seja nulo, vazio ou composto apenas por espaços, retornando HTTP 400 em caso de descumprimento.
- **FR-003**: O sistema MUST estruturar as requisições para a API do Gemini utilizando validação estrita baseada em esquemas (*Structured Outputs / JSON Schema*).
- **FR-004**: A resposta categorizada da IA MUST conter obrigatoriamente quatro atributos tipados: `severidade` (enum restrito: `Low`, `Medium`, `High`, `Critical`), `componenteAfetado` (string), `resumoDiagnostico` (string) e `acoesMitigacao` (lista de strings).
- **FR-005**: Em caso de falha na comunicação com a API do Gemini (timeout, erro HTTP 5xx, erro de autenticação ou estouro de cota), o sistema MUST aplicar *Graceful Degradation*, registrando o incidente com o status `TriagemPendente` no SQL Server e retornando HTTP 201 Created.
- **FR-006**: O sistema MUST registrar telemetria estruturada em cada chamada à IA (tempo de resposta em milissegundos e consumo de tokens do prompt/resposta), persistindo estes dados na tabela de incidentes para fins de auditoria e controle de custos.
- **FR-007**: O sistema MUST disponibilizar o endpoint REST HTTP GET `/api/v1/incidents/{id}` para consultar um incidente específico por seu identificador único.
- **FR-008**: O sistema MUST disponibilizar o endpoint REST HTTP GET `/api/v1/incidents` com suporte a filtro opcional por `status` (ex: `TriagemConcluida`, `TriagemPendente`).
- **FR-009**: Todas as alterações de estado e persistência dos incidentes MUST utilizar Entity Framework Core com mapeamento fortemente tipado para a base de dados SQL Server.

---

### Contratos dos Endpoints REST e Schemas JSON

#### 1. Endpoint: Registrar Incidente

- **Método**: `POST`
- **Rota**: `/api/v1/incidents`
- **Headers**: `Content-Type: application/json`

##### Request Body Schema (`POST /api/v1/incidents`)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["titulo", "descricao"],
  "properties": {
    "titulo": {
      "type": "string",
      "minLength": 3,
      "maxLength": 150,
      "description": "Título resumido do incidente ou chamado"
    },
    "descricao": {
      "type": "string",
      "minLength": 10,
      "maxLength": 4000,
      "description": "Relato detalhado em texto livre do problema enfrentado"
    }
  }
}
```

##### Exemplo de Request Body

```json
{
  "titulo": "Lentidão crítica no fluxo de checkout",
  "descricao": "Usuários estão relatando erro HTTP 504 Gateway Timeout ao tentar finalizar compras no carrinho desde às 14:00. O serviço de pagamentos parece não responder a tempo."
}
```

##### Response Body Schema - Sucesso na Triagem (`201 Created`)

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "titulo": "Lentidão crítica no fluxo de checkout",
  "descricao": "Usuários estão relatando erro HTTP 504 Gateway Timeout ao tentar finalizar compras no carrinho desde às 14:00. O serviço de pagamentos parece não responder a tempo.",
  "status": "TriagemConcluida",
  "triagem": {
    "severidade": "Critical",
    "componenteAfetado": "Gateway de Pagamentos / Checkout API",
    "resumoDiagnostico": "Timeout recorrente na comunicação entre a API de Checkout e o Gateway externo de processamento de pagamentos.",
    "acoesMitigacao": [
      "Verificar latência e status do provedor externo de pagamentos.",
      "Redirecionar tráfego para adquirente secundária se aplicável.",
      "Aumentar temporariamente o tempo de timeout do circuito de pagamentos."
    ]
  },
  "telemetria": {
    "tempoRespostaMs": 842,
    "tokensConsumidos": 215
  },
  "criadoEm": "2026-10-28T14:05:00Z"
}
```

##### Response Body Schema - Fallback / Triagem Pendente (`201 Created`)

```json
{
  "id": "8f93e112-9842-4a11-823d-1a89b0123ef4",
  "titulo": "Lentidão crítica no fluxo de checkout",
  "descricao": "Usuários estão relatando erro HTTP 504 Gateway Timeout...",
  "status": "TriagemPendente",
  "triagem": null,
  "telemetria": null,
  "mensagemAlerta": "A triagem automática via IA esteve indisponível. O incidente foi registrado para triagem manual.",
  "criadoEm": "2026-10-28T14:05:00Z"
}
```


---

#### 2. Endpoint: Consultar Incidente por ID

- **Método**: `GET`
- **Rota**: `/api/v1/incidents/{id}`

##### Response (`200 OK`)

Retorna a estrutura completa do objeto incidente exibida no `201 Created`.

##### Response (`404 Not Found`)

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Incidente não encontrado",
  "status": 404,
  "detail": "Nenhum incidente foi localizado com o ID '3fa85f64-5717-4562-b3fc-2c963f66afa6'."
}
```

---

#### 3. Endpoint: Listar Incidentes

- **Método**: `GET`
- **Rota**: `/api/v1/incidents?status={status}`
- **Parâmetros de Query**: `status` (opcional: `TriagemConcluida`, `TriagemPendente`)

##### Response (`200 OK`)

```json
[
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "titulo": "Lentidão crítica no fluxo de checkout",
    "status": "TriagemConcluida",
    "severidade": "Critical",
    "componenteAfetado": "Gateway de Pagamentos / Checkout API",
    "criadoEm": "2026-10-28T14:05:00Z"
  }
]
```

---

### Key Entities e Modelo Relacional da Tabela de Incidentes

A persistência no SQL Server utiliza uma estrutura de tabela única e otimizada para consulta e auditoria rápida:

#### Key Entity: `Incident`
Representa um chamado de incidente registrado no sistema, contendo o relato original, status do fluxo, resultado da análise pela IA e métricas de telemetria.

#### Tabela SQL Server: `Incidentes`

| Nome da Coluna | Tipo SQL Server | Requerido | Descrição / Restrição |
|---|---|---|---|
| `Id` | `UNIQUEIDENTIFIER` | Sim | Chave Primária (`PK`), gerado via `Guid.NewGuid()` |
| `Titulo` | `NVARCHAR(150)` | Sim | Título do incidente |
| `DescricaoOriginal` | `NVARCHAR(4000)` | Sim | Texto livre inserido pelo usuário no chamado |
| `Status` | `NVARCHAR(30)` | Sim | Status da triagem (`TriagemConcluida`, `TriagemPendente`) |
| `Severidade` | `NVARCHAR(20)` | Não | Nulo se `TriagemPendente`. Valores: `Low`, `Medium`, `High`, `Critical` |
| `ComponenteAfetado` | `NVARCHAR(200)` | Não | Nulo se `TriagemPendente`. Identificação do módulo afetado |
| `ResumoDiagnostico` | `NVARCHAR(2000)` | Não | Nulo se `TriagemPendente`. Diagnóstico preliminar da IA |
| `AcoesMitigacaoJson` | `NVARCHAR(MAX)` | Não | Array JSON serializado contendo as ações de mitigação |
| `TempoRespostaMs` | `INT` | Não | Tempo em ms levado para processar a chamada na IA |
| `TokensConsumidos` | `INT` | Não | Quantidade de tokens consumidos na operação de IA |
| `MensagemErroFallback`| `NVARCHAR(1000)` | Não | Detalhes internos da exceção capturada em caso de fallback |
| `CriadoEm` | `DATETIMEOFFSET` | Sim | Data e hora de criação no padrão UTC |
| `AtualizadoEm` | `DATETIMEOFFSET` | Não | Data e hora da última modificação |

#### Índices Recomendados

- `IX_Incidentes_Status`: Índice B-Tree na coluna `Status` para otimizar filtragem na listagem.
- `IX_Incidentes_CriadoEm`: Índice na coluna `CriadoEm` ordenado decrescente para suporte a paginação por data.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% dos incidentes registrados devem ser gravados no SQL Server com sucesso, independentemente do status de disponibilidade da API externa de IA (tolerância zero a falha na persistência).
- **SC-002**: A triagem via IA de um chamado em texto livre deve responder em menos de 3.000 milissegundos no 95º percentil de requisições operacionais.
- **SC-003**: 100% das respostas geradas pela IA quando a chamada for bem-sucedida devem cumprir o contrato estruturado em 4 campos (`severidade`, `componenteAfetado`, `resumoDiagnostico`, `acoesMitigacao`).
- **SC-004**: O mecanismo de fallback resiliente deve responder em até 100 milissegundos após detectar a falha/timeout da API de IA, registrando o estado `TriagemPendente`.

---

## Assumptions

- **SGBD**: O banco SQL Server estará disponível em um container Docker acessível via string de conexão válida configurada em `appsettings.json`.
- **API Gemini**: Uma chave de API válida para o serviço Gemini (`GEMINI_API_KEY`) será injetada via variáveis de ambiente ou configuração local.
- **Validação de Entrada**: O título e o texto do chamado não excederão 150 e 4.000 caracteres respectivamente.
- **Autenticação**: Por se tratar do escopo do MVP técnico de triagem, a autenticação de usuários e gerenciamento de perfis no endpoint REST não é exigida no escopo desta feature v1.

