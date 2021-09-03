# Programação assíncrona em Kotlin: dissecando coroutines



Que atire a primeira pedra quem nunca teve problemas com thread starvation, quem nunca se deu mal por bloquear a main thread ou quem nunca se perdeu no famoso Callback Hell.

Programação assíncrona é um dos grandes assuntos do momento e vem ganhando cada vez mais importância, tanto na área de **Back-End** quanto em Front-End.

Ainda utilizamos muito Java aqui na Movile e acreditamos que a solução de concorrência da linguagem é de difícil utilização (e nem estamos comentando a parte de depuração disso tudo).

Além disso, Java não incentiva programação reativa, muito menos non-blocking IO. Tivemos alguns avanços com o Java 8 e suas interfaces “””funcionais”””, mas elas ainda passam longe de serem ideais.

### ***Como será que vivemos com isso então?\***

Primeiro, vamos entender porque iniciar threads por impulso é uma péssima prática no geral:

- Trocas de contexto são excessivamente custosas.

   Para se realizar uma troca de contexto são necessários alguns passos:

  1. 1. salvar um ponteiro da instrução atual

  1. 1. salvar o registrador da CPU (há casos em que esse passo não é necessário)

  1. trocar a pilha de execução (isso necessita de pelo menos uma operação de escrita e algumas de leitura)

Pode parecer bem simples, mas em um sistema sobrecarregado, toda essa permutação de threads pode significar uma série daqueles queridos erros de timeout que todos adoramos.

- - **Threads exigem bloqueio.** Enquanto bloquear a si mesmo é relativamente barato, uma thread concorrendo por recursos do SO vai requerer trocas de contexto adicionais, já que, se posta em espera, ela não consegue continuar sua execução. E não é raro uma thread que conseguiu um recurso para si ter de ser colocada em espera alguns clocks depois pois ela requer o lock de *outro* recurso.

- - **OS dependent.** Uma das definições de thread é *“uma primitiva a nível de SO usada para agendamento”*. Os schedulers de SO são desenhados para uso geral e agendam diferentes tipos de programas. Mas uma thread rodando um encoder de áudio e uma thread recebendo pacotes de dados da sua rede se comportam de maneira diferenciada e o mesmo algoritmo de scheduling pode não ser ótimo para ambos.

- **Uso excessivo de memória.** Cada thread tem sua própria stack, que precisa ser alocada no momento de sua criação. O número de threads que podemos criar em uma aplicação fica limitada pela memória disponibilizada pelo SO para aquele processo. O FAQ da oracle chega até a abordar essa questão:

> ***1) My application has a lot of threads and is running out of memory, why?\***
>
> *A. You may be running into a problem with the default stack size for threads. In Java SE 6, the default on Sparc is* ***512k\*** *in the 32-bit VM, and* ***1024k\*** *in the 64-bit VM.
> **On x86 Solaris/Linux it is* ***320k\*** *in the 32-bit VM and* ***1024k\*** *in the 64-bit VM.
> **On Windows, the default thread stack size is read from the binary (java.exe). As of Java SE 6, this value is* ***320k\*** *in the 32-bit VM and* ***1024k\*** *in the 64-bit VM.
> **You can reduce your stack size by running with the -Xss option.
> **64k is the least amount of stack space allowed per thread.*

##  **Mas será que temos outras alternativas?**

