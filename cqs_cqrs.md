# [Architecture](README.md)

## CQR or CQRS
* CQRS is the broader **architectural pattern**, and **CQS** is the general **principle of behaviour**
* CQRS takes the defining principle of **CQS** and extends it to specific objects within a system
* CQS works more on the method or class level, whilst CQRS takes the concepts of CQS and takes it to the application level
* CQRS allows for separate data stores and models for Commands and Queries and usually synchronized asynchronously using a service bus
* CQRS extends this concept into a higher level for machine-machine APIs, separation of the message models and processing paths