Idea to Support Left-Recursive PEGs
===================================


Notations and Conventions
-------------------------

### Grammars

PEG syntax as described in Bryan Ford's _Parsing Expression Grammars: A Recognition-Based Syntactic Foundation_.


### Data structures in memory

Javascript's object, array and string literal notation. Similar to JSON but, without having to quote the field names of objects.


### Abstract Syntax Trees (ASTs)

S-expression syntax. An AST is a list of a symbol and zero or more sub-trees. A sub-tree is an AST or a string.

e.g.

Parsing the string "abc" with a parser from the following grammar:

----------------

    A <- 'a' B 'c'

    B <- 'b'

----------------


would give the following AST:

----------------

    (A "a" (B "b") "c")

----------------


### Pseudo Code

Similar to Python syntax.



A Basic Setup
-------------

Let's assume we have a parser generator that, given a valid PEG, will produce a parser for the language described by the grammar. The parser produced is in the form of a data structure and, in combination with an evaluation function, can be used to parse an input string. The result of parsing an input string is either an AST or a parse error.


If the parser generator doesn't accept left-recursive PEGs, it might do the following steps:

* parse the PEG string
* transform the AST representation of the PEG into a parser data structure
* conservatively check for left-recursion in the data structure and reject if needed

Where the parser data structure is a list of non-terminals with their expression trees.

e.g.

Parsing and transforming this grammar:

----------------

    A <- 'a' / B

    B <- 'b'

----------------

we get this data structure:

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"char", data:"a"},
            {type:"non-term", data:"B"}]}},
     {name:"B", expression:{type:"char", data:"b"}}]

----------------



Dealing with Left-Recursive PEGs
--------------------------------

If the parser generator does handle left-recursive PEGs, it would do the following steps:

* parse the PEG string
* transform the AST representation of the PEG into a parser data structure
* transform the parser data structure into one that handles left-recursion


Instead of the previous non-terminals that had fields for name and expression, our new non-terminals also have a `type` field. The `type` field has four possible values, "normal", "seed", "plant" and "grower".


### Normal

This type represents non-terminals that are not involved in direct or indirect left-recursion.


### Seed

This type represents the part of a non-terminal that doesn't have left-recursive references.


### Plant

This type represents the part of a non-terminal that does have left-recursive references. The `expression` field does not contain left-recursive references but, implicitly, is a sequence of a left-recursive reference followed by the actual expression. The `arg` field holds the name of the implicit left-recursive reference. If it is direct left-recursion, the argument name will be the same as the name of the plant. If it is indirect, the argument name will be different.


### Grower

This type has one or more seeds. Each seed is associated with a list of one or more plants. There is also a list of indexes to plant lists. Indexes are included in this list if the first plant expects an argument of the same name as the grower.



Transforming to the New Data Structures
---------------------------------------

When the parser generator transforms the original data structure, it does a conservative check for left-recursion on each non-terminal. This check is conservative in the sense that, we assume the `&` and `!` operators will always succeed.


### No Left-Recursion

If no left-recursion is found for that non-terminal, it just copies the name and expression to create a non-terminal of type "normal".

e.g.

----------------

    A <- 'a' A / 'a'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence",
             data:[{type:"char", data:"a"}
                   {type:"non-term", data:"A"}]},
            {type:"char", data:"a"}]}}]

----------------

    [{type:"normal", name:"A", expression:{type:"choice",
      data:[{type:"sequence",
             data:[{type:"char", data:"a"}
                   {type:"normal", data:"A"}]},
            {type:"char", data:"a"}]}}]

----------------

If the non-terminal being processed is not left-recursive but, contains a reference to one that is, that reference is of type "grower".


e.g.

Assume `B` is left-recursive and does not refer to `A` directly or indirectly.

----------------

    A <- B

----------------

    [{name:"A", expression:{type:"non-term", data:"B"}}]

----------------

    [{type:"normal", name:"A", expression:{type:"grower", data:"B"}}]

----------------




### Direct Left-Recursion


If the non-terminal has direct left recursion, that non-terminal is split into three seperate non-terminals. These non-terminals are of type "seed", "plant" and "grower".

e.g.

----------------

    A <- A 'a' / 'a'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence",
             data:[{type:"non-term", data:"A"},
                   {type:"char", data:"a"}]},
            {type:"char", data:"a"}]}}]

