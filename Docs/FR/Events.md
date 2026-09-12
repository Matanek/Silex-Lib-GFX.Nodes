# Communiquer entre Nodes

Une méthode demande une action à un objet connu. Un événement annonce un fait
sans connaître ses destinataires : un personnage est entré dans une zone, une
porte s'est ouverte, un score a changé. Utilisez des événements typés pour ces
notifications et gardez les commandes sous forme de méthodes ordinaires.

## Publier un événement

Le producteur garde un `EventEmitter<T>` privé et expose seulement son
`Event<T>`. Le consommateur peut écouter cet événement ; il ne peut pas l'émettre.

```sx
use GFX.Nodes

class Score:Nodes.Node {
    private var updates:Nodes.EventEmitter<int> = Nodes.EventEmitter<int>()
    var score_changed:Nodes.Event<int> { get { return self.updates.event } }
    private var value:int = 0

    func add(points:int) {
        self.value += points
        self.updates.emit(self.value)
    }
}

class ScoreDisplay:Nodes.Node {
    private var score:Score
    init(score:Score):super("ScoreDisplay") { self.score = score }

    override func on_ready() {
        self.listen(self.score.score_changed, self.show_score)
    }

    func show_score(value:int) { print("Score : $(value)") }
}
```

`listen` déduit le type du message. Le callback peut être une fonction libre
ou une méthode liée à une classe possédée : cette dernière conserve son objet
récepteur. Une closure qui emprunte des variables locales ou une méthode liée
à une structure reste lexicale et ne convient pas à un abonnement durable.

## Durée de vie

`node.listen(event, callback, delivery)` appartient au Node qui appelle
`listen`, même si le callback vise un autre objet. Le Node doit être monté.
`on_ready` est le point habituel d'inscription.

Le retour est une `Connection` : `is_connected()` lit son état et
`disconnect()` déconnecte immédiatement, même pendant une émission. Répéter
`disconnect()` est sans effet. Abandonner la variable qui contient la connexion
ne déconnecte pas l'abonnement.

Retirer le Node de l'arbre déconnecte ses abonnements dès la demande de retrait,
avant ses callbacks de sortie. L'arrêt de l'arbre suit la même règle. Un nouveau
montage rappelle `on_ready`, qui peut créer de nouvelles connexions.
La pause, `set_process(false)` et `enable(false)` conservent les abonnements :
une notification reçue pendant ces états reste délivrée.

## Ordre et livraison

La livraison `EventDelivery.immediate`, par défaut, appelle les destinataires
pendant `emit`, dans l'ordre d'inscription. Une émission prend un instantané
des connexions : un abonnement ajouté pendant un callback attend la prochaine
émission ; une connexion déconnectée avant son tour est ignorée. Une émission
récursive est immédiate elle aussi.

Pour différer un traitement :

```sx
self.listen(area.body_entered, self.on_body_entered, Nodes.EventDelivery.deferred)
```

Les messages différés rejoignent l'arbre du destinataire. Cet arbre en livre
un lot à la fin du démarrage, de chaque passage de mise à jour des Nodes et de
chaque pas physique terminé. Les messages ajoutés pendant ce lot attendent la
prochaine frontière ; une déconnexion annule aussi les messages encore en
attente. L'ordre du lot suit celui de la mise en file.

Le message est conservé selon la sémantique ordinaire des valeurs Silex : ses
objets de classe restent partagés. Ce n'est pas une copie profonde de leur état.
Un événement partagé explicitement entre deux arbres utilise une file par
arbre destinataire. Aucun bus global n'est installé.

Un callback immédiat s'exécute dans le contexte de son émetteur. Les accès
`resources()` du Node destinataire exigent qu'un passage de son propre arbre
soit actif. Utilisez la livraison différée pour un émetteur extérieur à cet
arbre lorsque le callback doit accéder à ses ressources. Les événements
physiques d'une scène sont émis dans le passage physique de son arbre.

Les preuves exécutables se trouvent dans
[`Events.sx`](../../Tests/Consumer/Tests/Events.sx).

Un exemple complet, exécutable sans fenêtre, assemble un score, une zone et
un rayon dans [`EventsAndSensors.sx`](../../Tests/Consumer/Smokes/EventsAndSensors.sx).
