# What's next

Streaming keeps the world resident around the player; the changes the player makes to it have to outlive the session. The next module, Save Games & Serialization, persists that mutable state: the object graph (you cannot write a pointer to disk), versioning so old saves still load, and the atomic write that keeps a crash from corrupting a save. The full path is on the series hub. 
