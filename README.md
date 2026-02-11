# Bitcoin RSI Chart — Documentação Completa para LLMs

## 📋 Visão Geral do Projeto

Este projeto é um **dashboard interativo de análise técnica do Bitcoin** implementado como um **único arquivo HTML autocontido** (`btc_chart.html`). Ele exibe:

1. **Gráfico de Preço do BTC** (escala logarítmica, cor laranja `#f57c00`)
2. **Gráfico de RSI (Relative Strength Index)** (cor verde `#00c850`)
3. **Tabela de Eventos RSI < 37** — mostra retornos do BTC em diferentes horizontes de tempo após cada evento onde o RSI semanal caiu abaixo de 37
4. **Engine de Desenho** — ferramentas de anotação (linhas, retângulos, setas, texto, etc.) sobre os gráficos
5. **Exportação de imagem** — botão "Export Image" que gera PNG com charts + tabela + desenhos

**Tecnologias:** HTML/CSS/JavaScript puro, Chart.js v4.4.1 (via CDN), chartjs-adapter-date-fns v3.0.0 (via CDN).

---

## 🏗️ Arquitetura do Arquivo

O arquivo `btc_chart.html` contém **tudo inline** (sem arquivos externos além dos CDNs):

```
btc_chart.html
├── <head>
│   ├── Chart.js CDN
│   ├── chartjs-adapter-date-fns CDN
│   └── <style> — Todo o CSS inline
├── <body>
│   ├── Sidebar (ferramentas de desenho)
│   ├── Layers Panel (camadas dos desenhos)
│   ├── Main Content
│   │   ├── Botão Export Image
│   │   ├── Chart Wrapper (1350×850px)
│   │   │   ├── Canvas #btcChart (550px altura)
│   │   │   ├── Canvas #rsiChart (300px altura)
│   │   │   └── Canvas #drawingOverlay (overlay de desenhos)
│   │   └── RSI Events Panel (tabela)
│   └── <script>
│       ├── Drawing Engine (drawState, setTool, etc.)
│       ├── CSV Data (btcCsvData, lwbcCsvData)
│       ├── Parse Data
│       ├── Chart 1: BTC Close (chart1)
│       ├── Chart 2: RSI (chart2)
│       ├── RSI Events Logic (cálculo de retornos)
│       ├── RSI Event Plugins (marcadores nos gráficos)
│       └── Export Image Function
```

---

## 📊 Dados CSV Embutidos

### Dataset 1: `btcCsvData` (BTC + RSI)

Os dados estão embutidos como uma **string CSV literal** dentro do JavaScript (variável `btcCsvData`). Formato:

```csv
time,close,RSI
1313366400,10,
1314576000,7.4,
...
1330300800,5,34.662576687116555
```

| Coluna  | Tipo     | Descrição                                              |
|---------|----------|--------------------------------------------------------|
| `time`  | Integer  | Unix timestamp em **segundos** (epoch)                |
| `close` | Float    | Preço de fechamento semanal do BTC em USD             |
| `RSI`   | Float    | RSI semanal (14 períodos). Vazio nas primeiras linhas |

**Localização no código:** Procure pela string `const btcCsvData = \`` — ela começa aproximadamente na **linha 920** e vai até a **linha 1170** do arquivo.

**Para adicionar novos dados de preço/RSI:**
1. Localize `const btcCsvData = \`` no `<script>`
2. Adicione novas linhas **antes do fechamento** `` \`; `` da template string
3. Formato de cada linha: `UNIX_TIMESTAMP_SECONDS,PRECO_USD,RSI_VALUE`
4. O timestamp deve ser em **segundos** (não milissegundos). Para converter uma data: `Math.floor(new Date('2026-01-01').getTime() / 1000)`
5. Se o RSI ainda não estiver calculado, deixe o campo vazio: `1769990400,69131,`

**Exemplo de adição:**
```javascript
// Antes (última linha):
1769990400,69131,34.97302186203588`;

// Depois:
1769990400,69131,34.97302186203588
1771200000,72500,38.5`;
```

### Dataset 2: `lwbcCsvData` (BTC + LWBC)

Segundo dataset embutido com o indicador LWBC (Long-term Weighted Bitcoin Cycles). Formato:

```csv
time,close,LWBC
1312156800,8,
...
1349049600,11,2041.2108608219885
```

| Coluna | Tipo    | Descrição                                              |
|--------|---------|--------------------------------------------------------|
| `time` | Integer | Unix timestamp em **segundos**                        |
| `close`| Float   | Preço de fechamento semanal do BTC em USD             |
| `LWBC` | Float   | Indicador LWBC. Vazio nas primeiras linhas             |

**Localização no código:** Procure pela string `const lwbcCsvData = \`` — logo após o fim de `btcCsvData`.

