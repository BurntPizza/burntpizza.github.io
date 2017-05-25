---
layout: post
title:  "Query me this"
---
<style>
code {
  display: inline-block;
}
span[ib] {
  display: inline-block;
  width: 100px;
}
[serif] {
  font-family: serif;
}
[pseudo-code] {
  font-family: monospace;
}
[indent] {
  text-indent: 20px;
}
[italic] {
  font-style: italic;
}
</style>


![Diagram]({{ site.github.url }}/assets/diagram_faded.png)  
  
{::options parse_block_html="true" /}
<div serif>
## Query  
  
<div style="color: #555555">
*verb*  <span style="font-size:150%">|</span>  \ˈkwir-ē\
1. to ask questions of especially with a desire for authoritative information
</div>
</div>
-----  
<br>
Yesterday [Logan](https://github.com/loganmhb) and I pair programmed on his Datomic-inspired [logos](https://github.com/loganmhb/logos) database, implementing query resolution with a backtracking query engine.  
  
At least that's the short way to put it. The long way, the detailed way, the *chronicle of prolonged mental suffering*, well that's what this post is.  
  
# Introduction  
  
The part of our work I will focus on is the query engine, as that was the lion's share of what we did today.  
First, some terms:  
  
<div serif>
* <span ib>Query</span>a question being asked
* <span ib>Database</span>a source of facts  
</div>
  
A query engine is a function that processes a query given a database and yields the answer to the asked question.  
In our case, a query takes the form:  
  
```prolog
find <variables> where <clauses>
```
  
Our example query will be the following:
```prolog
find ?b where (?a name "Bob")
              (?b parent ?a)
```  
  
And a clause is of the form `(entity attribute value)`, which is identical to our representation of facts in the database, except that clauses are *patterns* for facts, and can contain variables, such as in the example query.
  
What the example query is asking is to find *any value(s)* (which we are giving the name "b") that are entities defined to have an attribute `parent` that has *any value* (which we are calling "a") that in turn has the `name` attribute of `"Bob"`. All that is described in only two clauses! Whew!


# The Engine of Knowledge  
  
Alright, so given a database of facts such as the following, how is our query answered?  

```prolog
(0 name "Bob")
(1 name "John")
(1 parent 0)
```
<span style="font-size:75%">*(Note: here entities are designated by numbers)*</span>  

Our first attempt went something like the following:  
```ruby
results = []

foreach fact in database:
    discovered = {}
    
    foreach clause in query:
        if clause.entity is a literal and does not equal fact.entity:
            continue to next fact
        if clause.entity is a variable:
            add (clause.entity.variable is equal to fact.entity) to discovered

        if clause.attribute is a literal and does not equal fact.attribute:
            continue to next fact
        if clause.attribute is a variable:
            add (clause.attribute.variable is equal to fact.attribute) to discovered

        if clause.value is a literal and does not equal fact.value:
            continue to next fact
        if clause.value is a variable:
            add (clause.value.variable is equal to fact.value) to discovered

    add discovered to results

return results
```
In this way, we look through everything and bind all of the variables to what they match in the database, and throw out any candidate results that contain anything that doesn't match.  
  
And this works for simple queries like `find ?a where (?a name "Bob")` (get the entities with the name Bob) or even `find ?a ?b where (?a name ?b)` (get the entities that have a "name" attribute and the values of that attribute), like we had in our initial tests!  
  
However, this method can't even *begin* to correctly satisfy our more complex example query!  
Currently the algorithm can only extract information from a single fact per solution, but our query requires multiple facts to be considered and used to contrain the results *simultaneously*!  
How can this be done? I'll give you a minute.  
  
Brain hurting yet? Here's an answer: backtracking.  
  
# The Engine of Insight  

According to [Wikipedia](https://en.wikipedia.org/wiki/Backtracking):  
  
>Backtracking is a general algorithm for finding all (or some) solutions to some computational problems, notably constraint satisfaction problems, that incrementally builds candidates to the solutions, and abandons each partial candidate c ("backtracks") as soon as it determines that c cannot possibly be completed to a valid solution.  
  
Note, that unlike what our algorithm above, where we totally abandon the current candidate when a clause doesn't match, the backtracking algorithm only abandons the current *partial candidate*, i.e. it keeps trying different options, only restarting from scratch when all possible solutions based on current knowledge have been explored.  
Or in other words: any solution it finds will be considering and utilizing *however many facts it needs, simultaneously!* Ta-da!  
  
The code we came up goes like this:  
```ruby
initial_state = { discovered = {}, clause_idx = 0, fact_idx = 0 }
results = []
stack = [initial_state]

while stack is not empty:
    state = stack.pop()
    current_clause = null
    current_fact = null     

    if state.clause_idx >= query.clauses.len():
        # all clauses have succeeded, add current knowledge to results
        add state.discovered to results
        continue
    else:
        current_clause = query.clauses[clause_idx]
    
    if state.fact_idx >= database.facts.len():
        continue
    else:
        current_fact = database.facts[state.fact_idx]

    # unify() is essentially the same as the body of the 
    # inner loop in the old code above
    match unify(state.discovered, current_clause, current_fact):
        Success(newly_discovered):
            # Move on to next fact
            increment state.fact_idx
            push state onto stack

            # Investigate the next clause with new info
            new_state = {
                          discovered = state.discovered + newly_discovered, 
                          fact_idx = 0, 
                          clause_idx = state.clause_idx + 1 
                        }
            push new_state onto stack
        Failure:
            # Nothing new was learned, so just move on to next fact
            increment state.fact_idx
            push state onto stack

return results
```
  
The real code with details is [here](https://github.com/loganmhb/logos/blob/fdccdfaaa6b953081b91012f378120acc6e85758/src/lib.rs#L148-L206).