# MVP - Previsão do Tempo no WhatsApp com n8n

Este projeto é um chatbot de previsão do tempo para WhatsApp, desenvolvido como um MVP (Mínimo Produto Viável). O chatbot, apelidado de "Climinha", permite que os usuários solicitem a previsão do tempo para qualquer cidade e optem por receber notificações sobre mudanças climáticas significativas.

## Visão Geral do Projeto

O objetivo principal é criar um serviço de utilidade que, através de uma interface conversacional no WhatsApp, entrega informações meteorológicas precisas e personalizadas. O projeto demonstra habilidades em automação com n8n, integração de APIs e o uso de inteligência artificial para criar uma experiência de usuário natural e interativa.

## Funcionalidades Implementadas

O fluxo de trabalho em n8n foi projetado para cobrir todas as funcionalidades solicitadas no desafio:

1.  **Receber Pedidos de Previsão:**
    *   O sistema é ativado por um **Webhook** que escuta as mensagens recebidas no WhatsApp.
    *   Ele consegue processar tanto mensagens de texto quanto de áudio, utilizando a **API da OpenAI (Whisper)** para transcrever áudios em texto.

2.  **Entregar a Previsão Inicial:**
    *   Um **Agente de IA (AI Agent)**, alimentado pelo **GPT-4.1-mini da OpenAI**, interpreta a mensagem do usuário para extrair a cidade de interesse.
    *   O agente utiliza uma ferramenta que consulta a **API do OpenWeatherMap** para obter as coordenadas geográficas da cidade.
    *   Com as coordenadas, outra ferramenta busca a previsão do tempo detalhada (temperatura, condição do céu, umidade, chance de chuva).
    *   A resposta é formatada de maneira amigável e enviada ao usuário.

3.  **Oferecer Opção de Notificações:**
    *   Após entregar a previsão, o agente de IA proativamente pergunta se o usuário deseja receber alertas automáticos para aquela cidade.
    *   A conversa é mantida em contexto através do **Redis Chat Memory**, permitindo que o bot se lembre das interações anteriores com o mesmo usuário.

4.  **Registrar a Escolha do Usuário:**
    *   As preferências do usuário (cidade monitorada, status da notificação, etc.) são salvas e gerenciadas em uma planilha do **Google Sheets**.
    *   O sistema consegue consultar, salvar e editar as preferências de cada usuário, tratando usuários novos e existentes de forma diferente.

5.  **Enviar Alertas Sempre que Necessário:**
    *   Um segundo fluxo de trabalho, acionado por um **Schedule Trigger** (gatilho agendado), roda de hora em hora.
    *   Ele verifica a planilha do Google Sheets para encontrar todos os usuários com notificações ativas.
    *   Para cada usuário, o sistema consulta a previsão do tempo e, caso identifique uma condição climática adversa (chuva forte, tempestade, calor extremo), dispara uma mensagem de alerta para o WhatsApp do usuário.

## Estrutura do Fluxo de Trabalho (Workflow)

O projeto é composto por dois fluxos principais no n8n:

### 1. Fluxo Interativo (Acionado por Webhook)

Este fluxo gerencia a interação em tempo real com o usuário.

-   `Webhook`: Ponto de entrada que recebe os dados da mensagem do WhatsApp.
-   `Message Type Switch`: Direciona o fluxo com base no tipo de mensagem (texto ou áudio).
-   `OpenAI (Whisper)`: Transcreve mensagens de áudio.
-   `AI Agent (OpenAI GPT-4.1-mini)`: O cérebro da operação. Ele gerencia a conversa, decide quais ferramentas usar e formula as respostas.
    -   **Ferramentas Conectadas:**
        -   `obter_coordenadas_cidade`: Busca latitude e longitude na API do OpenWeatherMap.
        -   `consultar_previsao_tempo_por_coordenada`: Busca a previsão do tempo.
        -   `consultar/salvar/editar_preferencias_usuario`: Ferramentas que interagem com o Google Sheets para gerenciar os dados do usuário.
        -   `Think`: Ferramenta interna do agente para planejamento e raciocínio.
-   `Redis Chat Memory`: Armazena o histórico da conversa para manter o contexto.
-   `Evolution API Node`: Envia as respostas formatadas para o WhatsApp do usuário.

### 2. Fluxo de Notificações (Acionado por Agendamento)

Este fluxo opera de forma autônoma para monitorar e alertar os usuários.

-   `Schedule Trigger`: Inicia o fluxo a cada hora.
-   `Google Sheets Node`: Lê a lista de usuários que optaram por receber alertas.
-   `Loop Over Items`: Processa cada usuário individualmente.
-   `OpenAI (GPT-4.1-mini)`: Um agente mais simples que analisa as condições atuais, compara com as preferências do usuário (horário, tipo de alerta) e decide se uma notificação é necessária.
-   `If Node`: Verifica se a resposta da IA é um alerta ou um sinal para não fazer nada (`[NO_NOTIFICATION_NEEDED]`).
-   `Evolution API Node`: Envia o alerta, caso necessário.

## Tecnologias e Ferramentas

-   **Plataforma de Automação:** n8n
-   **Comunicação:** WhatsApp, integrado via Evolution API
-   **Inteligência Artificial:** OpenAI (GPT-4.1-mini para conversação, Whisper para transcrição)
-   **Dados Meteorológicos:** API OpenWeatherMap
-   **Banco de Dados/Persistência:** Google Sheets e Redis (para memória de curto prazo da conversa)

## Como Executar

Para replicar este projeto, você precisará seguir estes passos:

1.  **Configuração do Google Sheets:**
    *   Crie uma nova planilha no Google Sheets.
    *   O fluxo espera que a primeira página (aba) se chame `Página1`.
    *   Configure o cabeçalho da planilha exatamente com as seguintes colunas:

| remoteJid | NomeUsuario | cidadesMonitoradas | preferenciaNotificacao | horarioNotificacao | ultimaAtualizacao | NotificacaoAtiva |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `5511...`| `NomeExemplo` | `sao paulo` | `Diariamente` | `08:00` | `2025-07-11...` | `TRUE` |

2.  **Configurar as Credenciais no n8n:**
    *   Adicione suas chaves de API para OpenAI, OpenWeatherMap.
    *   Configure a autenticação OAuth2 para o Google Sheets.
    *   Configure as credenciais para sua instância da Evolution API.

3.  **Importar os Fluxos:**
    *   Importe o arquivo `workflow.json` para a sua instância n8n.
    *   Verifique se os nós que usam credenciais (Google Sheets, OpenAI, Evolution API) estão corretamente associados às credenciais que você configurou.

4.  **Configurar o Webhook:**
    *   Aponte a sua instância da Evolution API para o URL do webhook de teste (e depois de produção) gerado pelo nó `Webhook` no n8n.

5.  **Ativar os Fluxos:**
    *   Ative ambos os fluxos de trabalho: o interativo e o de notificações.
