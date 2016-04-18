# Programmation parallèle et asynchrone en C++
## Plan

- Threads
- Pool de threads
- Parallélisme vs Concurrence
- Map/Reduce
- OpenMP
- Programmation LockFree (Non blocking algorithms)
- Promesses et objets futurs
- Async (non blocking IO)
- Coroutines

---
## Threads

> Quand un programmeur a un problème de performance, il pense aux threads. Maintenant, il proba deux lèmes.

Qu'est-ce qu'un thread?

D'un point de vue technique, un *processus léger* est un ensemble de ressources système connues et gérées par l'OS:
- des zones mémoires réservées (pile, TLS, ...)
- un fil d'exécution supplémentaire dans l'ordonnanceur

Mais d'un point de vue plus abstrait, quel est le concept implémenté?

---

Un thread est une machine à états généraliste dont la particularité est d'être gérée par le système d'exploitation. Quand l'exécution d'un thread est bloquée (E/S, page fault...) ou interrompue par l'ordonnanceur, son état est sauvegardé pour être remplacé par le contexte d'exécution d'un autre thread en attente d'exécution (*context switch*). C'est donc l'OS qui gère de façon transparente les diverses transitions d'états (en cours d'exécution, en attente, bloqué...).


(2:30 https://www.youtube.com/watch?v=wxXIbaJBZlE)


Un thread c'est aussi une pile d'exécution qui occupe un espace mémoire non négligeable. Combinée au coût des changements de contexte, cette abstraction à usage général montre vite ses limites quand on cherche à exécuter des centaines voire des milliers de tâches en parallèle.

---
## const et mutable en C++11

C++11 a introduit le support des threads au niveau du langage. Cela ne se résume pas au simple ajout de la bibliothèque `<thread>`, mais à la modification du modèle mémoire du langage.

Par exemple, l'initialisation (et non l'utilisation) d'objects statiques est maintenant thread-safe (garantie nécessaire à la bonne initialisation des mutex).

Mais aussi, quel est le problème avec ce code en C++98?
```cpp
short n1 = 0;
short n2 = 0;

void thread1() {
    while (n1 < 10000) {
        n1 += 1;
    }
}
void thread2() {
    while (n2 < 10000) {
        n2 += 1;
    }
}```

---
## Parallélisme vs Concurrence
Ce sont deux concepts bien distincts:
- le parallélisme consiste à travailler en **parallèle** sur des jeux de données distincts (isolés). Chaque tâche ayant son propre espace de travail, il n'y a pas de précaution particulière à adopter.
- quand plusieurs tâches partagent une même donnée et qu'au moins l'une d'entre-elles la modifie, il y a risque de race condition. Les tâches sont alors en **concurrence** (compétition) pour se partager une même ressource qui devient un goulot d'étranglement. Une synchronisation (régulation) des accès devient nécessaire.

En plus de compliquer le code, l'utilisation de primitives de synchronisation ralentit son exécution. Une règle (XXX) stipule que si votre code passe x% de son temps à se synchronizer, vous ne pourrez pas avoir plus de (100/X) threads qui s'exécutent en même temps (quelque soit le nombre de CPU):
- 10% du temps à se synchronizer => 10 threads max à s'exécuter
- 25% du temps à se synchronizer => 4 threads max à s'exécuter

En pratique, à partir 4 CPU, il devient très facile de passer 30% de son temps à se synchronizer...

C'est pourquoi le temps passé dans les sections critiques doit être le plus court possible.

---
## Programmation lock-free

Une section critique est identifiée par un début (prise d'un verrou) et une fin (libération d'un verrou). C'est cette opération de verouillage (lock) qui force les flux parallèles à se linéariser et qui limite donc la scalibilité du traitement.

En général, il est nécessaire de recourir à un verrou logiciel (mutex) lors de l'accès concurrent à ses données pour éviter leur corruption. Mais certaines opérations atomiques (effectuées en une seule fois) peuvent être sécurisées via des primitives spéciales du processeur (de type *compare and swap*).
- un support hardware est donc nécessaire

