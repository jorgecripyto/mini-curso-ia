# Padrões de código

Estas regras valem para todo o código do projeto, no backend e no frontend.

## 1. Sem comentários

Não escreva comentários. Deixe o nome das variáveis e funções explicar o código. A única exceção é uma expressão regular complexa, que pode ter um comentário curto.

**Ruim:**

```ts
// soma os valores dos lançamentos
const t = lancamentos.reduce((a, l) => a + l.valor, 0);
```

**Bom:**

```ts
const total = lancamentos.reduce((soma, lancamento) => soma + lancamento.valor, 0);
```

**Exceção permitida:**

```ts
// aceita datas no formato AAAA-MM-DD
const FORMATO_DATA = /^\d{4}-\d{2}-\d{2}$/;
```

## 2. Arquivos e classes com no máximo 100 linhas

Se um arquivo passar de 100 linhas, divida por responsabilidade.

**Ruim:** um único `lancamento-service.ts` que valida, calcula o saldo e acessa o banco.

**Bom:**

```
services/validar-lancamento.ts
services/resumo-mensal.ts
data/lancamentos-repository.ts
```

## 3. Funções e métodos com no máximo 30 linhas

Se o comportamento for maior, divida em funções menores, cada uma com um nome claro.

**Ruim:** `registrarLancamento` valida, converte o valor, grava no banco e monta a resposta.

**Bom:**

```ts
function registrarLancamento(entrada: EntradaLancamento): Lancamento {
  validarLancamento(entrada);
  const lancamento = montarLancamento(entrada);
  return salvarLancamento(lancamento);
}
```

## 4. No máximo 3 níveis de if/else, com cláusulas guarda

Prefira retornar cedo (early return) a aninhar condições.

**Ruim:**

```ts
function calcularSaldo(lancamentos: Lancamento[]): number {
  if (lancamentos) {
    if (lancamentos.length > 0) {
      return lancamentos.reduce((saldo, item) => saldo + item.valor, 0);
    } else {
      return 0;
    }
  } else {
    return 0;
  }
}
```

**Bom:**

```ts
function calcularSaldo(lancamentos: Lancamento[]): number {
  if (lancamentos.length === 0) return 0;
  return lancamentos.reduce((saldo, item) => saldo + item.valor, 0);
}
```

## 5. No máximo 3 parâmetros

Se precisar de mais, crie um objeto parâmetro.

**Ruim:**

```ts
function criarLancamento(tipo: string, descricao: string, valor: number, categoria: string, data: string) {}
```

**Bom:**

```ts
interface EntradaLancamento {
  tipo: string;
  descricao: string;
  valor: number;
  categoria: string;
  data: string;
}

function criarLancamento(entrada: EntradaLancamento) {}
```

## 6. Sem linhas em branco dentro de funções

Linhas em branco são permitidas entre funções de um arquivo e entre membros de uma classe, mas não dentro de uma função.

**Ruim:**

```ts
function formatarValor(valor: number): string {
  const reais = valor / 100;

  return reais.toFixed(2);
}
```

**Bom:**

```ts
function formatarValor(valor: number): string {
  const reais = valor / 100;
  return reais.toFixed(2);
}
```

## 7. Constantes no lugar de números e textos mágicos

**Ruim:**

```ts
if (valorCentavos <= 0 || descricao.length > 80) {
  throw new Error('inválido');
}
```

**Bom:**

```ts
const TAMANHO_MAXIMO_DESCRICAO = 80;
const MENSAGEM_LANCAMENTO_INVALIDO = 'Lançamento inválido';

if (valorCentavos <= 0 || descricao.length > TAMANHO_MAXIMO_DESCRICAO) {
  throw new Error(MENSAGEM_LANCAMENTO_INVALIDO);
}
```

## 8. Variáveis declaradas perto de onde são usadas

**Ruim:**

```ts
function resumir(lancamentos: Lancamento[]) {
  const receitas = [];
  const despesas = [];
  const hoje = new Date();
  validar(lancamentos);
  return filtrarPorTipo(lancamentos, 'receita');
}
```

**Bom:**

```ts
function resumir(lancamentos: Lancamento[]) {
  validar(lancamentos);
  const receitas = filtrarPorTipo(lancamentos, 'receita');
  return somar(receitas);
}
```

## 9. Dados sensíveis nunca no código

Chaves de API, senhas e tokens ficam em um arquivo `.env`, fora do versionamento, e são lidos por variável de ambiente.

**Ruim:**

```ts
const chaveApi = 'sk-123456';
```

**Bom:**

```ts
const chaveApi = process.env.CHAVE_API;
```

Mantenha um `.env.example` com os nomes das variáveis, sem os valores reais.