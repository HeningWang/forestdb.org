---
layout: model
title: Bayesian Data Analysis
---

By: Michael Henry Tessler and Noah D. Goodman

In this book we are primarily concerned with probabilistic models of cognition: understanding inferences that people draw as Bayesian conditioning given a generative model that captures a persons models of the world. Bayesian statistics are equally useful to us as scientists, when we are trying to understand what our data means about psychological hypotheses. This can become confusing: a particular modeling assumption can be something we hypothesize that people assume about the world, or can be something that we as scientists want to assume (but don't assume that people assume). A pithy way of saying this is that we can make assumptions about "Bayes in the head" or about "Bayes in the notebook". We will illustrate by considering cognitive models of randomness judgements, as explored in Chapter 5.

Imagine that you are asked to judge whether a sequence of coin flips came from a fair coin or a trick (weighted) coin. The sequence "TTTTT" probably strikes you as almost certainly trick, while the sequence "THHTH" probably strikes you as likely coming from a fair coin. The sequence "HHHTT" may seem more ambiguous.
Cognitive scientists like to confirm such intuitions, and explore the borderline cases in a quantitative way. So we did an experiment asking 30 participants to make this judgement for a number of different sequences. You can do this experiment yourself [here](http://stanford.edu/~mtessler/experiments/subjective-randomness/experiment/experiment.html) (your data won't be recorded).

In Chapter 5 we discussed several models of how people might make this judgement. First we'll recall the simplest such model, then we'll think about how to compare it to the data we have gathered.

#A simple model of randomness judgements

The simplest model assumes that the sequence comes from a fair coin (with heads probability 0.5) or a biased coin whose probability of landing Heads (the `bias-weight`) is a parameter of the model.

~~~~
(define (biascoin-model sequence bias-weight)
  (enumeration-query

   (define fair-weight 0.5)

   (define isfair (flip))

   (define the-weight (if isfair fair-weight bias-weight))

   (define coin (lambda () 
                  (flip the-weight)))

   isfair

   (equal? sequence (repeat 5 coin))))

(barplot (biascoin-model (list false false false false true) 0.2) "TTTTH is fair?")

(barplot (biascoin-model (list false true true false true) 0.2) "THHTH is fair?")
~~~~

We can gain some intuition for the possible predictions of this model by exploring different bias parameters.
To simplify the results, we make a function `get-probability-of-faircoin` to extract the `#t` probability (i.e. the probability that the sequence came from a fair coin).

~~~
;;;fold:
(define (biascoin-model sequence bias-weight)
  (enumeration-query

   (define fair-weight 0.5)

   (define isfair (flip))

   (define the-weight (if isfair fair-weight bias-weight))

   (define coin (lambda () 
                  (flip the-weight)))


   isfair

   (equal? sequence (repeat 5 coin))))
;;;

; takes in "dist": output from an enumeration-query
; and "selection": the element from the posterior that you want
; returns the probability of that selection
(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
      (list-ref (second dist) index)))

(define many-biases (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9))

(define results-for-many-biases 
  (map 
   (lambda (bias-weight) 
     (get-probability-of-faircoin 
      (biascoin-model (list false false false false false) bias-weight)
      #t))
   many-biases))

(barplot (list many-biases results-for-many-biases) 
         "TTTTT is fair?, by bias-weight parameter")
~~~
      
We see that for lower values of `bias-weight`, we get the intuitive inference: TTTTT is not from a fair coin. The converse way of putting this is, if we imagine trying to enforce the judgement that TTTTT comes from a biased coin, we will be forced to assume the `bias-weight` is low. 

# Inferring model parameters

How can we ask more rigorously what the right value of `bias-weight` is, given the data we have collected? 
A standard method would be to optimize the fit (e.g. correlation) of data to model predictions, by adjusting the `bias-weight` parameter.
Another (more Bayesian) way to approach this is to say we (as scientists) have uncertainty about the true value of `bias-weight`, but we believe the data is the result of participants behaving according  to `biascoin-model` with some parameter value (This is our psychological theory). This naturally leads to a generative model based on sampling `bias-weight` and then using `biascoin-model` to predict the distribution from which responses are sampled; conditioning on the data then lets us form our (scientific) beliefs about the parameter after seeing the data.

Here is a sketch of this model (it doesn't run efficiently---we'll fix that shortly):

~~~
;;;fold:
(define biascoin-model 
  (lambda (sequence bias-weight)
    (enumeration-query

     (define fair-weight 0.5)

     (define isfair (flip))

     (define the-weight (if isfair fair-weight bias-weight))

     (define coin (lambda () 
                    (flip the-weight)))

     isfair

     (equal? sequence 
             (repeat 5 coin)))))
;;;

;actual data from the experiment:
; organized as a list of two lists
; list 1: lists of sequences (stimuli)
; list 2: lists of responses
(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences (stimuli)
(define all-seqs (first experiment-data))
; list of responses
(define all-responses (second experiment-data))


(define data-analysis 
  (lambda (experiment-data)
    (query

     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) (biascoin-model sequence biased-weight)) 
        all-seqs))

     ; query for: what is the best biased-weight?
     biased-weight

     ; given that we've observed this data
     (condition 
      (all (flatten
            ; map over sequences 
            ; (all-responses, and cognitive-model-predictions are organized by sequence)
            (map 
             (lambda (data-for-one-sequence model-for-one-sequence)
               ; map over data points in a given sequence
               (map 
                (lambda (single-data-point)
                  ; condition on data = model
                  (equal? single-data-point (apply multinomial model)))
                data-for-one-sequence))
             all-responses
             cognitive-model-predictions))))))
~~~

Notice that there are two queries: one as part of the cognitive model ('in the head') and one as part of the data analysis model ('in the notebook').

As written, this model is very inefficient, because for each response, it *samples* a model prediction (by way of `(apply multinomial model)`, which produces a sample from the result of an `enumeration-query`) and checks to see if all model predictions match up with the observed data. We can re-write this in a more efficient way by computing the probability of the responses directly (using our handy `get-probability-of-faircoin` function from before), and adding them with a factor statement (instead of a condition statement).

~~~
;;;fold:
(define biascoin-model 
  (mem (lambda (sequence bias-weight)
         (enumeration-query

          (define fair-weight 0.5)

          (define isfair (flip))

          (define the-weight (if isfair fair-weight bias-weight))

          (define coin (lambda () 
                         (flip the-weight)))


          isfair

          (equal? sequence (repeat 5 coin))))))


(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
      (list-ref (second dist) index)))

