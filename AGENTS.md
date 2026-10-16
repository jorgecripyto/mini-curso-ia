# Gerenciador Financeiro Pessoal

Aplicação para registrar receitas e despesas e acompanhar o saldo mensal. São dois projetos independentes na mesma raiz: `backend` e `frontend`.

## Backend (`backend/`)

- Papel: API HTTP do sistema.
- Tecnologia: Node.js, TypeScript, Express 5, dotenv e cors.
- Porta: 3000 (variável de ambiente `PORT`).
- Entrada: `src/index.ts`. Endpoint de saúde: `GET /health`.

## Frontend (`frontend/`)

- Papel: interface web do usuário.
- Tecnologia: React 19, TypeScript, Vite, Tailwind CSS 3 e componentes shadcn/ui.
- Porta: 5173 (padrão do Vite).
- Consome a API em `http://localhost:3000`.

## Comandos

Backend (dentro de `backend/`):

- Instalar dependências: `npm ci`
- Desenvolvimento: `npm run dev`
- Build: `npm run build`
- Produção: `npm start` (depois do build)
- Testes: ainda não configurados (o `npm test` atual é só um placeholder)

Frontend (dentro de `frontend/`):

- Instalar dependências: `npm ci`
- Desenvolvimento: `npm run dev`
- Build: `npm run build`
- Lint: `npm run lint`
- Checagem de tipos: `npm run typecheck`
- Testes: ainda não configurados

## Estrutura de pastas

```
backend/
  src/index.ts
frontend/
  src/
    App.tsx
    main.tsx
    components/ui/button.tsx
    lib/utils.ts
```

## Regras

Leia as regras abaixo antes de alterar o código:

- [Padrões de código](.agents/rules/code-standards.md)

- [Regras de testes](.agents/rules/tests.md)

- [Regras de React](.agents/rules/react.md)

- [Regras de Node](.agents/rules/node.md)
