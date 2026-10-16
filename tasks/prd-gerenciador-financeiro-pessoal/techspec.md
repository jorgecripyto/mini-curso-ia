# Especificação técnica

## Resumo

Esta especificação detalha a solução técnica do [PRD do Gerenciador Financeiro Pessoal](./prd.md). O backend (Express 5, TypeScript) expõe uma API REST em `/api` e grava os lançamentos em SQLite pelo módulo embutido `node:sqlite`, com consultas parametrizadas. O frontend (React 19, Vite, Tailwind) segue a arquitetura em camadas das rules: `view` usa `hook`, o `hook` chama um `service`, e o `service` fala com a API.

Os valores em reais trafegam como texto decimal com duas casas (`"12.50"`) entre tela, API e banco. Para evitar erros de arredondamento, toda soma é feita no backend, convertendo cada valor para centavos inteiros e voltando para texto decimal. O banco nunca soma colunas de texto. O mês consultado é filtrado por intervalo de datas (`AAAA-MM-DD`), e os testes seguem a pirâmide das rules: unidade, integração e E2E com Playwright.

## Arquitetura do sistema

### Visão dos componentes

**Backend (`backend/src/`)**

- `index.ts` (modificado): sobe o servidor e faz o graceful shutdown, fechando também o banco.
- `app.ts` (novo): monta o Express, registra as rotas e o tratador de erros.
- `config.ts` (novo): lê o `.env` com dotenv e exporta `porta` e `caminhoBanco`.
- `logger.ts` (novo): adaptador central de logs (`info` e `erro`).
- `routes/lancamentos-routes.ts` e `routes/categorias-routes.ts` (novos): protocolo HTTP, sem regra de negócio.
- `routes/error-handler.ts` (novo): converte erros em envelope `ErroApi`.
- `services/lancamentos-service.ts` (novo): orquestra validação, persistência e resumo.
- `services/validar-lancamento.ts`, `valor-centavos.ts`, `resumo-mensal.ts`, `periodo.ts`, `categorias.ts` (novos): regras puras.
- `data/database.ts` e `data/lancamentos-repository.ts` (novos): conexão, criação da tabela e queries.
- `types/` (novos, um arquivo por tipo): `lancamento.ts`, `tipo-lancamento.ts`, `entrada-lancamento.ts`, `resumo-mensal.ts`, `consulta-mensal.ts`, `periodo.ts`, `campo-invalido.ts`, `resultado-validacao.ts`, `erro-api.ts`, `erro-aplicacao.ts`.

**Frontend (`frontend/src/`)**

- `App.tsx` (modificado): apenas renderiza a view. O health check do template sai.
- `views/gerenciador-financeiro.tsx` (nova): tela única, controla o mês selecionado.
- `components/` (novos): `seletor-mes.tsx`, `resumo-mensal.tsx`, `lista-lancamentos.tsx`, `item-lancamento.tsx`, `formulario-lancamento.tsx`, `campo-formulario.tsx`, `mensagem-status.tsx`, `confirmacao-exclusao.tsx`.
- `hooks/` (novos): `use-consulta-mensal.ts`, `use-categorias.ts`, `use-formulario-lancamento.ts`.
- `services/` (novos): `lancamentos-api.ts`, `categorias-api.ts`, `apresentacao.ts` (moeda e data), `valor-formulario.ts`, `mes.ts`.
- `types/` (novos): os mesmos tipos de contrato do backend, um arquivo por tipo.

**Fluxo de dados:** a view usa um hook, o hook chama o service, o service chama a API, a rota chama o service do backend, que chama o repository, que lê e grava no SQLite. A resposta volta pelo mesmo caminho.

## Design de implementação

### Principais interfaces

```
LancamentosService (backend)
  listarPorMes(mes: string) -> Promise<ConsultaMensal>
  criar(entrada: unknown) -> Promise<Lancamento>
  atualizar(id: number, entrada: unknown) -> Promise<Lancamento>
  excluir(id: number) -> Promise<void>
```

```
LancamentosRepository (backend)
  listarPorPeriodo(periodo: Periodo) -> Promise<Lancamento[]>
  inserir(entrada: EntradaLancamento) -> Promise<Lancamento>
  atualizar(id: number, entrada: EntradaLancamento) -> Promise<Lancamento | null>
  excluir(id: number) -> Promise<boolean>
```

