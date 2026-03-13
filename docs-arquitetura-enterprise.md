# Arquitetura Enterprise de Ambientes: DEV → QA → UAT → PRODUÇÃO

## Visão Executiva

Este documento define uma arquitetura corporativa para engenharia de software com foco em **alta previsibilidade de entrega**, **qualidade contínua**, **segurança por padrão** e **resiliência operacional**. O fluxo de promoção segue o modelo:

**DEV → QA → UAT/STAGING → PRODUÇÃO**

A proposta combina:
- arquitetura de **microserviços em containers**;
- orquestração em **Kubernetes**;
- práticas de **DevOps + QA Engineering + SRE**;
- governança com **CI/CD, observabilidade e segurança em múltiplas camadas**.

---

## 1) Arquitetura Geral do Sistema

### 1.1 Microserviços

A solução é composta por serviços desacoplados por domínio (ex.: `billing-service`, `orders-service`, `identity-service`, `notification-service`), com contratos API versionados e comunicação síncrona/assíncrona.

**Princípios arquiteturais:**
- **Bounded contexts** por domínio de negócio.
- Deploy independente por serviço.
- Banco por serviço quando aplicável (evitar acoplamento por banco compartilhado).
- Resiliência com timeout, retry, circuit breaker e idempotência.

### 1.2 Containers

Cada microserviço é empacotado em imagem OCI (Docker) com:
- build reproduzível (multi-stage);
- runtime mínimo (distroless/alpine quando possível);
- scan de vulnerabilidade em pipeline;
- versionamento imutável (tag semântica + SHA de commit).

### 1.3 Cloud Infrastructure

Arquitetura agnóstica entre AWS/Azure/GCP, tipicamente com:
- VPC/VNet segmentada por sub-redes públicas e privadas;
- Load balancer gerenciado;
- Kubernetes gerenciado (EKS/AKS/GKE);
- serviços gerenciados de banco/cache/mensageria;
- criptografia em trânsito (TLS) e em repouso (KMS).

### 1.4 Kubernetes Cluster

Topologia recomendada:
- **1 cluster por ambiente crítico** (QA, UAT, PROD) ou **clusters dedicados para PROD e não-PROD**;
- namespaces por domínio/equipe;
- políticas de rede (NetworkPolicy) e quotas;
- HPA/VPA, PDB, affinities e probes de saúde.

### 1.5 API Gateway

Camada de entrada com responsabilidades de:
- autenticação/autorização (OIDC/JWT);
- rate limiting;
- roteamento por versão (`/v1`, `/v2`);
- transformação de payload e observabilidade de borda.

### 1.6 Service Mesh

Uso de Istio/Linkerd para:
- mTLS entre serviços;
- controle de tráfego (canary, mirror, weighted routing);
- telemetria padronizada;
- políticas de segurança leste-oeste.

### 1.7 Diagrama textual da arquitetura

```text
Usuário / Cliente
   ↓
CDN + WAF
   ↓
Load Balancer Público
   ↓
API Gateway / Ingress Controller
   ↓
Service Mesh (mTLS + observabilidade)
   ↓
Microserviços em Kubernetes
   ├─ Serviços síncronos (REST/gRPC)
   ├─ Serviços assíncronos (eventos/fila/stream)
   └─ Jobs/Workers
   ↓
Camada de Dados
   ├─ Banco relacional (PostgreSQL/MySQL)
   ├─ Banco NoSQL (quando necessário)
   ├─ Cache (Redis)
   └─ Mensageria/Streaming (Kafka/RabbitMQ)
```

---

## 2) Estrutura de Ambientes

## 2.1 DEV (Development)

**Objetivo:** máxima produtividade de desenvolvimento e feedback rápido.

**Características principais:**
- acesso restrito aos desenvolvedores;
- deploy frequente e iterativo;
- dados simulados ou sintéticos;
- ambiente naturalmente instável e sujeito a mudanças rápidas.

**Arquitetura sugerida em DEV:**
- Docker Compose para orquestração local;
- containers locais para API, banco e dependências;
- banco local (ou remoto dedicado de desenvolvimento);
- logs detalhados com nível `DEBUG`;
- debug habilitado e hot-reload.

**Como evitar “works on my machine”:**
- padronizar runtime via container e `.env.example`;
- versionar `docker-compose.dev.yml` e scripts de bootstrap;
- usar pre-commit hooks (lint/test);
- manter dependências fixadas (`lock files`);
- usar devcontainer/Codespaces quando possível;
- disponibilizar seeds e mocks determinísticos.

## 2.2 QA (Quality Assurance)

**Objetivo:** validar qualidade técnica e funcional antes da homologação.

**Características principais:**
- ambiente dedicado e estável para testes;
- validação funcional integrada entre serviços;
- regressão contínua com automação;
- gateway para confiabilidade de release.

**Tipos de testes no QA:**
- unit tests;
- integration tests;
- API tests;
- E2E tests;
- regression tests;
- performance tests.

