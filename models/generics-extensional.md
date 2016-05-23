---
layout: model
title: Generics (extensional)
model-status: code
model-language: webppl
---

<script src="http://web.stanford.edu/~erindb/webppl-viz/webppl.min.js"></script>  
<link rel="stylesheet" href="http://web.stanford.edu/~erindb/webppl-viz/viz.css">


~~~~
var subset = function(df, key, value){
  return filter(function(d){
    return (d[key]==value)
  },df)
}
var displayERP = function(erp){
  return _.object(map(function(x){return [x, Math.exp(erp.score([],x))]}, 
                      erp.support()
                     )
                 )
}

// discretized range between 0 - 1
var stateBins = [0.01,0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9,1]
var thresholdBins = [0,0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9]
var s1optimality = 5


var world = [
  {name: "fep0", kind: "fep", wings: true, legs: false, claws: false},
  {name: "fep1", kind: "fep", wings: true, legs: false, claws: false},
  {name: "fep2", kind: "fep", wings: true, legs: false, claws: false},
  {name: "fep3", kind: "fep", wings: true, legs: false, claws: false},
  {name: "fep4", kind: "fep", wings: true, legs: false, claws: false},
  {name: "fep5", kind: "fep", wings: false, legs: false, claws: false},
  {name: "fep6", kind: "fep", wings: false, legs: false, claws: false},
  {name: "fep7", kind: "fep", wings: false, legs: false, claws: false},
  {name: "fep8", kind: "fep", wings: false, legs: false, claws: false},
  {name: "fep9", kind: "fep", wings: false, legs: false, claws: false},

  {name: "wug0", kind: "wug", wings: true, legs: true, claws: true},
  {name: "wug1", kind: "wug", wings: true, legs: true, claws: true},
  {name: "wug2", kind: "wug", wings: true, legs: true, claws: true},
  {name: "wug3", kind: "wug", wings: true, legs: true, claws: true},
  {name: "wug4", kind: "wug", wings: true, legs: true, claws: false},
  {name: "wug5", kind: "wug", wings: false, legs: true, claws: false},
  {name: "wug6", kind: "wug", wings: false, legs: true, claws: false},
  {name: "wug7", kind: "wug", wings: false, legs: true, claws: false},
  {name: "wug8", kind: "wug", wings: false, legs: true, claws: false},
  {name: "wug9", kind: "wug", wings: false, legs: true, claws: false},

  {name: "lorch0", kind: "lorch", wings: true, legs: true, claws: true},
  {name: "lorch1", kind: "lorch", wings: true, legs: true, claws: true},
  {name: "lorch2", kind: "lorch", wings: true, legs: true, claws: false},
  {name: "lorch3", kind: "lorch", wings: true, legs: true, claws: false},
  {name: "lorch4", kind: "lorch", wings: true, legs: true, claws: false},
  {name: "lorch5", kind: "lorch", wings: false, legs: true, claws: false},
  {name: "lorch6", kind: "lorch", wings: false, legs: true, claws: false},
  {name: "lorch7", kind: "lorch", wings: false, legs: true, claws: false},
  {name: "lorch8", kind: "lorch", wings: false, legs: true, claws: false},
  {name: "lorch9", kind: "lorch", wings: false, legs: true, claws: false}
]

var allKinds = _.uniq(_.pluck(world, "kind"))
var allProperties = ["wings","legs","claws"]

var hasF = function(thing, property){
  return thing[property]
}

var prevalence = function(kind, property){
  var members = subset(world, "kind", kind)
  var v = filter(function(x){return hasF(x, property)}, members)
  return v.length==0 ? 0.01 : v.length / members.length
}

// init for reduce is 0.001 so that each state has > 0 prob
var prevalencePrior = function(property){
  var p =  map(function(k){return prevalence(k, property)}, allKinds)
  return map(function(s){
    return reduce(function(x, i){
      var k = x==s ? 1 : 0
      return i + k
    }, 0.001, p)
  }, stateBins)
}


var statePrior = function(probs){
  return stateBins[discrete(probs)]
}

var thresholdPrior = function() {
  var threshold = uniformDraw(thresholdBins)
  return threshold
}

var utterancePrior = function() {
  var utterances = ["generic", "null"]  
  //    var utterances = ["generic",
  //                 "generic is false"]  
  return flip(0.5) ? utterances[0] : utterances[1]
}

var meaning = function(utt,state, threshold) {
  return _.isNumber(utt) ? state == utt :
  utt=="generic"? state>threshold :
  utt=="generic is false"? state<=threshold :
  utt=='null'? true:
  utt=='some'? state>0:
  utt=='most'? state>= 0.5:
  utt=='all'? state >= 0.99:
  true
}

