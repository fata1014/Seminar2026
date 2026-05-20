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

Zuerst müssen wir einen Type `Pipeline` definieren, welcher die Daten hält. In den eckigen Klammern wird ein generischer Typparameter für das Strukt deklariert. `[A any]` bedeutet: Der Type A wird bei der Instanziierung gewählt. `any` bedeutet, dass A jeden möglichen Type annehmen kann.

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
Also benötigen wir die `Pipeline als Receiver`, weil sie die Items des Types A beinhaltet. Des Weiteren müssen wir einen neuen Typparameter für den Ziel-Type B einführen.

```Go
func (p Pipeline[A]) Map[B] (fn func(A) B) Pipeline[B] {            
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
Jetzt haben wir alles um einen Type in einen anderen zu transformieren. Auch sind jetzt sehr gut lesbare Verkettungen möglich. Erweitern wir den Code um eine Funktion, welche die Logik zum Encoden eines Strings in Base64 ermöglicht, dann könnte man dies so umsetzen:

```Go
fnEncodeBase64:= func(uname string) string {...}

result:= pipeline.Map(fnUserToString).Map(fnEncodeBase64)
```
Nun können wir es versuchen einmal auszuführen:
```diff
- syntax error: method must have no type parameters
```

Zwar unterstützt Go die Verwendung von Generics, aber nur mit Einschränkungen. Unser Code konnte nicht compiliert werden, weil Go es nicht erlaubt, neue Typparameter bei Methoden einzuführen – Go unterstützt also keine generischen Methoden. Die Methodensignatur unserer `Map` Methode ist unzulässig, weil wir versucht haben nach dem Methodennamen einen neuen Typparameter einzuführen: `... Map[B]`.
Dass Go keine generischen Methoden unterstützt bedeutet allerdings nicht, dass wir unser Problem nicht lösen können. Es gibt verschiedene Ansätze wie man diese Einschränkung umgehen kann, jeder dieser Ansätze hat seine Vor- und Nachteile, welche im nachfolgenden Abschnitt vorgestellt werden.


## Workarounds

### 1) Map als generische Funktion anstatt als generische Methode

Wenn die Pipeline nicht mehr als Receiver verwendet, sondern als Argument übergeben wird, kann man A sowie auch B in der Parameterliste der Funktion deklarieren (Beachte dass neue Typparameter nur bei Methoden nicht zulässig sind!). Da Pipeline nicht mehr der Receiver ist, gibt es auch keine Methode mehr, sondern "nur noch" eine Funktion und ist damit zulässig.

```Go
func Map[A, B any](p  Pipeline[A], fn func(A) B) Pipeline[U] {
    out := make([]B, len(p.items))
    for i, v := range p.items {
        out[i] = fn(v)
    }
    return Pipeline[B]{out}
}
```
Diese Funktion ermöglicht es uns einen beliebigen Type A zu einem beliebigen Type B umzuwandeln und dabei auch Typsicherheit zu garantieren. Leider geht dadurch aber auch die Möglichkeit der Verkettung verloren. Vorher konnten wir die Pipeline im Methodenaufruf elegant von links nach rechts darstellen, jetzt muss entweder jeder Transformationsschritt einzeln durchgeführt werden oder wenn man versucht die Aufrufe zu verschachteln, muss man den Aufruf von innen nach außen lesen.

```Go
// Jeder Schritt einzeln ausführen
names  := Map(pipeline, fnUserToString)
base65Names  := Map(names, fnEncodeBase64)

// Verkettung, aber! Von innen nach außen
result := Map(Map(pipeline, fnUserToString), fnEncodeBase64)
```

> [!TIP]
> - Typsicherheit garantiert
> - Funktion erfüllt: Type A in Type B überführen
> - Kein zusätzlicher Boilerplate-Code nötig

> [!WARNING]
> - Erlaubt keine Verkettung
> - Verschachtelte Aufrufe sind von innnen nach außen

### 2) Methode ohne Typeparameter

Eine andere Möglichkeit besteht darin, die Typparameter aus der Methodensignatur komplett zu entfernen. Dafür ist es notwendig, dass die Pipeline in ihrem `items[]` Feld `any`, also egal welchen Type speichern kann und der Typparameter entfernt wird. Dadurch ist es nicht mehr nötig die Map Methode mit dem Typparameter B zu versehen, da jetzt als Rückgabewert `[]any` verwendet werden kann. Dadurch muss aber jetzt der Caller der Funktion abschließend den Rückgabewert in den Ziel-Type casten. Dies hat den Nachteil, dass keine Typsicherheit mehr garantiert ist und es zu Panics zur Laufzeit kommen kann, wenn zur Laufzeit ein falscher Type gecastet wird.

```Go
type Pipeline struct { // keine typparameter mehr!
    items []any
}

// hier auf keine typparameter mehr!
func (p Pipeline) Map(fn func(any) any) Pipeline {
    out := make([]any, len(p.items))
    for i, v := range p.items {
        out[i] = fn(v)
    }
    return Pipeline{out}
}
```
In dieser Variante verliert man zwar die Typsicherheit, erhält aber die Möglichkeit die Methodenaufrufe zu verketten, wie in der ursprünglichen Version angedacht. Die Verkettung erkauft man sich mit dem Verlust der Typsicherheit und der Möglichkeit von Panics zur Laufzeit.

```Go
result := pipeline.
    Map(func(v any) any { return v.(User).Name }).
    Map(func(v any) any { return v.(string).EncodeBase64() }) // panics at runtime if wrong
```
> [!TIP]
> - Verkettung funktioniert von links nach rechts
> - Funktion erfüllt: Type A in Type B überführen
> - Kein zusätzlicher Boilerplate-Code nötig

> [!WARNING]
> - Keine Typsicherheit
> - Panics zur Laufzeit möglich

### 3) Wrapper Types 

Wenn man verkettete Aufrufe erst nach dem Mapping verwenden möchte und es zusätzlich nicht viele Mapping-Schritte gibt, ist diese Variante eventuell geeignet. Man führt für jeden Mapping-Schritt einen eigenen Wrapper-Type ein, welcher beide Types A und B sowie die vorherige Pipeline beinhaltet. Der zusätzliche Aufwand und Boilerplate-Code wächst enorm bei mehreren Mapping-Schritten, jedoch bietet er komplette Typsicherheit und erlaubt zumindest verkettete Aufrufe nach dem Mapping.
```Go
type Pipeline[A any]  struct { items []A }
type Mapped[A, B any] struct { items []B; src Pipeline[A] }

func Map[A, B any](p Pipeline[A], fn func(A) B) Mapped[A, B] {
    out := make([]B, len(p.items))
    for i, v := range p.items { out[i] = fn(v) }
    return Mapped[A, B]{out, p}
}

// Mapped hat noch andere Methoden: Filter, Reduce, etc.
// Filter operiert auf dem gemappten Type B, nicht auf dem ursprünglichen Type A
func (m Mapped[A, B]) Filter(fn func(B) bool) Mapped[A, B] { ... }

// Typsicher aber für jede Transformation ein neuer Type
mapped := Map(pipeline, fnUserToString)  // Mapped[User, string]
result := mapped.Filter(func(name string) bool { return name != "" })
```
> [!TIP]
> - Verkettung funktioniert sobald Mapping abgeschlossen
> - Typsicherheit

> [!WARNING]
> - Neuer Type für jeden Mapping-Schritt
> - Kann extremen Aufwand verursachen
