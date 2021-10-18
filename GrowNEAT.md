> ⚠️ This documentation is a work in progress. Forgive the haphazard notes and rambles!

# Domains
* The environment defines a number of domains, e.g. "PLAYER", "CARD", "ABILITY".
* During the runtime, instances of domains can start and cease.
  * e.g. where a network is playing a card game, an instance of the CARD domain may start when a new card comes into play, and once that card leaves play the instance may cease.
* The "BASE" domain refers to a domain that has only one instance. This instance exists for the entire lifetime of the runtime environment.

# Scopes
* A scope may refer to an instance of a domain, or a combination of instances.
* For example, say there are three instances of the CARD domain. In this case there is a <CARD> scope for each instance, a <CARD,CARD> scope for each pair of cards, and a <CARD,CARD,CARD> scope for the whole triad.
* Scopes are commutative. Say there are two instances of the CARD domain named A and B, there is only one resulting <CARD,CARD> scope. It may be <A,B> or <B,A> at the discretion of the implementation, however both are considered to be the same scope, not two separate scopes.

# Genomes/Genotype
* Like in standard NEAT, a genome is a sequence of nodes and connections.
* Nodes and connections have the same data as before (ids, innovation numbers, etc), but also additional information.
* Nodes now additionally have a scope.
* Connections now additionally have combination specifier, or _combspec_ for short.

```
NODE:
id    : int
kind  : INPUT, HIDDEN, OUTPUT
scope : scope

CONNECTION:
innovation : id
input      : node
ouput      : node
weight     : float
enabled    : boolean
combspec   : int
```

## Combination Specifiers
Okay this isn't going to be neatly written up because I've spent ages figuring it out and just need to get it down.

Let's say you have the following setup.
* There are two nodes, node 5 and node 6.
* Node 5 is scoped <CARD,CARD>.
* Node 6 is scoped <CARD>.
* There are two instances of the CARD domain, named A and B.
* Therefore there are three scopes, two <CARD> scopes named A and B, and one <CARD, CARD> scope named AB.

How should a connection gene connecting 5 to 6 work? Remember there is one 5 node in the AB scope, but two 6 nodes, one in the A scope and the other in the B scope. The connection could connect the five to both 6s. However, seen as AB connects to the scopes of two different cards, it would be nice if it could choose to treat each one differently. The connection could instead connect 5 to just 6A, or just 6B. But then how should it decide? And what if the network actually does want to connect the 5 node to both 6 nodes? What if it wants to do that, but wants each connection to have a different weight?

Here's how we solve it.

The 5-6 connection can only choose to connect to one of the 6 nodes. The combination specifier determines which one it connects to (1 for A, 2 for B). However, connections that connect the same nodes in the same direction BUT with different combination specifiers are considered different connections. Therefore it is valid for a genome to contain one genome for a 5-6(1) connection and a 5-6(2) connection.

The tricky bit is knowing how high the combination specifier should be able to go.

Take a simple case first. Node 7 is scoped to a group of x CARDS, and node 8 is scoped to a group of y CARDS. The number of possible ways an x y connection could be interpreted is a!/(b!*(a-b)!) where a is the largest of x and y and b is the smallest.

Now let's say the scopes can comprise multiple kinds of domains. To find the combination specifier, you go through each kind of domain, calculate a!/(b!*(a-b)!), and then multiply all of the results.

For example, say node 9 is scoped <PLAYER, PLAYER, CARD> and node 10 is scoped <CARD, CARD>, you could calculate the highest possible combination
specifier as follows:

```
PLAYER DOMAIN
x = 2
y = 0
a = max(x,y) = 2
b = min(x,y) = 0
R1 = a!/(b!*(a-b)!) = 1

CARD DOMAIN
x = 1
y = 2
a = max(x,y) = 2
b = min(x,y) = 1
R2 = a!/(b!*(a-b)!) = 2

COMBINATION SPECIFIER
S = R1 * R2 = 2
```
(A nice touch is that when x=0 and y=0, R=1, so all domain kinds not included in either scope can be ignored without affecting the final result.)

Though a bizarre situation, this result makes sense. Each pair of cards, e.g. A and B, has one <CARD, CARD> scope, and thus only one node 10. But between those two cards, there are two node 9s, one in the <PLAYER, PLAYER, A> scope and the other in the <PLAYER, PLAYER, B> scope. The connection specifier dictates which of the two node 9s should be connected to the node 10.

Another example. A <PLAYER, CARD> node to a <CARD, ABILITY> node.
```
PLAYER DOMAIN
R1 = 1!/(0!*(1-0)!) = 1

CARD DOMAIN
R2 = 1!/(1!*(1-1)!) = 1

ABILITY DOMAIN
R3 = 1!/(0!*(1-0)!) = 1

COMBINATION SPECIFIER
S = R1 * R2 * R3 = 1
```

