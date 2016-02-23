---
layout: model
title: Social Construction of Value
model-language: webppl
---

Suppose there are $M$ restaurants, which generate noisy reward signals $r_j \in \{0, 1\}$. Each agent $a_i$ in the population assigns some subjective utility $u_j$ to each restaurant $j$, such that $u_j = P(r_j = 1)$. These subjective utilities are drawn from a shared normal distribution, so all agents have relatively similar utilities functions. We will model a particular agent, Alice, as she infers her own utility function. She uses two sources of information. First, Alice assumes that all other agents know their own utility and decide which restaurants to visit according to a soft-max rule. Second, when Alice chooses according to her beliefs about her own utility, she observes a noisy reward signal from her true utility function. 

~~~~
///fold:
var condition = function(x){
  factor(x ? 0 : -Infinity);
};

var normalize = function(xs) {
  var Z = sum(xs);
  return map(function(x) {
    return x / Z;
  }, xs);
};

var normalizeVals = function(agentVals){
  var arr = _.values(agentVals);
  return normalize(arr);
};
///

// Alice doesn't know her own utilities
var uninformedPrior = function() {
  return {
    "Taco Town" : uniform(0,1),
    "Burger Barn" : uniform(0,1),
    "Stirfry Shack" : uniform(0,1)
  };
};

// All agents in the population have utilities drawn from a shared prior
var truePrior = function() {
  return {
    "Taco Town" : gaussian(.5, .1),
    "Burger Barn" : gaussian(.25, .1),
    "Stirfry Shack" : gaussian(.75, .1)
  };
};

// Create population of agents
var initializeAgents = function(numAgents) {
  return repeat(numAgents, function() {
    return {
      utility : truePrior()
    };
  });
};

// Sample noisy reward signal for r_j based on agent's utility
var observe = function(utility, restaurant) {
  return flip(utility[restaurant]) ? "good" : "bad";
};

// Each agent soft-maxes utility
var makeChoice = function(utility) {
  var restaurants = _.keys(utility);
  var choice = categorical(normalizeVals(utility), restaurants);
  var outcome = observe(utility, choice);
  return {choice: choice, outcome: outcome};
};

// How many agents do we want in our population?
var numAgents = 100;

// Fixed population of agents
var agents = initializeAgents(numAgents);
var otherChoices = map(function(a){return makeChoice(a.utility);}, agents);

// Fixed utility for main agent
var trueUtility = truePrior();
print(trueUtility);

var model = function() {
  var self = {
    inferredUtility : uninformedPrior(),
    trueUtility : trueUtility
  };

  // Take true reward signal into account
  var rewardSignal = observe(self.trueUtility,
                             makeChoice(self.inferredUtility)["choice"]);
  factor(rewardSignal === "good" ? 0 : -Infinity);

  // Try to minimize surprisal w.r.t. others' choices
  var surprisals = map(function(otherChoice) {
    return Math.log(self.inferredUtility[otherChoice.choice]
                    / sum(_.values(self.inferredUtility)));
  }, otherChoices);
  factor(sum(surprisals));

  return self.inferredUtility;
};

var results = MCMC(model, {samples : 10000, burn : 100});
print("done");
vizPrint(results);
~~~~

<!--
We see that after many time steps, our rational agent learns the true value of each location.

Next, we add social influence into our utility function. An agent has relationships with others, represented by a vector of weights $w$. If the weight $w_i$ corresponding to a particular other agent is positive, they will slightly increase the value they assign to a restaurant when that agent has a positive experience there, and slightly decrease it when that agent has a negative experience. For now, we do not allow agents to change their relationships over time. We simulate three agents simultaneously, all of whom have positive relationships with one another.

~~~~
///fold:
var condition = function(x){
  factor(x ? 0 : -Infinity);
};

var mean = function(thunk){
  return expectation(Enumerate(thunk), function(v){return v;});
};

var uniformDraw = function (xs) {
  return xs[randomInteger(xs.length)];
};

var normalize = function(arr) {
  var s = reduce(function(memo, num){ return memo + num; }, 0, arr);
  return map(function(val) {return val/s; }, arr);
};

var expectedVals = function(agentPrior){
  var outs = map(function(key) { 
    return mean(function(){
      return sample(agentPrior[key]);
    });
  }, _.keys(agentPrior));
  return normalize(outs);
};

var getOtherAgents = function(agentNames, name) {
  return _.without(agentNames, name);
}; 

///

var restaurants = ["Taco Town", "Burger Barn", "Stirfry Shack"];
var goodnessWeights = [.1,.2,.3,.4,.5,.6,.7,.8,.9];

var restaurantPrior = map(function(restaurant) {
  return Enumerate(function(v){
    return uniformDraw(goodnessWeights);
  });
}, restaurants);

