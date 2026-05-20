# Seminar2026
T5: Generic Go - The current compiler Go 1.24 has some limitations. Come up with your own examples that highlight these limitations. Develop some workarounds for these limitations.


## Intro
Wie nahezu alle modernen Programmiersprachen bietet auch Go ab Version 1.18 Unterstützung für Generics, die es Entwicklerinnen und Entwicklern ermöglichen, typsichere und wiederverwendbare Datenstrukturen sowie Funktionen zu implementieren. Bei näherer Beschäftigung mit diesem Sprachfeature, insbesondere im Vergleich zu etablierten Sprachen wie Java oder C++, werden jedoch schnell grundlegende Einschränkungen deutlich. 

Go erlaubt zwar generische Funktionen und Typen, jedoch können Methoden nicht mit eigenen Typparametern definiert werden. Das ist keine Bug oder eine fehlerhafte Umsetzung, sondern eine bewusste Designentscheidung. Und sie hat spürbare Auswirkungen auf die Praxis: Viele Implementierungsmuster, die in anderen Sprachen selbstverständlich sind, lassen sich in Go so nicht direkt umsetzen. Die vorliegende Seminararbeit untersucht diese Einschränkungen. 

Dazu wird zunächst der konzeptuelle Unterschied zwischen Funktionen und Methoden und einige Grundlagen in Go erläutert, bevor die konkreten Limitierungen des Typsystems im Umgang mit Generics analysiert werden. Abschließend werden mögliche Lösungsansätze und Umgehungsstrategien vorgestellt, die innerhalb der bestehenden Möglichkeiten, die Go hergibt, praktikabel sind.


## Go Grundlagen
Um den vollen Umfang der Limitierung des Nutzens von Generics in Go zu verstehen, werden wir uns zuerst grundlegende Bestandteile von Go genauer angucken und verstehen

### Structs
Ein Struct in Go ist ein bennanter Typ um eine Sammlung von zusammengehöriger Daten zu bündeln. Ein Struct besteht aus benannten Feldern mit einem explizieten Datentyp. Hat man schon einmal eine objekt-orientierte Programmiersprache wie Java/C++ verwendet, kann man sich ein Struct als Klasse ohne Vererbung vorstellen. 

```go
type User struct{
  Name string
  Email string
  Accountnummer int
}
```

Structs werden dann wie folgt entweder in Reihenfolge oder mit Feldreferenz iniziiert:

```go
var user1 User = User{"Max Mustermann", "maxmuster@mann.de", 1}
var user2 User = User{Name: "Erika Mustermann", Email: "erikamuster@mann.de", Accountnummer: 2}
```

Zugriff auf die Felder eines Structs werden mit dem Punktopertor `.` durchgeführt:

```go
var user1_Name string = user1.Name
```

### Methoden

Methoden in Go sind ganz normale Funktionen mit:
`Methodenname`,
`Parameterliste` und 
`Rückgabewert`.

Der entscheidende Unterschied zu normalen Funktionen ist der sogenannte Receiver. Der `Receiver` ist ein zusätzlicher Parameter vor dem Methodennamen, der angibt an welchen Typ die Methode gebunden wird. Wer die Klassen-Analogie von vorhin im Kopf hat, wird bemerkt haben dass im Struct die Methodendefinitionen fehlen. Das ist kein Zufall! In Go werden Methoden bewusst außerhalb des Structs definiert und erst durch den Receiver an einen Typ gebunden.

```go
//Normale Funktion
func WerBinIch(u User) string{
  return "Mein Name ist " + u.Name
}

//User wird als Parameter übergeben
WerBinIch(user1)
--> Mein Name ist Max Mustermann

//Methode mit Typ "User" als Receiver
func (u User) werBinIch() string{
  return "Mein Name ist " + u.Name
}

//Methode ist jetzt an den Typ "User" gebunden und kann mit dem Punktoperator benutzt werden
user1.werBinIch()
--> Mein Name ist Max Mustermann
```

### Interfaces

In Go ist ein Interface ein Typ, der eine Menge an Methodensignaturen definiert. Das Interface beschreibt welche Methoden ein bestimmter Typ haben muss ohne die Implementierung vorzugeben.
Anders als in anderen Sprachen wie Java wird ein Interface nicht mit `implements` implementiert, sondern erfolglt impliziet. Sobald ein Type alle Methoden implementiert dessen Methodensignaturen im Interface enthalten sind, erfüllt dieser das Interface. 

```Go
type printable interface{
  func printOut()
}

type Temperatrur struct{
  value float64
  unit string
}

func (t Temperatur) printOut(){
  println(t.value + "°" + t.unit)
}
```

In diesem Beispiel erfüllt der Type t (Temperatur) das Interface p (printable), weil t an alle in p definierten Methoden gebunden ist und sie implementiert. Das wäre in diesem Beispiel die Methode `printOut`. Da t das Interface p erfüllt wird t zu einem Sub-Type von p. Fortlaufent verwenden wir die folgende Notation dafür: `t <= p`. Das bedeutet dass immer wenn der Type p erwartet wird auch t gültig ist.

## Compiler-Limitierung: Go unterstützt keine generische Methoden

