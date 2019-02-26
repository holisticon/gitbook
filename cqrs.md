# CQRS and AxonFramework

[AxonFramework](https://www.axonframework.org/) is a lightweight and very promising framework to implement **C**ommand **Q**uery **R**esponsibility **S**egregation \(CQRS\) in JVM-based languages. Currently, I'm participating in a small educational project for building such a system using Kotlin and SpringBoot and had the opportunity to gather some experience about it. In this post I want to share my understanding of the basic concepts of CQRS and some design patterns applied during systems construction, following this architectural style.

The core idea behind the CQRS is to separate the application in two parts: a part modifying the data \(the so-called command side\) and a part reading the data \(so-called query part\). The query part holds data in a form best-suited for the query and gets informed on every data change. The query side is building a projection based on the events it receives.

A very good overview is given in the [Architecture Overview](https://docs.axonframework.org/part1/architecture-overview.html) of AxonFramework.

## Concepts
* [Modeling and visualizing CQRS systems](CQRS/modeling-and-visualizing-cqrs-system.md)

## Design Principles, Patterns, Anti-Patterns
* [Put your code into the right place](CQRS/putting-your-code-into-the-right-place.md)
* [Understanding Sagas](CQRS/understanding-sagas.md)