# Regras de React

Estas regras valem para todo o código em `frontend/`.

## 1. Componentes pequenos e reutilizáveis

Um componente tem no máximo 30 linhas. Se passar disso, extraia partes em componentes menores, cada um com uma única responsabilidade.

**Ruim:** um `App` que renderiza o formulário, o seletor de mês, o resumo e a lista inteiros.

**Bom:**

```tsx
export function GerenciadorFinanceiro() {
  return (
    <main className="mx-auto max-w-3xl p-4">
      <SeletorMes />
      <ResumoMensal />
      <FormularioLancamento />
      <ListaLancamentos />
    </main>
  );
}
```

Um componente reutilizável recebe tudo por props e não conhece o resto da tela:

```tsx
interface BotaoAcaoProps {
  rotulo: string;
  aoClicar: () => void;
}

export function BotaoAcao({ rotulo, aoClicar }: BotaoAcaoProps) {
  return (
    <button type="button" onClick={aoClicar} className="rounded bg-primary px-4 py-2 text-primary-foreground">
      {rotulo}
    </button>
  );
}
```

## 2. Não repasse props com spread

Passe cada prop de forma explícita. Assim fica claro o que o componente recebe.

**Ruim:**

```tsx
<CampoTexto {...props} />
```

**Bom:**

```tsx
<CampoTexto rotulo={rotulo} valor={valor} aoMudar={aoMudar} />
```

## 3. Evite useEffect desnecessário

Não use `useEffect` para derivar um valor de outro. Calcule durante a renderização. Use `useEffect` apenas para sincronizar com algo externo, como uma chamada à API quando o mês muda.

**Ruim:**

```tsx
const [total, setTotal] = useState(0);

useEffect(() => {
  setTotal(lancamentos.reduce((soma, item) => soma + item.valorCentavos, 0));
}, [lancamentos]);
```

**Bom:**

```tsx
const total = lancamentos.reduce((soma, item) => soma + item.valorCentavos, 0);
```

## 4. useMemo para cálculos pesados

Use `useMemo` quando o cálculo for caro e as entradas mudarem pouco entre renderizações. Não use em cálculos simples.

```tsx
const resumo = useMemo(() => calcularResumo(lancamentos), [lancamentos]);
```

## 5. Acesso ao backend fora do componente

O componente nunca chama `fetch` diretamente. As chamadas à API ficam em `services/`.

**Ruim:**

```tsx
function ListaLancamentos() {
  const [itens, setItens] = useState<Lancamento[]>([]);
  useEffect(() => {
    fetch('http://localhost:3000/api/lancamentos').then((r) => r.json()).then(setItens);
  }, []);
  return <ul>{itens.map((item) => <li key={item.id}>{item.descricao}</li>)}</ul>;
}
```

**Bom:** a função fica em `services/lancamentos-api.ts`.

```ts
export async function buscarLancamentos(mes: string): Promise<Lancamento[]> {
  const resposta = await fetch(`${URL_API}/api/lancamentos?mes=${mes}`);
  if (!resposta.ok) throw new Error(MENSAGEM_ERRO_CONSULTA);
  return resposta.json();
}
```

## 6. Lógica em hooks com prefixo "use"

Estado, efeitos e regras ficam em hooks, e o componente só cuida da apresentação.

```ts
export function useLancamentos(mes: string) {
  const [lancamentos, setLancamentos] = useState<Lancamento[]>([]);
  const [erro, setErro] = useState<string | null>(null);
  useEffect(() => {
    buscarLancamentos(mes).then(setLancamentos).catch((e: Error) => setErro(e.message));
  }, [mes]);
  return { lancamentos, erro };
}
```

```tsx
export function ListaLancamentos({ mes }: { mes: string }) {
  const { lancamentos, erro } = useLancamentos(mes);
  if (erro) return <p role="alert">{erro}</p>;
  return <ul>{lancamentos.map((item) => <ItemLancamento key={item.id} lancamento={item} />)}</ul>;
}
```

## 7. Prefira componentes funcionais

Não crie componentes de classe. Use funções com hooks.

**Ruim:** `class ResumoMensal extends React.Component`.

**Bom:** `function ResumoMensal() { ... }`.

## 8. Acessibilidade sempre

Use propriedades `aria-*` e elementos semânticos. Todo campo tem rótulo associado. Mensagens de sucesso e erro são anunciadas para leitores de tela.

```tsx
<label htmlFor="valor">Valor</label>
<input
  id="valor"
  type="number"
  aria-invalid={temErro}
  aria-describedby="erro-valor"
/>
{temErro && <p id="erro-valor" role="alert">Informe um valor maior que zero</p>}
<p role="status" aria-live="polite">{mensagemSucesso}</p>
```

Botões só com ícone precisam de `aria-label`:

```tsx
<button type="button" aria-label="Excluir lançamento">
  <Trash2 aria-hidden="true" />
</button>
```

## 9. Estilização com Tailwind

Estilize os componentes com classes do Tailwind. Não use `style` inline nem arquivos CSS por componente.

**Ruim:**

```tsx
<div style={{ padding: 16, backgroundColor: '#eee' }}>...</div>
```

**Bom:**

```tsx
<div className="bg-muted p-4">...</div>
```