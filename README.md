# 🚜 Agrotech - VigiaSafra

## 📋 Visão Geral
**Agrotech (VigiaSafra)** é uma plataforma avançada de gestão agrícola focada no monitoramento de talhões, análise climática e assistência inteligente via IA. O sistema permite que produtores rurais gerenciem suas terras, visualizem dados meteorológicos em tempo real e tomem decisões baseadas em dados com o auxílio do **VIGIAMIND**, um assistente virtual integrado.

### 🎯 Principais Objetivos
- **Gestão de Talhões**: Cadastro e monitoramento geoespacial de áreas produtivas.
- **Monitoramento Climático**: Visualização de precipitação, vento e temperatura sobre o mapa.
- **Inteligência Artificial**: Assistência em tempo real via **VIGIAMIND** (Google Gemini) para insights agronômicos.
- **Centralização de Dados**: Dashboard unificado para controle total da propriedade.

---

## 🏗️ Arquitetura Técnica

O projeto segue uma arquitetura **MVC (Model-View-Controller)** personalizada em PHP, com um frontend modular em JavaScript (ES6).

### 📂 Estrutura de Diretórios
- **`application/`**: Núcleo do backend.
  - **`Config/`**: Configurações globais (Banco de dados, Rotas).
  - **`controllers/`**: Lógica de negócios (ex: `GerenciarTalhoes`, `ia`).
  - **`core/`**: Classes base (`Router`, `Main`, `Template`, `Sec`).
  - **`DataBase/`**: Abstração de acesso a dados (`_pdo`).
- **`public/`**: Assets públicos.
  - **`js/modules/`**: Módulos JavaScript isolados (`ai.js`, `map.js`, `ui.js`).
  - **`assets/`**: Bibliotecas de terceiros (Leaflet, etc.).
- **`views/`**: Templates de visualização (`.tpl.php`).

---

## 🧩 Módulos e Funcionalidades

### 1. Sistema de Rotas (`Router`)
O roteamento é gerenciado pela classe `_router` (`application/core/Router/router.php`), que carrega definições de `application/Config/router.json`.
- **Fluxo**: A URL é analisada por parâmetros (ex: `?talhoes`). O roteador identifica o controlador correspondente e o instancia.
- **Exemplo**: `?talhoes&router=listar` -> Carrega o controlador de listagem de talhões.

### 2. Gestão de Talhões (`GerenciarTalhoes`)
Módulo responsável pelo CRUD de áreas produtivas.
- **Listagem**: `ListarTalhoes` (`_task.php`) busca dados no MySQL, gera ícones SVG dinâmicos baseados na geometria do polígono e renderiza a tabela.
- **Mapa**: Integração com Leaflet para desenhar e visualizar talhões.
- **Banco de Dados**: Tabela `talhoes` armazena metadados e geometria (GeoJSON).

### 3. Mapa Interativo (`map.js`)
O coração do dashboard. Utiliza a biblioteca **Leaflet**.
- **Camadas**: Alternância entre Satélite (Esri) e Mapa Padrão (OSM).
- **Dados Climáticos**: Integração com APIs (ex: Open-Meteo) para sobrepor camadas de precipitação e vento.
- **Ferramentas de Desenho**: `Leaflet.Draw` permite criar novos talhões diretamente no mapa, calculando a área em hectares automaticamente.

### 4. VIGIAMIND - IA Agrícola (`ai.js` & `GeminiController`)
Assistente virtual flutuante que utiliza a API do **Google Gemini**.
- **Frontend (`ai.js`)**: Gerencia a interface de chat, estado (aberto/fechado/expandido) e comunicação com o backend.
- **Backend (`GeminiController.php`)**: Recebe o prompt, adiciona contexto (se necessário) e comunica-se com a API do Google Gemini, retornando a resposta processada.
- **Fluxo**: Usuário digita -> `ai.js` (POST) -> `GeminiController` -> Google API -> Resposta JSON -> Renderização no Chat.

---

## 🛠️ Stack Tecnológico

### Backend
- **Linguagem**: PHP 8+
- **Banco de Dados**: MySQL (Driver PDO)
- **Arquitetura**: MVC Customizado

### Frontend
- **JavaScript**: ES6 Modules
- **CSS**: TailwindCSS (Estilização utilitária)
- **Mapas**: Leaflet.js + Plugins (Draw, Omnivore, GeometryUtil)
- **Ícones**: FontAwesome

---

## 🚀 Como Executar (Ambiente Local)

1. **Requisitos**: Servidor Web (Apache/Nginx), PHP 8+, MySQL.
2. **Configuração**:
   - Importe o banco de dados (schema não incluso neste repositório).
   - Configure as credenciais em `application/Config/database.php`.
   - Verifique a URL base em `application/Config/config.php`.
3. **Dependências**:
   - Execute `composer install` (se houver `composer.json`).
   - Assegure que as bibliotecas JS em `public/assets` estejam presentes.
4. **Acesso**:
   - Abra o navegador em `http://localhost/agrotech`.

---

## 🔒 Segurança
- **CSRF**: Proteção via tokens gerenciada pela classe `Main::CheckTokenCRFS`.
- **Sessão**: Gerenciamento seguro de sessão com `session_regenerate_id`.
- **PDO**: Consultas preparadas para prevenir SQL Injection.

---

> **Nota**: Documentação gerada automaticamente baseada na análise do código fonte em 25/11/2025.