(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))
;;;

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query

     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (biascoin-model sequence biased-weight)) 
        all-seqs))

     ; what is the best biased-weight?
     biased-weight

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            ; map over sequences 
                            (lambda (data-for-one-sequence model)
                              ; map over data points in a given sequence
                              (map (lambda (single-data-point)
                                     ; compute log p(data | model)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            all-responses
                            cognitive-model-predictions)))))))

(barplot (data-analysis experiment-data) "inferred bias weight")
~~~

We've taken some data, and written a cognitive model with some parameters (in this case, one parameter: bias-weight), and asked what the most likely value of that parameter is.
Our inferred parameter distribution reflects the beliefs we should have as scientists given the data and the assumptions that go into the model. 


# Posterior predictive

We often want to explore how well the predictions of the model match the actual data. When the model has parameters, the fit to data will depend on parameters... it is common to maximize the parameters and then look at model-data correlation. This can give a good sense of how well the model *could* do, but doesn't take into account our uncertainty (as scientists) about what values the parameters actually take on.
Another way to explore how well a model does is to look at the predictions of the model under the "inferred" parameter settings. 
This is called the *posterior predictive* distribution: it is the data that the model *actually* predicts, accounting for our beliefs about parameters after seeing the actual data. 

Here we look at the posterior predictive of the coin flip model, summarizing both the model predictions and the data by the condition means (i.e. the percent 'yes' responses to each coin sequence).

~~~
;;;fold:
(define biascoin-model 
  (mem (lambda (sequence bias-weight)
         (enumeration-query

          (define fair-weight 0.5)

          (define isfair (flip))

          (define the-weight (if isfair fair-weight bias-weight))

          (define coin (lambda () 
                         (flip the-weight)))


          isfair

          (equal? sequence (repeat 5 coin))))))

(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
      (list-ref (second dist) index)))

; compute mean "fair-ness" for each sequence (human data)
(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

; compute mean "fair-ness" for each sequence (model predictions)
(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))

; compute expected value of posterior distribution
(define expval-from-enum-analysis-of-enum-model 
  (lambda (results)
    (map sum 
         (transpose (map 
                     (lambda (lst prob)
                       (map (lambda (x)
                              (* prob x))
                            (second lst)))
                     (first results) 
                     (second results))))))
;;;

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query

     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (biascoin-model sequence biased-weight)) 
        all-seqs))


     ; what are the model predictions?
     (summarize-model cognitive-model-predictions)

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            (lambda (data-for-one-sequence model)
                              (map (lambda (single-data-point)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            all-responses
                            cognitive-model-predictions)))))))

(define results (data-analysis experiment-data))

(define posterior-predictive (expval-from-enum-analysis-of-enum-model results))

(scatter 
 (zip 
  posterior-predictive
  (second (summarize-data experiment-data)))
 "data vs. model")

(barplot (list all-seqs posterior-predictive) 
  "model: probability of fair?")

(barplot (list all-seqs (second (summarize-data experiment-data))) 
  "data: proportion of fair responses")
~~~

Our model provides an ok explanation of the data set, but we see from the scatter plot that there are some clear mismatches: the sequences that the model thinks are most likely fair are ones that people think are not fair. 
Looking more closely, these are sequences with many T values, such as TTTTT.

To gain more intuition, explore the following data set:

