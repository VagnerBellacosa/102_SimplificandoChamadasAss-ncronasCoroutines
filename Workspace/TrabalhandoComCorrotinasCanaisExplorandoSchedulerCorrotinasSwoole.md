# Trabalhando com corrotinas, canais e explorando um pouco mais o scheduler de corrotinas do Swoole

Neste artigo trabalharemos com os conceitos fundamentais de corrotinas, canais, defer etc, também exploraremos algumas opções do scheduler como, por exemplo, fazê-lo operar de forma preemptiva.

 mais de 2 anos atrás

[Artigos](https://www.treinaweb.com.br/blog)Trabalhando com corrotinas, canais e explorando um pouco mais o scheduler de corrotinas do Swoole

Neste artigo veremos de forma prática os aspectos essenciais do modelo de programação concorrente CSP (communicating sequential processes) com Swoole, usando Coroutine (corrotina), Channel (canal) e Defer (execução tardia). Se você já programou em Go verá muitas similaridades.

Antes, entretanto, é fundamental que você leia o artigo [Introdução ao Swoole, framework PHP assíncrono baseado em corrotinas](https://www.treinaweb.com.br/blog/introducao-ao-swoole-framework-php-assincrono-baseado-em-corrotinas), pois ele introduz toda a teoria fundamental para que possamos criar os nossos primeiros exemplos e adentrar um pouco mais nas possibilidades que o Swoole nos oferece.



![PHP Avançado](https://d2knvm16wkt3ia.cloudfront.net/assets/svg-icon/php.svg)

##### CursoPHP Avançado

[Conhecer o curso](https://www.treinaweb.com.br/curso/php-avancado)



Em uma execução sequencial e síncrona de duas funções, teríamos:

Copiar

```php
<?php

function a() {
    sleep(1);
    echo 'a';
}

function b() {
    sleep(2);
    echo 'b';
}

a();
b();
```

O resultado é bem previsível, aguarda um segundo, imprime `a`, aguarda dois segundos e imprime `b`.

Para que possamos executar uma tarefa dentro de uma corrotina, usamos a função `go()`. O exemplo acima poderia ser reescrito para:

Copiar

```php
<?php

go(static function () {
    sleep(1);
    echo 'a';
});

go(static function () {
    sleep(2);
    echo 'b';
});
```

O problema que temos agora é que a função `sleep()` do [PHP](https://www.treinaweb.com.br/blog/o-que-e-php/) é bloqueante, assim como são as funções de stream, por exemplo.

*Recomendação de leitura:* [Streams no PHP](https://www.treinaweb.com.br/blog/streams-no-php/)

Esse exemplo terá o exato mesmo comportamento que o anterior. Ele demorará três segundos pra finalizar a sua execução. Podemos resolver isso de duas formas, sendo que a primeira é adicionando a instrução `Swoole\Runtime::enableCoroutine();` no exemplo:

Copiar

```php
<?php

Swoole\Runtime::enableCoroutine();

go(static function () {
    sleep(1);
    echo 'a';
});

go(static function () {
    sleep(2);
    echo 'b';
});
```

Este é um hook “mágico” que fará com que o Swoole execute algumas funções que são nativamente síncronas mas de forma assíncrona (não bloqueante). E isso vale para a `sleep()`, como vale para as funções relacionadas a streams.

Agora sim, esse exemplo será executado em dois segundos. Ao invés da execução consumir a soma dos dois tempos das corrotinas, ela passa a consumir o tempo da maior.

Então, temos a seguinte relação:

- No modelo síncrono gasta-se o tempo de: **(a + b)**
- No modelo concorrente gasta-se: **MAX(a, b)**

A outra forma de resolver o problema anterior sem que precisemos aplicar o hook `enableCoroutine()`, é executando a função `sleep()` assíncrona da API do Swoole:

Copiar

```php
<?php

use Swoole\Coroutine\System;

go(static function () {
    System::sleep(1);
    echo 'a';
});

go(static function () {
    System::sleep(2);
    echo 'b';
});
```

Outras funções disponíveis na [API](https://www.treinaweb.com.br/blog/o-que-e-uma-api/) de corrotinas:

Copiar

```php
System::sleep(100);
System::fread($fp);
System::gethostbyname('www.google.com');
// Entre outras
```

Você verá muitos `System::sleep()` até o final desse artigo, pois é uma forma de emular uma operação de I/O (que é sabido que é mais custosa que uma operação de CPU).

**Executando os exemplos**

Se você usa Linux ou macOS pode instalar o Swoole diretamente no seu ambiente:

https://www.swoole.co.uk/docs/get-started/installation

Ou você pode usar o Docker, que é a opção escolhida desse artigo. Algumas das imagens disponíveis para o Swoole:

- https://github.com/deminy/swoole-by-examples
- https://github.com/roquie/docker-swoole-webapp
- https://github.com/leocavalcante/dwoole
- https://github.com/inhere/dockerenv

Para esse artigo eu estou usando como base a imagem do `swoole-by-examples`. Todos os exemplos desse artigo estão disponíveis nesse repositório:

https://github.com/KennedyTedesco/swoole-coroutines

Basta que você clone-o em seu computador e então execute o comando abaixo para inicializar o container:



![Docker - Fundamentos](https://d2knvm16wkt3ia.cloudfront.net/assets/svg-icon/docker.svg)

##### CursoDocker - Fundamentos

[Conhecer o curso](https://www.treinaweb.com.br/curso/docker-fundamentos)



Copiar

```bash
$ docker-compose up -d
```

E para executar o exemplo anteriormente criado:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co1.php"
```

O resultado no terminal será:

Copiar

```
ab
real	0m2.031s
user	0m0.010s
sys	0m0.010s
```

Usamos `time` na execução para que possamos ter a informação do tempo gasto.

Uma coisa importante de se pontuar é que esse projeto tem como dependência no composer o [swoole-ide-helper](https://github.com/swoft-cloud/swoole-ide-helper) que ajuda a sua IDE ou editor de código reconhecer as assinaturas das classes e métodos do Swoole. Mas é bom sempre lembrar que a [documentação](https://www.swoole.co.uk/docs/) é outro ótimo lugar para conhecer outros detalhes e características das APIs.

**Voltando …**

Um importante conceito de concorrência é que não é sobre execução ordenada, a ordem de execução das tarefas não é garantida, são vários os fatores que influenciam. Então, o observe o exemplo abaixo em que executamos 5000 corrotinas:

Copiar

```php
<?php

use Swoole\Coroutine\System;

for ($i = 0; $i < 5000; $i++) {
    go(static function () use ($i) {
        System::sleep(1);
        echo "$i\n";
    });
}
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co2.php"
```

Sempre que você executá-lo, terá um retorno diferente. Nesse exemplo criamos 5000 corrotinas que foram executadas em cerca de 1s. Não fossem executadas de forma concorrente gastaríamos 5000 segundos.

**Outras formas de criar corrotinas**

A função `go()` é muito conveniente para a criação de corrotinas, bastando que passemos para ela uma função anônima representando a tarefa. No entanto, existem outras formas de utilizá-la. O primeiro parâmetro dela espera por um `callable`:

Copiar

```php
/**
 * @param callable $func
 * @param ...$params
 * @return mixed
 */
function go(callable $func, ...$params){}
```

Portanto, poderíamos passar o nome de uma função:

Copiar

```php
<?php

use Swoole\Coroutine\System;

function someTask(int $i) : void {
    System::sleep(1);

    echo "$i\n";
}

for ($i = 0; $i < 1000; $i++) {
    go('someTask', $i);
}
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co3.php"
```

Como também poderíamos passar a instância de um objeto invocável:

Copiar

```php
<?php

use Swoole\Coroutine\System;

final class SomeTask
{
    public function __invoke(int $i): void
    {
        System::sleep(1);

        echo "$i\n";
    }
}

for ($i = 0; $i < 1000; $i++) {
    go(new SomeTask, $i);
}
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co4.php"
```

E as outras formas possíveis são:

Copiar

```php
<?php

use Swoole\Coroutine\System;

function someTask(int $i): void {
    System::sleep(1);

    echo "$i\n";
}

co::create('someTask', 1);

swoole_coroutine_create('someTask', 2);

Swoole\Coroutine::create('someTask', 3);
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co5.php"
```

E todas elas aceitam um valor `callable`, são formas alternativas a `go()`.

Outro conceito importante sobre corrotinas no Swoole é que o *scheduler* delas não é *multi-threaded* como em Go. Apenas uma corrotina é executada por vez, não são executadas em paralelo. Por exemplo, se temos duas tarefas e a tarefa 1 é executada, se tem um `sleep(1)` nela, essa tarefa é pausada e então a tarefa 2 é executada, depois o *scheduler* volta para a tarefa 1. Eventos de I/O pausam/resumem a execução das corrotinas a todo instante.

**Canais**

Outro ponto fundamental do modelo CSP são os canais. As corrotinas representam as atividades do programa e os canais representam as conexões entre elas. Um canal é basicamente um sistema de comunicação que permite uma corrotina enviar valores para outra. Em Go um canal precisa ter um tipo especificado previamente, enquanto que no Swoole podemos armazenar qualquer tipo de dado.

Um exemplo:

Copiar

```php
<?php

use Swoole\Coroutine\System;
use Swoole\Coroutine\Channel;

$chan = new Channel();

go(static function () use ($chan) {
    // Cria 10.000 corrotinas
    for ($i = 0; $i < 10000; $i++) {
        go(static function () use ($i, $chan) {
            // Emula uma operação de I/O
            System::sleep(1);

            // Adiciona o valor processado no canal
            $chan->push([
                'index' => $i,
                'value' => random_int(1, 10000),
            ]);
        });
    }
});

go(static function () use ($chan) {
    while (true) {
        $data = $chan->pop();
        echo "{$data['index']} -> {$data['value']}\n";
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./chan1.php"
```

Usamos o método `push()` para adicionar um item no canal, que no caso foi um array, mas poderia ser um inteiro, uma string etc. E usamos `pop()` para extrair um valor do canal. O `while(true)` dentro dessa corrotina em especial não é um problema, nisso que estamos realizando uma operação no canal, o estado dessa corrotina é controlado, ela não toma pra ela todo o tempo da CPU. Mas veremos mais adiante que operações pesadas de CPU podem impedir que outras corrotinas tenham a chance de serem executadas, mas isso pode ser resolvido se ativarmos o scheduler preemptivo do Swoole.

**Avaliando URLs de forma concorrente**

Um dos bons exemplos para visualizarmos na prática concorrência é quando envolvemos operações de rede na jogada. O exemplo que veremos a seguir, apesar de não tão sofisticado, foi desenvolvido para que possamos fazer uso de corrotinas, canais e defer.

Copiar

```php
<?php

use Swoole\Coroutine\Channel;
use Swoole\Coroutine\System;
use Swoole\Coroutine\Http\Client;

function httpHead(string $url) {
    $client = new Client($url, 80);
    $client->get('/');

    return $client;
}

$chan = new Channel();

go(static function () use ($chan) {
    // Abre um ponteiro para o arquivo
    $fp = fopen('sites.txt', 'rb');

    // Atrasa o fechamento do ponteiro do arquivo para o final da corrotina
    defer(static function () use ($fp) {
        fclose($fp);
    });

    while (feof($fp) === false) {
        // Lê linha a linha do arquivo
        $url = trim(System::fgets($fp));

        if ($url !== '') {
            // Cria uma corrotina para requisitar a URL e trazer o status code dela
            go(static function () use ($url, $chan) {
                $response = httpHead($url);

                // Insere no canal a resposta
                $chan->push([
                    'url' => $url,
                    'statusCode' => $response->statusCode,
                ]);
            });
        }
    }
});

// Corrotina que lê os valores do canal e imprime no output
go(static function () use ($chan) {
    while (true) {
        $data = $chan->pop();
        echo "{$data['url']} -> {$data['statusCode']}\n";
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./http1.php"
```

Na função `httpHead()` estamos usando o cliente HTTP de corrotina do Swoole, a documentação dele pode ser [consultada aqui](https://www.swoole.co.uk/docs/modules/swoole-coroutine-http-client).

Na primeira corrotina abrimos um ponteiro para o arquivo onde as URLs estão localizadas. A função `defer()` define uma tarefa para ser executada ao final da corrotina, então a estamos utilizamos para fechar o ponteiro de arquivo aberto anteriormente.

Iteramos sobre cada linha do arquivo usando a função assíncrona `co::fgets()` da própria API de corrotina e então, pra cada URL, criamos uma nova corrotina para fazer uma requisição `HEAD` e obter o código http da resposta. Essa corrotina envia para um canal o resultado, canal este que é utilizado pela segunda corrotina, que imprime todos os valores contidos nele.

O cliente HTTP padrão do Swoole não possui uma API muito rica e não é tão intuitivo de se usar, para isso existe a biblioteca **[saber](https://github.com/swlib/saber)** que encapsula toda a parte complicada, oferecendo uma API bem intuitiva e de alto nível para se trabalhar com requisições http concorrentes. Se você tiver interesse em praticar, recomendo alterar o exemplo anterior para usar a *saber*.

**E como ficam as tarefas que fazem um uso intensivo de CPU?**

Corrotinas são conhecidas por operarem por cooperação (a tarefa é dona do seu ciclo de vida, tendo o poder de se liberar do *scheduler* no fim de sua operação) em detrimento à preempção.

Esse diagrama ilustra melhor esse cenário:

![img](https://dkrn4sk0rn31v.cloudfront.net/2019/11/04124545/Corrotinas.png)

Enquanto nossas tarefas fazem mais uso de I/O que de CPU, tá tudo bem, pois deixamos os reactors fazerem a mágica. Agora, e se tivermos tarefas de uso pesado de CPU? O modo padrão do *scheduler* funcionar pode não ser o mais “justo” dependendo do caso, por exemplo:

Copiar

```php
<?php

use Swoole\Coroutine\System;

// Tarefa 1
go(static function() {
    System::sleep(1);
    for ($i = 0; $i <= 10; $i++) {
        echo "N{$i}";
    }
});

// Tarefa 2
go(static function() {
    $i = 0;
    while (true) {
        $i++;
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./scheduler1.php"
```

Ao executar esse exemplo, você notará que a primeira tarefa não terá a oportunidade de executar a sua lógica de imprimir N1, N2 etc, pois quando ela é despachada pelo scheduler para um worker, a primeira linha dela é `System::sleep(1);` que simula uma operação de I/O, isso faz com que ela seja pausada para que outra tarefa da fila seja executada. O problema é que a tarefa 2 não é muito espirituosa, ela fica num loop infinito incrementando uma variável, com isso, ela não deixa nenhuma oportunidade para que a outra tarefa irmã seja executada, ou seja, ela não é tão colaborativa assim.

Já sabemos que uma tarefa é pausada quando ela está aguardando por alguma operação de I/O para dar oportunidade a outra tarefa desempenhar o seu trabalho. Podemos emular isso na prática usando como base o exemplo anterior:

Copiar

```php
<?php

use Swoole\Coroutine\System;

// Tarefa 1
go(static function() {
    System::sleep(1);
    for ($i = 0; $i <= 10; $i++) {
        echo "N{$i}";
    }
});

// Tarefa 2
go(static function() {
    $i = 0;
    while (true) {
        $i++;

        // Quando estiver no centésimo loop, emula uma operação de I/O
        if ($i === 100) {
            echo "{$i} -> ";

            System::sleep(1);
        }
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./scheduler2.php"
```

O resultado da execução desse exemplo é:

Copiar

```
100 -> N0N1N2N3N4N5N6N7N8N9N10
```

No centésimo loop emulamos uma operação de I/O de 1 segundo, que fez com que a tarefa fosse pausada dando oportunidade para a tarefa 1 voltar a ser executada.

Como as corrotinas possuem controle do seu ciclo de vida, é possível que uma corrotina deliberadamente peça a suspensão do seu direito de execução para dar espaço a outra corrotina. É o que vemos nesse exemplo:

Copiar

```php
<?php

// Tarefa 1
$firstTaskId = go(static function() {
    echo 'a';
    co::yield();
    echo 'b';
    co::yield();
    echo 'c';
});

// Tarefa 2
go(static function() use($firstTaskId) {
    $i = 0;
    while (true) {
        $i++;

        if ($i === 1000 || $i === 2000) {
            echo " {$i} ";

            co::resume($firstTaskId);
        }
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./scheduler3.php"
```

O resultado:

Copiar

```
a 1000 b 2000 c
```

Quando criamos uma corrotina imediatamente recebemos o *id* dela, por isso definimos a variável `$firstTaskId`. A primeira tarefa imprime `a` e então abre mão do seu direito de execução, o que faz com que a segunda tarefa seja executada. Quando o contador chega em 1000, a segunda tarefa abre mão do seu direito de execução para que especificamente a primeira tarefa volte a ser executada e então `b` é impresso. Mas depois de imprimir `b`, a primeira tarefa novamente abre mão do seu direito de execução e então chegamos no contador 2000 da segunda tarefa que a resume novamente imprimindo, por fim, `c`.

Ok, mas e se existisse uma forma do scheduler cuidar dessas questões e não deixar que uma tarefa “sacana” tome todo o tempo da CPU dedicado ao processo? Existe, é possível ativarmos o modo preemptivo. Quando ativamos o modo preemptivo no scheduler, ele passa a funcionar de forma parecida com o scheduler do sistema operacional, dando um tempo justo pra cada linha de execução, sem deixar que uma tarefa impeça as outras de serem executadas. Esse modo preemptivo foi adicionado recentemente e ele parece ter um impacto positivo em aplicações de alto porte que envolvem uma mistura considerável de tarefas CPU bound e I/O bound. Talvez pra sua aplicação não mude muita coisa, ou talvez mude, você teria que testar essa carga nos dois modos (cooperativo e preemptivo) e então ver qual faz mais sentido pro seu caso de uso.

De qualquer forma, voltando no nosso caso hipotético do `while(true)`, usando o modo preemptivo, temos:

Copiar

```php
<?php

ini_set('swoole.enable_preemptive_scheduler', 1);

go(static function() {
    $i = 0;
    while (true) {
        $i++;
    }
});

go(static function() {
    for ($i = 0; $i <= 10; $i++) {
        echo "N{$i}";
    }
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./scheduler4.php"
```

O resultado:

Copiar

```
N0N1N2N3N4N5N6N7N8N9N10
```

Veja que a primeira tarefa é um `while (true)` , mas como o modo preemptivo foi ativado, ela terá um tempo de CPU em milissegundos (no máximo 10ms) e então terá que abrir espaço para que outra tarefa seja executada, depois o tempo da CPU volta pra ela novamente, algo controlado automaticamente pelo scheduler.

**Aninhamento de corrotinas**

Como já vimos anteriormente, é possível aninharmos corrotinas, criando novas sub-corrotinas. Um bom exemplo para entender a ordem de execução de corrotinas aninhadas:

Copiar

```php
<?php

go(static function () { //T1
    echo "[init]\n";
    go(static function () { //T2
        go(static function () { //T3
            echo "co3\n";
        });

        echo "co2\n";
    });

    echo "co1\n";
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co6.php"
```

O resultado:

Copiar

```
[init]
co3
co2
co1
```

Agora, a história muda quando as corrotinas realizam ou emulam alguma operação de I/O:

Copiar

```php
<?php

use Swoole\Coroutine\System;

go(static function () { //T1
    echo "[init]\n";
    go(static function () { //T2
        System::sleep(3);
        go(static function () { //T3
            System::sleep(2);
            echo "co3\n";
        });

        echo "co2\n";
    });

    System::sleep(1);
    echo "co1\n";
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co7.php"
```

O resultado será:

Copiar

```
[init]
co1
co2
co3
```

**WaitGroup**

Com um “grupo de espera” podemos aguardar a finalização de algumas corrotinas antes que executemos alguma outra instrução:

Copiar

```php
<?php

use Swoole\Coroutine\System;
use Swoole\Coroutine\WaitGroup;

$wg = new WaitGroup();

go(static function () use ($wg) {
    $wg->add(3);

    go(static function () use ($wg) {
        System::sleep(3);
        echo "T1\n";
        $wg->done();
    });

    go(static function () use ($wg) {
        System::sleep(2);
        echo "T2\n";
        $wg->done();
    });

    go(static function () use ($wg) {
        System::sleep(1);
        echo "T3\n";
        $wg->done();
    });

    // Aguarda a execução das corrotinas do grupo antes de executar as instruções abaixo
    $wg->wait();

    echo "\n---- \ ----\n";
    go(static function () {
        echo "\n[FIM]\n";
    });
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co8.php"
```

O resultado será:

Copiar

```
T3
T2
T1

---- \ ----

[FIM]
```

O método `add()` é para incrementar o contador de quantas corrotinas estão no grupo de espera, ele pode ser usado quantas vezes forem necessárias.

**Devo me preocupar com race conditions?**

Em implementações em que o scheduler usa o modelo multi-thread, como em Go, o desenvolvedor precisa se preocupar com o acesso aos recursos globais compartilhados, para garantir que duas ou mais corrotinas não os acessem ao mesmo tempo, o que invariavelmente causaria *race conditions* (condições de corrida). Mas esse não é o caso quando usamos Swoole, pois o scheduler dele é single-thread, portanto, não há necessidade de *lockings*.

Esse exemplo em Go que cria 5k gorrotinas incrementando uma variável global:

Copiar

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var count int
var waitGroup sync.WaitGroup

func main() {
	for i := 0; i < 5000; i++ {
		waitGroup.Add(1)
		go increment()
	}

	waitGroup.Wait()
	fmt.Println(count)
}

func increment() {
	time.Sleep(1 * time.Second)

	count++
	waitGroup.Done()
}
```

Para executá-lo:

Copiar

```bash
$ time go run go1.go
```

Você pode testá-lo inúmeras vezes e verá que sempre terá um resultado diferente de `5.000`, exatamente por causa das *race conditions* que acontecem, uma gorrotina atropelando a outra na hora de acessar a variável global.

Go implementa uma ferramenta para identificar *race conditions*, bastando adicionar o parâmetro `-race` na execução:

Copiar

```bash
$ time go run -race go1.go
```

Ele indicará que o programa é uma “fábrica” de *race conditions*:

Copiar

```
==================
WARNING: DATA RACE
Read at 0x000001229360 by goroutine 8:
  main.increment()
...
Found 4 data race(s)
exit status 66
```

Para que as evitemos, podemos usar *mutexe*s ou *operações atômicas*. Vamos com a primeira opção que é bem simples de assimilar:

Copiar

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var count int
var mu sync.Mutex
var waitGroup sync.WaitGroup

func main() {
	for i := 0; i < 5000; i++ {
		waitGroup.Add(1)
		go increment()
	}

	waitGroup.Wait()
	fmt.Println(count)
}

func increment() {
	time.Sleep(1 * time.Second)

	mu.Lock()
	count++
	mu.Unlock()

	waitGroup.Done()
}
```

Observe que envolvemos a operação de incremento com `mu.Lock()` e `mu.Unlock()` para garantir um único acesso por vez à variável global.

Ao executar novamente o exemplo:

Copiar

```bash
$ time go run -race go1.go
```

O resultado:

Copiar

```
5000
```

E não teremos nenhum erro da ferramenta de verificação de *race conditions*.

Podemos ter o mesmo exemplo escrito no Swoole sem que precisemos fazer nada de especial (em relação a *locks* etc):

Copiar

```php
<?php

use Swoole\Coroutine\System;
use Swoole\Coroutine\WaitGroup;

$count = 0;

go(static function() use(&$count) {
    $wg = new WaitGroup();

    for ($i = 0; $i < 5000; $i++) {
        $wg->add(1);

        go(static function () use($wg, &$count) {
            System::sleep(1);
            $count++;

            $wg->done();
        });
    }

    $wg->wait();

    echo $count;
});
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co9.php"
```

Usamos *WaitGroup* para fazer paralelo com a implementação em Go, mas nesse exemplo em especial, poderíamos ter cortado essa etapa e escrito assim:

Copiar

```php
<?php

use Swoole\Coroutine\System;

$count = 0;

Co\run(static function() use(&$count) {
    for ($i = 0; $i < 5000; $i++) {
        go(static function () use(&$count) {
            System::sleep(1);
            $count++;
        });
    }
});

echo $count;
```

Para executá-lo:

Copiar

```bash
$ docker-compose exec client bash -c "time php ./co10.php"
```

Esse exemplo produz o mesmo resultado que o anterior. `Co\run` aguarda as corrotinas serem finalizadas antes de seguir o fluxo da execução.

**O que mais posso fazer com corrotinas?**

Muito mais. Os clientes de corrotinas atualmente implementados/suportados pelo Swoole:

- TCP/UDP Client：[Swoole\Coroutine\Client](https://www.swoole.co.uk/docs/modules/swoole-coroutine-client)
- HTTP/WebSocket Client：[Swoole\Coroutine\Http\Client](https://www.swoole.co.uk/docs/modules/swoole-coroutine-http-client)
- HTTP2 Client：[Swoole\Coroutine\Http2\Client](https://www.swoole.co.uk/docs/modules/swoole-coroutine-http2-client)
- Redis Client：[Swoole\Coroutine\Redis](https://www.swoole.co.uk/docs/modules/swoole-coroutine-redis)
- MySQL Client：[Swoole\Coroutine\MySQL](https://www.swoole.co.uk/docs/modules/swoole-coroutine-mysql)
- PostgreSQL Client：[Swoole\Coroutine\Postgres](https://www.swoole.co.uk/docs/modules/swoole-coroutine-postgres)
- Coroutine Socket: [Swoole\Coroutine\Socket](https://www.swoole.co.uk/docs/modules/swoole-coroutine-socket)

E, claro, lembre-se sempre de acompanhar a [documentação](https://www.swoole.co.uk/docs/modules/swoole-coroutine).

**O que mais posso fazer com o Swoole?**

Recomendo acompanhar a lista **[awesome-swoole](https://github.com/swooletw/awesome-swoole)**. Muita coisa boa, frameworks, libraries etc.

**Considerações finais**

Vimos a teoria essencial de corrotinas que são atualmente o principal mecanismo interno do Swoole e cada vez mais ganharão importância em seu core.

Nos próximos artigos exploraremos outras APIs do Swoole. Até breve!



![Desenvolvedor PHP](https://d2knvm16wkt3ia.cloudfront.net/assets/svg-icon/php.svg)

##### FormaçãoDesenvolvedor PHP

[Conhecer a formação](https://www.treinaweb.com.br/formacao/desenvolvedor-php)

