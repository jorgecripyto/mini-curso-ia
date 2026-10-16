# Regras de Node

Estas regras valem para todo o código em `backend/`.

## 1. async/await em vez de callbacks

Use `async/await` para código assíncrono. Não use callbacks aninhados.

**Ruim:**

```ts
function buscarLancamentos(mes: string, aoTerminar: (erro: Error | null, itens?: Lancamento[]) => void) {
  repositorio.listar(mes, (erro, itens) => {
    if (erro) return aoTerminar(erro);
    aoTerminar(null, itens);
  });
}
```

**Bom:**

```ts
async function buscarLancamentos(mes: string): Promise<Lancamento[]> {
  return repositorio.listar(mes);
}
```

## 2. Nunca bloqueie o event loop sem necessidade

O Node processa as requisições em uma única thread. Cálculos pesados e operações síncronas travam a API inteira.

- Use as versões assíncronas de arquivo (`fs/promises`), nunca `readFileSync` dentro de rotas.
- Para cálculo pesado, mova o trabalho para um `worker_thread` ou divida em partes pequenas.

**Ruim:**

```ts
const conteudo = readFileSync('relatorio.json', 'utf-8');
```

**Bom:**

```ts
import { readFile } from 'node:fs/promises';

const conteudo = await readFile('relatorio.json', 'utf-8');
```

## 3. Variáveis de ambiente em .env com dotenv

Toda configuração que muda entre ambientes (porta, caminho do banco) fica em um arquivo `.env`, lido com a biblioteca `dotenv`.

- Carregue o `dotenv` uma única vez, em `config.ts`, e exporte os valores prontos.
- Mantenha o `.env` no `.gitignore` e versione um `.env.example` com os nomes das variáveis, sem valores sensíveis.

```ts
import dotenv from 'dotenv';

dotenv.config();

const PORTA_PADRAO = 3000;

export const config = {
  porta: Number(process.env.PORT ?? PORTA_PADRAO),
  caminhoBanco: process.env.DATABASE_PATH ?? 'data/financas.sqlite',
};
```

Exemplo de `.env.example`:

```
PORT=3000
DATABASE_PATH=data/financas.sqlite
```

## 4. Graceful shutdown do servidor HTTP

Ao receber o sinal de desligamento, o servidor para de aceitar novas conexões, termina as requisições em andamento e só então encerra o processo.

```ts
const servidor = app.listen(config.porta);

function encerrar(sinal: string): void {
  logger.info(`Sinal ${sinal} recebido, encerrando o servidor`);
  servidor.close(() => process.exit(0));
}

process.on('SIGTERM', () => encerrar('SIGTERM'));
process.on('SIGINT', () => encerrar('SIGINT'));
```

## 5. Logging por um adaptador centralizado

Não chame `console.log` nem `console.error` diretamente no código. Crie um adaptador único em `logger.ts` e use-o em todo o projeto. Assim, trocar a forma de registrar logs exige mudar um arquivo só.

```ts
export const logger = {
  info: (mensagem: string): void => console.log(mensagem),
  erro: (mensagem: string, causa?: unknown): void => console.error(mensagem, causa),
};
```

**Ruim:**

```ts
console.log('Lançamento salvo');
```

**Bom:**

```ts
logger.info('Lançamento salvo');
```

## 6. Versione os lock files

O `package-lock.json` vai para o repositório. Isso garante que todos instalem as mesmas versões. Instale as dependências com `npm ci`, que respeita o lock file.

## 7. Evite referências circulares

Dois módulos nunca devem importar um ao outro, direta ou indiretamente.

**Ruim:**

```ts
// resumo-mensal.ts importa de lancamentos-service.ts
// lancamentos-service.ts importa de resumo-mensal.ts
```

**Bom:** extraia o que é compartilhado para um terceiro módulo (por exemplo, um tipo em `types/`) que os dois importam, sem que um dependa do outro.

```ts
import type { Lancamento } from '../types/lancamento';
```