**Ferramentas recomendadas:**
- UI/E2E: Playwright, Cypress, Selenium;
- API: Postman/Newman, RestAssured;
- performance: k6, JMeter.

**SIT (System Integration Testing):**
É a validação ponta a ponta entre múltiplos sistemas integrados (internos e externos), cobrindo contratos, eventos, transformação de dados, tratamento de falhas e comportamento transacional distribuído.

## 2.3 UAT / STAGING (Homologação)

**Objetivo:** validação de negócio e aceite formal para produção.

**Características principais:**
- ambiente de staging altamente aderente à produção;
- validação de regras de negócio por PO/stakeholders;
- testes exploratórios orientados a cenários reais;
- aprovação final do software (go/no-go).

**Diretrizes:**
- infraestrutura equivalente à PROD (mesmas classes de serviço);
- dados anonimizados e governados;
- controles de acesso de negócio auditáveis;
- congelamento parcial de mudanças durante janela de aceite.

## 2.4 PRODUÇÃO

**Objetivo:** disponibilidade, segurança e desempenho para usuários finais.

**Características principais:**
- alta disponibilidade multi-AZ/região quando necessário;
- auto scaling horizontal e vertical;
- monitoramento 24x7 com SLO/SLI;
- observabilidade completa (logs, métricas, traces).

**Ferramentas recomendadas:**
- orquestração: Kubernetes;
- cloud: AWS / Azure / GCP;
- métricas: Prometheus + Grafana;
- logs: ELK/OpenSearch ou Loki;
- tracing: OpenTelemetry + Jaeger/Tempo;
- APM: Datadog (ou equivalente).

---

## 3) Fluxo Completo de CI/CD

```text
Developer Commit
   ↓
CI Pipeline
   ↓
Build
   ↓
Unit Tests
   ↓
Deploy DEV
   ↓
Deploy QA
   ↓
Automated Tests
   ↓
Deploy UAT
   ↓
Business Approval
   ↓
Deploy Production
```

**Etapas detalhadas:**
1. **Developer Commit**: commit + PR com revisão obrigatória e checks mínimos.
2. **CI Pipeline**: dispara validações automáticas por branch/PR.
3. **Build**: gera artefatos imutáveis (imagem + SBOM).
4. **Unit Tests**: valida comportamento interno e regras de domínio.
5. **Deploy DEV**: valida integração rápida e smoke tests.
6. **Deploy QA**: ambiente estável para testes funcionais/técnicos.
7. **Automated Tests**: execução de regressão, API, E2E e performance baseline.
8. **Deploy UAT**: release candidate para homologação.
9. **Business Approval**: aceite formal de negócio e compliance.
10. **Deploy Production**: estratégia segura de rollout com observação ativa.

---

## 4) Exemplo de Pipeline CI/CD Enterprise (GitHub Actions)

```yaml
name: enterprise-ci-cd

on:
  pull_request:
    branches: [ main, develop ]
  push:
    branches: [ develop, main ]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io/org/app
  IMAGE_NAME: platform-api

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Static analysis (lint + SAST)
        run: |
          npm run lint
          npm audit --audit-level=high

      - name: Unit tests
        run: npm test -- --ci --coverage

      - name: Build
        run: npm run build

      - name: Build Docker image
        run: docker build -t $REGISTRY/$IMAGE_NAME:${{ github.sha }} .

      - name: Container scan
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  deploy-qa:
    if: github.ref == 'refs/heads/develop'
    needs: ci
    runs-on: ubuntu-latest
    environment: qa
    steps:
      - name: Deploy to QA (Helm)
        run: helm upgrade --install app ./helm/app --namespace qa --set image.tag=${{ github.sha }}

      - name: Run API/E2E tests
        run: |
          npm run test:api
          npm run test:e2e

  deploy-uat:
    if: github.ref == 'refs/heads/main'
    needs: ci
    runs-on: ubuntu-latest
    environment: uat
    steps:
      - name: Deploy to UAT
        run: helm upgrade --install app ./helm/app --namespace uat --set image.tag=${{ github.sha }}

      - name: DAST
        run: docker run --rm -t owasp/zap2docker-stable zap-baseline.py -t https://uat.example.com

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: deploy-uat
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - name: Manual business gate
        run: echo "Aprovação no environment protection rule"

      - name: Deploy production (canary)
        run: helm upgrade --install app ./helm/app --namespace prod --set image.tag=${{ github.sha }} --set deployment.strategy=canary

      - name: Health check + automatic rollback
        run: |
          ./scripts/post_deploy_check.sh || \
          helm rollback app -n prod
```

**Cobertura de requisitos do pipeline:**
- build automatizado;
- testes automatizados;
- análise estática;
- segurança SAST/DAST;
- deploy automatizado;
- rollback automático por falha de saúde pós-deploy.

---

## 5) Estratégias Modernas de Deploy