**Para adicionar novos dados LWBC:**
- Mesmo processo que `btcCsvData`, adicionando linhas antes do `` \`; `` de fechamento.

---

## 🔧 Parsing dos Dados

O parsing é feito logo após as strings CSV:

```javascript
const btcLines = btcCsvData.trim().split('\n');
const btcData = btcLines.slice(1).map(line => {
    const parts = line.split(',');
    return {
        time: new Date(parseInt(parts[0]) * 1000),  // Converte seconds → milliseconds
        close: parseFloat(parts[1]),
        rsi: parts[2] ? parseFloat(parts[2]) : null
    };
});
```

**Filtragem:** Os dados são filtrados a partir de **abril de 2014**:
```javascript
const april2014 = new Date(2014, 3, 1);
const filteredBtcData = btcData.filter(d => d.time >= april2014);
```

**Para alterar a data de início do gráfico:** Modifique `new Date(2014, 3, 1)` — o segundo parâmetro é o mês (0-indexed, então 3 = abril).

**Range do eixo X:** O eixo X vai de `timeMin` (primeiro dado filtrado) até `timeMax` (último dado + 60 dias de margem):
```javascript
const timeMax = new Date(rawTimeMax.getTime() + 60 * 24 * 60 * 60 * 1000);
```

---

## 📈 Chart 1: Preço do BTC

**Variável:** `chart1`  
**Canvas ID:** `#btcChart`  
**Tipo:** `line`  
**Altura do painel:** `550px`

### Configurações importantes:

| Propriedade                    | Valor                          | Como alterar                                               |
|-------------------------------|--------------------------------|-----------------------------------------------------------|
| Cor da linha                  | `#f57c00` (laranja)            | Altere `borderColor` no dataset                           |
| Cor do gradiente              | `rgba(245, 124, 0, ...)`      | Altere `gradientBtc` (3 stops: 0.45 → 0.15 → 0)         |
| Espessura da linha            | `1.4`                          | Altere `borderWidth` no dataset                           |
| Escala Y                      | `logarithmic`                  | Altere `type` em `scales.y`                               |
| Ticks Y (valores exibidos)    | `[200,500,1000,...,100000]`    | Altere o array no `callback` de `scales.y.ticks`          |
| Posição do eixo Y             | `right`                        | Altere `position` em `scales.y`                           |
| Largura do eixo Y             | `75px`                         | Altere em `afterFit(axis) { axis.width = 75; }`          |
| Legenda customizada           | "BTCUSD"                       | Altere o texto no plugin `btcLegend`                      |
| Tooltip                       | Formato: `$XX,XXX`            | Altere o `callbacks.label` no plugin tooltip              |

### Para alterar o label do gráfico:
Procure pelo plugin inline `btcLegend` dentro de `chart1`:
```javascript
ctx.fillText('BTCUSD', x + 10, y);
```

### Para alterar os ticks do eixo Y:
```javascript
callback: v => [200,500,1000,3000,5000,10000,20000,30000,50000,100000].includes(v)
    ? '$' + v.toLocaleString() : ''
```
Adicione ou remova valores desse array.

---

## 📉 Chart 2: RSI

**Variável:** `chart2`  
**Canvas ID:** `#rsiChart`  
**Tipo:** `line`  
**Altura do painel:** `300px`

### Configurações importantes:

| Propriedade                    | Valor                          | Como alterar                                               |
|-------------------------------|--------------------------------|-----------------------------------------------------------|
| Cor da linha                  | `#00c850` (verde)              | Altere `borderColor` no dataset                           |
| Cor do gradiente              | `rgba(0, 200, 80, ...)`       | Altere `gradientRsi` (3 stops: 0.35 → 0.10 → 0)         |
| Escala Y min/max              | `20` / `100`                   | Altere `min` e `max` em `scales.y`                        |
| Linha de threshold (37)       | Dashed verde                   | Plugin `rsiThresholdLine`                                 |
| Legenda customizada           | "RSI"                          | Plugin `rsiLegend`                                        |

