# Multithreading en PHP: una mirada al futuro

## ¿Por qué este artículo?

Dentro de [RFC TrueAsync 1.7](https://wiki.php.net/rfc/true_async) surge la pregunta: ¿cómo interactuará la RFC propuesta con posibles cambios futuros en el núcleo de PHP? Tener al menos una idea de hacia dónde puede ir PHP es clave para diseñar bien el lenguaje durante muchos años. Por eso existe este artículo.

El proyecto [TrueAsync](https://github.com/true-async/) no son solo cambios en el núcleo de PHP para la asincronía, sino también otras investigaciones necesarias para responder preguntas como:

* ¿Hasta dónde puede avanzar PHP hacia el multithreading?
* ¿Existen límites fundamentales?
* ¿Qué cambios en el núcleo harían falta para que el multithreading sea real?
* ¿Qué abstracciones de lenguaje se podrían implementar?

No intenté que este texto fuera una revisión exhaustiva de cada aspecto del multithreading en PHP ni que fuera técnicamente preciso en cada detalle o fácil de leer para todos. Aun así, espero que sea útil para muchos desarrolladores PHP y que marque una dirección para discutir más.

## Historia

Hace algunos años, cuando necesitábamos añadir telemetría de gran volumen a una aplicación PHP, dije que era imposible. Pero al ver la arquitectura de `Swoole`, quise probarlo en la práctica. ¿Podíamos crear un API que generara y procesara gran cantidad de datos sin ralentizar al cliente?

Construimos una versión optimizada de `OpenTelemetry` para PHP que escribía datos por partes, juntándolos en bloques grandes para enviarlos a un servidor intermedio de telemetría. Se usaba compresión y las estructuras `JSON` se serializaban con `MessagePack`.

La hipótesis principal era: si usamos corrutinas monohilo, podemos formar la telemetría poco a poco y enviarla periódicamente al servidor con un temporizador o al llegar a un umbral de tamaño. El código debería ser rápido porque no hay interacción entre hilos. ¿Es cierto?

El experimento mostró que la telemetría redujo el rendimiento del API a la mitad. La hipótesis falló. ¿Por qué? A nivel _conceptual_ todo parecía lógico. `Swoole` ya hacía que las funciones PHP fueran no bloqueantes, así que las corrutinas debían ser eficientes. Algún fallo se coló.

En la segunda versión, la telemetría se recogía solo durante una petición y se enviaba de inmediato a un proceso de trabajo que agregaba, comprimía y mandaba la telemetría al servidor. Funcionó mucho mejor. ¡Pero no debería ser así! ¿O sí? Los datos entre procesos se enviaban por `pipe`; se serializaban en un lado y se deserializaban en el otro. Incluso si el pipe vive en memoria, las llamadas al SO cuestan.

Después vimos la causa: había mucha telemetría, así que la compresión consumía mucho tiempo de CPU comparado con atender las peticiones. Aunque las corrutinas de `Swoole` eran eficientes para I/O, no ayudaban en tareas intensivas de CPU.

Este caso es uno de muchos que muestran que las corrutinas monohilo no resuelven todo. También muestra cómo el multithreading puede complementar a las corrutinas, creando un conjunto de herramientas para una gran variedad de problemas.

## Single-threaded + offload

Descargar trabajo intensivo de CPU a otro proceso no es un “invento nuevo”. Es parte de un modelo más amplio que apareció de forma independiente en varios lenguajes y frameworks y se conoce como **Single-threaded + offload**.

Imagina a una persona clasificando cartas rapidísimo (miles por hora), mientras otros empleados cargan y llevan los paquetes pesados en camiones. Si el clasificador empieza a mover paquetes, ¿qué pasa? La cola de cartas crece hasta el techo.

El modelo `Single-threaded + offload` separa las tareas en dos tipos:

1. **Tareas I/O-bound** — leer archivos, llamadas de red, acceso a bases de datos. La mayor parte del tiempo se espera al mundo exterior. Miles de estas operaciones caben en un solo hilo con asincronía concurrente (`corrutinas`, `await`).

2. **Tareas CPU-bound** — compresión, cifrado, parseo, cálculo. Aquí la CPU trabaja al máximo y la simple concurrencia no basta: se necesitan más núcleos.

El modelo separa estas tareas **físicamente**: el hilo principal (`Event Loop`) solo maneja `I/O`, y las tareas `CPU` se envían a hilos o procesos aparte (`Workers`).

**Node.js** se hizo famoso por su `Event Loop` monohilo, ideal para apps de red. Pero cuando los desarrolladores procesaban imágenes o comprimían video dentro del manejador de peticiones, el servidor se volvía una calabaza. La solución llegó con `Worker Threads`: hilos separados para operaciones pesadas de CPU.

**Python** tomó un camino similar. Con `asyncio` el lenguaje obtuvo una gran herramienta para código I/O-bound, pero el GIL (Global Interpreter Lock) impedía paralelizar de verdad la CPU en un proceso (al escribir esto, el problema ya está resuelto). Para operaciones bloqueantes aparecieron `loop.run_in_executor()` y `asyncio.to_thread()` (desde Python 3.9), que envían la carga a un `thread pool` o `process pool`. El event loop sigue respondiendo y el cálculo corre en paralelo.

**PHP/Swoole** usa la misma arquitectura: los `Request Workers` manejan peticiones HTTP con corrutinas, y los `Task Workers` realizan cómputo pesado. Comunicándose por `UnixSocket` o `pipe`, un proceso puede manejar unas 100k operaciones por segundo.

### Ventajas del modelo

**1. Eficiencia de recursos**

Un event loop monohilo puede servir miles de operaciones I/O concurrentes con muy poco overhead. Cambiar entre corrutinas cuesta menos que los cambios de contexto de hilos a nivel SO. Las tareas `CPU-bound` logran paralelismo real en varios núcleos: cada worker exprime su núcleo sin molestar a los demás.

**2. Simplicidad de desarrollo**

El código en el event loop no necesita mutexes, semáforos ni demás “alegrías” del multithreading. En un modelo monohilo solo se ejecuta una tarea a la vez, así que no hay **race conditions**. Los `workers` corren en paralelo, pero si sigues `Shared Nothing`, no surgen problemas de sincronización.

La diferencia de complejidad entre código multihilo y código asíncrono monohilo es enorme. No extraña que lenguajes y frameworks modernos apunten al modelo monohilo con asincronía en lugar del multithreading clásico.

**3. Compilador/runtime más simple**

Las funciones async en un modelo monohilo son mucho más simples para el compilador y el runtime. Un buen lenguaje multihilo necesita su propia canalización de generación de código. PHP tiene una limitación seria: parte está escrita en C. Eso impide optimizaciones eficientes de bytecode, gestión de memoria y paso de parámetros pensadas para multithreading. El diseño de Go es complejo por una razón: pila propietaria, GC sofisticado, todo para goroutines y canales. Más adelante hablaremos del GC de PHP, así que no es hora de relajarse.

**4. Distribución manual de carga**

El programador puede separar conscientemente la carga entre el código que atiende la petición y el código del `worker pool`. El control manual permite exprimir al máximo el hardware. Pero al mismo tiempo es una desventaja.

### Desventajas del modelo

**1. Distribución manual de carga**

La distribución manual es un arma de doble filo. Un desarrollador puede optimizar para tareas específicas o equivocarse y decidir mal qué va en el código I/O y qué en los `workers`. El código I/O puede saturarse con tareas pesadas, degradando la respuesta y aumentando la latencia.

El modelo requiere que los programadores PHP sean lo bastante experimentados, o que dependan de autores de frameworks y sus soluciones.

**2. No sirve para todo**

`Single-threaded + offload` es excelente para servidores web, APIs y microservicios donde la carga principal es I/O con bases de datos, sistemas de archivos o red. Pero para cargas de cómputo intenso en cada paso —cálculo científico, renderizado, ML— puede ser menos eficaz; ahí conviene multithreading pleno.

Podrías decir: ¡con eso podemos vivir! ¡Estamos listos! ¿Pero PHP mismo está listo para ser multihilo?

## ¿Está PHP listo para el multithreading?

Al trabajar en `TrueAsync`, una de las discusiones más difíciles fue “por qué PHP no tiene asincronía”. Explicar por qué PHP no está listo para multithreading puede ser igual de difícil. Primero hablemos de multithreading. ¿Para qué sirve, o mejor: para qué **NO** sirve?

> El multithreading no es necesario para ejecutar código en paralelo.

La idea de que multithreading es imprescindible para el paralelismo se instaló hace tiempo en la mente de los programadores, igual que la idea de que los agujeros negros “absorben” materia en la mente popular.

La ejecución en paralelo la resuelven bien los procesos, que están aislados entre sí (desde la arquitectura 80386). Los procesos pueden comunicarse vía `IPC`, y su final se sigue con señales (eventos del SO). Entonces, ¿para qué hilos?

Para responder honestamente habría que volver al pasado e invitar a quienes tomaron decisiones para que las explicaran: Edsger Dijkstra, Fernando Corbató, Barbara Liskov, Richard Rashid... sería un gran programa. Pero aunque aceptaran, puede que no obtuviéramos una respuesta clara.

No sería correcto afirmar:

> Los hilos existen para que el código paralelo comparta memoria sin herramientas extra.

Los procesos también pueden compartir memoria, pero hay que mapear un segmento en el espacio de direcciones (herramienta extra). Los hilos comparten **toda** la memoria por defecto. Literalmente, si hay una variable `x` accesible en el hilo `A`, en el hilo `B` está en la misma dirección sin trucos... Pero no: varios hilos **no pueden** trabajar en la misma variable sin herramientas extra.

Una afirmación más honesta sería:

> Los hilos existen para pasar memoria entre tareas sin coste adicional.

Si los hilos usan memoria para pasar mensajes, garantizando que en cada momento solo un hilo accede a cierta región, es lo más eficiente en memoria y CPU. Al mismo tiempo, evitan zonas de memoria compartida. Ese modelo se llama `Shared Nothing`.

Los hilos sirven para transferir datos entre tareas de forma eficiente. Tan cierto como que “los agujeros negros no absorben”.

## Modelo de memoria de PHP

¿Cómo maneja PHP la memoria? Un modelo abstracto y simplificado:
1. Código
2. Datos
3. Estado de la VM de PHP

Compartir código PHP entre hilos ya es posible (se resolvió con PHP JIT). Los demás componentes están muy acoplados y no se separan fácil. Por ejemplo, PHP usa un `object_store` global con referencias a todos los objetos creados. El gestor de memoria de PHP está hecho para objetos de una sola VM y no orientado al multithreading. El **Garbage Collector** de PHP no puede trabajar con datos de varios hilos y hasta requiere pausar la VM porque modifica directamente el `refcount` de los objetos.

Así que PHP es un modelo estrictamente monohilo con GC que detiene el mundo.

### Mover la VM de PHP entre hilos

PHP usa **Thread-Local Storage (TLS)** para mantener el estado de la VM por hilo. Es crítico para la aislación en modo ZTS (Zend Thread Safety).

En compilaciones modernas, se usa una variable “estática” según C11 `__thread` (o `__declspec(thread)` en MSVC) para obtener el puntero al estado de la VM. Es rapidísimo; en `x86_64` se reduce a leer una dirección a un desplazamiento desde el registro `FS` o `GS`.

```asm
      ; offset - desplazamiento constante calculado en compilación
      ; fs - dirección base del segmento
      mov rax, QWORD PTR fs:offset
```

Como `FS/GS` es único por hilo (lo asegura el SO), leerlo devuelve siempre el puntero correcto al estado de la VM.

Poder mover el estado de la VM entre hilos ayuda a implementar cosas como corrutinas tipo Go o actores. Las VM modernas pasan contexto con codegen a medida, enviando el estado de VM en registros de CPU. En PHP eso no es posible porque debajo hay funciones C y en C no se puede pasar un contexto implícito a todas las funciones. Mover el estado de la VM de PHP entre hilos costará cierto porcentaje de rendimiento.

¿Y si solo movemos la parte de estado mínima necesaria para ejecutar código? Por ejemplo, `PHP Fiber` copia parte de los punteros a estructuras globales (`zend_executor_globals`) al cambiar.

Imagina dividir la VM de PHP en dos partes grandes:
1. `PHP VM` shared: clases, funciones, constantes, directivas ini, código ejecutable.
2. `PHP VM` movable: la porción de la VM que hay que mover.

![PHP VM shared vs private](diagrams/tls-globals-structure.svg)

Algunas estructuras podrían marcarse `shared`, otras `movable`; incluso `Executor Globals` podría dividirse en partes shared/movable para mover el estado de la VM entre hilos con eficiencia. Las estructuras globales de las extensiones no perderían rendimiento por más indirectas, ya las usan.

El problema aparece con estructuras ligadas a la compilación de código, porque PHP es dinámico con `include/require`, `eval`, autoloading. Estas características impiden separar de forma eficiente el estado shared/movable. Si se resuelve, PHP podría mover parte del estado de la VM entre hilos con mínimo overhead.

## Pasar objetos entre hilos

¿Qué habría que cambiar en PHP para pasar objetos entre hilos sin dolor? ¿Cómo funcionaría?

Veámoslo a nivel de lenguaje. Supón que tenemos una instancia de `SomeObject` en `$obj` y queremos enviarla a otro hilo. ¿Es posible?

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

$thread->join();
```

Como `SomeObject` solo pertenece a `$obj`, podríamos mover su dirección de un hilo a otro con seguridad. `$obj` en el hilo principal se destruiría:

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

// $obj is undefined here

$thread->join();
```

El código es un análogo 100% de la operación de movimiento que llegó a C++ y existe en Rust y otros lenguajes. Mover memoria entre hilos así tiene:
1. Seguridad: solo un hilo posee el objeto.
2. Sin overhead de copia o serialización.

Para que el comportamiento sea predecible y fácil para analizadores estáticos, conviene un sintaxis especial, por ejemplo:

```php
$obj = new SomeObject();

// consume $obj indica movimiento del objeto
$thread = new Thread(function () use (consume $obj) {
    echo $obj->someMethod();
});

// $obj is undefined here. Error should be reported here in PHP9.
echo $obj;
```

Suena genial, ¿no?

Pero mover objetos con `refcount = 1` tiene problemas. Considera este árbol de categorías:

```php
$electronics = new CategoryNode('Electronics');

$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $electronics);
$categoriesTree->addToPath('/popular/electronics', $electronics);  // ¡la misma categoría!
```

`$electronics` está dos veces en el árbol (`refcount = 2`). ¿Qué pasa al mover `$categoriesTree` a otro hilo?

Para mover de forma segura hay que garantizar que todos los objetos del grafo no tengan referencias externas:

```php
$node = new CategoryNode('Electronics');
$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $node);

$favourites = [$node];  // referencia externa

$thread = new Thread(function () use ($categoriesTree) {
    // $categoriesTree movido
});

// $favourites[0] ahora apunta a memoria en otro hilo
// Dangling pointer!
```

Para asegurar el movimiento se necesita:

1. **Recorrido completo del grafo** — comprobar todos los objetos anidados.
2. **Comprobar refcount** — para cada objeto del grafo.
3. **Preservar identidad** — duplicados dentro del grafo deben seguir siéndolo.

Se pueden crear algoritmos para esto, llamémoslo `deep copy`. Una implementación sencilla sería:

```php
// Pseudocódigo de deep copy
// Grafo origen en el hilo A
$node = new Node('A');        // addr: 0x1000
$tree->left = $node;          // addr: 0x1000
$tree->right = $node;         // addr: 0x1000 (misma referencia)

// Copia profunda al hilo B (pseudocódigo con MM)
$copied_map = [];  // hash: addr_source -> addr_target

function deepCopyToThread(object $obj, Thread $target_thread_mm) 
{
    $source_addr = get_object_address($obj);

    if (isset($copied_map[$source_addr])) {
        return $copied_map[$source_addr];  // ya copiado
    }

    // Alocar en el MM del otro hilo
    $new_addr = $target_thread_mm->allocate(sizeof($obj));
    $copied_map[$source_addr] = $new_addr;

    // Copiar datos del objeto
    memcpy($new_addr, $source_addr, sizeof($obj));

    // Recorrer propiedades
    foreach ($obj->properties as $prop) {
        if (is_object($prop)) {
            $new_prop_addr = deepCopyToThread($prop, $target_thread_mm);
            // Actualizar puntero en el nuevo objeto
            update_property($new_addr, $prop, $new_prop_addr);
        }
    }

    return $new_addr;
}

// Resultado en el hilo B:
// $newTree->left (addr: 0x2500) === $newTree->right (addr: 0x2500)
// ¡Identidad preservada!
```

**Complejidad temporal del deep copy**: `O(N + E)`, donde `N` es número de objetos y `E` número de referencias.
**Complejidad espacial**: `O(N)` — hash + nuevos objetos + pila de recursión.

Comparado con serialización, puede ser más rápido porque evita convertir a/desde un formato de transporte, pero la ganancia depende de la forma de los datos y el tamaño del grafo. También puedes crear un híbrido: mover datos con `refcount = 1` y ejecutar `deep copy` para el resto.

Resultado:
1. El programador PHP no tiene que pensar cómo se pasan objetos a otro hilo.
2. Mejor caso: la memoria se mueve (`refcount = 1`).
3. Peor caso: la memoria se copia con `deep copy` preservando la identidad (`refcount > 1`).

Suena bien:
* cambios mínimos en la sintaxis PHP
* cambios graduales posibles
* el multithreading se vuelve accesible

Pero en el núcleo no todo es bonito. Para que mover objetos sea real, PHP necesita algún mecanismo de gestión de memoria entre hilos. Hoy no existe.

## Gestor de memoria multihilo de PHP

El gestor de memoria de PHP se parece a asignadores modernos como `jemalloc` o `tcmalloc`. La diferencia: no tiene un algoritmo correcto para liberar memoria desde otro hilo.

Escenario:
* Objeto creado en el hilo `A`.
* Se pasa al hilo `B` por movimiento (tal cual).
* En `B` ya no se necesita y debe liberarse.

Cada hilo PHP tiene su propio `Memory Manager (MM)`. Cuando `B` intenta liberar memoria que alocó `A`, hay problema. El MM de `B` no conoce la memoria de `A` y liberarla fallaría. Acceder directamente a las estructuras del MM de `A` desde `B` es mala idea porque exige sincronización. Los asignadores multihilo de alto rendimiento modernos resuelven esto con `deferred free`.

La idea general de `deferred free`:
1. El MM de `B` ve un puntero desconocido.
2. Encuentra a qué MM pertenece y envía un mensaje a su cola diciendo que se puede liberar.
3. El MM de `A` procesa la cola y libera esos punteros en su propio contexto.

![Cross-thread deallocation](diagrams/cross-thread-free.svg)

Con estructuras lock-free, este algoritmo tiene alto throughput, permite liberar en paralelo y casi no usa bloqueos.

Un gestor de memoria multihilo para PHP abre puertas a otros cambios antes imposibles.

## Objetos compartidos

Poder pasar memoria de un hilo a otro con pocas operaciones es genial, pero ¿y si pudiéramos crear objetos pensados desde el inicio para compartirse entre hilos?

Muchos servicios pueden construirse como objetos inmutables y deberían compartirse entre procesos, ahorrando memoria y acelerando la inicialización de workers PHP.

Lamentablemente el `refcount` molesta, porque hace que todos los objetos PHP sean mutables. ¿Se puede evitar?

### Objetos proxy

Una vía es usar objetos proxy que referencien objetos reales en un pool de memoria compartido accesible a todos los hilos. Los proxys guardan solo un identificador o puntero y métodos de acceso. Desventajas:
* más tiempo de acceso a datos/propiedades
* más complejidad para `Reflection API` y cálculo de tipos

Por otro lado, PHP ya tiene un mecanismo maduro para proxys. En algunos casos, los objetos compartidos vía proxy son una gran opción, p. ej. para una tabla de contadores o una tabla de datos tipo Swoole/Table.

### Objetos compartidos con flag GC_SHARE

PHP tiene el flag `GC_IMMUTABLE` para elementos **inmutables**. Se usa para:

- **Strings internadas** (`IS_STR_INTERNED`) — constantes de string que viven todo el proceso.
- **Arrays inmutables** (`IS_ARRAY_IMMUTABLE`) — ej. `zend_empty_array`.
- **Constantes en opcache** — código compilado con datos constantes.

`GC_IMMUTABLE` permite al motor **saltar** cambios de refcount en esas estructuras:

```c
// Zend/zend_types.h
// Función que incrementa el refcount para zend_refcounted_h
static zend_always_inline void zend_gc_try_addref(zend_refcounted_h *p) {
    if (!(p->u.type_info & GC_IMMUTABLE)) {
        ZEND_RC_MOD_CHECK(p);
        ++p->refcount;
    }
}
```

Un mecanismo similar podría usarse para `SharedObjects`, por ejemplo con un flag `GC_SHARE`.

El análisis de rendimiento muestra que comprobar `GC_SHARE` añade **+34% de overhead** a un `refcount++` aislado (microbenchmark). En apps reales, donde el refcount es una fracción pequeña, el impacto debería ser casi invisible:

- **Operaciones realistas** (arrays/objetos): +3-9%
- **Apps reales**: +0.05-0.5%

Esto resuelve la mitad del problema; la otra es diseñar GC para estos objetos. Usar refcount atómico no es ideal por la posible ralentización cuando muchos hilos acceden al mismo objeto. Probablemente convenga un algoritmo de `deferred free`.

### Region-based memory

La memoria por regiones es popular hoy para gestionar memoria en lenguajes orientados a la Web.

La idea: asignar memoria para una tarea o hilo en regiones separadas que pueden liberarse casi por completo cuando ya no se necesitan, evitando la gestión por objeto y simplificando el GC.

Por ejemplo, `PHP MM` podría garantizar que los objetos se creen en un región asociada a un objeto PHP. La vida de la región iguala la del objeto.

Al destruirse el objeto, toda la región se libera sin recorrer hijos. Si hay que “mover” el objeto a otro hilo, se puede evitar el deep copy.

La VM de PHP tiene problemas para implementar region-based memory — lista global de objetos, caché de opcodes, etc. Pero la posibilidad de hacerlo eficientemente no es nula y merece investigación.

Un algoritmo de regiones operativo abre la puerta a implementar actores, objetos especiales con memoria aislada.

Los actores son la herramienta más conveniente, potente y segura para programar con múltiples hilos.

## Corrutinas y Threads

Para una corrutina, `Thread` es un objeto `Awaitable`. Puede esperar su resultado sin bloquear otras corrutinas. Así, un hilo puede alojar muchas corrutinas esperando tareas pesadas. El hilo que las sirve sigue respondiendo rápido a nuevas peticiones porque `await` a un `Thread` no bloquea el event loop.

```php
use Async\await;
use Async\Thread;

$thread = new Thread(function() {
    // tarea ligada a hardware
    return 42;
});

$result = await($thread); // La corrutina se pausa aquí hasta que el Thread termine
```

Este enfoque permite un chat con tareas CPU intensivas y lógica sencilla.

![Thread + Coroutine architecture](diagrams/chat-sequence.svg)

La figura muestra una arquitectura de ejemplo. La app tiene dos pools de hilos: hilos de petición con multitarea concurrente y hilos worker para tareas CPU intensivas. Una corrutina atiende una petición, puede pausarse mientras un worker hace la tarea pesada y luego sigue.

```php
use Async\await;
use Async\ThreadPool;

final readonly class ImageDto
{
    public function __construct(
    public int $width,
    public int $height,
    public string $text,
) {}
}

$pool = new ThreadPool(2);
$dto = new ImageDto(
    width: 200,
    height: 200,
    text: 'Hello TrueAsync!'
);

$image = $pool->enqueue(function (ImageDto $dto) {
    $img = imagecreatetruecolor($dto->width, $dto->height);

    $white = imagecolorallocate($img, 255, 255, 255);
    $black = imagecolorallocate($img, 0, 0, 0);

    imagefill($img, 0, 0, $white);
    imagestring($img, 5, 20, 90, $dto->text, $black);

    ob_start();
    imagepng($img);
    imagedestroy($img);
    return ob_get_clean();
}, $dto);

$response->setHeader('Content-Type', 'image/png');
$response->write($image);
$response->end();
```

El código de la corrutina es secuencial y se lee como si `ThreadPool::enqueue` llamara al callback en el mismo hilo. El DTO cruza hilos y la cadena resultante no se copia dos veces en memoria.

## Garbage Collector y modo stateful

Modernizar el gestor de memoria de PHP no es lo único necesario para mejorar el lenguaje en multithreading. Sin un GC eficiente, PHP multihilo sufrirá problemas de rendimiento y fugas de memoria por ciclos.

El GC de PHP usa dos algoritmos: **reference counting** como mecanismo principal y **Concurrent Cycle Collection** (Bacon-Rajan, 2001) para ciclos. Reference counting incrementa/decrementa en cada asignación, lo cual no es seguro en multithreading sin sincronización. Usar atómicos en cada asignación tendría un gran overhead; sin sincronización habría carreras y fugas. El cycle collector, aunque se llame “concurrente”, solo trabaja en un hilo y usa marcado por colores (**PURPLE** → **GREY** → **WHITE/BLACK**) para detectar ciclos; tampoco es thread-safe.

La parte positiva: la implementación actual del GC funciona en entornos multihilo porque está separada del gestor de memoria y no depende de dónde se haya alocado.

Pero si PHP quiere entrar en la era multihilo de apps stateful, el GC debe adaptarse para:
1. Trabajar **en paralelo** en un hilo separado sin afectar al código de negocio.
2. Liberar recursos lo antes posible.
3. Ofrecer herramientas extra para detectar/registrar fugas y telemetría (sobre todo en apps de larga ejecución).

El cycle collector puede modificarse para funcionar en multihilo procesando referencias en un hilo aparte, mejorando la respuesta general. Para empezar, puede bastar.

## Actores

`ThreadPool` y pasar objetos entre hilos son útiles, pero exigen atención, habilidades y esfuerzo. Hay una abstracción mejor para programar en multithreading que oculta la complejidad de hilos y memoria y encaja perfecto con la lógica de negocio: **actores**.

Los actores son un modelo de programación concurrente/paralela donde la unidad básica de cómputo es un **actor**.

Cada actor:
- Tiene estado aislado propio.
- Procesa mensajes de forma secuencial.
- Interactúa con otros actores solo mediante mensajes.
- Puede ejecutarse en un hilo separado.

Puedes pensar en un actor como en un objeto, lo que permite usar las mismas ideas OOP en PHP multihilo.

Imagina un servidor de chat con muchas salas. Cada sala es un objeto.

```php
use Async\Actor;

class ChatRoom extends Actor
{
    private array $messages = [];
    private string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function postMessage(string $user, string $text): void
    {
        $this->messages[] = [
            'user' => $user,
            'text' => $text,
            'time' => time()
        ];
    }

    public function getMessages(): array
    {
        return $this->messages;
    }
}

spawn(function() {
   $room = new ChatRoom('general');
   $room->postMessage('Alice', 'Hello!');  // Corre en otro hilo, suspende la corrutina
   $messages = $room->getMessages();       // Corre en otro hilo, suspende la corrutina
   echo json_encode($messages);
});
```

Los objetos `ChatRoom` son especiales. Sus datos y estado de VM están localizados para moverse fácil entre hilos. Cada método se ejecuta en su propio hilo, pero en cada momento solo un hilo puede ejecutar métodos de un actor.

Semánticamente, la clase base `Actor` define cómo trabajan la VM y el gestor de memoria para que los `ChatRoom` se ejecuten de forma segura en hilos separados. El tipo de la clase “guarda” no solo métodos y propiedades, sino cómo deben operar el MM y el GC para estos objetos. En otros lenguajes (Rust, C++) hay enfoques similares. Ventaja: no cambia la sintaxis y encaja con la filosofía OOP existente.

El ejemplo parece código secuencial en una corrutina. Pero como `postMessage` y `getMessages` se ejecutan en otro hilo, no se llaman directamente. La corrutina envía un **mensaje** a la cola del actor, pasa a espera y solo se reanuda cuando el actor ejecuta el método en otro hilo y devuelve el resultado.

Nada de esto contradice el OOP habitual en PHP, ya que `Actor` redefine `__call`:

```php
class Actor 
{
    private $threadPool;

    public function __call(string $name, array $arguments): mixed
    {
        if(current_thread_id() === $this->threadPool->getThreadIdForActor($this)) {
            // Ejecutar directamente si estamos en el mismo hilo
            return $this->$name(...$arguments);
        }
    
        // Si no, encolar la llamada del actor
        return $this->threadPool->enqueueActorMethod($this, $name, $arguments);
    }
}
```

`enqueueActorMethod` añade `postMessage` a la cola del actor, se suscribe al evento de resultado y llama a `Async\suspend()` para pausar la corrutina.

El código del actor corre secuencialmente, elimina condiciones de carrera y hace que el desarrollo multihilo sea transparente.

La paralelización se logra porque cada actor `ChatRoom` puede ejecutarse en un hilo propio:

```php
spawn(function() {
   $room = new ChatRoom('room1');
   $room->postMessage('Alice', 'Hello!');
   $messages = $room->getMessages();
   echo json_encode($messages);
});

spawn(function() {
   $room = new ChatRoom('room2');
   $room->postMessage('Bob', 'Hi there!');
   $messages = $room->getMessages();
   echo json_encode($messages);
});
```

Instancias distintas de `ChatRoom` pueden correr en paralelo en distintos hilos, porque cada actor tiene su propio hilo de ejecución, estado de VM y memoria.

Crea **100 salas de chat**:

```php
use Async\Actor;

$rooms = [
    'general' => new ChatRoom('general'),
    'random'  => new ChatRoom('random'),
    'tech'    => new ChatRoom('tech'),
    // ... 97 salas más
];

// Corrutina para manejar peticiones
HttpServer::onRequest(function(Request $request, Response $response) use ($rooms) {
   // Procesar petición HTTP
   $roomName = $request->getQueryParam('room');
   $room = $rooms[$roomName] ?? null;
   
   if (!$room) {
      $response->setStatus(404);
      $response->write('Room not found');
      $response->end();
      return;
   }
   
   // Las llamadas parecen síncronas pero corren en otro hilo
   $room->postMessage($request->getQueryParam('user'), $request->getQueryParam('text'));
   $messages = $room->getMessages();
   
   $response->setHeader('Content-Type',  'application/json');  
   $response->write(json_encode($messages));
   $response->end();
});
```

Cada sala procesa mensajes de forma secuencial y en paralelo respecto a otras.

Los actores no necesitan mutexes, bloqueos, sincronización compleja ni gestión manual del pool de hilos. Ofrecen una solución de alto nivel lista para paralelizar.

Si una sala necesita enviar un mensaje a otra, se puede, porque los actores son `SharedObject` y pueden interactuar en hilos distintos:

```php
class Rooms extends Actor
{
    private array $rooms = [];
    
    public function __construct(string ...$roomNames)
    {
       foreach ($roomNames as $name) {
           $this->rooms[$name] = new ChatRoom($name);
       }
    }
    
    public function broadcastMessage(string $fromRoom, string $user, string $text): void
    {
        foreach ($this->rooms as $name => $room) {
            if ($name !== $fromRoom) {
                // Llamada no bloqueante
                $room->postMessageAsync($user, $text);
            }
        }
    }
}

spawn(function() {
   $rooms = new Rooms('general', 'room1', 'room2', 'room3');
   $rooms->broadcastMessage('general', 'Alice', 'Hello!');
});

```        
    
### Arquitectura interna de los actores

La VM garantiza que todos los objetos dentro de un actor:
* o bien pertenecen solo a ese actor y están en su región exclusiva,
* o bien pertenecen y fueron movidos desde otras regiones o hilos,
* o bien son otro SharedObject u otro actor.

Un actor o bien posee su región o solo trabaja con objetos inmutables explícitamente compartidos; de lo contrario, hay carreras.

El gestor de memoria asegura que todas las operaciones de memoria dentro de los métodos del actor se asocien automáticamente a la región ligada a ese actor.

Los métodos se ejecutan mediante una cola `MPMC` servida por un `Scheduler`. El `Scheduler` reparte CPU entre actores para brindar ejecución concurrente y paralela.

![Actor model architecture](diagrams/actor-message-flow.svg)

## Conclusión

Todo esto suena bien, pero ¿cuándo lo veremos en realidad? Te preguntarás.

El modelo `Single-threaded + offload` puede aparecer en un futuro próximo porque muchos componentes ya están listos. `TrueAsync`: las corrutinas monohilo llegaron a beta. Hay una versión experimental del gestor de memoria multihilo y un `API` para crear hilos.

Los actores requieren más tiempo de desarrollo porque tocan muchas partes del núcleo de PHP y siguen siendo una meta realista para PHP 9, ofreciendo al mercado un lenguaje seguro para programar en multithreading.