~~~
(define experiment-data
  (list 
   (list 
    (list false false false false false)
    (list false false false false true)
    (list false false false true true)
    (list false false true true true) 
    (list false true true true true)
    (list true true true true true))

   (list 
    (list #f #f #f)
    (list #f #f #t)
    (list #f #t #t)
    (list #f #t #t)
    (list #f #f #t)
    (list #f #f #f))))
~~~

What is the posterior over the `bias weight`? How does the posterior predictive look? What can you conclude about our bias coin model (with respect to this data)?

# Response noise

Perhaps the cognitive model differs from the data in ways that aren't 'central' to the theory, that is in ways that we wouldn't want to include in the cognitive model per se, but would like to account for. A common case is *random guessing*: participants sometimes respond randomly, instead of attending to the task. 
We can capture this *response noise* by simply extending our model with the possibility that each response came from a random guess:

~~~
(define data-analysis
  (query
    (define cognitive-model-predictions (bc-model ...))
    (define guessing-parameter (uniform 0 1))

    (define thinking-plus-guessing 
      (lambda (guessing-parameter)
        (if (flip guessing-parameter)
            (flip 0.5)
            cognitive-model-predictions)))


    query-statement

    (condition 
      (equal? data (thinking-plus-guessing guessing-parameter)))))
~~~

This pseudo-code is saying there is some probability (or, equivalently, proportion of responses) that is attributable to response noise, or guessing; 
this probability is captured by `guessing-parameter`. It is the amount of the data that is better captured by guessing behavior than our cognitive model predictions.
Thus, the estimate of `guessing-parameter` can be thought of as a measure of the amount of data that the model can account for.

## Data analysis model with response noise

~~~
;;;fold:
(define (get-indices needle haystack)
  (define (loop rest-of-haystack index)
    (if (null? rest-of-haystack) '()
        (let ((rest-of-indices (loop (rest rest-of-haystack) (+ index 1))))
          (if (equal? (first rest-of-haystack) needle)
              (pair index rest-of-indices)
              rest-of-indices))))
  (loop haystack 1))

;; takes in the output of enumeration (a joint posterior)
;; and outputs the marginals
(define (marginalize output)
  (let ([states (first output)])
    (map (lambda (sub-output) 
           (let* ([probs (second output)]
                  [unique-states (unique sub-output)]
                  [unique-state-indices 
                   (map 
                    (lambda (x) (list x (get-indices x sub-output))) 
                    unique-states)])

             (list (map first unique-state-indices)
                   (map 
                    (lambda (y) (sum (map 
                                      (lambda (x) (list-elt probs x)) 
                                      (second y)))) 
                    unique-state-indices))))
         (transpose states))))

(define bc-model 
  (mem
   (lambda (sequence bias-weight)
     (enumeration-query

      (define fair-weight 0.5)
      (define isfair (flip))
      (define the-weight (if isfair fair-weight bias-weight))
      (define coin (lambda () (flip the-weight)))

      isfair

      (equal? sequence 
              (repeat 5 coin))))))

(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
    (list-ref (second dist) index)))

; takes the mean "true" responses for each sequence
(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))

(define expval-from-enum-analysis-of-enum-model 
  (lambda (results)
    (map sum 
         (transpose (map 
                     (lambda (lst prob)
                       (map (lambda (x)
                              (* prob x))
                            (second lst)))
                     (first results)
                     (second results))))))
;;;


(define thinking-and-guessing 
  (lambda (sequence bias-weight guessing-parameter)
    (enumeration-query
     (define thinking (bc-model sequence bias-weight))
     (define guessing (list '(#t #f) '(0.5 0.5)))
     (define response
       (if (flip guessing-parameter)
           guessing
           thinking))

     (apply multinomial response)

     true)))

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query

     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9)))

     (define response-noise 
       (uniform-draw (list 0 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9 1)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (bc-model sequence biased-weight)) 
        all-seqs))

     (define cognitive-plus-noise-predictions
       (map 
        (lambda (sequence)
          (thinking-and-guessing sequence biased-weight response-noise))
        all-seqs))

     ; joint query: 
     ; what are the model predictions? including our model of noise
     ; what are the cognitive model predictions? (idealized; no noise)
     ; what is the response noise?
     ; what is the biased-weight?
     (list 
      (summarize-model cognitive-plus-noise-predictions)
      (summarize-model cognitive-model-predictions)
      response-noise
      biased-weight)

     ; given that we've observed this data
     (factor (sum (flatten 
                   ; map over all of the predictions over our cogmodel+noise
                   (map 
                    (lambda (data-for-one-sequence model)
                      ; map over data points in a given sequence
                      (map (lambda (single-data-point)
                             (log (get-probability-of-faircoin model single-data-point)))
                           data-for-one-sequence))
                    all-responses
                    cognitive-plus-noise-predictions)))))))

(define results (marginalize (data-analysis experiment-data)))

(define posterior-predictive-withNoise-results (first results))
(define posterior-predictive-sansNoise-results (second results))
(define posterior-noise (third results))
(define posterior-bias (fourth results))


(define posterior-predictive-withNoise
  (expval-from-enum-analysis-of-enum-model posterior-predictive-withNoise-results))

(define posterior-predictive-sansNoise
  (expval-from-enum-analysis-of-enum-model posterior-predictive-sansNoise-results))

(scatter 
 (zip 
  posterior-predictive-withNoise
  (second (summarize-data experiment-data)))
 "data vs. cognitive model (including noise)")

(barplot (list all-seqs posterior-predictive-withNoise) 
         "model (with noise): probability of fair?")
(barplot (list all-seqs posterior-predictive-sansNoise) 
         "model (sans noise): probability of fair?")

(barplot (list all-seqs (second (summarize-data experiment-data))) 
         "data: proportion of fair responses")

