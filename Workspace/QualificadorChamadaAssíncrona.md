# Qualificador da Chamada Assíncrona

O qualificador de chamada assíncrona permite especificar que uma chamada assíncrona deve ocorrer como parte de uma transação do cliente. O qualificador determina quando a mensagem é enviada para o destino.

Uma chamada síncrona suspende um encadeamento até que ele seja concluído. Uma chamada assíncrona faz com que uma mensagem seja incluída em uma fila de mensagens; então o cliente continua com seu próprio processamento, sem aguardar uma resposta para sua mensagem.

**Local:** A chamada assíncrona é configurada em uma referência.

**Configurações:** O qualificador de chamada assíncrona pode ter as seguintes configurações:

- **Confirmação** - A mensagem só é enviada se a transação for confirmada. A transação das chamadas assíncronas que utilizam a referência será feita como parte de qualquer transação global do cliente ou transação local estendida (isto é, onde o qualificador **Valor** da transação para a implementação do cliente é **local**).
- **Call (padrão)** - Chamadas assíncronas utilizando a referência ocorre imediatamente. Isto é logicamente equivalente a suspender a transação atual e, em seguida, chamar o serviço de destino.

Consulte a discussão sobre quando usar **Commit** e **Call** na seção sobre inovação assíncrona em [Qualidade do Serviço: Qualificadores de Serviços de Negócios](https://www.ibm.com/docs/pt-br/SSTLXK_8.5.6/com.ibm.wbpm.wid.admin.doc/qos/topics/cpolicies.html).

## Notas sobre Programação

O sistema ignora a configuração da transação de junção quando um serviço é chamado de forma assíncrona.

Se você utilizar este qualificador na referência de um componente Java e se a implementação Java faz chamadas assíncronas com uma resposta adiada, este qualificador deve ter o valor "chamada". Se o valor é configurado como "confirmar", a implementação Java aguarda por uma resposta para uma mensagem que ainda não foi enviada.