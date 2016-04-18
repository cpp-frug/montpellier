# Programmation parallèle et asynchrone en C++
## Plan

- Rappels sur les threads
- Parallélisme vs Concurrence
- Programmation LockFree (Non blocking algorithms)
- Pool de threads
- Boost ASIO
- Map/Reduce
- Promesses et objets futurs

- Async (non blocking IO)
- Coroutines

---
## Threads

> Quand un programmeur a un problème de performance, il pense aux threads. Maintenant, il proba deux lèmes.

Qu'est-ce qu'un thread?

D'un point de vue technique, un *processus léger* est un ensemble de ressources système connues et gérées par l'OS:
- des zones mémoires réservées (pile, TLS, contexte...)
- un fil d'exécution enregistré auprès de l'ordonnanceur

Mais d'un point de vue plus abstrait, quel est le concept implémenté?

---

Un thread est automate fini dont les changements d'états sont gérés de façon transparente par le système d'exploitation (en exécution, en attente, bloqué...).
- Quand un thread est bloqué (E/S, page fault...) ou interrompu par l'ordonnanceur, son état est sauvegardé pour être remplacé par le contexte d'exécution d'un autre thread en attente d'exécution (*context switch*).

(2:30 https://www.youtube.com/watch?v=wxXIbaJBZlE)

Note: d'un point de vue utilisateur, un thread n'est pas qu'un *fil d'exécution*, c'est aussi une pile qui occupe un espace mémoire non négligeable.
- c'est une source importante de limitation à leur utilisation massive

---
## Les threads en C++11

C++11 a introduit le support des threads au niveau du langage. Cela ne se résume pas au simple ajout de la bibliothèque `<thread>`, mais à la modification du modèle mémoire du langage.
- Un thread est défini par le standard comme *un flot unique de contrôle dans un programme* ($1.10)

Par exemple, l'initialisation (et non l'utilisation) d'objets statiques est maintenant thread-safe (garantie nécessaire à la bonne initialisation des mutex).

Mais aussi, quel est le problème avec ce code en C++98?
```cpp
short n1 = 0;
short n2 = 0;

void thread1() {
    while (true) {
        n1 += 1;
    }
}
void thread2() {
    while (true) {
        n2 += 1;
    }
}```
---
## Les threads en C++11
### Travaux en cours

C++11 a posé les bases sur lesquelles construire des primitives plus évoluées en terme de programmation parallèle. Le comité travaille maintenant à introduire de meilleures abstractions dans les années qui viennent. 
En particulier:
- Standardisation de Boost.ASIO
- Algorithmes parallèles dans la STL
- Future composables
- Co-routines

---
## Parallélisme vs Concurrence
La **concurrence** est l'accès simultané (compétitif) à une même ressource. Elle fait intervenir la notion de synchronisation.
- Si au moins une tâche modifie une ressource pendant qu'au moins une autre la lit, une **data race** est possible. Il faut alors synchroniser (restreindre) l'accès à cette ressource partagée.

Le **parallélisme** consiste à exécuter de façon simultanée plusieurs tâches qui peuvent être totalement indépendante. Le parallélisme peut donc se faire sans concurrence et donc synchronisation explicite.
- Idéalement chaque tâche travaille (en écriture) sur une espace mémoire qui lui est dédié. Sans risque de collision, il n'y a pas de précaution particulière à prendre.

L'utilisation directe des threads dans sonc ode tend à mélanger intimement ces deux aspects. Mais elles gagnent à être distinguées.

Concrètement, la difficulté du multi-threading se trouve dans la concurrence, pas dans le parallélisme. L'idée des nombreux travaux en cours est de tendre vers un style explicitement parallèle en lieu et place d'une concurrence explicite.

---
## Le coût de la concurrence

Une section critique est identifiée par un début (verouillage) et une fin (libération du verrou). Elle forme ainsi un goulot d'étranglement par lequel les flux parallèles sont contraints de se sérialiser ("dé-paralléliser").

