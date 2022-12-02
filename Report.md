# Databases Project Report

> Master 2 Data Science | Gabriel Loiseau
> 

## 1 Project Description

The main goal of the project is to implement in python a solution to simulate First Order Logic operations and Non-Recursive Datalog operations, especially using visitor pattern. We will also implement to those the concept of Range Restriction and Free Variables.

## 2 Project Guide

The project is decomposed in 2 main folders and a markdown file (this report). 

- data : stores the data files for the projects, i.e the .txt files for Actors, Artists...
- src : stores the python files for the project

The python files starting with :

- "Datalog" are related to Datalog implementations
- "FOL" are related to First Order Logic implementations
- "test" are related to some unit tests for the project

## 3 Project Overview

In this part we will review each file of the project in chronological order in order to explain the decisions we made :

### main.py

This is used as a preliminary for the project, it aims to represent First Order Logic in Python without using Visitor Pattern, here we are using lambda function to represent FOL queries, for example the query :

> *Each artist is either an actor or a film director*
> 

Can be represented, in FOL, as :

$\forall x, \, Artist(x) \wedge(Actor(x) \, \vee \, FilmDirector(x)) \, \vee \lnot Artist(x)$

this can be interpreted as "Each entity of the domain which is an Artist is also an Actor or a Film Director"

In the programs this is represented with this lambda function :

```python
forall(lambda x: artist(x) and (actor(x) or filmDirector(x)) or not artist(x))
```

Remark : This file can be run to get the result of different queries we saw during the course.

### FOL_Objects.py

This file is dedicated to define the python classes we will use to represent FOL.

The goal here is to use the Visitor Design Pattern to compute recursively the result of a FOL formula. We will use Python classes to represent Predicates, Operators (like negation), Quantifiers and Primitive Objects (here Constant and Variables), due to visitor pattern, each class will have a similar structure like so :

- Class Name
- Constructor
- .accept() method
- string representation

n-ary predicates will have their n terms declared in the constructor, the same goes with operators with their formula(s). Concerning quantifier (ForAll and Exists) , we use this classic representation :

$\forall x, \phi$   where x is a variable and phi a formula is represented as :

```python
ForAll("x",phi)
```

In this case we will not specify that x is a variable (by writing Variable("x") instead of "x").

We also defined an abstract class Formula so we can use inheritance to force a subclass to implement a .accept() method.

### FOL_VisitFOL.py

This file is used to define an abstract class so any visitor aiming to visit our FOL objects will have to redefine all the methods defined there

### FOL_ComputeFOL.py

This defines our visitor to compute FOL queries using visitor pattern, first our Visitor will consider what we call a model, so the Visitor will act the same even if we will have different data in the future. Here we represent the model as a Python dictionary where keys are the "table names" for our database, and their values are the .txt files contained in ./data. 

Here we use a specific function to import the files, in fact the """literal_eval(str)""" function will convert a string into a proper python array. So when we read the file we obtain a string representing an array and this function converts it properly :

Example : literal_eval("['a','b','c','d']") = ['a','b','c','d']

The Visitor also introduce an environment in its attributes that we will present later.

- Visiting predicates and operators :

Here we use the Visitor pattern to visit recursively the objects of our query. So our last objects to visit are the predicates, and using the model our visitor will compute if a specific value is in the good table.

- Visiting primitive types and quantifiers :

First visiting a Constant will always return its string value. Example :

```python
Artist(Constant('Wayne John')).accept(ComputeFOL())
```

Will check if the string 'Wayne John' is contained in the table Artist.

Variables will be used to compute quantifiers, let us consider the visit_forall(self,e) method :

```python
def visit_forall(self,e):
        for value in self.model["DOMAIN"]:
            # We consider that e.value is always a variable
            if e.value in self.env.keys():
                self.env[e.value].append(value)
            else:
                self.env[e.value] = [value]
            if not e.formula.accept(self):
                self.env[e.value].pop()
                return False
        self.env[e.value].pop()
        return True
```

When we say "ForAll x Artist(Variable(x))" we want to check if any entity in the Domain is an Artist.

Here we will use the environment (env) to store each possible value of the Domain. 

At the beginning, the environment will be an empty dictionary, when we encounter a quantifier, it will fill the dictionary where the key will be the name of the variable and the value will be an empty list. So after visiting a quantifier the environment will look like :

env = {

"x" :  [ ]

}

For each value in the domain, this value will be appenned to the according list with respect to the variable name, so when we will visit recursively Variable :

```python
def visit_variable(self,e):
        return self.env[e.value][-1]
```

The visitor will return the last value appended to the environment linked to the variable name.

This has been done so if we have the situation :

ForAll x ForAll x ForAll x Actor(Variable(x)) 

