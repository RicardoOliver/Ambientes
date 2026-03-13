# Estratégia de Ambientes: DEV → QA → UAT → Produção

Este repositório contém um _blueprint_ para separar ambientes de **Desenvolvimento (DEV)**, **Qualidade (QA)** e **Aceitação do Usuário (UAT/Homologação)**, com promoção controlada até **Produção**.

## Objetivos

- Garantir isolamento entre estágios de entrega.
- Aumentar previsibilidade do deploy com Infraestrutura como Código (IaC).
- Evitar mudanças manuais em UAT e Produção.
- Fortalecer qualidade com testes automatizados no fluxo DEV → QA → UAT.

## Estrutura deste repositório

- `docker-compose.dev.yml`: stack local para DEV.
- `k8s/namespaces/*.yaml`: namespaces dedicados por ambiente.
- `k8s/overlays/*`: configuração base por ambiente (DEV/QA/UAT).
- `.github/workflows/deploy-qa.yml`: deploy automático em QA a partir de `develop`.
- `.github/workflows/deploy-uat.yml`: deploy manual/aprovado em UAT.

## Fluxo recomendado

1. **DEV**
   - Desenvolvimento e validações locais.
   - Dados fictícios e ferramentas de debug.
2. **QA**
   - Testes automáticos, integração e regressão.
   - Deploy automatizado via CI/CD em branch `develop`.
3. **UAT/Homologação**
   - Validação de negócio por PO/stakeholders.
   - Deploy manual com aprovação.
4. **Produção**
   - Publicação final após aprovação do UAT.

## Tabela comparativa

| Recurso | DEV | QA | UAT (Homologação) |
|---|---|---|---|
| Foco | Escrever código | Bugs funcionais | Aceitação de negócio |
| Estabilidade | Baixa | Média | Alta (espelho de Prod) |
| Dados | Fictícios | Mistos de teste | Anonimizados de produção |
| Deploy | Contínuo/Manual | Automático (CI/CD) | Manual/Aprovado |

## Boas práticas implementadas no blueprint

- **IaC** com artefatos versionados para reduzir drift.
- **Segregação de dados** entre ambientes.
- **Pipelines de promoção** por estágio.
- **Governança de acesso**: DEV restrito ao time técnico; UAT para negócio e stakeholders.
- **Otimização de custos**: possibilidade de desligamento programado de DEV/QA.

## Próximos passos sugeridos

- Parametrizar segredos com OIDC + Secret Manager.
- Adicionar política de aprovação obrigatória para UAT e Produção.
- Incluir testes E2E no pipeline de QA.
