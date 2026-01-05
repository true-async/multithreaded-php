# Multithreading en PHP : regard vers l’avenir

## Pourquoi cet article ?

Dans le cadre de [RFC TrueAsync 1.7](https://wiki.php.net/rfc/true_async), une question se pose : comment la RFC proposée interagira-t-elle avec de possibles évolutions futures du cœur de PHP ? Avoir au moins une idée de la direction que pourrait prendre PHP est essentiel pour bien concevoir le langage pour les années à venir. Voilà pourquoi cet article existe.

Le projet [TrueAsync](https://github.com/true-async/) n’est pas seulement des changements du cœur PHP pour l’async, mais aussi d’autres recherches nécessaires pour répondre à des questions comme :

* Jusqu’où PHP peut-il aller vers le multithreading ?
* Y a-t-il des limites fondamentales ?
* Quels changements du cœur seraient nécessaires pour rendre le multithreading réel ?
* Quelles abstractions de langage pourraient être implémentées ?

Je n’ai pas cherché à faire une revue exhaustive de tous les aspects du multithreading en PHP ni à être techniquement exact sur chaque détail ou accessible à tout le monde. J’espère néanmoins que le texte sera utile à de nombreux développeurs PHP et donnera une direction pour poursuivre la discussion.

## Histoire

Il y a quelques années, lorsqu’il a fallu ajouter de la télémétrie volumineuse à une application PHP, j’ai dit : impossible. Mais en découvrant l’architecture de `Swoole`, j’ai voulu tester cette affirmation. Peut-on créer une API capable de générer et traiter beaucoup de données sans ralentir le client ?

Nous avons construit une version optimisée d’`OpenTelemetry` pour PHP : les données étaient écrites par morceaux, regroupées en gros blocs pour être envoyées à un serveur de télémétrie intermédiaire. Les données étaient compressées, et les structures `JSON` sérialisées avec `MessagePack`.

L’hypothèse principale : avec des coroutines monothread, on peut construire la télémétrie progressivement et l’envoyer périodiquement au serveur via un timer ou un seuil de taille. Le code devrait être rapide car il n’y a pas d’interaction entre threads. Vrai ?

L’expérience a montré que la télémétrie divisait par deux le débit de l’API. Hypothèse invalidée. Pourquoi ? Sur le plan _conceptuel_, tout semblait logique. `Swoole` rendait déjà les fonctions PHP non bloquantes, donc les coroutines auraient dû être efficaces. Une erreur s’est glissée quelque part.

Dans la deuxième version, la télémétrie était collectée uniquement durant une requête, puis envoyée immédiatement à un processus job qui agrégait, compressait et expédiait au serveur. Cela a bien mieux marché. Mais ça ne devrait pas se passer ainsi ! Ou bien ? Les données interprocessus passaient par `pipe`, sérialisées d’un côté, désérialisées de l’autre. Même si le pipe est en mémoire, les appels système coûtent cher.

Plus tard, nous avons trouvé la raison : beaucoup de télémétrie, la compression consommait beaucoup de CPU par rapport au traitement des requêtes API. Ainsi, même si les coroutines `Swoole` étaient efficaces pour l’I/O, elles n’aidaient pas pour les tâches CPU intensives.

Ce cas, parmi d’autres, montre que les coroutines monothread ne résolvent pas tout. Il montre aussi comment le multithreading peut compléter les coroutines et former un ensemble d’outils pour une large gamme de problèmes.

## Single-threaded + offload

Déléguer le travail CPU intensif à un autre processus n’est pas une « invention nouvelle ». Cela fait partie d’un modèle plus large, apparu indépendamment dans plusieurs langages et frameworks, appelé **Single-threaded + offload**.

Imaginez une personne triant rapidement les lettres (des milliers par heure), tandis que d’autres employés chargent et transportent les colis lourds. Si le trieur commence à porter des colis, la pile de lettres monte jusqu’au plafond.

Le modèle `Single-threaded + offload` sépare les tâches en deux types :

1. **Tâches I/O-bound** — lecture de fichiers, appels réseau, accès BDD. La plupart du temps, le processus attend l’extérieur. Des milliers d’opérations de ce type tiennent dans un thread via l’async concurrent (`coroutines`, `await`).

2. **Tâches CPU-bound** — compression, chiffrement, parsing, calcul. Ici la CPU tourne à fond, la simple concurrence ne suffit pas : il faut plus de cœurs.

Le modèle sépare ces tâches **physiquement** : le thread principal (`Event Loop`) ne fait que l’`I/O`, les tâches `CPU` vont dans des threads ou processus séparés (`Workers`).

**Node.js** était célèbre pour son `Event Loop` monothread, idéal pour les apps réseau. Mais lorsqu’on traitait des images ou compressait des vidéos dans le handler de requête, le serveur se transformait en citrouille. La solution est venue avec `Worker Threads` — des threads séparés pour les opérations CPU lourdes.

**Python** a suivi une voie similaire. Avec `asyncio`, le langage a un outil puissant pour le code I/O-bound, mais le GIL (Global Interpreter Lock) empêchait le vrai parallélisme CPU dans un seul processus (problème déjà adressé au moment d’écrire). Pour les opérations bloquantes, `loop.run_in_executor()` et `asyncio.to_thread()` (depuis Python 3.9) sont apparus pour déléguer le travail lourd à un pool de threads ou de processus. L’event loop reste réactif et les calculs tournent en parallèle.

**PHP/Swoole** adopte la même architecture : les `Request Workers` gèrent les requêtes HTTP avec des coroutines, les `Task Workers` font le calcul lourd. Via `UnixSocket` ou `pipe`, un processus peut traiter ~100k opérations par seconde.

### Avantages du modèle

**1. Efficacité des ressources**

Un event loop monothread peut servir des milliers d’opérations I/O concurrentes avec très peu d’overhead. Le switch entre coroutines coûte moins cher qu’un changement de contexte de thread OS. Les tâches `CPU-bound` obtiennent du vrai parallélisme sur plusieurs cœurs — chaque worker charge son cœur sans gêner les autres.

**2. Simplicité de développement**

Dans le code du loop, pas besoin de mutex, sémaphores ni autres « joies » du multithreading. En monothread, une seule tâche s’exécute à la fois, donc pas de **race conditions**. Les `workers` tournent en parallèle, mais en suivant `Shared Nothing`, les soucis de synchro ne surgissent pas.

La différence de complexité entre du code multithread et du code async monothread est énorme. Pas surprenant que les langages/frameworks modernes visent l’async monothread plutôt que le multithreading classique.

**3. Compilateur/runtime plus simples**

Les fonctions async en monothread sont bien plus simples pour le compilateur et le runtime. Un bon langage multithread a besoin d’une pipeline de codegen dédiée. PHP a une contrainte majeure : certaines parties sont en C. Cela empêche des optimisations efficaces de bytecode, la gestion mémoire et le passage de paramètres adaptés au multithreading. Que Go soit complexe — pile propriétaire, GC sophistiqué — sert aux goroutines et canaux. Nous reparlerons du GC PHP plus loin.

**4. Répartition manuelle de charge**

Le développeur peut séparer consciemment la charge entre le code de traitement de requête et le code du `worker pool`. Le contrôle manuel permet d’extraire le maximum du matériel. C’est aussi un inconvénient.

### Inconvénients du modèle

**1. Répartition manuelle de charge**

Double tranchant : un dev peut optimiser ou se tromper et mettre des tâches lourdes dans le code I/O. Le code I/O peut être surchargé, dégradant la réactivité et augmentant la latence.

Le modèle exige des développeurs PHP suffisamment expérimentés ou de s’appuyer sur des solutions bien pensées des frameworks.

**2. Pas pour tout**

`Single-threaded + offload` est idéal pour serveurs web, APIs, microservices où la charge principale est l’I/O avec BDD, FS, réseau. Si chaque étape demande du calcul intensif — calcul scientifique, rendu, ML — il peut être moins efficace ; le multithreading complet est préférable.

Vous pourriez dire : on peut vivre avec ça ! On est prêts ! Mais PHP est-il prêt à être multithreadé ?

## PHP est-il prêt pour le multithreading ?

Pendant `TrueAsync`, l’une des discussions les plus ardues était : « pourquoi PHP n’a pas d’async ». Expliquer pourquoi PHP n’est pas prêt pour le multithreading peut être tout aussi difficile. D’abord, parlons multithreading. À quoi sert-il — ou plutôt : à quoi il **NE** sert pas ?

> Le multithreading n’est pas nécessaire pour exécuter du code en parallèle.

L’idée que le multithreading soit indispensable au parallèle est ancrée chez les développeurs, comme l’idée que les trous noirs « aspirent » la matière dans l’imaginaire collectif.

Le parallélisme est très bien géré par des processus, isolés les uns des autres (depuis l’architecture 80386). Les processus communiquent via `IPC`, leur fin est suivie par des signaux (événements OS). Alors, pourquoi des threads ?

Pour répondre honnêtement, il faudrait remonter le temps et demander aux décideurs : Edsger Dijkstra, Fernando Corbató, Barbara Liskov, Richard Rashid… On pourrait faire un super talk-show. Mais même avec eux, on n’aurait peut-être pas de réponse franche.

Il serait incorrect d’affirmer :

> Les threads sont nécessaires pour que du code parallèle partage de la mémoire sans outils supplémentaires.

Les processus peuvent aussi partager de la mémoire, mais il faut mapper un segment dans l’espace d’adressage (outil supplémentaire). Les threads partagent **toute** la mémoire par défaut. Littéralement, si la variable `x` est accessible dans le thread `A`, elle l’est à la même adresse dans `B` sans astuce… Mais non : plusieurs threads **ne peuvent pas** manipuler la même variable sans outils.

Une phrase plus honnête serait :

> Les threads servent à transmettre de la mémoire entre tâches sans surcoût.

Si les threads utilisent la mémoire pour passer des messages et garantissent qu’à un instant donné un seul thread accède à une zone, c’est le maximum d’efficacité en mémoire et CPU. En même temps, ils évitent les zones partagées. Ce modèle s’appelle `Shared Nothing`.

Les threads servent à transférer efficacement des données entre tâches. Aussi vrai que « les trous noirs n’aspirent pas ».

## Modèle mémoire de PHP

Comment PHP gère-t-il la mémoire ? Modèle abstrait simplifié :
1. Code
2. Données
3. État de la VM PHP

Partager le code PHP entre threads est déjà possible (résolu avec PHP JIT). Les autres composants sont étroitement couplés et difficiles à séparer. PHP utilise un `object_store` global avec des références à tous les objets créés. Le gestionnaire mémoire PHP est conçu pour les objets d’une seule VM et pas orienté multithreading. Le **Garbage Collector** PHP ne peut pas travailler avec des données de plusieurs threads et requiert même l’arrêt de la VM car il modifie directement les `refcount`.

Ainsi, PHP est un modèle strictement monothread, avec un GC stop-the-world.

### Déplacer la VM PHP entre threads

PHP utilise le **Thread-Local Storage (TLS)** pour conserver l’état de la VM par thread. C’est essentiel pour l’isolation en mode ZTS (Zend Thread Safety).

Dans les builds modernes, on utilise une variable « statique » C11 `__thread` (ou `__declspec(thread)` en MSVC) pour obtenir le pointeur vers l’état de VM. C’est très rapide ; en `x86_64`, c’est une lecture d’adresse à un offset depuis `FS` ou `GS`.

```asm
      ; offset - décalage constant calculé à la compilation
      ; fs - base du segment
      mov rax, QWORD PTR fs:offset
```

`FS/GS` est unique par thread (OS), donc la lecture donne toujours le bon pointeur VM.

Déplacer l’état VM entre threads peut aider à implémenter des coroutines façon Go ou des acteurs. Les VM modernes transfèrent le contexte via du codegen custom en utilisant des registres CPU. En PHP, impossible : sous le capot, des fonctions C, et en C pas de paramètre de contexte implicite pour toutes les fonctions. Déplacer l’état de VM PHP entre threads coûtera donc un pourcentage de performance.

Et si l’on ne déplaçait qu’une petite partie du state nécessaire à l’exécution ? Par exemple, `PHP Fiber` copie certains pointeurs vers des structures globales (`zend_executor_globals`) lors des bascules.

Et si l’on divisait la VM PHP en deux parties larges :
1. `PHP VM` shared. Classes, fonctions, constantes, directives ini, code exécutable.
2. `PHP VM` movable. La partie de VM qui doit se déplacer.

![PHP VM shared vs private](diagrams/tls-globals-structure.svg)

Certaines structures pourraient être marquées `shared`, d’autres `movable` ; même `Executor Globals` pourrait être scindé en parts shared/movable pour un déplacement efficace de la VM. Les structures globales des extensions ne perdraient pas en performance à cause d’une indirection supplémentaire, elles l’utilisent déjà.

Le problème vient des structures liées à la compilation de code, car PHP est dynamique avec `include/require`, `eval`, autoload. Ces traits rendent difficile une séparation efficace du state shared/movable. Si l’on résout ça, PHP pourrait déplacer une partie de l’état de la VM entre threads avec peu d’overhead.

## Passage d’objets entre threads

Quelles modifications pour que PHP passe des objets entre threads sans douleur ? Comment cela fonctionnerait-il ?

Regardons au niveau du langage. Supposons un objet `SomeObject` dans `$obj` à envoyer dans un autre thread. Possible ?

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

$thread->join();
```

`SomeObject` n’appartient qu’à `$obj`, on pourrait donc déplacer son adresse dans un autre thread en sécurité. `$obj` dans le thread principal serait détruit :

```php
$obj = new SomeObject();

$thread = new Thread(function () use ($obj) {
    echo $obj->someMethod();
});

// $obj is undefined here

$thread->join();
```

Ce code est l’équivalent des moves en C++/Rust. Cette façon de passer la mémoire offre :
1. Sécurité : un seul thread possède l’objet.
2. Pas d’overhead de copie ou de sérialisation.

Pour un comportement prédictible et lisible par les analyseurs statiques, ajoutons une syntaxe dédiée, par ex. :

```php
$obj = new SomeObject();

// consume $obj indique le move
$thread = new Thread(function () use (consume $obj) {
    echo $obj->someMethod();
});

// $obj is undefined here. Error should be reported here in PHP9.
echo $obj;
```

Ça a l’air superbe, non ?

Cependant, déplacer des objets avec `refcount = 1` pose des problèmes. Prenons un arbre de catégories :

```php
$electronics = new CategoryNode('Electronics');

$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $electronics);
$categoriesTree->addToPath('/popular/electronics', $electronics);  // même catégorie !
```

`$electronics` apparaît deux fois (`refcount = 2`). Que se passe-t-il si on déplace `$categoriesTree` dans un autre thread ?

Pour déplacer en sécurité, il faut garantir que tous les objets du graphe n’ont pas de références externes :

```php
$node = new CategoryNode('Electronics');
$categoriesTree = new Tree();
$categoriesTree->addToPath('/products/electronics', $node);

$favourites = [$node];  // référence externe !

$thread = new Thread(function () use ($categoriesTree) {
    // $categoriesTree déplacé
});

// $favourites[0] pointe maintenant vers la mémoire d’un autre thread
// Dangling pointer !
```

Pour un déplacement sûr, il faudrait :

1. **Parcours complet du graphe** — vérifier tous les objets imbriqués.
2. **Vérification des refcount** — pour chaque objet.
3. **Préserver l’identité** — les doublons internes restent des doublons.

On peut concevoir plusieurs algorithmes pour cela, appelons-le `deep copy`. Une implémentation simple :

```php
// Pseudocode deep copy
// Graphe source dans le thread A
$node = new Node('A');        // addr: 0x1000
$tree->left = $node;          // addr: 0x1000
$tree->right = $node;         // addr: 0x1000 (même référence)

// Copie profonde dans le thread B (pseudo avec MM)
$copied_map = [];  // hash: addr_source -> addr_target

function deepCopyToThread(object $obj, Thread $target_thread_mm) 
{
    $source_addr = get_object_address($obj);

    if (isset($copied_map[$source_addr])) {
        return $copied_map[$source_addr];  // déjà copié
    }

    // Allouer dans le MM de l’autre thread
    $new_addr = $target_thread_mm->allocate(sizeof($obj));
    $copied_map[$source_addr] = $new_addr;

    // Copier les données de l’objet
    memcpy($new_addr, $source_addr, sizeof($obj));

    // Parcourir les propriétés
    foreach ($obj->properties as $prop) {
        if (is_object($prop)) {
            $new_prop_addr = deepCopyToThread($prop, $target_thread_mm);
            // Mettre à jour le pointeur
            update_property($new_addr, $prop, $new_prop_addr);
        }
    }

    return $new_addr;
}

// Résultat dans le thread B :
// $newTree->left (addr: 0x2500) === $newTree->right (addr: 0x2500)
// Identité préservée !
```

**Complexité temporelle** : `O(N + E)`, `N` objets, `E` références.
**Complexité spatiale** : `O(N)` — table de hachage + nouveaux objets + pile de récursion.

Par rapport à la sérialisation, cela peut être plus rapide car on évite les conversions de format, mais le gain dépend de la forme des données et de la taille du graphe. On peut aussi faire hybride : déplacer `refcount = 1`, lancer `deep copy` pour le reste.

Résultat :
1. Le dev PHP n’a pas à penser à la manière de passer les objets.
2. Meilleur cas : la mémoire est déplacée (`refcount = 1`).
3. Pire cas : la mémoire est copiée avec `deep copy` en préservant l’identité (`refcount > 1`).

Ça semble bien :
* changements minimes de syntaxe PHP
* changements progressifs possibles
* multithreading disponible

Mais côté cœur du langage, tout n’est pas rose. Pour que le move d’objets soit réel, PHP a besoin d’un mécanisme de gestion mémoire entre threads. Actuellement impossible.

## Gestionnaire mémoire PHP multithread

Le gestionnaire mémoire de PHP ressemble à des allocateurs modernes comme `jemalloc` ou `tcmalloc`. Différence : il manque un algorithme correct pour libérer depuis un autre thread.

Scénario :
* Objet créé dans le thread `A`.
* Passé au thread `B` par move (tel quel).
* Dans `B`, il n’est plus nécessaire et doit être libéré.

Chaque thread PHP a son `Memory Manager (MM)`. Quand `B` tente de libérer de la mémoire allouée par `A`, problème. Le MM de `B` ne sait rien de la mémoire de `A` et la libération échouerait. Accéder directement aux structures du MM de `A` depuis `B` est une mauvaise idée (synchro nécessaire). Les allocateurs multithread hautes performances modernes résolvent avec `deferred free`.

Idée générale du `deferred free` :
1. Le MM de `B` voit un pointeur inconnu.
2. Il trouve à quel MM il appartient et envoie un message dans sa file disant que le pointeur peut être libéré.
3. Le MM de `A` traite la file et libère ces pointeurs dans son contexte.

![Cross-thread deallocation](diagrams/cross-thread-free.svg)

Avec des structures lock-free, cet algorithme a un haut débit, permet de libérer en parallèle et demande presque pas de verrous.

Un gestionnaire mémoire multithread PHP ouvre la porte à d’autres changements auparavant impossibles.

## Objets partagés

Pouvoir passer la mémoire d’un thread à l’autre avec peu d’opérations est super, mais si l’on pouvait créer des objets pensés dès le départ pour être partagés entre threads ?

Beaucoup de services peuvent être construits comme des objets immuables et devraient être partageables entre processus, ce qui économise de la mémoire et accélère l’initialisation des workers PHP.

Malheureusement, `refcount` gêne, car il rend effectivement tous les objets PHP mutables. Peut-on le contourner ?

### Objets proxy

Une première approche : des objets proxy qui référencent de vrais objets stockés dans un pool de mémoire partagé entre tous les threads. Les proxys contiennent seulement un identifiant ou un pointeur et des méthodes d’accès. Inconvénients :
* temps d’accès aux données/props plus long
* complexité accrue pour `Reflection API` et calcul du type

En revanche, PHP a déjà un mécanisme mature pour les proxys. Dans certains cas, les objets partagés via proxy sont très bien, par exemple pour une table de compteurs ou une table de données à la Swoole/Table.

### Objets partagés avec flag GC_SHARE

PHP a le flag `GC_IMMUTABLE` pour des éléments **immuables**, utilisé pour :

- **Chaînes internées** (`IS_STR_INTERNED`) — constantes string durant tout le process.
- **Tableaux immuables** (`IS_ARRAY_IMMUTABLE`) — ex. `zend_empty_array`.
- **Constantes dans opcache** — code compilé avec des données constantes.

`GC_IMMUTABLE` permet au moteur de **sauter** les changements de refcount pour ces structures :

```c
// Zend/zend_types.h
// Fonction qui incrémente le refcount
static zend_always_inline void zend_gc_try_addref(zend_refcounted_h *p) {
    if (!(p->u.type_info & GC_IMMUTABLE)) {
        ZEND_RC_MOD_CHECK(p);
        ++p->refcount;
    }
}
```

Un mécanisme similaire pourrait prendre en charge les `SharedObjects`, par exemple un flag `GC_SHARE`.

L’analyse des perfs montre que vérifier `GC_SHARE` ajoute **+34% d’overhead** à un `refcount++` isolé (microbenchmark). Dans des apps réelles où le refcount est une petite fraction du travail total, l’impact devrait être quasi invisible :

- **Opérations réalistes** (tableaux/objets) : +3-9%
- **Applis réelles** : +0.05-0.5%

Cela résout la moitié du problème ; l’autre moitié est de concevoir le GC pour ces objets. Utiliser un refcount atomique n’est pas idéal (risque de ralentir quand plusieurs threads accèdent au même objet). Un `deferred free` semble mieux adapté.

### Mémoire par région

La mémoire par régions est populaire aujourd’hui pour gérer la mémoire dans les langages orientés Web.

L’idée : allouer la mémoire pour une tâche ou un thread dans des régions séparées, pouvant être libérées presque entièrement quand elles ne sont plus nécessaires. Cela évite la gestion objet par objet et simplifie le GC.

Par exemple, `PHP MM` pourrait garantir la création d’objets dans une région liée à un objet PHP. La durée de vie de la région équivaut à celle de l’objet.

Quand l’objet est détruit, la région peut être libérée sans parcourir les enfants. Si l’objet doit être « déplacé » dans un autre thread, on peut éviter le deep copy.

La VM PHP a des difficultés à implémenter la mémoire par région — liste globale d’objets, cache d’opcodes. Mais la probabilité d’une implémentation efficace n’est pas nulle et mérite recherche.

Un algorithme de région fonctionnel ouvre la voie à l’implémentation d’acteurs, objets spéciaux à mémoire isolée.

Les acteurs sont l’outil le plus pratique, puissant et sûr pour le multithreading.

## Coroutines et Threads

Du point de vue d’une coroutine, `Thread` est un objet `Awaitable`. Une coroutine peut attendre son résultat sans bloquer les autres coroutines. Ainsi, un thread peut héberger de nombreuses coroutines en attente de tâches lourdes. Le thread qui les sert reste réactif aux nouvelles requêtes car `await` sur `Thread` ne bloque pas l’event loop.

```php
use Async\await;
use Async\Thread;

$thread = new Thread(function() {
    // tâche liée au matériel
    return 42;
});

$result = await($thread); // La coroutine s’arrête ici jusqu’à ce que le Thread termine
```

Cette approche permet un chat avec des tâches CPU intensives et une logique métier simple.

![Thread + Coroutine architecture](diagrams/chat-sequence.svg)

Le schéma montre une architecture exemple. L’app a deux pools de threads : threads de requêtes avec multitâche concurrent, et threads worker pour tâches CPU intensives. Une coroutine traite une requête, peut se mettre en pause totale le temps que le worker finisse la tâche lourde, puis continue.

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

Le code coroutine est séquentiel et se lit comme si `ThreadPool::enqueue` appelait le callback dans le même thread. Le DTO traverse les threads et la chaîne résultante n’est pas copiée deux fois en mémoire.

## Garbage Collector et mode stateful

La modernisation du gestionnaire mémoire n’est pas la seule chose à faire. Sans GC efficace, PHP multithread souffrira de problèmes de performance et de fuites mémoire dues aux cycles.

Le GC de PHP combine deux algos : **reference counting** comme mécanisme principal et **Concurrent Cycle Collection** (Bacon-Rajan, 2001) pour les cycles. Le reference counting incrémente/décrémente à chaque affectation, ce qui n’est pas sûr en multithread sans synchro. Des atomiques partout auraient un énorme overhead ; sans synchro, on aurait des races et des fuites. Le cycle collector, bien que dit « concurrent », ne travaille que dans un seul thread et utilise un marquage couleur (**PURPLE** → **GREY** → **WHITE/BLACK**) pour trouver les cycles, non thread-safe.

Point positif : l’implémentation actuelle du GC fonctionne en environnement multithread car elle est séparée du gestionnaire mémoire et ne dépend pas d’où la mémoire a été allouée.

Mais si PHP veut une ère multithread pour des apps stateful, le GC doit être adapté pour :
1. Pouvoir travailler **en parallèle** dans un thread séparé, sans impacter le code métier.
2. Libérer les ressources le plus vite possible.
3. Fournir des outils supplémentaires pour détecter/journaliser les fuites, faire de la télémétrie (crucial pour les apps long-running!).

Le cycle collector peut être modifié pour fonctionner en multithread en traitant les références dans un thread à part, améliorant la réactivité globale. Pour commencer, cela peut suffire.

## Acteurs

`ThreadPool` et le passage d’objets entre threads sont utiles, mais demandent attention, compétences et effort. Il existe une meilleure abstraction pour le multithreading qui cache la complexité des threads/mémoire et convient parfaitement à la logique métier : **les acteurs**.

Les acteurs sont un modèle de programmation concurrente/parallèle où l’unité de base est l’**acteur**.

Chaque acteur :
- Possède son état isolé
- Traite les messages de façon séquentielle
- Interagit avec d’autres acteurs uniquement par messages
- Peut s’exécuter dans un thread séparé

On peut voir un acteur comme un objet, ce qui permet d’utiliser des paradigmes POO familiers dans le PHP multithread.

Imaginez un serveur de chat avec de nombreuses salles. Chaque salle est un objet.

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
   $room->postMessage('Alice', 'Hello!');  // S’exécute dans un autre thread, suspend la coroutine
   $messages = $room->getMessages();       // S’exécute dans un autre thread, suspend la coroutine
   echo json_encode($messages);
});
```

Les objets `ChatRoom` sont particuliers : leurs données et état VM sont localisés pour se déplacer facilement entre threads. Chaque méthode s’exécute dans son propre thread, mais à tout moment un seul thread peut exécuter les méthodes d’un acteur donné.

Sémantiquement, la classe de base `Actor` définit le fonctionnement de la VM et du gestionnaire mémoire pour que les objets `ChatRoom` s’exécutent en sécurité dans des threads séparés. Le type de classe « stocke » non seulement méthodes et propriétés, mais aussi la façon dont MM et GC doivent fonctionner pour ces objets. Approches similaires existent dans d’autres langages (Rust, C++). Avantage : pas de changement de syntaxe, s’intègre à la POO existante.

L’exemple ressemble à du code séquentiel dans une coroutine. Mais comme `postMessage` et `getMessages` tournent dans un autre thread, ils ne s’exécutent pas directement. La coroutine envoie un **message** dans la file de l’acteur, passe en attente, et ne reprend que lorsque l’acteur exécute la méthode dans l’autre thread et renvoie le résultat.

Rien de tout cela ne contredit la POO habituelle en PHP, car `Actor` redéfinit `__call` :

```php
class Actor 
{
    private $threadPool;

    public function __call(string $name, array $arguments): mixed
    {
        if(current_thread_id() === $this->threadPool->getThreadIdForActor($this)) {
            // Exécuter directement si on est dans le même thread
            return $this->$name(...$arguments);
        }
    
        // Sinon, mettre l’appel dans la file de l’acteur
        return $this->threadPool->enqueueActorMethod($this, $name, $arguments);
    }
}
```

`enqueueActorMethod` ajoute `postMessage` à la file, s’abonne à l’événement de résultat et appelle `Async\suspend()` pour suspendre la coroutine.

Le code de l’acteur s’exécute séquentiellement, élimine les races et rend le multithreading transparent pour le développeur.

Le parallélisme est obtenu car chaque acteur `ChatRoom` peut s’exécuter dans un thread distinct :

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

Les différentes instances de `ChatRoom` peuvent s’exécuter en parallèle dans des threads distincts, chacune avec son thread d’exécution, son état VM et sa mémoire.

Créez **100 salles de chat** :

```php
use Async\Actor;

$rooms = [
    'general' => new ChatRoom('general'),
    'random'  => new ChatRoom('random'),
    'tech'    => new ChatRoom('tech'),
    // ... encore 97 salles
];

// Coroutine pour traiter les requêtes
HttpServer::onRequest(function(Request $request, Response $response) use ($rooms) {
   // Traitement HTTP
   $roomName = $request->getQueryParam('room');
   $room = $rooms[$roomName] ?? null;
   
   if (!$room) {
      $response->setStatus(404);
      $response->write('Room not found');
      $response->end();
      return;
   }
   
   // Appels qui paraissent synchrones mais tournent dans un autre thread
   $room->postMessage($request->getQueryParam('user'), $request->getQueryParam('text'));
   $messages = $room->getMessages();
   
   $response->setHeader('Content-Type',  'application/json');  
   $response->write(json_encode($messages));
   $response->end();
});
```

Chaque salle traite les messages séquentiellement et en parallèle des autres salles.

Les acteurs n’ont besoin ni de mutex, ni de verrous, ni de synchro complexe, ni de gestion manuelle de pool. Ils offrent une solution haut niveau prête pour le parallélisme.

Si une salle doit envoyer un message à une autre, c’est possible, car les acteurs sont `SharedObject` et peuvent interagir entre threads :

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
                // Appel non bloquant
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
    
### Architecture interne des acteurs

La VM PHP garantit que tous les objets à l’intérieur d’un acteur :
* appartiennent uniquement à cet acteur et sont alloués dans sa région,
* ou appartiennent et ont été déplacés d’autres régions/threads,
* ou sont un autre SharedObject ou un autre acteur.

Un acteur soit possède sa région, soit ne travaille qu’avec des objets immuables explicitement partagés — sinon il reste des races.

Le gestionnaire mémoire garantit que toutes les opérations mémoire dans les méthodes d’acteur sont automatiquement liées à la région associée à l’acteur.

Les méthodes s’exécutent via une file `MPMC` servie par un `Scheduler`. Le `Scheduler` répartit le temps CPU entre acteurs, assurant exécution concurrente et parallèle.

![Actor model architecture](diagrams/actor-message-flow.svg)

## Conclusion

Tout cela semble bien, mais quand le verra-t-on en vrai ? pourriez-vous demander.

Le modèle `Single-threaded + offload` pourrait apparaître bientôt, car beaucoup de composants sont prêts. `TrueAsync` : coroutines monothread en bêta. Une version expérimentale du gestionnaire mémoire multithread et un `API` de création de threads existent.

Les acteurs demandent plus de temps, car ils touchent de nombreuses parties du cœur PHP, et restent un objectif réaliste pour PHP 9, offrant au marché un langage multithread sûr.
