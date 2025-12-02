1. Webhook (Webhook)


Tipo: Webhook (HTTP POST)

O que faz / quando dispara: recebe a chamada HTTP externa que inicia o fluxo (método POST, caminho zabbix-n8n). 


Passo a passo de execução:




O n8n expõe um endpoint /webhook/zabbix-n8n esperando um POST. 


Quando o Zabbix (ou outro sistema) envia a notificação, o corpo (body) da requisição entra como $json.body no item.


O node encaminha o item recebido para o próximo node conectado (a primeira Switch). 




Dados importantes disponíveis: body.host_name, body.event_opdata ou outros campos enviados pelo Zabbix — usados nas expressões dos nodes seguintes. 



2. Zabbix HTTP Request 1 (Zabbix HTTP Request 1)


Tipo: HTTP Request (JSON-RPC para Zabbix)

O que faz: consulta a API do Zabbix para buscar itens (ex.: icmppingloss) filtrados pelo host recebido no webhook. 


Configuração chave observada:


{
  "jsonrpc": "2.0",
  "method": "item.get",
  "params": {
    "output": ["itemid","name","key_","lastvalue","units"],
    "search": {"key_":"icmppingloss"},
    "filter": {"host": ["{{ $json.body.host_name }}"]},
    "sortfield":"name","sortorder":"ASC"
  },
  "id":1
}



(trecho extraído do node). 


Passo a passo de execução:




Recebe o item vindo do Webhook.


Monta e envia a requisição POST para http://<zabbix-server>/zabbix/api_jsonrpc.php usando as credenciais definidas (credential Zabbix account). 


Aguarda a resposta JSON do Zabbix com result contendo lastvalue, itemid, etc.


Encaminha a resposta para o node Switch seguinte (ou outro conectado).




Observação: No pinData do fluxo de exemplo há um resultado simulado com lastvalue: "50" — útil para testes sem precisar do Zabbix ativo. 



3. Switch (Switch)


Tipo: Switch (roteamento condicional)

O que faz: avalia o valor retornado pelo Zabbix (result[0].lastvalue) e cria caminhos de execução diferentes conforme regras (ex.: 100%, 40%). 


Regras observadas (resumidas):




Se ={{ $json.result[0].lastvalue.toNumber() }} == 100 → saída 100%.


Se ={{ $json.result[0].lastvalue.toNumber() }} >= 40 → saída 40%.

(trechos do node). 




Passo a passo de execução:




Recebe o item com result do Zabbix HTTP Request 1.


Avalia a expressão com lastvalue convertida para número.


Escolhe qual saída seguir:



Saída 100% → neste fluxo a saída conecta para No Operation, do nothing (ou outra ação conforme sua configuração). 


Saída 40% (>= 40) → encaminha para o node Execute a command (SSH) para tentar uma ação corretiva. 








Dica: reveja a ordem das regras: switches avaliam sequencialmente — certifique-se que a regra mais específica venha antes da mais genérica.



4. No Operation, do nothing (No Operation, do nothing)


Tipo: NoOp

O que faz: ponto final/ramo que não realiza ação — serve para encerrar elegantemente um caminho sem erros. 


Passo a passo de execução:




Recebe o item do Switch quando a condição de encaminhamento corresponder.


Não realiza modificações nem chamadas externas; finaliza o fluxo para aquele ramo.





5. Execute a command (Execute a command) — SSH


Tipo: SSH

O que faz: executa um comando remoto via SSH no host alvo (ex.: ping -c 4 192.168.0.1 || restart). 


Configuração chave observada:




command: ping -c 4 192.168.0.1 || restart


Credencial: conta SSH (tipo sshPassword).


onError: continueRegularOutput (continua fluxo mesmo se houver erro). 




Passo a passo de execução:




Recebe o item do Switch (quando lastvalue >= 40 no exemplo).


Abre conexão SSH usando as credenciais configuradas.


Executa o comando especificado (no exemplo: tenta ping e, se falhar, executa restart localmente — fileira do comando depende do shell remoto).


Captura saída e status do comando e encaminha o resultado para o node Wait. 




Observações de segurança: confirme que a conta SSH tem permissões adequadas e avalie riscos de executar restart automaticamente.



6. Wait (Wait)


Tipo: Wait (espera)

