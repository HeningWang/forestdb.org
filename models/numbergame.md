---
layout: model
title: SAILORS Teaching Example
model-language: webppl
---

## Introduction

AI researchers have become extremely good at designing algorithms to process and make sense of the huge amount of messy, unstructured data we have on the Internet. It is commonly believed that "Big Data" will revolutionalize AI and enable machines to automatically acquire rich knowledge from data in order to behave intelligently. While the sheer amount of data that the ditigal world has accumulated is undoubtedly powerful, it is often useful to take a step back and think about how humans--the most intelligent machines we have around--reason about and learn from data. Humans don't always require thousands of data points to learn concepts and draw conclusions. Often, we see a small amount of data and draw highly reasonable conclusions about it. This is sometimes called an "inductive leap:" observing small pieces of evidence, making generalizations about them, and using these generalizations to draw larger conclusions and make predictions.

Let's make this idea of inductive reasoning a bit more concrete with a simple example.

Suppose your friend shows you this sequence of numbers and tells you that they are "positive examples" of a category of numbers: {16, 8, 2, 64}.

(Note: "positive examples" are examples of that category; "negative examples" are examples that are *not* of that category.)

*Do you think 4 belongs to this category?
*What about 7?
*Or 10?

Now, suppose someone shows you this sequence of numbers and tells you that they are "positive examples" of a different category of numbers: {60, 80, 10, 30}

*Do you think 4 belongs to this category?
*What about 7?
*Or 10?

How did you come up with these answers? What do you think these number categories are?

### Hypotheses

Let's zoom in on the first sequence: {2, 8, 16, 64}. What are some hypotheses of which number category these numbers came from? Another way to think of this is, what is the process that *generated* these numbers?

#### Generative Models

It is often intuitive and helpful to think of data samples (such as these numbers) as being "generated" from a category. Since there are several different number categories that we consider, there are also several different ways in which these numbers were generated from those categories.

Here is one way the numbers {2, 8, 16, 64} could have been generated:

1. Your friend was thinking of the concept (or number category): "powers of 2".
2. Your friend randomly samples some numbers that are powers of 2.

This is what that looks like in code:

~~~~
///fold:
// sample N items from an array, without replacement
var drawN = function(array, N) {
  if (N == 0) {
    return [];
  } else {
    var sampled = uniformDraw(array);
    var remaining = remove(sampled, array);
    return [sampled].concat(drawN(remaining, N-1));
  }
}
// sample 4 items from an array, without replacement
var draw4 = function(array) {
  return drawN(array, 4);
}
///

// first we make a list of all the powers of 2 less than 50:
var powers_of_2 = [1, 2, 4, 8, 16, 32];

// then we sample 4 of those powers of 2:
var get_examples = function() {return draw4(powers_of_2)};

get_examples();
~~~~

You can press the "run" button above multiple times to get different samples from this category.

