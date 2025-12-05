# Node por node do fluxo N8N

Descrição do funcionamento e lógica do fluxo [ping_loss](https://github.com/DeVxSantoS/Monitoramento-de-Redes-WI-FI-com-Zabbix-e-N8N/blob/950f40a04087aadc7dcc1b3748bc17b4bcbbfba8/docs/workflow)


## 1. Webhook (Webhook)

Tipo: Webhook (HTTP POST)

O que faz: recebe a chamada HTTP do Zabbix com os parâmetros do evento, iniciando o fluxo (método POST). 

[imagem do webhook]

Dados importantes disponíveis: body.host_name, body.host_ip, body.event_opdata ou outros campos enviados pelo Zabbix — usados nas expressões dos nodes seguintes. 



## 2. Zabbix HTTP Request 1

Tipo: API Call (HTTP Request)

O que faz: consulta a API do Zabbix para buscar itens (icmppingloss) filtrados pelo host recebido no webhook. 

[imagem do node]

Configuração chave observada:
```
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
```

Monta e envia a requisição POST para http://"zabbix-server-ip"/zabbix/api_jsonrpc.php usando as credenciais definidas (credential Zabbix account).


## 3. Switch 1


Tipo: Switch

O que faz: avalia o valor retornado pelo Zabbix (result[0].lastvalue) e cria caminhos de execução diferentes conforme regras (=100%, >40%). 

[imagem do switch]

Regras observadas:

Saída 100% → neste fluxo a saída conecta para No Operation, do nothing (ou outra ação conforme sua configuração). 

Saída 40% (>= 40) → encaminha para o node Execute a command (SSH) para tentar uma ação corretiva. 

Dica: reveja a ordem das regras: switches avaliam sequencialmente — certifique-se que a regra mais específica venha antes da mais genérica.


## 4. Execute a command — SSH


Tipo: SSH

O que faz: executa um comando remoto via SSH no host alvo. 

```
command: restart & exit
```

[imagem do Ssh]

Recebe o item do Switch (quando lastvalue >= 40 no exemplo), abre conexão SSH usando as credenciais configuradas, captura a saída e status do comando e encaminha o resultado para o node Wait. 

Observações de segurança: confirme que a conta SSH tem permissões adequadas, que os host da rede permitem SSH, que as credenciais dos hosts sejam iguais para todos os host e avalie riscos de executar restart automaticamente.


## 5. Wait

Tipo: Wait (espera)

O que faz: pausa o fluxo por um período configurado (2 minutos) antes de prosseguir — usado para dar tempo ao dispositivo/ação corretiva surtir efeito. 

[imagem do wait]

Após o tempo expirar, passa para o próximo node (Zabbix HTTP Request 2) para re-checar o estado.


## 6. Zabbix HTTP Request 2

Tipo: HTTP Request (nova chamada à API do Zabbix)

O que faz: reconsulta o Zabbix após o término do tempo de espera para verificar se o lastvalue (ex.: perda ICMP) mudou. Similar ao Zabbix HTTP Request 1.

Configuração chave observada: corpo JSON semelhante ao item.get buscando icmppingloss e filtrando pelo host.


## 7. Switch 2

No seu fluxo existem duas variantes em exports distintos: um Switch1 (com regras semelhantes ao primeiro Switch) e/ou um node If (que testa result[0].lastvalue >= 40). Ambas fazem o papel de decidir se tomar nova ação ou encerrar.

Tipo: Switch.

Recebe o resultado atualizado do Zabbix HTTP Request 2.

Avalia a condição:

Se o valor for igual a 100 (ex.: =100) → Segue para o node No Operation, isso indica que o host não reiniciou corretamente, evidenciando a necessidade de intervenção humana.

Se o valor ainda estiver alto (ex.: >= 40) → segue para Execute a command novamente (pode tentar nova intervenção). 

Se o valor estiver dentro do esperado (ex.: <= 40), encerra o fluxo.

Observação: esse padrão cria um laço de tentativa → espera → validação. Se for preciso limitar tentativas, adicione um contador (ex.: incrementar um campo no item e checar limite) para evitar loops infinitos.


# 8. No Operation, do nothing

Tipo: NoOp

O que faz(não faz:/): ponto final que não realiza ação — serve para encerrar elegantemente um caminho sem erros. 

[imagem do NoOp]

## 8. Conexões e fluxo lógico (resumo sequencial)

1. Webhook recebe evento externo. 
2. Zabbix HTTP Request 1 consulta o item (icmppingloss) do host. 
3. Switch 1, decide com base em lastvalue:
Se condição crítica → Execute a command (SSH). 
Senão → No Operation e fim de ramo. 
4. Execute a command faz ação corretiva. 
5. Wait (espera 2 minutos)
6. Zabbix HTTP Request 2 reconsulta o Zabbix. 
7. Switch 2, valida resultado pós-correção:
Se ainda fora do aceitável → repetir Execute a command (ou outro tratamento).
Se não envia sinal para NoOp.
8. No Operation (fim). 


## 9. Recomendações práticas / pontos a conferir

Confirme qual expressão de host usar na filter ({{ $json.body.host_name }}.

A função .toNumber adicionada ao valor do evento énecessária para converter o valor da perda de pacote (que é recebido como string) para um Int (um número inteiro é necessário para a comparaçãodo switch). Ex.: 
```
={{ $json.result[0].lastvalue.toNumber() }}
```

Evite loops infinitos: implemente contador de tentativas ou limite de re-execuções.

Valide permissões da conta SSH e a segurança do comando remoto (especialmente comandos que fazem restart). 

Teste com pinData (resultado simulado) para validar roteamento sem mexer no Zabbix de produção.
