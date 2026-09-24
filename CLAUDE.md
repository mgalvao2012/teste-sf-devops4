# teste-sf-devops4

Salesforce DevOps framework: policy overlay (`config/.sf-devops.yml`) sobre engine
sfdx-hardis (`config/.sfdx-hardis.yml`). Branches maiores: integration → uat → production,
cada uma mapeada a uma org em `config/branches/.sfdx-hardis.<branch>.yml`.
CI: `.github/workflows/check-deploy.yml` (PR, check-only) e `process-deploy.yml` (push, deploy real),
ambos rodando no container `ghcr.io/hardisgroupcom/sfdx-hardis-ubuntu:latest`.

## Gotchas

- **Imagem `megalinter-salesforce:v8.8.0` tem os linters Salesforce quebrados.**
  `SALESFORCE_SFDX_SCANNER_APEX/AURA/LWC` falham com `JitPluginInstallError`
  (npm `ENOTEMPTY` / tarball corrompido ao instalar `@salesforce/sfdx-scanner@4.12.0`
  em runtime); `SALESFORCE_LIGHTNING_FLOW_SCANNER` chama `sf flow:scan`, comando
  inexistente na CLI empacotada. São defeitos de tooling, não findings de código.
  Fix: desativados em `.mega-linter.yml` (`DISABLE_LINTERS`). A análise estática
  Salesforce real já roda no Code Analyzer dentro de `hardis:project:deploy:smart
  --check` (check-deploy.yml). Reavaliar ao subir a versão/flavor do MegaLinter.

- **`check-deploy.yml` estava SEM o passo `git safe.directory` que `process-deploy.yml` tem.**
  O container hardis roda como usuário != dono do checkout → git aborta com
  `fatal: detected dubious ownership` → `sfdx-git-delta` recebe diff vazio →
  "No deployment or destructive changes to perform" (delta vazio, não erro).
  Fix: `git config --global --add safe.directory "$GITHUB_WORKSPACE"` como passo
  após o Checkout, em TODO workflow que roda hardis no container. Manter os dois workflows sincronizados.

- **`AiAuthoringBundle` (aiAuthoringBundles / Agent Script `.agent`) É implantável via
  Metadata API (SOAP) em API 67.0.** Validado por `sf project deploy start --dry-run`:
  1/1 componente, `State: Created`. NÃO precisa de build/transpile para deploy direto.

- **CLI 2.150.6 manda versão de API inválida na URL para este org (67.0).** Sem
  `--api-version 67.0` explícito, `sf project deploy` retorna
  `UNSUPPORTED_API_VERSION: Invalid Api version specified on URL`.

- **`hardis:project:deploy:smart` gera package VAZIO quando o git delta falha ou não há
  base de PR.** Sempre checar no log do passo de deploy se `git log <base>..<head>` rodou sem
  erro antes de concluir que "não havia nada para implantar".

- **Guard de Jest no CI deve gatilhar em LWC real** (`force-app/**/lwc/*/`), não na presença do
  script npm `test:unit` (sempre presente no scaffold). E usar `npm install` (não `npm ci`) enquanto
  não houver `package-lock.json` versionado.

- **`sfdx-git-delta` 6.45.1 não tem typedefs próprios** — delega ao registry do SDR
  (`sdrMetadataAdapter`). Reconhecimento de tipos novos depende da versão do SDR no runner.
