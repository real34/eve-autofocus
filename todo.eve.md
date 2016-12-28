# Autofocus
## Mon application
Ceci une application de TODO List pour remplacer le carnet qui me suit partout.
Cette application sera utilisée en plusieurs phases : planification ou action.

Au lancement l'application est en mode planification.

```eve
commit
  [#app phase: "planification"]
```

Un gabarit de page basique contient le nom de l'application et permet de se repérer dans les différentes phases :
```
search
  app = [#app phase]

bind @browser
  [#h1 text: "TODO"]
  [#p children:
    [#span type: "phase" text: "Phase actuelle : {{app.phase}} "]
    [#button #switch-phase text: "Changer de phase"]]
  [#div #content]
```

### Liste des tâches
Elle affiche une liste de toutes mes tâches en cours (en phase de planification),
et permet de les manipuler selon le système "Autofocus" (voir section dédiée).

```eve
search @session @browser
  app = [#app phase: "planification"]
	[#tache libelle ordre]
  tache = [#tache]
  content = [#content]

bind @browser
  content.children :=
    [#ul children:
      [#li #todo tache
        sort: tache.ordre
        text: "{{tache.libelle}} ({{tache.ordre}})"]]
```

Les tâches sélectionnées sont affichées en rouge, et les tâches terminées sont barrées et grisées :

```
search @browser @session
  élémentSélectionné = [#todo tache: [#tache #sélectionnée]]
	élémentTerminé = [#todo tache: [#tache #terminée]]

bind @browser
	élémentSélectionné.style := [color: "red"]
  élémentTerminé.style := [
  	color: "grey"
    text-decoration: "line-through"
    font-style: "italic"
    font-size: "0.9em"]
```

Si aucune tâche n'existe on affiche un message zen

```eve
search
  not([#tache])

bind @browser
  [#h1 text: "Rien n'est plus beau que le vide"]
  [#p text: "Bravo pour ta productivité. Il est temps de profiter des tiens"]
```

Enfin, un bouton permet de passer automatiquement d'une phase à l'autre :

```
search @event @session @browser
  [#click element: [#switch-phase]]
  app = [#app phase]
  new_phase = if app.phase = "planification" then "action" else "planification"

commit
  app.phase := new_phase
```
### Nouvelle tâche

Il est possible de saisir une nouvelle tâche à faire à tout moment.

```eve
bind @browser
  [#div sort: 1000 children:
    [#input #nouvelle-tache placeholder: "acheter du lait"]]
```

Lors de l'ajout, la tâche est ajoutée à la fin des tâches en cours et non sélectionnée

```eve
search @event @session @browser
  element = [#nouvelle-tache value]
  kd = [#keydown element, key: "enter"]

commit @session @browser
  [#tache libelle: value]
  element.value := ""
```

## Autofocus
Le système Autofocus consiste en des étapes différentes.