```
Funções puras (backend/services)
  validarLancamento(entrada: unknown) -> ResultadoValidacao
  decimalParaCentavos(valor: string) -> number
  centavosParaDecimal(centavos: number) -> string
  calcularResumo(lancamentos: Lancamento[]) -> ResumoMensal
  criarPeriodo(mes: string) -> Periodo
```

```
Services do frontend
  buscarConsultaMensal(mes: string) -> Promise<ConsultaMensal>
  criarLancamento(entrada: EntradaLancamento) -> Promise<Lancamento>
  atualizarLancamento(id: number, entrada: EntradaLancamento) -> Promise<Lancamento>
  excluirLancamento(id: number) -> Promise<void>
  buscarCategorias() -> Promise<string[]>
  normalizarValorDigitado(texto: string) -> string | null
  formatarMoeda(valor: string) -> string
  formatarData(data: string) -> string
  mesAtual() -> string
```

```
Hooks do frontend
  useConsultaMensal(mes: string) -> { consulta, carregando, erro, recarregar }
  useCategorias() -> { categorias, erro }
  useFormularioLancamento(opcoes: OpcoesFormulario) -> { campos, erros, enviando, alterarCampo, enviar, limpar }
```

### Modelos de dados

#### `Lancamento` — lançamento gravado

| Campo | Tipo | Obrigatório | Descrição |
| ----- | ---- | ----------- | --------- |
| `id` | `number` | sim | Identificador inteiro gerado pelo banco |
| `tipo` | `"receita" \| "despesa"` | sim | Tipo do lançamento |
| `descricao` | `string` | sim | Texto de 1 a 80 caracteres, sem espaços nas pontas |
| `valor` | `string` | sim | Decimal positivo com exatamente 2 casas, usando ponto (`"12.50"`) |
| `categoria` | `string` | sim | Uma das categorias da lista fixa |
| `data` | `string` | sim | Data válida no formato `AAAA-MM-DD` |

```text
{
  "id": 7,
  "tipo": "despesa",
  "descricao": "Supermercado",
  "valor": "154.90",
  "categoria": "Alimentação",
  "data": "2026-10-12"
}
```

#### `EntradaLancamento` — corpo de POST e PUT

Os mesmos campos de `Lancamento`, exceto `id`. Todos são obrigatórios.

```text
{
  "tipo": "receita",
  "descricao": "Salário de outubro",
  "valor": "3000.00",
  "categoria": "Salário",
  "data": "2026-10-05"
}
```

#### `ResumoMensal` — totais do mês

| Campo | Tipo | Obrigatório | Descrição |
| ----- | ---- | ----------- | --------- |
| `receitas` | `string` | sim | Soma das receitas, decimal com 2 casas |
| `despesas` | `string` | sim | Soma das despesas, decimal com 2 casas |
| `saldo` | `string` | sim | Receitas menos despesas. Pode ser negativo (`"-5.00"`) |

```text
{
  "receitas": "3000.00",
  "despesas": "1200.00",
  "saldo": "1800.00"
}
```

#### `ConsultaMensal` — resposta da consulta de um mês

| Campo | Tipo | Obrigatório | Descrição |
| ----- | ---- | ----------- | --------- |
| `mes` | `string` | sim | Mês consultado, formato `AAAA-MM` |
| `lancamentos` | `Lancamento[]` | sim | Lista do mês, da data mais recente para a mais antiga |
| `resumo` | `ResumoMensal` | sim | Totais calculados no backend |

> **Mês sem lançamentos:** `lancamentos` vem como `[]` e o resumo traz `"0.00"` nos três campos.

```text
{
  "mes": "2026-11",
  "lancamentos": [],
  "resumo": { "receitas": "0.00", "despesas": "0.00", "saldo": "0.00" }
}
```

#### `ListaCategorias` — categorias permitidas

A lista é fixa e vive só no backend (`services/categorias.ts`): Alimentação, Moradia, Transporte, Saúde, Lazer, Salário e Outros.

```text
{
  "categorias": ["Alimentação", "Moradia", "Transporte", "Saúde", "Lazer", "Salário", "Outros"]
}
```

#### `ErroApi` — envelope de erro

| Código | HTTP | Significado |
| ------ | ---- | ----------- |
| `VALIDACAO_INVALIDA` | `400` | Algum campo do lançamento é inválido ou está vazio |
| `CORPO_INVALIDO` | `400` | O corpo da requisição não é um JSON válido |
| `MES_INVALIDO` | `400` | O parâmetro `mes` não segue `AAAA-MM` |
| `ID_INVALIDO` | `400` | O `id` da rota não é um inteiro positivo |
| `LANCAMENTO_NAO_ENCONTRADO` | `404` | Não existe lançamento com o `id` informado |
| `ERRO_INTERNO` | `500` | Falha inesperada, sem detalhes internos na mensagem |