(Which is writable since ForAll takes a Formula as a second argument and is itself a Formula)

Is computable, and the environement will look like :

env = {

"x" :  [ first_forall_values, second_forall_values , third_forall_values  ]

}

We will hence check all the values of the last ForAll, then the second, then the first.

Since it is a ForAll, as soon as a value is not computed as False, we will stop the loop and return False (poping the last value of the array so we can process other quantifiers with the same variable name) but the same principle is used for the Exists quantifier.

### FOL_RRI.py

In this file we compute the range restricted variables using the table we saw in the course.

Here we use union of sets to construct recursively the set of range restricted variables. The most important thing to mention is that negation returns an empty set as it creates an unbounded formula. Also encountering a ForAll raises an exception but the Exists formula raises an exception only if its variable cannot be removed from the set of range restricted variables of its sub-formula.

### FOL_VisitFV.py

In this file we define a visitor to compute the set of free variables of a formula, we follow this table :

$$FV(\phi_1 \wedge \phi_2) = FV(\phi_1) \vee FV(\phi_2)$$
$$FV(\phi_1 \vee \phi_2) = FV(\phi_1) \vee FV(\phi_2)$$
$$FV(\lnot \phi) = FV(\phi)$$
$$FV(\forall x, \phi) = FV(\phi) - {x}$$
$$FV(\exists x, \phi) = FV(\phi) - {x}$$
$$FV(x) = x$$
$$FV(Constant) = \emptyset$$
Using the same principe as before with the union of sets, we can construct recursively our set of Free Variables.

### FOL_RemoveForAll.py

In this file we define a visitor to remove every ForAll in a FOL formula. 

The only thing we have to do is to iterate recursively on the formula and replace each ForAll encountered by a Not(Exists(e.value,Not(e.formula))). Moreover will will use another visitor in order to push the negations in a formula.

### FOL_PushNegation.py

This file defines the PushNegation visitor to simplify a formula by pushing as far as possible negations.

In order to do this, we will use the **kwargs magic variable in python.

We create an optional argument in the visitor methods to represent the "polarity" of a formula. Let us take the visit_neg(*self*,*e*,***kwargs*) method :

```python
def visit_neg(self,e,**kwargs):
        if 'negation' in kwargs.keys():
            updated_negation = not kwargs['negation']
        else:
            updated_negation = True
        return e.formula.accept(self,negation=updated_negation)
```

if we encounter a negation, checking if 'negation' is in kwargs[] refers as checking the polarity of the formula. the first time we encounter a negation we set the polarity to True (True is the formula as to be negated, False otherwise).

If we encounter another negation, it will reverse the polarity of the formula.

When we encounter a predicate, we are at the end of our iteration so we can put the negation here if the polarity is True.

Considering quantifiers and the And/Or operators, we just have to check to polarity of our formula (and also if we have already encountered a negation) to apply or not Morgan formulas. We just have to stop pushing the negation if we encounter an Exists quantifier with a True polarity.

### Datalog_Objects.py

We represent Datalog programs as a class with a set of Datalog rules as its attributes. The reason we are doing this is so we can implement a visitor function for this objects instead of creating a function with raw list type.

A Datalog Rule is defined with its id, its terms, and with optional arguments related to its body. For example a Datalog Rule list with this form :

- p1(x,y) :- a(x), p2(x,y).
- p2(x,y) :- b(x,y), c(y).

Will be represented with a list of these two python variables :

- p2 = Datalog_Rule(2,(x,y),b(x,y), c(y))
- p1 = Datalog_Rule(1,(x,y),a(x), p2)

For instance the body of a Datalog Rule will accept Datalog_Rule classes but also the predicates we created in First Order Logic; so we can directly overwrite their visit method to be used in the Datalog setting.

### Datalog_VisitDL.py

This file is used to define an abstract class so any visitor aiming to visit our Datalog objects will have to redefine all the methods defined here.

### FOL_to_Datalog.py

In this file we create a visitor to convert any FOL formula (preprocessed with removeForAll and PushNegation) into a working datalog program.

Here we will use once again the **kwargs magic variable with the optionnal parameters "DL_id" and "DL_terms" these will help us to keep track of the current id and terms of our datalog rules.

Using the same notation seen in the course using a tree indexing rules with "0" when the tree separates to the left and "1" when the tree goes to the right, we can create multiple rules using these indexes. 

Let us start with the visit_actor method as an example :

 

```python
def visit_actor(self,e,**kwargs):
        if 'DL_id' in kwargs.keys() and "DL_terms" in kwargs.keys():
            return Datalog_Rule(kwargs["DL_id"], kwargs["DL_terms"], e)
        return Datalog_Rule("0", e.accept(self), e)
```

If we receive the optional "DL_id" and "DL_terms" as an input, this means that there is at least one object above our predicate. So we convert it as a rule with its ID and Terms.