### Para alterar o threshold do RSI:
1. **Linha no gráfico:** No plugin `rsiThresholdLine`, altere `const py = y.getPixelForValue(37);` e `ctx.fillText('37', ...)`.
2. **Lógica dos eventos:** Altere a constante `const RSI_THRESHOLD = 37;` (isso afeta os eventos na tabela e as marcações nos gráficos).
3. **Título da tabela:** O título é gerado dinâmicamente usando `RSI_THRESHOLD`.

---

## 📋 Tabela de Eventos RSI

### Lógica de detecção de eventos:

```javascript
const RSI_THRESHOLD = 37;
for (let i = 0; i < rsiData.length; i++) {
    if (rsiData[i].rsi < RSI_THRESHOLD) {
        if (!inDip) {
            rsiEvents.push(rsiData[i]);
            inDip = true;
        }
    } else {
        inDip = false;
    }
}
```

O algoritmo detecta o **primeiro ponto** de cada "dip" (queda abaixo do threshold). Após o RSI cair abaixo de 37, ele só registra um novo evento quando o RSI voltar acima de 37 e cair novamente.

### Períodos de retorno calculados:

| Variável | Dias | Coluna na tabela |
|----------|------|------------------|
| `1sem`   | 7    | 1 Semana         |
| `1mes`   | 30   | 1 Mês            |
| `3mes`   | 91   | 3 Meses          |
| `6mes`   | 182  | 6 Meses          |
| `1ano`   | 365  | 1 Ano            |

### Para adicionar/remover períodos de retorno:
Modifique o array `periods`:
```javascript
const periods = [
    { label: '1sem', days: 7 },
    { label: '1mes', days: 30 },
    { label: '3mes', days: 91 },
    { label: '6mes', days: 182 },
    { label: '1ano', days: 365 }
];
```

**Importante:** Se adicionar um período, também precisa adicionar um `<th>` correspondente no HTML da tabela:
```html
<thead>
    <tr>
        <th>#</th>
        <th>Data</th>
        <th>RSI</th>
        <th>Preço BTC</th>
        <th>1 Semana</th>
        <th>1 Mês</th>
        <th>3 Meses</th>
        <th>6 Meses</th>
        <th>1 Ano</th>
        <!-- Adicione novo <th> aqui -->
    </tr>
</thead>
```

### Cálculo de retorno:
```javascript
returns[p.label] = ((futurePrice - price0) / price0) * 100;
```
Se o preço futuro não existir nos dados (dado ainda não disponível), retorna `null` e mostra "-" na tabela.

### Linhas de Média e Mediana:
A tabela inclui automaticamente linhas de **média** e **mediana** ao final, calculadas sobre todos os eventos.

### Formatação visual:
- Retornos positivos: verde (`#00c850`, classe `.ret-pos`)
- Retornos negativos: vermelho (`#ff4444`, classe `.ret-neg`)
- Dados não disponíveis: cinza (`#555`, classe `.ret-na`)

---

## 🎨 Plugins dos Gráficos (Marcadores de Eventos)

### `rsiEventPluginBTC`:
- **Antes dos datasets:** Desenha faixas verticais verdes semitransparentes (`rgba(0, 200, 80, 0.08)`) nos timestamps dos eventos RSI.
- **Depois dos datasets:** Desenha bolinhas verdes nos pontos do gráfico de preço correspondentes aos eventos.

### `rsiEventPluginRSI`:
- Mesmo comportamento, mas no gráfico de RSI.

### `watermarkPlugin`:
- Renderiza uma marca d'água (logo VWV) no canto inferior esquerdo do gráfico de preço.
- A imagem está embutida como **base64** na variável `watermarkImg.src`.
- Opacidade: `0.055` (5.5%), altura: `55%` da chart area.

**Para alterar a marca d'água:** Substitua o valor de `watermarkImg.src` por outro data URL base64, ou remova/comente o plugin `watermarkPlugin` para remover a marca d'água.

---

## 🖌️ Engine de Desenho

### Estado global: `drawState`
```javascript
const drawState = {
    tool: 'select',      // Ferramenta ativa
    color: '#ff4444',     // Cor atual
    strokeWidth: 2,       // Espessura da linha
    fontSize: 14,         // Tamanho da fonte (texto)
    dashed: false,        // Linha tracejada
    drawing: false,       // Se está desenhando
    shiftKey: false,      // Se Shift está pressionado
    startX: 0, startY: 0,
    items: [],            // Desenhos completados
    currentPath: []       // Para desenho livre (pencil)
};
```

