# Referência: servidor de pasta (TypeScript, stdio, somente leitura)

## package.json

```json
{
  "name": "nome-mcp",
  "version": "1.0.0",
  "type": "module",
  "bin": { "nome-mcp": "dist/index.js" },
  "scripts": { "build": "tsc", "start": "node dist/index.js" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "latest",
    "zod": "^3.23.0"
  },
  "devDependencies": { "typescript": "^5.5.0", "@types/node": "^20.0.0" }
}
```

Fixar a versão do SDK após o primeiro build funcionar. `latest` é só para o bootstrap.

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src"]
}
```

## src/index.ts

```ts
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import fs from "node:fs/promises";
import path from "node:path";

const RAIZ_ENV = process.env.MCP_LOCAL ?? process.argv[2] ?? "";
if (!RAIZ_ENV) {
  console.error("MCP_LOCAL vazio. Defina a raiz absoluta do alvo.");
  process.exit(1);
}
const RAIZ = await fs.realpath(path.resolve(RAIZ_ENV));

async function confinar(pedido: string): Promise<string> {
  const alvo = path.resolve(RAIZ, pedido);
  const real = await fs.realpath(alvo).catch(() => alvo);
  const rel = path.relative(RAIZ, real);
  if (rel.startsWith("..") || path.isAbsolute(rel)) {
    throw new Error(`Caminho recusado: "${pedido}". Raiz configurada: ${RAIZ}`);
  }
  return real;
}

const server = new McpServer({ name: "nome-mcp", version: "1.0.0" });

const somenteLeitura = {
  readOnlyHint: true,
  destructiveHint: false,
  idempotentHint: true,
  openWorldHint: false,
};

server.registerTool(
  "listar",
  {
    description: "Lista arquivos e pastas dentro da raiz configurada.",
    inputSchema: { caminho: z.string().default(".") },
    annotations: somenteLeitura,
  },
  async ({ caminho }) => {
    const dir = await confinar(caminho);
    const itens = await fs.readdir(dir, { withFileTypes: true });
    const texto = itens.map((i) => `${i.isDirectory() ? "[D]" : "[F]"} ${i.name}`).join("\n");
    return { content: [{ type: "text", text: texto || "(vazio)" }] };
  }
);

server.registerTool(
  "ler",
  {
    description: "Lê um arquivo de texto dentro da raiz configurada.",
    inputSchema: { caminho: z.string() },
    annotations: somenteLeitura,
  },
  async ({ caminho }) => {
    const arq = await confinar(caminho);
    const stat = await fs.stat(arq);
    if (stat.size > 1_000_000) {
      throw new Error(`Arquivo com ${stat.size} bytes excede 1 MB. Peça um trecho.`);
    }
    return { content: [{ type: "text", text: await fs.readFile(arq, "utf8") }] };
  }
);

server.registerTool(
  "buscar",
  {
    description: "Busca arquivos por trecho do nome, recursivo, dentro da raiz.",
    inputSchema: { termo: z.string().min(1), limite: z.number().int().max(200).default(50) },
    annotations: somenteLeitura,
  },
  async ({ termo, limite }) => {
    const achados: string[] = [];
    const t = termo.toLowerCase();
    async function varrer(dir: string) {
      if (achados.length >= limite) return;
      for (const e of await fs.readdir(dir, { withFileTypes: true })) {
        if (achados.length >= limite) return;
        if (e.name === "node_modules" || e.name.startsWith(".git")) continue;
        const p = path.join(dir, e.name);
        if (e.name.toLowerCase().includes(t)) achados.push(path.relative(RAIZ, p));
        if (e.isDirectory()) await varrer(p);
      }
    }
    await varrer(RAIZ);
    return { content: [{ type: "text", text: achados.join("\n") || "Nada encontrado." }] };
  }
);

await server.connect(new StdioServerTransport());
console.error(`nome-mcp ativo. Raiz: ${RAIZ}`);
```

## Build e teste rápido

```bash
npm install && npm run build
MCP_LOCAL="C:/caminho/do/alvo" node dist/index.js   # deve ficar esperando stdin
npx @modelcontextprotocol/inspector node dist/index.js   # testar tools e o bloqueio de ../
```

## Escrita (só sob pedido explícito)

Acrescentar `criar`, `editar`, `mover`, `apagar` com `readOnlyHint: false`. `apagar` e sobrescrita: `destructiveHint: true`. Reusar `confinar()` também no destino de `mover`. Recomendado: parâmetro `confirmar: true` obrigatório nas destrutivas.
