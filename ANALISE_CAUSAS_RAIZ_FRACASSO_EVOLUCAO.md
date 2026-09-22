# DIAGNÓSTICO ARQUITETURAL: CAUSAS-RAIZ DA FRAGILIDADE E IMPEDIMENTOS DE EVOLUÇÃO DO MDTXTRT

**Projeto:** MDTXTRT  
**Data:** 2026-09-22  
**Documento:** Análise Sistêmica de Engenharia de Software  
**Objetivo:** Identificar os vícios arquiteturais, acoplamentos estruturais, armadilhas de renderização e deficiências intrínsecas da base de código que provocam o fenômeno de falhas em cadeia e regressões a cada tentativa de evolução.

---

## 1. RESUMO DA ANOMALIA SISTÊMICA

O MDTXTRT apresenta uma arquitetura comumente categorizada em engenharia de software como **"Monólito Frágil Acoplado ao DOM"**. 

Nesse padrão, a interface gráfica, o estado da aplicação, as regras de negócio, o motor de renderização de rich text e a integração com APIs externas não estão desacoplados em módulos independentes. Em vez disso, estão compactados em apenas dois arquivos centrais (`index.html` e `app.js`), operando por meio de mutações imperativas diretas na árvore do DOM.

**O Fenômeno do Fracasso na Evolução:**  
Quando se tenta implementar qualquer melhoria pontual (seja no visual de vidro líquido, no tema claro/escuro, na conversão Markdown ou nos novos recursos da Telegram Bot API 10.3), o código sofre **efeitos colaterais em cascata**. Uma alteração aparentemente inofensiva no CSS ou no HTML quebra suposições implícitas feitas no JavaScript, e a tentativa de corrigir o script quebra o estado do editor, criando um ciclo vicioso de regressão e retrabalho.

---

## 2. OS 7 GARGALOS ESTRUTURAIS DO CÓDIGO QUE IMPEDEM SUA EVOLUÇÃO

---

### GARGALO 1: Incompatibilidade do Modelo de Composição WebKit com a Arquitetura de `<iframe>` de Fundo

* **Localização no Código:**
  * `/index.html`, linhas 87–93 e linhas 264–267:
    ```html
    <div class="wash" aria-hidden="true">
      <iframe class="wash-light" src="backgrounds/light.html"></iframe>
      <iframe class="wash-dark" src="backgrounds/dark.html"></iframe>
    </div>
    ```
  * `/app.js`, linhas 28–31 e 57–58:
    ```javascript
    function setWash(kind, url){
      const el = document.querySelector('iframe.wash-' + kind);
      if(el && url) el.src = url;
    }
    ```

* **Por que o código trava a evolução:**
  1. O motor WebKit (usado no Safari iOS, macOS e nos WebViews embutidos do Telegram para iPhone) trata elementos `<iframe>` como superfícies de renderização isoladas na GPU (`GraphicsLayer`).
  2. A especificação do WebKit impede que a propriedade `-webkit-backdrop-filter: blur(...)` leia ou compute os pixels de um documento filho embutido em um iframe.
  3. **A Armadilha:** Qualquer desenvolvedor ou agente que tente ajustar o efeito de vidro líquido (*liquid glass*) ajustando apenas CSS (opacidade, saturação, sombra) falha obrigatoriamente, porque o efeito visual é nulo em tempo de execução no Safari móvel. Ao tentar remover o iframe, quebra-se a função `setWash` e o ciclo de manifestos em `app.js`, provocando erros de execução.

---

### GARGALO 2: Invalidação de Propriedades CSS por Filtros SVG Inválidos no Safari

* **Localização no Código:**
  * `/index.html`, linhas 106–113 e 268–274:
    ```css
    .frost{
      position: relative;
      background: var(--glass);
      border: 0;
      box-shadow: var(--glass-shadow);
      backdrop-filter: url(#frosted);
      -webkit-backdrop-filter: blur(20px) saturate(var(--glass-saturation,1.15));
    }
    ```
    ```html
    <svg class="glass-surface__filter" aria-hidden="true">
      <filter id="frosted" ...> ... </filter>
    </svg>
    ```

* **Por que o código trava a evolução:**
  1. A sintaxe `backdrop-filter: url(#id)` apontando para um filtro SVG (`<feDisplacementMap>`) não possui suporte estável no Safari e causa a invalidação imediata da regra de estilo pelo parser do WebKit.
  2. Em vez de utilizar o fallback `-webkit-backdrop-filter`, o navegador descarta o bloco de desfoque, resultando em menus completamente opacos ou acinzentados.
  3. **A Armadilha:** A presença desse filtro SVG morto no HTML e no CSS cria a falsa ilusão de que o efeito de distorção de vidro foi implementado, quando na verdade ele é o responsável direto por desligar o motor de desfoque no iOS.

---