```text
{
  "error": {
    "code": "VALIDACAO_INVALIDA",
    "message": "Há campos inválidos no lançamento",
    "campos": [
      { "campo": "valor", "mensagem": "Informe um valor maior que zero" }
    ]
  }
}
```

> O campo `campos` só aparece em `VALIDACAO_INVALIDA`. As mensagens são em português, porque a interface as exibe diretamente.

#### `lancamentos` — tabela do banco

| Coluna | Tipo SQLite | Regras |
| ------ | ----------- | ------ |
| `id` | `INTEGER` | Chave primária com autoincremento |
| `tipo` | `TEXT` | Obrigatório, restrito a `receita` ou `despesa` |
| `descricao` | `TEXT` | Obrigatório |
| `valor` | `TEXT` | Obrigatório, decimal com 2 casas |
| `categoria` | `TEXT` | Obrigatório |
| `data` | `TEXT` | Obrigatório, `AAAA-MM-DD` |

```sql
CREATE TABLE IF NOT EXISTS lancamentos (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  tipo TEXT NOT NULL CHECK (tipo IN ('receita', 'despesa')),
  descricao TEXT NOT NULL,
  valor TEXT NOT NULL,
  categoria TEXT NOT NULL,
  data TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_lancamentos_data ON lancamentos (data);
```

#### Consultas parametrizadas

Toda consulta usa `?` para os valores. Nenhum valor é concatenado no texto SQL.

| Operação | SQL |
| -------- | --- |
| Listar por período | `SELECT id, tipo, descricao, valor, categoria, data FROM lancamentos WHERE data >= ? AND data < ? ORDER BY data DESC, id DESC` |
| Inserir | `INSERT INTO lancamentos (tipo, descricao, valor, categoria, data) VALUES (?, ?, ?, ?, ?)` |
| Atualizar | `UPDATE lancamentos SET tipo = ?, descricao = ?, valor = ?, categoria = ?, data = ? WHERE id = ?` |
| Excluir | `DELETE FROM lancamentos WHERE id = ?` |

> O período usa o primeiro dia do mês como limite inicial (incluso) e o primeiro dia do mês seguinte como limite final (excluso). A virada de dezembro para janeiro é tratada em `criarPeriodo`. **Proibido usar `SUM` sobre a coluna `valor`**: ela é texto, e a soma é feita em `calcularResumo`.

#### Variáveis de ambiente

| Variável | Onde | Padrão | Descrição |
| -------- | ---- | ------ | --------- |
| `PORT` | backend | `3000` | Porta da API |
| `DATABASE_PATH` | backend | `data/financas.sqlite` | Arquivo do banco. A pasta é criada se não existir |
| `VITE_API_URL` | frontend | `http://localhost:3000` | Endereço base da API |

Cada projeto mantém um `.env.example` com esses nomes.

### Endpoints da API

#### Visão geral

| Método | Rota | Descrição |
| ------ | ---- | --------- |
| `GET` | `/api/categorias` | Lista as categorias permitidas |
| `GET` | `/api/lancamentos?mes=AAAA-MM` | Lançamentos e resumo de um mês |
| `POST` | `/api/lancamentos` | Cria um lançamento |
| `PUT` | `/api/lancamentos/:id` | Atualiza um lançamento |
| `DELETE` | `/api/lancamentos/:id` | Exclui um lançamento |

O endpoint `GET /health` do template continua existindo, sem alterações.

---

#### `GET /api/categorias`

Devolve a lista fixa de categorias, para o formulário montar o seletor.

**Respostas**

| Status | Corpo | Quando |
| ------ | ----- | ------ |
| `200` | `ListaCategorias` | Sempre que a API está de pé |

**Exemplo — sucesso**

```http
GET /api/categorias
```

Corpo: o exemplo de `ListaCategorias`.

---

#### `GET /api/lancamentos?mes=AAAA-MM`

Devolve os lançamentos do mês e os totais já calculados.

**Parâmetros de consulta**

| Parâmetro | Tipo | Padrão | Regras |
| --------- | ---- | ------ | ------ |
| `mes` | `string` | — | Obrigatório. Formato `AAAA-MM`, com mês de `01` a `12` |