(barplot posterior-noise "posterior on response noise")

(barplot posterior-bias "posterior on biased-weight")
~~~

The posterior on response noise is peaked around 0.5. Our best guess is that fifty percent of the data comes from noise---does this seem high to you? 
What is the difference between the model with noise and the model without noise? (Theoretically, but also how do the predictions differ?)

Notice that while the scatter plot looks a bit better, our problem isn't really solved by factoring in response noise (though it is useful and informative to do so).
What is our problem again? Our model makes good predictions for most of these sequences, but is failing with: TTTTT, TTTTH, and so on.
Why might this be the case? To gain an intuition, let's reexamine the bias-weight parameter value. 
The biased-weight is peaked at 0.9 now. What does this mean in terms of our cognitive model?
Recall the biased-coin-model: it is a psychological theory that says subjects compare the sequence a fair coin would generate vs. one that a biased-coin would generate.
The biased-coin sequence has it's own weight, in this case the sequences it prefers are going to be sequences with lots of Heads (since our inferred weight is = 0.9).
This hints at a fundamental flaw of this model: it can only predict biased-sequences in one direction; 'unfair coin' responses for sequences going the other way have to get attributed to random response noise! How could we get around this issue? 


# Moving the coin-weight "into" the cognitive model 

We can posit that people don't entertain just one biased-weight value, but rather a distribution of possible values.
(Note: this is what Griffiths & Tenenbaum (2001) did.)
Thus, while our above data analysis model put uncertainty over the biased-weight, but assumed that people used only one weight, whatever it was, we now want to place this uncertainty *within* the cognitive model.
Forgetting about the data analysis model for the time being, we move the `biased-weight` parameter from outside the model to inside the model.
Let's examine the behavior of this model with respect to two sequences of interest.

~~~
(define enriched-biascoin-model 
  (lambda (sequence)
    (enumeration-query

     (define fair-weight 0.5)
     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

     (define isfair (flip))


     (define the-weight (if isfair 
                            fair-weight 
                            biased-weight))

     (define coin (lambda () 
                    (flip the-weight)))


     isfair

     (equal? sequence (repeat 5 coin)))))

(barplot (enriched-biascoin-model (list true true true true true)) "HHHHH is fair?")
(barplot (enriched-biascoin-model (list false false false false false)) "TTTTT is fair?")
~~~

This model matches our intuition for the fairness of these sequences. Do you see why?
To gain more insight, let's look at the posterior over the bias weight for different observed sequences:

~~~
(define enriched-biascoin-model 
  (lambda (sequence)
    (enumeration-query

     (define fair-weight 0.5)
     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

     (define isfair (flip))


     (define the-weight (if isfair 
                            fair-weight 
                            biased-weight))

     (define coin (lambda () 
                    (flip the-weight)))


     biased-weight

     (equal? sequence 
             (repeat 5 coin)))))

(barplot (enriched-biascoin-model (list true true true true true)) "what weight generated HHHHH")
(barplot (enriched-biascoin-model (list false false false false false)) "what weight generated TTTTT")
~~~

The model has the flexibility to infer different biased coin weights for different sequences. Let's now add back in the data analysis model to compare this to our data set.

~~~
;;;fold:
(define enriched-biascoin-model 
  (mem (lambda (sequence)
    (enumeration-query

     (define fair-weight 0.5)
     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

     (define isfair (flip))


     (define the-weight (if isfair 
                            fair-weight 
                            biased-weight))

     (define coin (lambda () 
                    (flip the-weight)))


     isfair

     (equal? sequence (repeat 5 coin))))))

(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

(define all-seqs (first experiment-data))
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (let ([index (list-index (first dist) selection)])
      (list-ref (second dist) index))))

(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))

(define expval-from-enum-analysis-of-enum-model 
  (lambda (results)
    (map sum 
         (transpose (map 
                     (lambda (lst prob)
                       (map (lambda (x)
                              (* prob x))
                            (second lst)))
                     (first results) 
                     (second results))))))
;;;

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query


     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (enriched-biascoin-model sequence)) 
        all-seqs))


     ; what are the model predictions?
     (summarize-model cognitive-model-predictions)

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            (lambda (data-for-one-sequence model)
                              ; map over data points in a given sequence
                              (map (lambda (single-data-point)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            (second experiment-data)
                            cognitive-model-predictions)))))))

(define results (data-analysis experiment-data))

(define posterior-predictive (expval-from-enum-analysis-of-enum-model results))

(scatter 
 (zip 
  posterior-predictive
  (second (summarize-data experiment-data)))
 "data vs. model")

(barplot (list all-seqs posterior-predictive) 
  "model: probability of fair?")

(barplot (list all-seqs (second (summarize-data experiment-data))) 
  "data: proportion of fair responses")
~~~

This is great. The model doesn't suffer from the same *lower-bias* flaw that it did previously.
Notice that our cognitive model now has no parameters for the data analysis model to infer; the posterior predictive is thus the same as the 'prior predictive', that is, what we would expect before seeing any data. 

## Generalized, enriched bias coin model 

