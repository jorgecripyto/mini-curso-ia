# Documento de Requisitos do Produto (PRD)

## Visão geral

O Gerenciador Financeiro Pessoal é uma aplicação web simples para registrar receitas e despesas e acompanhar o saldo de cada mês. Ele resolve a falta de visibilidade de quem controla o dinheiro de cabeça ou em anotações soltas: com poucos cliques a pessoa registra o que entrou e o que saiu e vê na hora quanto sobrou.

O público é uma pessoa física que quer controlar suas finanças pessoais, sem conhecimento técnico, usando computador ou celular. O valor oferecido é clareza: lançamentos organizados por mês, totais corretos e dados que não se perdem.

## Objetivos

- Registrar um lançamento completo em menos de 1 minuto.
- Exibir o saldo do mês correto em 100% dos casos, calculado como total de receitas menos total de despesas, sem erros de arredondamento.
- Manter 100% dos lançamentos salvos após recarregar a página e após reiniciar a aplicação.
- Recusar 100% dos lançamentos com valor menor ou igual a zero ou com campo obrigatório vazio, tanto na tela quanto na API.
- Carregar a lista e o resumo de um mês em até 2 segundos.
- Mostrar mensagem clara de sucesso, validação ou erro em toda ação do usuário.

## Histórias de usuário

- US1: Como usuário, quero cadastrar um lançamento com tipo, descrição, valor, categoria e data para registrar o que entrou ou saiu do meu dinheiro.
- US2: Como usuário, quero ver a lista de lançamentos do mês selecionado para conferir o que registrei.
- US3: Como usuário, quero ver o total de receitas, o total de despesas e o saldo do mês para saber quanto sobrou.
- US4: Como usuário, quero trocar o mês consultado para acompanhar meses diferentes.
- US5: Como usuário, quero editar um lançamento para corrigir um erro de digitação.
- US6: Como usuário, quero excluir um lançamento para remover um registro feito por engano.
- US7: Como usuário, quero receber mensagens claras quando algo estiver errado para saber como corrigir.
- US8: Como usuário, quero que meus lançamentos continuem salvos depois de fechar e reabrir a aplicação para não perder meu histórico.
- US9: Como usuário, quero usar a aplicação no celular com a mesma facilidade do computador.
- Casos de borda: mês sem lançamentos, valor inválido, campo obrigatório vazio, falha ao consultar ou gravar.

## Principais funcionalidades

### Cadastro de lançamento

Permite registrar uma receita ou despesa. É a ação central do produto.

- RF1: O usuário informa tipo (receita ou despesa), descrição, valor, categoria e data.
- RF2: Todos os campos são obrigatórios.
- RF3: O valor deve ser maior que zero e é informado em reais.
- RF4: A categoria é escolhida em uma lista fixa: Alimentação, Moradia, Transporte, Saúde, Lazer, Salário e Outros. A mesma lista vale para receitas e despesas (premissa).
- RF5: Ao salvar com sucesso, o lançamento aparece na lista do mês da sua data e o usuário vê uma mensagem de sucesso.

### Listagem por mês

Mostra os lançamentos de um mês escolhido.

- RF6: O usuário seleciona o mês consultado. Ao abrir a aplicação, o mês atual vem selecionado (premissa).
- RF7: A lista mostra, para cada lançamento, tipo, descrição, valor em reais, categoria e data.
- RF8: Um mês sem lançamentos mostra lista vazia com uma mensagem informando isso.

### Resumo financeiro

Mostra a situação do mês de relance.

- RF9: O resumo exibe o total de receitas, o total de despesas e o saldo do mês.
- RF10: O saldo é igual ao total de receitas menos o total de despesas.
- RF11: Um mês sem lançamentos mostra saldo R$ 0,00.
- RF12: Os valores monetários são somados e exibidos sem erros de arredondamento.

### Edição e exclusão

Permite corrigir ou remover registros.

- RF13: O usuário pode editar qualquer campo de um lançamento, com as mesmas validações do cadastro.
- RF14: O usuário pode excluir um lançamento, após confirmar a exclusão.
- RF15: Depois de editar ou excluir, a lista e o resumo do mês são atualizados.

### Persistência, validação e mensagens

- RF16: Os lançamentos permanecem salvos após recarregar a página e após reiniciar a aplicação.
- RF17: As validações acontecem na tela e também na API.
- RF18: A interface exibe mensagens claras de sucesso, validação e erro, em português.

## Critérios de aceitação

