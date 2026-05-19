# Seminar2026
T5: Generic Go - The current compiler Go 1.24 has some limitations. Come up with your own examples that highlight these limitations. Develop some workarounds for these limitations.


## Intro
Wie nahezu alle modernen Programmiersprachen bietet auch Go ab Version 1.18 Unterstützung für Generics, die es Entwicklerinnen und Entwicklern ermöglichen, typsichere und wiederverwendbare Datenstrukturen sowie Funktionen zu implementieren. Bei näherer Beschäftigung mit diesem Sprachfeature, insbesondere im Vergleich zu etablierten Sprachen wie Java oder C++, werden jedoch schnell grundlegende Einschränkungen deutlich. 

Go erlaubt zwar generische Funktionen und Typen, jedoch können Methoden nicht mit eigenen Typparametern definiert werden. Das ist keine Bug oder eine fehlerhafte Umsetzung, sondern eine bewusste Designentscheidung. Und sie hat spürbare Auswirkungen auf die Praxis: Viele Implementierungsmuster, die in anderen Sprachen selbstverständlich sind, lassen sich in Go so nicht direkt umsetzen. Die vorliegende Seminararbeit untersucht diese Einschränkungen. 

Dazu wird zunächst der konzeptuelle Unterschied zwischen Funktionen und Methoden und einige Grundlagen in Go erläutert, bevor die konkreten Limitierungen des Typsystems im Umgang mit Generics analysiert werden. Abschließend werden mögliche Lösungsansätze und Umgehungsstrategien vorgestellt, die innerhalb der bestehenden Möglichkeiten, die Go hergibt, praktikabel sind.


## Go Grundlagen
Um den vollen Umfang der Limitierung des Nutzens von Generics in Go zu verstehen, werden wir uns zuerst grundlegende Bestandteile von Go genauer angucken und verstehen

### Structs

