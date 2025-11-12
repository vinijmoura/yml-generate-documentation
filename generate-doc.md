# Auto Document Workflows (generate-doc.yml)

Documentação do workflow que cria/atualiza documentação dos arquivos de workflow do GitHub Actions deste repositório.

> Resumo rápido: toda vez que você altera um arquivo `.github/workflows/*.yml` (exceto este próprio `generate-doc.yml`), o workflow roda, identifica quais arquivos mudaram, usa o GitHub Copilot CLI para gerar um markdown amigável sobre cada workflow alterado e faz commit automático em `docs/workflows/*.md`.

---

## 🔔 Disparadores (on)
- `workflow_dispatch`: permite executar manualmente pela aba Actions.
- `push` (com filtro de paths):
  - Inclui: `.github/workflows/*.yml`, `.github/workflows/*.yaml`
  - Exclui: `.github/workflows/generate-doc.yml` (evita loop/auto-documentar a si mesmo)

Observação: a identificação de arquivos alterados (no passo "Get changed YML files") atualmente filtra apenas `*.yml` com `grep`. Se você alterar um `*.yaml`, o gatilho será ativado, mas o arquivo pode não ser documentado. Veja a seção "⚠️ Limitações & Notas" para opções de ajuste.

---

## 🔐 Permissões
- `contents: write` no job, para permitir commit e push dos arquivos gerados em `docs/workflows`.

---

## 🧰 Pré-requisitos
- Secret `COPILOT_CLI` configurado no repositório/ambiente, usado como `GH_TOKEN` para autenticar o GitHub Copilot CLI.
- Ação usa Node.js 22 e instala o pacote `@github/copilot` globalmente.
- O repositório deve permitir que o GitHub Actions faça push (branch protections podem exigir ajustes).

---

## 🧭 Como funciona (visão geral)
1. Faz checkout do código.
2. Prepara Node.js 22.
3. Instala o GitHub Copilot CLI (`@github/copilot`).
4. Descobre quais arquivos `.github/workflows/*.yml` mudaram no push atual, comparando `github.event.before` com `github.sha`.
5. Para cada arquivo alterado, executa o Copilot CLI com um prompt que:
   - Lê o conteúdo do workflow
   - Gera um markdown claro, com emojis/ícones e um diagrama Mermaid
   - Salva em `docs/workflows/<nome-do-workflow>.md`
6. Faz commit e push dos arquivos gerados/atualizados.

---

## 🧱 Job e passos (detalhado)
Job: `autodoc` (runner `ubuntu-latest`)

Passos:
1. Checkout do repositório
   - `uses: actions/checkout@v4`

2. Instalação do Node 22
   - `uses: actions/setup-node@v4`
   - `with: node-version: 22`

3. Instala Copilot CLI
   - `run: npm install -g @github/copilot@latest`

4. Identifica YMLs alterados
   - `id: changed`
   - Executa `git fetch` do commit anterior (`github.event.before`) e faz `git diff --name-only` entre esse commit e o atual (`github.sha`), filtrando por `.github/workflows/*.yml`.
   - Exporta `changed_files` como saída do step.

5. Gera documentação (condicional)
   - Só roda se `steps.changed.outputs.changed_files != ''`.
   - Cria `docs/workflows` (se necessário).
   - Para cada arquivo em `changed_files`:
     - Define `outFile` como `docs/workflows/<basename>.md`
     - Executa `copilot --model gpt-5 -p "..." > $outFile`, passando o conteúdo do workflow no prompt.
   - Usa `GH_TOKEN: ${{ secrets.COPILOT_CLI }}` para autenticação.

6. Commit & Push (condicional)
   - Configura usuário/bot do GitHub Actions
   - `git add docs/workflows`
   - `git commit -m "docs: Refresh workflow documentation"`
   - `git push`

---

