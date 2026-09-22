# ROTEIRO TÉCNICO DE NEUTRALIZAÇÃO ARQUITETURAL E EVOLUÇÃO ESTÁVEL DO MDTXTRT

**Projeto:** MDTXTRT  
**Data:** 2026-09-22  
**Documento:** Plano de Engenharia para Neutralização dos Gargalos e Implementação Segura  
**Finalidade:** Estabelecer uma solução definitiva, atômica e estruturada para eliminar os 7 gargalos identificados, garantindo evolução contínua sem regressões visuais, de performance ou de compatibilidade no Safari (iOS) e no Telegram WebApp (Bot API 10.3).

---

## 1. PRINCÍPIOS DA SOLUÇÃO ARQUITETURAL

Para neutralizar de forma definitiva os fenômenos de falha recente, a solução deve obedecer a 4 premissas invioláveis:

1. **Unificação da Árvore de Composição da GPU:** O efeito de vidro líquido (*liquid glass*) só é computado pelo WebKit/Safari se o fundo e os elementos com `backdrop-filter` residirem no mesmo documento DOM. Nenhum `iframe` de fundo pode ser mantido.
2. **Desacoplamento Declarativo de Interface:** O JavaScript não deve depender de listas estáticas de IDs hardcoded. O controle de menus suspensos, botões de ação e atalhos deve operar via atributos de dados declarativos (`data-sheet`, `data-cmd`, `data-block`).
3. **Serialização Semântica baseada em AST/DOM (Sem Regex Frágeis):** A conversão entre o editor visual, Markdown e as estruturas da Telegram Bot API 10.3 deve processar nós DOM reais em vez de substituições lineares por expressões regulares.
4. **Ciclo de Vida de Tema Determinístico:** Eliminar scripts concorrentes e centralizar a alternância de cores em um único despachante reativo integrado aos eventos `Telegram.WebApp.onEvent('themeChanged')` e `matchMedia`.

---

## 2. ROTEIRO DE EXECUÇÃO EM 4 ETAPAS ATÔMICAS

