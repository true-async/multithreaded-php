# Multithreading in PHP: uno sguardo al futuro

## Perché questo articolo?

All'interno di [RFC TrueAsync 1.7](https://wiki.php.net/rfc/true_async) nasce una domanda: come interagirà la RFC proposta con possibili futuri cambiamenti al core di PHP? Avere almeno un'idea di dove potrebbe andare PHP è fondamentale per progettare bene il linguaggio per molti anni. Ecco perché esiste questo articolo.

Il progetto [TrueAsync](https://github.com/true-async/) non è solo modifiche al core PHP per l'asincronia, ma anche altre ricerche necessarie per rispondere a domande come:

* Fin dove può spingersi PHP verso il multithreading?
* Ci sono limiti fondamentali?
* Quali cambiamenti al core servirebbero per rendere reale il multithreading?
* Quali astrazioni di linguaggio si potrebbero implementare?

Non ho cercato di fare una rassegna esaustiva di ogni aspetto del multithreading in PHP, né di essere tecnicamente preciso in ogni dettaglio o comprensibile a tutti. Spero comunque che l'articolo sia utile a molti sviluppatori PHP e indichi una direzione per discutere oltre.

## Storia

Qualche anno fa, quando dovevamo aggiungere telemetria ad alto volume a un'app PHP, dissi che era impossibile. Ma, vista l'architettura di `Swoole`, è venuta voglia di verificare. Si può costruire un API capace di generare e processare molti dati senza rallentare il cliente?

Abbiamo creato una versione ottimizzata di `OpenTelemetry` per PHP: scriveva i dati a pezzi, raggruppandoli in blocchi più grandi da inviare a un server intermedio di telemetria. I dati venivano compressi e le strutture `JSON` serializzate con `MessagePack`.

L'ipotesi era: se usiamo coroutine monothread, possiamo costruire la telemetria gradualmente e inviarla periodicamente al server con un timer o a soglia di volume. Il codice dovrebbe essere veloce perché non c'è interazione tra thread. È davvero così?

L'esperimento ha mostrato che la telemetria dimezzava il throughput dell'API. Ipotesi fallita. Perché? A livello _concettuale_ tutto sembrava logico. `Swoole` già rendeva non bloccanti le funzioni PHP, quindi le coroutine dovevano essere efficienti. Da qualche parte c'era un errore.

Nella seconda versione, la telemetria veniva raccolta solo durante una richiesta e subito inviata a un processo job che aggregava, comprimiva e spediva la telemetria al server. È andata molto meglio. Ma non dovrebbe essere così! O sì? I dati tra processi passavano via `pipe`; venivano serializzati da un lato e deserializzati dall'altro. Anche se il pipe vive in memoria, le chiamate di sistema costano.

Più tardi abbiamo trovato il motivo: tanta telemetria, la compressione consumava molto tempo CPU rispetto alle richieste API. Quindi, sebbene le coroutine `Swoole` fossero efficienti per I/O, non aiutavano per compiti CPU intensivi.

Questo è uno dei tanti casi che mostrano che le coroutine monothread non risolvono tutto. E mostrano come il multithreading possa completarle, formando un set di strumenti per tanti problemi.

## Single-threaded + offload

Scaricare il lavoro CPU intensivo in un altro processo non è una “nuova invenzione”. Fa parte di un modello più ampio, emerso indipendentemente in vari linguaggi e framework, noto come **Single-threaded + offload**.

Immagina una persona che smista rapidamente le lettere (migliaia l'ora), mentre altri caricano e trasportano i pacchi pesanti. Se il selezionatore inizia a trasportare pacchi, cosa succede? La coda di lettere cresce fino al soffitto.

Il modello `Single-threaded + offload` separa i compiti in due tipi:

1. **Tâche I/O-bound** — lettura file, chiamate di rete, accesso DB. Per la maggior parte del tempo si attende l'esterno. Migliaia di queste operazioni stanno in un thread grazie all'asincronia concorrente (`coroutine`, `await`).

2. **Tâche CPU-bound** — compressione, cifratura, parsing, calcolo. Qui la CPU lavora al massimo e la sola concorrenza non basta: servono più core.

Il modello separa fisicamente questi compiti: il thread principale (`Event Loop`) gestisce solo `I/O`, i compiti `CPU` vanno in thread o processi separati (`Workers`).

**Node.js** è famoso per il `Event Loop` monothread, ideale per app di rete. Ma quando si provava a elaborare immagini o comprimere video nel gestore della richiesta, il server diventava una zucca. La soluzione: `Worker Threads`, thread separati per operazioni CPU pesanti.

**Python** ha seguito una strada simile. Con `asyncio` il linguaggio ha uno strumento potente per il codice I/O-bound, ma il GIL (Global Interpreter Lock) impediva il vero parallelismo CPU in un processo (problema risolto al momento di scrivere). Per operazioni bloccanti sono arrivati `loop.run_in_executor()` e `asyncio.to_thread()` (da Python 3.9) per inviare lavoro pesante a un pool di thread o di processi. L'event loop resta reattivo e il calcolo va in parallelo.

**PHP/Swoole** ha la stessa architettura: i `Request Workers` gestiscono le richieste HTTP con coroutine, i `Task Workers` fanno il calcolo pesante. Con `UnixSocket` o `pipe`, un processo gestisce ~100k operazioni al secondo.

### Vantaggi del modello

**1. Efficienza delle risorse**

Un event loop monothread serve migliaia di operazioni I/O concorrenti con overhead minimo. Il passaggio tra task in coroutine costa meno del cambio di contesto dei thread di sistema. I compiti `CPU-bound` ottengono vero parallelismo su più core: ogni worker carica il proprio core senza disturbare gli altri.

**2. Semplicità per lo sviluppo**

Nel codice dell'event loop non servono mutex, semafori e altre “gioie” del multithreading. In un modello monothread gira una sola task alla volta, quindi niente **race condition**. I `worker` lavorano in parallelo ma, seguendo `Shared Nothing`, i problemi di sincronizzazione non emergono.

La differenza di complessità tra codice multithread e asincrono monothread è enorme. Non sorprende che linguaggi e framework moderni puntino al modello monothread asincrono e non al multithreading classico.

**3. Compilatore/runtime più semplici**

Le funzioni async in un modello monothread sono molto più semplici per compilatore e runtime. Un buon linguaggio multithread richiede una pipeline di codegen dedicata. PHP ha un limite serio: parti sono scritte in C. Questo impedisce ottimizzazioni di bytecode, gestione memoria e passaggio parametri orientati al multithreading. Il design complesso di Go — stack proprietario, GC sofisticato — serve a goroutine e channel. Parleremo del GC di PHP più avanti.

**4. Distribuzione manuale del carico**

Lo sviluppatore può dividere consapevolmente il carico tra il codice di gestione richiesta e il codice del `worker pool`. Il controllo manuale consente di spremere al massimo l'hardware. Ma è anche uno svantaggio.

### Svantaggi del modello

**1. Distribuzione manuale del carico**

È una lama a doppio taglio. Un dev può ottimizzare per casi specifici oppure sbagliare, mettendo compiti pesanti nel codice I/O. Il codice I/O può sovraccaricarsi, degradando la reattività e aumentando la latenza.

Il modello richiede sviluppatori PHP esperti o l'affidamento a soluzioni ben pensate dei framework.

**2. Non per ogni scenario**

`Single-threaded + offload` è ottimo per server web, API, microservizi dove la maggior parte del carico è I/O con DB, file system, rete. Ma se ogni passo richiede calcolo intenso — calcolo scientifico, rendering, ML — può essere meno efficace; serve multithreading pieno.

Si potrebbe dire: ci sta! Siamo pronti! Ma PHP lo è?

## PHP è pronto per il multithreading?

Durante lo sviluppo di `TrueAsync`, una discussione dura è stata “perché PHP non ha asincronia”. Spiegare perché PHP non è pronto al multithreading può essere altrettanto difficile. Prima però: a cosa serve il multithreading, o meglio: a cosa **NON** serve?

> Il multithreading non serve per eseguire codice in parallelo.

L'idea che serva al parallelismo è radicata tra i programmatori, come l'idea che i buchi neri “risucchino” materia.

Il parallelismo lo gestiscono bene i processi, isolati tra loro (dall'80386). I processi comunicano via `IPC` e il loro termine è tracciato con segnali. Perché allora i thread?

Per rispondere onestamente, bisognerebbe tornare indietro e chiedere a chi prese le decisioni: Edsger Dijkstra, Fernando Corbató, Barbara Liskov, Richard Rashid... sarebbe un bel talk show. Ma anche con loro, forse non avremmo una risposta netta.

Non sarebbe corretto dire:

> I thread servono perché il codice parallelo condivida memoria senza strumenti extra.

Anche i processi possono condividere memoria, ma occorre mappare un segmento nello spazio indirizzi (uno strumento in più). I thread condividono **tutta** la memoria di default. Letteralmente, se una variabile `x` è accessibile nel thread `A`, lo è allo stesso indirizzo nel thread `B` senza trucchi... Però no: più thread **non possono** lavorare sulla stessa variabile senza strumenti.

Una frase più onesta:

> I thread servono a passare memoria tra task senza costi aggiuntivi.

Se i thread usano la memoria per passare messaggi, garantendo che solo uno acceda a una regione in un dato momento, è il massimo in efficienza di memoria e CPU. E evitano aree condivise. Questo modello è `Shared Nothing`.

I thread servono a trasferire dati tra task in modo efficiente. Tanto vero quanto che “i buchi neri non risucchiano”.

## Modello di memoria di PHP

Come gestisce PHP la memoria? Modello astratto e semplificato:
1. Codice
2. Dati
3. Stato della VM PHP

Condividere il codice PHP tra thread già si può (risolto con PHP JIT). Il resto è molto accoppiato e non si separa facilmente. PHP usa un `object_store` globale con i riferimenti a tutti gli oggetti creati. Il memory manager di PHP è fatto per una sola VM e non è orientato al multithreading. Il **Garbage Collector** non lavora con dati di thread diversi e chiede lo stop della VM perché modifica direttamente i `refcount`.

Dunque PHP è un modello rigorosamente monothread, con GC stop-the-world.

### Spostare la VM PHP tra thread

PHP usa il **Thread-Local Storage (TLS)** per mantenere lo stato della VM per thread. È fondamentale per l'isolamento in modalità ZTS (Zend Thread Safety).

Nelle build moderne, una variabile “statica” in C11 `__thread` (o `__declspec(thread)` in MSVC) serve per ottenere il puntatore allo stato VM. È velocissimo; su `x86_64` è una lettura da un offset rispetto a `FS` o `GS`.

```asm
      ; offset - offset costante calcolato a compile time
      ; fs - base del segmento
      mov rax, QWORD PTR fs:offset
```

Poiché `FS/GS` è unico per thread (gestito dal SO), leggerlo dà sempre il puntatore corretto allo stato VM.

Poter spostare lo stato VM tra thread aiuta a implementare feature come coroutine stile Go o attori. Le VM moderne passano il contesto con codegen personalizzato usando registri CPU. In PHP è impossibile perché sotto ci sono funzioni C e in C non c'è un parametro di contesto implicito per tutte le funzioni. Spostare lo stato della VM PHP tra thread costerà una certa percentuale di performance.

E se spostassimo solo la parte minima necessaria a eseguire il codice? Ad esempio, `PHP Fiber` copia parte dei puntatori alle strutture globali (`zend_executor_globals`) al cambio.

Immagina di dividere la VM in due grandi parti:
1. `PHP VM` shared: classi, funzioni, costanti, direttive ini, codice eseguibile.
2. `PHP VM` movable: la parte da spostare.

![PHP VM shared vs private](diagrams/tls-globals-structure.svg)

Alcune strutture potrebbero essere `shared`, altre `movable`; persino gli `Executor Globals` potrebbero essere divisi per permettere uno spostamento efficiente della VM tra thread. Le strutture globali delle estensioni non perderebbero performance per l'indirezione extra, che già usano.

Il problema nasce con le strutture legate alla compilazione del codice: `include/require`, `eval`, autoload. Queste dinamiche ostacolano la divisione efficiente dello stato shared/movable. Se si risolve, PHP potrebbe spostare parte dello stato VM tra thread con poco overhead.

## Passaggio di oggetti tra thread

Cosa va cambiato in PHP per passare oggetti tra thread senza problemi? Come potrebbe funzionare?

Guardiamolo a livello di linguaggio. Abbiamo un `SomeObject` in `$obj` da mandare a un altro thread. È possibile?

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

$thread->join();
```

Poiché `SomeObject` appartiene solo a `$obj`, potremmo spostarne l'indirizzo in sicurezza verso un altro thread. `$obj` nel thread principale verrebbe distrutta:

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

// $obj is undefined here

$thread->join();
```

Questo è l'analogo delle move semantics di C++ (e in Rust). Passare memoria così offre:
1. Sicurezza: un solo thread possiede l'oggetto.
2. Niente overhead di copia o serializzazione.

Per rendere il comportamento predicibile e chiaro agli analyzer statici, aggiungiamo una sintassi dedicata, es.:

```php
$obj = new SomeObject();

// consume $obj indica lo spostamento
$thread = new Thread(function () use (consume $obj) {
    echo $obj->someMethod();
});

// $obj is undefined here. Error should be reported here in PHP9.
echo $obj;
```

Sembra ottimo, no?

Ma lo spostamento con `refcount = 1` ha problemi. Considera un albero di categorie:

```php
$electronics = new CategoryNode('Electronics');

$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $electronics);
$categoriesTree->addToPath('/popular/electronics', $electronics);  // stessa categoria!
```

`$electronics` appare due volte (`refcount = 2`). Se spostiamo `$categoriesTree` in un altro thread?

Per spostare in sicurezza, occorre garantire che tutti gli oggetti del grafo non abbiano riferimenti esterni:

```php
$node = new CategoryNode('Electronics');
$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $node);

$favourites = [$node];  // riferimento esterno

$thread = new Thread(function () use ($categoriesTree) {
    // $categoriesTree spostato
});

// $favourites[0] ora punta a memoria di un altro thread
// Dangling pointer!
```

Per garantire lo spostamento servirebbe:

1. **Visita completa del grafo** — controllare tutti gli oggetti annidati.
2. **Verifica refcount** — per ogni oggetto.
3. **Preservare l'identità** — i duplicati nel grafo restano tali.

Si possono ideare algoritmi per questo, chiamato `deep copy`. Un'implementazione semplice potrebbe essere:

```php
// Pseudocodice deep copy
// Grafo sorgente nel thread A
$node = new Node('A');        // addr: 0x1000
$tree->left = $node;          // addr: 0x1000
$tree->right = $node;         // addr: 0x1000 (stesso riferimento)

// Copia nel thread B (pseudo con MM)
$copied_map = [];  // hash: addr_source -> addr_target

function deepCopyToThread(object $obj, Thread $target_thread_mm) 
{
    $source_addr = get_object_address($obj);

    if (isset($copied_map[$source_addr])) {
        return $copied_map[$source_addr];  // già copiato
    }

    // Alloca nel MM dell'altro thread
    $new_addr = $target_thread_mm->allocate(sizeof($obj));
    $copied_map[$source_addr] = $new_addr;

    // Copia dati
    memcpy($new_addr, $source_addr, sizeof($obj));

    // Visita proprietà
    foreach ($obj->properties as $prop) {
        if (is_object($prop)) {
            $new_prop_addr = deepCopyToThread($prop, $target_thread_mm);
            // Aggiorna puntatore
            update_property($new_addr, $prop, $new_prop_addr);
        }
    }

    return $new_addr;
}

// Risultato nel thread B:
// $newTree->left (addr: 0x2500) === $newTree->right (addr: 0x2500)
// Identità preservata!
```

**Complessità temporale**: `O(N + E)`, `N` oggetti, `E` riferimenti.
**Complessità spaziale**: `O(N)` — hash + nuovi oggetti + stack di ricorsione.

Rispetto alla serializzazione può essere più veloce perché evita conversioni di formato, ma il guadagno dipende dalla forma dei dati e dalla dimensione del grafo. Si può anche fare un ibrido: spostare dati con `refcount = 1` e usare `deep copy` per il resto.

Risultato:
1. Il dev PHP non deve pensare a come passare oggetti ad altri thread.
2. Miglior caso: memoria spostata (`refcount = 1`).
3. Peggior caso: memoria copiata con `deep copy` preservando l'identità (`refcount > 1`).

Suona bene:
* minimi cambi in sintassi
* cambi graduali possibili
* multithreading accessibile

Ma a livello core non è tutto roseo. Per rendere reale lo spostamento, PHP ha bisogno di un meccanismo di gestione memoria tra thread. Al momento non c'è.

## Memory Manager PHP multithread

Il memory manager di PHP assomiglia a allocatori moderni come `jemalloc` o `tcmalloc`. Differenza: manca un algoritmo corretto per liberare memoria da un altro thread.

Scenario:
* Un oggetto nasce in thread `A`.
* Viene passato al thread `B` per movimento (inalterato).
* In `B` non serve più e va liberato.

Ogni thread PHP ha il proprio `Memory Manager (MM)`. Quando `B` tenta di liberare memoria allocata da `A`, c'è un problema. Il MM di `B` non conosce la memoria di `A`, e liberarla causerebbe errori. Accedere dal `B` alle strutture del MM di `A` è una cattiva idea perché richiede sincronizzazione. Gli allocatori multithread ad alte prestazioni risolvono con `deferred free`.

Idea di `deferred free`:
1. Il MM di `B` vede un puntatore sconosciuto.
2. Trova a quale MM appartiene e invia un messaggio alla sua coda dicendo che si può liberare.
3. Il MM di `A` processa la coda e libera quei puntatori nel proprio contesto.

![Cross-thread deallocation](diagrams/cross-thread-free.svg)

Con strutture lock-free, l'algoritmo ha alto throughput, permette di liberare in parallelo e quasi non usa lock.

Un memory manager multithread per PHP apre ad altri cambi prima impossibili.

## Oggetti condivisi

Passare memoria tra thread con poche operazioni è ottimo, ma se creassimo oggetti pensati per la condivisione tra thread?

Molti servizi possono essere oggetti immutabili e dovrebbero condividersi tra processi, risparmiando memoria e accelerando l'avvio dei worker PHP.

Purtroppo il `refcount` intralcia, rendendo tutti gli oggetti PHP mutabili. Possiamo aggirarlo?

### Oggetti proxy

Una via è usare oggetti proxy che puntano a oggetti reali in un pool di memoria condiviso da tutti i thread. I proxy contengono un identificatore o puntatore e metodi di accesso. Svantaggi:
* più tempo di accesso a dati/proprietà
* complessità per `Reflection API` e calcolo dei tipi

D'altra parte, PHP ha già un meccanismo maturo per i proxy. In alcuni casi, i proxy shared sono ottimi, ad esempio per una tabella di contatori o una tabella stile Swoole/Table.

### Shared objects con flag GC_SHARE

PHP ha il flag `GC_IMMUTABLE` per elementi **immutabili**, usato per:

- **Stringhe internate** (`IS_STR_INTERNED`) — costanti stringa per tutta la vita del processo.
- **Array immutabili** (`IS_ARRAY_IMMUTABLE`) — es. `zend_empty_array`.
- **Costanti in opcache** — codice compilato con dati costanti.

`GC_IMMUTABLE` permette di **saltare** le modifiche al refcount per tali strutture:

```c
// Zend/zend_types.h
// Incrementa il refcount per zend_refcounted_h
static zend_always_inline void zend_gc_try_addref(zend_refcounted_h *p) {
    if (!(p->u.type_info & GC_IMMUTABLE)) {
        ZEND_RC_MOD_CHECK(p);
        ++p->refcount;
    }
}
```

Un meccanismo simile può supportare i `SharedObjects`, ad esempio un flag `GC_SHARE`.

L'analisi delle performance mostra che controllare `GC_SHARE` aggiunge **+34% di overhead** a un `refcount++` isolato (microbenchmark). Nelle app reali, dove il refcount è una piccola parte, l'impatto dovrebbe essere trascurabile:

- **Operazioni realistiche** (array/oggetti): +3-9%
- **App reali**: +0.05-0.5%

Questo risolve metà del problema; l'altra è progettare il GC per tali oggetti. Usare refcount atomico non è ideale per l'eventuale rallentamento con molti accessi multi-thread allo stesso oggetto. Probabilmente conviene `deferred free`.

### Region-based memory

La memory per regioni è oggi popolare per gestire memoria in linguaggi orientati al Web.

Idea: allocare memoria per un compito o thread in regioni separate che si possano liberare quasi interamente quando non servono più, evitando la gestione per oggetto e semplificando il GC.

Ad esempio, `PHP MM` può garantire la creazione di oggetti in una regione legata a un oggetto PHP. La vita della regione coincide con quella dell'oggetto.

Quando l'oggetto viene distrutto, l'intera regione si libera senza attraversare i figli. Se l'oggetto va “spostato” a un altro thread, si evita il deep copy.

La VM PHP ha difficoltà a implementare region-based memory — lista globale di oggetti, cache di opcode. Ma le probabilità di un'implementazione efficiente non sono zero e meritano ricerca.

Un algoritmo di region funzionante apre la via all'implementazione di attori, oggetti speciali con memoria isolata.

Gli attori sono lo strumento più comodo, potente e sicuro per programmare in multithreading.

## Coroutine e Thread insieme

Dal punto di vista di una coroutine, `Thread` è un oggetto `Awaitable`. Una coroutine può attendere il risultato di un `Thread` senza bloccare le altre. Quindi, in un thread possono esistere molte coroutine in attesa di compiti pesanti. Il thread che le serve resta reattivo a nuove richieste perché `await` su un `Thread` non blocca l'event loop.

```php
use Async\await;
use Async\Thread;

$thread = new Thread(function() {
    // compito legato all'hardware
    return 42;
});

$result = await($thread); // La coroutine si ferma qui finché il Thread non termina
```

Con questo approccio si può implementare un chat con task CPU intensivi e logica semplice.

![Thread + Coroutine architecture](diagrams/chat-sequence.svg)

Il diagramma mostra un'architettura di esempio. L'app ha due pool di thread: uno per gestire le richieste con multitasking concorrente, uno di worker per compiti CPU intensivi. Una coroutine elabora una richiesta, può fermarsi finché un worker svolge il compito pesante e poi continua.

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

Il codice della coroutine è sequenziale e si legge come se `ThreadPool::enqueue` invocasse il callback nello stesso thread. Il DTO attraversa i thread e la stringa finale non viene copiata due volte.

## Garbage Collector e modalità stateful

Modernizzare il memory manager non basta per migliorare PHP nel multithreading. Senza un GC efficiente, il PHP multithread soffrirà di performance e leak per i cicli.

Il GC di PHP usa due algoritmi: **reference counting** come meccanismo principale e **Concurrent Cycle Collection** (Bacon-Rajan, 2001) per i cicli. Il reference counting incrementa/decrementa a ogni assegnazione, quindi non è sicuro nel multithreading senza sincronizzazione. Fare atomici ovunque avrebbe enorme overhead; senza sincronizzazione ci sarebbero race e leak. Il cycle collector, pur chiamandosi “concurrent”, opera in un solo thread e usa marcatura a colori (**PURPLE** → **GREY** → **WHITE/BLACK**) per trovare cicli, non è thread-safe.

L'aspetto positivo: l'attuale GC funziona in ambiente multithread perché è separato dal memory manager e non dipende dal punto di allocazione.

Ma se PHP vuole un'era multithread di app stateful, il GC va adattato per:
1. Lavorare **in parallelo** in un thread dedicato senza influire sul codice di business.
2. Liberare risorse il più rapidamente possibile.
3. Offrire strumenti per rilevare/loggare leak e telemetria (cruciale per app long-running).

Il cycle collector può essere modificato per lavorare in multithread elaborando i riferimenti in un thread separato, migliorando la reattività complessiva. Per iniziare, può bastare.

## Attori

`ThreadPool` e il passaggio di oggetti tra thread sono utili ma richiedono attenzione e competenza. C'è un'astrazione migliore per il multithreading che nasconde la complessità di thread e memoria ed è ideale per la logica di business: **gli attori**.

Gli attori sono un modello di programmazione concorrente/parallela in cui l'unità di base è l'**attore**.

Ogni attore:
- Ha uno stato isolato.
- Processa i messaggi in modo sequenziale.
- Interagisce con altri attori solo tramite messaggi.
- Può girare in un thread separato.

Si può pensare a un attore come a un oggetto, permettendo l'uso di paradigmi OOP familiari in PHP multithread.

Immagina un server chat con molte stanze. Ogni stanza è un oggetto.

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
   $room->postMessage('Alice', 'Hello!');  // Esegue in altro thread, sospende la coroutine
   $messages = $room->getMessages();       // Esegue in altro thread, sospende la coroutine
   echo json_encode($messages);
});
```

Gli oggetti `ChatRoom` sono speciali: dati e stato di VM sono localizzati per spostarsi facilmente tra thread. Ogni metodo gira nel proprio thread, ma in ogni momento solo un thread può eseguire i metodi dello stesso attore.

Dal punto di vista semantico, la classe base `Actor` definisce il comportamento della VM e del memory manager affinché i `ChatRoom` possano eseguire in thread separati. Il tipo della classe “contiene” non solo metodi e proprietà, ma anche come MM e GC devono lavorare per tali oggetti. Approcci simili si vedono in altri linguaggi (Rust, C++). Vantaggio: nessuna modifica sintattica, aderisce all'OOP esistente.

L'esempio sembra codice sequenziale in una coroutine. Ma poiché `postMessage` e `getMessages` girano in un altro thread, non vengono eseguiti direttamente. La coroutine invia un **messaggio** alla coda dell'attore, entra in attesa e si riprende solo quando l'attore esegue il metodo in un altro thread e restituisce il risultato.

Nulla di questo contraddice l'OOP di PHP, perché `Actor` ridefinisce `__call`:

```php
class Actor 
{
    private $threadPool;

    public function __call(string $name, array $arguments): mixed
    {
        if(current_thread_id() === $this->threadPool->getThreadIdForActor($this)) {
            // Esegui direttamente se siamo nello stesso thread
            return $this->$name(...$arguments);
        }
    
        // Altrimenti metti in coda la chiamata
        return $this->threadPool->enqueueActorMethod($this, $name, $arguments);
    }
}
```

`enqueueActorMethod` aggiunge `postMessage` alla coda, si sottoscrive al risultato e chiama `Async\suspend()` per sospendere la coroutine.

Il codice dell'attore è sequenziale, eliminando race condition e rendendo trasparente lo sviluppo multithread.

Il parallelismo si ottiene perché ogni attore `ChatRoom` può girare in un thread separato:

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

Oggetti `ChatRoom` diversi possono eseguire in parallelo su thread diversi, ognuno con il proprio thread di esecuzione, stato VM e memoria.

Crea **100 stanze**:

```php
use Async\Actor;

$rooms = [
    'general' => new ChatRoom('general'),
    'random'  => new ChatRoom('random'),
    'tech'    => new ChatRoom('tech'),
    // ... altre 97 stanze
];

// Coroutine per gestire le richieste
HttpServer::onRequest(function(Request $request, Response $response) use ($rooms) {
   // Gestione richiesta HTTP
   $roomName = $request->getQueryParam('room');
   $room = $rooms[$roomName] ?? null;
   
   if (!$room) {
      $response->setStatus(404);
      $response->write('Room not found');
      $response->end();
      return;
   }
   
   // Le chiamate sembrano sincrone ma girano in altro thread
   $room->postMessage($request->getQueryParam('user'), $request->getQueryParam('text'));
   $messages = $room->getMessages();
   
   $response->setHeader('Content-Type',  'application/json');  
   $response->write(json_encode($messages));
   $response->end();
});
```

Ogni stanza elabora i messaggi in sequenza e in parallelo rispetto alle altre.

Gli attori non richiedono mutex, lock, sincronizzazione complessa o gestione manuale del pool. Offrono una soluzione di alto livello pronta per il parallelismo.

Se una stanza deve inviare un messaggio a un'altra, è possibile perché gli attori sono `SharedObject` e possono interagire su thread diversi:

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
                // Chiamata non bloccante
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
    
### Architettura interna degli attori

La VM garantisce che tutti gli oggetti dentro un attore:
* appartengano solo a quell'attore e siano nella sua regione,
* oppure appartengano e siano stati spostati da altre regioni o thread,
* oppure siano un altro SharedObject o un altro attore.

Un attore o possiede la propria regione, o lavora solo con oggetti immutabili esplicitamente condivisi: altrimenti ci sono race.

Il memory manager garantisce che tutte le operazioni di memoria nei metodi dell'attore si leghino automaticamente alla regione associata all'attore.

I metodi sono eseguiti tramite una coda `MPMC` gestita da un `Scheduler`. Il `Scheduler` assegna tempo CPU tra gli attori, fornendo esecuzione concorrente e parallela.

![Actor model architecture](diagrams/actor-message-flow.svg)

## Conclusione

Tutto questo suona bene, ma quando lo vedremo davvero? ti chiederai.

Il modello `Single-threaded + offload` potrebbe arrivare presto perché molti componenti sono pronti. `TrueAsync`: le coroutine monothread sono in beta. Esiste una versione sperimentale del memory manager multithread e un `API` per creare thread.

Gli attori richiedono più tempo perché toccano molte parti del core PHP e restano un obiettivo realistico per PHP 9, offrendo al mercato un linguaggio multithread sicuro.
