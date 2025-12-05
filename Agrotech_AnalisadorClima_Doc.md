# Documentação Técnica: AnalisadorAgricola (ControllerAnalisadorClima)

## 1. Visão Geral
A classe `AnalisadorAgricola` atua como um motor de **Business Intelligence (BI)** para o agronegócio. Ela processa dados climáticos (diários e horários) e parâmetros específicos de culturas para gerar recomendações operacionais e alertas estratégicos (Plantio, Colheita, Pulverização, Doenças e Estresses).

**Versão do Algoritmo**: 2.0 (Lógica Híbrida - Horária e Diária)

---

## 2. Estrutura de Dados e Inicialização

### Input de Dados
A classe é instanciada recebendo:
- **Dados Diários**: Médias e máximas gerais (Precipitação, Vento, Temperatura, etc).
- **Dados Horários**: Detalhamento hora-a-hora (essencial para janelas operacionais precisas).
- **Cultura**: Tipo da lavoura (Define os limiares de risco).
- **Datas**: Semeadura e Previsão de Colheita (Define a fenologia).

### Parâmetros por Cultura (`definirParametrosCultura`)
Os limiares de alerta são ajustados automaticamente baseados na cultura:

| Cultura | Temp. Crítica (Calor) | Umidade Doença | Molhamento Crítico | Ciclo Médio |
| :--- | :--- | :--- | :--- | :--- |
| **Soja** | 34.0°C | 80% | 10 horas | 120 dias |
| **Milho** | 35.0°C | 85% | 12 horas | 140 dias |
| **Algodão** | 38.0°C | 85% | 8 horas | 165 dias |

---

## 3. Motores de Decisão (Analisadores)

O método `gerarAlertasCompletos()` orquestra a execução de seis motores de análise distintos. Abaixo, o detalhamento da lógica de cada um:

### 3.1. Analisador de Plantio (`analisarPlantio`)
Objetivo: Identificar janelas de solo apto para semeadura.

**Fluxo de Decisão:**
1.  **Checagem de Ocupação**: Verifica se o campo está livre (Data atual > Data Colheita anterior).
2.  **Saturação do Solo**: 
    - Se chuva diária > **40mm** → **VETADO** (Solo saturado/lama).
3.  **Janela Operacional (Análise Híbrida)**:
    - Se chuva entre **10mm e 40mm**: Verifica hora a hora.
    - Se houver < **8 horas sem chuva** no dia → **COMPROMETIDO** (Sem tempo hábil).
    - Caso contrário → **ATENÇÃO** (Janela parcial, risco de atolamento).
4.  **Risco de Seca (Longo Prazo)**:
    - Se acumulado de chuva em 7 dias < **5mm** → **ALERTA DE SECA** (Solo inviável para germinação).
5.  **Cenário Ideal**: Se nenhuma das anteriores for atingida, retorna status **FAVORÁVEL**.

### 3.2. Analisador de Pulverização (`analisarPulverizacao`)
Objetivo: Garantir eficácia da aplicação e evitar deriva/perda.

**Parâmetros Limites:**
- Vento: Configurado pelo Tenant (Padrão geralmente < 10-15km/h).
- Temperatura: < 30°C.
- Umidade: > 55%.

**Fluxo de Decisão:**
1.  **Risco de Chuva Imediata**: 
    - Probabilidade > 70% E Chuva > 5mm → **RISCO** (Lavagem do produto).
2.  **Análise Horária (Próximos 3 dias)**:
    - Conta quantas horas do dia atendem a **TODOS** os requisitos (Vento, Temp e Umidade ideais + Sem Chuva).
    - **0 horas boas** → **VETADO**.
    - **< 6 horas boas** → **RESTRITO** (Janela curta).
3.  **Análise Diária (Longo Prazo)**:
    - Se rajada máxima de vento > 20km/h → **VETADO** (Previsão de ventania).

### 3.3. Analisador de Colheita (`analisarColheita`)
Objetivo: Evitar colher grão úmido ou ardido.

**Fluxo de Decisão:**
1.  **Validação de Data**: Só roda se estiver próximo (14 dias) da data estimada de colheita.
2.  **Janela de Umidade (Hora a Hora)**:
    - Conta horas com **Precipitação = 0** E **Umidade Ar < 75%**.
    - **< 4 horas aptas** → **VETADO** (Umidade excessiva/Orvalho).
    - **< 8 horas aptas** → **ATENÇÃO** (Janela curta).
3.  **Aceleração (Estratégico)**:
    - Se acumulado de chuva total previsa > 40mm → **ALERTA DE ACELERAÇÃO** (Colher antes da chuvarada).

### 3.4. Analisador de Doenças (`analisarRiscoDoencas`)
Objetivo: Prever surtos fúngicos (Ex: Ferrugem). Usa o conceito de **Leaf Wetness Duration** (Duração do Molhamento Foliar).

**Lógica:**
1.  Varre os dados hora a hora calculando sequências contínuas onde:
    - (Umidade >= 90% OU Chuva > 0) **E** (Temperatura dentro da faixa ideal do fungo).
2.  Mantém a conta da maior sequência de horas ininterruptas.
3.  Se a sequência > **Horas Críticas da Cultura** (Ex: 10h para Soja) → **ALERTA FÚNGICO**.

### 3.5. Analisador de Estresse Hídrico (`analisarEstresseHidrico`)
Objetivo: Detectar "Veranicos" (períodos secos com alta demanda atmosférica).

**Critério de Veranico (Diário):**
- Chuva < 2mm
- Evapotranspiração (ET0) > 4mm
- Radiação Solar > 15MJ/m²
- **Gatilho**: 3 dias consecutivos atendendo a essas condições.

### 3.6. Analisador de Estresse Térmico (`analisarEstresseTermicoCalor`)
Objetivo: Evitar abortamento de flores/vagens por calor extremo.

**Critério:**
- Temperatura Máxima > **Temp. Crítica da Cultura** (Ex: 34°C Soja).
- **Gatilho**: 2 dias consecutivos de calor extremo.

---

## 4. Retorno e Formatação
O método final filtra os alertas nulos, adiciona um sufixo de data amigável (ex: "Hoje", "Amanhã", "12/05") e retorna um array limpo para o Frontend exibir no Dashboard.
