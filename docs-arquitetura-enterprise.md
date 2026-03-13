# Arquitetura Big Tech de Ambientes (DEV → QA → UAT/STAGING → PRODUÇÃO)

## Sumário Executivo

Este documento descreve uma arquitetura de referência **cloud-native e orientada a produto** para organizações em escala Big Tech, combinando:

- **Microserviços em Kubernetes** com isolamento por ambiente.
- **CI/CD + GitOps** para entrega rápida, rastreável e reversível.
- **Service Mesh + Observabilidade + SRE** para confiabilidade operacional.
- **QA Engineering contínuo** com gates técnicos e de negócio.
- **DevSecOps + IaC + FinOps** para segurança e eficiência de custos.

Fluxo de promoção:

**DEV → QA → UAT/STAGING → PRODUÇÃO**

---

## 1) Arquitetura Geral do Sistema

### 1.1 Princípios arquiteturais

1. **Desacoplamento por domínio** (DDD / bounded context).
2. **Deploy independente** por microserviço.
3. **Automação por padrão** (infra, deploy, testes, observabilidade).
4. **Resiliência distribuída** (timeouts, retries, circuit breaking, bulkheads).
5. **Segurança by design** (zero trust, least privilege, defense-in-depth).

### 1.2 Componentes obrigatórios

- **CDN**: acelera conteúdo estático e reduz latência global.
- **Load Balancer**: distribui tráfego L4/L7 com health checks.
- **API Gateway**: autenticação, rate limit, quota, roteamento por versão.
- **Kubernetes Cluster**: orquestração, escalabilidade, auto-healing.
- **Service Mesh**: mTLS, telemetria L7, controle de tráfego entre serviços.
- **Microservices**: serviços independentes por capacidade de negócio.
- **Databases**: poliglota (SQL/NoSQL) conforme caso de uso.
- **Cache (Redis)**: aceleração de leitura, session/token cache, locks distribuídos.
- **Messaging (Kafka/RabbitMQ)**: comunicação assíncrona, event-driven.

### 1.3 Diagrama textual da arquitetura (alto nível)

```text
Users / Mobile / Web / Partners
            ↓
       CDN + WAF
            ↓
   Global Load Balancer
            ↓
 API Gateway / Ingress
            ↓
 Service Mesh Data Plane
            ↓
Microservices no Kubernetes
  ├─ Serviços síncronos (REST/gRPC)
  ├─ Serviços assíncronos (consumidores/eventos)
  ├─ Workers/Jobs/CronJobs
  └─ BFFs por canal (web/mobile)
            ↓
Data Layer + Platform Services
  ├─ PostgreSQL / MySQL
  ├─ MongoDB / DynamoDB / Cassandra
  ├─ Redis (cache)
  ├─ Kafka / RabbitMQ (mensageria)
  └─ Object Storage / Data Lake
```

### 1.4 Fluxos funcionais recomendados

- **Fluxo síncrono crítico**: Gateway → Mesh → Serviço A → Serviço B → DB.
- **Fluxo assíncrono de escala**: Serviço publica evento em Kafka → múltiplos consumidores processam.
- **CQRS opcional**: escrita em banco transacional e leitura otimizada via cache/read model.

---

## 2) Estrutura de Ambientes

### 2.1 Visão comparativa

| Ambiente | Objetivo | Estabilidade | Dados | Acesso | Frequência de deploy | Infraestrutura |
|---|---|---|---|---|---|---|
| DEV | Construção e depuração | Baixa | Sintéticos/mocks | Devs e engenharia | Alta (várias vezes/dia) | Local + cluster dev compartilhado |
| QA | Garantia técnica e integração | Média/Alta | Massa controlada | QA + dev + automação | Alta (por pipeline) | Cluster dedicado não-prod |
| UAT/STAGING | Validação de negócio | Alta | Anonimizados e realistas | PO, QA, stakeholders | Média (release candidate) | Espelho de produção |
| PROD | Operação real do negócio | Muito alta | Reais | SRE/Operações restritos | Controlada por política | Multi-AZ, segurança máxima |

