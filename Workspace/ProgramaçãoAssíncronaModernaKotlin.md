# Programação assíncrona moderna com Kotlin

# ## Palestra de Roman Elizarov no QCon SF

02 MAR 2018 3 MIN(S) DE LEITURA por [Amit K Gupta](https://www.infoq.com/br/profile/Amit-K-Gupta/) traduzido por [Danilo Pereira de Luca](https://www.infoq.com/br/profile/Danilo-Pereira-de-Luca/)

[Roman Elizarov](https://qconsf.com/sf2017/speakers/roman-elizarov), líder técnico das bibliotecas de Kotlin na JetBrains, palestrou sobre "Programação assíncrona moderna [com Kotlin](https://qconsf.com/sf2017/presentation/fresh-async-kotlin)" no [QCon São Francisco](https://qconsf.com/). Durante sua apresentação, Elizarov demonstrou como o Kotlin aborda as dificuldades enfrentadas na escrita de código assíncrono em linguagens como Java, C# e Javascript. Código assíncrono em Kotlin se parece muito com o código síncrono que muitos desenvolvedores estão acostumados a escrever em linguagens como Java.

Tradicionalmente, as aplicações usam muitos elementos computacionais e processamento, embora muitas aplicações modernas utilizem serviços com REST APIs ou microservices. Esse estilo de arquitetura geralmente envolve aguardar o retorno para continuar o fluxo.

![Image 1](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%203.12.19%20PM-1510973068421.png)

Nesse código, o desenvolvedor chama o método requestToken() e espera seu resultado para então prosseguir para a próxima chamada que é um createPost(), que também espera ser finalizado para então chamar o último passo, processPost(), para renderizar o post.

Uma maneira simples de implementar esse código é usando threads. No entanto, embora geralmente seja correto usar threads para casos simples, como o código de três etapas anterior, mas depender fortemente das threads pode não ser viável para aplicações complexas que envolvem inúmeras chamadas de serviço. Cada thread tem uma memória e agendamento e, exatamente por isso, máquinas modernas podem ter um número limitado de threads ao mesmo tempo.

Callbacks podem ser uma alternativa à utilização de threads, especialmente em ambientes que sejam single threads e usam Javascript. Códigos que usam callbacks tendem a ter diversas cadeias de códigos chamados de "callback hell", como mostrado no exemplo a seguir:

![Image 2](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%204.15.18%20PM-1510973067686.png)

Estes trechos de códigos aninhados tendem a piorar quanto mais complexo for o código - códigos com callbacks intrinsecamente difíceis de ler e de debugar.

Nos últimos anos, as Futures/Promises/Rx, que são maneiras de implementar uma mesma ideia em diferentes linguagens, se tornaram conhecidas como uma boa alternativa para evitar o problema de "callback hell".

![img](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%204.20.40%20PM-1510973067904.png)

Os códigos escritos com Futures/Promises são mais fáceis de ler e de serem entendidos do que os com callback. Esta abordagem também fornece um mecanismo de tratamento de exceções, o que torna mais fácil a depuração do código.

Contudo, ainda existem potenciais problemas ao usar Futures/Promises. Por exemplo, lidar com múltiplas combinações de métodos no escopo da Future do objeto. O código anterior utiliza duas combinações: .thenCompose e .thenAccept. As combinações podem ter diferentes nomes em diferentes linguagens, e o desenvolvedor também deve se lembrar de combinações para code branching, loops, etc.

As rotinas cooperativas de Kotlin (coroutines) ajudam a mitigar os problemas que um desenvolvedor enfrenta com Futures/Promises. As coroutines permitem que o desenvolvedor escreva o código com assinaturas naturais usadas por ele ao escrever o código síncrono. Coroutines usam a palavra-chave suspend no início da definição de coroutine para indicar uma operação assíncrona.

![Image 4](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%204.46.34%20PM-1510973068126.png)

Porém, a desvantagem desta abordagem é que é difícil dizer quais das chamadas de serviço são assíncronas. No código anterior, por exemplo, não há como dizer se as chamadas requestToken() e createPost() são assíncronas. A IDE pode ajudar aqui mostrando marcadores de pontos de suspensão.

![img](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%204.50.10%20PM-1510973068269.png)

Ao usar coroutines, um desenvolvedor agora escreve loops regulares, manipulação de exceção regulares e funções regulares de ordem superior, por exemplo: forEach, let, apply, repeat, filter, map, use, etc. O desenvolvedor também pode criar funções personalizadas de ordem superior com o estilo de codificação usado na codificação síncrona. Por baixo dos panos, quando o compilador Kotlin identifica a palavra-chave suspend, ele adiciona um parâmetro invisível chamado Continuation para a chamada da função a nível de bytecode na JVM. Esse parâmetro Continuation é basicamente um callback.

![Image 5](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%205.06.17%20PM-1510973068533.png)

Quando o Kotlin identifica uma sequência de pontos com suspend, ele compila os pontos com suspend para uma máquina de estado.

![Image 6](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2017/11/fresh-async-kotlin-qconsf/en/resources/1Screen%20Shot%202017-11-17%20at%205.09.29%20PM-1510973068716.png)

Os desenvolvedores são encorajados a explorar a biblioteca Kotlin de rotinas cooperativas disponíveis para suportar programação assíncrona. Essas bibliotecas ainda estão evoluindo, sendo experimentais no momento. Contudo, elas incluem integrações com outras abordagens assíncronas, como: CompleteableFuture, RxJava e Reactive Streams. Elas também têm a possibilidade de integrar o Kotlin com UI frameworks single-threads, incluindo Android e JavaFX.

#### Conteúdo publicado no tópico [Java](https://www.infoq.com/br/java/)

SEGUIR TÓPICO

##### Tópicos Relacionados:

 

- DESENVOLVIMENTO 

- QCON 

- JAVA 

- QCON SAN FRANCISCO 2017 

- KOTLIN

  

- #### CONTEÚDO EDITORIAL RELACIONADO

  - ##### [Apache Arrow e Java: Transferência de Big Data na velocidade da luz](https://www.infoq.com/br/articles/apache-arrow-java/?itm_source=infoq&itm_medium=related_content_link&itm_campaign=relatedContent_news_clk)

  - ##### [Destaque do recurso Java: Classes seladas](https://www.infoq.com/br/articles/java-sealed-classes/?itm_source=infoq&itm_medium=related_content_link&itm_campaign=relatedContent_news_clk)

  - ##### [O último conteúdo do InfoQ Brasil](https://www.infoq.com/br/news/2021/02/ultimo-conteudo-do-infoq-brasil/?itm_source=infoq&itm_medium=related_content_link&itm_campaign=relatedContent_news_clk)

  - ##### [O fim do Privacy Shield pode levar a um desastre para os provedores de nuvem em hiperescala](https://www.infoq.com/br/articles/privacy-shield-hyperscale-clouds/?itm_source=infoq&itm_medium=related_content_link&itm_campaign=relatedContent_news_clk)

  - ##### [Entendendo Os Valores e Princípios Ágeis](https://www.infoq.com/br/minibooks/valores-principios-ageis/?itm_source=infoq&itm_medium=related_content_link&itm_campaign=relatedContent_news_clk)