### GARGALO 3: Acoplamento Extremo de Seletores e Nomes de Classes Rígidos no JavaScript

* **Localização no Código:**
  * `/app.js`, linhas 1–9, 109–135, 432–448:
    ```javascript
    const sheets = ['#plusMenu','#publishSheet','#headingMenu','#quoteMenu','#importMenu'];
    // ...
    $$('#typebar [data-cmd]').forEach(btn => ...);
    $$('#headingMenu [data-block]').forEach(btn => ...);
    $$('#plusMenu [data-insert], #quoteMenu [data-insert]').forEach(btn => ...);
    ```

* **Por que o código trava a evolução:**
  1. O código possui dezenas de IDs e seletores CSS hardcoded em arrays e funções dispersas.
  2. Não há uma camada intermediária de componentes ou despacho de eventos estruturado.
  3. **A Armadilha:** Se alguém tentar adicionar um novo menu, renomear uma aba, reorganizar o DOM ou migrar um botão da barra para um menu suspenso, a lista estática `sheets` e os listeners globais perdem a referência. Janelas modais deixam de fechar no backdrop, seleções de texto deixam de atualizar o estado visual dos botões (`is-current`, `on`), e o app aparenta estar "congelado".

---

### GARGALO 4: Ausência de Modelo de Estado (State Model) e Dependência do DOM como Fonte da Verdade

* **Localização no Código:**
  * `/app.js`, linhas 10, 153–163, 265–273:
    ```javascript
    let savedRange = null, hist = [], histI = -1, histLock = false, composing = false;
    function pushHist(){
      const html = editor.innerHTML;
      if(hist[histI] === html) return;
      hist.push(html);
      // ...
    }
    function applyHist(html){ histLock = true; editor.innerHTML = html; histLock = false; markDirty(); }
    ```

* **Por que o código trava a evolução:**
  1. O editor não mantém uma árvore de sintaxe abstrata (AST) nem um modelo estruturado de dados em memória. A única fonte da verdade é a string bruta `editor.innerHTML`.
  2. O histórico de Undo/Redo (`hist`) é um array de strings de HTML cru com até 80 posições, substituindo todo o `innerHTML` a cada ação.
  3. **A Armadilha:** Qualquer manipulação de novos nós (ex.: blocos da Telegram API 10.3, citações expansíveis, tabelas, blocos de documento) altera o `innerHTML`. Isso faz com que a restauração de seleção (`savedRange` / `restoreSel()`) aponte para nós destruídos na memória do browser, perdendo o foco do teclado no mobile, colapsando a seleção e quebrando a experiência de digitação do usuário.

---

### GARGALO 5: Dependência de APIs Deprecadas (`document.execCommand`) com Comportamento Divergente Entre Plataformas

* **Localização no Código:**
  * `/app.js`, linhas 177–239:
    ```javascript
    function exec(cmd, value=null){
      restoreSel(); expandWord();
      document.execCommand(cmd, false, value);
      saveSel(); pushHist(); markDirty();
    }
    function formatBlock(tag){
      // ...
      document.execCommand('formatBlock', false, nextTag);
      // ...
    }
    ```

* **Por que o código trava a evolução:**
  1. `document.execCommand` foi declarado obsoleto pelos órgãos de padronização web (W3C/WHATWG) e sua execução varia drasticamente entre navegadores:
     * O Chrome insere tags semânticas `<p>`, `<h1>`.
     * O WebKit/Safari insere tags proprietárias `<div>`, `<span>` com atributos de estilo inline (`style="font-weight: bold;"`).
  2. Funções auxiliares no código como `unwrapBold()`, `currentKind()` e `replace()` foram escritas como "gambiarras" para tentar limpar o lixo gerado pelo `execCommand`.
  3. **A Armadilha:** Tentar estender o editor para novos formatos sem abandonar o `execCommand` faz com que o código entre em conflito direto com as heurísticas de limpeza, gerando tags aninhadas inválidas (`<p><p><h6>...</h6></p></p>`).

---

### GARGALO 6: Parsers e Serializadores de Markdown / HTML Baseados em Expressões Regulares Frágeis