var listener0 = cache(function(utterance, threshold, prior) {
  Enumerate(function(){
    var state = statePrior(prior)
    var m = meaning(utterance, state, threshold)
    condition(m)
    return state
  })
})

// listener0("generic", 0.9, prevalencePrior("wings"))

var speaker1 = cache(function(state, threshold, prior) {
  Enumerate(function(){
    var utterance = utterancePrior()
    var L0 = listener0(utterance, threshold, prior)
    factor(s1optimality*L0.score([],state))
    return utterance
  })
})

// speaker1(0.99, 0.9, prevalencePrior("wings"))

var listener1 = function(utterance, prior) {
  Enumerate(function(){
    var state = statePrior(prior)
    var threshold = thresholdPrior()
    var S1 = speaker1(state, threshold, prior)
    factor(S1.score([],utterance))
    return state
  })
}

// listener1("generic", prevalencePrior("wings"))

var speaker2 = function(prev, prior, k, f){
  Enumerate(function(){
    var utterance = utterancePrior()
    var wL1 = listener1(utterance, prior)
    factor(wL1.score([], prev))
    var generic  = k + "s have " + f

    return utterance=="generic" ?
      generic :
    "I don't think " + generic
  })
}

var sayAllOfTheThings =Enumerate(function(){

  var kind = uniformDraw(allKinds)
  var property = uniformDraw(allProperties)
  
  var prior = prevalencePrior(property)
  var prev = prevalence(kind, property)

  var utterance = utterancePrior()
  var wL1 = listener1(utterance, prior)
  factor(wL1.score([], prev))
  
  var generic  = kind + "s have " + property
  return utterance=="generic" ? 
    generic :
  "I don't think " + generic

})

print(sayAllOfTheThings)

var saySomething = function(){
  var kind = uniformDraw(allKinds)
  var property = uniformDraw(allProperties)
  var prior = prevalencePrior(property)
  var prev = prevalence(kind, property)
  return speaker2(prev, prior, kind, property)
}


var results = displayERP(saySomething())
print(results)
~~~~


## Extensional generics with gradable adjectives


~~~~
// time webppl generics-extensional.wppl --require genericsUtils

var subset = function(df, key, value){
  return filter(function(d){
    return (d[key]==value)
  },df)
}


// discretized range between 0 - 1
var stateBins = [0.01,0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9,1]
var thresholdBins = [0,0.1,0.2,0.3,0.4,0.5,0.6,0.7,0.8,0.9]
var s1optimality = 5


var world = [
  {name: "fep0", kind: "fep", wings: true, legs: false, claws: false, height: 0.3},
  {name: "fep1", kind: "fep", wings: true, legs: false, claws: false, height: 0.4},
  {name: "fep2", kind: "fep", wings: true, legs: false, claws: false, height: 0.2},
  {name: "fep3", kind: "fep", wings: true, legs: false, claws: false, height: 0.8},
  {name: "fep4", kind: "fep", wings: true, legs: false, claws: false, height: 0.5},
  {name: "fep5", kind: "fep", wings: false, legs: false, claws: false, height: 0.3},
  {name: "fep6", kind: "fep", wings: false, legs: false, claws: false, height: 0.4},
  {name: "fep7", kind: "fep", wings: false, legs: false, claws: false, height: 0.5},
  {name: "fep8", kind: "fep", wings: false, legs: false, claws: false, height: 0.3},
  {name: "fep9", kind: "fep", wings: false, legs: false, claws: false, height: 0.4},

  {name: "wug0", kind: "wug", wings: true, legs: true, claws: true, height: 0.2},
  {name: "wug1", kind: "wug", wings: true, legs: true, claws: true, height: 0.1},
  {name: "wug2", kind: "wug", wings: true, legs: true, claws: true, height: 0.2},
  {name: "wug3", kind: "wug", wings: true, legs: true, claws: true, height: 0.1},
  {name: "wug4", kind: "wug", wings: true, legs: true, claws: false, height: 0.2},
  {name: "wug5", kind: "wug", wings: false, legs: true, claws: false, height: 0.3},
  {name: "wug6", kind: "wug", wings: false, legs: true, claws: false, height: 0.2},
  {name: "wug7", kind: "wug", wings: false, legs: true, claws: false, height: 0.3},
  {name: "wug8", kind: "wug", wings: false, legs: true, claws: false, height: 0.2},
  {name: "wug9", kind: "wug", wings: false, legs: true, claws: false, height: 0.1},

  {name: "lorch0", kind: "lorch", wings: true, legs: true, claws: true, height: 0.7},
  {name: "lorch1", kind: "lorch", wings: true, legs: true, claws: true, height: 0.6},
  {name: "lorch2", kind: "lorch", wings: true, legs: true, claws: false, height: 0.7},
  {name: "lorch3", kind: "lorch", wings: true, legs: true, claws: false, height: 0.6},
  {name: "lorch4", kind: "lorch", wings: true, legs: true, claws: false, height: 0.7},
  {name: "lorch5", kind: "lorch", wings: false, legs: true, claws: false, height: 0.6},
  {name: "lorch6", kind: "lorch", wings: false, legs: true, claws: false, height: 0.7},
  {name: "lorch7", kind: "lorch", wings: false, legs: true, claws: false, height: 0.6},
  {name: "lorch8", kind: "lorch", wings: false, legs: true, claws: false, height: 0.7},
  {name: "lorch9", kind: "lorch", wings: false, legs: true, claws: false, height: 0.6}
]