```
┌─────────────────────────────────────────────────────────────────────────┐
│ ETAPA 1: Neutralização da Camada de Renderização & Fundo DOM            │
│ (Remoção de IFrames, Purificação do CSS de Vidro, Limpeza de :root)     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ETAPA 2: Neutralização do Acoplamento de Scripts & Gestão de Menus      │
│ (Refatoração de setWash, Delegação Declarativa de Sheets e Modais)      │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ETAPA 3: Neutralização do Motor de Edição & Serialização API 10.3       │
│ (Manipulação Segura de Blocos, Preservação de Range, Parser Semântico)  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ ETAPA 4: Validação Atômica, Matriz de Testes & Critérios de Aceite     │
│ (Testes no Safari iOS, Telegram WebView, Undo/Redo e Rich Message)      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. ESPECIFICAÇÃO TÉCNICA DETALHADA DE CADA ETAPA

---

### ETAPA 1: Neutralização da Camada de Renderização e Vidro Líquido

#### 1.1 Eliminação Definitiva dos IFrames de Fundo
* **Ação:** Substituir os elementos `<iframe>` em `/index.html` por contêineres `<div>` nativos de aceleração gráfica direta.
* **Implementação no HTML:**
  ```html
  <div class="wash" aria-hidden="true">
    <div class="wash-bg wash-light"></div>
    <div class="wash-bg wash-dark"></div>
  </div>
  ```
* **Implementação no CSS:**
  ```css
  .wash {
    position: fixed;
    inset: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
    pointer-events: none;
    overflow: hidden;
  }
  .wash-bg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    background-size: cover;
    transition: opacity 0.25s ease-out;
  }
  .wash-light {
    opacity: 0;
    background: #f8fbff;
    background-image:
      radial-gradient(92% 30% at -10% 0%, rgba(246,236,255,0.72) 0%, rgba(248,243,255,0.42) 44%, rgba(250,251,255,0) 78%),
      radial-gradient(84% 30% at 112% 19%, rgba(255,237,245,0.78) 0%, rgba(255,247,250,0.42) 47%, rgba(250,252,255,0) 80%),
      radial-gradient(100% 38% at -18% 40%, rgba(230,255,248,0.88) 0%, rgba(239,255,251,0.56) 39%, rgba(248,253,255,0.18) 63%, rgba(250,252,255,0) 84%),
      radial-gradient(96% 40% at 56% 49%, rgba(231,246,255,0.88) 0%, rgba(240,249,255,0.58) 46%, rgba(248,252,255,0.18) 69%, rgba(250,252,255,0) 86%),
      radial-gradient(82% 31% at 112% 72%, rgba(247,244,255,0.52) 0%, rgba(250,248,255,0.30) 49%, rgba(250,252,255,0) 82%),
      radial-gradient(76% 28% at 108% 100%, rgba(255,247,242,0.70) 0%, rgba(255,251,248,0.38) 48%, rgba(250,252,255,0) 81%),
      radial-gradient(42% 16% at 57% 98%, rgba(229,252,255,0.46) 0%, rgba(242,253,255,0.20) 54%, rgba(250,252,255,0) 82%),
      linear-gradient(180deg, #faf8fe 0%, #f7fbff 28%, #f4fbff 52%, #f8fbff 78%, #fbfaff 100%);
  }
  .wash-dark {
    opacity: 1;
    background: #12131c;
    background-image:
      radial-gradient(95% 32% at -12% 0%, rgba(48,27,51,0.42) 0%, rgba(34,24,42,0.24) 38%, rgba(18,20,29,0) 76%),
      radial-gradient(86% 34% at 112% 17%, rgba(61,34,49,0.42) 0%, rgba(43,27,41,0.24) 42%, rgba(18,20,29,0) 78%),
      radial-gradient(104% 42% at -23% 39%, rgba(19,72,66,0.52) 0%, rgba(18,52,52,0.34) 36%, rgba(17,31,39,0.14) 62%, rgba(17,21,30,0) 83%),
      radial-gradient(90% 33% at 114% 45%, rgba(48,31,43,0.29) 0%, rgba(33,27,39,0.16) 48%, rgba(17,21,30,0) 80%),
      radial-gradient(94% 38% at -20% 78%, rgba(26,52,77,0.43) 0%, rgba(21,39,58,0.27) 42%, rgba(17,23,34,0.11) 66%, rgba(17,20,29,0) 86%),
      radial-gradient(84% 30% at 111% 91%, rgba(46,31,43,0.26) 0%, rgba(33,27,39,0.14) 49%, rgba(18,20,29,0) 82%),
      linear-gradient(180deg, #12131c 0%, #111720 31%, #10171d 50%, #10151d 69%, #11131c 100%);
  }
  html.light .wash-light { opacity: 1; }
  html.light .wash-dark { opacity: 0; }
  html.dark .wash-light { opacity: 0; }
  html.dark .wash-dark { opacity: 1; }
  ```

#### 1.2 Purificação do Efeito Frosted Glass no Safari
* **Ação:** Remover o nó `<svg class="glass-surface__filter">` e as referências `backdrop-filter: url(#frosted)`.
* **Implementação no CSS:**
  ```css
  .frost {
    position: relative;
    background: var(--glass);
    border: 0;
    box-shadow: var(--glass-shadow);
    -webkit-backdrop-filter: blur(28px) saturate(var(--glass-saturation, 1.25));
    backdrop-filter: blur(28px) saturate(var(--glass-saturation, 1.25));
  }
  .sheet {
    -webkit-backdrop-filter: blur(32px) saturate(1.4);
    backdrop-filter: blur(32px) saturate(1.4);
  }
  ```

#### 1.3 Limpeza de Variáveis Fantasma no CSS Global
* **Ação:** Deletar variáveis sem uso em `:root` (`--surface-glass-*`, `--elevation-*`) e manter apenas as variáveis ativas com fallback seguro.

---

### ETAPA 2: Neutralização do Acoplamento de Scripts e Gestão de Menus

#### 2.1 Refatoração Polimórfica de `setWash` em `app.js`
* **Ação:** Permitir que `setWash` atualize nós `<div>` com background-image sem quebrar chamadas legadas.
* **Implementação no JS:**
  ```javascript
  function setWash(kind, url){
    const el = document.querySelector('.wash-' + kind);
    if(!el || !url) return;
    if(el.tagName === 'IFRAME') {
      el.src = url;
    } else {
      el.style.backgroundImage = 'url("' + url + '")';
    }
  }
  ```

#### 2.2 Desacoplamento Declarativo de Menus e Modais
* **Ação:** Substituir o array fixo `sheets = ['#plusMenu', ...]` por um seletor dinâmico `.sheet[data-sheet]`.
* **Implementação no JS:**
  ```javascript
  function getAllSheets() {
    return Array.from(document.querySelectorAll('.sheet'));
  }
  function closePanels() {
    getAllSheets().forEach(el => el.classList.remove('on', 'is-top'));
    backdrop.classList.remove('on');
    document.dispatchEvent(new Event('selectionchange'));
  }
  function openPanel(sel, anchor) {
    const panel = typeof sel === 'string' ? $(sel) : sel;
    if(!panel) return;
    const ref = anchor || document.activeElement;
    const rect = ref?.getBoundingClientRect?.();
    getAllSheets().forEach(el => el.classList.remove('on', 'is-top'));
    
    const placeTop = panel.id === 'importMenu' || panel.id === 'publishSheet';
    panel.classList.toggle('is-top', placeTop);
    if(placeTop && rect){
      panel.style.setProperty('--sheet-top', Math.round(rect.bottom + 8) + 'px');
    } else {
      panel.style.removeProperty('--sheet-top');
    }
    
    panel.style.visibility = 'hidden';
    panel.classList.add('on');
    const width = Math.min(panel.offsetWidth || 280, innerWidth - 28);
    const center = rect ? rect.left + rect.width / 2 : innerWidth / 2;
    const left = Math.max(14, Math.min(innerWidth - width - 14, center - width / 2));
    panel.style.setProperty('--sheet-left', left + 'px');
    panel.style.setProperty('--sheet-origin', Math.max(20, Math.min(width - 20, center - left)) + 'px');
    panel.style.visibility = '';
    backdrop.classList.add('on');
    document.dispatchEvent(new Event('selectionchange'));
  }
  ```

#### 2.3 Unificação do Despachante de Tema
* **Ação:** Garantir sincronia bidirecional entre `Telegram.WebApp.colorScheme`, sistema operacional e CSS.
* **Implementação no JS:**
  ```javascript
  function applyScheme(){
    const tg = window.Telegram?.WebApp;
    const inTg = !!(tg && ((tg.initData && tg.initData.length) || tg.initDataUnsafe?.user?.id));
    const isLight = inTg && tg.colorScheme ? tg.colorScheme === 'light' : window.matchMedia('(prefers-color-scheme: light)').matches;
    
    document.documentElement.classList.toggle('light', isLight);
    document.documentElement.classList.toggle('dark', !isLight);
    
    if(inTg){
      const headerColor = isLight ? '#f8fbff' : '#12131c';
      tg?.setHeaderColor?.(headerColor);
      tg?.setBackgroundColor?.(headerColor);
    }
  }
  ```

---

### ETAPA 3: Neutralização do Motor de Edição e Compatibilidade com Telegram Bot API 10.3

#### 3.1 Manipulação Segura de Blocos Sem Perda de Seleção
* **Ação:** Isolar a criação de blocos especiais (citação expansível, footer, blocos de código) em construtores semânticos com preservação de cursor.
* **Suporte Completo às Novas Entidades 10.3:**
  * **Citação Expansível:** `<blockquote data-expandable="true">` serializada como `<blockquote expandable>` para a API de Rich Messages do Telegram.
  * **Rodapé Semântico:** `<footer class="tg-footer">` serializado como `<aside>` para Telegraph e `<footer>` para Rich Messages.
  * **Tabelas e Blocos Estruturados:** Serialização tabular limpa sem caracteres de controle corrompidos.

#### 3.2 Parser DOM Recursivo (Substituição Definitiva de Regex Lineares)
* **Ação:** Refatorar `toRichHTML(root)` e `htmlToMarkdown(html)` para percorrer a árvore DOM por meio de um `TreeWalker` ou recursão estruturada de nós (`Node.ELEMENT_NODE` e `Node.TEXT_NODE`).
* **Implementação no JS:**
  ```javascript
  function toRichHTML(root){
    const walkInline = node => {
      if(node.nodeType === 3) return escapeHTML(node.textContent || '');
      if(node.nodeType !== 1) return '';
      const tag = node.tagName.toLowerCase();
      const content = Array.from(node.childNodes).map(walkInline).join('');
      
      switch(tag){
        case 'strong': case 'b': return '<b>' + content + '</b>';
        case 'em': case 'i': return '<i>' + content + '</i>';
        case 'u': return '<u>' + content + '</u>';
        case 's': case 'del': return '<s>' + content + '</s>';
        case 'code': return '<code>' + content + '</code>';
        case 'a': return '<a href="' + escapeHTML(node.getAttribute('href') || '') + '">' + content + '</a>';
        case 'br': return '<br/>';
        default: return content;
      }
    };

    const blocks = [];
    Array.from(root.childNodes).forEach(node => {
      if(node.nodeType === 3){
        const txt = (node.textContent || '').trim();
        if(txt) blocks.push('<p>' + escapeHTML(txt) + '</p>');
        return;
      }
      if(node.nodeType !== 1) return;
      
      const tag = node.tagName.toLowerCase();
      const inlineContent = Array.from(node.childNodes).map(walkInline).join('');
      
      if(/^h[1-6]$/.test(tag)){
        blocks.push('<' + tag + '>' + inlineContent + '</' + tag + '>');
      } else if(tag === 'footer' || node.classList.contains('tg-footer')){
        blocks.push('<footer>' + inlineContent + '</footer>');
      } else if(tag === 'blockquote'){
        const isExp = node.getAttribute('data-expandable') === 'true' || node.hasAttribute('expandable');
        blocks.push(isExp ? '<blockquote expandable>' + inlineContent + '</blockquote>' : '<blockquote>' + inlineContent + '</blockquote>');
      } else if(tag === 'pre'){
        blocks.push('<pre>' + escapeHTML(node.textContent || '') + '</pre>');
      } else if(tag === 'ul' || tag === 'ol'){
        const items = Array.from(node.children).map(li => '<li>' + walkInline(li) + '</li>').join('');
        blocks.push('<' + tag + '>' + items + '</' + tag + '>');
      } else {
        blocks.push('<p>' + inlineContent + '</p>');
      }
    });

    return blocks.join('') || '<p></p>';
  }
  ```

---

## 4. MATRIZ DE TESTES E CRITÉRIOS DE ACEITE

| Teste | Procedimento | Critério de Aceite Inegociável |
| :--- | :--- | :--- |
| **1. Desfoque de Vidro no Safari (iOS)** | Abrir o app no Safari mobile ou Telegram iOS e acionar `#typebar` ou `#plusBtn`. | O menu flutuante deve desfocar claramente as cores do fundo com efeito translúcido (sem cor sólida ou cinza). |
| **2. Alternância de Temas** | Alternar o tema no Telegram ou no SO (Dark <-> Light). | Transição suave de cores sem flash branco e sem perda do estado de contraste. |
| **3. Ciclo de Vida dos Modais** | Clicar nos botões da barra e depois no backdrop. | Apenas o menu acionado se abre; clique no fundo fecha o painel instantaneamente. |
| **4. Integridade de Undo/Redo** | Digitar texto, aplicar cabeçalho `H2`, citação expansível e pressionar `Desfazer`/`Refazer`. | O texto e a seleção voltam ao estado exato sem quebra de tags nem salto de foco do teclado. |
| **5. Carga Útil da Bot API 10.3** | Clicar em "Publicar via Telegram" com citações expansíveis e rodapés. | O payload gerado em `buildRich()` contém `<blockquote expandable>` e `<footer>` válidos e conformes. |

---

## 5. CHECKLIST DE IMPLEMENTAÇÃO SEQUENCIAL

- [ ] **Passo 1:** Modificar `/index.html` substituindo iframes por divs de aceleração gráfica direta.
- [ ] **Passo 2:** Remover o nó SVG `#frosted` e atualizar regras `.frost` e `.sheet` com `-webkit-backdrop-filter` nativo.
- [ ] **Passo 3:** Atualizar `setWash()` e o despachante `applyScheme()` em `/app.js`.
- [ ] **Passo 4:** Substituir a lista estática de sheets em `app.js` pela função declarativa `getAllSheets()`.
- [ ] **Passo 5:** Integrar o parser recursivo `toRichHTML()` com suporte total a nós da Telegram API 10.3.
- [ ] **Passo 6:** Executar compilação, reiniciar o servidor de desenvolvimento e validar na matriz de testes.

---
*Roteiro técnico aprovado para execução atômica.*
