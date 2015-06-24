---
layout: model
title: Syllogisms
model-status: code
model-category: Reasoning about Reasoning
model-tags: reasoning, pragmatics, QUD
model-language: church
---

A model of syllogistic reasoning as communication.

The reasoner imagines an experimenter who chooses an argument (i.e. premises) conditioned on the reasoner inferring an intended conclusion.

A syllogism is a two-sentence (premise) argument. Each sentence consists of 2 terms, and between the two sentences, 1 of the terms is shared.
For example,

> B - A
>
> C - B

In this argument, B is the shared term. The task for the reasoner is to generate a conclusion relating the "end-terms"-- A & C.

The relations between the terms are quantifiers. 
In classical syllogisms, the quantifiers are those from Aristotle's "square of opposition": {all, some, none, not-all}.

For example,

> No B are A
>
> All C are B
>
> {All, Some, No, Not-all} C are A

In this model, the reasoner imagines these sentences apply to concrete situations which are composed of objects with properties. Sentences (premises or conclusion) are then either true or false of a given situation.

Note: The model presented below is for transparency and/or pedagogical purposes. 
The program itself is computationally expensive. The state space grows exponentially with the number-of-objects parameter.
However, we are interested not in the distribution of objects per se, but in the distribution of true-sentences that those objects imply.
As such, we can derive an equivalence-class representation of the situation space (objects comprise situations), which is projection of the distribution of situations onto the sentences of interest. 
The equivalence-class of situations does not grow with the number of objects and is much faster to run. However, the number-of-objects and base-rate parameters cannot be changed inside the model.
The version of that model (with best-fit parameters n_objects = 5, br = 0.25 used in Ref:tessler2014syllogisms) can be found [here](http://forestdb.org/models/syllogisms-equivalence-cogsci14.html).



    (define all-true (lambda (lst) (apply and lst)))
    (define some-true (lambda (lst) (apply or lst)))
    
    ; assume the situations constructed have at least one object with each of the properties
    (define existential-import (lambda (A B objects) 
                                 (and (some-true (map A objects)) (some-true (map B objects)))))
    
    (define all (lambda (A B)
                  (if (existential-import A B objects)
                      (all-true (map (lambda (x) (if (A x) (B x) true)) 
                                     objects))
                      false)))
    
    (define some (lambda (A B)
                   (if (existential-import A B objects)
                       (some-true (map (lambda (x) (if (A x) (B x) false)) 
                                       objects))
                       false)))
    
    (define none (lambda (A B)
                   (if (existential-import A B objects)
                       (all-true (map (lambda (x) (if (A x) (not (B x)) true)) 
                                      objects))
                       false)))
    
    (define not-all (lambda (A B)
                     (if (existential-import A B objects)
                         (some-true (map (lambda (x) (if (A x) (not (B x)) false)) 
                                         objects))
                         false)))
    
    
    (define (raise-to-power dist alph)
      (list (first dist) (map (lambda (x) (pow x alph)) (second dist))))
    
    
    ; pass strings, which then call functions of the same name
    (define (meaning word)
      (case word
            (('all) all)
            (('some) some)
            (('not-all) not-all)
            (('none) none)))
    
    ; the reasoner has uninformative prior beliefs about what the conclusion should be
    (define (conclusion-prior) (uniform-draw (list 'all 'some 'not-all 'none)))
    ; rhe reasoner has uninformative prior beliefs about what arguments the experimenter could give
    (define (premise-prior) (uniform-draw (list 'all 'some 'not-all 'none)))
    
    ; parameter 1: the reasoner's prior beliefs about the rarity of properties
    (define br 0.25)
    ; parameter 2: number of objects in the situation the reasoner imagines
    (define objects (list 'o1 'o2 'o3))
    
    (define argument-strength 
      (mem
      ; argument-strength is a function of the argument's premises
       (lambda (premise-one premise-two)
         (enumeration-query
          ; properties map objects to truth-values i.e. whether or not the object has the property
          ; together with the list of objects, these represent situations over which reasoning occurs
          (define A (mem (lambda (x) (flip br))))
          (define B (mem (lambda (x) (flip br))))
          (define C (mem (lambda (x) (flip br))))
          ; the reasoner is also uncertain about what conclusion is true
          (define conclusion (conclusion-prior))
          
          ; what do we want to know? what is the conclusion.
          conclusion
          
          ; condition on the syllogism applying to the situation
          ; notice the form of the syllogism present in this part of the query
          (and
            ; the premise quantifiers apply to: A-B & B-C
                ((meaning premise-one) A B)
                ((meaning premise-two) B C)
            ; the conclusion quantifier is true of terms: A & C
                ((meaning conclusion) A C)
           )))))
    
    
    (define experimenter
      (mem
      ; the experimenter takes in the conclusion as an argument (i.e. he has a conclusion in mind)
       (lambda (conclusion)
         (enumeration-query
          ; the experimenter draws premises from the premise-prior (uniform) distribution
          (define premise-one (premise-prior))
          (define premise-two (premise-prior))
          
          ; the experimenter produces two premises
          (list premise-one premise-two)
          
          ; the experimenter wants the reasoner to draw a particular conclusion, gives the premises 
          ; parameter 3: "optimality" -- the degree to which the experimenter's argument is optimal for the conclusion
         (equal? conclusion (apply multinomial (raise-to-power (argument-strength premise-one premise-two) 4.75)))))))
    
    (define pragmatic-reasoner 
      (mem
      ; the reasoner takes in two premises as arguments
       (lambda (premise-one premise-two)
         (enumeration-query
          ; the pragmatic reasoner has the exact same generative model as the argument strength model
          (define A (mem (lambda (x) (flip br))))
          (define B (mem (lambda (x) (flip br))))
          (define C (mem (lambda (x) (flip br))))
          (define conclusion (conclusion-prior))
          
          ; the reasoner produces a conclusion
          conclusion
          
          (and
          ; the conclusion quantifier is true of terms: A & C
           ((meaning conclusion) A C)
          ; the premises now serve a different role; the premises are imagined to come from an experimenter
          ; who has a particular conclusion in mind
            (equal? (list premise-one premise-two) (apply multinomial (experimenter conclusion))))
           ))))
    
    
    ; All A are B
    ; No B are C [cogsci paper, Fig 2 [1]]
    
    ;(pragmatic-reasoner 'all 'none)
    (argument-strength 'all 'none)
        
    ; this is a valid syllogism with 2 valid conclusions
    ; using the argument-strength alone produces no preference among valid conclusions
    ; the pragmatic reasonser draws the conclusion not only true of the situation
    ; but also which was intended to be drawn, inferring that "None" is the more likely intended conclusion



References:

- Cite:tessler2014syllogisms

