# Kotlin Coroutines — O que são?

[![Gabriel Kirsten](https://miro.medium.com/fit/c/96/96/1*WNdtxtt_4cMq50aX3IDx8A.jpeg)](https://medium.com/@gabrielkirsten?source=post_page-----f9375c314480-----------------------------------)

[Gabriel Kirsten](https://medium.com/@gabrielkirsten?source=post_page-----f9375c314480-----------------------------------) - [Jun 25, 2020](https://coderef.com.br/kotlin-coroutines-o-que-são-f9375c314480?source=post_page-----f9375c314480-----------------------------------) · 6 min read

*Nesse artigo vamos abordar um pouco dos problemas que as* ***coroutines\*** *resolvem e como elas funcionam na linguagem Kotlin. Iremos apontar principalmente as diferenças entre Threads e Coroutines.*

Esse artigo faz parte da nossa série sobre coroutines no Kotlin:

- [Kotlin Coroutines — O que são?](https://coderef.com.br/kotlin-coroutines-o-que-são-f9375c314480) (você está aqui)
- [Kotlin Coroutines — Como funcionam?](https://medium.com/p/79364d903937)

As **coroutines** são componentes dentro de um software que podem ter a sua execução suspensa e ter a sua atividade retomada em um momento futuro, possivelmente em outra Thread.

Hoje temos a tendência de aplicações e frameworks utilizando computação assíncrona, reativa ou não bloqueante e várias maneiras de resolver esse problema (como *callbacks* explicitos ou o [RxJava](https://github.com/ReactiveX/RxJava)), as coroutines são uma delas que a linguagem Kotlin oferece a nível de linguagem.

![img](https://miro.medium.com/max/304/0*6-yYBWAKp-RFIOPw.png)

Logo do Kotlin

O conceito de coroutines não é exclusivo do Kotlin, algumas linguagens como Clojure e Rust implementam através de bibliotecas terceiras, outras linguagens como Kotlin e Go implementam de maneira nativa. Também encontramos coroutines expressas com um nome mais “artístico” e especifico de acordo da linguagem como: Goroutines (Go) e Cloroutine (Clojure).

No Kotlin, as coroutines estão disponíveis desde a versão 1.3 (Revision 3.3) e de maneira experimental na versão 1.1 e 1.2.

# Diferença entre execução: bloqueante vs não bloqueantes e Threads vs Coroutines

Nessa sessão vamos abordar um pouco sobre os conceitos de bloqueante/não bloqueante e explicar um pouco a diferença entre Threads e Coroutines.

Para ilustrar esse problema, vamos considerar o seguinte cenário: Temos uma aplicação e será necessário executar um requisição via HTTP para um servidor remoto. Cada request HTTP conta com uma latência imposta pelo meio físico do trafego de dados e pelo processamento no servidor remoto. A imagem a seguir ilustra o cenário:

![img](https://miro.medium.com/max/1210/1*t1l6gGHRi-wWIIQd6wDoiQ.png)

Duas aplicações trocando informações com o protocolo HTTP, a comunicação entre elas está sujeita a uma latência (destacada em azul), iremos analisar somente o fluxo que acontece na aplicação "Your App".

## Solução síncrona e bloqueante (sem coroutines)

Vamos resolver o problema de maneira mais simples possível. Conforme a imagem a seguir, temos nossa **Thread de processamento principal** chamada de **Main Thread,** responsável por controlar o fluxo principal da aplicação. Quando enviamos a requisição na Main Thread vamos precisar aguardar a resposta do servidor remoto (trecho representado em vermelho), você pode hipoteticamente imaginar esse trecho de código como um simples loop infinito verificando se o servidor remoto respondeu. Enquanto o servidor não responder sua Thread fica bloqueada consumindo preciosos recursos de processamento.

![img](https://miro.medium.com/max/56/1*7JzXHCSnt18d8ywnYnXpKQ.png?q=20)

![img](https://miro.medium.com/max/424/1*7JzXHCSnt18d8ywnYnXpKQ.png)

Representação da execução de uma operação de consulta a um servidor HTTP, o trecho em vermelho representa o momento em que a Main Thread é bloqueada para esperar a resposta do servidor remoto.

## Solução assíncrona e bloqueante (ainda sem coroutines)

Qual o problema do exemplo anterior? Basicamente consumo de recursos desnecessariamente alocados. Podemos encarar o exemplo como uma solução síncrona e bloqueante pois a execução do request foi realizada de forma que exista somente um fluxo de execução (somente uma Thread) e a Thread seja bloqueada executando um código que está somente esperando por algo.

Então como podemos resolver esse possível problema? Simplesmente executando o request HTTP em uma nova Thread, liberando a Thread principal para realizar outras ações, como renderizar uma interface de usuário por exemplo. Confira a imagem a seguir:

![img](https://miro.medium.com/max/46/1*EahaJ7mQQLxvCg2VFUOftg.png?q=20)

![img](https://miro.medium.com/max/399/1*EahaJ7mQQLxvCg2VFUOftg.png)

O fluxo de realização do request HTTP agora é executado em uma nova Thread (representado em azul), a Main Thread fica liberada (representado em verde) enquanto o request é realizado.

## Solução Assíncrona e Não Bloqueante (com coroutines)

E agora? Qual o problema da nova solução? Por mais que estamos executando duas operações de maneira a liberar a Main Thread, ainda bloqueamos uma nova Thread que está provavelmente utilizando o poder de processamento de uma unidade de processamento enquanto aguardar o request.

Podemos então utilizar uma abordagem utilizando **funções suspensas**, onde podemos utilizar um client HTTP não bloqueante (*Caso queira conhecer um exemplo, o* [*WebClient*](https://docs.spring.io/spring-boot/docs/2.1.0.RELEASE/reference/html/boot-features-webclient.html) *do Spring implementa um cliente HTTP não bloqueante*), o cliente HTTP não bloqueante pode utilizar uma `suspend function` do Kotlin para deixar a função suspensa até que seja recebida uma resposta do servidor HTTP, liberando a Thread para outro processamento até que seja chamado um callback para continuar a execução que dependa do response HTTP.

Veja o exemplo:

![img](https://miro.medium.com/max/60/1*VQBU_-mGDb1OoK02vX2b-Q.png?q=20)

![img](https://miro.medium.com/max/470/1*VQBU_-mGDb1OoK02vX2b-Q.png)

Solução de processamento HTTP através de uma abordagem não bloqueante.

Essa solução não ocupa processamento aguardando uma resposta HTTP e também poderia ser associada com outras Thread, mesclando um modelo de multi-threads com a abordagem de funções suspensas.

**Ps**. Nesse ponto é importante lembrar que coroutines não são sinônimos de paralelismo e sim de assíncronia, entretanto elas podem sim executar tarefas de maneira paralela dependendo da maneira em que o seu contexto for configurado, veremos em outro post sobre contexto de corroutines.

# Comparações com as Threads

A propria [documentação do Kotlin](https://kotlinlang.org/docs/tutorials/coroutines/coroutines-basic-jvm.html) fala que você pode pensar em coroutines como uma *light-weight thread*, pois oferecem a possibilidade de ser executadas em paralelo, aguardar a sua execução e comunicar uma com as outras, porém é uma solução mais leve, podendo ser criada em centenas.

Várias coroutines podem ser criadas na mesma threads, em grandes quantidades. Tente executar um código que cria 100 mil coroutines, agora tente fazer o mesmo com Threads, provavelmente conhecerá (ou revisitará) o *Out of Memory*.

<iframe src="https://cdn.embedly.com/widgets/media.html?src=https%3A%2F%2Fgiphy.com%2Fembed%2Fg79am6uuZJKSc%2Ftwitter%2Fiframe&amp;display_name=Giphy&amp;url=https%3A%2F%2Fmedia.giphy.com%2Fmedia%2Fg79am6uuZJKSc%2Fgiphy.gif&amp;image=https%3A%2F%2Fi.giphy.com%2Fmedia%2Fg79am6uuZJKSc%2F200.gif&amp;key=a19fcc184b9711e1b4764040d3dc5c07&amp;type=text%2Fhtml&amp;schema=giphy" allowfullscreen="" frameborder="0" height="223" width="435" title="Burning The It Crowd GIF - Find &amp; Share on GIPHY" class="t u v md aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 348.594px;"></iframe>

## Diferença entre Thread e Coroutines na prática

Vamos implementar duas simples aplicações em Kotlin para exemplificar a diferença de uma Thread e uma Coroutine. A aplicação terá a simples tarefa de receber um número inteiro como parâmetro e retornar a sua soma com 10, essa função será chamada de `addTen()`, essa função vai ter um atraso de 1 segundo simulando uma operação demorada. Chamaremos a função duas vezes com os parametros 1 e 2 e vamos analisar o tempo de resposta de cada uma delas.

- **Solução utilizando Threads**

O exemplo a seguir mostra então a aplicação utilizando Threads:

<iframe src="https://coderef.com.br/media/79207741effa94b5dc76a3db539e6ac8" allowfullscreen="" frameborder="0" height="347" width="680" title="Example Kotlin Thread" class="t u v md aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 347px;"></iframe>

Ao analisar a resposta da aplicação, podemos ver o cenário que esperávamos. Para o primeiro caso (parâmetro 1) e para o segundo caso (parâmetro 2) temos as respostas 11 e 12, respectivamente, e o tempo de execução foi de 2 segundos, considerando 1 segundo para cada cálculo de `addTen()`.

```
async task 1: 11
async task 2: 12
execution time: [2001] ms
```

- **Solução utilizando Coroutines**

O Kotlin permite trabalhar com os famosos `async`/ `await` , bem conhecidos na comunidade de desenvolvedores Javascript ou C#. Veja o exemplo a seguir:

<iframe src="https://coderef.com.br/media/51873301581629d9e3f342303d8618f9" allowfullscreen="" frameborder="0" height="369" width="680" title="kotlin_coroutine_example.kt" class="t u v md aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 369px;"></iframe>

Podemos ver também a declaração da função `addTen()` o modificador `suspend` indicando que essa função pode ser suspensa, além de alterar o `Thread.sleep()`por uma chamada a função `delay()`, uma alternativa para pausar a execução de maneira não bloqueante utilizando coroutines. Outro ponto importante é que temos a utilização do `runBlocking()`na função `main()`, que executa uma nova coroutine e aguarda até a sua conclusão.

Vamos executar esse exemplo e analisar a resposta:

```
task 1: 11
task 2: 12
execution time: [1023] ms
```

Os resultados das operações são os mesmos, mas o tempo de execução foi cerca de 1 segundo, é praticamente a metade do tempo utilizado quando implementamos o exemplo utilizando Threads. Isso aconteceu pois a execução do `delay()`foi uma operação que **não bloqueou** a Thread, **suspendendo a sua execução** de maneira que fosse possível executar a próxima chamada da função `addTen()`.

Lembrando novamente que como as coroutines, por padrão, não executam o código em paralelo, consequentemente, por mais que pareça a solução com coroutines não executou o código em paralelo mas suspendeu a execução pois a função `delay()` é uma operação não bloqueante que liberou a Thread. Tente executar o mesmo exemplo utilizando apenas alterando a chamada de `delay()` para `Thread.sleep()` e verá que mesmo a função `addTen()` sendo suspensa ela não conseguirá executar em metade do tempo pois a Thread não estará liberada.

# What's Next

Neste post demos uma introdução importante sobre coroutines de maneira mais abstrata, no proximo post iremos abordar mais a fundo a sintaxe e como o Kotlin implementa esse padrão, a importante interface Continuation e o que acontece com uma *suspend function* depois de compilada.

Até mais