En réalité il y a toujours un verrouillage d'effectué mais au niveau du processeur (au niveau du cache partagé et de la RAM)
- c'est plus rapide mais ça reste plus lent qu'un accès classique non protégé (à cause 

La programmation *lock-free* est très tentante mais il est encore plus facile de faire des bêtises qu'avec les mutex:

```cpp
std::atomic<int> count{0};
MyClass * ptr = nullptr;

void alloc() {
    if (count == 0) {
        ptr = new MyClass;
    }
    ++count:
    return ptr;
}

void release() {
    if (--count == 0) {
        delete ptr;
    }
}
```

---

Certaines structures de données peuvent être implémentées sans lock via des primitives atomiques:
- pile (stack)
- liste simplement chaînées (queue)
- set
- table de hachage

D'autres peuvent même être implémentées sans primitive atomique du tout:
- liste avec producteur unique et consommateur unique (single-reader single-writer)

Mais il semble impossible de le faire pour certaines structures de données classique commme la liste doublement chaînée (std::list).

---
## Boost.LockFree

Boost LockFree fournit des conteneurs capables de fonctionner sans lock:
- queue
- stack
- single writer single producer queue

On spécifie lors de leur instantiation si l'on souhaite un redimensionnement dynamique ou non:
- auquel cas le conteneur peut être bloquant si un redimensionnement est nécessaire

```cpp

```
---
## Thread pool

Dans la mesure où un thread est une ressource coûteuse à créer, une première façon d'abstraire leur utilisation est d'architecturer son code pour qu'il manipule des tâches dont l'exécution est déléguée à un objet spécifique (un *task runner*).

Cet objet va en interne créer un groupe de threads constament en attente d'exécuter une nouvelle tâche. Le *task runner* s'occupe donc de répartir le travail en fonction de la capacité de la machine (nombre de CPU). L'intérêt majeur est:
- rentabiliser le coût de création d'un thread en le recyclant une fois sont travail terminé
- rendre le code plus simple à lire et écrire via le concept de tâche 

Exemple de bibliothèques:
- Microsoft PPL
- QtConcurrent
- std::async
- Win32 Thread Pool API

---
## Boost Thread Group

Boost ne fournit pas de thread pool à proprement parler.

Boost.ThreadGroup est une classe très simple qui facilite la gestion d'un groupe de threads, mais ne permet pas de leur assigner une liste de tâches à exécuter.

```cpp
boost::thread_group threads;

for (size_t i = 0; i < std::thread::hardware_concurrency(); ++i) {
    threads.create_thread([]{
        // do something
    });
}

threads.join_all();```

En tant que tel, cette classe n'est donc pas très utile. Mais elle se combien bien avec Boost.Asio par exemple (qui au contraire ne s'occupe que de l'ordonancement de tâches à exécuter).

---
## Boost Asio

La vocation première de Boost.Asio n'est pas d'être un thread pool mais il est assez facile de l'utiliser à cette fin. Tout comme Boost.ThreagGroup, c'est une lib header only.

Les deux fonctions de base de `service_io` sont:
- `post`: ajouter une tâche à exécuter
- `run` : exécuter les tâches en cours d'attente

Il suffit de créer plusieurs threads en leur donnant à chacun la fonction `service_io::run()` à exécuter pour obtenir un traitement en parallèle des tâches dans la liste.

```cpp
asio::io_service asio_service;

// créer des tâches à exécuter
for (int i = 0; i < 100; ++i) {
	asio_service.post([]{
		// faire quelque chose
	});
}

boost::thread_group threads;
// créer autant de threads que l'on a de CPU
for (size_t i = 0; i < std::thread::hardware_concurrency(); ++i) {
    threads.create_thread([]{
        asio_service.run();
    });
}

// attendre la fin du traitement
threads.join_all();```
---
## Boost.Asio

`service_io::run()` rend la main dès qu'il n'y a plus de tâches en attente d'exécution. Les threads créés terminent alors leur exécution et ne peuvent pas être recyclés ultérieurement pour un autre lot de tâches à traiter.

Pour éviter cela, il suffit de créer une instance de `asio::work`. Cette classe ne fait rien d'autre qu'informer Asio qu'il y a toujours un travail en cours et qu'il ne faut donc pas terminer l'exécution de `run()`.

Le code devient:

```cpp
asio::io_service asio_service;
auto asio_work = std::make_unique_ptr<asio::io_service::work>();

boost::thread_group threads;
// créer autant de threads que l'on a de CPU
for (size_t i = 0; i < std::thread::hardware_concurrency(); ++i) {
    threads.add(std::bind(&asio::service_io::run, asio_service));
}

// A partir d'ici, les threads sont en attente d'un tâche```

```cpp
// 1er lot de tâches à exécuter
for (size_t i : irange(0, 100)) {
    service.post([]{
        std::this_thread::sleep_for(100ms);
    });
}

// participer au traitement
service.poll();

// 2eme lot de tâches à exécuter
for (size_t i : irange(0, 100)) {
	service.post(std::bind(&std::this_thread::sleep_for, 100ms));
}

// attendre la fin du traitement
asio_work.reset();
threads.join_all();```

---
## Parallélisation de boucles

OpenMP
std::parallel_for

---
## Parallélisation d'un traitement (map/reduce)

std::vector<int> numbers;

// calculer la racine carrée de chaque élément

version maison:
  - la race condition est sur l'iterateur, pas le vecteur

```cpp
std::atomic<int> pos{0};

auto process = [&pos]{
	for (;;) {
		const size_t i = pos++;
		if (i &lt; numbers.size()) {
			numbers[i] = sqrt(numbers[i]);
		}
		else {
			break;
		}
	}
};
QtConcurrent::run(process);

thread_group pool;
for (int i = 0; i &lt; thread::hardware_concurrency(); ++i)) {
	pool.create(process);
}
pool.join_all();
```

QtConcurrent::Map
---
template: inverse

## Concurrence
---
## Producteur - Consommateur

Boost.LockFree
---
## Async
Principe: exécution asynchrone (concurrente) des entrées sorties.
Bien qu'il y ait concurrence, il n'y a pas nécessairement parallélisme
  - Par exemple, lire sur plusieurs sockets depuis le même thread
L'intérêt est de maximiser la capacité de travail d'un même thread.
  - Pas besoin de synchronisation
  - Pas de changement de contexte entre threads
  - Conçu pour permettre l'exécution concurrente de dizaines de milliers d'opérations

Pour que les opérations de lecture/écriture soient asynchrones, elles doivent être non bloquantes.
  - Exemple typique: lecture/écriture sur les sockets (poll)

Nécessite un support de la part du système d'exploitation
  - Windows: IOPC, Completion Routine, APC: lecture/écriture asynchrone possible sur les fichiers
---
## Exemple: envoie de (gros) fichier sur un serveur

```cpp
// envoie un fichier à un client sur le réseau
void sendFile(filesystem::path fileName, Socket &amp; socket) {
	// ouvrir le fichier
	File file(fileName);
	
	// lire par blocks de 4Ko et envoyer sur le réseau
	uint8_t buffer[4096];
	while (const auto size = file.Read(buffer, sizeof(buffer))) {
		socket.Write(buffer, size);
	}
}```

Avantages de ce code synchrone:
- simple à comprendre

Inconvénient:
- pendant que l'on envoie un fichier on ne peut rien faire d'autre
---
## Utilisation de threads

Une façon simple de régler le problème est de faire exécuter cette fonction dans un thread séparé.

C'est une approche qui fonctionne bien quand on a un long traitement à effectuer en tâche de fond.

Mais l'utilisation de threads devient problématique dès qu'on dépasse le nombre de CPU disponibles:
  - un thread est une ressource système coûteuse (consommation mémoire de la pile dédiée, temps CPU lors des changements de contexte)

Dans le cas d'un serveur où des milliers de clients peuvent se connecter pour télécharger des fichiers, cette solution n'est pas adaptée.
---
La solution la plus performante est d'avoir un seul thread qui gère toutes les entrées-sorties sur la carte réseau.

Chaque opération doit être courte et rendre rapidement la main pour permettre à une autre opération de s'exécuter.

struct Data {
    File file;
	uint8_t buffer[4096];
	Socket socket;
};

// envoie un fichier à un client sur le réseau
void sendFileAsync(filesystem::path fileName, Socket &amp; socket) {
    auto data = make_unique<Data>();	
	[data=move(data)]{
		const auto size = data->file.Read(data->buffer, sizeof(data->buffer));
		if (size > 0) {
			return [data=move(data)]{
				socket.Write(data->buffer, size);
			};
		}
	}

	void sendNext(unique_ptr<Data> data) {
		service.post([data=move(data)]{
			const auto size = data->file.Read(data->buffer, sizeof(data->buffer));
			if (size > 0) {
				service.post([data=move(data)]{
					socket.Write(data->buffer, size);
					sendNext(std::move(data));
				});
			}
		});
	}
	
	sendNext(data){
		// risque un stack overflow!
		file.ReadAsync(data->buffer, sizeof(data->buffer), [](data, size){
			data->socket.WriteAsync(data->buffer, size, []{
				sendNext(std::move(data));
			});
		});
	}
	
	// pas de stack overflow - la gestion des erreurs est à revoir
	sendNext(data){
		file.ReadAsync(data->buffer, sizeof(data->buffer)).then([](data, size){
			socket.WriteAsync(data->buffer, size).then([]{ 
				sendNext();
			});
		});
	}

	void sendNext(unique_ptr<Data> data) {
		service.post([data=move(data)]{
			io.async_read
			const auto size = data->file.Read(data->buffer, sizeof(data->buffer));
			if (size > 0) {
				service.post([data=move(data)]{
					socket.Write(data->buffer, size);
					sendNext(std::move(data));
				});
			}
		});
	}```
---
## Parallélisme et Concurrence

Le comité C++ dispose d'un groupe d'étude dédié à ces questions: le WG XX Parallelism and Concurrency.

Plusieurs propositions sont à l'étude:
- Concurrency TS: améliorer std::future pour le rendre composable (std::future::thend). S'inspire de boost::future.
- Coroutines

Il y a peu de chances de voir cela intégré à C++17 (qui sera finalement une norme mineure) mais le compilateurs supportent déjà certaines de ces fonctionnalités expérimentales.
---
## Comment ça marche?

Le principe est de fournir des callbacks (lambdas) qui sont exécutées une fois l'opération demandée terminée.

Sous Windows, on parle de Completion Routine.

Les callbacks peuvent être chaînées entre-elles à la javascript.
Exemple avec la PPL:
```cpp
async([]{
    read();
}).then([]{
    write()
});
```
On connecte ainsi des callback pour former un graphe.

Les noeuds du graphe représentent les valeurs finales calculées qui sont manipulées via des promesses futures.

Avec certaines bibliothèques, le fait d'abandonner un future propage l'annulation en amont dans le graphe (cancellable).

La bibliothèque standard C++ n'offre aucune facilité de cette sorte.
---
## Coroutines (resumable function)
Le commité C++ focalise son travail sur l'ajout de coroutines à C++.

Une routine classique (fonction, procédure) permet d'être appelée ainsi que de rendre la main à l'appelant.
Une coroutine est une "routine généralisée" qui supporte comme opération supplémentaire d'être suspendue et relancée (là où elle avait été suspendue).

On trouve ce mécanisme dans divers langages : C# (await/yield), Python, Ruby...

Une coroutine est une fonction dont l'exécution peut être suspendue puis reprise.

L'idée d'une coroutine est d'offrir une vue synchrone de plusieurs blocs de code conçus pour être exécutés de façon asynchrone.

Nos callback précédentes sont regroupées au sein d'une même fonction ce qui rend le code beaucoup plus lisible.

```cpp
// La coroutine offre l'illusion d'un exécution synchrone
future<void> f(){
    co_await read();
    co_return write()
}

auto r = co_await f();
```

Au final, c'est comme si on avait une routine synchrone classique. Sauf qu'une coroutine accepte d'être interrompue en plein milieu de son exécution.

Elle reprendra là où elle s'était arrêtée à son prochain appel (au lieu de commencer au début de la fonction).

Contrairement à la composition de future, les coroutines demandent un support spécifique au niveau du langage et donc une modification de ce dernier pour supporter de nouveaux mot-clés: co_await, co_yield, co_return.

Les coroutines sont une proposition en cours d'étude par le comité.

Elles sont supportées par Visual C++ 2015. Quid de Clang?

Elles sont de type *stackless*, c.à.d que l'OS n'alloue pas de pile pour la coroutine: c'est au compilateur d'allouer la mémoire nécessaire sur le tas pour sauvegarder le contexte d'appel et le restaurer (activation frame). Cela demande plus de travail mais permet de créer des millions de coroutines.

---
.left-column[
  ## Boost. LockFree
  ## Concurrence
  ### Thread
  ### Concurrence
  ### Parallélisme
  ### Race condition
]
.right-column[
  Modèle traditionnel du producteur-consomateur.
]
---
.left-column[
  ## Boost.Asio
]
.right-column[
  A la base, il s'agit d'une bibliothèque de programmation réseau
  En cours de normalisation (sous forme de TS)
]
---
Threads, async & future, Boost.Asio, coroutines

```cpp
class X {
public:
    int n; // OK
};
```