- CA-01 (US1, RF1, RF5): Dado um formulário preenchido com dados válidos, quando o usuário salva, então o lançamento aparece na lista do mês da data informada e uma mensagem de sucesso é exibida.
- CA-02 (US1, US7, RF2): Dado um formulário com algum campo obrigatório vazio, quando o usuário tenta salvar, então uma mensagem indica qual campo falta e nenhum lançamento é gravado.
- CA-03 (US7, RF3): Dado um valor igual a zero ou negativo, quando o usuário tenta salvar, então o lançamento é recusado com uma mensagem clara e nada é gravado.
- CA-04 (US2, RF7): Dado um mês com lançamentos, quando o usuário o seleciona, então cada item da lista mostra tipo, descrição, valor em reais, categoria e data.
- CA-05 (US3, RF9, RF10): Dado um mês com receitas somando R$ 3.000,00 e despesas somando R$ 1.200,00, quando o resumo é exibido, então mostra receitas R$ 3.000,00, despesas R$ 1.200,00 e saldo R$ 1.800,00.
- CA-06 (US3, RF12): Dado um mês com duas despesas, uma de R$ 0,10 e outra de R$ 0,20, quando o resumo é exibido, então o total de despesas é exatamente R$ 0,30.
- CA-07 (US4, RF6): Dado que existem lançamentos em meses diferentes, quando o usuário troca o mês, então a lista e o resumo mostram somente os dados do mês escolhido.
- CA-08 (US4, RF8, RF11): Dado um mês sem lançamentos, quando o usuário o seleciona, então a lista aparece vazia com uma mensagem informativa e o saldo é R$ 0,00.
- CA-09 (US8, RF16): Dado que há lançamentos salvos, quando a página é recarregada e a aplicação é reiniciada, então os mesmos lançamentos continuam disponíveis.
- CA-10 (US5, RF13, RF15): Dado um lançamento existente, quando o usuário altera valor, categoria ou data e salva, então a lista e o resumo refletem a alteração e, se a data mudou de mês, o item some do mês anterior e aparece no novo.
- CA-11 (US6, RF14, RF15): Dado um lançamento existente, quando o usuário exclui e confirma, então ele some da lista e o resumo é recalculado; se o usuário cancelar a confirmação, nada é excluído.
- CA-12 (US7, RF17): Dado um envio direto à API com dados inválidos, quando a requisição chega, então a API recusa com uma mensagem de erro e nada é gravado.
- CA-13 (US7, RF18): Dado que a consulta ou a gravação falha, quando o erro acontece, então a interface exibe uma mensagem de erro clara e o que o usuário digitou no formulário é mantido.
- CA-14 (US9): Dado um celular com 360 px de largura, quando o usuário usa a aplicação, então todas as funções ficam acessíveis sem rolagem horizontal; o mesmo vale para telas de computador.
- CA-15 (acessibilidade): Dado o uso apenas com teclado, quando o usuário navega pelo formulário, seletor de mês e lista, então todas as ações podem ser feitas e as mensagens de sucesso e erro são anunciadas para leitores de tela.

## Experiência do usuário

- Perfil: pessoa física, sem conhecimento técnico, usando computador ou celular.
- Necessidades: registrar rápido, ver o saldo do mês de relance e confiar nos números.
- Fluxo principal: abrir a aplicação no mês atual, preencher o formulário, salvar, ver o item na lista e o resumo atualizado.
- Fluxos secundários: trocar o mês, editar um lançamento, excluir um lançamento com confirmação.
- Interface em português, limpa e responsiva, com formulário de lançamento, seletor de mês, resumo financeiro e listagem.
- Valores formatados em reais e datas no formato dia/mês/ano.
- Mensagens de sucesso, validação e erro claras e próximas da ação.
- Acessibilidade: campos com rótulos, mensagens anunciadas para leitores de tela, navegação completa pelo teclado e contraste adequado.

## Restrições técnicas de alto nível

- Aplicação com interface web em React com TypeScript e API em Node.js com TypeScript, como já definido no projeto.
- Os dados ficam em um banco de dados local, sem dependência de serviço externo, e as consultas ao banco são parametrizadas.
- Valores monetários tratados sem erros de arredondamento.
- Validação obrigatória na API, e não só na tela.
- Dados financeiros pessoais não são enviados a terceiros.
- Desempenho: consulta de um mês em até 2 segundos para uso pessoal (premissa de até alguns milhares de lançamentos).

## Fora do escopo

- Autenticação e múltiplos usuários.
- Gráficos e relatórios visuais.
- Importação bancária e qualquer integração externa.
- Categorias personalizadas pelo usuário.
- Lançamentos recorrentes ou parcelados.
- Múltiplas moedas.
- Exportação de dados, anexos e orçamentos ou metas.