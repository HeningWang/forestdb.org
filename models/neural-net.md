---
layout: model
title: Bayesian Neural Network
model-status: code
model-category: Machine Learning
model-tags: neural net, continuous
---

This models is a neural network that learns the XOR function. The model is based on the [Anglican implementation of a neural net](http://www.robots.ox.ac.uk/~fwood/anglican/examples/neural_net/).

    ;; Define expected inputs and outputs as lists
    (define inputs  (list (list -1 -1) (list -1  1) (list  1 -1) (list  1  1)))
    (define outputs (list           0            1            1            0))
    
    ;; Helper functions for accessing expected inputs / outputs
    (define (I i l j) (list-ref (list-ref inputs i) j))
    (define (O i)     (list-ref outputs i))
    
    ;; Define number of layers
    (define num-layers 3)
    
    
    (define samples
      (mh-query 
    
       1000 1
    
       ;; Define number of nodes in each layer
       (define num-nodes
         (mem
          (lambda (l)
            (- (cond ((= l 0) 2)
                     ((= l 1) 4)
                     ((= l 2) 2)
                     ((= l 3) 1)) 1))))
    
       ;; Define weights dynamically as required
       ;; l = layer, n = node id; j = input node id in layer l-1
       (define w
         (mem
          (lambda (l n j) (gaussian 0 0.5))))
    
       ;; Define activation function
       (define (activate v)
         (if (<= v 0.5) -1 1))
    
       ;; Evaluate a single weighted input
       ;; f = function to evaluate to get input value, i = input training pattern id
       (define (get-input f i l n j)
         (* ((eval f) i l j) (w l n j)))
    
       ;; Sum all weighted inputs to a node
       (define (sum-inputs f i l n j)
         (if (= j 0)
             (get-input f i l n j)
             (+ (sum-inputs f i l n (- j 1)) (get-input f i l n j))))
    
       ;; Evaluate node activation
       (define N
         (mem
          (lambda (i l n)
            (activate
             (if (= l 1)
                 (sum-inputs 'I i (- l 1) n (num-nodes 0))
                 (sum-inputs 'N i (- l 1) n (num-nodes (- l 1))))))))
    
       ;; Compute output value based on layer 2 nodes
       (define p-O
         (mem
          (lambda (i)
            (if (>= (N i num-layers (num-nodes num-layers)) 0) 1 0))))
    
       ;; Predict response to the four input values
       (list (list (I 0 0 0) (I 0 0 1) (p-O 0))
             (list (I 1 0 0) (I 1 0 1) (p-O 1))
             (list (I 2 0 0) (I 2 0 1) (p-O 2))
             (list (I 3 0 0) (I 3 0 1) (p-O 3)))
    
       ;; Train the network using our training data
       (and (= (gaussian (p-O 0) 0.1 (O 0)) (O 0))
            (= (gaussian (p-O 1) 0.1 (O 1)) (O 1))
            (= (gaussian (p-O 2) 0.1 (O 2)) (O 2))
            (= (gaussian (p-O 3) 0.1 (O 3)) (O 3)))))
    
    
    (hist (map first samples) "Input -1, -1")
    (hist (map second samples) "Input -1, 1")
    (hist (map third samples) "Input 1, -1")
    (hist (map fourth samples) "Input 1, 1")
    
References:

- [Bayesian neural net in Anglican](http://www.robots.ox.ac.uk/~fwood/anglican/examples/neural_net/)
