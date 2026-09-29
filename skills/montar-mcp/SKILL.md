---
name: montar-mcp
description: >-
  Constrói servidores MCP e registra no Claude (Claude Code e Claude Desktop)
  para qualquer local: pasta, vault, workspace, URL ou API. O alvo nunca fica
  fixo no código: entra por argumento, MCP_LOCAL, MCP_BASE_URL ou variável de
  ambiente do registro. Use sempre que Gustavo pedir criar, montar, ligar,
  configurar, instalar, registrar ou apontar um MCP para qualquer pasta,
  caminho, vault, projeto, máquina ou serviço, mesmo sem citar a palavra MCP
  (ex.: "dá acesso do Claude a essa pasta", "conecta o Claude nessa API").
---
# Montar MCP em qualquer local (Claude)

O **local** é o alvo (pasta, vault, URL ou API). O servidor é o programa. Um servidor serve vários locais; cada local é uma entrada de registro.

Antes de escrever código, classifique o pedido. Se já existe pacote ou URL MCP, só registre. Código novo só quando não houver servidor.

Para servidor grande de API de terceiro (muitas tools, avaliação), leia também a skill `mcp-builder`, se existir. Esta skill manda no local, no registro e no confinamento.

Contexto corporativo: a Capricórnio pode restringir conectores e servidores por política do admin. Se o registro falhar ou a entrada não aparecer, verificar política antes de depurar código.

## 1. Perguntar o cliente (obrigatório)

O registro muda por cliente. Se não for inferível, perguntar em uma única pergunta: **Claude Code, Claude Desktop ou claude.ai (conector remoto)?**

| Cliente | Onde registra | Variáveis no registro | Reinício |
|---|---|---|---|
| Claude Code | CLI `claude mcp add` ou `.mcp.json` (projeto) | `${VAR}` e `${VAR:-padrão}` expandem em `command`, `args`, `env`, `url`, `headers` | Nova sessão. Ver com `/mcp` |
| Claude Desktop | `claude_desktop_config.json` | **Não expande.** Caminho absoluto literal | Fechar e reabrir o app por completo |
| claude.ai (web/app) | Configurações > Conectores > Adicionar conector personalizado | Não se aplica | Não |

Caminho do config do Desktop:
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

Regra: servidor **stdio** (processo local) roda em Claude Code e Desktop. claude.ai só aceita servidor **remoto** (URL pública HTTPS). Se Gustavo quer pasta local no claude.ai, avisar que não é possível sem expor o servidor por URL, e recomendar Desktop ou Code.

## 2. Classificar o que existe

| O que existe | Fazer |
|---|---|
| URL de MCP pronta (`https://…/mcp`) | Só registro remoto. Não criar processo. |
| Pacote npm/PyPI (`npx`, `uvx`) | Registro stdio com o pacote. O local vai em `args` ou `env`. |
| Pasta, vault ou diretório, sem servidor | Criar servidor de pasta. Raiz = `MCP_LOCAL`. |
| API REST sem MCP | Criar servidor de API. Base = `MCP_BASE_URL`. Token por variável de ambiente. |

Antes de criar, checar o que já está conectado na sessão (ex.: `vault-capricornio`, `vault-pmo-projetos`, `monday-mcp`, `github`). Se um servidor de pasta existente já cobre o local, não criar outro: apontar nova entrada para o mesmo binário.

Perguntar só o que não der para inferir: qual local, se vale para todos os projetos ou só um, e se pode escrever ou só ler.

## 3. Onde o servidor mora

| Alcance | Código | Registro |
|---|---|---|
| Todos os projetos (Claude Code) | `~/.claude/mcp-servers/<nome>/` | `claude mcp add --scope user` |
| Só este repositório (Claude Code) | `tools/<nome>-mcp/` | `.mcp.json` na raiz (`--scope project`) |
| Só este projeto, privado (Claude Code) | `tools/<nome>-mcp/` | `--scope local` (padrão, não vai pro git) |
| Claude Desktop | `%USERPROFILE%/.claude/mcp-servers/<nome>/` | `claude_desktop_config.json` |

Precedência no Claude Code: local > project > user quando o nome é o mesmo.

Não gravar o caminho do alvo dentro do fonte. O fonte lê ambiente:
- Pasta: `MCP_LOCAL` (absoluto). Recusar vazio.
- API: `MCP_BASE_URL`. Recusar vazio.
- Segredo: nunca literal em arquivo versionado. Claude Code: `${NOME}` no `.mcp.json`. Desktop: o config é local e não versionado, mas ainda assim preferir token com escopo mínimo e nunca colar em chat ou log.

Se o local é a pasta do projeto no Claude Code, `MCP_LOCAL` pode usar `${PWD}` só se o cliente iniciar na raiz; na dúvida, caminho absoluto. No Desktop, sempre absoluto.

## 4. Registrar sem apagar o resto

**Nunca sobrescrever.** Ler o arquivo existente, parsear, acrescentar a chave, gravar. Preservar todas as outras entradas. Fazer backup (`.bak`) antes de editar `claude_desktop_config.json`.

