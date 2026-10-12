# From Extensible Templates to Composable Rules: My D3.0 Architecture Experiment

*An unfinished attempt to change how a system responds to constantly changing requirements, rather than adding another special-purpose template every time.*

[简体中文](README.zh-CN.md)

## 01 | The problem was never just one trading strategy

Imagine building a market-monitoring application.

One user wants an alert when the lows of successive 15-minute candlesticks keep rising.

Another wants several consecutive bullish daily candles, but only if they meet additional pattern constraints.

A third describes something less precise: “First the price rises for a while, then pulls back, then rises for a few more candles, and pulls back again. I also want to compare the size of those moves.”

These are illustrative descriptions of the kinds of requests I was trying to handle, not verbatim customer quotations.

Implementing any one of them is not necessarily the hard part. The hard part is that the next request may be something I never anticipated.

If every variation requires another template extension, a special branch, or a new strategy program, will the system always have to chase the next requirement?

I wanted to explore a different approach: give the system a set of reusable capabilities, then describe new rules by composing the capabilities it already understands. For requirements that fit within that vocabulary, perhaps we would no longer need a separate implementation for each strategy.

I called this experiment D3.0.

It is not a finished trading product. It is a partially implemented prototype of a rule interpretation architecture, built to investigate that question.


## 02 | Why I moved from extensible templates to rule composition

Before D3.0, I worked on what I called D1.0 and D2.0.

They were not rigid, unchangeable templates. They were extensible: start with a structure for a class of monitoring requests, then add capabilities as new requirements appear.

For stable, well-understood requirements, I still think templates may be the simplest and most sensible solution.

My doubts began when new requests no longer fit the structure of the template itself.

Consider the difference between “three bullish candles in a row” and “several alternating periods of rising and falling prices, each with its own duration and magnitude, with comparisons between the periods.” The second request changes more than a number. It introduces questions about where each period begins and ends, the order of events, and which earlier value another calculation should reference.

Under the extensible-template approach, I could keep adding stages, reference rules, and special handling. It might work, but the template could become harder to understand and maintain with every exception.

That led me to a different question:

Instead of asking, “Which template does this requirement belong to?”, could I ask, “Which underlying values, operations, comparisons, and relationships make up this requirement?”

Here is the conceptual shift I was trying to make:

| Aspect | D1/D2: Extensible templates | D3.0: Composable-rule approach |
| --- | --- | --- |
| New requirement | Extend a template or add a branch | First try composing existing capabilities |
| Comparisons | Add template-specific handling | Reuse comparison and calculation primitives |
| Temporal conditions | Extend stages and state logic | Model continuity, stages, and history explicitly |
| New semantics | Still requires more code | May still require a new reusable primitive |
| Goal | Support known request families | Reduce repeated special-purpose implementations |

This is a comparison of design directions, not a measured comparison of code size, speed, or maintenance cost. I have not carried out that benchmark.

D3.0 was never meant to eliminate programming. If a rule introduces genuinely new semantics, the underlying engine will still need new capabilities. My hope was that changes would happen in reusable components, rather than forcing a fresh strategy program for every customer request.

The real question D3.0 tried to change was: When requirements change, which part of the system should change?


## 03 | Three different problems: understanding, expressing, and executing a rule

As I worked through these requests, I began to see three problems that were too easy to mix together.

First: understanding the requirement.

A user might say “a few candles,” “near the previous high,” or “a significant pullback.” Those phrases are not precise computational definitions. How many candles? What counts as a rise? Does “after” mean immediately after, or can there be a gap?

A system that executes the wrong interpretation perfectly is still doing the wrong job.

Second: expressing the confirmed rule.

Once the meaning is agreed upon, we need a structured representation that can be checked and combined. Different requests may reuse ways of reading values, calculating quantities, comparing results, combining conditions, counting consecutive matches, describing stages, or referring to past events.

Third: executing the rule.

A relatively unified evaluation mechanism should interpret the structured rule, produce a result, and ideally retain enough information to explain why a condition did or did not match.

My long-term sketch looked something like this:

```text
    A person describes a need in ordinary language
                       |
                       v
             Clarify ambiguous terms
                       |
                       v
           Express a confirmed rule
                       |
                       v
         Evaluate it using a common engine
                       |
                       v
          Return a result and an explanation
```

This is a conceptual workflow, not a claim that D3.0 implemented every step.

In particular, I imagined a front-end AI assistant that would ask follow-up questions, confirm the precise meaning, and help turn the answer into a rule. I sometimes thought of it as a “reception brain.” In more conventional terms, it would be a requirements clarification agent. That part remained a product idea, not a completed system.