----------------

    [{type:"seed", name:"A", expression:{type:"char", data:"a"}},
     {type:"plant", name:"A", arg:"A", expression:{type:"char", data:"a"}},
     {type:"grower", name:"A", seeds:["A"], plants:[[{name:"A", arg:"A"}]], indexes:[0]}]

----------------



### Mixed Direct Left-Recursion and Non-Left-Recursion

To get left-associative trees, non-left-recursive references to the non-terminal being transformed become type "seed".

e.g.

----------------

    A <- A '+' A / 'a'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence", data:[{type:"non-term", data:"A"},
                                    {type:"char", data:"+"},
                                    {type:"non-term", data:"A"}]},
            {type:"char", data:"a"}]}}]

----------------

    [{type:"seed", name:"A", expression:{type:"char", data:"a"}},
     {type:"plant", name:"A", arg:"A", expression:{type:"sequence",
      data:[{type:"char", data:"+"},
            {type:"seed", data:"A"}]}},
     {type:"grower", name:"A", seeds:["A"], plants:[[{name:"A", arg:"A"}]], indexes:[0]}]

----------------



### Indirect Left-Recursion


e.g.

----------------

    A <- B 'a' / 'a'

    B <- A 'b'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence", data:[{type:"non-term", data:"B"},
                                    {type:"char", data:"a"}]},
            {type:"char", data:"a"}]},

     {name:"B", expression:{type:"sequence",
      data:[{type:"non-term", data:"A"},
            {type:"char", data:"b"}]}}]

----------------

    [{type:"seed", name:"A", expression:{type:"char", data:"a"}},
     {type:"plant", name:"A", arg:"B", expression:{type:"char", data:"a"}},
     {type:"grower", name:"A", seeds:["A"], plants:[[{name:"B", arg:"A"}, {name:"A", arg:"B"}]],
      indexes:[0]},

     {type:"plant", name:"B", arg:"A", expression:{type:"char", data:"b"}},
     {type:"grower", name:"B", seeds:["A"], plants:[[{name:"B", arg:"A"}],
                                                    [{name:"A", arg:"B"}, {name:"B", arg:"A"}]],
      indexes:[1]}]

----------------


e.g.

