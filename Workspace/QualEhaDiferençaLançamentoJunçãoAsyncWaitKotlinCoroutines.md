# Qual é a diferença entre o lançamento / junção e async / wait em Kotlin coroutines

Na biblioteca `kotlinx.coroutines` você pode iniciar uma nova co-rotina usando `launch` (com `join`) ou `async` (com `await`). Qual a diferença entre eles?

**[asynchronous](https://www.ti-enxame.com/pt/asynchronous/)****[kotlin](https://www.ti-enxame.com/pt/kotlin/)****[coroutine](https://www.ti-enxame.com/pt/coroutine/)****[kotlin-coroutines](https://www.ti-enxame.com/pt/kotlin-coroutines/)**



14 de set. de 2017[Roman Elizarov](https://stackoverflow.com/users/1051598/roman-elizarov)

- [`launch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html) é usado para **disparar e esquecer a co-rotina** . É como começar um novo tópico. Se o código dentro do `launch` terminar com exceção, então ele é tratado como *ncaught* exception em um thread - normalmente impresso em stderr em aplicações JVM de backend e crashes Android applications. [`join`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/join.html) é usado para aguardar a conclusão da co-rotina lançada e não propaga sua exceção. No entanto, uma co-batida com falha *child* cancela seu pai com a exceção correspondente também.
- [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html) é usado para **iniciar uma co-rotina que calcula algum resultado** . O resultado é representado por uma instância de [`Deferred`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html) e você **deve** usar [`await`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/await.html) sobre ele. Uma exceção não capturada dentro do código `async` é armazenada dentro do `Deferred` resultante e não é entregue em nenhum outro lugar, ela será descartada silenciosamente, a menos que seja processada. **Você NÃO DEVE se esquecer da co-rotina iniciada com async** .

 149