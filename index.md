## On _Understanding The Four Rules of Simple Design_

**"Striving for a simple design — one that is adaptable to changing needs — is the key to a “better design.”**

Corey Haines’ _Understanding The Four Rules of Simple Design_ is a succinctly written exploration of Kent Beck’s four rules for writing simple code. In a series of essays and examples, Haines discusses different ways to think about developing code within the context of Beck’s rules. 

A foundational idea that he tries to get across immediately is that when talking about simple code and good design, we are talking about code that is adaptable. “The one constant that we know for sure in software development is that things are going to change,” Haines writes, “and these changes to the system require changes to the underlying codebase. If we don’t pay attention, our code rots.”

Simple code is flexible at its core, and we create simple code by applying the four rules of simple design as principles. The rules, listed in order of priority, are as follows:

1. Tests Pass
2. Expressed Intent
3. No Duplication (DRY)
4. Small

**Tests Pass**

The first rule seems obvious. Your code must pass all tests to verify that your system works. But by what metrics are we writing these tests?

Using Conway’s Game of Life as the backdrop, Haines asks the reader to consider the difference between testing state and testing behavior. When testing state, we perform an operation and afterwards check if a state change has occurred. Behavior driven testing, however, asks us to center our tests around the behaviors that we expect from our system. It builds only what is absolutely necessary for our system to work and only at the time that it’s required, keeping our codebase as streamlined as possible. 


Because I have very limited experience with writing test driven code, the distinction between state and behavior driven testing is something I had not considered before, and I feel that I need to do more research regarding the advantages and disadvantages of these types of testing. I wish Haines’ book had gone a little more in-depth in its discussion of this topic.

**Expressed Intent**

Expressed intent refers to the code’s readability. Is it easy to understand? How do we improve its readability?

Haines writes that “one of the most important qualities of a codebase, when it comes time to change, is how quickly you can find the part that should be changed.” A quick identification can only occur if our code expresses its intent--what it fundamentally does--wherever required. Therefore the name of a test, or a method, should always be reflective of the thing itself.

The organization of the code itself is another important facet to writing simple, readable code. Determining the right place for a method or a behavior can be tricky, but we must always try to keep unrelated ideas separate so as to not create confusion about the code’s intent. 

**No Duplication (DRY)**

The idea behind the ‘No Duplication’ rule is the same idea that drives the DRY (Don’t Repeat Yourself) principle: “Every piece of knowledge should have one and only one representation.” Your code should have no duplicate knowledge. It sounds simple enough but Haines reminds us that we are not talking about the code itself but the information held within the code. So, when refactoring code to employ the “No Duplication” rule we must consider more than whether two lines of code are literally the same or not. Haines writes, “A good way to detect knowledge duplication is to ask what happens if we want to change something.” Will multiple places within our codebase require that change, and will these separate representations of information fall out of step with another when the codebase changes? If either answer is yes, then we are looking at knowledge duplication. 

To me, this was the most interesting section of Haines’ book as it brought into further relief my superficial understanding of the DRY principle. In my implementations of DRY, I’ve always looked for code duplication without giving any real consideration to what the code was doing. This actually came up during one of my interviews for this role. I was given a codebase that I had to refactor into cleaner code. I employed a naive version of “DRY” and immediately broke the code because I’d abstracted something that needed to be there to maintain the method’s functionality. 

**Small**

The “Small” rule dictates that our code should rid itself of anything that does not serve the first three rules. The code must have no superfluous parts. Haines recommends going over the final product to make sure that every bit of code is being used and that there aren’t any leftover duplicates. 
____________________________________________________________________________________________________________________________________________________________________

Overall, I found this to be an enlightening and instructive read. I am looking forward to putting the “Tests pass” and “No Duplication” rules into practice so as to gain a more nuanced understanding of how they work and interact with each other in a practical setting. 