### Ferramentas disponíveis:
| Tool      | Descrição            | Atalho     |
|-----------|----------------------|------------|
| `select`  | Selecionar/mover     | Escape     |
| `pencil`  | Desenho livre        |            |
| `line`    | Linha reta           | +Shift = snap 45° |
| `hline`   | Linha horizontal     |            |
| `vline`   | Linha vertical       |            |
| `rect`    | Retângulo            | +Shift = quadrado |
| `circle`  | Círculo/Elipse       | +Shift = círculo perfeito |
| `arrow`   | Seta                 | +Shift = snap 45° |
| `text`    | Texto (prompt input) |            |

### Cores rápidas na sidebar:
```
#ff4444 (vermelho), #f57c00 (laranja), #ffeb3b (amarelo),
#00c850 (verde), #2196f3 (azul), #ffffff (branco)
```

### Atalhos de teclado:
- `Ctrl+Z`: Desfazer último desenho
- `Escape`: Voltar para ferramenta Select

---

## 📤 Exportação de Imagem

A função `exportImage()` gera um PNG combinando:
1. Os gráficos (renderizados via canvas)
2. O overlay de desenhos
3. A tabela de eventos (renderizada manualmente no canvas)

**Resolução:** `2x scale` para alta qualidade (Retina).

**Dimensões do canvas exportado:**
- Largura: `wrapper.offsetWidth × 2`
- Altura: `(chartH + padding + tableH) × 2`

A tabela é desenhada manualmente pixel a pixel no canvas de exportação, replicando o estilo visual da tabela HTML.

### Colunas da tabela na exportação:
As larguras são proporcionais usando `colRatios`:
```javascript
const colRatios = [3, 9, 5, 9, 9, 9, 9, 9, 9];
```

---

## 🎨 Estilização (CSS)

### Cores do tema escuro:
| Elemento            | Cor         |
|---------------------|-------------|
| Fundo principal     | `#000000`   |
| Sidebar             | `#0d0d0d`   |
| Bordas              | `#333`      |
| Grid do gráfico     | `rgba(255,255,255,0.08)` |
| Texto principal     | `#e0e0e0`   |
| Ticks dos eixos     | `#999`      |
| Tooltip fundo       | `#1a1a1a`   |
| Tabela header       | `#1a1a1a`   |
| Tabela body hover   | `#111`      |

### Dimensões fixas:
| Elemento      | Largura  | Altura  |
|---------------|----------|---------|
| Wrapper       | 1350px   | 850px   |
| BTC Chart     | 100%     | 550px   |
| RSI Chart     | 100%     | 300px   |
| Sidebar       | 52px     | 100vh   |
| Tabela        | 1350px   | max 400px (scroll) |

**Para alterar o tamanho dos gráficos:** Modifique:
1. `.wrapper` no CSS: `width: 1350px; height: 850px;`
2. Os `<div class="chart-panel">` no HTML: `style="height: 550px;"` e `style="height: 300px;"`
3. `.rsi-events-panel` no CSS: `width: 1350px;`

---

## 📁 Arquivos Auxiliares

| Arquivo          | Descrição                                                 |
|------------------|-----------------------------------------------------------|
| `1.csv`          | Dados brutos BTC com RSI e divergências (381 linhas)     |
| `2.csv`          | Dados brutos BTC com LWBC (176 linhas)                    |
| `wm.png`         | Imagem da marca d'água (logo VWV)                        |
| `wm_b64.txt`     | Marca d'água codificada em base64                        |
| `wm_dataurl.txt` | Marca d'água como data URL completo                      |

**Nota:** Os CSVs `1.csv` e `2.csv` são os dados fonte originais. Os dados usados no HTML são cópias embutidas inline. Para atualizar o gráfico, é necessário atualizar as strings CSV **dentro do HTML**, não apenas os arquivos CSV externos.

---

## 🔄 Guia: Como Atualizar Dados (Passo a Passo)

### 1. Adicionar novos pontos de dados BTC/RSI:

1. Abra `btc_chart.html`
2. Localize `const btcCsvData = \`` (busque por essa string)
3. Vá até a **última linha de dados** antes do `` \`; `` de fechamento
4. Adicione novas linhas no formato: `UNIX_TIMESTAMP_SECONDS,PRECO,RSI`
5. Para obter o timestamp Unix em segundos de uma data:
   ```
   Data: 2026-03-01 → Math.floor(new Date('2026-03-01').getTime() / 1000) = 1772265600
   ```
6. Repita o mesmo processo para `const lwbcCsvData = \`` se tiver dados LWBC