O que faz: pausa o fluxo por um período configurado (ex.: 2 minutes) antes de prosseguir — usado para dar tempo ao dispositivo/ação corretiva surtir efeito. 


Passo a passo de execução:




Recebe saída do node Execute a command.


Suspende a execução por amount: 2 unit: minutes (duas minutos). 


Após o tempo expirar, passa para o próximo node (Zabbix HTTP Request 2) para re-checar o estado.




Observação: Wait insere execução agendada no banco de execução do n8n — verifique carga/tempo se for um fluxo com alto volume.



7. Zabbix HTTP Request 2 (Zabbix HTTP Request 2)


Tipo: HTTP Request (nova chamada à API do Zabbix)

O que faz: reconsulta o Zabbix após a ação/espera para verificar se o lastvalue (ex.: perda ICMP) mudou. Similar ao Zabbix HTTP Request 1, mas pode usar outra URL/servidor. 


Configuração chave observada: corpo JSON semelhante ao item.get buscando icmppingloss e filtrando pelo host (expressão usa $('Webhook').item.json.body.host_name em alguns casos). 


Passo a passo de execução:




Após o Wait, monta nova requisição item.get para o Zabbix.


Envia a requisição e obtém result atualizado com lastvalue.


Encaminha o retorno para o Switch1 (ou If) que avaliará se a ação foi eficaz. 




Observação: em alguns exports há duas variações do body: uma usa {{ $('Webhook').item.json.body.host_name }} e outra usa {{ $json.body.host_name }} — confirme qual expressão funciona no seu contexto. 



8. Switch1 / If (ver variação)


No seu fluxo existem duas variantes em exports distintos: um Switch1 (com regras semelhantes ao primeiro Switch) e/ou um node If (que testa result[0].lastvalue >= 40). Ambas fazem o papel de decidir se tomar nova ação ou encerrar.


Tipo: Switch ou If dependendo da versão do fluxo.


Passo a passo de execução (comportamento geral):




Recebe o resultado atualizado do Zabbix HTTP Request 2.


Avalia a condição:



Se o valor ainda estiver alto (ex.: >= 40) → segue para Execute a command novamente (pode tentar nova intervenção). 


Se o valor estiver dentro do esperado → segue para No Operation, do nothing (encerra o ramo). 








Observação: esse padrão cria um laço de tentativa → espera → validação. Se for preciso limitar tentativas, adicione um contador (ex.: incrementar um campo no item e checar limite) para evitar loops infinitos.



9. Conexões e fluxo lógico (resumo sequencial)




Webhook recebe evento externo. 


Zabbix HTTP Request 1 consulta o item (icmppingloss) do host. 


Switch decide com base em lastvalue:



Se condição crítica → Execute a command (SSH). 


Senão → No Operation e fim de ramo. 






Execute a command faz ação corretiva → Wait (espera 2 minutos). 


Wait → Zabbix HTTP Request 2 reconsulta o Zabbix. 


Switch1 / If valida resultado pós-correção:



Se ainda fora do aceitável → repetir Execute a command (ou outro tratamento).


Senão → No Operation (fim). 









10. Recomendações práticas / pontos a conferir




Confirme qual expressão de host usar na filter ({{ $json.body.host_name }} vs {{ $('Webhook')... }}) — ambas aparecem nas variantes do fluxo.


Evite loops infinitos: implemente contador de tentativas ou limite de re-execuções.


Valide permissões da conta SSH e a segurança do comando remoto (especialmente comandos que fazem restart). 


Teste com pinData (resultado simulado) para validar roteamento sem mexer no Zabbix de produção. 





11. Trechos úteis (expressões) — para copiar/colar






Exemplo de expressão usada no Switch para comparar valor numérico:


={{ $json.result[0].lastvalue.toNumber() }}



(usada com operadores equals / gte). 






Exemplo de body JSON para item.get:


{
  "jsonrpc": "2.0",
  "method": "item.get",
  "params": {
    "output": ["itemid","name","key_","lastvalue","units"],
    "search": {"key_":"icmppingloss"},
    "filter": {"host": ["{{ $json.body.host_name }}"]},
    "sortfield":"name",
    "sortorder":"ASC"
  },
  "id":1
}



(ver node Zabbix HTTP Request).