La [loi d'Amdahl](https://fr.wikipedia.org/wiki/Loi_d%27Amdahl) montre que le gain maximal attendu est inversement proportionnel à la taille du code non parallélisable (sections critiques):
- 5% du temps en SC => vitesse x6 avec 8 coeurs
- 10% du temps en SC => vitesse x3 avec 4 coeurs (x8 à partir de 32 coeurs!)
- 25% du temps en SC => vitesse x3 avec 8 coeurs

C'est pourquoi le temps passé dans les sections critiques doit être le plus court possible.
- L'idéal est de ne pas avoir besoin de se synchroniser!

---
## Programmation lock-free

En général, on recourt à un verrou logiciel (mutex) pour éviter la corruption des données partagées. Mais certaines modification simples (effectuable en une seule instruction machine) peut-être sécurisées via des primitives spéciales du processeur (de type *compare and swap*).
- Un support hardware est donc nécessaire

En réalité il y a toujours un verrouillage d'effectué mais au niveau hardware (cache du processeur)
- c'est beaucoup plus rapide qu'un aller-retour dans le noyau de l'OS mais ça reste plus lent qu'une instruction machine classique

---

La programmation *lock-free* est très tentante mais elle se révèle encore plus difficile et subtile que l'utilisation correcte des mutex.

```cpp
std::atomic<int> count{0};
MyClass * ptr = nullptr;

MyClass * useMyClass() {
    if (count == 0) {
        ptr = new MyClass;
    }
    ++count:
    return ptr;
}

void releaseMyClass() {
    if (--count == 0) {
        delete ptr;
    }
}```

---
## Boost.LockFree

Certaines structures de données peuvent être implémentées sans lock via des primitives atomiques (stack, queue, set, hash map). D'autres en revanche semblent impossible à implémenter sans verrou global (std::list).

Boost LockFree fournit des conteneurs capables de fonctionner sans lock:
- queue
- stack
- single writer single producer queue

Ils sont adaptés au modèle producteur / consommateur.

Le spsc_queue est une implémentation qui se passe même des primitives atomiques (ne dépend que des barrières mémoire) mais qui ne fonctionne que si elle est utilisée avec un seul producteur et un seul consommateur (single-reader single-writer)

---

## Boost.LockFree
### Usage

On spécifie lors de la declaration si l'on souhaite un redimensionnement dynamique ou non
- auquel cas le conteneur peut être bloquant si un redimensionnement est nécessaire

```cpp

```
---
## Utilisation de threads

Une façon simple de régler le problème est de faire exécuter cette fonction dans un thread séparé.

C'est une approche qui fonctionne bien quand on a un long traitement à effectuer en tâche de fond.

Mais l'utilisation de threads devient problématique dès qu'on dépasse le nombre de CPU disponibles:
  - un thread est une ressource système coûteuse (consommation mémoire de la pile dédiée, temps CPU lors des changements de contexte)

Dans le cas d'un serveur où des milliers de clients peuvent se connecter pour télécharger des fichiers, cette solution n'est pas adaptée.
---
## Thread pools

Dans la mesure où un thread est une ressource coûteuse à créer, une première façon d'abstraire leur utilisation est d'architecturer son code pour qu'il manipule des tâches dont l'exécution est déléguée à un objet spécifique (un *task runner*).

Cet objet va en interne créer un groupe de threads constament en attente d'exécuter une nouvelle tâche (des *worker threads*). Le *task runner* s'occupe donc de répartir le travail en fonction de la capacité de la machine (nombre de CPU). L'intérêt majeur est:
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

Astuce: donner un identifiant (chaîne de caractères) aux tâches à exécuter qui sert à nommer temporairement le worker thread qui exécute notre tâche. Cela facilite le débogage sur des machines avec beaucoup de processeurs.

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
## Parallélisation avec Map/Reduce (Qt)

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

---
## Async

A l'usage les threads posent vite problème parce qu'ils ne sont pas composables
- en particulier, retourner un résultat depuis un thread et le "brancher" comme entrée d'un autre thread n'est pas trivial

### Async

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
## Travaux en cours du comité C++

OpenMP

Le comité C++ dispose d'un groupe d'étude dédié à ces questions: le WG XX Parallelism and Concurrency.

Plusieurs propositions sont à l'étude:
- Concurrency TS: améliorer std::future pour le rendre composable (std::future::thend). S'inspire de boost::future.
- Coroutines
- Algorithmes parallèles dans la STL (std::parallel_for, ...)

Il y a peu de chances de voir cela intégré à C++17 (qui sera finalement une norme mineure) mais le compilateurs supportent déjà certaines de ces fonctionnalités expérimentales.


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
