# Documentação Técnica Geral: Plataforma VigiaSafra

## 1. Visão Geral da Arquitetura
O **VigiaSafra** é uma plataforma de inteligência agronômica de precisão. Sua arquitetura é projetada para processar grandes volumes de dados climáticos e geoespaciais, transformando-os em insights acionáveis através de interfaces visuais (Mapas/Heatmaps) e assistentes cognitivos (VigiaMind).

*   **Backend**: PHP 8.x (MVC Próprio)
*   **Frontend**: JavaScript Vanilla (Modular - ES6 Modules)
*   **Database**: MySQL (Dados Cadastrais) + SQLite (Cache de Alta Performance)
*   **Maps Engine**: Leaflet.js
*   **AI Engine**: Google Gemini 1.5 Flash (via VigiaMind Controller)
*   **Weather Provider**: Open-Meteo (Modelos ECMWF/GFS)

---

## 2. VigiaMind: O Núcleo de Inteligência Artificial
O **VigiaMind** não é apenas um chatbot, é um agente contextual que "enxerga" os dados da fazenda antes de responder. A atuaação dele segue um pipeline estrito:

### 2.1. Injeção de Contexto (RAG - Retrieval-Augmented Generation)
Antes de enviar qualquer pergunta do usuário para o modelo de linguagem (LLM), o backend intercepta a requisição e realiza o seguinte processo:

1.  **Detecção de Intenção/Talhão**: O sistema varre a frase do usuário buscando nomes de talhões cadastrados no banco de dados.
    *   *Ex*: Se o usuário diz "Como está a chuva no **Talhão 15**?", o sistema identifica o ID e a geometria do "Talhão 15".
2.  **Consulta de Business Intelligence (BI)**:
    *   O sistema calcula o **centróide** do talhão identificado.
    *   Executa uma consulta climática interna (sem custo de API externa se estiver em cache) para recuperar dados horários e diários.
    *   Processa os dados pelo `ControllerAnalisadorClima` para gerar alertas técnicos (ex: "Risco de Ferrugem detectado").
3.  **Montagem do System Prompt**:
    *   Um prompt dinâmico é construído contendo:
        *   **Persona**: Especialista Agronômico, cético e baseado em probabilidades.
        *   **Dados Hard**: O JSON cru dos dados climáticos e os alertas gerados são inseridos no prompt.
        *   **Metadados**: Cultura (Soja/Milho), Data de Plantio e Área.
4.  **Resposta**: O modelo responde com base nesses dados injetados, garantindo que ele não alucine sobre o clima, mas sim interprete os dados reais fornecidos pelo sistema.

---

## 3. Mapas e Heatmaps (Map Engine)
Os mapas do VigiaSafra não são imagens estáticas, são vetores dinâmicos renderizados no navegador.

### 3.1. Origem dos Dados (Data Source)
Os dados climáticos provêm da API **Open-Meteo**, priorizando o modelo **ECMWF IFS** (European Centre for Medium-Range Weather Forecasts), considerado o padrão-ouro para precisão em agronegócio.
*   **Resolução**: ~9-11km (interpolado para coordenadas exatas).
*   **Variáveis**: Chuva, Temperatura, Vento, Rajada, Umidade, Evapotranspiração (ET0) e Probabilidade de Precipitação.

### 3.2. Definição de Coordenadas de Monitoramento
Para otimizar o processamento, o sistema não consulta o clima para cada pixel do polígono.
1.  **Cálculo de Centróide**: Para cada talhão cadastrado, o sistema calcula matematicamente o ponto central (Centróide Geométrico).
2.  **Clustering de Precisão**: As coordenadas são arredondadas para 4 casas decimais (~11 metros). Se dois talhões tiverem centróides muito próximos, eles compartilham o mesmo "ponto de grade" de previsão, economizando requisições e processamento via Cache SQLite.

### 3.3. Geração dos Heatmaps
O efeito de "Mapa de Calor" é, na verdade, uma **Coroplética Vetorial**.
*   O Frontend (`modules/map.js` e `ui.js`) mantém os polígonos dos talhões desenhados na tela.
*   Ao selecionar uma variável (ex: "Chuva Acumulada 7 dias"), o JS itera sobre cada talhão.
*   O valor do dado é comparado com uma escala de cores (ex: 0mm = Vermelho, 50mm = Azul).
*   A propriedade `fillColor` do polígono SVG é atualizada instantaneamente. Isso permite interatividade total (clique, hover) que um heatmap de imagem (raster) não permitiria.

---

## 4. Gestão de Talhões (Entrada de Dados)
O sistema aceita duas formas de entrada, ambas normalizadas para coordenadas geográficas WGS84.

### 4.1. Importação KML (`ImportKmlController.php`)
Ideal para cargas massivas de dados vindos de maquinário ou softwares de GIS (Google Earth, QGIS).
1.  **Parsing**: O sistema lê o arquivo XML/KML e extrai as coordenadas usando Expressões Regulares (Regex).
2.  **Validação Geométrica**:
    *   Calcula a área usando a **Fórmula Geodésica** (considerando a curvatura da Terra) para garantir precisão em hectares.
    *   **Filtro de Ruído**: Talhões muito pequenos (ex: < 5-10ha, configurável) detectados como "sujeira" de desenho são descartados automaticamente.
3.  **Persistência**: Salva a geometria (WKT/JSON) e o centróide pré-calculado no banco MySQL.

### 4.2. Desenho Manual (`Leaflet.draw`)
Para ajustes finos ou cadastros rápidos.
*   Utiliza a biblioteca `Leaflet.Draw` no frontend.
*   Conforme o usuário clica no mapa, a área é calculada em tempo real (JS) para feedback imediato.
*   Ao salvar, os vértices do polígono são enviados para a API que realiza o mesmo processo de validação do importador.

---

## 5. Estratégia de Cache e Performance (`ControllerApiClima.php`)
Dada a natureza pesada dos dados climáticos, o sistema implementa um **Cache Híbrido com File Locking**:
1.  **SQLite como Cache**: Um arquivo `.db` local armazena as respostas da API externa.
2.  **Atomicidade**: Uso de `flock` (File Locking) para evitar que duas requisições tentem escrever no cache ao mesmo tempo (Condição de Corrida).
3.  **Batch Processing**: As requisições para a API externa são agrupadas em lotes e executadas em paralelo (`curl_multi`), permitindo atualizar centenas de talhões em segundos.

---

## 6. Fluxo de Dados Resumido
1.  **Usuário** cadastra talhão (KML ou Desenho) -> **Banco de Dados** (Salva Geometria + Centróide).
2.  **Usuário** acessa Dashboard.
3.  **Frontend** solicita dados climáticos para os IDs dos talhões visíveis.
4.  **Backend** verifica Cache SQLite -> Se ausente, busca na **Open-Meteo**.
5.  **Backend** processa dados brutos -> Gera Alertas (AnalisadorAgricola).
6.  **Frontend** recebe JSON -> Colore os mapas (Heatmap) e exibe cards de alerta.
7.  **Usuário** pergunta ao **VigiaMind** -> AI lê esse mesmo contexto e responde.