### 2.2 DEV (Development)

**Objetivo:** produtividade máxima com feedback rápido.

**Características:**
- Deploy muito frequente.
- Logs verbosos (`DEBUG`) e tracing habilitado.
- Falhas toleradas (ambiente instável por natureza).
- Testes locais + testes rápidos em PR.

**Arquitetura recomendada em DEV:**
- Docker Compose para stack local.
- Containers locais para API, Redis e banco local.
- Kubernetes local com **minikube/kind** para paridade básica.
- Mocks/stubs para terceiros (pagamento, e-mail, antifraude).

**Como eliminar “funciona na minha máquina”:**
- Definir ambiente padrão via container/devcontainer.
- Padronizar comandos `make`/scripts (`make up`, `make test`, `make seed`).
- Versionar `.env.example` e contratos OpenAPI/AsyncAPI.
- Fixar dependências (lockfiles).
- Executar validações mínimas no pre-commit/pre-push.

### 2.3 QA (Quality Assurance)

**Objetivo:** validar qualidade funcional, técnica e integração sistêmica.

**Papel do QA em arquitetura moderna:**
- Qualidade contínua desde PR (não apenas no fim).
- Qualidade por risco (risk-based testing).
- Quality gates automatizados para impedir promoção insegura.

**Tipos de testes no QA:**
- Unit tests.
- Integration tests.
- API tests.
- End-to-end tests.
- Regression tests.
- Performance tests.

**Ferramentas recomendadas:**
- **UI/E2E**: Playwright, Cypress, Selenium.
- **API**: Postman/Newman, RestAssured.
- **Performance**: k6, JMeter.

**SIT (System Integration Testing):**
- Valida integração entre múltiplos sistemas internos/externos.
- Testa contratos, idempotência, retries, timeouts e DLQ.
- Simula indisponibilidade parcial de dependências.

### 2.4 UAT / STAGING

**Objetivo:** homologação funcional e aceite de negócio.

**Características essenciais:**
- Réplica fiel de produção (topologia, policies, limites).
- Dados anonimizados/mask de PII.
- Testes exploratórios e smoke tests antes do go-live.
- Aprovação formal do Product Owner / área de negócio.

### 2.5 PRODUÇÃO

**Objetivo:** máxima disponibilidade, segurança e performance.

**Características:**
- Escala automática (HPA/KEDA + autoscaling de nós).
- SLO/SLI e monitoramento contínuo 24x7.
- Estratégias de deploy seguras (canary/blue-green).
- Segurança avançada com trilha de auditoria completa.

**Stack comum de mercado:**
- Kubernetes (EKS/AKS/GKE).
- AWS/Azure/GCP.
- Prometheus + Grafana.
- ELK Stack ou Loki.
- OpenTelemetry + Jaeger/Tempo.

---

## 3) Kubernetes como Base da Infraestrutura

### 3.1 Organização por namespaces

```text
Cluster Kubernetes
 ├─ namespace: dev
 ├─ namespace: qa
 ├─ namespace: staging
 └─ namespace: production
```

Cada namespace deve conter:
- Deployments/StatefulSets de microserviços.
- Services internos.
- ConfigMaps e Secrets.
- Jobs/CronJobs.
- Policies (NetworkPolicy, ResourceQuota, LimitRange).

### 3.2 Padrões operacionais

- **Requests/Limits** obrigatórios.
- **Liveness/Readiness/Startup probes**.
- **PodDisruptionBudget** para continuidade em manutenção.
- **HPA/VPA** para elasticidade.
- **Ingress + cert-manager** para TLS automatizado.

### 3.3 Helm e Kustomize

- **Helm**: empacotamento e versionamento de charts.
- **Kustomize**: overlays por ambiente (dev/qa/staging/prod).
- Prática recomendada: chart base + values por ambiente + validação policy-as-code.

---

## 4) GitOps para Deploy Automatizado

### 4.1 Conceito

