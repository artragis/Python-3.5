Une nouvelle version du langage [Python](https://www.python.org/) (et par conséquent de son implémentation principale CPython) sort aujourd'hui. Estampillée 3.5, cette version fait logiquement suite à [la version 3.4](https://docs.python.org/3/whatsnew/3.4.html) parue il y a un an et demi[^ndbp_date_34]. Tandis que cette dernière apportait principalement des ajouts dans la bibliothèque standard ([asyncio](https://docs.python.org/3/library/asyncio.html), [enum](https://docs.python.org/3/library/enum.html), [ensurepip](https://docs.python.org/3/library/ensurepip.html), [pathlib](https://docs.python.org/3/library/pathlib.html), etc.), les nouveautés les plus visibles de la version 3.5 concernent des changements syntaxiques avec deux nouveaux mot-clés, un nouvel opérateur binaire, la généralisation de l'*unpacking* et la standardisation des annotations de fonctions.

[^ndbp_date_34]: Le 16 Mars 2014 pour être précis. 

# TL;DR - Résumé des nouveautés

Les plus pressés peuvent profiter de ce court résumé [des principales nouveautés](https://docs.python.org/3.5/whatsnew/3.5.html) :

 - [PEP 492](https://www.python.org/dev/peps/pep-0492) : les coroutines deviennent une construction spécifique du langage. Cette gestion dans l'interpréteur se fait via deux nouveaux mots-clés (`async` et `await`) et vise à compléter le support de la « programmation asynchrone » dans Python.
 - [PEP 465](http://www.python.org/dev/peps/pep-0465) : un nouvel opérateur binaire `@` est introduit pour gérer la multiplication matricielle.
 - [PEP 484](https://www.python.org/dev/peps/pep-0484/) : les annotations apposables sur les paramètres et la valeur de retour des fonctions sont maintenant standardisées et servent uniquement à préciser le type de ces éléments.
 - [PEP 448](https://www.python.org/dev/peps/pep-0448/) : les opérations d'*unpacking* sont généralisées et permettent maintenant d'être combinées.

<--COMMENT

Pour les 4 principaux thèmes, une section avec le contexte, ce que ça apporte et une note de conclusion

COMMENT-->

# Principales nouveautés

## Support des coroutines -- PEP 492

[[i]]
| Il n'est malheureusement pas possible de tout détailler dans cet article. Cette section peut nécessiter que vous ayez quelques notions de *programmation asynchrone* pour bien comprendre cette nouveauté.

### Contexte

Remise récemment à la mode suite à la grande popularité acquise par [node.js](https://nodejs.org/), la programmation dite "asynchrone" est possible depuis de nombreuses années en Python, notamment pour créer des applications web,  grâce à certaines bibliothèques : [Twisted](https://twistedmatrix.com/trac/) existe depuis 12 ans (2002), [Tornado](http://www.tornadoweb.org/en/stable/) a été libéré par [FriendFeed](http://blog.friendfeed.com/) il y a 6 ans (2009) au même moment où le développement de [gevent](http://www.gevent.org/) commençait. 

La possibilité d'utiliser une boucle d’événements pour ordonnancer les opérations et ne pas bloquer le *thread* principal de l'application durant les opérations d'entrées et sorties est particulièrement utile pour la réalisation d'applications web. Cette popularité récente a poussé [Guido van Rossum](https://fr.wikipedia.org/wiki/Guido_van_Rossum), créateur et BDFL[^ndbp_bdfl] de Python, à standardiser cette approche en synthétisant les idées des bibliothèques populaires précédemment citées et de proposer une implémentation typique dans la bibliothèque standard : c'est le module [asyncio](https://docs.python.org/3/library/asyncio.html) disponible depuis Python 3.4. Ce module n'a pas pour but de remplacer les bibliothèques cités précédemment mais de proposer une implémentation standardisée d'une boucle événementielle et de mettre à disposition quelques fonctionnalités bas niveaux (*Tornado* peut ainsi [être utilisé avec asyncio](http://tornado.readthedocs.org/en/latest/asyncio.html)). 

[^ndbp_bdfl]: *"Benevolent Dictator for Life"* ("dictateur bienveillant à vie")

Tandis que *node.js* utilise, par exemple, un système de "fonctions de rappel" (*callback*) pour ordonnancer les différentes étapes d'un algorithme, Python et *asyncio* utilisent des coroutines permettant d'écrire des fonctions asynchrones qui ressemblent à des fonctions procédurales.  Les [coroutines](https://fr.wikipedia.org/wiki/Coroutine) ressemblent beaucoup aux fonctions à ceci prêt que leur exécution peuvent être suspendu et reprendre à plusieurs endroit dans la fonction. Python dispose déjà de constructions de ce genre : les générateurs[^ndbp_gen]. C'est ainsi avec les générateurs que *asyncio* a été initialement développé.

Pour l'exemple, avec Python 3.4, *asyncio*, la bibliothèque [aiohttp](https://github.com/KeepSafe/aiohttp)[^ndbp_aiohttp] pour faire des requêtes http et la bibliothèque [aiofiles](https://github.com/Tinche/aiofiles/) pour écrire dans des fichiers locaux, voici un code qui va télécharger le contenu de la page d’accueil de *zeste de savoir* et l'enregistrer sur le disque par morceau de 512 octets, le tout de façon asynchrone :

```python
import asyncio
import aiohttp
import aiofiles

chunk_size = 512

@asyncio.coroutine           # On déclare cette fonction comme étant une coroutine
def fetch_page(url, filename):
    # À la ligne suivante, la fonction est interrompu tant que la requête n'est pas revenu
    response = yield from aiohttp.request('GET', url)
    assert response.status == 200
    
    # Création du fichier de sortie en asynchrone
    fd = yield from aiofiles.open(filename, mode='wb')
    try:
        while True:
            # On lit le contenu au fur et à mesure de son arrivée et on interromps la fonction en attendant
            chunk = yield from response.content.read(chunk_size)
            if not chunk:
                break
            # on interromps la fonction le temps de l'écriture dans le fichier
            yield from fd.write(chunk)

    finally:
        # On ferme le fichier
        yield from fd.close()

if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(fetch_page('http://zestedesavoir.com/', 'out.html'))
```

[^ndbp_gen]: Les générateurs sont en réalité [des formes particulières de coroutines](https://en.wikipedia.org/wiki/Coroutine#Comparison_with_generators)

[^ndbp_aiohttp]: Bibliothèque qui propose des fonctions d'entrées/sorties sur le protocole http.

### Les nouveaux mot-clés

Python 3.5 introduit deux nouveaux mot-clés : `async` et `await` (comme en C#). De l'extérieure, `async` vient remplacer le décorateur `asyncio.coroutine` et `await` l'expression `yield from`. Le code précédent peut donc s'écrire avec Python 3.5 de la façon suivante :

```python hl_lines="8 10 14 18 22 26"
import asyncio
import aiohttp
import aiofiles

chunk_size = 512

# On déclare cette fonction comme étant une coroutine
async def fetch_page(url, filename):
    # À la ligne suivante, la fonction est interrompu tant que la requête n'est pas revenu
    response = await aiohttp.request('GET', url)
    assert response.status == 200
    
    # Création du fichier de sortie en asynchrone
    fd = await aiofiles.open(filename, mode='wb')
    try:
        while True:
            # On lit le contenu au fur et à mesure de son arrivée et on interromps la fonction en attendant
            chunk = await response.content.read(chunk_size)
            if not chunk:
                break
            # on interromps la fonction le temps de l'écriture dans le fichier
            await fd.write(chunk)

    finally:
        # On ferme le fichier
        await fd.close()

if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(fetch_page('http://zestedesavoir.com/', 'out.html'))
```

Mais les modifications sont plus importantes que ces simples synonymes. Tout d'abord une coroutine déclaré avec `async` n'est PAS un générateur. Même si en interne les deux types de fonctions partagent une grande partie de leur implémentation, il s'agit de types différents et il est possible que les différences se creusent dans les prochaines versions de Python. Pour marquer cette différence et le fait que des générateurs sont une forme retreinte de coroutines, il est possible dans Python 3.5 d'utiliser des générateurs partout où une coroutine est attendu, mais pas l'inverse. Le module *asyncio* continue ainsi à supporter les deux formes.

L'ajout de ces mots clés a aussi été l'occasion d'ajouter la possibilité d'itérer de manière asynchrone sur des objets. Ce support, assuré en interne par les nouvelles méthodes `__aiter__` et `__anext__` pourra être utiliser par des bibliothèques comme *aiohttp* pour simplifier les itération grâce à la nouvelle instruction `async for` :

```python hl_lines="17"
import asyncio
import aiohttp
import aiofiles

chunk_size = 512

# On déclare cette fonction comme étant une coroutine
async def fetch_page(url, filename):
    # À la ligne suivante, la fonction est interrompu tant que la requête n'est pas revenu
    response = await aiohttp.request('GET', url)
    assert response.status == 200
    
    # Création du fichier de sortie
    fd = await aiofiles.open(filename, mode='wb')
    try:
        # On lit le contenu au fur et à mesure de son arrivée et on interromps la fonction en attendant
        async for chunk in response.content.read_chunk(chunk_size):
            await fd.write(chunk)
    finally:
        await fd.close()
    
if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(fetch_page('http://zestedesavoir.com/', 'out.html'))
```

De la même façon, des *context manager* asynchrone, utilisant l'instruction `async with`, font leur apparition, en utilisant les méthodes `__aenter__` et `__aexit__`, permettant en Python 3.5 d'écrire des coroutines de la forme suivante :

```python hl_lines="14"
import asyncio
import aiohttp
import aiofiles

chunk_size = 512

# On déclare cette fonction comme étant une coroutine
async def fetch_page(url, filename):
    # À la ligne suivante, la fonction est interrompu tant que la requête n'est pas revenu
    response = await aiohttp.request('GET', url)
    assert response.status == 200
    
    # Ouverture du fichier de sortie
    async with aiofiles.open(filename, mode='wb') as fd:
        # On lit le contenu au fur et à mesure de son arrivée et on interromps la fonction en attendant
        async for chunk in esponse.content.read_chunk(chunk_size):
            await fd.write(chunk)

if __name__ == "__main__":
    asyncio.get_event_loop().run_until_complete(fetch_page('http://zestedesavoir.com/', 'out.html'))
```

L'exemple ci-dessus montre clairement l'objectif de ces ajouts : Si les `async` et `await` sont ignorés, la coroutine est fortement similaire à une implémentation utilisant des fonctions classiques.

Enfin notez que les expressions `await`, au delà de leur précédence beaucoup plus faible, sont moins restreintes que les `yield from` et peuvent être placé partout où une expression est attendu. Ainsi les codes suivants sont valides :

```python
if await foo:
    pass

# ! Différent de "async with"
with await bar:
     pass

while await spam:
     pass

await spam + await egg
```

### Impacte sur vos codes

Pour vos codes existants cette version voit logiquement la dépréciation de l'utilisation de `async` et `await` comme nom de variable. Pour le moment un simple avertissement sera émit qui sera transformé en erreur dans Python 3.7. Ces modifications restent donc parfaitement rétro-compatible avec Python 3.4. 

Si vous utilisez *asyncio*, rien ne change pour vous et vous pouvez continuer à utiliser les générateurs si vous souhaitez conserver le support de Python 3.4. L'ajout des méthodes de la forme `__a****__` peut vous permettre cependant de supporter les nouveautés de Python 3.5 mais n'est pas obligatoire.

## Opérateur de multiplication matricielle -- PEP 465

L'introduction d'un opérateur binaire n'est pas courant dans python. Aucun dans la série 3.x, le dernier ajout semble être [l'opérateur `//` dans Python 2.2](https://www.python.org/dev/peps/pep-0238/)[^ndbp_op_bin]. Regardons donc pour quels raisons celui-ci a été introduit.


[^ndbp_op_bin]: Ces ajouts sont tellement peu courant qu'il est difficile de trouver des traces de ces modifications. L'opérateur `//` semble être le seul ajout de toute la série 2.

### Signification

Ce nouvel opérateur est dédié à la multiplication matricielle. En effet, comme tous ceux qui ont fait un peu de mathématiques algébriques doivent le savoir, pour une matrice on définit généralement la multiplication par l'opération suivante (pour des matrices $2*2$ :

$$
\begin{pmatrix}a&b \\ c&d \end{pmatrix} \cdot \begin{pmatrix}e&f \\ g&h \end{pmatrix} = 
\begin{pmatrix}a*e+b*g&a*f+b*h \\ c*e+d*g&c*f+d*h \end{pmatrix}
$$

or il est souvent aussi nécessaire d'effectuer des multiplications termes à termes :

$$
\begin{pmatrix}a&b \\ c&d \end{pmatrix} * \begin{pmatrix}e&f \\ g&h \end{pmatrix} = 
\begin{pmatrix}a*e&b*f \\ c*e&d*h \end{pmatrix}
$$

Tandis que certains langages spécialisés possèdent des opérateurs dédiés pour chacune de ces opérations[^ndbp_op_matmatlab] il n'y a en Python rien de similaire. Avec la bibliothèque [numpy](http://www.numpy.org), la plus populaire pour le calcul numérique dans l'éco-système Python, il est pour le moment nécessaire d'utiliser la méthode `dot` et d'écrire des lignes de la forme :

```python
S = (H.dot(beta) - r).T.dot(inv(H.dot(V).dot(H.T))).dot(H.dot(beta) - r)
```

ce qui n'aide pas à la lecture... Le nouvel opérateur `@` est introduit et dédié à la multiplication matricielle. Il permettra d'obtenir des expressions équivalentes de la forme :

```python
S = (H @ beta - r).T @ inv(H @ V @ H.T) @ (H @ beta - r)
```

Un peu mieux, non ?

[^ndbp_op_matmatlab]: Par exemple, avec Matlab et Julia, `*` / `.*` servent respectivement pour la multiplication matricielle et terme à terme.

### Impacte sur vos codes

Cette introduction devrait être anodine pour beaucoup d'utilisateurs. En effet aucun objet de la bibliothèque standard ne va l'utiliser[^ndbp_nused]. Cet opérateur binaire sera principalement utilisé par des bibliothèques annexes, à commencer par *numpy*. 

[^ndbp_nused]: Ce qui est rare mais existe déjà. En particulier l'objet `Elipsis` créé avec `...`.

Si vous souhaitez supporter cet opérateur, trois méthodes spéciales peuvent être implémentés : `__matmul__` et `__rmatmul__` pour la forme `a @ b` et `__imatmul__` pour la forme `a @= b`, de façon similaire aux autres opérateurs opérateurs.

A noter qu'il est déconseillé d'utiliser cet opérateur pour autre chose que les multiplications matricielles.

### Motivation

L'introduction de cet opérateur est un modèle du genre. L'ajout de cet opérateur peut sembler très spécifique mais est pleinement justifié. La lecture de la PEP est très instructive et développé à ce sujet. Pour la faire adopté, les principales bibliothèques scientifiques en Python ont préparé cette PEP ensemble pour arriver à une solution convenant à la grande partie de la communauté scientifique, assurant dès lors l'adoption rapide de cet opérateur. La PEP précise ainsi l’intérêt et les problèmes engendrés par les autres solutions utilisés jusque là. Enfin cette PEP a été fortement appuyé par [la grande popularité de Python dans le monde scientifique](https://www.python.org/dev/peps/pep-0465/#but-isn-t-matrix-multiplication-a-pretty-niche-requirement). Ainsi on apprend que `numpy` est le module n’appartenant pas à la bibliothèque standard le plus utilisé parmit tous les codes Python présent sur Github et ce sans compter d'autres bibliothèques comme `pylab` ou `scipy` qui vont aussi profiter de cette modification et présentent parmi la liste des bibliothèques les plus communes.

## Annotations de types -- PEP 484

Les annotations de fonctions [existent depuis Python 3](https://www.python.org/dev/peps/pep-3107/) et permettent d'attacher des objets python quelconques aux arguments et valeur de retours des fonctions :

```python
def ma_fonction(param: "Mon annotation", param2: 42) -> str:
    pass
```

Depuis le début, il est possible d'utiliser n'importe quels objets comme annotation et l'interpréteur les ignores totalement. C'est à la charge de l'utilisateur d'en faire une utilisation particulière. Avec Python 3.5, bien que rien 

## *Unpacking* généralisé -- PEP 448

L'*unpacking* est une opération permetant de séparer un itérable en plusieurs 
variables :

```python
>>> l = (1, 2, 3, 4)
>>> a, b, *c = l
>>> a
1
>>> b
2
>>> c
(3, 4)
>>> d, e = c
>>> d
3
>>> e
4
```

Il est alors possible de passer un itérable comme argument de fonction comme 
si son contenu était passé élément par élément grâce à l'opérateur `*` ou de 
passer un dictionnaire comme arguments nommés avec l'opérateur `**` :

```python
>>> def spam(a, b):
...    return a + b
...
>>> l = (1, 2)
>>> spam(*l)
3    # 1 + 2
>>> d = {"b": 2, "a": 3}
>>> spam(**d)
5    # 3 + 2
```

Cette fonctionnalité restait jusqu'à maintenant limitée et les conditions 
d'utilisation très strictes. Deux de ces contraintes ont été levées...

## Support de l'*unpacking* dans les déclarations d'itérables

Lorsque vous souhaitez définir un `tuple`, `list`, `set` ou `dict` litéral, il 
est maintenant possible d'utiliser l'*unpacking*.

<table>
<tr>
<td>Python 3.4</td>
<td>Python 3.5</td>
</tr>
<tr>
<td>
```python
>>> tuple(range(4)) + (4,)
(0, 1, 2, 3, 4)
```
</td>
<td>
```python
>>> (*range(4), 4)
(0, 1, 2, 3, 4)
```
</td>
</tr>
<tr>
<td>
```python
>>> list(range(4)) + [4]
[0, 1, 2, 3, 4]
```
</td>
<td>
```python
>>> [*range(4), 4]
[0, 1, 2, 3, 4]
```
</td>
</tr>
<tr>
<td>
```python
>>> set(range(4)) + {4}
set(0, 1, 2, 3, 4)
```
</td>
<td>
```python
>>> {*range(4), 4}
set(0, 1, 2, 3, 4)
```
</td>
</tr>
<tr>
<td>
```python
>>> d = {'x': 1}
>>> d.update({'y': 2})
>>> d
{'x': 1, 'y': 2}
```
</td>
<td>
```python
>>> {'x': 1, **{'y': 2}}
{'x': 1, 'y': 2}
```
</td>
</tr>
<tr>
<td>
```python
>>> combinaison = long_dict.copy()
>>> combination.update({'b': 2})
```
</td>
<td>
```python
>>> combinaison = {**long_dict, 'b': 2}
```
</td>
</tr>
</table>

Notamment, il devient maintenant facile de sommer des itérables pour en former 
un autre.

```python
>>> l1 = (1, 2)
>>> l2 = [3, 4]
>>> l3 = range(5, 7)

# Python 3.5
>>> combinaison = [*l1, *l2, *l3, 7]
>>> combinaison
[1, 2, 3, 4, 5, 6, 7]

# Python 3.4
>>> combinaison = list(l1) + list(l2) + list(l3) + [7]
>>> combinaison
[1, 2, 3, 4, 5, 6, 7]
```

Cette dernière généralisation permet ainsi d'apporter une symétrie par rapport 
aux possibilités précédentes.

```python
# Possible en Python 3.4 et 3.5
>>> elems = [1, 2, 3, 4]
>>> fst, *other, lst = elems
>>> fst
1
>>> other
[2, 3]
>>> lst
4

# Possible uniquement en Python 3.5
>>> fst = 1
>>> other = [2, 3]
>>> lst = 4
>>> elems = fst, *other, lst 
>>> elems
(1, 2, 3, 4)
```

## Support de plusieurs *unpacking* dans les appels de fonctions

Cette généralisation se répercute sur les appels de fonctions ou méthodes : 
jusqu'à Python 3.4, un seul itérable pouvait être utilisé lors de l'appel à 
une fonction. Cette restriction est maintenant levée.

```python
>>> def spam(a, b, c, d, e):
...    return a + b + c + d + e
...
>>> l1 = (2, 1)
>>> l2 = (4, 5)
>>> spam(*l1, 3, *l2)        # Légal en Python 3.5, impossible en Python 3.4
15                           # 2 + 1 + 3 + 4 + 5
>>> d1 = {"b": 2}
>>> d2 = {"d": 1, "a": 5, "e": 4}
>>> spam(**d1, c=3, **d2)    # Légal en Python 3.5, impossible en Python 3.4
15                           # 5 + 2 + 3 + 1 + 4
>>> spam(*(1, 2), **{"c": 3, "e": 4, "d": 5})
15                           # 1 + 2 + 3 + 5 + 4
```

Notez que si un nom de paramètre est présent dans plusieurs dictionnaires, 
c'est la valeur du dernier dictionnaire qui sera prise en compte. 

D'autres généralisations [sont mentionnées dans la PEP](https://www.python.org/dev/peps/pep-0448/#variations) 
mais n'ont pas été implémentées à cause d'un manque de popularité parmi les 
developpeurs de Python. Il n'est cependant pas impossible que d'autres 
fonctionnalités soient introduites dans les prochaines versions.

# De plus petits changements

 - L'ajout d'un module `zipapp` améliorant le support et la création d'applications packagés sous forme d'archive `zip` introduit dans Python 2.6. Voir a [PEP 441](http://www.python.org/dev/peps/pep-0441).
 - Le retour de l'opérateur de formatage `%` pour les types de données `byte` et `bytearray` de la même façon qu'il était utilisable pour les types chaînes (`str`) dans Python 2.7. Voir la [PEP 461](https://www.python.org/dev/peps/pep-0461/).
 - Toujours pour les types `byte` et `bytearray`, une nouvelle méthode `hex()` permet maintenant de récupérer une chaîne représentant le contenu sous forme hexadécimale. Voir le [ticket 9951](https://bugs.python.org/issue9951).
 - La suppression des fichiers `pyo`. Voir [PEP 488](https://www.python.org/dev/peps/pep-0488/).
 - Un changement de la procédure d'import des modules externes. Voir la [PEP 489](https://www.python.org/dev/peps/pep-0489/).
 - L'implémentation en C, plutôt qu'en Python, des `OrderedDict` apportant des gains de 4 à 100 sur les performances. Voir [le ticket 16991](https://bugs.python.org/issue16991).
 - Une nouvelle fonction `os.scandir()` pour itérer plus efficacement sur le contenu d'un dossier sur le disque qu'en utilisant `os.walk()`. Cette dernière fonction l'utilise maintenant en interne et est de 3 à 5 fois plus rapide sur les système POSIX et de 7 à 20 fois plus rapide sur Windows. Voir la [PEP 471](http://www.python.org/dev/peps/pep-0471).
 - L'interpréteur va maintenant automatiquement retenter le dernier appel système lorsqu'un signal `EINTR` est reçu, évitant aux codes utilisateurs de s'en occuper. Voir la [PEP 475](http://www.python.org/dev/peps/pep-0475).
 - TODO: https://docs.python.org/3.5/whatsnew/3.5.html#pep-479-change-stopiteration-handling-inside-generators
 - TODO: https://docs.python.org/3.5/whatsnew/3.5.html#pep-486-make-the-python-launcher-aware-of-virtual-environments
 - TODO: https://docs.python.org/3.5/whatsnew/3.5.html#pep-485-a-function-for-testing-approximate-equality
De plus, comme d’habitude, tout un tas de fonctions, de petites modifications et de corrections de bugs ont été apportés à la bibliothèque standard qu'il serait trop long de citer entièrement. 

Notons enfin que le support de Windows XP est supprimé (cela ne veut pas dire que Python 3.5 ne fonctionnera pas dessus, mais rien ne sera fait par les développeurs pour cela). 

<--COMMENT  Si on a le temps :

# Support de Python 3.5


Support dans les IDE

Support des autres implémentations.

COMMENT-->


<--COMMENT   Si ce n'est pas déjà trop long :

# Point sur l'adoption de Python 3


COMMENT-->


# Ce que l'on peut attendre pour la version 3.6

<--COMMENT
Accepté pour la 3.5 mais non implémentés

    PEP 431 , improved support for time zone databases
    PEP 432 , simplifying Python's startup sequence
    PEP 436 , a build tool generating boilerplate for extension modules
    PEP 447 , support for __locallookup__ metaclass method
    PEP 468 , preserving the order of **kwargs in a function

ideas:

http://code.activestate.com/lists/python-ideas/34051/
+ fstrings


+ probablement la continuité du support de la programmation asynchrone et l'extension des coroutines

+ suivre https://www.python.org/dev/peps/pep-0494/ pour les pep acceptés et dates quand connues

COMMENT-->
