# Torre de Controle · Atlas

Painel da Torre de Controle com duas partes:

- **Torre de controle** (`/`): quadro da esteira de implantação e do discovery, com lista de projetos e linha do tempo de go-live. Os projetos são cadastrados manualmente ou por planilha e ficam guardados numa tabela do Azure Storage.
- **Gestão Azure** (`/devops.html`): sprints, burndown, velocidade, bugs e work items lidos ao vivo do Azure DevOps.

O acesso exige login com a conta Microsoft da empresa. O token do Azure DevOps fica só nas configurações do Azure e nunca chega ao navegador.

```
GitHub (código) ──push──▶ GitHub Actions ──deploy──▶ Azure Static Web Apps
                                                    ├─ frontend/  (páginas)
                                                    └─ api/       (Azure Functions)
                                                         ├─ /api/projetos  ──▶ Azure Table Storage
                                                         └─ /api/devops    ──▶ Azure DevOps REST API
```

## Estrutura

```
frontend/
  index.html                  Quadro da Torre (esteira, lista, linha do tempo)
  devops.html                 Gestão Azure (sprints e work items)
  staticwebapp.config.json    Login, rotas protegidas e runtime da API
  assets/                     CSS, JavaScript e modelo de planilha
api/
  src/functions/projetos.js   GET/POST /api/projetos, PUT/DELETE /api/projetos/{id}, POST /api/importar-projetos
  src/functions/devops.js     GET /api/devops[?atualizar=1]
  src/lib/                    Armazenamento, Azure DevOps e autenticação
.github/workflows/deploy.yml  Deploy automático a cada push na main
exemplos/                     Configuração alternativa para o plano Free
```

---

## Passo a passo para colocar no ar

### 1. Subir o código no GitHub

Crie um repositório **privado** no GitHub (por exemplo `torre-controle`) e, na pasta deste projeto:

```bash
git init
git add .
git commit -m "Torre de Controle: primeira versão"
git branch -M main
git remote add origin https://github.com/SUA-ORG/torre-controle.git
git push -u origin main
```

O primeiro deploy vai falhar até o passo 3 ser concluído. Isso é esperado.

### 2. Criar a tabela de projetos (Azure Storage)

1. No portal do Azure, crie uma **Storage Account** (padrão *Standard / LRS* é suficiente).
2. Em **Chaves de acesso**, copie a **Cadeia de conexão**. Ela será o valor de `TABELA_CONEXAO`.

A tabela `Projetos` é criada sozinha na primeira gravação.

### 3. Criar o Static Web App

1. No portal, crie um **Static Web App**.
2. Plano: **Standard**. Ele é necessário para restringir o login ao tenant da nstech (veja a seção "Plano Free" se quiser começar sem custo).
3. Em **Origem da implantação**, escolha **Outro** (o deploy já está pronto em `.github/workflows/deploy.yml`).
4. Depois de criado, abra **Gerenciar token de implantação** e copie o token.
5. No GitHub, em **Settings > Secrets and variables > Actions**, crie o segredo `AZURE_STATIC_WEB_APPS_API_TOKEN` com esse token.
6. Rode de novo o workflow em **Actions** (ou faça um novo push).

### 4. Configurar o login com a conta da empresa (Entra ID)

1. No **Microsoft Entra ID > Registros de aplicativo > Novo registro**:
   - Nome: `Torre de Controle`
   - Tipos de conta: **somente este diretório organizacional**
   - URI de redirecionamento (Web): `https://SEU-SITE.azurestaticapps.net/.auth/login/aad/callback`
2. Em **Certificados e segredos**, crie um segredo do cliente e copie o valor.
3. Em **Autenticação**, marque **Tokens de ID**.
4. Copie o **ID do aplicativo (cliente)** e o **ID do diretório (tenant)**.
5. Em `frontend/staticwebapp.config.json`, troque `SEU_TENANT_ID` pelo ID do tenant e faça push.

### 5. Criar o token do Azure DevOps

1. No Azure DevOps, **User settings > Personal access tokens > New Token**.
2. Escopo: **Work Items: Read** (nada além disso).
3. Validade curta (por exemplo 90 dias), com lembrete de renovação.
4. De preferência, crie o token numa conta de serviço, não numa conta pessoal.

### 6. Variáveis de ambiente do Static Web App

No Static Web App, **Configurações > Variáveis de ambiente**, crie:

| Variável | Obrigatória | O que é |
|---|---|---|
| `AZURE_CLIENT_ID` | sim | ID do aplicativo (passo 4) |
| `AZURE_CLIENT_SECRET` | sim | Segredo do cliente (passo 4) |
| `TABELA_CONEXAO` | sim | Cadeia de conexão da Storage Account (passo 2) |
| `TABELA_NOME` | não | Nome da tabela. Padrão: `Projetos` |
| `DEVOPS_ORG` | sim | Organização: o `xxx` de `dev.azure.com/xxx` |
| `DEVOPS_PROJECT` | sim | Nome do projeto no Azure DevOps |
| `DEVOPS_TEAM` | não | Time cujos sprints serão mostrados. Padrão: `<Projeto> Team` |
| `DEVOPS_PAT` | sim | Token do passo 5 |
| `DEVOPS_DIAS` | não | Quantos dias de histórico buscar. Padrão: `120` |
| `DEVOPS_WIQL` | não | Consulta WIQL própria, se quiser outro filtro de itens |
| `AREA_MAP` | não | Traduz Area Paths para as áreas da Torre. Ex.: `{"Squad Rastreamento":"Tecnologia","N1":"Suporte"}` |

Pronto: abra o endereço do site, faça login e comece a cadastrar os projetos.

---

## Alimentando os dados

**Projetos da esteira**: use **Novo projeto** ou **Importar planilha**. O modelo está em **Baixar modelo** (quando o quadro está vazio) ou em `frontend/assets/modelo-projetos.csv`. Colunas:

`Cliente; Responsável; Tipo; Status; Fluxo; Etapa; Início; Go-live; Progresso; Observações; ID`

- Tipo: `Essential` ou `Migração`. Status: `Em andamento`, `Congelado` ou `Atrasado`. Fluxo: `Esteira` ou `Discovery`.
- Etapa pelo nome que aparece no quadro (ex.: `Logística`, `Pré Go-Live`).
- Datas em `dd/mm/aaaa`.
- Deixe `ID` vazio para criar. Para editar em massa: **Exportar planilha**, altere no Excel e importe de volta. Linhas com ID existente são atualizadas.
- Se alguma linha tiver erro, nada é gravado e o painel mostra quais linhas corrigir.

**Gestão Azure**: lê o Azure DevOps sozinho, com cache de 5 minutos (o botão **Atualizar agora** força a leitura). Itens com a tag `Bloqueado`, `Blocked` ou `Impedimento` contam como bloqueados. Se a integração ainda não estiver configurada, dá para importar um CSV exportado de uma consulta do Boards.

---

## Rodar no seu computador

Requisitos: Node.js 20, [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local), SWA CLI e Azurite.

```bash
npm install -g @azure/static-web-apps-cli azurite
cp api/local.settings.example.json api/local.settings.json   # preencha DEVOPS_* se quiser testar a integração
(cd api && npm install)
azurite --silent --location .azurite &                      # emula a tabela localmente
swa start                                                   # abre em http://localhost:4280
```

O SWA CLI mostra uma tela de login simulada: digite qualquer e-mail. O arquivo `api/local.settings.json` está no `.gitignore` e nunca deve ir para o GitHub.

---

## Plano Free (sem custo, com limite de usuários)

O plano Free não permite restringir o login ao tenant da empresa. A alternativa é liberar acesso por convite:

1. Substitua `frontend/staticwebapp.config.json` pelo arquivo `exemplos/staticwebapp.config.plano-free.json`.
2. Crie a variável `PAPEL_EXIGIDO` = `torre` (a API passa a exigir esse papel).
3. No Static Web App, **Gerenciamento de funções > Convidar**: provedor Microsoft Entra ID, e-mail da pessoa, função `torre`.

Os convites têm limite de usuários e precisam ser renovados. Para o time todo, o plano Standard é mais prático.

---

## Próximos passos sugeridos

- **Esteira vinda do Azure Boards**: se cada implantação virar um work item (Epic/Feature) com Cliente, Tipo, Etapa e datas, a rota `/api/projetos` pode ler de lá em vez da tabela, e arrastar um card atualizaria o work item.
- **Histórico de etapas**: gravar cada mudança de etapa permite medir o tempo médio por etapa e por responsável.
- **Resumo semanal**: um endpoint com os números da semana pode alimentar o relatório de sexta-feira.

## Observações técnicas

- Gravações seguem "a última vence": se duas pessoas editarem o mesmo projeto ao mesmo tempo, fica a última alteração. O quadro recarrega a cada minuto e ao voltar para a aba.
- Cada projeto guarda quem fez a última alteração e quando (aparece no rodapé da edição).
- A API só pede ao Azure DevOps os campos que existem na organização, então funciona com processos Agile, Scrum ou CMMI (pontos vêm de Story Points, Effort ou Size).