**Respostas**

| Status | Corpo | Quando |
| ------ | ----- | ------ |
| `200` | `ConsultaMensal` | Mês válido, com ou sem lançamentos |
| `400` | `ErroApi` | `mes` ausente ou inválido (`MES_INVALIDO`) |
| `500` | `ErroApi` | Falha no banco (`ERRO_INTERNO`) |

**Exemplo — sucesso**

```http
GET /api/lancamentos?mes=2026-10
```

```text
{
  "mes": "2026-10",
  "lancamentos": [
    { "id": 7, "tipo": "despesa", "descricao": "Supermercado", "valor": "154.90", "categoria": "Alimentação", "data": "2026-10-12" },
    { "id": 6, "tipo": "receita", "descricao": "Salário de outubro", "valor": "3000.00", "categoria": "Salário", "data": "2026-10-05" }
  ],
  "resumo": { "receitas": "3000.00", "despesas": "154.90", "saldo": "2845.10" }
}
```

**Exemplo — mês sem lançamentos**

```http
GET /api/lancamentos?mes=2026-11
```

Corpo: o exemplo de mês vazio em `ConsultaMensal`.

> O frontend mostra a mensagem de lista vazia e o saldo `R$ 0,00` quando `lancamentos` vem vazio.

**Exemplo — erro**

```http
GET /api/lancamentos?mes=2026-13
```

```text
{
  "error": { "code": "MES_INVALIDO", "message": "Informe o mês no formato AAAA-MM" }
}
```

---

#### `POST /api/lancamentos`

Cria um lançamento.

**Corpo**

| Parâmetro | Tipo | Padrão | Regras |
| --------- | ---- | ------ | ------ |
| `tipo` | `string` | — | Obrigatório. `receita` ou `despesa` |
| `descricao` | `string` | — | Obrigatório. De 1 a 80 caracteres após remover espaços das pontas |
| `valor` | `string` | — | Obrigatório. Regex `^\d{1,9}\.\d{2}$` e maior que zero |
| `categoria` | `string` | — | Obrigatório. Uma das categorias da lista fixa |
| `data` | `string` | — | Obrigatório. `AAAA-MM-DD` e data existente no calendário |

**Respostas**

| Status | Corpo | Quando |
| ------ | ----- | ------ |
| `201` | `Lancamento` | Lançamento criado |
| `400` | `ErroApi` | Campo inválido (`VALIDACAO_INVALIDA`) ou JSON quebrado (`CORPO_INVALIDO`) |
| `500` | `ErroApi` | Falha no banco (`ERRO_INTERNO`) |

**Exemplo — sucesso**

```http
POST /api/lancamentos
```

Corpo da requisição: o exemplo de `EntradaLancamento`. Resposta: um `Lancamento` com o `id` gerado.

**Exemplo — erro de validação**

```http
POST /api/lancamentos
```

```text
{
  "tipo": "despesa",
  "descricao": "",
  "valor": "0.00",
  "categoria": "Lazer",
  "data": "2026-10-15"
}
```

Resposta `400`:

```text
{
  "error": {
    "code": "VALIDACAO_INVALIDA",
    "message": "Há campos inválidos no lançamento",
    "campos": [
      { "campo": "descricao", "mensagem": "Informe a descrição" },
      { "campo": "valor", "mensagem": "Informe um valor maior que zero" }
    ]
  }
}
```

> Quando há vários campos inválidos, todos são listados de uma vez. Nada é gravado.

---

#### `PUT /api/lancamentos/:id`

Substitui todos os campos de um lançamento existente. Usa o mesmo corpo e as mesmas regras do `POST`.

**Parâmetros de rota**

| Parâmetro | Tipo | Padrão | Regras |
| --------- | ---- | ------ | ------ |
| `id` | `number` | — | Inteiro positivo |

**Respostas**

| Status | Corpo | Quando |
| ------ | ----- | ------ |
| `200` | `Lancamento` | Lançamento atualizado |
| `400` | `ErroApi` | `ID_INVALIDO`, `VALIDACAO_INVALIDA` ou `CORPO_INVALIDO` |
| `404` | `ErroApi` | `LANCAMENTO_NAO_ENCONTRADO` |
| `500` | `ErroApi` | Falha no banco (`ERRO_INTERNO`) |

**Exemplo — sucesso**

```http
PUT /api/lancamentos/7
```

Corpo da requisição: um `EntradaLancamento`. Resposta: o `Lancamento` com os novos dados e o mesmo `id`.

