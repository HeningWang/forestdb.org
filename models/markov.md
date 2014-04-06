---
layout: model
title: Markov Model
model-status: code
model-category: Machine Learning
model-tags: temporal models
---

A Markov model is a model of a sequence of unobserved states. Each state depends only on the previous state.

    (define (transition state)
      (cond
       ((eq? state 'a) (multinomial '(a b c) '(0.7 0.2 0.1)))
       ((eq? state 'b) (multinomial '(a b c) '(0.3 0.3 0.4)))
       ((eq? state 'c) (multinomial '(a b c) '(0.3 0.65 0.05)))))
    
    (define (markov state n)
      (if (= n 0)
          '()
          (pair state (markov (transition state) (- n 1)))))
    
    (markov 'a 10)

See also:

- [Hidden Markov Model](/models/hmm.html)