### Saisie des tâches l'une après l'autre
Lorsqu'une tâche est ajoutée au système, elle l'est toujours en fin de liste (car c'est la plus récente).

```
search
	tache = [#tache]
  not(tache = [ordre])

  sort[value: plusGrandOrdre direction: "down"] = 1
  tachesAnciennes = [#tache ordre: plusGrandOrdre]

commit
	tache.ordre := plusGrandOrdre + 1
```

### Sélection de la tâche la plus ancienne
La planification débute par la sélection de la tâche la plus ancienne

```eve
search
  [#app phase: "planification"]
  sort[value: ordre] = 1
  plusAncienneTâche = [#tache not(#terminée) ordre]

commit
	plusAncienneTâche += #sélectionnée
```

### Sélection des tâches suivantes que l'on veut faire avant
Le but dans cette étape est de parcourir la liste de tâches situées *après* la dernière
tâche sélectionnée ("tâche X")

```
search
	[#app phase: "planification"]
  sort[value: ordreSelectionnée direction: "down"] = 1
  [#tache #sélectionnée ordre: ordreSelectionnée]
  tâchesSélectionnables = [#tache ordre > ordreSelectionnée not(#terminée)]
  tâchesNonSélectionnables = [#tache ordre <= ordreSelectionnée]

commit
	tâchesSélectionnables += #sélectionnable
  tâchesNonSélectionnables -= #sélectionnable
```

et de se demander pour chacune d'entre elle : "Est-ce que je veux réaliser cette tâche Y avant la tâche X ?"

La réponse dépendra du contexte, de la volonté et de l'urgence par exemple. Si
on répond "Oui" à cette question, alors sélectionner cette tâche et recommencer
l'opération pour les tâches ultérieures à celles-ci.

```eve
search @browser @session @event
	tache = [#tache #sélectionnable]
  [#click element: [#todo tache]]

commit
  tache += #sélectionnée
```

Pour aider à la détection visuelle des tâches sélectionnables, l'application rendra plus transparentes les non sélectionnables et rendra cliquables les sélectionnables.

```
search @browser @session
  élémentSélectionnables = [#todo tache: [#tache #sélectionnable]]
  élémentNonSélectionnables = [#todo tache: [#tache not(#sélectionnable)]]

bind @browser
	élémentSélectionnables.style := [color: "green" cursor: "pointer"]
  élémentSélectionnables.title := "Sélectionner"
	élémentNonSélectionnables.style <- [opacity: 0.3]
```

### Passage en phase d'action

Une fois qu'aucune tâche ultérieure à la dernière sélectionnée ne souhaite être
réalisée (ou que la tâche la plus récente est sélectionnée), il est temps de repasser en mode "Action".

```
search
  sort[value: ordre direction: "down"] = 1
  tâcheLaPlusRécente = [#tache ordre]

bind @view
  [#value | value : tâcheLaPlusRécente.libelle]
```

```
search
  select = [#tache #sélectionnée]
bind @view
  [#value | value : select.libelle]
```

Au passage en mode action, le but est de se focaliser sur la tâche la plus récente
sélectionnée. L'interface présente donc principalement son contenu à l'utilisateur.

```eve
search @session @browser
  app = [#app phase: "action"]
  sort[value: ordre direction: "down"] = 1
	tacheFocalisée = [#tache #sélectionnée libelle ordre]
  content = [#content]

bind @browser
  content.children :=
    [#div #action children:
      [#h2 sort:10 text: "En cours : {{tacheFocalisée.libelle}}"]]
```

L'utilisateur décide alors de travailler sur cette tâche autant de temps qu'il le souhaite
(5 minutes comme 2 heures).

Afin de donner une vision de ce qui suit, un récapitulatif des tâches à venir est
affiché à l'utilisateur après l'action en cours :

```eve
search @session @browser
  aVenirIndex = sort[value: ordre direction: "down"]
  aVenirIndex > 1
	tacheAVenir = [#tache #sélectionnée libelle ordre]
  actionEnCours = [#div #action]

bind @browser
  actionEnCours.children +=
    [#div sort:100 children:
      [#h2 text: "À venir"]
      [#ul children:
        [#li #todo tacheAVenir
          sort: [value: tacheAVenir.ordre direction: "down"]
          text: "{{tacheAVenir.libelle}}"]]]
```

### Fin d'action sur une tâche

Lorsque l'utilisateur souhaite arrêter de travailler sur la tâche en cours
(qu'elle soit terminée ou non), il peut cliquer sur un bouton pour notifier l'application.

```
search @browser
  actionEnCours = [#div #action]
bind @browser
  actionEnCours.children += [sort: 20 #button #finirActionSurTâche text: "Finir de travailler sur cette tâche"]
```

La tâche en cours est alors marquée comme terminée et ajoutée à nouveau en fin de liste.

```
search @session @browser @event
  [#click element: [#finirActionSurTâche]]
  sort[value: ordre direction: "down"] = 1
	tacheFocalisée = [#tache #sélectionnée ordre]

commit @session
  tacheFocalisée -= #sélectionnée
  tacheFocalisée += #terminée
  [#tache libelle: tacheFocalisée.libelle]
```

Dans la cas où d'autres tâches (antérieures) avaient été sélectionnées en phase de planification,
la tâche sélectionnée la plus récente devient alors la tâche en cours et la session de travail peut reprendre.

```
// Rien à faire ici, car automatique d'après les autres blocs
```

Dans le cas où la tâche venant de se terminer était la dernière sélectionnée
(donc la tâche la plus ancienne), l'application repasse en phase "Planification".

```
search
  app = [#app phase: "action"]
  not([#tache #sélectionnée])
commit
  app.phase := "planification"
```

## Jeu de données
Afin d'avoir une interface un peu cohérente voici des exemples de données :

```
search
	app = [#app]
  not(app = [#init])

commit
	app += #init

  [#tache libelle: "Naître" #terminée ordre: -21]
  [#tache libelle: "Se doucher" ordre: -20]
	[#tache libelle: "Nettoyer" #terminée ordre: 0]
  [#tache libelle: "Ranger" #sélectionnée ordre: -2]
  [#tache libelle: "Dormir" ordre: 20]
  [#tache libelle: "Ronfler" ordre: 21]
  [#tache libelle: "Faire du café"]

```

# Bugs connus
## Le champ ne se vide pas
Pour reproduire : saisir une nouvelle tâche et taper entrée
Ce qu'il se passe : la tâche est ajoutée, mais le champ reste rempli avec la valeur saisie
Attendu : tâche ajoutée mais champ vidé

## Les tâches "sautées" en planification ne sont pas transparentes
Si l'on a sélectionné "A, B et E", la tâche "F" sera bien à sélectionner et en clair, en revanche "C et D" ne seront pas avec opacity

## La sélection de la dernière tâche ne passe pas automatiquement en phase "action"
Je ne suis pas parvenu à trouver le déclencheur qui va bien !

# À comprendre / Questions
## Comment faire une différence de collection ?
Exemple sur ce jeu de données :

```
commit
  [#foo label: "Foo 1" #bar]
  [#foo label: "Foo 2"]
  [#foo label: "Foo 3" #bar]
  [#foo label: "Foo 4" #bar]
  [#foo label: "Foo 5"]
```

Il est facile de récupérer tous les #foo #bar, mais comment faire ci-après pour afficher les #foo pas #bar ?

```
search
  fooBars = [#foo #bar]
  fooNotBars = [#foo]
  not(fooNotBars = fooBars) // ??? does not work!

bind @view
  [#value | value: "FOOBAR : {{fooBars.label}}"]
  [#value | value: "NOT FOOBAR : {{fooNotBars.label}}"]
```