### 2. Alterar o threshold do RSI:

1. Busque `const RSI_THRESHOLD = 37;`
2. Altere o valor `37` para o desejado
3. O gráfico, a tabela e os marcadores se ajustam automaticamente

### 3. Alterar cores dos gráficos:

**BTC (laranja):**
- Linha: busque `borderColor: '#f57c00'` no dataset do `chart1`
- Gradiente: busque `gradientBtc.addColorStop` (3 linhas)

**RSI (verde):**
- Linha: busque `borderColor: '#00c850'` no dataset do `chart2`
- Gradiente: busque `gradientRsi.addColorStop` (3 linhas)

### 4. Alterar dimensões do gráfico:

1. CSS: `.wrapper { width: 1350px; height: 850px; }`
2. HTML: `<div class="chart-panel" style="height: 550px;">` (BTC)
3. HTML: `<div class="chart-panel" style="height: 300px;">` (RSI)
4. CSS: `.rsi-events-panel { width: 1350px; }`
5. Gradientes: recrie com a nova altura se necessário

### 5. Alterar a marca d'água:

1. Converta a nova imagem para base64
2. Busque `watermarkImg.src = 'data:image/png;base64,...'`
3. Substitua toda a string base64
4. Ajuste `ctx.globalAlpha = 0.055` para alterar opacidade
5. Ajuste `chartArea.height * 0.55` para alterar tamanho relativo

---

## ⚙️ Referência de Variáveis JavaScript Importantes

| Variável              | Tipo       | Descrição                                          |
|-----------------------|------------|-----------------------------------------------------|
| `btcCsvData`          | String     | Dados CSV inline do BTC + RSI                      |
| `lwbcCsvData`         | String     | Dados CSV inline do BTC + LWBC                     |
| `btcData`             | Array      | Dados parseados do BTC [{time, close, rsi}]        |
| `lwbcData`            | Array      | Dados parseados do LWBC [{time, close, lwbc}]      |
| `filteredBtcData`     | Array      | Dados BTC filtrados (a partir de abril 2014)       |
| `rsiData`             | Array      | Apenas pontos com RSI não-nulo                     |
| `chart1`              | Chart      | Instância Chart.js do gráfico BTC                  |
| `chart2`              | Chart      | Instância Chart.js do gráfico RSI                  |
| `RSI_THRESHOLD`       | Number     | Valor de corte do RSI para eventos (padrão: 37)    |
| `rsiEvents`           | Array      | Eventos detectados (RSI < threshold)               |
| `eventsWithReturns`   | Array      | Eventos com retornos calculados                    |
| `eventTimestamps`     | Array      | Timestamps dos eventos (usado pelos plugins)       |
| `drawState`           | Object     | Estado do engine de desenho                        |
| `overlay`             | Canvas     | Canvas de desenho overlay                          |
| `watermarkImg`        | Image      | Imagem da marca d'água (base64)                    |
| `timeMin` / `timeMax` | Date       | Range temporal dos eixos X                         |

---

## 📐 Estrutura HTML dos Elementos

### IDs importantes:
| ID                | Elemento       | Descrição                            |
|-------------------|----------------|--------------------------------------|
| `btcChart`        | `<canvas>`     | Canvas do gráfico de preço BTC      |
| `rsiChart`        | `<canvas>`     | Canvas do gráfico RSI               |
| `drawingOverlay`  | `<canvas>`     | Canvas do overlay de desenhos        |
| `chartWrapper`    | `<div>`        | Container dos gráficos (1350×850)   |
| `rsiEventsPanel`  | `<div>`        | Container da tabela de eventos      |
| `rsiEventsBody`   | `<tbody>`      | Body da tabela (populado via JS)    |
| `sidebar`         | `<div>`        | Barra lateral de ferramentas        |
| `layersPanel`     | `<div>`        | Painel de camadas de desenhos       |

---

## 🚀 Como Abrir

Simplesmente abra o arquivo `btc_chart.html` em qualquer navegador moderno. Não requer servidor — tudo é autocontido (dados inline, dependências via CDN).

**Requisitos:** Conexão com internet (para carregar Chart.js e date-fns adapter via CDN).