GitOps usa o repositório Git como **fonte única da verdade** do estado desejado da infraestrutura e aplicações.

### 4.2 Ferramentas

- **ArgoCD**
- **FluxCD**

### 4.3 Fluxo GitOps

```text
Developer Commit
      ↓
Git Repository (manifests/Helm/Kustomize)
      ↓
ArgoCD/Flux detecta mudança
      ↓
Reconciliação automática no Kubernetes
      ↓
Status de sync + health + histórico de auditoria
```

### 4.4 Vantagens

- Rastreabilidade completa (quem mudou, quando, o quê).
- Rollback rápido via Git revert.
- Consistência entre ambientes.
- Menor drift operacional.

---

## 5) Pipeline CI/CD Enterprise

### 5.1 Pipeline textual completo

```text
Developer Commit
↓
CI Pipeline
↓
Build
↓
Unit Tests
↓
Static Code Analysis
↓
Security Scan
↓
Build Container
↓
Push Docker Registry
↓
Deploy DEV
↓
Deploy QA
↓
Automated Tests
↓
Deploy STAGING
↓
Business Approval
↓
Deploy Production
```

### 5.2 Gates recomendados por fase

| Fase | Gate mínimo |
|---|---|
| PR | lint + unit + SAST + secret scan |
| Build | SBOM + assinatura de imagem + scan de container |
| QA | integração + API + E2E + regressão |
| Staging | smoke + performance baseline + UAT checklist |
| Prod | aprovação + canary + métricas de saúde + rollback automático |

### 5.3 Ferramentas de referência

- **GitHub Actions**, **GitLab CI**, **Jenkins**.
- **ArgoCD** como motor de entrega GitOps.

---

## 6) Estratégias Modernas de Deploy

| Estratégia | Como funciona | Quando usar | Risco | Custo |
|---|---|---|---|---|
| Rolling | Atualiza pods gradualmente | Releases frequentes padrão | Médio | Baixo |
| Blue-Green | Dois ambientes idênticos com troca de tráfego | Mudança crítica com rollback imediato | Baixo | Alto |
| Canary | Percentual pequeno de usuários recebe nova versão | Alto tráfego e mitigação de risco | Muito baixo | Médio |
| Feature Flags | Ativa/desativa funcionalidade sem redeploy | Experimentação, A/B, kill switch | Baixo | Baixo |

Combinação típica Big Tech: **canary + feature flags + rollback automático por SLO**.

---

## 7) Service Mesh

### 7.1 Por que usar

Em grandes plataformas, a complexidade de comunicação entre serviços cresce rapidamente. O service mesh abstrai funcionalidades transversais sem duplicação em código.

### 7.2 Ferramentas

- **Istio** (ecossistema robusto, recursos avançados)
- **Linkerd** (simplicidade e leveza)

### 7.3 Funcionalidades-chave

- Controle de tráfego L7 (split, mirror, failover).
- Retries/timeouts/circuit breaking padronizados.
- mTLS entre serviços.
- Telemetria nativa para métricas e tracing.

---

## 8) Observabilidade

### 8.1 Três pilares

1. **Metrics**: saúde e capacidade (latência, erro, saturação).
2. **Logs**: eventos detalhados e auditoria.
3. **Tracing**: jornada fim a fim entre múltiplos serviços.

### 8.2 Stack recomendada

- **Prometheus** (métricas)
- **Grafana** (dashboards/alertas)
- **Loki** ou **ELK Stack** (logs)
- **Jaeger** (tracing)
- **OpenTelemetry** (instrumentação padronizada)

### 8.3 Arquitetura textual de monitoramento

```text
Aplicações instrumentadas (OpenTelemetry)
          ↓
OTel Collector (agent/gateway)
    ├─ Metrics → Prometheus → Grafana + Alertmanager
    ├─ Logs    → Loki/ELK    → Dashboards + correlação
    └─ Traces  → Jaeger/Tempo → análise de latência
```