// initialize with uniform prior and neutral relationships ([.5,.5])
var createAgent = function(name, agentNames) {
  var otherAgents = getOtherAgents(name, agentNames);
  var otherWeights = repeat(otherAgents.length, function(){return .5;});
  var relationships = _.object(otherAgents, otherWeights);
  return {values: _.object(restaurants, restaurantPrior), 
          relationships: relationships};
};

var initializeAgents = function(agentNames) {
  var agentProperties = map(function(agent) {
    return createAgent(agent, agentNames);
  }, agentNames);
  return _.object(agentNames, agentProperties);
};

var observe = function(restaurant) {
  return flip() ? "good" : "bad";
};

var makeChoices = function(agents) {
  return mapObject(function(agentName, agentProps) {
    var restaurants = _.keys(agentProps.values);
    var choice = categorical(expectedVals(agentProps.values), restaurants);
    var outcome = observe(choice);
    return {choice: choice, outcome: outcome};
  }, agents);
};

// Adjust relationships based on shared experience
var updateRelationships = function(agentProps, info){
  return info.relationships;
};

// Sample outcomes of other agents propotionally to strength of 
// relationship
var sampleOtherOutcomes = function(restaurant, info) {
  var relevantOthers = filter(function(key) {
    return (info.socialObs[key].choice === restaurant ?
            flip(info.relationships[key]) :
            false);
  }, _.keys(info.socialObs));

  return map(function(person){
    return info.socialObs[person].outcome;
  }, relevantOthers);
  
};

// Adjust values proportionally to utility of each restaurant
var updateValues = function(agentProps, info){
  return mapObject(function(restaurant, value) {
    return Enumerate(function() {
      var possibleWeight = sample(value);    
      var otherOutcomes = sampleOtherOutcomes(restaurant, info);      
      var trueOutcomeSet = (restaurant === info.ownObs.choice ? 
                          append(otherOutcomes, info.ownObs.outcome) : 
                          otherOutcomes);
      var simulatedOutcomeSet = repeat(trueOutcomeSet.length, function(){
        return flip(possibleWeight) ? "good" : "bad";
      });
      condition(_.isEqual(trueOutcomeSet, simulatedOutcomeSet));
      return possibleWeight;
    });
  }, agentProps.values);
};

// Loop through each agent, update their values and their relationships
var updateBeliefs = function(agents, allAgentsChoices) {
  return mapObject(function(agentName, agentProps) {
    var info = {ownObs : allAgentsChoices[agentName],
                relationships: agentProps.relationships,
                socialObs: _.omit(allAgentsChoices, agentName)};
    var newValues = updateValues(agentProps, info);
    var newRelationships = updateRelationships(agentProps, info);
    return {values: newValues, relationships: newRelationships};
  }, agents);
};

// Advance simulation by one timeStep:
// 1) Each agent chooses a restaurant and observes how good it is
// 2) Agents update their beliefs, using own observation and social cues
var timeStep = function(agents, remainingIterations){
  if (remainingIterations == 0) {
    return agents;
  } else {
    var allAgentsChoices = makeChoices(agents);
    var updatedAgents = updateBeliefs(agents, allAgentsChoices);
    return timeStep(updatedAgents, remainingIterations - 1);
  }
};

var agents = initializeAgents(["Alice", "Bob", "Carol"]);
var simResults = timeStep(agents, 500);
print("Alice's values:");
print(simResults['Alice'].values['Taco Town']);
print(simResults['Alice'].values['Burger Barn']);
print(simResults['Alice'].values['Stirfry Shack']);

print("Bob's values:");
print(simResults['Bob'].values['Taco Town']);
print(simResults['Bob'].values['Burger Barn']);
print(simResults['Bob'].values['Stirfry Shack']);

print("Carol's values:");
print(simResults['Carol'].values['Taco Town']);
print(simResults['Carol'].values['Burger Barn']);
print(simResults['Carol'].values['Stirfry Shack']);
~~~~

We see that after enough steps, the three friends coordinate their values on the same location. If we turn the social relationship weights down to 0, we recover the phenomena we observed earlier: each agent evolves independently, and we do not see any coordination.

  Next, we allow agents to update their relationships based on common experiences. If one agent highly values the restaurant that another agent chooses, they will increase their opinion of that agent, allowing that agent's experiences to influence their utility more strongly in the future. On the other hand, if one agent assigns low value to the restaurant that another agent chooses, they will decrease their opinion of that agent, so that they'll be less influenced by that agent in the future. This relationship update rule is meant to reflect a basic "like attracts like" social dynamic. We tend to associate with people who have similar values (as evidenced through their choices), and therefore allow their experiences to influence our values more strongly. For example, if I have a friend who shares my taste in music, I'm more likely to take their recommendation of a new album than the recommendation of a stranger.