### Claude Code (CLI, preferido)

```bash
# stdio, servidor de pasta local
claude mcp add --scope user --transport stdio nome-do-local \
  --env MCP_LOCAL="C:/caminho/do/alvo" \
  -- node "C:/Users/USUARIO/.claude/mcp-servers/nome/dist/index.js"

# pacote que recebe a pasta como argumento
claude mcp add --scope user --transport stdio nome-do-local \
  -- npx -y pacote-mcp "C:/caminho/do/alvo"

# remoto HTTP
claude mcp add --scope user --transport http nome https://exemplo.com/mcp \
  --header "Authorization: Bearer $TOKEN"
```

Opções (`--env`, `--scope`, `--transport`) vêm **antes** do nome; `--` separa o nome do comando. Windows nativo: `npx` precisa de wrapper, usar `-- cmd /c npx -y pacote-mcp "C:/caminho"`.

### Claude Code (`.mcp.json` de projeto)

```json
{
  "mcpServers": {
    "nome-do-local": {
      "type": "stdio",
      "command": "node",
      "args": ["${HOME}/.claude/mcp-servers/nome/dist/index.js"],
      "env": {
        "MCP_LOCAL": "${MCP_LOCAL:-C:/caminho/do/alvo}",
        "API_TOKEN": "${API_TOKEN}"
      }
    },
    "nome-remoto": {
      "type": "http",
      "url": "https://exemplo.com/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    }
  }
}
```

Variável sem valor e sem padrão faz o parse falhar. Sempre dar `:-padrão` ou garantir que exista. Servidor de `.mcp.json` pede aprovação do usuário na primeira vez.

### Claude Desktop

```json
{
  "mcpServers": {
    "nome-do-local": {
      "command": "node",
      "args": ["C:/Users/USUARIO/.claude/mcp-servers/nome/dist/index.js"],
      "env": { "MCP_LOCAL": "C:/caminho/do/alvo" }
    },
    "nome-pacote": {
      "command": "npx",
      "args": ["-y", "pacote-mcp", "C:/caminho/do/alvo"]
    }
  }
}
```

Caminhos Windows em JSON: barra normal `/` ou `\\` escapada. Nunca `\` simples. Sem `${...}`: nada é expandido. MCP remoto no Desktop não entra por este arquivo; usar Configurações > Conectores.

Nome da entrada: curto, estável, diz o local (`vault-capricornio`, não `mcp` nem `server`).

## 5. Servidor novo

Padrão: TypeScript, transporte stdio, SDK MCP atual (`registerTool`, Zod). Python/FastMCP só se Gustavo pedir ou o projeto já for Python.

Contrato de pasta, toda tool de arquivo:
1. `path.resolve` da raiz `MCP_LOCAL`.
2. `path.resolve` da raiz + caminho pedido.
3. `path.relative(raiz, alvo)` não pode começar com `..` nem ser absoluto.
4. Resolver symlink (`fs.realpath`) antes de validar, para não escapar da raiz.
5. Erro acionável: dizer a raiz configurada e o caminho recusado.

Tools mínimas de pasta, só leitura, até pedirem escrita: `listar`, `ler`, `buscar`. Escrita (`criar`, `editar`, `mover`, `apagar`) só com pedido explícito. Anotar `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`.

Tools de API: uma por operação que ele nomeou, prefixo estável (`servico_buscar`, `servico_criar`). Paginar. Erro da API vira texto com status e próximo passo, sem despejar o token.

**Stdio: nunca escrever em stdout** (`console.log`). Corrompe o protocolo. Log só em stderr (`console.error`).

Esqueleto, confinamento e `package.json`: [reference.md](reference.md).

## 6. Verificar

Checklist:
- [ ] JSON parseia e as entradas antigas continuam.
- [ ] A pasta em `MCP_LOCAL` existe, ou a URL responde.
- [ ] `node dist/index.js` não morre no import. Ficar esperando stdin é sucesso. Crash imediato não é.
- [ ] Tool de leitura no alvo funciona. Caminho `../` fora da raiz falha.
- [ ] Segredo não aparece em arquivo commitável. `.env` fora do git.

Diagnóstico por cliente:
- Claude Code: `claude mcp list`, `claude mcp get <nome>`, `/mcp` dentro da sessão. Depurar com `claude --mcp-debug`.
- Claude Desktop: fechar por completo e reabrir. Logs em `%APPDATA%\Claude\logs\mcp*.log` (Windows) ou `~/Library/Logs/Claude/` (macOS).
- Servidor não aparece: primeiro política do admin e aprovação do usuário, depois caminho e permissão de execução.

## Não fazer

- Hardcodar vault, disco ou host no fonte.
- Criar servidor novo quando `npx` ou URL já resolve.
- Apagar `monday-mcp`, `github`, `obsidian`, `vault-*` ou outra chave ao editar config.
- Usar `${VAR}` no `claude_desktop_config.json` (não expande).
- Usar `console.log` em servidor stdio.
- Commitar token. `.mcp.json` de projeto só com `${VAR}` e só se Gustavo pedir.
- Prometer acesso a pasta local via claude.ai web.