I often described the approach to myself as “building with blocks.” But the difficult part is not naming the blocks. It is defining what each part means, deciding which combinations are valid, and making sure the evaluator gives consistent answers.


## 04 | Two ordinary rules reveal a surprisingly important problem

Consider a user request:

“Alert me when the lows of five consecutive 15-minute candles keep rising.”

That sounds straightforward. But it might mean either of these:

A. Examine five candles and compare their five lows. There are four adjacent comparisons.

B. Make five consecutive comparisons, each asking whether one candle's low is higher than the previous candle's low. That requires six candles.

Those are different rules. We cannot silently treat them as the same thing.

The D3.0 prototype contains node definitions and test definitions related to consecutively rising lows. To demonstrate the semantic distinction, I am using interpretation B here: five comparisons require six candles. Whether the prototype fully follows this exact convention still needs to be checked by running the relevant code and tests.

For illustration, assume the candles are closed and the lows below are ordered from oldest to newest:

| Illustrative inputs (oldest to newest) | Expected outcome |
| --- | --- |
| `100, 101, 102, 103, 104, 105` | Five strictly increasing comparisons; match |
| `100, 101, 99, 103, 104, 105` | One comparison fails; no match |

These are explanatory inputs and expected outcomes, not recorded results from a fresh D3.0 run. The “closed candles only” condition is a convention for this example; it does not imply that the prototype has a complete candle-close gating mechanism.

Now consider a second request: “Alert me when volume rises while price falls, for several consecutive candles.”

This requires at least two values, more than one comparison, a way to combine conditions, and a definition of consecutive matches. Even “price falls” is ambiguous: does it mean the close is below the previous close, or simply that the current candle closes below its own open?

In D3.0, I explored using reusable value-reading, comparison, logical-combination, and consecutive-counting capabilities for cases like these. The project contains corresponding rule-node construction and test definitions.

Neither example is an advanced trading strategy. That is precisely why I find them useful. Two different requests should not automatically require two entirely separate strategy engines. If the required semantics are already supported, composition may be enough to express the difference.

In the prototype, developers still assembled node definitions in code. It was not a finished no-code interface where end users could freely describe, build, and run any rule.


## 05 | Time and state were where the simple idea became difficult

Things changed when I tried to think about requests like this:

“First there is a large rise, then a pullback, then several consecutive candles of recovery.”

Where does the first stage begin? When has it ended? Is a gap allowed? Can stages overlap? What if historical data is missing? Should the answer be false, or should the system report that it could not evaluate the rule?

These questions cannot be solved by adding only a few greater-than and less-than comparisons.

My prototype gradually gained modules and test definitions for consecutive conditions, segmented patterns, local extrema, historical anchors, and state references. There is code for representing some multi-stage patterns. But the combined behavior and complex temporal semantics have not gone through a complete new execution verification in this review.

The remaining limits are substantial: restricted combinations, unfinished temporal relationships, incomplete state lifecycles, and unresolved integration details.

I learned that putting several capabilities into one system does not automatically mean they can be combined correctly. Execution order, historical state, caching, error propagation, and missing input data can all affect the meaning of the result. “The condition was false” is not the same as “the condition could not be evaluated.”

And there was a larger trade-off I had underestimated: the more expressive the rule system becomes, the more work its core must do to preserve consistent semantics. Abstraction is not free.

A composable architecture is not inherently simpler or cheaper than templates. When the requirements are stable, a template may be the better engineering choice. The extra engineering effort of a more flexible engine becomes worthwhile only if recurring variation and special-case maintenance justify that additional complexity.

That trade-off was one reason I eventually paused D3.0 instead of continuing indefinitely.


## 06 | What I actually built — and what the evidence does not yet prove

D3.0 was more than a diagram.

I built a Python prototype around a rule interpreter, a handler registry, a common result structure, and separate implementations for values, calculations, comparisons, logic, and some temporal operations, with AI coding tools assisting the implementation.

A read-only inventory in October 2026 identified 30 D3-related Python source files and 42 related test files. These numbers describe what was present in that inventory; they are not a percentage-complete score.

I find it useful to separate three levels of engineering evidence:

| Level | What it establishes |
| --- | --- |
| **A — Source exists** | Modules, functions, contracts, and implementations can be inspected. |
| **B — Tests are defined** | Test cases and assertions describe expected behavior. Their existence alone does not prove successful execution. |
| **C — Actual execution is verified** | A known version is run with documented inputs and outputs, producing a result that can be independently checked. |

The recent review established mainly A- and B-level evidence. It did not include a new full C-level integration verification. Some tests use mock components, and one group of composite tests has an incomplete handler-registration setup. Elsewhere, there are separate test setups that do register the relevant logic handler.

