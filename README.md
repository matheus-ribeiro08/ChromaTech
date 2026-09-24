# ChromaFormula — Motor de Formulação Química e Dosagem de Tintas

[![Java](https://img.shields.io/badge/Java-21-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-4.1.1-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-Time_Series-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com/)

Microsserviço backend corporativo desenvolvido para a **Chromatech Industrial Coatings** com o objetivo de gerenciar a formulação de tintas de alta precisão, o dimensionamento estequiométrico de matérias-primas e a recepção de telemetria em tempo real de dispensadores robóticos para os setores automotivo, aeronáutico e industrial.
---

## Visão Geral Técnica

O ChromaFormula alimenta o motor de formulação responsável pelo escalonamento de receitas de tintas em sistemas de cores normatizados (RAL / Pantone). A plataforma garante a consistência do espectro de cor (ΔE < 0.5), evita a degradação de lotes por erros de arredondamento de densidade ou volume e orquestra a telemetria em tempo real proveniente dos dispensadores robóticos da fábrica.

```
                  +-------------------------------------------------+
                  |          Cliente / Painel do Laboratório        |
                  +------------------------+------------------------+
                                           |
                                  [ REST / WebSockets ]
                                           |
                                           v
                  +-------------------------------------------------+
                  |          ChromaFormula API (Spring Boot)        |
                  +-------+----------------+----------------+-------+
                          |                |                |
                          v                v                v
                  +---------------+ +--------------+ +---------------+
                  |  PostgreSQL   | |   MongoDB    | |  API Externa  |
                  | (Relacional)  | | (Time-Series)| | (Commodities) |
                  +---------------+ +--------------+ +---------------+
```

---

## Especificações e Regras de Engenharia

### 1. Dimensionamento Estequiométrico e Cálculo de Dosagem

Ao receber o código da cor (ex: RAL 3020 Vermelho Tráfego) e o volume total solicitado (V<sub>solicitado</sub> em Litros), o motor calcula a massa exata (M) em Quilogramas e o volume em Mililitros para cada insumo (pigmentos, resinas, solventes e aditivos):

$$M_{\text{insumo}} = V_{\text{solicitado}} \times \text{Proporção}_{\text{insumo}} \times \text{Densidade}_{\text{insumo}}$$

- **Tolerância Rígida de Mistura:** as formulações são validadas contra uma regra obrigatória de balanço composicional de 100% (± 0.01%).
- **Reserva ACID:** ordens aprovadas disparam um fluxo atômico `@Transactional`:
  1. Criação da ordem de produção (`STATUS_RESERVADA`).
  2. Abate exato das quantidades no estoque de matérias-primas do laboratório.
  3. Emissão da Ficha de Segurança Química (FISPQ).
- **Rollback Automático:** qualquer escassez de estoque ou divergência na fração composicional aborta instantaneamente a transação para preservar a integridade do inventário.

### 2. Persistência Híbrida (SQL + NoSQL)

- **PostgreSQL:** gerencia os modelos de domínio transacionais (Matérias-Primas, Padrões de Cor, Matrizes Base e Ordens de Produção). Todos os endpoints de listagem oferecem suporte a paginação e ordenação dinâmica via `Pageable`/`Sort`.
- **MongoDB (Séries Temporais):** armazena a telemetria IoT de alta frequência emitida a cada 1 segundo pelos dispensadores robóticos (massa na balança, pressão do bico, temperatura da mistura e ΔE instantâneo). Índices TTL removem automaticamente os dados brutos após 120 dias.

### 3. Telemetria em Tempo Real e Integrações Assíncronas

**WebSockets (STOMP):**
- `/topic/producao-tintas-global`: progresso do lote em tempo real e status operacional para toda a fábrica.
- `/user/queue/telemetria-dispensador`: canal dedicado de streaming para engenheiros químicos acompanharem a dosagem e o desvio de ΔE ao vivo.

**Consumo Resiliente de Preços:** consome APIs externas do mercado químico para obter cotações de resinas/solventes em tempo real, utilizando fallback para taxas em cache caso esteja offline.

**Webhooks Assinados:** dispara notificações assíncronas HTTP POST assinadas com cabeçalhos HMAC-SHA256 (`DOSING_BATCH_COMPLETED`) para o ERP corporativo após a aprovação de cor (ΔE < 0.5).

### 4. Segurança e Hardening (Conformidade OWASP)

- **Autenticação Stateless:** Spring Security com tokens JWT.
- **Controle de Acesso Baseado em Perfis (RBAC):**
  - `ROLE_ADMIN`: acesso total a matrizes, parâmetros de custo e gestão de padrões de cores.
  - `ROLE_ENGENHEIRO_QUIMICO`: autorização de receitas, ajuste de frações de aditivos e configuração de Webhooks.
  - `ROLE_OPERADOR_DISPENSADOR`: solicitação de dosagens, consulta de receitas e consumo dos canais WebSocket.
- **Controles Defensivos:** limitação de taxa de requisições (Rate Limiting em 150 req/min), sanitização contra SQLi/XSS, proteção CSWSH em WebSockets e tratamento centralizado de exceções via RFC 7807 (`ProblemDetail`).

---

## Stack Tecnológica

| Categoria | Tecnologias |
|---|---|
| Linguagem | Java 21 |
| Framework | Spring Boot 3.x |
| Acesso a Dados | Spring Data JPA (Hibernate), Spring Data MongoDB |
| Segurança | Spring Security, JJWT |
| Tempo Real | Spring WebSocket (STOMP, SockJS) |
| Validação e Produtividade | Hibernate Validator, Lombok, Spring Boot DevTools |
| Testes e Qualidade | JUnit 5, Mockito, Testcontainers, JaCoCo |
| Observabilidade e Documentação | Spring Boot Actuator, OpenAPI 3 (Swagger UI) |

---

## Estrutura do Projeto

```
br.chroma.chromatech
├── config          # Configurações do Spring Security, WebSocket e Bancos de Dados
├── controller      # Controllers REST com Tratamento de Erros RFC 7807
├── dto             # Imutáveis / Java Records para Entrada e Saída da API
├── model           # Entidades JPA e Documentos de Séries Temporais do MongoDB
├── repository      # Repositórios Spring Data JPA e MongoDB
├── service         # Regras de Negócio, Engine Estequiométrico em TDD e Webhooks
└── security        # Filtros JWT, UserDetailsService e Políticas RBAC
```

---

## Documentação da API e Testes

- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **Métricas de Saúde:** `http://localhost:8080/actuator/health`

### Suíte de Testes e Cobertura

O projeto adota o Desenvolvimento Guiado por Testes (TDD) para a implementação das fórmulas matemáticas e transações ACID de reserva.

- **Execução dos Testes:** suíte automatizada com JUnit 5, Mockito e Testcontainers.
---

## Diretrizes de Commit

Este repositório adota estritamente o padrão [Conventional Commits](https://www.conventionalcommits.org/):

| Prefixo | Descrição |
|---|---|
| `feat:` | Novas funcionalidades ou inclusão de endpoints |
| `fix:` | Correção de bugs em regras de negócio ou segurança |
| `test:` | Adição ou ajuste de cenários de teste (Ciclo TDD) |
| `docs:` | Atualizações na documentação ou arquivos de instrução |
| `refactor:` | Melhorias no código sem alterar o comportamento externo |