- **Blue/Green**: dois ambientes idênticos; troca de tráfego instantânea. Ideal para releases críticas com rollback rápido.
- **Canary**: libera pequena porcentagem para monitorar impacto real antes da expansão. Ideal para mitigar risco em produção de alto tráfego.
- **Rolling**: atualiza gradualmente pods/instâncias sem downtime total. Bom custo-benefício operacional.
- **Feature Flags**: habilita/desabilita funcionalidades sem novo deploy. Excelente para experimentos, kill switch e rollout por segmento.

---

## 6) Observabilidade e Monitoramento

### Modelo de observabilidade
- **Logs centralizados**: coleta estruturada (JSON), correlação por `trace_id`.
- **Métricas**: infraestrutura, aplicação e negócio (RED/USE + SLI/SLO).
- **Tracing distribuído**: visibilidade de latência entre múltiplos serviços.

### Stack sugerida
- Prometheus: coleta de métricas.
- Grafana: dashboards e alertas.
- Loki: agregação de logs com custo otimizado.
- Jaeger: tracing distribuído.
- OpenTelemetry: padrão único de instrumentação.

**Boas práticas:**
- definir SLO por jornada crítica;
- alertas acionáveis (evitar ruído);
- runbooks de resposta a incidentes;
- pós-incidente com análise de causa raiz.

---

## 7) Segurança de Ambientes

**Controles essenciais:**
- **IAM** por princípio do menor privilégio;
- **RBAC** no Kubernetes por namespace e função;
- **Secrets management** via Vault/Secrets Manager/KMS;
- rotação automática de credenciais;
- criptografia de dados em trânsito e repouso;
- assinaturas e proveniência de artefatos (supply chain security).

**Segregação entre ambientes:**
- contas/projetos separados por ambiente (principalmente PROD);
- bancos e chaves distintas por ambiente;
- política de não reutilização de credenciais;
- dados de produção anonimizados quando usados fora de PROD.

---

## 8) Infrastructure as Code (IaC)

Ferramentas usuais:
- **Terraform**: provisionamento cloud declarativo;
- **Ansible**: configuração e automação operacional;
- **Pulumi**: IaC com linguagens de programação;
- **CloudFormation**: gestão nativa em AWS.

**Como garantir consistência DEV/QA/UAT/PROD:**
- módulos reutilizáveis por ambiente;
- variações apenas por parâmetros (`tfvars`, overlays, values);
- policy as code (OPA/Conftest);
- plano/aprovação obrigatória para mudanças;
- detecção de drift e reconciliação automatizada;
- GitOps para estado desejado em Kubernetes.

---

## 9) Estratégia de QA Engineering

- **Test Strategy**: define escopo, critérios de entrada/saída e cobertura por risco.
- **Test Pyramid**: maior base em unit tests, camada média de integration/API, topo com E2E seletivo.
- **Continuous Testing**: testes executados continuamente no pipeline, não apenas no fim.
- **Test Data Management**: dados mascarados, sintéticos e versionados para cenários reproduzíveis.

**Integração QA + DevOps:**
- quality gates por estágio;
- automação desde PR;
- telemetria de testes (flaky rate, tempo médio, cobertura crítica);
- feedback rápido para equipes de produto e engenharia.

---

## 10) Tabela Comparativa de Ambientes

| Critério | DEV | QA | UAT / STAGING | PRODUÇÃO |
|---|---|---|---|---|
| Estabilidade | Baixa (mudança constante) | Média/Alta | Alta | Muito alta |
| Dados utilizados | Sintéticos/fake | Massa de teste controlada | Anonimizados e próximos ao real | Reais |
| Deploy | Muito frequente, flexível | Automatizado por pipeline | Controlado por aprovação | Controlado + janelas/políticas |
| Objetivo | Construção e depuração | Garantia de qualidade técnica | Validação de negócio e aceite | Operação de negócio |
| Acesso | Desenvolvedores/engenharia | Engenharia + QA | QA + PO + stakeholders autorizados | Operações/SRE com controle estrito |
| Infraestrutura | Local/compartilhada | Cluster dedicado não-PROD | Espelho de produção | Alta disponibilidade, resiliente |

---

## 11) Estratégias de Redução de Custos

- **Auto scaling** com limites e políticas por métrica real;
- **Containers** para maior densidade computacional;
- **ambientes efêmeros** por PR para validação temporária;
- uso de **instâncias spot/preemptivas** em cargas tolerantes;
- desligamento automático fora de horário comercial em DEV/QA;
- rightsizing periódico (CPU/memória/storage);
- observabilidade de custo por serviço (FinOps).

---

## Recomendações Finais de Governança

1. Definir arquitetura de referência (ADR) por domínio.
2. Adotar modelo de promoção imutável de artefatos.
3. Estabelecer SLOs por produto e error budget por time.
4. Implantar DevSecOps com segurança contínua em pipeline.
5. Criar trilha de auditoria ponta a ponta (commit → deploy → incidente).

Com esse modelo, a organização alcança maior previsibilidade de releases, redução de risco operacional e evolução contínua de qualidade técnica e de negócio.