We can further extend the above cognitive model. For instance, while it appears that people consider different weights for biased coins, do they have an overall bias to prefer heads-weighted or tails-weighted coins? We can explore this hypothesis by generalizing the uniform distribution to some distribution of coin weights, whose mean and variance have uncertainty. If the uncertain mean and variance are outside the cognitive model, then they are scientific hypotheses about the bias people bring to this problem; if they are inside, they represent the hypothesis that people reason about the mean and variance of the prior distribution.

Here's a model extended to capture uncertainty about the biases of participants. Note: This will take about 30 seconds to run.

~~~
;;;fold:
(define discretize-beta 
  (lambda (gamma delta bins)
    (define shape_alpha (* gamma delta))
    (define shape_beta (* (- 1 gamma) delta))
    (define beta-pdf (lambda (x) 
                       (*
                        (pow x (- shape_alpha 1))
                        (pow (- 1 x) (- shape_beta 1)))))
    (map beta-pdf bins)))

(define bins '(0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9))

(define (get-indices needle haystack)
  (define (loop rest-of-haystack index)
    (if (null? rest-of-haystack) '()
        (let ((rest-of-indices (loop (rest rest-of-haystack) (+ index 1))))
          (if (equal? (first rest-of-haystack) needle)
              (pair index rest-of-indices)
              rest-of-indices))))
  (loop haystack 1))

(define (list-map lst)
  (if (all (map null? (map rest lst))) 
      lst
      (list (map first lst) (list-map (map rest lst)))))

(define (marginalize output)
  (let ([states (first output)])
    (map (lambda (sub-output) 
           (let* ([probs (second output)]
                  [unique-states (unique sub-output)]
                  [unique-state-indices 
                   (map 
                    (lambda (x) (list x (get-indices x sub-output))) 
                    unique-states)])

             (list (map first unique-state-indices)
                   (map 
                    (lambda (y) (sum (map 
                                      (lambda (x) (list-elt probs x)) 
                                      (second y)))) 
                    unique-state-indices))))

         (transpose states))))


(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))


; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
    (list-ref (second dist) index)))

(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))

(define expval-from-enum-analysis-of-enum-model 
  (lambda (results)
    (map sum 
         (transpose (map 
                     (lambda (lst prob)
                       (map (lambda (x)
                              (* prob x))
                            (second lst)))
                     (first results)
                     (second results))))))
;;;

(define generalized-biascoin-model 
  (mem (lambda (sequence gamma delta)
         (enumeration-query

          (define fair-weight 0.5)

          (define biased-weight
            (multinomial bins (discretize-beta gamma delta bins)))

          ; (define biased-weight 
          ;    (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

          (define isfair (flip))


          (define the-weight (if isfair 
                                 fair-weight 
                                 biased-weight))

          (define coin (lambda () 
                         (flip the-weight)))

          isfair

          (equal? sequence 
                  (repeat 5 coin))))))

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query

     (define gamma (uniform-draw (list 0.1 0.3 0.5 0.7 0.9)))
     (define delta (uniform-draw (list 0.1 0.5 1 3 7 15)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (generalized-biascoin-model sequence gamma delta)) 
        all-seqs))


     ; what are the model predictions?
     (list 
      (summarize-model cognitive-model-predictions)
      gamma
      delta)

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            (lambda (data-for-one-sequence model)
                              ; map over data points in a given sequence
                              (map (lambda (single-data-point)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            (second experiment-data)
                            cognitive-model-predictions)))))))



(define results (marginalize (data-analysis experiment-data)))


(define posterior-predictive-results (first results))
(define posterior-gamma (second results))
(define posterior-delta (third results))
(define posterior-predictive (expval-from-enum-analysis-of-enum-model posterior-predictive-results))

(scatter 
 (zip 
  posterior-predictive
  (second (summarize-data experiment-data)))
 "data vs. cognitive model")

(barplot (list all-seqs posterior-predictive) 
         "cognitive model: probability of fair?")

(barplot (list all-seqs (second (summarize-data experiment-data))) 
         "data: proportion of fair responses")

(barplot posterior-gamma "posterior on mean biased-weight")
(barplot posterior-delta "posterior on variance of biased-weight")
~~~

Does it appear that participants in our experiment have an overall bias toward heads or tails?
While this model does even better at predicting the data, it is not perfect.
Let's see what happens when we factor in response noise to determine whether thats a better explanation for some of the data.
Note: This will also take a while. Chrome may ask you to kill the page; power through.

~~~
;;;fold:
(define discretize-beta 
  (lambda (gamma delta bins)
    (define shape_alpha (* gamma delta))
    (define shape_beta (* (- 1 gamma) delta))
    (define beta-pdf (lambda (x) 
                       (*
                        (pow x (- shape_alpha 1))
                        (pow (- 1 x) (- shape_beta 1)))))
    (map beta-pdf bins)))

