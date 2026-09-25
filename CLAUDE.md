# teste-sf-devops4

Salesforce DevOps framework: policy overlay (`config/.sf-devops.yml`) sobre engine
sfdx-hardis (`config/.sfdx-hardis.yml`). Branches maiores: integration → uat → production,
cada uma mapeada a uma org em `config/branches/.sfdx-hardis.<branch>.yml`.
CI: `.github/workflows/check-deploy.yml` (PR, check-only) e `process-deploy.yml` (push, deploy real),
ambos rodando no container `ghcr.io/hardisgroupcom/sfdx-hardis-ubuntu:latest`.

## Gotchas

- **Metadado commitado DIRETO numa branch maior (fora do fluxo feature→PR) nunca entra
  num delta de deploy.** O ASA foi criado direto na `integration` (`c5c81b9`); depois
  `feature/ASA` fez merge da integration nela mesma (`97fc65e`), então o merge da PR #1
  (`affb14b`) não teve delta de árvore vs. a base/head da PR → `sgd --from 97fc65e --to
  affb14b` = package.xml VAZIO → "No deployment or destructive changes to perform".
  (NÃO é falta de reconhecimento de tipo: sgd 6.45.1 gera `AiAuthoringBundle: ASA`
  corretamente em `--from e24b9a2 --to affb14b`.) Regra: só criar/alterar metadado em
  feature branch → PR → merge; nunca commitar direto em integration/uat/production, e
  não fazer merge da target de volta na feature logo antes do merge da PR (vira no-op de
  árvore). Correção pontual quando já caiu no buraco: deploy manual do componente.

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

- **Sem `useDeltaDeployment: true`, o `deploy:smart` no PUSH (process-deploy) montava
  escopo vazio e dava short-circuit ("No deployment or destructive changes") ANTES do Quick
  Deploy reaproveitar a validação da PR** — o artefato validado no `--check` nunca chegava à
  org. Reproduzido: sgd 6.45.1 reconhece `AiAuthoringBundle` e `HEAD^..HEAD` do merge da PR
  #3 (`9be8190..a395946`) gera `ASA` corretamente; o vazio vinha do escopo que o hardis
  escolhia no modo default. Fix: `useDeltaDeployment: true` em `config/.sfdx-hardis.yml` →
  push passa a implantar `sgd --from HEAD^ --to HEAD` (só os artefatos do PR, nunca full).
  ATENÇÃO: config NÃO blinda topologia degenerada — se o branch contém a target inteira
  (caso `fix/ASA-adjust`, criada do tip da integration sem divergência), `HEAD^..HEAD` vira
  no-op e o delta volta a ser vazio (ver gotcha do ASA acima). Garantia = config + branch
  que diverge de verdade via feature→PR→merge.

- **Delta mode faz `deploy final = manifesto-base ∩ git-delta`, e o `deploy:smart` exige o
  manifesto-base como ARQUIVO.** `initPackageXmlAndDestructiveChanges` resolve
  `this.packageXmlFile` na cadeia `--packagexml` → `PACKAGE_XML_TO_DEPLOY` →
  `packageXmlToDeploy` (config) → `manifest/package.xml` → `config/package.xml` (último
  fallback). Full mode tolera ausência (linha 307 do smart.js trata inexistente como
  "vazio" → "No deployment or destructive changes"). Delta NÃO: `handleDeltaDeployment` faz
  `fs.copy(this.packageXmlFile, packageDelta.xml)` ANTES de qualquer checagem → sem
  `manifest/package.xml` nem `config/package.xml`, crash `ENOENT ./config/package.xml`.
  Corolário perigoso: como o deploy é `base ∩ delta` (`removePackageXmlContent(base, gitDelta,
  removedOnly=true, context:'delta')` = "keep matching items"), um `manifest/package.xml`
  ESTÁTICO defasado descarta silenciosamente qualquer tipo que ele não liste — repete o
  silent-drop do ASA. Fix: gerar `manifest/package.xml` a partir do `force-app` NO CI a cada
  run (`sf project generate manifest --source-dir force-app --output-dir manifest --name
  package`), antes do `deploy:smart`, nos dois workflows; nunca commitar o arquivo (fica no
  `.gitignore`). Assim base ⊇ delta sempre → interseção = exatamente os artefatos do PR.