Seit der Einführung von Generics in Go 1.18 besteht eine zentrale Einschränkung: Generics können zwar für Structs, Funktionen und Interfaces verwendet werden, nicht aber für Methoden. Das hat zur Folge, dass bekannte Implementierungsmuster aus anderen Sprachen nicht einfach übernommen werden können und man stattdessen auf andere Konzepte ausweichen muss. In dieser Arbeit wird untersucht, welche konkreten Auswirkungen das hat, welche Alternativen es gibt und was diese Workarounds jeweils mit sich bringen. Als Einstieg betrachten wir zunächst ein typisches Problem, an dem sich die Einschränkung besonders deutlich zeigt.

### Generic Transformation Pipeline

Oft ist es notwendig, Daten zu transformieren. Beispielsweise an API-Übergängen oder beim Wechsel in eine andere Schicht einer Anwendung. Schreibt man für jeden solchen Übergang eine eigene Konvertierungsroutine, entsteht schnell redundanter, schwer wartbarer Code, der bei Änderungen an einem Datenmodell an vielen Stellen gleichzeitig angepasst werden muss. Um diesem Problem entgegezuwirken, bietet sich das Konzept einer Transformationspipeline an. Dabei wird eine Mapping-Funktion definiert, die Daten eines Eingangstyps aufnimmt und in einen Zieltyp überführt. Einzelne solcher Schritte lassen sich zu einer Pipeline zusammensetzen, sodass der Output eines Schritts direkt als Input des nächsten dient. Daten vom Typ A werden eingespeist, durchlaufen mehrere Transformationsschritte und kommen schließlich als Typ B heraus.
Der eigentliche Mehrwert dieses Ansatzes liegt in der Komposierbarkeit. Jeder Transformationsschritt ist eine eigenständige, wiederverwendbare Einheit, die unabhängig entwickelt und getestet werden kann. Die Pipeline selbst kennt keine konkrete Transformationslogik, sie orchestriert lediglich den Datenfluss. Dadurch lassen sich neue Übergänge durch Kombination bestehender Schritte abbilden, ohne jedes Mal neuen Konvertierungscode schreiben zu müssen.

![Logo](https://i.imgur.com/0NhXu92.png)

Zuerst müssen wir einen Type `Pipeline` definieren, welches die Daten hält. In den eckigen Klammern wird ein generischer Typparameter für das Strukt vorgestellt. `[A any]` bedeutet: Der Type A wird bei der Instanziierung gewählt. `Any` bedeutet, dass A jeden möglichen Type annehmen kann.

```Go
type Pipeline[A any] struct{
  items []A
}
```

Wir möchten jetzt versuchen einen User, wie oben vorgestellt, in einen String zu transformieren. Also soll aus einem User ein String werden, der den Namen des Users beinhaltet. Dazu bereiten wir zwei User vor und laden sie in die Pipeline.

```Go
type User struct{
  Name string
  Email string
  Accountnummer int
}

user1:= User{"Max Mustermann", "maxmuster@mann.de", 1}
user2:= User{"Erika Mustermann", "erikamuster@mann.de", 2}

pipeline := Pipeline[User]{items: []User{user1, user2}}
```
Jetzt benötigen wir noch eine Mapping Methode für die Pipeline um die Transformation durchzuführen. Wie oben beschrieben soll diese Methode es ermöglichen einen beliebigen Type in einen beliebigen Ziel-Type zu transformieren. 
Also benötigen wir die `Pipeline als Receiver`, weil sie die Items des Types A beinhaltet. Desweiteren müssen wir einen neuen Typparameter vorstellen für den Ziel-Type B. 

```Go
func (p Pipeline[A]) Map[B] (fn func(A) B) Pipeline[U] {            
    out := make([]B, len(p.items))
    for i, v := range p.items {
        out[i] = fn(v)
    }
    return Pipeline[B]{out}
}
```

Da die Mapping Methode für jeden Type sowie auch Ziel-Type funktionieren soll, muss der Mapping Methode als Parameter eine Funktion mitgegeben werden, welche die Logik der Transformation beinhaltet. Für diesen Fall, Users in Strings zu transformieren, würde die Function so aussehen:

```Go
fnUserToString:= func(u User) string { return u.Name }
```
Jetzt haben wir alles um jeden beliebigen Type in einen anderen zu transformieren. Auch sind jetzt sehr gut lesbare Verkettungen möglich. Erweitern wir den Code um eine Funktion, welche die Logik zum Encoden eines Strings in Base64 ermöglicht, dann könnte man dies so umsetzten:

```Go
fnEncodeBase64:= func(uname string) string {...}

result:= pipeline.Map(fnUserToString).Map(fnEncodeBase64)
```
Nun können wir es versuchen einmal auszuführen:
```diff
- syntax error: method must have no type parameters
```

Zwar unterstützt Go die Verwendung von Generics, aber nur mit Einschränkungen. Unser Code konnte nicht compiliert werden, weil: Go erlaubt nicht das Vorstellen neuer Typparameter bei Methoden und damit unterstützt Go keine generischen Methoden! Die Methodensignatur unserer `Map` Methode ist unzulässig, weil wir versucht haben nach dem Methodennamen einen neuen Typparameter vorzustellen: `... Map[B]`. 

Dass Go keine generischen Methoden unterstützt bedeutet allerdings nicht dass wir unser Problem nicht lösen können. Es gibt verschiedene Anstätze wie man diese Einschränkung umgehen kann, jede von diesen haben ihre Vor- und Nachteile welche im nachfolgenden Abschnitt vorgestellt werden.
