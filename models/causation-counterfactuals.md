---
layout: model
title: Causation and counterfactuls
---

*This page argues that a counterfactual model in Church must be based on intervention rather than querying.*

# The problem

In the current counterfactual model, causation is only considered in its effect on the correlation between two variables. This is because dependence between the variables of interest are found with a condition statement, not intervention. Thus, the current model makes the correlation implies causation fallacy. A model that relies entirely on co-occurrence cannot possibly capture causality.

### Example: coughs and colds

Here is an example grounded in the real world of illness and symptoms. We think of illnesses causing symptoms and not the other way around even if p(illness|symptom) is higher than p(symptom|illness). The current counterfactual model does not produce these results:

~~~

;;;fold:
;;first we have a bunch of helper code to do meta-transforms.. converts name to 
;;shadow-name and wraps top-level defines
(define (names model)
  (map (lambda (def)
         (if (is-function-definition? def)
             (first (second def))
             (second def)))
       model))

(define (is-function-definition? def)
  (list? (second def)))

(define (shadow-symbol name)
  (string->symbol (string-append "shadow-" name)))

(define (rename expr from-name to-name)
  (cond [(list? expr) (map (lambda (x) (rename x from-name to-name)) expr)]
        [(eq? expr from-name) to-name]
        [else expr]))

(define (shadow-rename expr name)
  (rename expr name (shadow-symbol name)))

(define (shadow-rename-all expr names)
  (if (null? names)
      expr
      (shadow-rename-all (shadow-rename expr (first names))
                         (rest names))))

(define (make-shadow-defines model)
  (define ns (names model))
  (map (lambda (def)
         (if (is-function-definition? def)
             (shadow-rename-all def ns)
             (let ([name (second def)])
               '(define ,(shadow-symbol name) (if (flip eps) 
                                                  ,(shadow-rename-all (third def) ns) 
                                                  ,name)))))
       model))

;;the meaning function constructs a church expression from an utterance. 
;;for 'because it uses quasiquote mojo to dynamically construct the right expression.
;;(in principle this handles embedded "because", but currently expand-because doesn't do 
;;the right thing since the model is a fixed global.)
(define (meaning utt)
  (define (because? u) (if (list? u) (eq? (first u) 'because) false))
  (if (list? utt)
      (if (because? utt)
          (expand-because (map meaning utt))
          (map meaning utt))
      utt))

;;expand an expr with form '(because a b), ie "a because b", into the (hypothesized) 
;;counterfactual meaning:
(define (expand-because expr) 
  (define a (second expr))
  (define b (third expr))
  '(and ,a ,b
        (apply multinomial
               (enumeration-query
                (define eps 0.01)
                ,@(make-shadow-defines model) ;;the shadow model
                (not ,(shadow-rename-all a (names model)))
                (condition (not ,(shadow-rename-all b (names model))))))))

;;listener is standard RSA literal listener, except we dynamically construct the 
;;query to allow complex meanings that include because:
(define listener 
  (mem (lambda (utt qud)
         (eval
          '(enumeration-query
            ,@model
            ,qud
            (condition ,(meaning utt)))))))

;;the speaker is no different from ordinary RSA
(define (speaker val qud) ;;want to communicate val as value of qud
  (enumeration-query
   (define utt (utt-prior))
   utt
   (condition (equal? val (apply multinomial (listener utt qud))))))

;;;
;; returns value <= p# with probability p#
;;this takes the role of a uniform function if we only care about the value's relation
;; to the critical values used as input
(define (get-U p1 p2 p3)
  (multinomial (list p1 p2 p3 1)
               (list p1 (- p2 p1) (- p3 p2) (- 1 p3))))

(define model 
  '(
    ;; causal strengths
    (define B→cold .1)
    (define cold→cough (if (flip) .8 .2))
    
    (define U1 (get-U 0 0 .1))
    (define (cold) (<= U1 .1))

    (define U2 (get-U .1 .2 .8))
    (define (cough) (if (cold) (<= U2 cold→cough) (<= U2 .1)))
    ))

(define (utt-prior) (uniform-draw '( (because (cough) (cold) )
                                     (because (cold) (cough) ))))
(barplot (speaker (list #t #t .8) '(list (cough) (cold) cold→cough)))
~~~

We see here that in a situation where the symptom implies the illness more strongly than the illness implies the symptom, the model prefers to say "illness because symptom." This is problematic.