~~~~

var restaurants = ["Taco Town", "Burger Barn", "Stirfry Shack"];
var restaurantPrior = Enumerate(function(){
  return uniformDraw(restaurants);
});

var agentList = ["Alice", "Bob", "Carol"];
var getOtherAgents = function(agentList, name) {
  return _.without(agentList, name);
};

var indWeight = 2;
var initSocWeight = .5;

// initialize with uniform prior and neutral relationships ([1,1])
var createAgent = function(name) {
  var otherAgents = getOtherAgents(agentList, name);
  var relationships = _.object(otherAgents, [initSocWeight,initSocWeight]);
  return {
    values: restaurantPrior,
    relationships: relationships
  };
};

var initializeAgents = function() {
  var agentProperties = map(function(agent) {
    return createAgent(agent);
  }, agentList);
  return _.object(agentList, agentProperties);
};

var observe = function(restaurant) {
  return flip() ? "good" : "bad";
};

var makeChoices = function(agents) {
  return mapObject(function(agentName, agentProps) {
    var restaurantChoice = sample(agentProps.values);
    var outcome = observe(restaurantChoice);
    return {choice: restaurantChoice, outcome: outcome};
  }, agents);
};

var calcIndividualUtil = function(possibility, info) {
  var weight = info.ownObs.outcome === "good" ? indWeight : -indWeight;
  return info.ownObs.choice === possibility ? weight : 0;
};

// possibility has more utility if another agent liked it, weighted by
// strength of relationship to that person
var calcSocialUtil = function(possibility, info) {
  return sum(_.values(mapObject(function(name, obs) {
    var relWeight = info.relationships[name];
    var weight = obs.outcome === "good" ? relWeight : -relWeight;
    return obs.choice === possibility ? weight : 0;
  }, info.socialObs)));
};

// Utility can come from own observation, as well as some
// intrinsic benefit to doing what one's friends are doing.
// only get input on individual utility about the place you chose
var calcUtility = function(possibility, info) {
  var indUtil = calcIndividualUtil(possibility, info);
  var pairwiseSocialUtil = calcSocialUtil(possibility, info);
  return indUtil + pairwiseSocialUtil;
};

// Adjust relationships based on shared experience:
//   if agent 1 highly values the restaurant agent 2 chooses,
//   relationship gets stronger
var updateRelationships = function(agentProps, info){
  return mapObject(function(otherName, otherWeight) {
    var otherObs = info.socialObs[otherName];
    var choiceVal = Math.exp(agentProps.values.score([], otherObs.choice));
    var newWeight = otherWeight * (choiceVal + 2/3);
    return newWeight > 1 ? 1 : newWeight;
  }, info.relationships);
};

// Adjust values proportionally to utility of each restaurant
var updateValues = function(agentProps, info){
  return Enumerate(function() {
    var possibility = sample(agentProps.values);
    factor(calcUtility(possibility, info))
    return possibility;});
};

// Loop through each agent, update their values and their relationships
var updateBeliefs = function(agents, allAgentsChoices) {
  return mapObject(function(agentName, agentProps) {
    var info = {ownObs : allAgentsChoices[agentName],
                relationships: agentProps.relationships,
                socialObs: _.omit(allAgentsChoices, agentName)};
    var newValues = updateValues(agentProps, info);
    var newRelationships = updateRelationships(agentProps, info);
    return {values: newValues, relationships: newRelationships};
  }, agents);
};

// Advance simulation by one timeStep:
// 1) Each agent chooses a restaurant and observes how good it is
// 2) Agents update their beliefs, using own observation and social cues
var timeStep = function(agents, remainingIterations){
  //  print(agents);
  if (remainingIterations == 0) {
    return agents;
  } else {
    var allAgentsChoices = makeChoices(agents);
    var updatedAgents = updateBeliefs(agents, allAgentsChoices);
    return timeStep(updatedAgents, remainingIterations - 1);
  }
};

var agents = initializeAgents();
var simResults = timeStep(agents, 100);

print("Alice's values:");
print(simResults['Alice'].values);
print(simResults['Alice'].relationships);

print("Bob's values:");
print(simResults['Bob'].values);
print(simResults['Bob'].relationships);

print("Carol's values:");
print(simResults['Carol'].values);
print(simResults['Carol'].relationships);

~~~~

We see that several different equilibria can form, depending on the exact sequence of early observations. The most obvious is what happened above: all three agents converge on a single restaurant, and all have strong pairwise relationships. Another possibility is two agents with mutually strong relationships who have converged on the same restaurant, while the third agent has weak social bonds and prefers a different restaurant. A third possibility is all three agents prefering different restaurants and mutually disregarding one another. 
-->
