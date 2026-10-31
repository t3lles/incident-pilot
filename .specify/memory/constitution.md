<!--
Sync Impact Report:
- Version change: Initial scaffold -> 1.0.0
- List of modified principles:
  - Added I. Separação de Responsabilidades (Layered Architecture)
  - Added II. Confiabilidade da IA (Schema-First Validation)
  - Added III. Resiliência e Fallbacks (Graceful Degradation)
  - Added IV. Observabilidade e Auditoria (Structured Telemetry)
- Added sections:
  - Stack Tecnológica e Decisões de Arquitetura
  - Estrutura do Projeto e Organização de Código
- Removed sections: None
- Follow-up TODOs: None
-->

# IncidentPilot Constitution

## Core Principles

### I. Separação de Responsabilidades (Layered Architecture)
A arquitetura do sistema MUST manter rígida isolação entre as camadas do projeto.
- **Camada de Domínio e Contratos**: Contém entidades de negócio, DTOs de comunicação e esquemas formais de contrato com a inteligência artificial.
- **Camada de Infraestrutura**: Gerencia clientes HTTP e SDKs externos de LLM, bem como repositórios de acesso a dados SQL Server via Entity Framework Core ou Dapper.
- **Camada de Aplicação e Endpoints**: Orquestra fluxos de trabalho do negócio por meio de ASP.NET Core Minimal APIs sem expor detalhes de infraestrutura.

*Rationale*: A separação clara entre domínio, infraestrutura e transporte impede vazamento de abstrações, facilita testes unitários isolados e permite trocar dependências externas sem impactar regras de negócio.

### II. Confiabilidade da IA (Schema-First Validation)
Todas as interações com modelos de linguagem (LLMs) MUST seguir o modelo de validação estrita baseada em esquemas (*Structured Outputs*).
- Nenhuma resposta não estruturada ou texto bruto proveniente da LLM pode ser gravado diretamente na base de dados sem passar por desserialização e verificação formal de contrato.
- A classificação gerada pela IA MUST conter obrigatoriamente quatro campos validados: `Severidade` (enum restrito a Low, Medium, High, Critical), `Componente Afetado`, `Resumo do Diagnóstico` e `Ações de Mitigação Sugeridas`.
- DTOs de integração e validação MUST utilizar `C# Records` combinados com FluentValidation ou DataAnnotations para garantir a integridade dos contratos.

*Rationale*: Saídas de LLM são estocásticas. Garantir validação estrita no ponto de desserialização impede corrupção de dados e falhas em tempo de execução nas camadas subsequentes.

### III. Resiliência e Fallbacks (Graceful Degradation)
Falhas na API de IA (timeout, erro 5xx ou esgotamento de quota/rate limiting) MUST NOT interromper a disponibilidade do serviço ou impedir a criação do incidente.
- Em caso de falha de comunicação ou rejeição pelo modelo de IA, a aplicação MUST capturar a exceção e persistir o incidente com o status de triagem `TriagemPendente`.
- O processamento de incidentes com `TriagemPendente` será redirecionado para análise manual ou reprocessamento assíncrono.

*Rationale*: A resiliência do sistema primário de registro de falhas não pode depender da disponibilidade contínua de serviços externos de IA.

### IV. Observabilidade e Auditoria (Structured Telemetry)
Todas as operações e chamadas externas à API de IA MUST emitir registros de log estruturado detalhados.
- Os logs de chamadas à LLM MUST registrar o identificador único do incidente associado, tempo de resposta em milissegundos, total de tokens consumidos (prompt e resposta) e status do resultado (sucesso ou fallback).
- Históricos de triagem e alterações de estado de incidentes MUST ser auditáveis na base de dados SQL Server.

*Rationale*: Métricas de consumo, latência e diagnóstico detalhado são vitais para controle de custos, otimização de prompts e auditoria pós-incidente.

## Stack Tecnológica e Decisões de Arquitetura

O ecossistema do IncidentPilot é padronizado com os seguintes componentes e restrições:
- **Linguagem & Runtime**: C# / .NET 8 utilizando ASP.NET Core Minimal APIs para endpoints leves e eficientes.
- **Acesso a Dados**: Entity Framework Core e/ou Dapper com migrações versionadas e fortemente tipadas.
- **Banco de Dados**: Microsoft SQL Server executado em container Docker.
- **Integração LLM**: Gemini API ou OpenAI API via SDK oficial com suporte a JSON Schema/Structured Outputs.
- **Containerização**: Docker e Docker Compose orquestrando o serviço de API e o banco de dados SQL Server.
- **Garantia de Qualidade**: Tipagem forte via C# Records, immutability por padrão e validação centralizada.

## Estrutura do Projeto e Organização de Código

A árvore de diretórios do repositório MUST respeitar a seguinte organização padronizada:

```text
meu-estudo-ia/
├── src/
│   ├── IncidentPilot.Api/
│   │   ├── Endpoints/
│   │   ├── Models/
│   │   ├── Services/
│   │   │   └── Ai/
│   │   ├── Data/
│   │   │   └── Context/
│   │   ├── Program.cs
│   │   └── appsettings.json
├── docker-compose.yml
├── Dockerfile
└── README.md
```

- Modificações na estrutura de diretórios requerem atualização prévia ou simultânea nesta constituição.

## Governance

1. **Supremacia da Constituição**:
   - As diretrizes e princípios estabelecidos neste documento se sobrepõem a qualquer outra convenção de código não formalizada.
   - Mudanças arquiteturais ou introdução de novas dependências requerem atualização desta Constituição.

2. **Política de Versionamento**:
   - O versionamento segue o padrão Semantic Versioning (`MAJOR.MINOR.PATCH`).
   - **MAJOR**: Remoção ou alteração incompatível de princípios fundamentais ou mudanças estruturais de arquitetura.
   - **MINOR**: Adição de novos princípios, novas seções técnicas ou expansão substancial de diretrizes.
   - **PATCH**: Correções ortográficas, refatoração de redação ou pequenos esclarecimentos sem alteração semântica.

3. **Procedimento de Emenda**:
   - Emendas devem ser propostas via Pull Request detalhando a motivação, o relatório de impacto e o plano de transição.
   - Toda alteração deve atualizar o cabeçalho `Sync Impact Report` e os campos de versão e data.

**Version**: 1.0.0 | **Ratified**: 2026-10-28 | **Last Amended**: 2026-10-28