## 🔎 Diagrama do fluxo (Mermaid)
```mermaid
flowchart TD
  A[push ou workflow_dispatch] --> B[Job autodoc (ubuntu-latest)]
  B --> C[Checkout]
  C --> D[Setup Node 22]
  D --> E[Instala Copilot CLI]
  E --> F[Identifica .yml alterados]
  F -->|se há arquivos| G[Gera docs com Copilot CLI]
  F -->|se vazio| H[Finaliza]
  G --> I[Commit & Push docs]
  I --> J[docs/workflows/*.md atualizados]
```

---

## ▶️ Como executar
- Manualmente: na aba Actions do GitHub, escolha "Auto Document Workflows" e clique em "Run workflow".
  - Nota: em execuções manuais (`workflow_dispatch`), a lógica de diff usa `github.event.before`; se esse valor não estiver disponível no contexto, o step "Get changed YML files" pode não encontrar alterações. Para testar manualmente, prefira fazer um commit que altere um `*.yml` em `.github/workflows`.
- Automaticamente: faça push de commits que alterem `.github/workflows/*.yml` (nota sobre `*.yaml` abaixo).

---

## ⚙️ Personalizações úteis
- Incluir `*.yaml` na detecção de alterados:
  - Ajuste o grep em "Get changed YML files" para algo como `grep -E ".github/workflows/.*\.(yml|yaml)$"` e trate o `basename` no passo seguinte para ambos os sufixos.
- Evitar falhas em `workflow_dispatch`:
  - Adicionar checagens para quando `github.event.before` não existir e, por exemplo, assumir `git diff HEAD^ HEAD` ou analisar todos os arquivos `*.yml`.
- Pinar a versão do Copilot CLI:
  - Em vez de `@latest`, usar uma versão fixa para reprodutibilidade.
- Modelo do Copilot (`--model gpt-5`):
  - Ajuste conforme disponibilidade/licenciamento no seu org. Você pode trocar por outro modelo suportado.

---

## 🧪 Saída gerada
- Arquivos em `docs/workflows/` com o mesmo nome do workflow (`.yml`) e extensão `.md`.
- Ex.: `docs/workflows/ci-build.md` para `.github/workflows/ci-build.yml`.

---

## 🧯 Solução de problemas
- "Permissão negada ao fazer push":
  - Verifique branch protections e se `contents: write` é suficiente para o cenário. Em alguns casos, use um token com permissões elevadas.
- "Copilot CLI não autentica":
  - Confirme a secret `COPILOT_CLI` e se a org/repo permite o uso do Copilot CLI via `GH_TOKEN`.
- "Arquivo .yaml não foi documentado":
  - Veja a dica de personalização para o grep que detecta alterações.
- "Execução manual não encontra alterações":
  - A fonte `github.event.before` pode não existir; ajuste a lógica de diff.

---

## 📎 Referência do prompt usado
O passo de geração chama algo equivalente a:

```
copilot --model gpt-5 -p "Ignore any other files or workspace context. Use only the following content for analysis: $(cat \"$f\") Please explain in detail this workflow and describe stages, jobs and steps on markdown format. Create a mermaid that represent this workflow. Please do not duplicate the generated markdown content! Please make the documentation very user-friendly and intuitive, using clear Markdown formatting, icons, and emojis to improve readability." > docs/workflows/<arquivo>.md
```

---

## ✅ Boas práticas
- Mantenha os workflows pequenos e com nomes/descrições claras nos steps.
- Limite permissões (principle of least privilege).
- Fixe versões de actions sempre que possível (ex.: `actions/checkout@v4`).
- Evite loops de documentação recursiva (já coberto pela exclusão de `generate-doc.yml`).

---

## 📌 Escopo deste arquivo
Este `generate-doc.md` documenta o funcionamento do workflow definido em `.github/workflows/generate-doc.yml`. A documentação específica de outros workflows será gerada automaticamente em `docs/workflows/` quando eles forem alterados.
