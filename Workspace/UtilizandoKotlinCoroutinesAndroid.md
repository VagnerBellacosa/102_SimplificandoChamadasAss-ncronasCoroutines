# Utilizando Kotlin Coroutines no Android

[![Wellington Costa](https://miro.medium.com/fit/c/96/96/1*nx3SuSMTE_5q2HG7BgxjXA.jpeg)](https://medium.com/@wellingtoncosta128?source=post_page-----c73fcda71e27-----------------------------------)- [Wellington Costa](https://medium.com/@wellingtoncosta128?source=post_page-----c73fcda71e27-----------------------------------) - [Feb 12, 2019](https://medium.com/android-dev-br/utilizando-kotlin-coroutines-no-android-c73fcda71e27?source=post_page-----c73fcda71e27-----------------------------------) · 6 min read



Como trabalhar com operações assíncrona no Android de forma descomplicada e sem muitos mistérios.

![img](https://miro.medium.com/max/1400/1*J9C7HXwLafBDpROhUP0vxw.png)

Sabemos que aplicativos que apresentam uma boa *performance* e uma boa experiência de uso são bem recebidos pelo público-alvo. Não bloquear a interface de usuário enquanto uma operação está sendo realizada, como o carregamento de informações do banco de dados ou de um serviço *web* por exemplo, é uma boa prática que impacta diretamente na experiência de uso. E com o crescimento da *internet* e da quantidade de dados que são trafegados, se torna inviável realizar operações bloqueantes.

Dentre as várias formas de implementar operações assíncronas, o *Kotlin* trouxe a capacidade de realizar operações suspensíveis e não bloqueantes, chamada de *coroutines*.

Durante esse artigo iremos abordar as *suspending functions*, o conceito de c*oroutines*, os d*ispatchers*, que representam os contextos de execução de uma c*oroutine*, como paralelizar processamento com c*oroutines*, e por fim iremos demonstrar um exemplo prático no Android.

Antes de começar, recomendamos uma leitura sobre o conceito de [*thread*](https://pt.wikipedia.org/wiki/Thread_(computação)) caso o leitor não tenha muito conhecimento a respeito.

# Suspending Functions

*Suspending functions* são funções que podem ser pausadas e resumidas durante sua execução sem bloquear a *thread*, até que sejam finalizadas.

![img](https://miro.medium.com/max/1000/1*D0L3qxMBqzfYVsQ5xTB5JQ.jpeg)

Mas como assim suspender sem bloquear?

Conceitualmente falando, uma operação bloqueante, como o próprio nome sugere, tem o comportamento de bloquear a t*hread* até que a operação seja finalizada, seja com sucesso ou porque uma exceção foi lançada. Em contra partida, operações suspensíveis caracterizam execuções que pode ser pausadas e retomadas em outro momento sem bloquear a *thread*, permitindo [*multitasking*](https://en.wikipedia.org/wiki/Computer_multitasking). Trazendo isso para o mundo Android, operações bloqueantes realizadas na *main thread* irão congelar a tela até que a operação seja finaliza, já as operações suspensíveis não terão o comportamento de congelar a tela.

Para criar uma *suspending function*, é necessário apenas adicionar a palavra reservada ***suspend\*** na declaração da função:

<iframe src="https://medium.com/media/9272e9add1903127630cd7282b747550" allowfullscreen="" frameborder="0" height="106" width="680" title="suspend_function.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 105.99px;"></iframe>

Funções marcadas com o modificador ***suspend\*** só podem ser chamadas por outras ***suspending functions\*** ou dentro de uma ***coroutine\***.

# Mas afinal, o que é uma coroutine?

*Coroutines* são em sua essência t*hreads* mais leves que consomem menos recursos computacionais e que são destinadas a execução de tarefas paralelas e não bloqueantes. As c*oroutines* podem ser executadas em determinados contextos de acordo com o tipo de operação a ser executada, seja manipulando operações de *I/O*, processamento que envolve *CPU* ou manipulação de eventos de *UI*, mas detalharemos isso mais adiante.

Um exemplo de uma *coroutine*:

<iframe src="https://medium.com/media/a30a8b13a52ce471cbc34a3d019ea970" allowfullscreen="" frameborder="0" height="172" width="680" title="coroutine_example.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 171.997px;"></iframe>

Para criar uma *coroutine*, existem duas funções para fazer isso, [*launch*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) e [*async*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html).

## launch

A função *launch* cria uma *coroutine* e inicia esta em *background* e sem bloquear a *thread* a qual ela está associada. Caso não seja passado qual contexto onde a *coroutine* irá executar, ela herdará o contexto de onde ela está sendo inicializada. O tipo de retorno da função *launch* é do tipo [*Job*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html), e este não produz nenhum tipo de retorno ao final da execução.

<iframe src="https://medium.com/media/9e256c05e1b1da119df9b9f15a83c975" allowfullscreen="" frameborder="0" height="150" width="680" title="launch_coroutine.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 149.983px;"></iframe>

## async

Para operações onde existe a necessidade de ter um resultado ao final da execução existe a função *async*. Assim como o *launch*, o *async* é uma *coroutine* em *background* que possui o mesmo ciclo de vida, pode ser executada em um contexto diferente, mas que produz um resultado ao final da execução. O tipo de retorno da função *async* é o tipo [*Deferred*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html), e este possui uma função chamada [*await()*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html) que retorna o resultado produzido ao final da execução da função *async*.

Exemplo do *async*:

<iframe src="https://medium.com/media/d0b721f664e6e594f4e43e36449c2c88" allowfullscreen="" frameborder="0" height="568" width="680" title="coroutine_async_example.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 567.986px;"></iframe>

## withContext

Para executar uma *coroutine* em um contexto específico, como uma operação que faz acesso ao disco ou que realiza uma série de processamentos matemáticos, existem os [*Dispatchers*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/index.html) e a função [*withContext*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html).

Existem os seguintes *Dispatchers*:

- **IO**: para operações de entrada / saída de dados
- **Default**: designado para operações que utilizam mais recursos de *CPU*
- **Main**: utilizado em operações que manipulam componentes de *UI*
- **Unconfined**: destinado para operações onde não existe a real necessidade de serem executadas em *thread* específica

E para aplicar um determinado *dispatcher* a uma *coroutine*, se faz necessário o uso da função *withContext* como mostra a seguir:

<iframe src="https://medium.com/media/cbf6a7e92f4b2e372729ab908bb07e28" allowfullscreen="" frameborder="0" height="238" width="680" title="with_context_coroutine.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 237.986px;"></iframe>

## coroutineScope

Existem situações onde se faz necessário a paralelização de operações, como o carregamento dos dados de um usuário do Github e seus respectivos repositórios, e isso pode ser muito fácil de fazer com *coroutines*:

<iframe src="https://medium.com/media/5270c33a7127bf6484c0a182c9d24548" allowfullscreen="" frameborder="0" height="193" width="680" title="coroutine_scope_coroutines.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 192.986px;"></iframe>

O ponto chave para permitir essa paralelização dos dados de forma correta é a função [*coroutineScope*](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html), pois ela cria um escopo pai para os *asyncs* que estão rodando em paralelo, e isso quer dizer que se algum *async* falhar, seja por falha de rede ou falha do serviço ou qualquer erro que seja, uma *exception* será lançada e essa irá subir para o escopo pai do *coroutineScope*, onde comportamento será de cancelar a outra operação que ainda pode estar em execução.

# Aplicando Coroutines no Android

Durante esta seção iremos exemplificar o uso de Kotlin Coroutines no Android na construção de uma app que consulta dados dos usuários do Github através da API publica deles.

Primeiro de tudo vamos definir nossa *interface* para se comunicar com a API do Github, e para isso vamos utilizar o *Retrofit*:

<iframe src="https://medium.com/media/ecc8a9de0ab55d61354455722820ed65" allowfullscreen="" frameborder="0" height="238" width="680" title="github_api.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 237.986px;"></iframe>

Note que o tipo de retorno das funções é ***Deferred\***, e esse é o tipo de retorno da função *async*, a qual permite criar *coroutines* com um retorno ao final da execução, como foi dito anteriormente.

A seguir vamos demonstrar como fica a camada de repositório da app:

<iframe src="https://medium.com/media/b1aac9d2396c521e52ea033be9e961a9" allowfullscreen="" frameborder="0" height="193" width="680" title="user_repository_interface.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 192.986px;"></iframe>

E a implementação:

<iframe src="https://medium.com/media/6301d3d6e90a667f12cdd8d106cfa48d" allowfullscreen="" frameborder="0" height="326" width="680" title="user_data_repository.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 325.99px;"></iframe>

Note que em cada função estamos dizendo em qual contexto ela deverá executar, no caso serão executadas no contexto de **IO** pelo fato de serem operações que envolvem leitura de dados.

E agora vem a implementação da nossa camada de *ViewModel*. Resolvemos criar uma implementação base que engloba as características necessárias que qualquer *ViewModel* deverá ter:

<iframe src="https://medium.com/media/8ae8ae52ce45c6b3fbaa2d2b0ac85031" allowfullscreen="" frameborder="0" height="348" width="680" title="base_coroutine_viewmodel.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 347.986px;"></iframe>

Essa implementação implementa a *interface* *CoroutineScope* que basicamente serve para dizer em qual contexto as *coroutines* daquele escopo serão executadas.

Na linha 3 estamos sobrescrevendo o valor padrão da propriedade *coroutineContext* para que as *coroutines* daquele contexto sejam executadas na *main thread* do Android.

Na linha 5 estamos definindo uma propriedade que representa uma lista do tipo ***Job\***, onde este é o tipo de retorno das funções *launch*.

Na linha 7 está sendo declarado uma [função infixa](https://kotlinlang.org/docs/reference/functions.html#infix-notation) para facilitar a adição de uma *launch coroutine* a lista de *jobs*.

Por fim, na linha 9 apresenta-se a sobrescrita da função *onCleared()* que é definida na classe *ViewModel*, e esta função é executada sempre que o *ViewModel* chega ao seu fim do ciclo de vida. Neste caso estamos sobrescrevendo para que todos os *jobs* que ainda não foram cancelados sejam cancelados, evitando possíveis *memory leaks*.

E por fim, temos a implementação do *ViewModel* que busca os usuários:

<iframe src="https://medium.com/media/843f9cae688a76a6e5edb605ec7f3c03" allowfullscreen="" frameborder="0" height="1007" width="680" title="users_viewmodel.kt" class="t u v si aj" scrolling="auto" style="box-sizing: inherit; position: absolute; top: 0px; left: 0px; width: 680px; height: 1007px;"></iframe>

Estamos utilizando *LiveData* na camada de *ViewModel* para observar e aplicar mudanças na *View*.

Nas linhas 5, 6 e 7 estamos definindo nossas propriedades mutáveis que serão utilizadas internamente no *ViewModel* para guardar o estado dos dados.

Nas linhas 9, 10 e 11 estamos expondo as respectivas propriedades definidas nas linhas 5, 6 e 7, porém encapsuladas em funções e imutáveis.

Na linha 13 está sendo definido a função *getAll()*, onde esta busca todos os usuários no *UserRepository* e muda os valores das propriedades do *ViewModel* de acordo com a execução.

E por fim, na linha 28 apresenta-se a função *getByUsername()*, onde esta busca um usuário específico pelo seu username.

Vale ressaltar que todas as funções desse *ViewModel* executam *coroutines* do tipo *launch*, e essas serão executadas no contexto da *main thread* do Android como foi definido no *CoroutineViewModel*.

Então é isso, pessoal. Espero que este artigo ajude em dúvidas ou curiosidades sobre como utilizar Kotlin Coroutines no Android. Qualquer dúvidas deixe um comentário.

Os códigos deste artigo estão disponíveis no meu [Github](https://github.com/WellingtonCosta/android-kotlin-coroutines).

Até a próxima!