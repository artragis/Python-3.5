Une nouvelle version du langage [Python](https://www.python.org/) (et par conséquent de son implémentation principale CPython) sort aujourd'hui. Estampillée 3.5, cette version fait logiquement suite à [la version 3.4](https://docs.python.org/3/whatsnew/3.4.html) parue il y a un an et demi[^ndbp_date_34]. Tandis que cette dernière apportait principalement des ajouts dans la bibliothèque standard ([asyncio](https://docs.python.org/3/library/asyncio.html), [enum](https://docs.python.org/3/library/enum.html), [ensurepip](https://docs.python.org/3/library/ensurepip.html), [pathlib](https://docs.python.org/3/library/pathlib.html), etc.), les nouveautés les plus visibles de la version 3.5 concernent des changements syntaxiques avec deux nouveaux mot-clés, un nouvel opérateur binaire, la généralisation de l'*unpacking* et la standardisation des annotations de fonctions.

[^ndbp_date_34]: Le 16 Mars 2014 pour être précis. 

# TL;DR - Résumé des principales nouveautés

Les plus pressés peuvent profiter de ce court résumé [des principales nouveautés](https://docs.python.org/3.5/whatsnew/3.5.html) :

 - [PEP 492](https://www.python.org/dev/peps/pep-0492) : les coroutines deviennent une construction spécifique du langage. Cette gestion dans l'interpréteur se fait via deux nouveaux mots-clés (`async` et `await`) et vise à compléter le support de la « programmation asynchrone » dans Python.
 - [PEP 465](http://www.python.org/dev/peps/pep-0465) : l'opérateur binaire `@` est introduit pour gérer la multiplication matricielle.
 - [PEP 484](https://www.python.org/dev/peps/pep-0484/) : les annotations apposables sur les paramètres et la valeur de retour des fonctions et méthodes sont maintenant standardisées et servent uniquement à préciser le type de ces éléments. Les annotations ne sont toujours pas utilisées par l'interpréteur et cette PEP n'est constituée que de conventions.
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

L'introduction d'un opérateur binaire n'est pas courant dans Python. 
Aucun dans la série 3.x, le dernier ajout semble être 
[l'opérateur `//` dans Python 2.2](https://www.python.org/dev/peps/pep-0238/)[^ndbp_op_bin]. 
Regardons donc pour quelles raisons celui-ci a été introduit.


[^ndbp_op_bin]: Ces ajouts sont tellement peu courants qu'il est difficile de 
trouver des traces de ces modifications. L'opérateur `//` semble être le seul 
ajout de toute la série 2.

### Signification

Ce nouvel opérateur est dédié à la multiplication matricielle. En effet, 
comme tous ceux qui ont fait un peu de mathématiques algébriques doivent le 
savoir, pour une matrice on définit généralement la multiplication par 
l'opération suivante (pour des matrices $2*2$) :

$$
\begin{pmatrix}a&b \\ c&d \end{pmatrix} \cdot \begin{pmatrix}e&f \\ g&h \end{pmatrix} = 
\begin{pmatrix}a*e+b*g&a*f+b*h \\ c*e+d*g&c*f+d*h \end{pmatrix}
$$

Or il est souvent aussi nécessaire d'effectuer des multiplications 
terme à terme :

$$
\begin{pmatrix}a&b \\ c&d \end{pmatrix} * \begin{pmatrix}e&f \\ g&h \end{pmatrix} = 
\begin{pmatrix}a*e&b*f \\ c*e&d*h \end{pmatrix}
$$

Tandis que certains langages spécialisés possèdent des opérateurs dédiés pour 
chacune de ces opérations[^ndbp_op_matmatlab] il n'y a en Python rien de 
similaire. Avec la bibliothèque [numpy](http://www.numpy.org), la plus 
populaire pour le calcul numérique dans l'éco-système Python, il est possible 
d'utiliser l'opérateur natif `*` pour effectuer une multiplication terme à 
terme, surcharge d'opérateur se rencontrant dans la plupart des bibliothèques 
faisant intervenir les matrices. Mais il n'existe ainsi plus de moyen simple 
pour effectuer une multiplication matricielle (l'opérateur `*` étant déjà pris). 
Avec `numpy`, il est pour le moment nécessaire d'utiliser la méthode `dot` et 
d'écrire des lignes de la forme :

```python
S = (H.dot(beta) - r).T.dot(inv(H.dot(V).dot(H.T))).dot(H.dot(beta) - r)
```

Celle-ci traduit cette formule :

$$
(H \times \beta - r)^T \times (H \times V \times H^T)^{-1} \times (H \times \beta - r)
$$

Cela n'aide pas à la lecture... Le nouvel opérateur `@` est introduit et 
dédié à la multiplication matricielle. Il permettra d'obtenir des expressions 
équivalentes de la forme :

```python
S = (H @ beta - r).T @ inv(H @ V @ H.T) @ (H @ beta - r)
```

Un peu mieux, non ?

[^ndbp_op_matmatlab]: Par exemple, avec Matlab et Julia, `*` / `.*` servent respectivement pour la multiplication matricielle et terme à terme.

### Impact sur vos codes

Cette introduction devrait être anodine pour beaucoup d'utilisateurs. En effet, 
aucun objet de la bibliothèque standard ne va l'utiliser[^ndbp_nused]. Cet 
opérateur binaire servira principalement pour des bibliothèques annexes, à 
commencer par *numpy*. 

[^ndbp_nused]: Ce qui est rare mais existe déjà. En particulier l'objet `Elipsis` créé avec `...`.

Si vous souhaitez supporter cet opérateur, trois méthodes spéciales peuvent 
être implémentées : `__matmul__` et `__rmatmul__` pour la forme `a @ b` et 
`__imatmul__` pour la forme `a @= b`, de façon similaire aux autres opérateurs.

À noter qu'il est déconseillé d'utiliser cet opérateur pour autre chose que 
les multiplications matricielles.

### Motivation

L'introduction de cet opérateur est un modèle du genre : il peut sembler très 
spécifique mais est pleinement justifié. La lecture de la PEP, développée, est 
très instructive. Pour la faire adopter, les principales bibliothèques 
scientifiques en Python ont préparé cette PEP ensemble pour arriver à une 
solution convenant à la grande partie de la communauté scientifique, 
assurant dès lors l'adoption rapide de cet opérateur. La PEP précise ainsi 
l’intérêt et les problèmes engendrés par les autres solutions utilisées 
jusque-là. Enfin cette PEP a été fortement appuyée par 
[la grande popularité de Python dans le monde scientifique](https://www.python.org/dev/peps/pep-0465/#but-isn-t-matrix-multiplication-a-pretty-niche-requirement). Ainsi on apprend que `numpy` est le module n’appartenant pas 
à la bibliothèque standard le plus utilisé parmi tous les codes Python présents 
sur Github et ce sans compter d'autres bibliothèques comme `pylab` ou `scipy` 
qui vont aussi profiter de cette modification et sont présentes parmis la liste 
des bibliothèques les plus communes.

## Annotations de types -- PEP 484

Les annotations de fonctions [existent depuis Python 3](https://www.python.org/dev/peps/pep-3107/) 
et permettent d'attacher des objets Python quelconques aux arguments  
et à la valeur de retour d'une fonction ou d'un méthode :

```python
def ma_fonction(param: str, param2: 42) -> ["Mon", "annotation"]:
    pass
```

Depuis le début, il est possible d'utiliser n'importe quel objet valide comme 
annotation. Ce qui peut surprendre, c'est que l'interpréteur n'en fait pas 
d'utilisation particulière : il se contente de les stocker dans l'attribut 
`__annotations__` de la fonction correspondante :

```python
>>> ma_fonction.__annotations__
{'param2': 42, 'return': ['Mon', 'annotation'], 'param': <class 'str'>}
```

La PEP 484 introduite avec Python 3.5 fixe des conventions pour ces annotations : 
elles deviennent réservées à « l'allusion de types » (*Type Hinting*), qui 
consiste à indiquer le type des arguments et de la valeur de retour des 
fonctions :

```python
# Cette fonction prend en argument une chaîne de caractères et en retourne une autre.
def bonjour(nom: str) -> str:
    return 'Zestueusement ' + nom
```

Toutefois, il ne s'agit encore que de conventions : l'interpréteur Python ne 
fera rien d'autre que de les stocker dans l'attribut `__annotations__`. Il ne 
sera même pas gêné par une annotation d'une autre forme qu'un type.

Pour des indications plus complètes sur les types, un module `typing` est 
introduit. Il permet de définir des types génériques, des tableaux, etc. Il serait 
trop long de détailler ici toutes les possibilités de ce module, donc nous nous 
contentons de l'exemple suivant.

```python
# On importe depuis le module typing :
# - TypeVar pour définir un ensemble de types possibles
# - Iterable pour décrire un élément itérable
# - Tuple pour décrire un tuple
from typing import TypeVar, Iterable, Tuple

# Nous définissons ici un type générique pouvant être un des types de nombres cités.
T = TypeVar('T', int, float, complex)
# Vecteur représente un itérateur (list, tuple, etc.) contenant des tuples comportant chacun deux éléments de type T.
Vecteur = Iterable[Tuple[T, T]]

# Pour déclarer un itérable de tuples, sans spécifier le contenu de ces derniers, nous pouvons utiliser le type natif :
# Vecteur2 = Iterable[tuple]
# Les éléments présents dans le module typing sont là pour permettre une description plus complète des types de base.

# Nous définissons une fonction prenant un vecteur en argument et renvoyant un nombre
def inproduct(v: Vecteur) -> T:
    return sum(x*y for x, y in v)

vec = [(1, 2), (3, 4), (5, 6)]
res = inproduct(vec)
# res == 1 * 2 + 3 * 4 + 5 * 6 == 44
```

Néanmoins, les annotations peuvent surcharger les déclarations et les rendre 
peu lisibles. Cette PEP a donc introduit une convention supplémentaire : les 
fichiers `Stub`. Il a été convenu que de tels fichiers, facultifs, contiendraient 
les annotations de types, permettant ainsi de profiter de ces informations sans 
polluer le code source. Ces fichiers permettent aussi d'annoter les fonctions 
de bibliothèques écrites en C ou de fournir les annotations de types pour des 
modules utilisant les annotations pour une autre fonctionnalité. Par exemple, 
le module `datetime` de la bliotèque standard comporterait un fichier `Stub` de
la forme suivante.


```python
class date(object):
    def __init__(self, year: int, month: int, day: int): ...
    @classmethod
    def fromtimestamp(cls, timestamp: int or float) -> date: ...
    @classmethod
    def fromordinal(cls, ordinal: int) -> date: ...
    @classmethod
    def today(self) -> date: ...
    def ctime(self) -> str: ...
    def weekday(self) -> int: ...
```

Tout comme rien ne vous oblige à utiliser les annotations pour le typage, rien 
ne vous force à vous servir du module `typing` ou des fichiers `Stub` : 
l'interpréteur n'utilisera ni ne vérifiera toujours pas ces annotations ; le 
contenu des fichiers `Stub` ne sera même pas intégré à l'attribut `__annotations__`. Ne 
vous attendez donc pas à une augmentation de performances en utilisant les 
annotations de types : l'objectif de cette PEP est de normaliser ces informations 
pour permettre à des outils externes de les utiliser. Les premiers bénéficiaires 
seront donc les EDI, les générateurs de documentation (comme [Sphinx](http://sphinx-doc.org/)) 
ou encore les analyseurs de code statiques comme [mypy](http://mypy-lang.org/), 
lequel a fortement inspiré cette PEP et sera probablement le premier outil à 
proposer un support complet de cette fonctionnalité.

Les annotations de types sont donc qu'un ensemble de conventions, comme il en 
existe déjà plusieurs dans le monde Python ([PEP 333](https://www.python.org/dev/peps/pep-0333/), 
[PEP 8](https://www.python.org/dev/peps/pep-0008/), [PEP 257](https://www.python.org/dev/peps/pep-0257/), 
etc.). Cet ajout n'a donc aucun impact direct sur vos codes mais permettra aux 
outils externes de vous fournir du support supplémentaire.

## *Unpacking* généralisé -- PEP 448

# De plus petits changements

 - [PEP 441](http://www.python.org/dev/peps/pep-0441) : Ajout d'un module `zipapp` améliorant le support et la création d'applications packagés sous forme d'archive `zip` introduit dans Python 2.6.
 - [PEP 461](https://www.python.org/dev/peps/pep-0461/) : Retour de l'opérateur de formatage `%` pour les types de données `byte` et `bytearray` de la même façon qu'il était utilisable pour les types chaînes (`str`) dans Python 2.7.
 - [PEP 488](https://www.python.org/dev/peps/pep-0488/) : La suppression des fichiers `pyo`.
 - [PEP 489](https://www.python.org/dev/peps/pep-0489/) : Un changement de la procédure d'import des modules externes.
 - [PEP 471](http://www.python.org/dev/peps/pep-0471) : Une nouvelle fonction `os.scandir()` pour itérer plus efficacement sur le contenu d'un dossier sur le disque qu'en utilisant `os.walk()`. Cette dernière fonction l'utilise maintenant en interne et est de 3 à 5 fois plus rapide sur les système POSIX et de 7 à 20 fois plus rapide sur Windows.
 - [PEP 475](http://www.python.org/dev/peps/pep-0475) : L'interpréteur va maintenant automatiquement retenter le dernier appel système lorsqu'un signal `EINTR` est reçu, évitant aux codes utilisateurs de s'en occuper.
 - [PEP 479](https://www.python.org/dev/peps/pep-0479/) : Le comportement des générateurs changent lorsqu'une exception `StopIteration` est déclenché à l'intérieur. Avec cette PEP, cette exception est transformé en `RuntimeError` plutôt que de quitter silencieusement le générateur. Cette modification cherche à faciliter le deboguage et clarifie la façon de quitter un générateur : utiliser `return` plutôt que de générer une exception. Cette PEP n'étant pas rétro-compatible, vous devez manuellement l'activer avec `from __future__ import generator_stop` pour en profiter.
 - [PEP 485](https://www.python.org/dev/peps/pep-0485/) : Une fonction `isclose` est rajouté pour tester la "proximité" de deux nombres flottants.
 - [PEP 486](https://www.python.org/dev/peps/pep-0486/) : Le support de `virtualenv` dans l'installateur de Python sous Windows est amélioré et simplifié. 
 - [Ticket 9951](https://bugs.python.org/issue9951) : Pour les types `byte` et `bytearray`, une nouvelle méthode `hex()` permet maintenant de récupérer une chaîne représentant le contenu sous forme hexadécimale.
 - [Ticket 16991](https://bugs.python.org/issue16991) : L'implémentation en C, plutôt qu'en Python, des `OrderedDict` (dictionnaires ordonnées) apportant des gains de 4 à 100 sur les performances.

De plus, comme d’habitude, tout un tas de fonctions, de petites modifications et de corrections de bugs ont été apportés à la bibliothèque standard qu'il serait trop long de citer entièrement. 

Notons aussi que le support de Windows XP est supprimé (cela ne veut pas dire que Python 3.5 ne fonctionnera pas dessus, mais rien ne sera fait par les développeurs pour cela). 

Enfin, le développement de CPython 3.5 s'est effectué sur [BitBucket](https://bitbucket.org/larry/cpython350) plutôt que sur le traditionnel [dépôt auto-hébergé](https://hg.python.org/cpython) de la fondation, afin de bénéficier des fonctionnalités proposées par le site ; le gestionnaire de versions est toujours Mercurial. Il n'est pas encore connu si les prochaines versions suivront ce changement et si le développement de CPython se déplacera sur BitBucket : il s'agit d'une expérimentation que les développeurs vont évaluer. Pour le moment, les branches pour les versions 3.5.1 et 3.6 restent sur [https://hg.python.org/cpython](https://hg.python.org/cpython) au moins jusqu'à la sortie de Python 3.5.

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
