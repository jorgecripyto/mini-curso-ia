# Regras de JavaScript e TypeScript

Estas regras valem para todo o código TypeScript do projeto, no backend e no frontend.

## 1. const em vez de let, nunca var

Declare com `const`. Use `let` só quando o valor realmente precisar ser reatribuído. Nunca use `var`.

**Ruim:**

```ts
var total = 0;
let nome = 'Salário';
```

**Bom:**

```ts
const total = somarLancamentos(lancamentos);
const nome = 'Salário';
```

## 2. Comparação estrita

Compare sempre com `===` e `!==`. Nunca use `==` nem `!=`.

**Ruim:**

```ts
if (lancamento.tipo == 'receita') {}
```

**Bom:**

```ts
if (lancamento.tipo === 'receita') {}
```

## 3. Nunca use any

`any` desliga a checagem de tipos. Use tipos explícitos. Para dados que chegam de fora e ainda não foram validados, use `unknown` e valide antes de usar.

**Ruim:**

```ts
function validar(entrada: any) {
  return entrada.valorCentavos > 0;
}
```

**Bom:**

```ts
function validar(entrada: unknown): boolean {
  if (!ehEntradaLancamento(entrada)) return false;
  return entrada.valorCentavos > 0;
}
```

## 4. Prefira os tipos

Modele o domínio com `type` e `interface`. Use uniões de textos literais no lugar de `string` solto.

**Ruim:**

```ts
function criar(tipo: string) {}
```

**Bom:**

```ts
type TipoLancamento = 'receita' | 'despesa';

function criar(tipo: TipoLancamento): void {}
```

## 5. Arrow function nos callbacks

Use arrow function em callbacks, como em `map`, `filter` e `reduce`. As funções principais do arquivo, aquelas que ele exporta e que dão nome à sua responsabilidade, usam `function`.

**Ruim:**

```ts
export const calcularSaldo = (lancamentos: Lancamento[]): number => {
  return lancamentos.reduce(function (saldo, item) {
    return saldo + item.valorCentavos;
  }, 0);
};
```

**Bom:**

```ts
export function calcularSaldo(lancamentos: Lancamento[]): number {
  return lancamentos.reduce((saldo, item) => saldo + item.valorCentavos, 0);
}
```

## 6. Ternário sem aninhamento

Use ternário apenas para atribuição condicional simples. Nunca coloque um ternário dentro de outro. Se houver mais de duas opções, use cláusulas guarda ou um objeto de mapeamento.

**Ruim:**

```ts
const rotulo = tipo === 'receita' ? 'Receita' : tipo === 'despesa' ? 'Despesa' : 'Outro';
```

**Bom:**

```ts
const ROTULOS_TIPO: Record<TipoLancamento, string> = {
  receita: 'Receita',
  despesa: 'Despesa',
};

const rotulo = ROTULOS_TIPO[tipo];
```

Ternário simples permitido:

```ts
const sinal = tipo === 'despesa' ? -1 : 1;
```

## 7. Tipar parâmetros e retornos que envolvem objetos

Toda função que recebe ou devolve um objeto declara os tipos de forma explícita.

**Ruim:**

```ts
function resumir(lancamentos) {
  return { receitas: 0, despesas: 0 };
}
```

**Bom:**

```ts
interface Resumo {
  receitasCentavos: number;
  despesasCentavos: number;
  saldoCentavos: number;
}

function resumir(lancamentos: Lancamento[]): Resumo {
  const receitasCentavos = somarPorTipo(lancamentos, 'receita');
  const despesasCentavos = somarPorTipo(lancamentos, 'despesa');
  return { receitasCentavos, despesasCentavos, saldoCentavos: receitasCentavos - despesasCentavos };
}
```

## 8. Rode o linter ao terminar cada tarefa

Ao concluir uma tarefa, rode o linter e corrija tudo o que ele apontar antes de considerar o trabalho pronto.

- Frontend: `npm run lint` dentro de `frontend/`.
- Backend: rode o script de lint quando ele existir. Enquanto não existir, rode `npm run build`, que checa os tipos.

## 9. Outras boas práticas

Use `import type` para importar apenas tipos.

```ts
import type { Lancamento } from '../types/lancamento';
```

Não altere objetos e listas recebidos. Crie cópias com spread.

**Ruim:**

```ts
lancamentos.push(novo);
```

**Bom:**

```ts
const atualizados = [...lancamentos, novo];
```

Use `??` para valor padrão e `?.` para acessar propriedade que pode não existir.

```ts
const porta = Number(process.env.PORT ?? PORTA_PADRAO);
const descricao = lancamento?.descricao ?? DESCRICAO_PADRAO;
```

Use `readonly` em dados que não devem mudar.

```ts
interface Categoria {
  readonly id: string;
  readonly nome: string;
}
```