**Exemplo — erro**

```http
PUT /api/lancamentos/999
```

```text
{
  "error": { "code": "LANCAMENTO_NAO_ENCONTRADO", "message": "Lançamento não encontrado" }
}
```

> Se a `data` mudar de mês, o lançamento sai do resumo do mês antigo e entra no do novo, porque o resumo é sempre recalculado na consulta.

---

#### `DELETE /api/lancamentos/:id`

Exclui definitivamente um lançamento.

**Parâmetros de rota**

| Parâmetro | Tipo | Padrão | Regras |
| --------- | ---- | ------ | ------ |
| `id` | `number` | — | Inteiro positivo |

**Respostas**

| Status | Corpo | Quando |
| ------ | ----- | ------ |
| `204` | sem corpo | Lançamento excluído |
| `400` | `ErroApi` | `ID_INVALIDO` |
| `404` | `ErroApi` | `LANCAMENTO_NAO_ENCONTRADO` |
| `500` | `ErroApi` | Falha no banco (`ERRO_INTERNO`) |

**Exemplo — sucesso**

```http
DELETE /api/lancamentos/7
```

Resposta `204`, sem corpo.

**Exemplo — erro**

```http
DELETE /api/lancamentos/999
```

Resposta `404` com o mesmo corpo do exemplo de erro do `PUT`.

> A confirmação da exclusão acontece só na interface. A API exclui assim que recebe a requisição.

---

## Pontos de integração

Não se aplica: o PRD deixa integrações externas fora do escopo. A única comunicação entre sistemas é frontend para backend, via `VITE_API_URL`, e o CORS já está habilitado no template. Falhas da API aparecem na interface como mensagem de erro clara (CA-13).

## Abordagem de testes

Seguem as rules: vitest no backend e no frontend, cobertura mínima de 80% (linhas, funções, branches e statements) configurada como limite que quebra o `npm test`, e Playwright na pasta `e2e/`. Os testes ficam ao lado do arquivo testado (`.test.ts` e `.test.tsx`) e seguem Arrange, Act e Assert.

### Testes de unidade

| ID | Nome do caso de teste | Critérios de aceitação | Resultado esperado |
|----|-----------------------|------------------------|--------------------|
| TU-01 | `decimalParaCentavos` converte `"12.50"` | CA-06 | `1250` |
| TU-02 | `decimalParaCentavos` converte `"0.10"` e `"0.20"` | CA-06 | `10` e `20`, sem ponto flutuante |
| TU-03 | `centavosParaDecimal` converte `30` e `180000` | CA-05, CA-06 | `"0.30"` e `"1800.00"` |
| TU-04 | `centavosParaDecimal` converte `-500` | CA-05 | `"-5.00"` |
| TU-05 | `calcularResumo` com receitas `3000.00` e despesas `1200.00` | CA-05 | saldo `"1800.00"` |
| TU-06 | `calcularResumo` com despesas `0.10` e `0.20` | CA-06 | despesas `"0.30"` |
| TU-07 | `calcularResumo` com lista vazia | CA-08 | `"0.00"` nos três totais |
| TU-08 | `validarLancamento` com entrada válida | CA-01 | `valida: true` com a entrada normalizada |
| TU-09 | `validarLancamento` com campos vazios | CA-02, CA-12 | um item em `campos` para cada campo vazio |
| TU-10 | `validarLancamento` com valor `"0.00"`, `"-1.00"`, `"12.345"` e `"abc"` | CA-03, CA-12 | recusa com erro no campo `valor` |
| TU-11 | `validarLancamento` com categoria fora da lista e tipo inválido | CA-12 | erros em `categoria` e `tipo` |
| TU-12 | `validarLancamento` com data `"2026-02-30"` | CA-12 | erro no campo `data` |
| TU-13 | `criarPeriodo("2026-12")` | CA-07 | início `2026-12-01`, fim `2027-01-01` |
| TU-14 | `criarPeriodo` com `"2026-13"` e `"outubro"` | CA-12 | lança `MES_INVALIDO` |
| TU-15 | `config` sem variáveis de ambiente | CA-09 | porta `3000` e caminho `data/financas.sqlite` |
| TU-16 | `normalizarValorDigitado` com `"12,5"` e `"12"` | CA-01 | `"12.50"` e `"12.00"` |
| TU-17 | `normalizarValorDigitado` com 3 casas e texto | CA-03 | `null` |
| TU-18 | `formatarMoeda("1800.00")` | CA-05 | `"R$ 1.800,00"` |
| TU-19 | `formatarData("2026-10-15")` | CA-04 | `"15/10/2026"` |
| TU-20 | `buscarConsultaMensal` com `fetch` simulado | CA-13 | monta a URL com o mês e converte erro da API em mensagem |
| TU-21 | `criarLancamento` com falha de rede simulada | CA-13 | lança mensagem clara em português |
| TU-22 | `mesAtual` com relógio simulado em 15/10/2026 | CA-07 | `"2026-10"` |
| TU-23 | `useConsultaMensal` | CA-13 | passa por carregando, sucesso e erro |
| TU-24 | `ItemLancamento` renderizado | CA-04 | mostra tipo, descrição, valor em reais, categoria e data |
| TU-25 | `ResumoMensal` renderizado, com mês vazio | CA-05, CA-08 | mostra os três totais e `R$ 0,00` no mês vazio |

