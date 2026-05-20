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

Seit der Einführung von Generics in Go 1.18 gibt es bei Generics eine große Einschränkung, zwar können Generics zur Definition von Structs, Funktionen und Interfaces verwendet werden aber nicht für Methoden. Das führt dazu, dass aus anderen Sprachen bekannte Programmiermuster nicht einfach wiederverwendet werden können und man sich anderen Konzepten bedienen muss. Warum das überhaupt ein Problem für manche Entwickler darstellt