var allKinds = _.uniq(_.pluck(world, "kind"))
var propertyDegrees = {"wings":"wings",
                       "legs":"legs",
                       "claws":"claws",
                       "tall":"height"}

var hasF = function(thing, property){
  return thing[property]
}

var prevalence = function(wrld, kind, property){
  var members = subset(wrld, "kind", kind)
  var v = filter(function(x){return hasF(x, property)}, members)
  var mean =  v.length==0 ? 0.01 : sum(_.pluck(v, property)) / members.length
  // round to nearest bin
  return mean == 0.01 ? 0.01 : Math.floor(10*mean)/10
}

// init for reduce is 0.001 so that each state has > 0 prob
var prevalencePrior = function(property, wrld){
  var p =  map(function(k){return prevalence(wrld, k, property)}, allKinds)
  return map(function(s){
    return reduce(function(x, i){
      var k = x==s ? 1 : 0
      return i + k
    }, 0.001, p)
  }, stateBins)
}

var scalePrior = function(property){
  var p = _.pluck(world, property)
  return map(function(s){
    return reduce(function(x, i){
      var k = x==s ? 1 : 0
      return i + k
    }, 0.001, p)
  }, stateBins)
}


var statePrior = function(probs){
  return stateBins[discrete(probs)]
}

var thresholdPrior = function() {
  var threshold = uniformDraw(thresholdBins)
  return threshold
}

var utterancePrior = function(predicate) {
  var utterances = predicate == "height" ? 
      ["tall","null"] :
  ["generic", "null"]  
  //    var utterances = ["generic is true",
  //                 "generic is false"]  
  return flip(0.5) ? utterances[0] : utterances[1]
}

var meaning = function(utt,state, threshold) {
  return _.isNumber(utt) ? state == utt :
  utt=="generic"? state>threshold :
  utt=="tall"? state>threshold :
  utt=="generic is false"? state<=threshold :
  utt=='null'? true:
  utt=='some'? state>0:
  utt=='most'? state>= 0.5:
  utt=='all'? state >= 0.99:
  true
}

var listener0 = cache(function(utterance, threshold, prior) {
  Enumerate(function(){
    var state = statePrior(prior)
    var m = meaning(utterance, state, threshold)
    condition(m)
    return state
  })
})

// listener0("tall", 0.9, scalePrior("height"))

var speaker1 = cache(function(state, threshold, prior, predicate) {
  Enumerate(function(){
    var utterance = utterancePrior(predicate)
    var L0 = listener0(utterance, threshold, prior)
    factor(s1optimality*L0.score([],state))
    return utterance
  })
})

var listener1 = cache(function(utterance, predicate, wrld) {
  Enumerate(function(){
    var prior = predicate == "height" ? scalePrior(predicate) : prevalencePrior(predicate, wrld)
    var state = statePrior(prior)
    var threshold = thresholdPrior()
    var S1 = speaker1(state, threshold, prior, predicate)
    factor(S1.score([],utterance))
    return state
  })
})

// speaker1(1, 0.9, scalePrior("height"), "height")
// listener1("tall", "height")

var worldWithTallness = map(function(thing){
  var tallERP = Enumerate(function(){
    var utterance = utterancePrior("height")
    var tallnessPosterior = listener1(utterance, "height")

    factor(tallnessPosterior.score([], thing["height"]))

    return utterance
  })
  return _.extend(thing, {tall: Math.exp(tallERP.score([], "tall"))})
}, world)

var speaker2 = function(k, f){
  Enumerate(function(){

    var property = f.split(' ')[1]
    var degree = propertyDegrees[property]
    var wrld = _.isNumber(world[0][degree]) ? 
                    worldWithTallness : world

    var prev = prevalence(wrld, k, property)
    var utterance = utterancePrior(property)

    var wL1 = listener1(utterance, property, wrld)
    factor(wL1.score([], prev))

    return utterance=="generic" ?
      k + "s " + f :
    "I don't think " + k + "s " + f

  })
}


print(speaker2("lorch", "are tall"))
~~~~