Estratégia: as regras puras (valor, período, validação, resumo) são o núcleo e têm a maior bateria de casos. Usa-se mock só no que é externo ou instável: `fetch` e o relógio. O service do backend é testado com um repository substituto, e o repository real é coberto na integração.

### Testes de integração

| ID | Nome do caso de teste | Critérios de aceitação | Resultado esperado |
|----|-----------------------|------------------------|--------------------|
| TI-01 | POST com dados válidos, depois GET do mês | CA-01, CA-04 | `201` e o item aparece na lista com todos os campos |
| TI-02 | POST com campos vazios | CA-02, CA-12 | `400 VALIDACAO_INVALIDA` listando os campos, nada gravado |
| TI-03 | POST com valor `"0.00"` | CA-03, CA-12 | `400` e nenhum registro novo |
| TI-04 | GET com receitas e despesas, incluindo `0.10` e `0.20` | CA-05, CA-06 | totais e saldo exatos |
| TI-05 | GET de meses diferentes e de mês vazio | CA-07, CA-08 | cada mês traz só os seus itens, o vazio traz `[]` e `"0.00"` |
| TI-06 | Gravar, fechar e reabrir o banco em arquivo temporário | CA-09 | os mesmos lançamentos continuam disponíveis |
| TI-07 | PUT trocando a data para outro mês, e PUT em `id` inexistente | CA-10 | o item muda de mês nos dois resumos, e o inexistente devolve `404` |
| TI-08 | DELETE de item existente e de `id` inexistente | CA-11 | `204` com resumo recalculado, e `404` |
| TI-09 | GET com `mes` inválido | CA-12 | `400 MES_INVALIDO` |
| TI-10 | POST com descrição contendo aspas e `'; DROP TABLE lancamentos; --` | CA-12 | gravada como texto e a tabela continua íntegra |
| TI-11 | Falha simulada do banco | CA-13 | `500 ERRO_INTERNO` sem detalhes internos |
| TI-12 | View carrega o mês atual com `fetch` simulado | CA-04, CA-05 | lista e resumo aparecem |
| TI-13 | Formulário válido enviado | CA-01 | mensagem em `role="status"` e lista atualizada |
| TI-14 | Formulário com campo vazio e com valor zero | CA-02, CA-03, CA-15 | erro por campo com `role="alert"` e `aria-describedby` |
| TI-15 | Editar e excluir, confirmando e cancelando | CA-10, CA-11 | confirmar atualiza a lista, cancelar não chama a API |
| TI-16 | Falha da API ao salvar | CA-13 | mensagem de erro e campos do formulário preservados |

Os testes de API sobem o `app` em porta efêmera e usam o `fetch` nativo do Node, sem biblioteca extra. Cada teste usa um banco `:memory:` próprio (TI-06 usa arquivo temporário), para serem independentes. Os de frontend usam Testing Library com `fetch` simulado.

### Testes E2E

