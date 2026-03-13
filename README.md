# Estratégia de Ambientes: DEV → QA → UAT → Produção

Este repositório contém um _blueprint_ de separação de ambientes e promoção controlada.

## Documento técnico completo

A versão completa da arquitetura enterprise (DevOps + QA Engineering) está em:

- [`docs-arquitetura-enterprise.md`](docs-arquitetura-enterprise.md)

## Estrutura deste repositório

- `docker-compose.dev.yml`: stack local para DEV.
- `k8s/namespaces/*.yaml`: namespaces dedicados por ambiente.
- `k8s/overlays/*`: configuração base por ambiente (DEV/QA/UAT).
- `.github/workflows/deploy-qa.yml`: deploy automático em QA a partir de `develop`.
- `.github/workflows/deploy-uat.yml`: deploy manual/aprovado em UAT.