* **Localização no Código:**
  * `/app.js`, linhas 281–326 e 353–386:
    ```javascript
    function parseInlineMD(s){
      return escapeHTML(s)
        .replace(/`([^`]+)`/g, '<code>$1</code>')
        .replace(/\[([^\]]+)\]\(([^)]+)\)/g, ...)
        .replace(/\*\*([^*]+)\*\*/g, '<strong>$1</strong>')
        .replace(/\*([^*]+)\*/g, '<em>$1</em>')
        .replace(/~~([^~]+)~~/g, '<s>$1</s>');
    }
    ```

* **Por que o código trava a evolução:**
  1. A conversão de Markdown e HTML não utiliza um parser lexico/sintático real. Ela depende de substituições sequenciais por Expressões Regulares (`RegExp`).
  2. Expressões regulares lineares são incapazes de lidar com elementos aninhados (ex.: um link contendo negrito `[**texto**](url)`, código dentro de citação, ou múltiplos asteriscos consecutivos).
  3. **A Armadilha:** Ao tentar implementar novas tags da Bot API 10.3 (como `<blockquote expandable>`, rodapés `<footer>` e tabelas), a serialização para Markdown em `htmlToMarkdown` e a desserialização em `mdToBasicHTML` corrompem o texto, deletando atributos customizados ou escapando entidades indevidamente.

---

### GARGALO 7: Ciclos Concorrentes de Tema e Injeção Caótica de Variáveis CSS

* **Localização no Código:**
  * `/index.html`, linhas 19, 21–76:
    ```html
    <script>(function(){try{var t=window.Telegram&&window.Telegram.WebApp; ...document.documentElement.classList.toggle("light",l);...}})();</script>
    ```
  * `/app.js`, linhas 61–73:
    ```javascript
    function applyScheme(){
      const tg = window.Telegram?.WebApp;
      const inTg = !!(tg && ((tg.initData && tg.initData.length) || tg.initDataUnsafe?.user?.id));
      const light = inTg && tg.colorScheme ? tg.colorScheme === 'light' : window.matchMedia('(prefers-color-scheme: light)').matches;
      document.documentElement.classList.toggle('light', light);
      document.documentElement.classList.toggle('dark', !light);
    }
    ```

* **Por que o código trava a evolução:**
  1. Existem múltiplos pontos no ciclo de vida da aplicação concorrendo para manipular as classes `.light` e `.dark` no elemento `<html>`.
  2. O CSS possui declarações duplicadas e sobrepostas em `:root`, `@media (prefers-color-scheme: light)`, `html.light` e `html.dark`.
  3. Foram adicionadas diversas variáveis estéticas (`--surface-glass-default`, `--elevation-3`) que não possuem nenhum consumidor real nas regras CSS das classes `.sheet`, `#typebar` ou `.seg`.
  4. **A Armadilha:** Ao tentar alterar as cores de um tema, o desenvolvedor altera uma variável em `:root`, mas ela é imediatamente sobrescrita por `html.dark` ou ignorada porque o seletor visual utiliza cores literais com `color-mix()` em vez da variável declarada. O resultado é a percepção de que "nada muda" no layout.

---

## 3. QUADRO COMPARATIVO: TENTATIVA DE EVOLUÇÃO vs. RESPOSTA DA ARQUITETURA

| O que se tenta evoluir | Onde o código reage negativamente | Consequência / Regressão Gerada |
| :--- | :--- | :--- |
| **Melhorar o vidro líquido no Safari** | A camada do `iframe` e o filtro `url(#frosted)` bloqueiam a renderização na GPU. | O Safari não processa o desfoque; menus permanecem opacos. |
| **Remover os IFrames de fundo** | A função `setWash()` em `app.js` tenta acessar `iframe.wash-*` e falha silenciosamente. | Quebra a troca de temas dinâmicos e manifestos de assets. |
| **Adicionar novas tags da API 10.3** | O `execCommand` e os parsers Regex de Markdown não reconhecem nós customizados. | O parser deleta os atributos ou o Telegram rejeita com erro 400. |
| **Ajustar cores do tema claro/escuro** | Múltiplas regras CSS concorrentes e scripts inline sobrepõem valores no `:root`. | O tema pisca com cores erradas ou congela no modo escuro padrão. |
| **Melhorar botões e atalhos na barra** | A lista estática `sheets` e os listeners globais de clique perdem a sincronia de nós. | Botões não abrem menus ou não fecham ao clicar fora. |

---

## 4. DIRETRIZES DE ENGENHARIA PARA DESTRAVAR O CÓDIGO

Para que este aplicativo possa evoluir com segurança e estabilidade, as seguintes intervenções estruturais são mandatórias:

1. **Eliminação do IFrame de Fundo:** Substituir definitivamente os iframes por contêineres nativos no DOM (`<div class="wash-bg">`), adaptando `setWash` no `app.js`.
2. **Purificação do CSS de Vidro:** Remover o filtro SVG `#frosted` e padronizar `-webkit-backdrop-filter: blur(28px) saturate(1.8)` diretamente nas classes de superfície translúcida.
3. **Desacoplamento do Gerenciamento de Janelas (`sheets`):** Substituir o array fixo de IDs por seletores declarativos com `data-sheet` e delegação de eventos genérica.
4. **Parser Estruturado para Markdown e API 10.3:** Tratar blocos ricos como árvores de nós semânticas, garantindo preservação de atributos como `expandable` e `tg-footer`.
5. **Unificação da Cascata de Temas:** Centralizar as variáveis em uma única tabela de tokens de design, eliminando código CSS morto e variáveis não consumidas.

---
*Documento registrado no repositório para referência técnica permanente.*