(define bins '(0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9))

(define (get-indices needle haystack)
  (define (loop rest-of-haystack index)
    (if (null? rest-of-haystack) '()
        (let ((rest-of-indices (loop (rest rest-of-haystack) (+ index 1))))
          (if (equal? (first rest-of-haystack) needle)
              (pair index rest-of-indices)
              rest-of-indices))))
  (loop haystack 1))

(define (list-map lst)
  (if (all (map null? (map rest lst))) 
      lst
      (list (map first lst) (list-map (map rest lst)))))

(define (marginalize output)
  (let ([states (first output)])
    (map (lambda (sub-output) 
           (let* ([probs (second output)]
                  [unique-states (unique sub-output)]
                  [unique-state-indices 
                   (map 
                    (lambda (x) (list x (get-indices x sub-output))) 
                    unique-states)])

             (list (map first unique-state-indices)
                   (map 
                    (lambda (y) (sum (map 
                                      (lambda (x) (list-elt probs x)) 
                                      (second y)))) 
                    unique-state-indices))))

         (transpose states))))


(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
    (list-ref (second dist) index)))

(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))


(define expval-from-enum-analysis-of-enum-model 
  (lambda (results)
    (map sum 
         (transpose (map 
                     (lambda (lst prob)
                       (map (lambda (x)
                              (* prob x))
                            (second lst)))
                     (first results)
                     (second results))))))
;;;


