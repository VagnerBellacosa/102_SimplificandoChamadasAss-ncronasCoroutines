# Kotlin Coroutines

[![Gabrielle Rodrigues](https://miro.medium.com/fit/c/96/96/0*oIsxSxqIaNiuDnJW.)](https://medium.com/@gabriellerodrigues?source=post_page-----e6d048a59c40-----------------------------------)

[Gabrielle Rodrigues](https://medium.com/@gabriellerodrigues?source=post_page-----e6d048a59c40-----------------------------------) - [Mar 26, 2020](https://medium.com/ifood-tech/kotlin-coroutines-e6d048a59c40?source=post_page-----e6d048a59c40-----------------------------------) · 5 min read

*Como escrever códigos assíncronos e sequenciais de um jeito mais fácil*

![img](https://miro.medium.com/max/1400/0*M_neM8JazJtngmYd.jpg)

O uso de Coroutines do Kotlin tem se tornado cada vez mais frequente. Um dos principais motivos é que ele se mostra eficiente na forma de trabalhar assíncronamente por já fazer parte da linguagem Kotlin, sem a necessidade de usar bibliotecas externas (como por exemplo o Rx). Veremos nesse artigo os pontos principais de Coroutines para quem quer começar a usar, ou até mesmo para quem já usa, mas ainda não entende muito bem alguns dos conceitos.

# O que é Coroutines?

Coroutines é uma feature do Kotlin na qual possibilita escrever códigos assíncronos mais facilmente e de maneira sequencial, sem usar o padrão de Callback (o famoso Callback Hell). Coroutines está disponível desde o Kotlin 1.1 como experimental, ou a partir do Kotlin 1.3 como versão estável.

# Para que serve Coroutines?

Coroutines têm um menor custo na criação e troca de contexto comparado com threads, sendo muito mais eficientes. Várias coroutines podem rodar usando uma mesma thread, podendo ser criadas quantas coroutines forem necessárias, ao contrário de threads em que o uso é limitado.

Na documentação do Kotlin é mostrado um exemplo de código que cria 100 mil coroutines (em um loop) e toda coroutine exibe um ponto (*println(".")*). A execução desse código levou apenas 1 segundo e todos os 100 mil pontos foram exibidos. E eles desafiam a quem quiser fazer o mesmo com threads. O que acontece com threads? Possivelmente teríamos muitos *Out of Memory*.

Agora que sabemos o que é Coroutines, vamos entender os principais conceitos.

# Suspend functions

Funções *suspend* (declaradas como *suspend fun*) são funções que podem ser suspensas sem bloquear a thread. Ou seja, uma função *suspend* pode ser pausada e resumida sem bloquear a thread atual.

Vamos analisar a diferença entre funções *blocking* (bloqueantes) e *suspending.* Uma função é bloqueante quando ela só libera a thread na qual está executando após terminar a execução de tudo, enquanto uma função *suspend* pode pausar durante sua execução para que outra função possa executar na mesma thread. Quando essa segunda função termina, a primeira (que é a *suspend*) volta a executar.

Por baixo dos panos, uma *suspend function* é uma função regular (ou seja, sem o *suspend*) mas com um parâmetro a mais do tipo *Continuation<T>.* O *Continuation<T>* é uma interface com dois métodos para resumir: um para resumir quando deu sucesso e o outro para resumir quando deu erro.

Fazendo um comparativo com funções regulares, uma função regular tem duas operações comuns: *invoke* (ou call) e *return*. Coroutines têm essas operações, mas também têm a mais: *suspend* e *resume*.

- **suspend:** Pausa a execução da coroutine atual, salvando todas as variáveis locais
- **resume:** Continua a execução de uma coroutine que foi suspendida, do ponto em que ela pausou (suspendeu)

Um ponto importante sobre *suspend functions* é que elas só podem ser chamadas por outras *suspend functions* ou por uma coroutine, mas caso você esqueça disso, a IDE irá alertar em tempo de compilação.

<iframe src="https://medium.com/media/4c4774d39cd55b4db21a75ffdd2e8f4d" allowfullscreen="" frameborder="0" height="106" width="680" title="Suspend function" class="t u v te ak" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 105.99px;"></iframe>

Existem duas funções básicas para criar uma coroutine: *launch* e *async*. Em ambas funções é preciso passar um contexto (chamado *Dispatchers*) no qual a sua coroutine vai executar (main thread, IO e etc). Falaremos em breve sobre esse contexto, mas vamos primeiro ver como funciona o *launch* e *async*.

## launch

O *launch* irá criar uma coroutine de acordo com o contexto que for passado (*Dispatchers*). A função *launch* vai retornar um tipo *Job.*

<iframe src="https://medium.com/media/1a34d9d812c276f51f0a2354d300b73f" allowfullscreen="" frameborder="0" height="150" width="680" title="LaunchExample.kt" class="t u v te ak" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 149.983px;"></iframe>

## async

O *async* também irá criar uma coroutine de acordo com o contexto (*Dispatchers*) que for passado. O que diferencia o *async* do *launch* é o tipo de retorno dessas funções. Vimos que o *launch* retorna um *Job*, enquanto que o *async* retorna um *Deferred*. O *Deferred* possui o método *await()*, que quando chamado vai aguardar o retorno da coroutine. Portanto, o que for executar logo abaixo do *await* só será executado após o retorno dessa coroutine.

<iframe src="https://medium.com/media/0c926d03a39472cbc2ddf4ce3d191776" allowfullscreen="" frameborder="0" height="150" width="680" title="AsyncExample.kt" class="t u v te ak" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 149.983px;"></iframe>

<iframe src="https://medium.com/media/46b5105f4f13c63a536894631cdad862" allowfullscreen="" frameborder="0" height="172" width="680" title="AsyncAwaitExample.kt" class="t u v te ak" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 171.997px;"></iframe>

## runBlocking

O *runBlocking* é uma função de coroutine ao qual não passamos nenhum contexto (*Dispatchers*) e por conta disso, o seu código rodará na main thread. Ele bloqueia a thread interruptamente até completar sua execução. Por isso, o Kotlin, em sua documentação, recomenda fortemente que o *runBlocking* não seja usado por uma coroutine, sendo somente recomendado usar em funções main e em testes — em testes podemos priorizar usar outras opções antes do *runBlocking* como por exemplo o [*runBlockingTest*](https://github.com/Kotlin/kotlinx.coroutines/tree/master/kotlinx-coroutines-test).

## Dispatchers ("contexto")

Falamos que coroutine pode se suspender ao declararmos uma função como *suspend*, mas é o *Dispatcher* que sabe como resumir ("retomar") essa coroutine. Vamos ver então os tipos de *Dispatchers* que podem ser usados:

- **Main:** usa a thread de UI (user interface). Portanto, só é recomendado usar quando realmente precisar interagir com a interface de usuário;
- **IO:** usado para operações de input/output. Geralmente é usado quando precisa esperar uma resposta, como por exemplo: requisições para um servidor, leitura e/ou escrita num banco de dados, etc;
- **Default:** usado para usos intensivos de CPU, como ordenação de listas, parse de JSON, DiffUtils, etc;
- **Unconfined:** para operações que não precisam de uma thread específica. É recomendado usar quando não consome tempo de CPU nem atualiza dados compartilhados (como a interface de usuário), confinados em uma thread específica. A coroutine que usa esse *Dispatcher* é executada na mesma thread de quem a chamou, mas só se mantém nessa thread até o primeiro ponto de suspensão (primeira *suspend fun*). Depois de suspendida, é resumida na thread.

Em diversos momentos precisamos alternar entre contextos (*Dispatchers*) das coroutines. Vamos imaginar que faremos uma request para o servidor (usando *Dispatchers.IO*) e o que retornar será enviado para um LiveData (possivelmente usando *Dispatchers.Main* por se comunicar com a UI). Esse exemplo é bem simples e básico, mas que ilustra o possível problema de troca de contextos. Para resolver casos como esse, usamos o *withContext*.

## withContext

É uma função que recebe um *dispatcher* que será usado para execução do código. Para o exemplo citado acima, poderíamos resolver da seguinte maneira:

<iframe src="https://medium.com/media/58afab85ef5859d2eef7f3f124bb8f7b" allowfullscreen="" frameborder="0" height="238" width="680" title="WithContextExample.kt" class="t u v te ak" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 237.986px;"></iframe>

# Conclusão

Coroutines é uma feature eficiente e prática para trabalharmos com assincronia. No iFood, usamos bastante Coroutines atualmente e um caso de uso impactante foi o que fizemos na inicialização do aplicativo. Nós fizemos com que algumas bibliotecas sejam inicializadas em background e em paralelo, sendo que antes eram inicializadas sincronicamente e na main thread. Com isso, conseguimos reduzir o tempo de inicialização do aplicativo usando Coroutines.

Deixo abaixo alguns links com mais detalhes sobre a implementação de Coroutines. Bons estudos! 🙂