Claro! [Go](https://golang.org/) apresenta uma solução muito bem desenhada para computação assíncrona com suas [Goroutines](https://gobyexample.com/goroutines) e [Channels](https://gobyexample.com/channels); C# com [async/await](https://msdn.microsoft.com/en-us/library/hh191443(v=vs.120).aspx); [Clojure](https://clojure.org/) com seu [core.async](https://github.com/clojure/core.async) e o [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/) do [Node.js](https://nodejs.org/en/). E **inspirado principalmente pelas duas primeiras**, temos, enfim, [Kotlin](https://kotlinlang.org/) com suas [coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md).

##  **Vamos falar sobre Kotlin**

(Se você ainda não conhece Kotlin, o Bernardo Amaral escreveu um [ótimo artigo](https://movile.blog/motivos-para-trocar-o-java-pelo-kotlin-ainda-hoje/) introdutório sobre a linguagem.)

Junto com a versão 1.1, o Kotlin apresentou a feature experimental de coroutines. E claro que quisemos testar aqui na Movile.

A JetBrains gosta de descrevê-las como *“light-weight threads”* (do mesmo jeito que Go trata suas Goroutines).

Coroutines proveem um jeito de executar código de forma assíncrona, sem a maioria dos overheads encontrados nas Threads do Java e de forma muito parecida com código sequencial que todos estamos acostumados. E tudo isso abstraído de forma simples pra você.

Elas também não são mapeadas para threads nativas, ou seja, não tem tantas limitações quanto as famigeradas threads.

Mas calma, não jogue fora sua cópia de [“Java Concurrency in Practice”](http://jcip.net/)!

**Coroutines** representam operações que **esperam** por algo na maior parte do tempo. Como por exemplo: requests HTTP, escrita e leitura a um banco de dados… Em sua grande maioria: operações IO bound

**Threads** ainda são uma boa pedida para tarefas que pedem **poder de processamento** (CPU bound). Exemplos: cálculo de hashes, renderização de gráficos 3D, transcoding de vídeos…

## **A magia por trás**

Coroutines (conceitualmente falando), ao invés de bloquear a thread em que são executadas, **podem suspender** uma operação **a qualquer momento** para que esta seja completada mais tarde (isso pode acontecer até em outra thread).

Funções que podem ser suspensas são chamadas de *“suspending functions”* e compõem o core da API de coroutines.

Suspender uma coroutine basicamente diz à thread em que ela é executada: “hey, estou esperando umas coisas aqui pra poder finalizar essas operações, então pode seguir em frente e passar para a próxima! Depois a gente se fala”

Isso significa que não há necessidade de ter threads paradas esperando, só consumindo recursos. Pelo contrário, você pode criar [100.000 coroutines em uma única thread](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md#coroutines-are-light-weight). [1]

Mas claro, como os cientistas que somos (ou queremos ser), não acreditamos em mágica.

E aí? Como tudo isso é implementado?

A resposta é bem simples: *Continuation-Passing Style* (CPS) ou no seu nome menos formal, *Callbacks*.

## **Um exemplo prático**

Digamos que eu queira escrever uma suspending function que me retorne informações de um usuário da minha base de dados:

```
// a keyword `suspend` define uma suspending function
suspend fun queryUser(id: Int): User {
  delay(100) // simulando latência de rede ou outra operação lenta
  return User(id = id, name = "gente.firme", email = "talentos@movile.com")
}
```

Como isso é visto do ponto de vista da JVM?

- Object queryUser(int id, Continuation<User> cont) { … }

Mas o que é esse Continuation que veio do além?

- Uma interface de Callback:

```
interface Continuation<in T> {
    val context: CoroutineContext
    fun resume(value: T)
    fun resumeWithException(exception: Throwable)
}
```

Então quer dizer que todo código que eu escrever em uma coroutine vai virar um callback? 

- Sim e de um jeito muito parecido com o modo que C# implementa tudo isso.

Vamos continuar nosso exemplo imaginando que queremos enviar um email para esse nosso usuário e caso isso não seja possível, enviamos um sms como fallback:

```
suspend fun queryUser(id: Int): User {
    delay(500) // simulando latência de rede ou outra operação lenta
    return User(id = id, name = "gente.firme", email = "talentos@movile.com", phone = 5511000000000L)
}


suspend fun sendEmail(destination: String, body: String): Boolean {
    println("Sending email to $destination with body: $body")
    delay(1000) // simulando um request/operação lenta
    val couldSentEmail = Random().nextBoolean()
    if (couldSentEmail) {
        println("Successfully sent email!")
    } else {
        println("Uh, something went terribly wrong")
    }
    return couldSentEmail
}


suspend fun sendSMS(destination: Long, message: String): Boolean {
    println("Sending sms to $destination with message: $message")
    delay(500) // simulando um request/operação lenta
    println("Successfully sent SMS!")
    return true
}


suspend fun sendMessageToUser(userId: Int, message: String) {
    val user = queryUser(userId)
    val emailStatus = sendEmail(user.email, message)
    if (!emailStatus) {
        sendSMS(user.phone, message)
    }
}
```

Qual a cara da nossa função *sendMessageToUser* por baixo dos panos?

```
fun sendMessageToUser(userId: Int, message: String, cont: Continuation) {
    val stateMachine = cont as? CoroutineImpl // verifica se já temos uma máquina de estados instanciada (ou seja, já passamos pelo menos uma vez por essa função)
                       ?: object : CoroutineImpl { // cria uma máquina de estados que chama a própria função
        fun resume(…) {
            sendMessageToUser(userId, message, this) // Hooray, Callback
        }
    }


    switch(stateMachine.label) {
    case 0: // Kotlin usa labels para saber em qual passo estamos
        stateMachine.userId = userId
        stateMachine.message = message
        stateMachine.label = 1 // guarda a label do **próximo** passo
        queryUser(userId, stateMachine) // executa a função usando a máquina de estados para guardar o estado
    case 1:
        val user = stateMachine.result as User // Usa o resultado da função que foi chamada anteriormente
        stateMachine.label = 2
        sendEmail(user.email, message, stateMachine)
    …
    }
  …
}
```

**O código acima é fictício**, porém representa bem como as coroutines são implementadas.

Alguns dos pontos principais:

- - A conversão de “código sequencial” (direct style) para callback é feita via **Labels**. Isso facilita, por exemplo, a conversão de loops.

- Os callbacks são controlados por uma **máquina de estados** ([State Machine](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md#state-machines)) ao invés de se criar várias funções novas. Isso retira o overhead de se criar funções anônimas para todas as chamadas.

## 

## **Compartilhando estados**

Bom, você tem duas escolhas: o jeito “clássico”, mais conhecido por “[Shared Mutable State](http://henrikeichenhardt.blogspot.com.br/2013/06/why-shared-mutable-state-is-root-of-all.html)” ou o jeito encorajado pelo Kotlin:

### **“Share by Communication”**

Essa abordagem se baseia nos princípios apresentados pelos Atores de [Erlang](https://www.erlang.org/) (ou se você está mais acostumado com a JVM: [Akka](https://akka.io/)):

- - Sem estados compartilhados

- - Processos leves

- - Envio de mensagens assíncronas

- - Mailboxes fazem o buffer de mensagens recebidas

- Processamento da mailbox com pattern matching

Em resumo: um estado que deveria ser compartilhado entre coroutines é encapsulado e administrado por um [Actor](https://doc.akka.io/docs/akka/current/guide/actors-intro.html?language=scala) (ator).

Um ator é uma combinação de **um estado**, **um comportamento** e **um Channel** para o envio e o recebimento de mensagens. Ele fica encarregado de alterar o estado conforme a mensagem recebida.

Em Kotlin, um ator simples pode ser escrito como uma função, mas para estados mais complexos recomenda-se o uso de uma [classe especializada](https://github.com/Kotlin/kotlinx.coroutines/issues/87):

```
sealed class ActorCommand<out T> {
    class REGISTER<T>(val id: Int, val name: String, val response: SendChannel<T>? = null) : ActorCommand<T>()
    class REMOVE<T>(val id: Int, val response: SendChannel<T>? = null) : ActorCommand<T>()
    class QUERY<T>(val response: SendChannel<T>) : ActorCommand<T>()
}


val actorJob = actor<ActorCommand<Any?>> {
    val state = mutableMapOf<Int, String>()
    for (msg in channel) {
        when (msg) {
            is ActorCommand.REGISTER -> {
                val result = state.getOrPut(msg.id) {
                    msg.name
                }
                msg.response?.send(result)
            }
            is ActorCommand.REMOVE -> {
                val result = state.remove(msg.id)
                msg.response?.send(result)
            }
            is ActorCommand.QUERY -> msg.response.send(state.toMap())
        }
    }
}
```

Como essas mensagens são recebidas sequencialmente, processadas uma a uma e não há necessidade de troca de contexto (não trocamos de thread!), o modelo de Atores resolve o problema de compartilhamento de estados sem os overheads de lock/bloqueio para sincronismo.

A aplicação continua seu flow de execução baseado-se “apenas” na comunicação entre seus atores.

## **Future / Deferred**

Um Future define um valor que inicialmente é desconhecido, pois será definido em um processamento que ainda não acabou.

Em javascript (ECMAScript) temos isso como Promises; Scala nos apresenta tanto Futures (read-only) quanto Promises (writable) e isso também não é nenhuma novidade para os programadores Java:

- Temos os CompletableFutures desde a versão 8;
- Os ListenableFutures do Guava;
- Os Observables do RxJava;
- e mais um monte de implementações via outras libs externas.

Kotlin prefere usar o termo mais abrangente: Deferred. Mas ao invés de definir sua própria API de Future, ele propõe unificar o uso dos Futures da JVM escondendo a implementação e definindo integrações com boa parte das libs mais populares em Java.

Como isso é feito?

```
// define uma função para criar um future
// exemplo de uso:
// val stringFuture = future { “yay” }
// println(stringFuture.get())
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
 CompletableFutureCoroutine<T>(context).also {
 block.startCoroutine(completion = it)
 }
 
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
 override fun resume(value: T) {
 complete(value)
 }
 override fun resumeWithException(exception: Throwable) {
 completeExceptionally(exception)
 }
}
```

Simples né?

Ele também define uma extension function para marcar um Future como suspenso:

```
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) {
                cont.resume(result)
            } else {
                cont.resumeWithException(exception)


            }
        }
    }
```

## **Finalizando…**

Nada disso é novo, tanto em relação aos conceitos quanto em relação ao modo como isso é implementado. Porém o Kotlin nos traz isso de forma abstraída e fácil.

Para as pessoas que não podem sair da JVM e procuram as soluções apresentadas por outras linguagens para programação orientada a eventos e non-blocking IO de forma interoperável e com menos dor de cabeça, essa API experimental pode ser uma boa opção.

Por que não dar uma chance?

[1]. [Aparentemente isso também é possível com Threads e alguns fine-tunings](https://stackoverflow.com/questions/17593699/tcp-ip-solving-the-c10k-with-the-thread-per-client-approach/17771219#17771219)Programação assíncrona em Kotlin: dissecando coroutines

## COMPARTILHE

Compartilhar no facebook

 

Compartilhar no google

 

Compartilhar no twitter

 

Compartilhar no linkedin

Que atire a primeira pedra quem nunca teve problemas com thread starvation, quem nunca se deu mal por bloquear a main thread ou quem nunca se perdeu no famoso Callback Hell.

Programação assíncrona é um dos grandes assuntos do momento e vem ganhando cada vez mais importância, tanto na área de **Back-End** quanto em Front-End.

Ainda utilizamos muito Java aqui na Movile e acreditamos que a solução de concorrência da linguagem é de difícil utilização (e nem estamos comentando a parte de depuração disso tudo).

Além disso, Java não incentiva programação reativa, muito menos non-blocking IO. Tivemos alguns avanços com o Java 8 e suas interfaces “””funcionais”””, mas elas ainda passam longe de serem ideais.

### ***Como será que vivemos com isso então?\***

Primeiro, vamos entender porque iniciar threads por impulso é uma péssima prática no geral:

- Trocas de contexto são excessivamente custosas.

   Para se realizar uma troca de contexto são necessários alguns passos:

  1. 1. salvar um ponteiro da instrução atual

  1. 1. salvar o registrador da CPU (há casos em que esse passo não é necessário)

  1. trocar a pilha de execução (isso necessita de pelo menos uma operação de escrita e algumas de leitura)

Pode parecer bem simples, mas em um sistema sobrecarregado, toda essa permutação de threads pode significar uma série daqueles queridos erros de timeout que todos adoramos.

- - **Threads exigem bloqueio.** Enquanto bloquear a si mesmo é relativamente barato, uma thread concorrendo por recursos do SO vai requerer trocas de contexto adicionais, já que, se posta em espera, ela não consegue continuar sua execução. E não é raro uma thread que conseguiu um recurso para si ter de ser colocada em espera alguns clocks depois pois ela requer o lock de *outro* recurso.

- - **OS dependent.** Uma das definições de thread é *“uma primitiva a nível de SO usada para agendamento”*. Os schedulers de SO são desenhados para uso geral e agendam diferentes tipos de programas. Mas uma thread rodando um encoder de áudio e uma thread recebendo pacotes de dados da sua rede se comportam de maneira diferenciada e o mesmo algoritmo de scheduling pode não ser ótimo para ambos.

- **Uso excessivo de memória.** Cada thread tem sua própria stack, que precisa ser alocada no momento de sua criação. O número de threads que podemos criar em uma aplicação fica limitada pela memória disponibilizada pelo SO para aquele processo. O FAQ da oracle chega até a abordar essa questão:

> ***1) My application has a lot of threads and is running out of memory, why?\***
>
> *A. You may be running into a problem with the default stack size for threads. In Java SE 6, the default on Sparc is* ***512k\*** *in the 32-bit VM, and* ***1024k\*** *in the 64-bit VM.
> **On x86 Solaris/Linux it is* ***320k\*** *in the 32-bit VM and* ***1024k\*** *in the 64-bit VM.
> **On Windows, the default thread stack size is read from the binary (java.exe). As of Java SE 6, this value is* ***320k\*** *in the 32-bit VM and* ***1024k\*** *in the 64-bit VM.
> **You can reduce your stack size by running with the -Xss option.
> **64k is the least amount of stack space allowed per thread.*

##  **Mas será que temos outras alternativas?**

Claro! [Go](https://golang.org/) apresenta uma solução muito bem desenhada para computação assíncrona com suas [Goroutines](https://gobyexample.com/goroutines) e [Channels](https://gobyexample.com/channels); C# com [async/await](https://msdn.microsoft.com/en-us/library/hh191443(v=vs.120).aspx); [Clojure](https://clojure.org/) com seu [core.async](https://github.com/clojure/core.async) e o [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/) do [Node.js](https://nodejs.org/en/). E **inspirado principalmente pelas duas primeiras**, temos, enfim, [Kotlin](https://kotlinlang.org/) com suas [coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md).

##  **Vamos falar sobre Kotlin**

(Se você ainda não conhece Kotlin, o Bernardo Amaral escreveu um [ótimo artigo](https://movile.blog/motivos-para-trocar-o-java-pelo-kotlin-ainda-hoje/) introdutório sobre a linguagem.)

Junto com a versão 1.1, o Kotlin apresentou a feature experimental de coroutines. E claro que quisemos testar aqui na Movile.

A JetBrains gosta de descrevê-las como *“light-weight threads”* (do mesmo jeito que Go trata suas Goroutines).

Coroutines proveem um jeito de executar código de forma assíncrona, sem a maioria dos overheads encontrados nas Threads do Java e de forma muito parecida com código sequencial que todos estamos acostumados. E tudo isso abstraído de forma simples pra você.

Elas também não são mapeadas para threads nativas, ou seja, não tem tantas limitações quanto as famigeradas threads.

Mas calma, não jogue fora sua cópia de [“Java Concurrency in Practice”](http://jcip.net/)!

**Coroutines** representam operações que **esperam** por algo na maior parte do tempo. Como por exemplo: requests HTTP, escrita e leitura a um banco de dados… Em sua grande maioria: operações IO bound

**Threads** ainda são uma boa pedida para tarefas que pedem **poder de processamento** (CPU bound). Exemplos: cálculo de hashes, renderização de gráficos 3D, transcoding de vídeos…

## **A magia por trás**

Coroutines (conceitualmente falando), ao invés de bloquear a thread em que são executadas, **podem suspender** uma operação **a qualquer momento** para que esta seja completada mais tarde (isso pode acontecer até em outra thread).

Funções que podem ser suspensas são chamadas de *“suspending functions”* e compõem o core da API de coroutines.

Suspender uma coroutine basicamente diz à thread em que ela é executada: “hey, estou esperando umas coisas aqui pra poder finalizar essas operações, então pode seguir em frente e passar para a próxima! Depois a gente se fala”

Isso significa que não há necessidade de ter threads paradas esperando, só consumindo recursos. Pelo contrário, você pode criar [100.000 coroutines em uma única thread](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md#coroutines-are-light-weight). [1]

Mas claro, como os cientistas que somos (ou queremos ser), não acreditamos em mágica.

E aí? Como tudo isso é implementado?

A resposta é bem simples: *Continuation-Passing Style* (CPS) ou no seu nome menos formal, *Callbacks*.

## **Um exemplo prático**

Digamos que eu queira escrever uma suspending function que me retorne informações de um usuário da minha base de dados:

```
// a keyword `suspend` define uma suspending function
suspend fun queryUser(id: Int): User {
  delay(100) // simulando latência de rede ou outra operação lenta
  return User(id = id, name = "gente.firme", email = "talentos@movile.com")
}
```

Como isso é visto do ponto de vista da JVM?

- Object queryUser(int id, Continuation<User> cont) { … }

Mas o que é esse Continuation que veio do além?

- Uma interface de Callback:

```
interface Continuation<in T> {
    val context: CoroutineContext
    fun resume(value: T)
    fun resumeWithException(exception: Throwable)
}
```

Então quer dizer que todo código que eu escrever em uma coroutine vai virar um callback? 

- Sim e de um jeito muito parecido com o modo que C# implementa tudo isso.

Vamos continuar nosso exemplo imaginando que queremos enviar um email para esse nosso usuário e caso isso não seja possível, enviamos um sms como fallback:

```
suspend fun queryUser(id: Int): User {
    delay(500) // simulando latência de rede ou outra operação lenta
    return User(id = id, name = "gente.firme", email = "talentos@movile.com", phone = 5511000000000L)
}


suspend fun sendEmail(destination: String, body: String): Boolean {
    println("Sending email to $destination with body: $body")
    delay(1000) // simulando um request/operação lenta
    val couldSentEmail = Random().nextBoolean()
    if (couldSentEmail) {
        println("Successfully sent email!")
    } else {
        println("Uh, something went terribly wrong")
    }
    return couldSentEmail
}


suspend fun sendSMS(destination: Long, message: String): Boolean {
    println("Sending sms to $destination with message: $message")
    delay(500) // simulando um request/operação lenta
    println("Successfully sent SMS!")
    return true
}


suspend fun sendMessageToUser(userId: Int, message: String) {
    val user = queryUser(userId)
    val emailStatus = sendEmail(user.email, message)
    if (!emailStatus) {
        sendSMS(user.phone, message)
    }
}
```

Qual a cara da nossa função *sendMessageToUser* por baixo dos panos?

```
fun sendMessageToUser(userId: Int, message: String, cont: Continuation) {
    val stateMachine = cont as? CoroutineImpl // verifica se já temos uma máquina de estados instanciada (ou seja, já passamos pelo menos uma vez por essa função)
                       ?: object : CoroutineImpl { // cria uma máquina de estados que chama a própria função
        fun resume(…) {
            sendMessageToUser(userId, message, this) // Hooray, Callback
        }
    }


    switch(stateMachine.label) {
    case 0: // Kotlin usa labels para saber em qual passo estamos
        stateMachine.userId = userId
        stateMachine.message = message
        stateMachine.label = 1 // guarda a label do **próximo** passo
        queryUser(userId, stateMachine) // executa a função usando a máquina de estados para guardar o estado
    case 1:
        val user = stateMachine.result as User // Usa o resultado da função que foi chamada anteriormente
        stateMachine.label = 2
        sendEmail(user.email, message, stateMachine)
    …
    }
  …
}
```

**O código acima é fictício**, porém representa bem como as coroutines são implementadas.

Alguns dos pontos principais:

- - A conversão de “código sequencial” (direct style) para callback é feita via **Labels**. Isso facilita, por exemplo, a conversão de loops.

- Os callbacks são controlados por uma **máquina de estados** ([State Machine](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md#state-machines)) ao invés de se criar várias funções novas. Isso retira o overhead de se criar funções anônimas para todas as chamadas.

## 

## **Compartilhando estados**

Bom, você tem duas escolhas: o jeito “clássico”, mais conhecido por “[Shared Mutable State](http://henrikeichenhardt.blogspot.com.br/2013/06/why-shared-mutable-state-is-root-of-all.html)” ou o jeito encorajado pelo Kotlin:

### **“Share by Communication”**

Essa abordagem se baseia nos princípios apresentados pelos Atores de [Erlang](https://www.erlang.org/) (ou se você está mais acostumado com a JVM: [Akka](https://akka.io/)):

- - Sem estados compartilhados

- - Processos leves

- - Envio de mensagens assíncronas

- - Mailboxes fazem o buffer de mensagens recebidas

- Processamento da mailbox com pattern matching

Em resumo: um estado que deveria ser compartilhado entre coroutines é encapsulado e administrado por um [Actor](https://doc.akka.io/docs/akka/current/guide/actors-intro.html?language=scala) (ator).

Um ator é uma combinação de **um estado**, **um comportamento** e **um Channel** para o envio e o recebimento de mensagens. Ele fica encarregado de alterar o estado conforme a mensagem recebida.

Em Kotlin, um ator simples pode ser escrito como uma função, mas para estados mais complexos recomenda-se o uso de uma [classe especializada](https://github.com/Kotlin/kotlinx.coroutines/issues/87):

```
sealed class ActorCommand<out T> {
    class REGISTER<T>(val id: Int, val name: String, val response: SendChannel<T>? = null) : ActorCommand<T>()
    class REMOVE<T>(val id: Int, val response: SendChannel<T>? = null) : ActorCommand<T>()
    class QUERY<T>(val response: SendChannel<T>) : ActorCommand<T>()
}


val actorJob = actor<ActorCommand<Any?>> {
    val state = mutableMapOf<Int, String>()
    for (msg in channel) {
        when (msg) {
            is ActorCommand.REGISTER -> {
                val result = state.getOrPut(msg.id) {
                    msg.name
                }
                msg.response?.send(result)
            }
            is ActorCommand.REMOVE -> {
                val result = state.remove(msg.id)
                msg.response?.send(result)
            }
            is ActorCommand.QUERY -> msg.response.send(state.toMap())
        }
    }
}
```

Como essas mensagens são recebidas sequencialmente, processadas uma a uma e não há necessidade de troca de contexto (não trocamos de thread!), o modelo de Atores resolve o problema de compartilhamento de estados sem os overheads de lock/bloqueio para sincronismo.

A aplicação continua seu flow de execução baseado-se “apenas” na comunicação entre seus atores.

## **Future / Deferred**

Um Future define um valor que inicialmente é desconhecido, pois será definido em um processamento que ainda não acabou.

Em javascript (ECMAScript) temos isso como Promises; Scala nos apresenta tanto Futures (read-only) quanto Promises (writable) e isso também não é nenhuma novidade para os programadores Java:

- Temos os CompletableFutures desde a versão 8;
- Os ListenableFutures do Guava;
- Os Observables do RxJava;
- e mais um monte de implementações via outras libs externas.

Kotlin prefere usar o termo mais abrangente: Deferred. Mas ao invés de definir sua própria API de Future, ele propõe unificar o uso dos Futures da JVM escondendo a implementação e definindo integrações com boa parte das libs mais populares em Java.

Como isso é feito?

```
// define uma função para criar um future
// exemplo de uso:
// val stringFuture = future { “yay” }
// println(stringFuture.get())
fun <T> future(context: CoroutineContext = CommonPool, block: suspend () -> T): CompletableFuture<T> =
 CompletableFutureCoroutine<T>(context).also {
 block.startCoroutine(completion = it)
 }
 
class CompletableFutureCoroutine<T>(override val context: CoroutineContext) : CompletableFuture<T>(), Continuation<T> {
 override fun resume(value: T) {
 complete(value)
 }
 override fun resumeWithException(exception: Throwable) {
 completeExceptionally(exception)
 }
}
```

Simples né?

Ele também define uma extension function para marcar um Future como suspenso:

```
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) {
                cont.resume(result)
            } else {
                cont.resumeWithException(exception)


            }
        }
    }
```

## **Finalizando…**

Nada disso é novo, tanto em relação aos conceitos quanto em relação ao modo como isso é implementado. Porém o Kotlin nos traz isso de forma abstraída e fácil.

Para as pessoas que não podem sair da JVM e procuram as soluções apresentadas por outras linguagens para programação orientada a eventos e non-blocking IO de forma interoperável e com menos dor de cabeça, essa API experimental pode ser uma boa opção.

Por que não dar uma chance?

[1]. [Aparentemente isso também é possível com Threads e alguns fine-tunings](https://stackoverflow.com/questions/17593699/tcp-ip-solving-the-c10k-with-the-thread-per-client-approach/17771219#17771219)