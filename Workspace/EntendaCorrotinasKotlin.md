# Entenda as corrotinas do Kotlin



![img](https://ichi.pro/assets/images/max/724/1*c-dfNp_RM7MLP95LhnAwWA.png)Compreendendo as corrotinas de Kotlin

Recentemente, Kotlin Coroutines introduziu uma abordagem avançada e eficiente de simultaneidade, que pode ser usada no Android para simplificar tarefas assíncronas. Na verdade, essa abordagem é muito mais simples, abrangente e robusta em comparação com outras abordagens no Android. Este ensaio tem como objetivo apresentar e discutir alguns conceitos básicos das corrotinas Kotlin para desenvolvedores Android.

## visão global

A programação assíncrona é um meio de programação paralela em que uma unidade de trabalho é executada separadamente da thread principal do aplicativo e notifica a thread de chamada sobre sua conclusão, falha ou progresso. Portanto, pode ser considerado um problema essencial em aplicativos Android avançados porque, como um desenvolvedor Android, você tem que lidar com tarefas caras e pesadas longe do thread de IU. Isso significa que você pode gerenciar algumas tarefas em segundo plano sem evitar congelamentos da IU e experiências de usuário irritantes. O Android oferece suporte a algumas soluções de programação assíncrona tradicionais, como Thread e AsyncTask. No entanto, o problema ainda não foi totalmente resolvido. Em outras palavras, AsyncTask e Thread podem facilmente produzir vazamentos de memória e overheads, e o problema de Callback Hell também é uma das principais consequências negativas na programação tradicional.

Além disso, o Google estudou com precisão algumas práticas recomendadas e alguns comentários sobre simultaneidade recentemente usados por desenvolvedores Android. Na verdade, a abordagem de simultaneidade moderna no Android é classificada em três categorias pelo Google para desenvolvedores: Coroutines (cálculos com capacidade de suspensão), RxKotlin ou RxJava (Schedulers, Observer e Observable) e LiveData (Detentor de dados observáveis). Primeiro, Kotlin Coroutines apresenta uma nova abordagem de simultaneidade, que pode ser usada no Android para simplificar as tarefas Async. Além disso, os desenvolvedores indicaram que essa forma poderia ser a melhor solução no Android. Em segundo lugar, embora RxKotlin ou RxJava seja um dos mecanismos eficientes do Android para simultaneidade e programação assíncrona, leva muito tempo para conhecê-lo e usá-lo efetivamente na prática pelos desenvolvedores. Terceiro, os desenvolvedores mencionaram naquele estudo, que LiveData poderia ser uma abordagem útil para simultaneidade no Android, mas eles querem uma solução completa para resolver esse problema.



Eventualmente, a melhor solução deve seguir três princípios principais na visualização do Google, e eles acreditam que Kotlin Coroutines fornece as seguintes idéias:

1. Simples: deve ser uma solução fácil de aprender.
2. Abrangente: deve ser uma solução que abranja todos os casos e soluções.
3. Robusto: deve ser uma história de provocação embutida.

Fundamentalmente, as corrotinas Kotlin usam o conceito de programação de estilo de passagem de continuação (CPS). Este método de programação inclui passar o fluxo de controle do programa como um argumento para funções. Este argumento é denominado Continuação em Kotlin. Em uma palavra, uma continuação é semelhante a um retorno de chamada, mas geralmente é mais no nível do sistema. Além disso, as corrotinas podem ser suspensas e reiniciadas. Isso significa que você pode ter uma tarefa de longa duração que pode ser executada gradualmente. Portanto, você pode pausá-lo quantas vezes quiser e retomá-lo quando quiser usá-lo novamente. Uma observação importante é que o uso de várias corrotinas Kotlin não produzirá sobrecargas de memória para seu aplicativo. Suspenda e retome o trabalho um com o outro para substituir os retornos de chamada tradicionais. Para obter mais informações, se uma corrotina for suspensa, o frame da pilha atual em Kotlin será copiado e salvo para usos futuros. Quando ele é reiniciado, o stack frame é copiado de onde estava armazenado e começa a ser executado novamente. Além disso, quando todas as corrotinas são suspensas, o thread principal fica livre para seus trabalhos de rotina. Além desta seção, main-safety é outro recurso-chave das corrotinas Kotlin. Resumindo, isso significa que as funções de suspensão apropriadas sempre podem ser chamadas com segurança a partir do thread principal. Eles devem sempre permitir que qualquer thread os chame sem saber o que eles querem fazer.

***Corrotinas = Rotinas Co +\***

**Co** significa cooperação e **Rotinas** significa funções **.** Isso significa que, quando as funções cooperam entre si, nós o chamamos de corrotinas.

## Suspensão de funções

Suspender funções pode suspender a execução da co-rotina atual sem bloquear a thread atual. Além disso, em vez de retornar um valor, ele sabe em qual contexto o chamador o suspendeu. Assim, ao usar esse recurso, ele pode retomar apropriadamente quando estiver pronto. Isso ajuda ainda mais nas otimizações de memória da CPU e na multitarefa. Apesar de bloquear um thread, as funções de suspensão são menos caras porque não precisam de uma troca de contexto. Uma função de suspensão pode ser criada adicionando a palavra-chave suspender a uma função. Uma função de suspensão pode ser chamada apenas a partir de uma co-rotina ou de outra função de suspensão. Por exemplo:



```
suspend fun sampleSuspend(){
println("Kotlin Coroutines")
}
```



Basicamente, as funções de suspensão não podem ser chamadas de funções regulares. Portanto, a biblioteca fornece um conjunto de funções para iniciar as corrotinas a partir delas. Existem muitos construtores Coroutine fornecidos por Kotlin Coroutines, incluindo:

- ***lançamento\*** : Isso cria uma nova co-rotina. Ele apenas dispara e se esquece, e não espera pela resposta. Se ocorrer uma exceção não detectada, isso abrupta o fluxo do programa.
- ***assíncrono\*** : dispara e aguarda a resposta.
- ***runBlocking\*** : é semelhante ao launch, exceto que dentro de um runBlocking tudo estaria na mesma co-rotina.
- ***run\*** : Esta é uma corrotina básica.
- ***CoroutineScope\*** : ajuda a definir o ciclo de vida das corrotinas Kotlin. Pode ser em todo o aplicativo ou vinculado a uma atividade.
- ***Dispatchers\*** : define pools de threads para iniciar suas corrotinas Kotlin.

1. *construtor de lançamento* : ele não espera pela resposta.
2. *construtor assíncrono* : iniciará uma nova co-rotina e permite retornar um resultado (retorna uma instância de ***Deffered <T>\*** ), com uma função suspender chamada await.



```
suspend fun sampleSuspend() {
println("Kotlin Coroutines")
}
fun main(args: Array<String>) = runBlocking {
println("Start")
    val x = launch(CommonPool) {
        delay(2000)   
        sampleSuspend()
        println("launch")
    }
    println("Finish")
    x.join()   //The sampleSuspend method does not run because the function just only fires and forgets. We should use the join function, which waits for the completion.
}
```

A diferença entre ***join\*** e ***await\*** é a espera de junção para a conclusão do lançamento. aguarde procura o resultado retornado.

## Despachantes

Todas as corrotinas Kotlin devem ser executadas em um dispatcher, mesmo quando estão em execução no thread principal. As corrotinas podem se suspender, e o despachante é o elemento que sabe retomá-las. Você pode usar esses despachantes para as seguintes situações:

***1.Dispatchers.Default\*** *:* ***Trabalho que consome muita\*** CPU, como classificar grandes listas e realizar cálculos complexos.



***2. Dispatchers.IO\*** *:* rede ou leitura e gravação de arquivos (qualquer entrada e saída)



***3. Dispatchers.Main\*** *:* dispatcher recomendado para realizar eventos relacionados à IU, como mostrar listas em um RecyclerView, atualizar a visualização e assim por diante.

Por exemplo:



```
suspend fun fetchUser(): User {
return GlobalScope.async(Dispatchers.IO) {
        // make network call
        // return user
    }.await()
}
```

Os escopos nas corrotinas do Kotlin são extremamente úteis porque temos que cancelar a tarefa em segundo plano assim que a atividade for destruída. As corrotinas do Kotlin devem ser executadas em um elemento, que é denominado CoroutinesScope. Um CoroutinesScope mantém o controle de suas corrotinas, até mesmo corrotinas que estão suspensas. Em outras palavras, CoroutinesScope apenas garante que você os mantenha rastreados e cancela todas as corrotinas iniciadas nele.

Quando precisamos usar o escopo global, que é o nosso escopo de aplicativo, não o escopo da atividade, podemos usar o GlobalScope.

## **withContext**

withContext é uma outra maneira de escrever o async, onde não precisamos escrever **await ()** . Por exemplo:



```
suspend fun fetchUser(): User {
return withContext(Dispatchers.IO) {
        // make network call
        // return user
    }
}
```