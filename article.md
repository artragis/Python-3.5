Une nouvelle version du langage [Python](https://www.python.org/) (et par conséquent de son implémentation principale CPython) sort aujourd'hui. Estampillée 3.5, cette version fait logiquement suite à [la version 3.4](https://docs.python.org/3/whatsnew/3.4.html) parue il y a un an et demi[^ndbp_date_34]. Tandis que cette dernière apportait principalement des ajouts dans la bibliothèque standard ([asyncio](https://docs.python.org/3/library/asyncio.html), [enum](https://docs.python.org/3/library/enum.html), [ensurepip](https://docs.python.org/3/library/ensurepip.html), [pathlib](https://docs.python.org/3/library/pathlib.html), etc.), les nouveautés les plus visibles de la version 3.5 concernent des changements syntaxiques avec deux nouveaux mot-clés, un nouvel opérateur binaire, la généralisation de l'*unpacking* et la standardisation des annotations de fonctions.

[^ndbp_date_34]: Le 16 Mars 2014 pour être précis. 

# TL;DR - Résumé des principales nouveautés

Les plus pressés peuvent profiter de ce court résumé [des principales nouveautés](https://docs.python.org/3.5/whatsnew/3.5.html) :

 - [PEP 492](https://www.python.org/dev/peps/pep-0492) : les coroutines deviennent une construction spécifique du langage, avec l'introduction des mots-clés `async` et `await`. L'objectif étant de compléter le support de la programmation asynchrone dans Python, ils permettent d'écrire des coroutines utilisables avec `asyncio` de façon similaire à des fonctions Python synchrones classiques. Par exemple :
   
   ```python
   async def fetch_page(url, filename):
       # L'appel à aiohttp.request est asynchrone
       response = await aiohttp.request('GET', url)
       # Ici, on a récupéré le résultat de la requête
       assert response.status == 200
     
       async with aiofiles.open(filename, mode='wb') as fd:
          async for chunk in response.content.read_chunk(chunk_size):
              await fd.write(chunk)
   ```
   
 - [PEP 465](http://www.python.org/dev/peps/pep-0465) : l'opérateur binaire `@` est introduit pour gérer la multiplication matricielle et permet d'améliorer la lisibilité d'expressions mathématiques. Par exemple, l'équation $A \times B^T - C^{-1}$ correspondrait au code suivant avec `numpy`. 
    
    ```python
    >>> A @ B.T - inv(C)
    ```

 - [PEP 484](https://www.python.org/dev/peps/pep-0484/) : les annotations apposables sur les paramètres et la valeur de retour des fonctions et méthodes sont maintenant standardisées et ne devraient servir qu'à préciser le type de ces éléments. Les annotations ne sont toujours pas utilisées par l'interpréteur et cette PEP n'est constituée que de conventions.
 
    ```python
    def bonjour(nom: str) -> str:
        return 'Zestueusement ' + nom
    ```

 - [PEP 448](https://www.python.org/dev/peps/pep-0448/) : les opérations d'*unpacking* sont généralisées et peuvent maintenant être combinées et utilisées plusieurs fois dans un appel de fonction.

    ```python
    >>> def spam(a, b, c, d, e, g, h, i, j):
    ...    return a + b + c + d + e + f + g + h + i 
    ...
    >>> l1 = (1, *[2])
    >>> l1
    (1, 2)
    >>> d1 = {"j": 9, **{"i": 8}}
    >>> d1 
    {"j": 9, "i": 8}
    >>> spam(*l1, 3, *(4, 5) g=6, **d1, **{'h':7})
    45    # 1 + 2 + 3 + 4 + 5 + 6 + 7 + 8 + 9
    ```

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

### Support de l'*unpacking* dans les déclarations d'itérables

Lorsque vous souhaitez définir un `tuple`, `list`, `set` ou `dict` litéral, il 
est maintenant possible d'utiliser l'*unpacking*.

```python
# Python 3.4
>>> tuple(range(4)) + (4,)
(0, 1, 2, 3, 4)
# Python 3.5
>>> (*range(4), 4)
(0, 1, 2, 3, 4)

# Python 3.4
>>> list(range(4)) + [4]
[0, 1, 2, 3, 4]
# Python 3.5
>>> [*range(4), 4]
[0, 1, 2, 3, 4]

# Python 3.4
>>> set(range(4)) + {4}
set(0, 1, 2, 3, 4)
# Python 3.5
>>> {*range(4), 4}
set(0, 1, 2, 3, 4)

# Python 3.4
>>> d = {'x': 1}
>>> d.update({'y': 2})
>>> d
{'x': 1, 'y': 2}
# Python 3.5
>>> {'x': 1, **{'y': 2}}
{'x': 1, 'y': 2}

# Python 3.4
>>> combinaison = long_dict.copy()
>>> combination.update({'b': 2})
# Python 3.5
>>> combinaison = {**long_dict, 'b': 2}
```

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

### Support de plusieurs *unpacking* dans les appels de fonctions

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

Notons enfin que le support de Windows XP est supprimé (cela ne veut pas dire que Python 3.5 ne fonctionnera pas dessus, mais rien ne sera fait par les développeurs pour cela). 


# Ce que l'on peut attendre pour la version 3.6

Il est impossible de savoir précisément ce qui sera disponible dans la prochaine version du langage et de l'interpréteur. 
En effet aucune entreprise ou groupe ne décide par avance des fonctionnalités. Les changements inclus dépendent des
propositions faites, majoritairement, sur deux *mailling list* :

 - [*python-dev*](https://mail.python.org/mailman/listinfo/python-dev) quand cela concerne des changements interne à l'intepréteur CPython.
 - [*python-ideas*](https://mail.python.org/mailman/listinfo/python-ideas) quand cela concerne des idées générale concernant le langage.

S'en suivient une série de débat qui, si les idées sont acceptés par une majorité de developpeurs principaux, conduit
à la rédaction d'une [PEP](https://www.python.org/dev/peps/) qui sera accepté, ou refusé, par le BDFL (Guido ou un autre
developpeur si Guido est l'auteur de la PEP). En l'absence de feuilles de routes définit à l'avance, les PEP peuvent 
arriver tard dans le cycle de développement. Ainsi la PEP 492 sur `async` et `await` 
n'a été créé qu'un mois avant la sortie de la première beta de pytohn 3.5.

[[a]]
| Malgré ces incertitudes, on peut deviner quelques modifications probables. Attention cette section reste très spéculative...

## Continuité des changements introduits dans Python 3.5

Trois thématiques de modifications, amorcés dans Python 3.4 et surtout Pytohn 3.5 pourraient être de nouveau source d'ajouts
important dans Python 3.6 :

 - La programmation asynchrone avec *asyncio* et les coroutines
 - Les indications de types
 - La généralisation de l'*unpacking*
 
Les deux premiers éléments sont officiellement en attente de retours de la part des utilisateur de Python les utilisant. 
Les développeurs de l'interpréteurs attendent de connaitre les problèmes et limitations rencontré en utilisant ces 
fonctionnalités pour les peaufiner dans les prochaines versions. Python 3.6 devrait donc logiquement voir des améliorations 
dans ces deux sections en profitant des retours. Déjà quelques remarques ont été faite comme 
[la complexité de méler des fonctions synchrones et asynchrone](http://code.activestate.com/lists/python-ideas/34419/).

La généralisation de l'*unpacking* pourrait elle aussi continuer. Dans un premier temps la 
[PEP 448](https://www.python.org/dev/peps/pep-0448/#variations) proposait initialement plus de généralisation qui n'ont 
pas été retenu pour Pytohn 3.5 par manque de temps pour trouver un concensus ou étudier les problèmes possés par ces 
modifications de syntaxe. Elles seront donc propablement rapidement re-discuté. De plus, ces ajouts dans Python 3.5 a 
donné des idées à d'autres développeurs et quelques [nouvelles modifications](http://code.activestate.com/lists/python-ideas/35074/)
ont été déjà proposés.


## Conservation de l'ordre des arguments fournies à `**kwargs` lors de l'appel aux fonctions

Cette [PEP](http://www.python.org/dev/peps/pep-0468) a déjà été accepté mais n'a pas put être implémenté dans la version 3.5. 
Il est donc possible qu'elle soit finalement présente dans la prochaine version. L'idée est que les fonctions puissent connaitre
l'ordre dans lequel les arguments nommés ont été passé. Prenons un exemple :

```python
def spam(**kwargs):
    for k, v in kwargs.items():
        print(k, v)

spam(a=1, b=2)
```

Avec Python 3.5, comme toutes versions de CPython et presques toutes les autres implémentations (sauf PyPy), nous ne pouvons
pas savoir si le résultat sera :

```
a 1
b 2
```

ou 


```
b 2
a 1
```

En effet un dictionnaire est utilisé pour passer les arguments. Or ceux-ci ne garantissent l'ordre d'itération. Pourtant
Python dispose d'un [`OrderedDict`](https://docs.python.org/3/library/collections.html#collections.OrderedDict) depuis 
Python 3.1. L'implémentation de cette PEP se résumerait donc de remplacer l'objet utilisé en interne d'un dictionnaire à
un dictionnaire ordonné. Cependant jusqu'à maintenant cet objet était définit en Python. L'implémentation était donc loin
d'être la plus efficace possible et ne pouvait pas être utilisé pour un élément aussi critique du langage. C'est pour cette
raison que l'implémentation a été refaite en C pour python 3.5 comme noté dans la section "De plus petits changements". 
Maintenant plus rien ne semble bloqué l'implémentation de cette PEP qui devrait donc voir le jour dans Python 3.6.

Le principal intérêt de cette modification est que le fonctionnement actuel est contre intuitif pour bon nombre d'utilisateurs
connaissant mal le fonctionnement interne de Python. D'autres raisons sont invoqués dans la PEP :

 - De manière évidente les `OrederedDict` pourraient maintenant être créées en utilisant des arguments nommés, comme les
   dictionnaires.
 - La sérialization : dans certains formats l'ordre d'apparition des données à de l'importance (ex: l'ordre des colonnes
   dans un fichier CSV). Cette nouvelle possibilité permettrait de les définir plus facilement en même temps que des 
   valeurs par defaults. Elle permettrait aussi à des formats comme XML, Json ou Yaml de garantir l'ordre d'apparition
   des attributs ou clés qui sont enregistrés dans les fichiers.
 - Le debuggage : le fonctionnement actuel pouvant être aléatoire, il peut être compliqué de reproduire certains bugs. 
   Si un bug apparait selon l'ordre de définition des arguments, le nouveau comportement facilitera leur correction.
 - La priorité à donner aux arguments pourrait être spécifier selon l'ordre de leur déclaration.
 - Les `namedtuple` pourraient être définit avec une valeur par défaut simplement.

et probablement d'autres utilisations qui n'ont pas encore été envisagées.

## Proprités de classes

Toujours dans les petites modifications, un nouveau décorateur disponible de base devrait être introduit : `@classproperty`.
Si vous connaissez le modèle objet de Python, son nom devrait vous suffir pour deviner son but : permettre de définir des
propriétés au niveau de classes. Par exemple si vous souhaitez conserver au niveau de la class le nombre d'instances 
créées et des statistiques sur vos appels:

```python
class Spam:
    _n_instance_created = 0
    _n_egg_call = 0
    
    def __init__(self):
        self.__class__._n_instance_created += 1
    
    def egg(self):
        print("egg")
        self.__class__._n_egg_call += 1
    
    @classproperty
    def mean_egg_call(cls):
        return (cls._n_egg_call / cls._n_instance_created) if cls._n_instance_created > 0 else 0

spam = Spam()

spam.egg()
spam.egg()
spam.egg()

print(spam.mean_egg_call)   # => 3

spam2 = Spam()

# Les 3 expressions suivantes sont équivalentes
print(spam.mean_egg_call)    # => 1.5
print(spam2.mean_egg_call)   # => 1.5
print(Spam.mean_egg_call)    # => 1.5  , propriété au niveau de la class !
```

Cet ajout a été proposé sur la *mailling-list python-ideas*. La mise en place d'une implémentation propre et complète de 
ce comportement est compliqué sans définir une méta-classe. Une solution générique implique donc de le rajouter dans le 
modèle objet de Python. Guido a rapidement donné son aprobation et un [ticket a été créé](http://bugs.python.org/issue24941)
en attendant qu'un des developpeurs de CPython l'implémente dans le coeur de l'intrépreteur. Cela pourrait donc être
l'une des première nouvelle fonctionnalité confirmé pour la version 3.6.

## Sous-interpréteurs

[[a]]
| Cette section peut nécessiter des notions avancés de programmation système pour être comprise, en particulier la différence entre un *thread* et un *process* ainsi que leur gestion dans Python.

L'écriture de codes concurent en Python est toujours un sujet chaud. A ce jour trois solutions existent dans la 
bibliotèque standard :

 - *asyncio* bien que limiter à un seul *thread*, il permet d'écrire des coroutine s'excutant en concurence en tirant 
 partie des temps d'attentes introduits par les entrées/sorties.
 - *threading* qui permet de facilement executer du code sur plusieurs *thread* mais qui restent limiter à un seul 
 coeur de processeur à cause du GIL[^ndbp_gil].
 - *multiprocessing* en *forkant* l'interpréteur permet d'éxecuter plusieurs codes Python en parralèle sans limitation
 et exploitant pleinnement les ressources calculatoir des processeurs.

[[a]]
| Le [GIL](https://en.wikipedia.org/wiki/Global_Interpreter_Lock) est une construction implémenté dans de nompreux interpéteurs (CPython, Pypy, Ruby, etc.). Ce mécanisme bloque l'interpréteur pour qu'a chaque instant un seul bytecode puisse être executé. Ce sytème permet de s'assurer que du code éxécuté sur plusieurs *threads* ne va pas poser de problèmes de concurence sur la mémoire, sans vraiment ralentir les codes n'utilisant qu'un seul *thread*. Malheureusement cela empeche d'exploiter les architectures multi-coeurs de nos processeurs.

La multiplicité des solutions ne résouds pas tout. En effet les limitation des *threads* en python les rendent inutile
quand le traitement ce fait principalement sur le processeur. L'utilisation de *multiprocessing* est alors possible mais 
a un cout :

 - Le lancement d'un process entraine un *fork* au niveau du système d'exploitation, ce qui prend plus de temps et de 
 mémoire que lancement d'un *thread* qui est lui presque gratuit à ce niveau.
 - La communication entre les processus est aussi plus longue. Tandis que les *thread* permetent de partager la mémoire,
 rendant les communcations direct, les processus necessitent de mettre en places des mecanismes complexes, appelés 
 [*IPC* pour "communications inter-processus"](https://fr.wikipedia.org/wiki/Communication_inter-processus).

[Une proposition sur *python-ideas*](http://code.activestate.com/lists/python-ideas/34051/) a été formulé au début de 
l'été pour offrir une nouvelle solution intermédiare entre les *threads* et les *process*, des sous-interpréteurs 
(*subinterpreters*), en espérant que cela puisse définitvement contenter les développeurs écrivants des codes concurents.

A partir d'un nouveau module *subinterpreters*, reprendant l'interface exposé par les modules *threading* et *multiprocessing*,
CPython permettrait de lancer dans des *threads* différents interpréteurs, chacun chargé d'executer un code Python. Le fait
que ce soit des *threads* qui soient utilisé rendrait ce module beaucoup plus leger à utiliser que *multiprocessing*. 
L'interpréteur se chargerait de lancer ces *threads* et partagerait le maximum de l'implémentation de l'interpréteur.
Le plus interessant est que ce mécanisme ferait "disparaitre" le *GIL*. En effet chaque sous-interpréteur disposeraient
d'un espace de nom de propre et indépendant. Chacun aurait donc un *GIL* mais ceux-ci seraient indépendants. D'un point 
de l'ensemble, la majorité des opérations transformeraient le *GIL* en *LIL* : *Local interpreter lock*. Chaque 
sous-interpréteur pourrait ainsi fonctionner sur un coeur du processus séparré sans aucun problèmes, permetant de tirer
pleinnement partie des processeurs modernes. Enfin il faut savoir que CPython possède déjà en interne la notion de 
sous-interpréteurs en interne, le travail a réaliser consciterai donc à exposer ce code C pour le rendre disponible 
en Python.

Evidement cette proposition n'est pas une solution miracle. Tout n'est pas si simple et beaucoup d'élements doivent encore
être discutés. En particulier les moyens de communications entre les sous-interpréteurs (probablemens via des queues comme
pour *multiprocessing*) et tout un tas de petits détails d'implémentations. Mais la proposition a reçu un acceuil très positif
et l'auteur est actuellement en train de préparer une PEP à ce sujet.

[^ndbp_gil]: *Global Interpreter Lock*

## Interpolation de chaines

Les interpolations directs de chaines de caractèrent existent dans de nombreux langages : PHP, C#, Ruby, Swift, Perl, etc.
Un exemple en Perl est par exemple :

```perl
my $a = 1;
my $b = 2;

print "Resultat = a + b = $a + $b = @{[$a+$b]}\n";
# Imprime "Resultat = a + b = 1 + 2 = 3"
```

Il n'existe pas d'équivalents directs en Python, le plus proche pourrait ressembler à l'exemple suivant :

```python
a, b = 1, 2

print("Resultat = a + b = %d + %d = %d" % (a, b, a + b))
# ou
print("Resultat = a + b = {a} + {b} = {c}".format(a=a, b=b, c=a + b))
```

Nous voyons ainsi qu'il est necessaire de passer explicitement les variables et, même si il est possible d'utiliser 
`locals()` ou `globals()` pour s'en passer, il reste impossible d'évaluer une expression, comme ici l'addition, ailleurs 
qu'à l'exterieur de la chaine de caractère. 

La première proposition effectué, formalisé par le [PEP 498](https://www.python.org/dev/peps/pep-0498/) propose de rajouter
cette possibilité dans python gràce à un nouveau préfixe de chaine, `f` pour `format-string`, dont voici un exemple 
d'utilisation issue de la PEP :

```python
>>> import datetime
>>> name = 'Fred'
>>> age = 50
>>> anniversary = datetime.date(1991, 10, 12)
>>> f'My name is {name}, my age next year is {age+1}, my anniversary is {anniversary:%A, %B %d, %Y}.'
'My name is Fred, my age next year is 51, my anniversary is Saturday, October 12, 1991.'
>>> f'He said his name is {name!r}.'
"He said his name is 'Fred'."
```

Cette proposition a fait des émules. Plusieurs développeurs ont mit en évidence que la méthodes d'interpolation peut 
dépendre du contexte. Par exemple un module de base de donnée comme [`sqlite3`](https://docs.python.org/3.4/library/sqlite3.html)
propose un système de substitution personnalisé pour éviter les attaques par injection ou encore les moteurs de *templates* pour 
produire du HTML vont aussi faire des substitutions pour échapper certains caractères comme remplacer `<` ou `>` par, 
respectivement, `&lt;` ou `&gt;`. La [PEP 501](https://www.python.org/dev/peps/pep-0501/) propose aux developpeurs de définir
leurs propre méthodes d'interpolations adapté au contexte.

Guido c'est clairement prononcé pour la première proposition afin qu'elle soit implémenté dans Python 3.6. La deuxième est
plus générale mais plus complexe et peu ainsi poser plusieurs problèmes. Aucune décision n'a encore été prise et les débats
à ce sujet on encore lieu sur la *mailling list* concernant cette possibilité et la forme qu'elle pourrait prendre. Il est
donc très probable qu'au moins la première proposition soit implémenté. La deuxième dépendra de la suite des discussions
mais, à défaut de consensus, les *f-string* simples seront implémentées dans Python 3.6.

-------

TODO: Conclusion général