For that reason, I cannot claim that all written tests pass, that every expressible rule executes correctly, or that the system is ready for production.

Based on the code inspection, the core execution structure and portions of the composition model are real. But some accepted rule syntax is not aligned with what the runtime actually supports, and the larger problems of temporal state, automatic rule generation, cross-domain adapters, and production reliability remain unfinished.

That is why I call D3.0 an unfinished architecture experiment.

If I publish a runnable demonstration in the future, I want it to include a sanitized dataset, an exact version, the real output, and the limits of what was tested.


## 07 | Related ideas existed long before my project

As I learned more, I discovered mature approaches to composable rules, expression languages, rule engines, and complex event processing. [JsonLogic](https://jsonlogic.com/), the [Common Expression Language (CEL)](https://cel.dev/), and [Drools](https://docs.drools.org/latest/drools-docs/drools/rule-engine/index.html) are examples of systems I can learn from.

I do not claim D3.0 invented rule composition, or that it is faster, more general, or more reliable than established engines. I have no benchmark supporting such a claim.

What makes this work personal to me is the route I took to it. I first encountered a problem through my own business requirements. I began questioning the architecture, tried to define an alternative, and built part of that alternative with AI assistance before gaining a broader understanding of the existing field.

A fair question would be: why not use an established rule engine instead? I think that is a useful question, not a criticism to avoid. D3.0 was an exploration, not a rejection of mature tools. If an existing system solves the problem better, I would rather learn from it than insist on rebuilding everything.


## 08 | A little about the person behind the experiment

So far, I have talked about the technical problem. Here is my background.

I was born in 1991 and graduated from a junior college in China. My background is in e-commerce, trading, and hands-on business operations rather than computer science.

December 15, 2025 was the date I first started using AI chat and experimenting with AI-assisted software development. Since then, I have become something of an AI enthusiast. I keep wanting to turn ideas in my head into working tools.

D1.0, D2.0, and D3.0 were part of that journey. What began as an attempt to handle changing market-monitoring requests became a roughly one-to-two-month effort to reconsider how those requests should be represented and executed.

I brought real business scenarios into the project, worked through architecture and implementation choices, and refined the prototype through repeated development and code review.

The experience also strengthened a habit of mine: looking for structures that might be shared between problems that initially seem unrelated.

Eventually, I put D3.0 on hold as the engineering challenges grew and I needed to prioritize other projects.

The prototype remains unfinished, but it left me with concrete code to examine and a different way of approaching complex requirements.


## 09 | Why I kept returning to this idea in other fields

Some time after putting D3.0 aside, I spoke with an e-commerce operations agency managing hundreds of online stores across different retail categories.

Each shop had its own products, policies, exceptions, and customer questions. I wondered whether simply loading more and more documents into a knowledge base would solve the deeper problem. The system still had to identify what a customer needed, which rule applied, which facts should come from live business systems, and when a human should take over.

The question reminded me of D3.0: could we separate user needs, product attributes, constraints, and decision relationships, while supplying different knowledge and tools for each business?

That remained an architectural thought experiment. I did not deploy a multi-store customer-service system built on D3.0, nor did I port the D3.0 code into that domain.

Later, I started researching HAIT, an experimental project aimed at organizing multiple AI agents to cooperate on software tasks.

Its users might eventually need tools for retail, finance, industrial workflows, or other specialized fields. The professional knowledge and safety requirements are different in every domain. Still, I find myself asking whether some parts of the work — clarifying goals, representing constraints, selecting tools, evaluating results — can share a reusable structure.

I treat these as questions about reusable design methods, not a claim that one codebase can work unchanged in every industry.

D3.0 simply left me with a useful question: what is truly different between these problems, what repeats, and where should the system draw the boundary between the two?


## 10 | Why I am sharing an unfinished project

I am publishing these notes because I would like to exchange ideas with people who work on rule engines, software architecture, business automation, and AI-assisted development.

If you have ever found yourself adding another special case whenever a customer describes a new workflow, how did you handle that growing complexity?

If you have worked with JsonLogic, CEL, Drools, or other systems, I would be interested in learning which problems benefit from rule composition and which are better solved with straightforward application code.

I also hope this encourages people whose background does not follow a traditional software-engineering path. You do not need a perfect résumé to start exploring a real problem. But you do need to be willing to test your ideas, discover their limits, and improve them.

If these questions resonate with your own experience, you can follow future updates through [my GitHub profile](https://github.com/juhaoxianshen). I would be glad to discuss them with other developers.

*Note: AI assisted with the English translation and editing.*