----------------

    A <- B 'a' / 'a'
    B <- C 'b' / 'b'
    C <- A 'c' / 'c'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence", data:[{type:"non-term", data:"B"},
                                    {type:"char", data:"a"}]},
            {type:"char", data:"a"}]},

     {name:"B", expression:{type:"choice",
      data:[{type:"sequence", data:[{type:"non-term", data:"C"},
                                    {type:"char", data:"b"}]},
            {type:"char", data:"b"}]},

     {name:"C", expression:{type:"choice",
      data:[{type:"sequence", data:[{type:"non-term", data:"A"},
                                    {type:"char", data:"c"}]},
            {type:"char", data:"c"}]}]

----------------

    [{type:"seed", name:"A", expression:{type:"char", data:"a"}},
     {type:"plant", name:"A", arg:"B", expression:{type:"char", data:"a"}},
     {type:"grower", name:"A", seeds:["A", "B", "C"], plants:[[{name:"C", arg:"A"},
                                                               {name:"B", arg:"C"},
                                                               {name:"A", arg:"B"}],

                                                              [{name:"A", arg:"B"}],

                                                              [{name:"B", arg:"C"},
                                                               {name:"A", arg:"B"}]],
      indexes:[0]},

     {type:"seed", name:"B", expression:{type:"char", data:"c"}},
     {type:"plant", name:"B", arg:"C", expression:{type:"char", data:"b"}},
     {type:"grower", name:"B", seeds:["B", "C", "A"], plants:[[{name:"A", arg:"B"},
                                                               {name:"C", arg:"A"},
                                                               {name:"B", arg:"C"}],

                                                              [{name:"B", arg:"C"}],

                                                              [{name:"C", arg:"A"},
                                                               {name:"B", arg:"C"}]],
      indexes:[0]},

     {type:"seed", name:"C", expression:{type:"char", data:"c"}},
     {type:"plant", name:"C", arg:"A", expression:{type:"char", data:"c"}},
     {type:"grower", name:"C", seeds:["C", "A", "B"], plants:[[{name:"B", arg:"C"},
                                                               {name:"A", arg:"B"},
                                                               {name:"C", arg:"A"}],

                                                              [{name:"C", arg:"A"}],

                                                              [{name:"A", arg:"B"},
                                                               {name:"C", arg:"A"}]],
      indexes:[0]}]

----------------



### Mixed Indirect Left-Recursion and Non-Left-Recursion

Similar to direct left-recursion, all non-left-recursive references refer to the seed non-terminal.

#### Notes:

What if a non-terminal doesn't have a seed part?

e.g.

----------------

    A <- B 'a'

    B <- A

----------------



### Mixed Direct and Indirect Left-Recursion

Direct and indirect left-recursion can be within the same non-terminal without problems. The order of the recursive references and, therefore, the order of the plants, determine whether direct or indirect recursion will be tried first.

e.g.

----------------

    A <- A 'a' / B 'a' / 'a'

    B <- A 'b'

----------------


e.g.

----------------

    A <- B 'a'

    # Here the direct left-recursion is not on the start non-terminal.
    # Because of this, the grower would need to keep evaluating the same plant
    # as long as the name of the current plant is the same as the expected arg.
    B <- B 'b' / A 'b' / 'b'

----------------





### Plant Choices Prefixed with Non-Consuming Expressions

e.g.

----------------

    A <- !'aaab' A / 'a'

----------------

#### Notes:

Perhaps, the plant non-terminal type could have a field for a non-consuming prefix expression. The grower would check that the prefix expression successfully evaluates from the input position the grower started at before it tried to evaluate the plant.

Not sure how one would handle recursive references within the non-consuming expression.




Order of Seed and Plant Choices
-------------------------------

The ordered choice property stays the same between seeds and between plants.


e.g.

An example where ordering has no impact.

----------------

    A <- A 'a' / 'a'

----------------

    A <- 'a' / A 'a'

----------------


e.g.

The first seed in the grammar is the first one that gets tried.

----------------

    A <- 'a' / [0-9a-z]+ / A 'a'

----------------


e.g.

----------------

    A <- A 'a' / B 'a' / 'a'

    B <- A 'a'

----------------

Using 'A' to parse "aaa" will give `(A (A (A "a") "a") "a")`, not `(A (B (A "a") "a") "a")`, because `A 'a'` is ordered before `B 'a'`.




The Evaluation Function
-----------------------

The original evaluation function implemented the semantics of non-left-recursive PEGs. To handle the new non-terminal expression types, we need to create a new evaluation function. Our new evaluation function is the same as the original except that, where the original had a single rule for evaluating a non-terminal, the new function has four.

### Normal

The same as a non-terminal in the original evaluation function.


### Seed

The same as a non-terminal in the original evaluation function.


### Plant

The "plant" type is only evaluated from within an evaluation of a "grower". The rule to evaluate a "plant" non-terminal is supplied an additional argument. The additional argument is the result of evaluating the left-recursive reference that is implicitly part of the plant non-terminal's expression.

The expression of the plant non-terminal is evaluated and combined with the argument. The combined result is returned as the result of evaluating the plant non-terminal.

e.g.

----------------

    A <- A 'a' / 'a'

----------------

    [{name:"A", expression:{type:"choice",
      data:[{type:"sequence",
             data:[{type:"non-term", data:"A"},
                   {type:"char", data:"a"}]},
            {type:"char", data:"a"}]}]

----------------

    [{type:"seed", name:"A", expression:{type:"char", data:"a"}},
     {type:"plant", name:"A", arg:"A", expression:{type:"char", data:"a"}},
     {type:"grower", name:"A", seeds:["A"], plants:[[{name:"A", arg:"A"}]], indexes:[0]}]

----------------

Here our "plant" has an implicit reference to an `A` non-terminal. When we evaluate our "plant", we supply an argument that was the result of evaluating an `A` non-terminal.

If that argument is an AST `(A "a")` and the next character in the input string is "a", the result of evaluating our "plant" will be `(A (A "a") "a")`.


### Grower

When a grower non-terminal is evaluated, each seed of the grower is evaluated, in order, until the result of evaluating a seed is successful.

If the evaluation of each seed fails, the result of evaluating the grower is also failure.

If a seed is successfully evaluated, the grower starts evaluating its plants. During this stage, the grower keeps track of the last result that could be considered a successful evaluation of the grower non-terminal. Initially, this is the result of evaluating a seed. Later, it could be the result of evaluating one or more lists of plants.

After successfully evaluating a seed, the list of plants associated with the seed is used as the current plant list.

The grower supplies the result of the seed as the argument to the first plant in the list.

If that plant is successful, the result is given to the next plant in the list until the list is empty.

If all plants in the list have successfully been evaluated and the end of the list is reached, the result of the last plant is a possible result of the grower. The result is then used as the argument to evaluate more plant lists. All plant lists that start with a plant that expects an argument of the same name as the grower will be evaluated.

If evaluating a plant in the current plant list failed, the whole plant list fails. The result of the grower is either a previous possible result or, failure.



----------------

    # steps to evaluate a plant non-terminal
    def eval_grower(grower):
      result = FAILURE

      seed_index = 0
      for seed in grower.seeds:
        s = eval_seed(seed)
        if s != FAILURE:
          result = s
          break
        seed_index += 1

      if result == FAILURE:
        return result

      # evaluate the initial plant list
      plant_list = grower.plants[seed_index]
      p = eval_plant_list(plant_list, result)

      # evaluate the secondary plant lists
      while p != FAILURE:
        result = p
        p = FAILURE

        # evaluate plant lists until one succeeds
        for index in grower.indexes:
          plant_list = grower.plants[index]
          p = eval_plant_list(plant_list, result)
          if p != FAILURE:
            break

      return result


    def eval_plant_list(plant_list, arg):
      p = FAILURE
      for plant int plant_list:
        p = eval_plant(plant, arg)
        if p == FAILURE:
          return FAILURE
        else:
          arg = p
      return p

----------------




The Semantics of Left-Recursive PEGs
------------------------------------

For PEGs without left-recursion, the semantics are unchanged.

PEGs with left-recursion would not be considered ill-formed. Instead, a left-recursive PEG would have the semantics that it describes the language matched by the parser generated from the above process.

There are some cases where a left-recursive grammar may not make sense.

e.g.

A grammar that produces growers with plants but, no seed non-terminals.

----------------

    A <- A 'a'

----------------

    B <- C 'b'

    C <- B 'c'

----------------


e.g.

A grammar that produces a plant non-terminal with an empty `expression`.

----------------

    A <- A / 'a'

----------------

    B <- C / 'b'

    C <- B

----------------

In this case, the plant of 'B' could be ignored and the parser would still match the same language.

Parsing the string "b" would give `(B "b")` from the `B` non-terminal and `(C (B "b"))` from `C`.




Memoization and Packrat Parsing
-------------------------------

The method, as presented above, does not rely on any form of memoization. Because of this, it can be used in a PEG-based system that does not use packrat parsers.

However, if we want to ensure a linear run-time, we can use memoization in a way similar to traditional packrat parsing.

As with packrat parsing, we memoize the results of evaluating parsing expression. The difference, compared to traditional packrat parsing, is that a non-terminal expression can be one of four possible types. Three of the non-terminal types can be memoized and retrieved in the same way as traditional packrat parsing.

For each of the types "normal", "seed" and "grower", the result of evaluating the expression is memoized against a key. The key is the string input position before evaluation and a unique identifier for the expression. This unique identifier must take into account both the name of the non-terminal and its type.

When the evaluation function evaluates one of the these expressions, it first checks that a memoized result for the expression and input position do not exist. If a memoized result exists, that result is returned. Otherwise, the non-terminal is evaluated in full.

For the "plant" non-terminal type, the result of evaluating the expression is memoized but, the argument that was supplied by the grower is not memoized. This is because the same plant expression can be evaluated at the same position but, with a different argument. The argument that is given to a plant non-terminal will depend on where the grower is evaluated.

When the evaluation function retrieves the memoized plant result, it combines the argument used in the current evaluation with the retrieved result. The combined result is returned by the evaluation function.



License
-------

Copyright &copy; 2010 Orlando Hill

This work is licensed under a [Creative Commons Attribution-ShareAlike 3.0 Unported License](http://creativecommons.org/licenses/by-sa/3.0/).
