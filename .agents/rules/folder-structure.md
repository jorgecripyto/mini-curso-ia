# Estrutura de pastas

## Raiz do projeto

```
backend/
frontend/
e2e/
.agents/rules/
```

- `backend/`: API.
- `frontend/`: interface web.
- `e2e/`: testes de ponta a ponta com playwright, fora de `backend` e `frontend`.
- `.agents/rules/`: regras para o agente.

## Backend

```
backend/src/
  index.ts
  app.ts
  config.ts
  logger.ts
  routes/
  services/
  data/
  types/
```

- `index.ts`: sobe o servidor e faz o graceful shutdown.
- `app.ts`: monta o Express e registra as rotas.
- `config.ts`: lê o `.env` e exporta a configuração.
- `logger.ts`: adaptador central de logs.
- `routes/`: lida com o protocolo HTTP. Lê a requisição, chama um service e devolve a resposta com o status correto. Não tem regra de negócio.
- `services/`: regras de negócio e comportamento da aplicação, como validação, conversão de valores e cálculo do saldo. Não conhece HTTP nem SQL.
- `data/`: dados. Banco, queries parametrizadas e acesso a APIs externas.
- `types/`: um arquivo por tipo. O nome do arquivo é o nome do tipo, em minúsculas e separado por hífen.

Direção das dependências: `routes` chama `services`, e `services` chama `data`. Nunca no sentido contrário.

## Frontend

```
frontend/src/
  components/
  views/
  services/
  hooks/
  assets/
  types/
  lib/
```

- `components/`: componentes pequenos e reutilizáveis. A subpasta `components/ui/` guarda os componentes base do shadcn/ui.
- `views/`: telas. Cada tela junta componentes e usa hooks.
- `services/`: acesso ao backend. Nenhum componente chama `fetch` diretamente.
- `hooks/`: lógica separada dos componentes, sempre com o prefixo `use`.
- `assets/`: imagens e arquivos estáticos.
- `types/`: um arquivo por tipo.
- `lib/`: utilitários gerais, como o `utils.ts` do shadcn/ui.

Fluxo: a `view` usa um `hook`, o `hook` chama um `service`, e o `service` fala com a API.

## Testes

Cada teste de unidade ou de integração fica ao lado do arquivo testado, com o sufixo `.test.ts` ou `.test.tsx`. Testes E2E ficam em `e2e/`.