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
<img width="1400" height="727" alt="Captura de tela 2025-11-24 184624" src="https://github.com/user-attachments/assets/639da265-6723-4063-b369-ab29479aba08" />




1.2 Action

A ação é necessária para o envio dos parâmetros. Essa deve ser configurada de acordo ao problema. Nesse exemplo, a condição para o acionamento da ação(imagem 1) é o status do trigger ser igual a `TRUE`. Quando a condição for cumprida, a ação tomada(imagem 2) sera a operação de notificar o perfil admin e o mediatype n8n_ping_loss.

Condição da ação:
<img width="1400" height="434" alt="Captura de tela 2025-11-24 184445" src="https://github.com/user-attachments/assets/a3248536-f1e8-4065-8ec3-d547a74a7c4d" />

Operação da ação:
<img width="1407" height="727" alt="Captura de tela 2025-11-24 184512" src="https://github.com/user-attachments/assets/1e370fcc-73af-42ac-a361-5a3fa226aa46" />


1.3 Mediatype

O mediatype [zbx_export_mediatypes.yaml](https://github.com/DeVxSantoS/Monitoramento-de-Redes-WI-FI-com-Zabbix-e-N8N/blob/b12dea5edbe08d928bc66d08127ac5dcdc03abbd/docs/zbx_export_mediatypes.yaml) é o que enviará as métricas para o N8N. Nele deve ser modificado o valor do campo URL para o endereço URL do webhook do workflow no N8N. O mediatype deve ser adicionado na aba user > notification(imagem 1).

Notificação:
<img width="1849" height="489" alt="Captura de tela 2025-11-24 184714" src="https://github.com/user-attachments/assets/705de839-c4d1-4fce-89e8-cd3f402eb757" />


1.4 Webhook

O webhook irá receber os parametros do mediatype via HTTP call na URL correpondente ao webhook:
<img width="576" height="804" alt="Captura de tela 2025-11-28 183123" src="https://github.com/user-attachments/assets/42311452-650b-4f4b-974e-5e656120f328" />


### 2 HTTP request - API zabbix


A API do Zabbix possibilita coletar metricas de hosts através de um HTTP request. Esse método é usado para verificar a gravidade do problema.


2.1 Node HTTP request

O N8N possui um node nativa para o zabbix. Através dele é possivel obter as metricas do evento desejado. Para isso foi configurado um HTTP request pelo metodo post, assim os parametros serão carregados pelo corpo da mensagem.

<img width="520" height="808" alt="image" src="https://github.com/user-attachments/assets/307fdc7f-41be-4665-9385-77875110552a" />

Para requisitar apenas as metricas necessarias e preciso especificar o corpo da requisição. O corpo usado tem o seguinte formato:

'''{
  "jsonrpc": "2.0",
  "method": "item.get",
  "params": {
    "output": ["itemid", "name", "key_", "lastvalue", "units"],
    "search": {
    "key_": "icmppingloss"
    },
    "filter": {"host": ["{{ $json.body.host_name }}"] },
    "sortfield": "name",
    "sortorder": "ASC"
  },
  "id": 1
}'''

2.2 Credencial Zabbix

Para que a requisição seja aceita pela API do zabbix é preciso de uma credencial de acesso. São necessarios a URL do zabbix e um token da API, o segundo pode ser gerado dentro do zabbix na aba Users>API tokens.

<img width="1911" height="901" alt="image" src="https://github.com/user-attachments/assets/c6d2358c-6042-42ae-beb6-9d1ca6133cf2" />



## ⚠️ Importante


Este projeto é apenas um protótipo e que não deve ser usado em ambientes reais devido as suas brechas de segurança.