You can also see graphs of this simulation. (Let's put our 4 numbers in order, so the graph looks nicer)

~~~~
///fold:
// sample N items from an array, without replacement
var drawN = function(array, N) {
  if (N == 0) {
    return [];
  } else {
    var sampled = uniformDraw(array);
    var remaining = remove(sampled, array);
    // NOTE: now we're sorting the result.
    return sort([sampled].concat(drawN(remaining, N-1)));
  }
}
// sample 4 items from an array, without replacement
var draw4 = function(array) {
  return drawN(array, 4);
}

// first we make a list of all the powers of 2 less than 50:
var powers_of_2 = [1, 2, 4, 8, 16, 32];

// then we sample 4 of those powers of 2:
var get_examples = function() {return draw4(powers_of_2)};

get_examples();
///
print(Enumerate(get_examples));
~~~~  

#### Powers of 3 or Multiples of 3?

Of course, there are other ways the numbers {2, 8, 16, 64} could have been generated:

1. Your friend was thinking of the concept (or number category): "multiples of 2".
2. Your friend randomly samples some numbers that are multiples of 2.

Or even this generative process:

1. Your friend was thinking of the concept (or number category): "integers less than 50".
2. Your friend randomly samples some numbers that are integers smaller than 50.


##### Generative Model

~~~~
///fold:
// sample N items from an array, without replacement
var drawN = function(array, N) {
  if (N == 0) {
    return [];
  } else {
    var sampled = uniformDraw(array);
    var remaining = remove(sampled, array);
    // NOTE: now we're sorting the result.
    return sort([sampled].concat(drawN(remaining, N-1)));
  }
}
// sample 4 items from an array, without replacement
var draw4 = function(array) {
  return drawN(array, 4);
}

// first we make a list of all the powers of 2 less than 50:
var powers_of_2 = [1, 2, 4, 8, 16, 32];

// then we sample 4 of those powers of 2:
var get_examples = function() {return draw4(powers_of_2)};

get_examples();
///
print(Enumerate(get_examples));
~~~~  



    ///fold:
    var seq = function(a, b, include_end_point) {
    
      // if 1 argument is given, that's "end" and "start" is 0.
      // if 2 arguments are given, the first is the "start" and the second is the "end"
    
      var start = b ? a : 0;
      var end = b? b : a;
    
      if (end <= start) {
        // if the end is equal to the start, return an empty list
        return [];
      } else {
        // if not, recursively call "seq" on a smaller interval
        // (move "start" closer and closer to "end", while adding
        // each of the "start" values")
        return [start].concat(seq(start+1, end));
      }
    }
    
    var sample_without_replacement = function(list, N) {
      if (N <= 0) {
        return [];
      } else {
        var next_sample = uniformDraw(list);
        var new_list = remove(next_sample, list);
        return [next_sample].concat(sample_without_replacement(new_list, N-1));
      }
    }
    
    var fns = {
      "powers_of_3": function(x) {return x * 3;},
      "multiples_of_3": function(x) {return x + 3;}
    }
    
    var first_numbers = {
      "powers_of_3": 1,
      "multiples_of_3": 0
    }
    
    var max_number = 10;
    
    //I've re-written this code a bit, to be more general.
    var generate = cache(function(concept) {
      var accumulate = function(previous_number) {
        // first power of 2 is 2^0 = 1
        var previous_number = previous_number ? previous_number : first_numbers[concept];
    
        // next power of 2.
        var fn = fns[concept];
        var next_number = fn(previous_number);
    
        // only keep powers of 2 up to 50.
        if (next_number > max_number) {
          return [previous_number]
        } else {
          return [previous_number].concat(accumulate(next_number));
        }
      }
      return accumulate();
    })
    
    var sample_from_concept = function(concept, N) {
      // if no N (number of samples) is given, give 4 samples
      var N = N ? N : 2;
      var all_in_concept = generate(concept);
      return sample_without_replacement(all_in_concept, N);
    }
    
    var list_eq = function(l1, l2) {
      return all(function(x) {return x == 1;}, map2(eq, l1, l2));
    }
    ///
    
    var generative_model = function() {
      var concept = uniformDraw(["powers_of_3", "multiples_of_3"]);
      var samples = sample_from_concept(concept);
      return samples;
    }
    
    print(Enumerate(generative_model));

##### Infer category given number sequence

Now that we have different hypotheses about how these numbers were generated, how do we determine which hypothesis is the most likely one given the data?

We can do this using Bayes' Rule.

    ///fold:
    var seq = function(a, b, include_end_point) {
    
      // if 1 argument is given, that's "end" and "start" is 0.
      // if 2 arguments are given, the first is the "start" and the second is the "end"
    
      var start = b ? a : 0;
      var end = b? b : a;
    
      if (end <= start) {
        // if the end is equal to the start, return an empty list
        return [];
      } else {
        // if not, recursively call "seq" on a smaller interval
        // (move "start" closer and closer to "end", while adding
        // each of the "start" values")
        return [start].concat(seq(start+1, end));
      }
    }

    var sample_without_replacement = function(list, N) {
      if (N <= 0) {
        return [];
      } else {
        var next_sample = uniformDraw(list);
        var new_list = remove(next_sample, list);
        return [next_sample].concat(sample_without_replacement(new_list, N-1));
      }
    }
    
    var fns = {
      "powers_of_3": function(x) {return x * 3;},
      "multiples_of_3": function(x) {return x + 3;}
    }
    
    var first_numbers = {
      "powers_of_3": 1,
      "multiples_of_3": 0
    }
    
    var max_number = 10;
    
    //I've re-written this code a bit, to be more general.
    var generate = cache(function(concept) {
      var accumulate = function(previous_number) {
        // first power of 2 is 2^0 = 1
        var previous_number = previous_number ? previous_number : first_numbers[concept];
    
        // next power of 2.
        var fn = fns[concept];
        var next_number = fn(previous_number);
    
        // only keep powers of 2 up to 50.
        if (next_number > max_number) {
          return [previous_number]
        } else {
          return [previous_number].concat(accumulate(next_number));
        }
      }
      return accumulate();
    })
    
    var sample_from_concept = function(concept, N) {
      // if no N (number of samples) is given, give 4 samples
      var N = N ? N : 2;
      var all_in_concept = generate(concept);
      return sample_without_replacement(all_in_concept, N);
    }
    
    var list_eq = function(l1, l2) {
      return all(function(x) {return x == 1;}, map2(eq, l1, l2));
    }
    ///
    
    var generative_model = function() {
      var concept = uniformDraw(["powers_of_3", "multiples_of_3"]);
      var samples = sample_from_concept(concept);
      factor( list_eq(samples, [3, 9]) ? 0 : -Infinity )
      return concept;
    }
    
    print(Enumerate(generative_model));

#### Priors

What if we change the prior probabilities of different hypotheses? 

Suppose the friend who generated the number sequence {16, 8, 2, 64} is a precocious 4th-grader. You know that she has not yet learned the concept of "powers." You believe that it is quite unlikely that she would be thinking about a number category as complex as "powers of 2." How does this change your belief about what number category she is thinking of, given that she generated the number sequence {16, 8, 2, 64}?

#### Sampling method

Notice that there are many different ways your friend could have generated these examples. She could have sampled the numbers randomly, as we assumed in the code boxes before. Or, she could have sampled them in an intentional manner, to purposefully help you understand the number category and not confuse it with a different number category. Or, she could have sampled them in a different intentional manner--to intentionally deceive you into thinking that it is a different number category.

* positive examples
* positive and negative examples
* labelling things in the world

OED? If you have 2 competing hypotheses, what number should you test next?


#### Why is this important?


#### Backup examples?

* word/concept learning (rational rules)
* 

#### Resources/References

* 

