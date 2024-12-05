## Basics
#### Build your understanding of software architecture
- Software architecture is like a house's structure, defined by its shape, dimensions, and layout, represented through a building plan. 
- Key structural elements like these are crucial and costly to change later.
- Architecture is also essential for building software systems
- If we don't place enough emphasis on a  system’s architecture. it will likely not scale, will be unreliable and difficult to maintain.
- A software architecture diagram specifies its structure , guidelines, constraints and vision of the final result.(user interfaces, services, databases and communication protocols)
- A software architecture diagram just like a building architecture diagram doesn't go in to design details(whats the network bandwidth ? ram, SSD of the instances ? docker vs containerd etc ). It instead focuses on structural part.
- Exercise
     - Q1. Gardening is another useful metaphor for describing software architecture. can you describe how planning a garden might relate to software architecture?
     <details><summary> Solution  </summary> <br>
	-Layout of garden ? need for sprinklers ? how different flowers should be delegeated to certain areas to make it beautiful and self sustaining?  maintainance schedule to remove weeds ? choosing certain plants that make weeds removing less likely ?
	</br>
     - Q2. What features in a home can you list trhat are structural and related to its architecture ?
		 <details><summary> Solution  </summary> <br>
		 no of floors ? floor layout ? room layout ? dimensions inside the floor layout ? ventilation design ? chimney ? fire escape design ? plumbing and electricity design ? 
		 </br>
		

#### Dimensions of Software Architecture
 - ###### Architectural Characteristics
	 -  this dimension describes what aspects of the system the architecture needs to support(scalability ? testability ? availability ? )
- ###### Architectural Decisions
	 -  this dimension includes important decisions that have long term or significant implications for the system(SQL vs NoSQL ? microservices vs monolithic vs AWS vs GCP etc etc?)
- ###### Logical Components
	 -  this dimension focuses on functionality aspect of the system and how different functionalities interact with each other(payment processing , order management , inventory management, payment successful -> order placed -> reduce inventory etc etc )
- ###### Architectural Style
	- this dimension focuses on how the structure of our system should look like and how our code design/philosophy should look like(Microservices + Event Driven Design , Monolith + Behavioral Driven Design + Test Driven Design ?)


#### Puzzling out dimensions 
 ![[sa_dimensions.png]]

all the dimensions of software architecture must fit together and each of them should fit together like this , while all the pieces must fit together some of the pieces are more dependent on the pieces next to them, like this. Depending on the system we aim, these puzzle pieces keep changing the dependence on each other.

 ![[sa_puzzle_pieces_seperate.png]]![[sa_puzzle_pieces_fitting.png]]
**Question**
-> Do we need all four dimensions when creating an architecture? are any of them disposable?
<details><summary>Solution </summary>The answer is no, while you can modestly create a system that might work that runs and hangs by a thread by considering 2 or 3 dimensions, if you dont consider all the four then the software would not survive long term without major re architecting  </details>

#### Dive IN : Architectural characterstics


