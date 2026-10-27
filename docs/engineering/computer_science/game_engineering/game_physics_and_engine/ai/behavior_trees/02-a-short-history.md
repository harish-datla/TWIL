# A short history

Why does an idea from a 2004 shooter now drive Unreal's AI editor and robot navigation software? Behavior trees came out of game AI and were formalized later by robotics researchers, and the textbook terms you'll meet in engine docs come from both sides.
Timeline: from Halo 2 (2004) to the first BT textbook (2018)

In short: game AI built the idea, starting with Isla's Halo 2 AI (shipped 2004, presented at GDC 2005)[1] and Champandard's 2007 article that set out sequences, selectors and decorators as the standard nodes[2]. Robotics formalized it, from the 2014 ICRA paper[4] to Colledanchise and Ögren's 2018 book[3]. A 2022 survey credits the key milestones to Michael Mateas and Andrew Stern's ABL behavior language and to Isla's talk[5]. Isla himself claims no priority, and calls his architecture a "hierarchical finite state machine (HFSM) or a behavior tree, or even more specifically, a behavior DAG"[1]: a directed acyclic graph, because one behavior subtree can sit in several places.