If there is no such argument this means we are on the top of the tree and we can just instantiate the rule using the "0" index and the good terms.

Here the Exists quantifier will have the role of creating the top rules and sending the right terms: 

```python
def visit_exists(self,e,**kwargs):
        if 'DL_id' in kwargs.keys() and "DL_terms" in kwargs.keys():
            return Datalog_Rule(kwargs["DL_id"], kwargs["DL_terms"].append(e.value), e.formula.accept(self))
        return Datalog_Rule("0",(e.value),e.formula.accept(self,DL_id="00",DL_terms=(e.value)))
```

Following what we have said before, if we start our Formula by this quantifier it will instantiate a new Datalog Rule but this time it will take as a term the variable inside the quantifier. 

With binary operators like OR or AND, we apply the same principle but this time we append a "0" to the id of the left subformula and a "1" to the id of the right subformula.

We can resume our approach with some examples :

FOL formula :

Exists('x',And(Actor(Variable('x')),Artist(Variable('y'))))

Datalog result from our code :

 p0x :- p00x :- p000x :- Actor(Var(x)),p001x :- Artist(Var(y))

- p0x is created from the Exists quantifier
- p00x is created from the And operator
- p000x is created from the left part of the And operator (the Actor predicate)
- p001x is created from the right part (the Artist predicate)

One issue with this implementation is the consequence it has when we are using OR operators. Example with our above formula :

FOL formula :

Exists('x',Or(Actor(Variable('x')),Artist(Variable('y'))))

Datalog result from our code :

 p0x :- (
     p00x :- p000x :- Actor(Var(x)), 
     p00x :- p001x :- Artist(Var(y)
     )

- p0x is created from the Exists quantifier
- p00x is created from the Or operator
- p000x is created from the left part of the And operator (the Actor predicate)
- p001x is created from the right part (the Artist predicate)

This works because there are two rules with the same ID but in more complex formulas the fact that everything holds in only one python objects is not the best option to represent Datalog programs.

### Datalog_Test_RRI.py

We define a visitor to check if a Datalog Program is range restricted.

To recall : a Datalog rule is RRI if every variable in the head is in the body, and every variable in the body has a positive occurrence. So we just have to check those two conditions. (A Datalog Program will be range restricted if all of its rule is).

To check if every variable in the head is in the body of the rule. We just visit every element in the body in order to get every variable (with a positive occurrence). And after that we just check if every variable in the head of the rule is contained in the list of body variables.

We deal with positive occurrence by neglect variables within a not operator. So we need at least one positive occurrence of the variable so the Rule can be range restricted.

### Datalog_CompileDL.py

Here we create a visitor to check if a Datalog program will return a null value (or will exceture properly).  To do this we check if the visitor is able to visit every part of every rule of the datalog program. Since we used FOL objects in our datalog rule, we overwrite the visit method on unwanted objects (Like AND or EXISTS) so they will raise an error when they are visited by this visitor.

### Datalog_ComputeDL.py

In this file we define a possible visitor to execute Datalog Programs (also works with Datalog rules since evaluating Datalog programs is the same as evaluating each of its rule and returning the union of the result).

We will use the same model idea as we did with First Order Logic. Here, the model acts like extensional  predicates (it can be unary (like artist) or binary predicates (like acts)) Also to implement the idea that a Datalog program can return a combination of possible variable output, we will use what we call in the project an "index_table". It will take the form of a Python dictionary and will associate to every possible variable name within a Datalog program every possible output of this variable. An index table like this :

{

"x" : [ 1, 2, 3],

"y" : [4,5]

}

Will represent every possible combination of x*y = {(1,4) (1,5) (2,4) (2,5) (3,4) (3,5)} 

Now everything that the visitor will do is to check :

- If the index table for a fixed variable is empty, we put every possible value of the table in the list.
- If the index table for this fixed variable contains values, then we take the intersection of these values (to express the And operator in Datalog).

Remark : here we don't take into account constant since it will require to adapt our code if the formula contains only variables, only constants, of both.

Also here the Not operator will act in order to return everything  that is not in its subformula, we implemented it by returning the difference between the entire domain and the result of the subformula. 

### test_FOL.py and test_Datalog.py

These file are used for debug and unit tests, they mainly rely on the implementation of the string method we can also use the isInstance() function of Python to assert variable types.

## 4 Conclusion / Further work

Some possible enhancement for the project :

- Find a way to deal with constant during execution in Datalog, so we would be able to return boolean queries
- Implement unfolding visitor for Datalog
- Use another way to transform FOL into Datalog since using only one python variable non-unfolded is not the best option for memory and general usage.
- Go deeper into Datalog compilation, se we can check if the result of the program is null without checking the content of tables.