Boas práticas:
- Alertas baseados em SLO, não só em infraestrutura.
- Correlação por `trace_id` em logs.
- Runbooks e pós-mortem sem culpabilização.

---

## 9) Chaos Engineering

### 9.1 Objetivo

Validar resiliência real em condições adversas controladas.

### 9.2 Ferramentas

- **Chaos Monkey**
- **LitmusChaos**
- **Gremlin**

### 9.3 Cenários de caos essenciais

- Queda de nós/pods críticos.
- Aumento artificial de latência de rede.
- Falha de dependência externa.
- Perda de conexão com broker de mensagens.

Prática recomendada: começar em QA/Staging, com hipóteses claras, métricas de sucesso e rollback.

---

## 10) Infrastructure as Code (IaC)

### 10.1 Ferramentas

- **Terraform**
- **Pulumi**
- **CloudFormation**
- **Ansible**

### 10.2 Garantia de consistência entre ambientes

- Módulos reutilizáveis + parâmetros por ambiente.
- Revisão obrigatória de mudança infra (PR + plan).
- Policy as code (OPA/Conftest/Sentinel).
- Detecção de drift e reconciliação automatizada.
- Inventário e versionamento de estado remoto seguro.

---

## 11) Segurança de Ambientes

### 11.1 Controles avançados

- **IAM** com mínimo privilégio.
- **RBAC** por namespace/equipe.
- **Secrets management** centralizado (Vault/Secrets Manager).
- Criptografia em trânsito (TLS/mTLS) e em repouso (KMS).
- Segregação forte de dados entre ambientes.
- **Data masking/tokenização** para dados sensíveis fora de produção.

### 11.2 DevSecOps no pipeline

- SAST, DAST, SCA, container scan, IaC scan.
- Assinatura de artefatos e verificação de proveniência.
- Bloqueio automático para vulnerabilidades críticas sem exceção aprovada.

---

## 12) Estratégia de QA Engineering

### 12.1 Test Strategy

- Matriz de risco por domínio.
- Critérios de entrada/saída por ambiente.
- Cobertura mínima por tipo de teste.

### 12.2 Test Pyramid (aplicação prática)

- Base: unit tests rápidos e massivos.
- Meio: integração/API cobrindo contratos.
- Topo: E2E seletivo para fluxos críticos de negócio.

### 12.3 Continuous Testing

- Testes em PR, build, deploy e pós-deploy.
- Gate de qualidade obrigatório para promoção.
- Monitoramento de flaky tests e tempo de feedback.

### 12.4 Test Data Management

- Dados sintéticos e anonimizados.
- Massa versionada por cenário.
- Estratégia de reset/reseed automática por execução.

---

## 13) Otimização de Custos (FinOps)

### 13.1 Práticas de alto impacto

- Auto scaling com limites e políticas inteligentes.
- Uso de containers para melhor densidade.
- Ambientes efêmeros por branch/PR.
- Instâncias spot/preemptivas para workloads tolerantes.
- Desligamento automático de DEV/QA fora de horário.

### 13.2 Modelo operacional

- Tagging obrigatório por time/produto/ambiente.
- Dashboards de custo por microserviço.
- Rightsizing contínuo (CPU, memória, storage).
- Revisão mensal de custo vs SLO para evitar “economia que quebra confiabilidade”.

---

## Blueprint Final (Big Tech Ready)

```text
[Developer Experience]
  Dev Containers + Local Compose + Kind + Contracts + Fast Feedback

[Delivery Engine]
  CI (build/test/scan/sign) + GitOps (ArgoCD/Flux) + Progressive Delivery

[Runtime Platform]
  Kubernetes + Service Mesh + API Gateway + Event Streaming + Managed Data

[Reliability]
  SLO/SLI + Observability (M/L/T) + Incident Response + Chaos Engineering

[Trust & Governance]
  DevSecOps + IAM/RBAC + Secrets + IaC + Audit Trail + FinOps
```

Essa arquitetura permite **velocidade com controle**, reduzindo lead time sem sacrificar qualidade, segurança e resiliência.