Another example. A <PLAYER, CARD, CARD> node to a <PLAYER, PLAYER, CARD, CARD, CARD, CARD> node.
```
PLAYER DOMAIN
R1 = 2!/(1!*(2-1)!) = 2

CARD DOMAIN
R2 = 4!/(2!*(4-2)!) = 6

COMBINATION SPECIFIER
S = R1 * R2 = 12
```

# Mutation of the genotype

## Creating a new connection
This is reasonably straight forward. Just as before, two random nodes will be selected and connected with a [random weight]>[check that's how neat does it], and a random combination specifier will be selected. Just as before, duplicate connections are not allowed. Whereas before however a connection was considered duplicate if it connected the same nodes in the same direction, now there is the additional requirement that it must have the same combination specifier.

## Creating a new node
This is more complicated. Just as before, a new node is made by disabling a connection, creating a node, and then creating two new connections that connect the node at the start of the old connection to the node and the node to the node at the end of the old connection.

The challenge is determining how the new node should be scoped. My current thinking is this:
* I am reasonably certain that scoping nodes doesn't mess up the innovation system vital to NEAT working. I think so long as the following rule is stuck to we're fine: If two networks independently mutate the same connection gene to add a new node, but the resultant nodes are of different scopes, the newly created connection genomes must all have different innovation numbers.
* Assuming I'm right in saying this, then there's a question of "how should new nodes be scoped?". They could stick to the scopes of their parents, which would be fantastic for constraining the possibility space. Given one domain kind, there are an infinite number of possible scopes. That said, constraining the possibility space inherently risks blocking off potentially valid or even more effective solutions.
* However, I think constraining is the best rule. Therefore, I would say the following rules make the most sense for when a new node is added.
  * If both nodes on either end of the new connection have different scopes, one is selected at random for the new node. The weights of the new connections are set such that this change has minimal impact on the network.
  * If both nodes on either end of the old connection have the same scope, the new node has that scope. (this is really just a sub-case of the first rule though)
* Depending on which scope the new node ends up in, either it's inbound or outbound connection will have the same combination specifier as the old connection.
* A node on constraining the possibility space. If there is a scope you want the network to be able to explore, but you don't have any inputs or outputs in that scope, you can still specify a single hidden node in that scope. That way the network will be able to make connection and build up a network there if it so pleases.

# Networks/Phenotype
* During runtime, when an instance of a domain is started to ceased, all of the resultant scope instances are in turn started or ceased.
* When a new scope is started, any nodes of that scope specified in the genome are created, along with any connections to or from them. When the scope is ceased, the node is destroyed, along with inbound or outbound connections.
* That's basically it.
* Note the input and output nodes are also scoped. The use of scopes, therefore, allows for a variable number of inputs and outputs at runtime.

# My Only Unanswered Question!
Let's say you connect a <CARD> node to a <BASE> node. Should that weight of that connection scale with the number of <CARD> scopes?

That's it. That's the question. And honestly, I have no idea.

I guess it depends whether the network should want to "count" the number of cards sending a certain signal, or average the signal sent from a number of cards. This could be a property of the connection itself, because I'm not sure it's possible to say in the abstract that one of those would be useful to the network and the other would not.

Is there a way it can create one using the other? I don't think there is you know? I guess you could provide as input "here are how many card scopes there are", then it could count and divide to get the average. A quick google shows however that neural networks aren't great at division or multiplication. That also makes it difficult to send the average over the connection and then multiply by the count.

My temptation then is to say that the connection should just count. I.e. no, it shouldn't scale to the number of scopes. My inclination for this is that nodes work on an activation principle. Let's say there's lots of cards, I think it would be most useful for the network for a card node to be able to signal "I am a 3", causing the base node to activate and say "I have a 3 in my hand". I suppose there could be a utility to "what proportion of my cards are 3s, and activate if I hit a certain percentage", but I can't see that being useful as often. Like, say the network wants a base node that activates once there are 5 3s, it can just set the weight of the connection low enough so that it takes 5 active connections to trigger it. "Please trigger if 20% of my cards are 3s" - eh. I can totally imagine that in a different scenario that's what you need, but if I have to choose I think I'm going to choose counting.

It has the nice touch as well that it perfectly mimics the existing behaviour of connections, and so feels like a logical extension. Having connections that don't scale to the number of scopes also means that you don't have to revaluate the network when scopes are started or ceased.

Okay awesome! I think that's GrowNEAT specified! Nice!