(define bc-model 
  (mem 
   (lambda (sequence gamma delta)
     (enumeration-query

      (define fair-weight 0.5)

      (define biased-weight
        (multinomial bins (discretize-beta gamma delta bins)))

      ; (define biased-weight 
      ;    (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

      (define isfair (flip))


      (define the-weight (if isfair 
                             fair-weight 
                             biased-weight))

      (define coin (lambda () 
                     (flip the-weight)))


      isfair

      (equal? sequence 
              (repeat 5 coin))))))

(define thinking-and-guessing 
  (lambda (sequence gamma delta guessing-parameter)
    (enumeration-query
     (define thinking (bc-model sequence gamma delta))
     (define guessing (list '(#t #f) '(0.5 0.5)))
     (define response
       (if (flip guessing-parameter)
           guessing
           thinking))

     (apply multinomial response)

     true)))

(define data-analysis 
  (lambda (experiment-data)
    (enumeration-query

     (define gamma (uniform-draw (list 0.1 0.3 0.5 0.7 0.9)))
     (define delta (uniform-draw (list 0.1 0.5 1 3 7 15)))

     ; generate predictions for all sequences
     (define cognitive-model-predictions
       (map 
        (lambda (sequence) 
          (bc-model sequence gamma delta)) 
        all-seqs))

     (define response-noise (uniform-draw (list 0 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9 1)))


     (define cognitive-plus-noise-predictions
       (map 
        (lambda (sequence)
          (thinking-and-guessing sequence gamma delta response-noise))
        all-seqs))


     ; what are the model predictions?
     (list 
      (summarize-model cognitive-plus-noise-predictions)
      (summarize-model cognitive-model-predictions)
      response-noise
      gamma
      delta)

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            (lambda (data-for-one-sequence model)
                              ; map over data points in a given sequence
                              (map (lambda (single-data-point)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            all-responses
                            cognitive-plus-noise-predictions)))))))



(define results (marginalize (data-analysis experiment-data)))

(define posterior-predictive-withNoise-results (first results))
(define posterior-predictive-sansNoise-results (second results))
(define posterior-noise (third results))
(define posterior-gamma (fourth results))
(define posterior-delta (fifth results))

(define posterior-predictive-withNoise 
  (expval-from-enum-analysis-of-enum-model posterior-predictive-withNoise-results))

(define posterior-predictive-sansNoise 
  (expval-from-enum-analysis-of-enum-model posterior-predictive-sansNoise-results))

(scatter 
 (zip 
  posterior-predictive-withNoise
  (second (summarize-data experiment-data)))
 "data vs. cognitive model")

(barplot (list all-seqs posterior-predictive-withNoise) 
         "cognitive model (with noise): probability of fair?")
(barplot (list all-seqs posterior-predictive-sansNoise) 
         "cognitive model (sans noise): probability of fair?")
(barplot (list all-seqs (second (summarize-data experiment-data))) 
         "data: proportion of fair responses")

(barplot posterior-noise "posterior on noise parameter")
(barplot posterior-gamma "posterior on mean biased-weight")
(barplot posterior-delta "posterior on varaince of biased-weight")
~~~

How much of the data must be explained as noise in this extended model?


# Model selection

We've explored a number of different models and seen that some seem better, explaining more of the data, though they differ in their complexity. How can we quantify which model is  better? We can set up the question like this: we, as scientists, are a priori uncertain which cognitive model actually gave rise to the data we have collected; after seeing the data, how do our beliefs about the correct model change? In a way, this is no different than the inference problems we've faced before.
In pseudocode this might look like:

~~~
(define model-comparion
  (query
    (define model-1 (simple-bc-model ...))
    (define model-2 (complex-bc-model ...))

    (define is-model-1? (flip 0.5))

    (define best-model
        (if is-model1?
            model-1
            model-2))

    is-model-1?

    (condition 
      (equal? data (best-model)))))
~~~


Let's try to write this in full:

~~~
;;;fold:
(define experiment-data
  (list
   (list
    (list #t #t #t #t #t) 
    (list #t #t #t #t #f) 
    (list #t #t #t #f #t) 
    (list #t #t #t #f #f) 
    (list #t #t #f #t #t) 
    (list #t #t #f #t #f) 
    (list #t #t #f #f #t) 
    (list #t #t #f #f #f) 
    (list #t #f #t #t #t) 
    (list #t #f #t #t #f) 
    (list #t #f #t #f #t) 
    (list #t #f #t #f #f) 
    (list #t #f #f #t #t) 
    (list #t #f #f #t #f) 
    (list #t #f #f #f #t) 
    (list #t #f #f #f #f) 
    (list #f #t #t #t #t) 
    (list #f #t #t #t #f) 
    (list #f #t #t #f #t) 
    (list #f #t #t #f #f) 
    (list #f #t #f #t #t) 
    (list #f #t #f #t #f) 
    (list #f #t #f #f #t) 
    (list #f #t #f #f #f)
    (list #f #f #t #t #t) 
    (list #f #f #t #t #f) 
    (list #f #f #t #f #t) 
    (list #f #f #t #f #f) 
    (list #f #f #f #t #t) 
    (list #f #f #f #t #f) 
    (list #f #f #f #f #t) 
    (list #f #f #f #f #f))
   (list 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #t #f #f #t #f) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #t #f #f #f #t #f #f #f #t #f #f #t #f #f #f #t #f #t #t #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #f #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #f #t #f #t #f #t #t #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #t #f #t #f #f #f #f #f #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #f #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #t #t #f #f #f #t #f #f #f #f #f #f #f #t #f) 
    (list #f #t #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #f #t #f #t #t #f #t #f #t #t #t #t #t #f #t #t #t #t #f #f #t #f) 
    (list #f #t #t #t #t #t #t #f #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #f) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #f #t #f #f #t #t #t #f #t #f #t #t #f #f #t #f #t #f #f #t #t #f #f #t #f #f #t #f #t #t) 
    (list #f #t #f #t #t #t #f #f #f #t #t #t #t #f #t #f #t #t #t #f #t #t #f #t #t #t #t #f #t #f) 
    (list #f #t #f #t #t #t #t #f #t #t #t #t #t #f #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t #t) 
    (list #t #t #f #f #t #t #f #f #t #f #f #f #t #f #t #f #t #f #t #f #t #f #t #t #f #f #t #f #t #t) 
    (list #t #t #t #t #t #t #f #f #t #t #t #t #t #t #t #f #t #f #t #t #t #t #t #t #t #t #t #f #t #f) 
    (list #t #t #f #f #t #f #f #f #t #t #f #f #f #f #f #f #t #f #f #f #t #f #f #t #f #f #f #f #t #f) 
    (list #f #t #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f) 
    (list #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #f #t #f #f #f #f #f #f #f #f #f #f #f #t #f))))

; list of sequences
(define all-seqs (first experiment-data))
; list of responses 
(define all-responses (second experiment-data))

(define get-probability-of-faircoin
  (lambda (dist selection)
    (define index (list-index (first dist) selection))
    (list-ref (second dist) index)))

(define summarize-data 
  (lambda (dataset)
    (list (first dataset)
          (map 
           (lambda (lst) (mean (map boolean->number lst)))
           (second dataset)))))

(define summarize-model
  (lambda (modelpreds)
    (list 
     all-seqs
     (map 
      (lambda (dist) 
        (get-probability-of-faircoin dist #t))
      modelpreds))))

;;;

(define complex-bc-model 
  (mem (lambda (sequence)
         (enumeration-query

          (define fair-weight 0.5)
          (define biased-weight 
            (uniform-draw (list 0.1 0.2 0.3 0.4 0.5 0.6 0.7 0.8 0.9)))

          (define isfair (flip))
          (define the-weight (if isfair 
                                 fair-weight 
                                 biased-weight))

          (define coin (lambda () 
                         (flip the-weight)))


          isfair

          (equal? sequence (repeat 5 coin))))))


(define simple-bc-model 
  (mem
   (lambda (sequence bias-weight)
     (enumeration-query

      (define fair-weight 0.5)
      (define isfair (flip))
      (define the-weight (if isfair fair-weight bias-weight))
      (define coin (lambda () (flip the-weight)))

      isfair

      (equal? sequence (repeat 5 coin))))))



(define model-comparison 
  (lambda (experiment-data)
    (enumeration-query

     (define biased-weight 
       (uniform-draw (list 0.1 0.2 0.3 0.4 0.6 0.7 0.8 0.9)))



     (define is-model1? (flip 0.5))

     ; model1: generate predictions for all sequences
     (define model1
       (map 
        (lambda (sequence) 
          (simple-bc-model sequence biased-weight)) 
        all-seqs))

     ; model2: generate predictions for all sequences
     (define model2
       (map 
        (lambda (sequence) 
          (complex-bc-model sequence))
        all-seqs))

     (define best-model (if is-model1?
                            model1
                            model2))

     ; which is the best model?
     is-model1?

     ; given that we've observed this data
     (factor (sum (flatten (map 
                            (lambda (data-for-one-sequence model)
                              ; map over data points in a given sequence
                              (map (lambda (single-data-point)
                                     (log (get-probability-of-faircoin model single-data-point)))
                                   data-for-one-sequence))       
                            (second experiment-data)
                            best-model)))))))

(define results (model-comparison experiment-data))

(barplot results "is model 1 the best?")
~~~

Our data strongly favors the more complex model: we can be very confident in the more complex model being the better of the two, given our data and analysis assumptions.


# Exercises

**1. Bayes in the head vs. Bayes in the notebook.** We've seen in this chapter how we can precisely separate assumptions about our computational-level theory of cognition from the assumptions that go into analyzing our data (and our theory). In this exercise, we will try to go between the two ways of looking at these things: by going from a theory and analysis in words, to a theory and analysis in Church (and back).
	
Consider the [reflectance and luminance model](https://probmods.org/patterns-of-inference.html#a-case-study-in-modularity-visual-perception-of-surface-lightness-and-color) from Chapter 4. This model captured the illusion of increased reflectance in terms of explaining away the observed luminance by the decreased illumination (caused by the shadow). Here is the model again
	
~~~
(define observed-luminance 3.0)

(define samples
   (mh-query
    1000 10

    (define reflectance (gaussian 1 1))
    (define illumination (gaussian 3 0.5))
    (define luminance (* reflectance illumination))

    luminance

    ;true
    (= luminance (gaussian observed-luminance 0.1))))

(display (list "Mean reflectance:" (mean samples)))
(hist samples "Reflectance")
~~~
	
Here I have included a commented `true` condition to make it easy for you to explore this model. 
	
A. **Warmup: What does the prior for luminance look like? How does the prior change when we condition on the observed luminance.**
	
Just as a reminder, the illusion is observed in the model when we condition on this statement about illumination  `(condition (= illumination (gaussian 0.5  0.1)))`, which is a stand-in for the effect of the shadow from the cylinder on the scene.
	
B. **How many parameters does this model of perception have?** (Hint: Go through each `define` and `condition`: Are the constituent variables of the statements (a) modeling assumptions or (b) part of the experimental setup / manipulation) **For all of the variables you've categorized as (a), which ones do you think refer to aspects of the perceptual system and which refer to aspects of the environment? What do you think these parameters represent in terms of the perceptual system or environment? (Feel free to use super general, even colloquial, terms to answer this.)**
	
C. Replace the hard-coded parameters of this model with variables, defined outside the query. Give them the most intuitive names you can fashion. Use this starter (pseudo) code.
	
	
~~~
(define parameter1 ...)
(define parameter2 ...)
;...

(define observed-luminance 3.0)

(query

 (define reflectance (gaussian 1 1))
 (define illumination (gaussian 3 0.5))
 (define luminance (* reflectance illumination))

 luminance

 (= luminance (gaussian observed-luminance 0.1))))
~~~
	
D. **Are all of these parameters independent?** (If you had to specify values for them, would you have to consider values of other parameters when specifying them?) If two are not independent, can you think of a reparameterization that would be more independent? (Hint: If you have two non-independent parameters, you could keep only one of them and introduce a parameter specifying the relation between the two. E.g., two points that are linearly associated can be expressed as an one of them and the distance between them).
	
E. Writing data analysis models requires specifying priors over parameters. Without much prior knowledge in a domain, we want to pick priors that make the fewest assumptions. A good place to start is to think about the possible values the parameter could take on. **For each parameter, write down what you know about the possible values it could take on. **
	
F. We're now in a position to write a data analysis model. The most common distributional forms for priors are [uniform](http://en.wikipedia.org/wiki/Uniform_distribution_(continuous)), [gaussian](http://en.wikipedia.org/wiki/Normal_distribution), [beta](http://en.wikipedia.org/wiki/Beta_distribution), and [exponential](http://en.wikipedia.org/wiki/Exponential_distribution). Put priors on your parameters from part C. Use this starter (pseudo) code.
	
~~~
(define perceptual-model
  (lambda (parameter1 parameter2 ...))
  (query

   ; fill in, copying where appropriate from the original model specification
   (define reflectance ...)
   (define illumination ...)
   (define luminance (* reflectance illumination))

   luminance

   (= luminance ...))))



(define data-analysis-model
  (query
   ; replace with parameters you specified in Part C
   ; put priors over parameters
   (define parameter1 ...)
   (define parameter2 ...)
   (define ...)
   (define perceptual-model-predictions 
     (perceptual-model parameter1 parameter2 ...))
    
   ;;; what are you going to query for?
   ...  
   
   (condition (= experimental-data perceptual-model-predictions))))


~~~ 
	
G. What are you going to query for? Add that to your pseudocode above. What do each of things that you are querying for in the data analysis model represent?


**2. Parameter fitting vs. Parameter integration** One of the strongest motivations for using Bayesian techniques for model-data evaluation is in how "nuisance" parameters are treated. "Nuisance" parameters are parameters of no theoretical interest; their only purpose is to fill in a slot in the model. Classically, the most prominant technique (from the frequentist tradition) for dealing with these parameters is to fit them to the data, i.e., to set the value equal to whatever maximizes the model-data fit (or, equivalently, minimizes some cost function). 

The Bayesian approach is different. Since we have *a priori* uncertainty about the value of our parameter (as you specified in Part F of Exercise 1), we will also have *a posteriori* uncertainty (though hopefully the uncertainty will be a little less). What the Bayesian does is *integrate* over her posterior distribution of parameter values to make predictions. Intuitively, rather than taking the value corresponding to the peak of the distribution, she's considering all values with their respective heights.
	
Why might this be important for model assessment? 
