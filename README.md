# Monitoramento de Redes com Zabbix e N8N



## 📒 Descrição


Esse projeto foi desenvolvido com o objetivo de criar um sistema capaz de monitorar, identificar e resolver problemas em redes LAN de forma imediata e automática. Tal projeto foi elaborado ao CETEP Bacia do Jacuípe como parte das atividades acadêmicas do Curso de Redes de Computadores.




## 🛠 Softwares Utilizados


a) Ubuntu Server, ambiente escolhido para suportar o projeto;

b) Zabbix Server, responsável pelo monitoramento da rede e identificação de problemas;

d) Docker, utilizado como gerenciador de containers;

e) N8N, executado dentro de um container Docker, responsável pela automação de ações corretivas.




## 🤖 Lógica da Automação 


A automação de ações corretivas foi realizada utilizando a plataforma n8n, que, mediante alertas enviados pelo Zabbix, executava fluxos de análise e tomada de decisão. A sequência de execuções pode ser entendida da seguinte forma:


1. Um trigger(gatilho) é acionado quando o zabbix detecta algum problema.

2. O problema é notificado para um perfil no zabbix através de uma action(ação).

3. Um mediatype vinculado ao perfil cria um HTTP call para enviar os parâmetros do evento para o N8N

4. O N8N recebe os parâmetros pelo trigger node webhook

5. Com os parâmetros do evento, o fluxo no N8N executa uma ação para resolver o problema.

6. O fluxo checa se o problema foi resolvido e, caso não, repete a ação, até que o problema seja resolvido.



## 🌐 Integração


### 1. HTTP call - Webhook N8N


Para a correta execução do workflow é necessário que a integração entre o zabbix e o N8N seja feita de maneira correta.


1.1 Trigger

A maioria dos gatilhos já vem configurados por padrão no zabbix, caso seu problema não esteja entre eles, é necessário criá-lo. O gatilho usado para esse projeto foi o de perda de pacotes:
<img width="1400" height="434" alt="Captura de tela 2025-11-24 184445" src="https://github.com/user-attachments/assets/eb2e9c09-f672-49a5-a81c-1b6374dc6d56" />


1.2 Action

A ação é necessária para o envio dos parâmetros. Essa deve ser configurada de acordo ao problema. Nesse exemplo, a condição para o acionamento da ação(imagem 1) é o status do trigger ser igual a `TRUE`. Quando a condição for cumprida, a ação tomada(imagem 2) sera a operação de notificar o perfil admin e o mediatype n8n_ping_loss.


1.3 Mediatype

O médiatype [arquivo.yaml] é o que enviará as métricas para o N8N. Nele deve ser modificado o valor do campo URL para o endereço URL do webhook do workflow no N8N. O mediatype deve ser adicionado na aba user > notification(imagem 1).


1.4 Webhook

Escrever em sala.


### 2 HTTP request - API zabbix


A API do Zabbix possibilita coletar metricas de hosts através de um HTTP request. Esse método é usado para verificar a gravidade do problema.


2.1 Node HTTP request

Escrever em sala


2.1 Credencial Zabbix

Escrever em sala


## ⚠️ Importante


Este projeto é apenas um protótipo e que não deve ser usado em ambientes reais devido as suas brechas de segurança.
