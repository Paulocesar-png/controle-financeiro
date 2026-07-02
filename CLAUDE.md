# CLAUDE.md

Este arquivo fornece orientações para o Claude Code (claude.ai/code) ao trabalhar com o código deste repositório.

## Idioma

Responda sempre em português (pt-BR), independentemente do idioma usado na pergunta.

## Visão geral do projeto

Controle Financeiro é uma PWA (Progressive Web App) instalável, de página única, em português, para controle pessoal de gastos e proventos. Não há sistema de build, gerenciador de pacotes nem suíte de testes — a aplicação inteira é um único arquivo HTML estático, mais um manifest e ícones.

## Executando a aplicação

Não existe comando de build/dev/test. Abra [index.html](index.html) diretamente no navegador, ou sirva o diretório como estático (ex.: `npx serve .`) caso precise que os recursos de PWA (service worker/manifest) funcionem corretamente via `http://`. Qualquer alteração é testada recarregando a página no navegador e usando a interface manualmente — não há executor de testes automatizado.

## Arquitetura

Tudo vive em [index.html](index.html): `<style>` inline para o CSS, `<script>` inline para o JS, sem módulos/imports, sem dependências JS externas. O único recurso externo é um `@import` de fontes do Google Fonts.

**Modelo de dados**: um único array em memória `transactions`, persistido no IndexedDB do navegador (banco `ControleFinanceiroDB`, object store `transactions`, com chave `id`). `saveTransactions()` limpa todo o store e regrava todas as transações a cada salvamento — não há atualização incremental. Um objeto de transação tem este formato:

```js
{
  id,            // Date.now() (ou Date.now() + Math.random() para lotes replicados)
  type,          // 'income' | 'expense'
  description,
  amount,        // valor total atual (valor base + ajustes aplicados)
  date,          // 'YYYY-MM-DD'
  adjustments,   // [{ type: 'add'|'remove', amount, description, date, recurring }]
  paid,          // bool, status de "pago" (somente para gastos)
  customOrder,   // { [monthKey]: sortIndex } — ordenação manual (drag) por mês
}
```

**Visualização por mês**: a interface sempre exibe um mês por vez (globais `currentMonth`/`currentYear`, navegados pelas setas do cabeçalho). `getMonthKey(month, year)` → `'YYYY-MM'` é a chave usada em toda parte para filtragem (`getCurrentMonthTransactions()`) e no mapa `customOrder` por mês. As datas são interpretadas quebrando a string `YYYY-MM-DD` diretamente (e não via `new Date()`) especificamente para evitar problemas de fuso horário — preserve esse padrão ao mexer em lógica de datas.

**Ajustes, não edições**: o `amount` de um gasto/provento nunca é editado livremente em cima do valor existente. Em vez disso, `openModal(type, transactionId)` em modo de edição registra um `adjustment` (adicionar/remover, com descrição própria, timestamp e flag opcional `recurring`) e o `amount` da transação é incrementado/decrementado de acordo. O histórico completo de ajustes é mantido e pode ser editado/excluído individualmente (`editAdjustment`, `deleteAdjustment`), cada operação revertendo e reaplicando o delta em `transaction.amount`. Sobrescrever o total diretamente via "✏️ Editar" (`saveNewAmount`) é um caminho separado que não passa pelo array de ajustes.

**Replicar mês anterior** (`replicatePreviousMonth`): copia todas as transações do mês anterior para o mês atual, preservando a ordem customizada, reiniciando `paid` para false, atribuindo novos ids/datas, e levando adiante apenas os ajustes marcados como `recurring: true` (o valor base é recalculado revertendo antes todos os ajustes não recorrentes).

**Reordenação**: o drag-and-drop é implementado manualmente de duas formas — eventos HTML5 DnD para desktop (`dragstart`/`dragover`/`drop`) e handlers de touch manuais para mobile (`touchstart`/`touchmove`/`touchend`, com auto-scroll perto das bordas da tela). Ambos convergem para `reorderTransactions(fromIndex, toIndex)`, que reordena apenas dentro do mês atual e grava o resultado em `customOrder[monthKey]` de cada transação.

**Renderização**: sem framework/virtual DOM — `updateDisplay()` recalcula os totais do mês e faz um re-render completo via `innerHTML` de `#transactionsList` a cada mudança de estado, religando em seguida os listeners de drag/touch (`setupDragAndDrop()`). Não há diffing, então qualquer novo elemento interativo adicionado ao template da linha de transação precisa ter seu tratamento de evento religado dentro dessa função.

**Backup/restauração**: `exportData()`/`importData()` serializam/desserializam o array `transactions` completo como JSON, via download de arquivo e um `<input type="file">` oculto — esse é o único mecanismo de backup (sem servidor, sem sincronização em nuvem).

## Convenções

- Todas as strings voltadas ao usuário são em português (pt-BR); a moeda é formatada via `formatCurrency()` usando `toLocaleString('pt-BR', ...)`, prefixada com `R$`.
- Os campos de valor monetário são `type="text"` (não `type="number"`) e normalizados por `formatCurrencyInput()`/`parseInputAmount()` para aceitar vírgula e ponto como separador decimal — não troque esses campos de volta para inputs numéricos nativos.
- Os valores monetários são armazenados como float do JS (sem representação em centavos inteiros); tome cuidado com arredondamento de ponto flutuante ao fazer aritmética entre ajustes.