| ID | Nome do caso de teste | Critérios de aceitação | Resultado esperado |
|----|-----------------------|------------------------|--------------------|
| E2E-01 | Cadastrar uma receita e uma despesa | CA-01, CA-04, CA-05 | itens na lista, mensagem de sucesso e saldo correto |
| E2E-02 | Enviar formulário inválido | CA-02, CA-03 | mensagens por campo e nada na lista |
| E2E-03 | Trocar de mês e abrir um mês vazio | CA-07, CA-08 | listas separadas, mensagem de vazio e `R$ 0,00` |
| E2E-04 | Recarregar a página e reiniciar a API | CA-09 | os lançamentos continuam na tela |
| E2E-05 | Editar um lançamento mudando o mês | CA-10 | sai do mês antigo e aparece no novo |
| E2E-06 | Excluir confirmando e cancelando | CA-11 | cancelar mantém o item, confirmar remove e recalcula |
| E2E-07 | API indisponível ao salvar | CA-13 | mensagem de erro e formulário preservado |
| E2E-08 | Viewport de 360 px | CA-14 | todas as funções acessíveis, sem rolagem horizontal |
| E2E-09 | Fluxo completo só pelo teclado | CA-15 | cadastro feito sem mouse e mensagens anunciadas por `aria-live` |

O projeto `e2e/` tem o próprio `package.json` e o `playwright.config.ts`, que sobe a API (com `DATABASE_PATH` temporário) e o frontend em portas próprias (`3100` e `5273`), sem conflitar com o ambiente de desenvolvimento.

## Sequenciamento do desenvolvimento

### Ordem de construção

1. **Ferramentas de teste**: vitest e cobertura no backend e no frontend, scripts `test` e `test:coverage`, `tsconfig` do backend excluindo os testes do build. Vem primeiro para que tudo nasça com teste.
2. **Backend, regras puras**: `types`, `valor-centavos`, `periodo`, `categorias`, `validar-lancamento` e `resumo-mensal`, cada um com seus TU. É o núcleo do PRD e não depende de nada.
3. **Backend, infraestrutura**: `config`, `logger`, `database` e `lancamentos-repository`, com TI-06 e TI-10.
4. **Backend, API**: `lancamentos-service`, rotas, `error-handler`, `app` e `index` com graceful shutdown, com os demais TI.
5. **Frontend, base**: tipos, `services` (API, apresentação, valor, mês) e hooks, com seus TU.
6. **Frontend, interface**: componentes, view e `App`, com TI-12 a TI-16.
7. **E2E**: Playwright, depois que backend e frontend funcionam juntos.
8. **Fechamento**: lint, build, cobertura mínima, e atualização do `AGENTS.md` com os comandos de teste.

### Dependências técnicas

- Node.js 22.13 ou superior (`node:sqlite` sem flag).
- Backend, desenvolvimento: `vitest` e `@vitest/coverage-v8`.
- Frontend, desenvolvimento: `vitest`, `@vitest/coverage-v8`, `jsdom`, `@testing-library/react`, `@testing-library/user-event` e `@testing-library/jest-dom`.
- E2E: `@playwright/test` e download dos navegadores do Playwright (exige rede na primeira instalação).

## Monitoramento e observabilidade

- Health check: `GET /health` do template, sem alterações.
- Logs sempre pelo adaptador `logger`, nunca por `console` direto.

| Evento | Nível |
| ------ | ----- |
| Servidor iniciado, com a porta | `info` |
| Sinal de desligamento recebido, servidor e banco fechados | `info` |
| Lançamento criado, atualizado ou excluído, com o `id` | `info` |
| Falha ao processar uma requisição, com a causa | `erro` |

A causa completa vai para o log. O cliente recebe apenas `ERRO_INTERNO` com mensagem genérica.

## Considerações técnicas

### Principais decisões

