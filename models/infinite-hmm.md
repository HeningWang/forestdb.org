---
layout: model
title: Infinite HMM
model-status: code
model-category: Nonparametric Models
model-tags: mem, nonparametrics, mixture, hmm
---

This Hidden Markov model has a potentially infinite number of latent states:

    (define vocabulary ’(chef omelet soup eat work bake))
    
    (define (get-state) (DPmem 0.5 gensym))
    
    (define state->transition-model 
      (mem 
        (lambda (state) 
          (DPmem 1.0 (get-state)))))
    
    (define (transition state) 
      (sample (state->transition-model state)))
    
    (define state->observation-model 
      (mem 
        (lambda (state) 
          (dirichlet (make-list (length vocabulary) 1)))))
    
    (define (observation state) 
      (multinomial vocabulary (state->observation-model state)))
    
    (define (sample-words last-state) 
      (if (flip 0.2) 
          '() 
          (pair (observation last-state) 
                (sample-words (transition last-state)))))
    
    (sample-words ‘start) 

Source: [probmods.org](https://probmods.org/observing-sequences.html#hidden-markov-models)

See also:

- [Hidden Markov Model](/models/hmm.html)
