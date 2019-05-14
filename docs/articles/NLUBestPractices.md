## Best practices

EBA represents a paradigm shift in the way conversational AI is implemented for businesses. It is not based on intent-classification or ownership of a particular question which belongs to a single agent. The system is designed to be collaborative, where multiple agents work together to produce understanding and meaningful assistance. Free collaboration will always imply that one agent can introduce side effects across the total environment. For instance, our [Riddle agent](../../samples/Riddles.md) can entirely take over a conversation--by design, of course. The implication for developers is that a particular agent should follow a set of best practices to avoid introducing unwanted interference to their NLU pipeline. To this end, we outline key steps to developing a quality agent, one which can serve as a good citizen within a larger system.

* [The Golden Rule](#the-golden-rule)
* [Thorough ontology](#thorough-ontology)
* [Baseline NLU](#baseline-nlu)

Additionally, by way of negation, we provide a list of areas where best practices should be carefully considered to avoid common pitfalls:

* [Conceptual semantics](#correct-semantics)
* [Implicit knowledge](#implicit-knowledge)
* [Extending core functionality](#extending-core-functionality)
* [Atomic actions](#atomic-actions)

### The Golden Rule

The golden rule of EBA is to _describe_ your business domain to the machine in a simple, consistent, complete and straightforward way. We say this because EBA is not a chat bot. The typical workflow for chat bot development is to be provided with a list of questions or dialogs and to begin to implement each one in turn, the result of which is often a patch-work agent with poor NLU. Your goal as an EBA developer, on the other hand, is to explain your world, your domain, your business in a way that makes sense to the machine. When you begin your development process, you should begin by understanding your own domain and formulating this at a conceptual level. In fact, using our [debug mechanism](./DebuggingEBA.md#reasoning-in-debug-mode) you can even verify that your agent is conceptually consistent without executing one line of code. 

### Thorough ontology

Following the golden rule, it is only natural that, as you begin your development process, you concentrate on providing a consistent and thorough semantic ontology. It is here that you will want to consider basic questions like 'what entities does my business have?', 'what fields comprise my business entity?', 'what are the data types of these fields?'. Answering these questions, everything will be in place to implement the basic requirements of a coherent ontology, an ontology that will unlock the most out-of-the-box core functionality from EBA. The basic requirements here are to implement a set of foundational predicate relationships, viz. subclasses, attributes, and lists. Regarding subclasses, you will want to ensure that your entities are showable and that subtypes are organized appropriately, e.g. 'sales order' can be a derivative of the more generic 'order'. Additionally, for each entity, there will be a set of attribute fields which will be modeled as concepts within their own right. You will want to map these attributes to an appropriate json field. For each attribute concept, you will want to ensure that it subclasses from the appropriate core attribute concepts. For instance, if an attribute is a date attribute, it should subclass from `:Attribtue, :DateAttribute`. With these predicate declarations in place, EBA will be enabled to support data retrieval and data aggregation questions out-of-the-box once NLU is provided. Failing to implement the necessary predicate relationships can lead to issues such as a failure to distinguish between plural and singular references, the inability to perform data aggregation, and the inability to show data.


### Baseline NLU

Following the advice posed by the golden rule, rather than implementing an arbitrary set of questions, developers should initially focus on a foundational baseline for NLU. At the outset, patterns should capture basic references to entities and attributes declared in the ontology, even a one-to-one proportion between NL and concept annotation can suffice initially. Likewise, in the case of patterns, simple data retrieval actions will suffice, e.g. fetch object by id and fetch unqualified collection. These actions in conjunction with our lazy interface in the `@force` endpoint will enable you to ask a thorough set of data retrieval and aggregation questions, which are optimized to leverage your businesses APIs. In particular, implicit knowledge, follow up questions, more advanced KPI questions, etc. should not be modeled in the initial development phase. The reason for this baseline approach is that it ensures a coherent foundation for your agent, which can be incrementally extended. Developers who try to tackle more complex, implicit, or ambiguous questions during the beginning of the development process are liable to introduce side effects or certain design decisions which skew the remainder of the development process. Paraphrasing the remark of Aristotle, a small error at the outset can lead to large proportions.

### Correct semantics

Every agent loaded into an environment is invoked in an effort to provide understanding of natural language for its particular domain. In other words, each agent will provide a set of annotations to introduce conceptual knowledge into the system. Consequently, developers should be careful to introduce conceptual knowledge that actually makes sense, i.e. conveys the correct semantics. Consider this NL pattern and its directive to the system:

* `when was this order [delivered](example:DeliveryDate)`

In the first pattern, the intention of the developer is to support questions requesting the delivery date of some business entity such as an order or a shipment. However, this pattern introduces a side effect to the system. Any time the word 'delivered' occurs in a similar usage, it _must_ be considered as potentially a reference to some delivery date. This means that when we reason about questions such as 'the package was delivered', 'we delivered the software update', or 'show me the delivered goods', the system must consider all possible ways in which it can potentially resolve a _delivery date_. Clearly this represents an unwanted performance overhead for such questions which have nothing to do with dates. One could argue that this pattern is _prima facie_ incorrect. `example:DeliveryDate` is an date attribute, but there is no notion of _date_ expressed in the pattern, only the notion of _delivered_. The word 'delivered' can signify a multitude of things, and it should be considered as an aggressive move to annotate all references as a tied to some _date_. The developer's original intention can be better served here by encoding some more context. What the developer truly means to capture is that an inquisition like 'when' in conjunction with the notion of 'delivered' can be treated as a request to show a delivery date. For instance, the question 'when will the order be delivered?' clearly stands as a request for the delivery date. A simple adjustment to the pattern allows it to be rewritten as `[when will](:ShowDeliveryDate) the order [be delivered](:ShowDeliveryDate)?`. Additionally, a rewriting rule can then be added to indicate that `:ShowDeliveryDate` in conjunction with `:Order` should fetch the delivery date. This conveys the intended semantics in a safe way; we search for a delivery date if a request to show it is found in conjunction with an entity like `:Order`.

### Implicit knowledge

We must accept that conversation in general is often unqualified and implicit. Any true AI must be able to cope with this and understand implicit connections between different conceptual entities. To this end, EBA provides multiple mechanisms for capturing implicit knowledge within your agent. However such mechanisms can be abused and, in some cases, even result in performance issues. In the case of rewriting rules and certain actions, EBA allows developers to identify a set of higher level concepts into a set of lower level concepts. The ability to create or invent new concepts which do not exist within NL itself is a powerful capability, and, for this reason, we provide a penalty score for such created concepts in our scoring model. Consider the following rewriting rule: 

* `TimeframeWithFilter: a subClassOf :Showable => a(:Timeframe) -> :Filter(:Timeframe)`

This rule states that every time any showable entity is referenced in NL along with a timeframe, an additional `:Filter` concept should be introduced. This leads to performance and scoring side effect.

The performance side effect is that within a single question, there may be multiple showable entities and for each of these showable entities, we will consider how to make use of some filter operation. For agents focused on data retrieval and KPI support, the majority of its actions will expose a filter. This can lead to quite a substantive search at reasoning time. Furthermore, this rewriting rules violates [conceptual semantics](#correct-semantics), as it is a wrong and aggressive assumption that every time a showable entity and a timeframe occurs within a question that a filter must be applied. For instance, in the question 'show me my appointment for this Thursday', we are fetching a singular entity and there is no collection or filter involved. Of course, this can lead to a performance overhead for such questions which do not fit this use case at all.

The scoring side effect is that, since `:Timeframe` occurs on both LHS and RHS, this concept can be used both in the production of a new concept via this rule as well as by a separate action. Additionally the `:Filter` concept will inherit the score of the root node used to create it, viz. some showable entity. Such entities typically hold higher score as they can span multiple tokens, and, furthermore, there POS tag is that of nouns, which hold the highest score in our model. 

### Extending core functionality

While we enable developers to extend core functionality in cases where it makes sense, it requires caution. There is a fine line after all between _extending_ core functionality and _overriding_ core functionality. The latter will almost certainly lead to poor NLU. Our team has provided some safe guards to warn users about potentially overriding core functionality within NL patterns. Consider the following pattern:

* `[are there](:ActionShow) any orders with shipment number [matching](:Equivalent) "__"`

This pattern is an example of overriding core functionality, insofar as ambiguation is introduced in the place where existing functionality would otherwise be clear and expected. For instance, consider the annotation `[are there](:ActionShow)`. EBA intentionally distinguishes between existential questions (via `:IsThere`) and requests to show data (via `:ActionShow`). This annotation confuses the two, which leads to additional complexity as well as to potential system disambiguation follow ups. The difference between a chat bot and AI is that AI should actually understand, in a fine grained way, what is being asked. Forcing all types of interactions to be show requests, on the other hand, intimates that, in a contrived effort, one is seeking to build the former rather than the latter. Additionally, consider the annotation `[matching](:Equivalent)`. In EBA core, `:Equivalent` denotes strict equivalence and is reserved for NL triggers like 'equal to', 'equivalent', 'is', '=', etc. EBA also supports a fuzzy search via the concept `:Like`. 'Match' is more aligned with the latter rather than the former, but this annotation blurs this line across the system. Note that if there is ever a doubt about what concepts our core agents supports and their expected usage, we have exposed their implementation, including patterns and action signatures, to all users within the dev lab. 

### Atomic actions

Within EBA, actions are composed of two components, viz. conceptual signatures and execution logic. It is particularly important to think of actions with respect the first component. Actions represent conceptual signatures which allow your assistant to reason about actionable outcomes when resolving a particular question. We allow for a degree of polymorphism within actions. This enables reusable functionality across the same of class of entities, but, like other convenience features, it too can be abused. Actions which are highly polymorphic can lead to unpredictable runtime behavior as they are given permission to consume many concepts within a question. Regarding these signatures, there is a golden rule which should be followed--actions should be atomic in the functionality they provide. An action should not _both_ fetch unqualified entities as well as search against this entity for a subset. Following this rule allows the system to have atomic predictable building blocks when reasoning. To illustrate these points, consider the following action:

```
name:
example:GetObjectById

constraint:
object subClassOf example:Object, field subClassOf example:Field

input:
object (optional field (optional example:Identifier (data :UserString)))

output:
data object
```

Based on the provided name, you would expect this action to do one thing--to fetch an object by its id. However, a close look at the signature reveals that it _optionally_ can take any given field and identifier to perform data filtering. This action is clearly not atomic. Furthermore, the constraints are loosely provided, it allows any object and any field but does not necessarily constrain this field to belong to this object. Because these constraints are so loose, we can reasonably expect that, in the course of reasoning, this action can be used to consume _some_ attribute for _some_ entity in _some_ unexpected way. Decomposing this action into two atomic pieces, removing hidden optional functionality, and providing more substantive constraints will lead to a much more predictable behavior at reasoning time.