- **Banco `node:sqlite`** (escolha do solicitante): nada para instalar. A [documentação do Node](https://nodejs.org/api/sqlite.html) indica que, desde a 22.13, ele não exige mais flag, mas segue experimental. Alternativa descartada: `better-sqlite3`, por ser uma dependência nativa externa.
- **Valor como texto decimal** (escolha do solicitante): o contrato fica legível (`"12.50"`). O custo é a conversão obrigatória para centavos em `valor-centavos.ts` antes de qualquer conta, e a proibição de `SUM` no SQL. Alternativa descartada: inteiro em centavos no contrato, mais simples de somar, mas não escolhida.
- **Categorias servidas pela API** (escolha do solicitante): fonte única em `services/categorias.ts`, usada na validação e em `GET /api/categorias`. Alternativa descartada: lista duplicada no frontend e no backend, por risco de divergência.
- **Totais calculados no backend**: o frontend só exibe o que a API devolve, o que mantém a regra do saldo num único lugar.
- **Seletor de mês com dois `select`** (mês e ano), em vez de `input type="month"`, para funcionar igual em todos os navegadores. Cada um tem rótulo e é acessível por teclado.
- **Após salvar ou editar**, a view passa a mostrar o mês da data do lançamento, para o item aparecer na tela conforme CA-01 e CA-10.
- **Descrição limitada a 80 caracteres**: o PRD não define limite, então esta é uma premissa técnica, mantida em constante.
- **Validação manual**, sem biblioteca de esquemas, para manter o escopo enxuto. Alternativa descartada: `zod`, por adicionar dependência para poucas regras.
- **Exclusão definitiva**, com confirmação inline na interface (não `window.confirm`), para ser testável e acessível. Alternativa descartada: exclusão lógica, que o PRD não pede.
- **Sem `supertest`**: testes de API usam `fetch` nativo.

### Riscos conhecidos

- **Erro de arredondamento por descuido**: qualquer soma feita com `Number` ou `SUM` sobre texto quebra o CA-06. Mitigação: só `valor-centavos.ts` converte valores, e TU-01 a TU-07 e TI-04 cobrem o caso `0.10 + 0.20`.
- **`node:sqlite` é síncrono e experimental**: a API pode mudar entre versões do Node, e as chamadas bloqueiam o event loop. Mitigação: as consultas locais são rápidas, o repository expõe métodos `async` para isolar essa troca e o `engines` do backend fixa o Node mínimo.
- **Fuso horário**: o mês atual é calculado no navegador. Datas são guardadas como texto `AAAA-MM-DD`, sem fuso, para evitar deslocamento de dia. Cobertura: TU-22 com relógio simulado.
- **Template fora das rules**: o `index.ts` e o `App.tsx` do template usam `any`, `useEffect` desnecessário, URL fixa no código e parâmetros não usados (que, pelas opções do `tsconfig`, devem falhar no build). Eles serão reescritos, e o `npm run build` confirma. Parâmetros não usados levam prefixo `_`.
- **Testes dentro de `src`**: sem ajuste, o build do backend os compilaria para `dist`. Mitigação: `exclude` de `**/*.test.ts` no `tsconfig`.
- **Download dos navegadores do Playwright** exige rede, e pode atrasar o E2E.
- **Backend sem script de lint**: conforme as rules, o `npm run build` faz a checagem de tipos enquanto não houver lint.

### Conformidade com o AGENTS.md e as rules

Foram lidos o `AGENTS.md` e todas as rules em `.agents/rules/`: `code-standards`, `tests`, `react`, `node`, `folder-structure` e `javascript-typescript`. Restrições aplicadas a esta especificação:

- Arquivos com até 100 linhas, funções com até 30 linhas e até 3 parâmetros (por isso `OpcoesFormulario` e `EntradaLancamento` são objetos).
- Sem comentários, constantes no lugar de números e textos mágicos, cláusulas guarda no lugar de `if` aninhado.
- Sem `any`, comparação com `===`, ternários sem aninhamento, tipos explícitos em parâmetros e retornos com objetos.
- Estrutura do backend em `routes`, `services`, `data` e `types` (um arquivo por tipo) e do frontend em `components`, `views`, `services`, `hooks`, `assets` e `types`.
- Componentes React com até 30 linhas, props explícitas, acessibilidade com `aria-*`, estilo com Tailwind, sem `fetch` dentro de componente.
- Node: `async/await`, `.env` com dotenv, graceful shutdown, logger centralizado, lock files versionados e sem referências circulares.
- Testes: vitest, cobertura mínima de 80%, FIRST, pirâmide, E2E com Playwright em `e2e/`, e lint ao concluir cada tarefa.

### Conformidade com skills

- `criar-prd`: o PRD que origina esta especificação segue o template da skill e está em `tasks/prd-gerenciador-financeiro-pessoal/prd.md`.
- `criar-techspec`: este documento segue o template da skill. Sem desvios a registrar.

### Arquivos relevantes e dependentes

- Modificados: `backend/src/index.ts`, `backend/package.json`, `backend/tsconfig.json`, `backend/.gitignore`, `frontend/src/App.tsx`, `frontend/package.json`, `frontend/vite.config.ts` e `AGENTS.md`.
- Novos no backend: `.env.example`, `vitest.config.ts` e os arquivos de `src` listados na visão dos componentes.
- Novos no frontend: `.env.example`, `vitest.config.ts` e os arquivos de `src` listados na visão dos componentes.
- Novos na raiz: a pasta `e2e/` (`package.json`, `playwright.config.ts` e os testes).
- Referência: `tasks/prd-gerenciador-financeiro-pessoal/prd.md`.