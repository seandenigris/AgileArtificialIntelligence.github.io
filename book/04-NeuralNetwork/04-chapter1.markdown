
# Neural Networks

The previous chapter covers the design and implementation of an individual neuron. This chapter builds upon the effort initiated in previous chapters by connecting multiple neurons. We provide a complete implementation of a neural network and a backpropagation algorithm, which bring us at the core of the book.

## General architecture

An artificial neural network is a computing system inspired by biological neural networks part of an animal brain. 
An artificial neural network is a collection of connected artificial neurons. Each connection between artificial neurons can transmit a signal from one to another. The artificial neuron that receives the signal can process it, and then signal neurons connected to it. Artificial neural networks are commonly employed to perform particular tasks, including clustering, classification, prediction, and pattern recognition. Similarly than with the perceptron and sigmoid neuron, a neural network acquires knowledge through learning.

![Example of a neural network](04-NeuralNetwork/figures/generalStructure.png){#fig:generalStructure width=300}


Figure @fig:generalStructure shows a simple neural network made of five neurons, three inputs, and two outputs. The left-most column is called the input layer. The input layers simply transmits some values to the hidden layer, without doing anything in particular. In the figure, the input layer is made of three inputs, `x1`, `x2`, and `x3`. The middle of the network contains the hidden layers. The network above contains only one hidden layer, made of three neurons. However, a network may contains several hidden layers. The right-most column of the network is called output layer, and contains of two neurons. 

All values transmitted between neurons are numerical values. The output values, `o1` and `o2` are number ranging between 0 and 1.

The depicted neural network is qualified as _fully-connected_ since each neuron of the hidden layer is connected with _all_ the neurons of the input layer and _all_ the neurons of the output layer. The next sections provide an implementation of abstraction we informally presented.

## Neural layer

We define a layer as a set of neurons. Each layer knows about the preceding layer using the variable `previousLayer` and the following layer using `nextLayer`. The `learningRate` variable refers to the learning rate of the layer. We define the class `NeuronLayer` as follows:

```Smalltalk
Object subclass: #NeuronLayer
	instanceVariableNames: 'previousLayer nextLayer neurons learningRate'
	classVariableNames: ''
	package: 'NeuralNetwork'
```

A layer contains some neurons, kept in the variable `neurons`. We can set the learning rate of a layer as 0.1 per default. A neuron layer may be initialized using the following method:

```Smalltalk
NeuronLayer>>initializeNbOfNeurons: nbOfNeurons nbOfWeights: nbOfWeights using: random
	"Main method to initialize a neuron layer
	nbOfNeurons : number of neurons the layer should be made of
	nbOfWeights : number of weights each neuron should have
	random : a random number generator
	"
	| weights |
	learningRate := 0.1.
	neurons := (1 to: nbOfNeurons) collect: [ :i |
		weights := (1 to: nbOfWeights) collect: [ :ii | random next * 4 - 2 ].
		Neuron new sigmoid; weights: weights; bias: (random next * 4 - 2) ]  

```


The method `initializeNbOfNeurons:nbOfWeights:using:` accepts three arguments. The first one, `nbOfNeurons` is an integer value and represents the number of neurons the layer should contains. The second argument, `nbOfWeights`, is an integer that indicates the number of weights each neuron should have. This number of weights reflects the number of input values the layer is accepting. The last argument, `random`, is a random number generator. As in the previous chapter, using a random number generator is useful to make the behavior deterministic. This random generator will be useful to initialize each individual neuron.

The method first sets the learning rate to `0.1`, and creates `nbOfNeurons` different neurons, each having `nbOfWeights` weight values. Each weight is a random number between -2 and +2. The expression `random next` produces a random number within 0 and 1. Multiplying it by 4 and subtracting 2 produces a value between -2 and +2. Each neuron has a sigmoid activation function thanks to the message `sigmoid`.

Forward feeding the layer is an essential operation. It consists in feeding each neuron and forwarding the values to the next layer. We define the method `feed:` as:

```Smalltalk
NeuronLayer>>feed: someInputValues
	"Feed the neuron layer with some inputs"

	| someOutputs |
	someOutputs := neurons collect: [ :n | n feed: someInputValues ] as: Array.
	^ self isOutputLayer
		ifTrue: [ someOutputs ]
		ifFalse: [ nextLayer feed: someOutputs ]
```

The method invokes `feed:` on each of its neurons (we have have seen the method `Neuron>>feed:` in the previous chapter). The results is then kept as an array. The method then checks if the layer is an output layer. If this is the case, the result of the method is simply the results of each neurons. If the layer is not an output (_i.e.,_ it is  an hidden layer), we feed-forward the computed values to the next layer.


We need to determine if a neuron layer is the output layer or not. We can easily achieve this using the predicate `isOutputLayer`:
```Smalltalk
NeuronLayer>>isOutputLayer
	"Return true of the layer is the output layer (i.e., the last layer in the network)"
	^ self nextLayer isNil
```

We will also need a way to hook layers together:
```Smalltalk
NeuronLayer>>nextLayer: aLayer
	"Set the next layer"
	nextLayer := aLayer
```

To access the next layer, we need the method:

```Smalltalk
NeuronLayer>>nextLayer
	"Return the next layer connected to me"
	^ nextLayer
```

Similarly, we need a way to set and access the previous layer:

```Smalltalk
NeuronLayer>>previousLayer: aLayer
	"Set the previous layer"
	previousLayer := aLayer
```

```Smalltalk
NeuronLayer>>previousLayer
	"Return the previous layer connected to me"
	^ previousLayer
```

Neurons for a given layer needs to be accessed:
```Smalltalk
NeuronLayer>>neurons
	"Return the neurons I am composed of"
	^ neurons
```


We have now defined most of the `NeuronLayer` class. We can now begin testing the class:
```Smalltalk
TestCase subclass: #NeuronLayerTest
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'NeuralNetwork'
```

A simple test may be:

```Smalltalk
NeuronLayerTest>>testBasic
	| nl result r |
	r := Random seed: 42.
	nl := NeuronLayer new.
	nl initializeNbOfNeurons: 3 nbOfWeights: 4 using: r.

	self assert: nl isOutputLayer.

	result := nl feed: { 1 . 2 . 3 . 4 }.
	self assert: result size equals: 3.
 	self assert: result closeTo: #(0.037000501309787576 0.9051275824569505 0.9815269659126287)
```

The method `testBasic` create a new neuron layer, composed of 3 neurons, each having 4 weights and 1 bias. The weights and biases are initialized using the random number generator `r`.

We can also build up a chain of layers and see how they perform:
```Smalltalk
NeuronLayerTest>>testOutputLayer
	| nl1 nl2 result random |
	random := Random seed: 42.
	nl1 := NeuronLayer new.
	nl1 initializeNbOfNeurons: 3 nbOfWeights: 4 using: random.

	nl2 := NeuronLayer new.
	nl2 initializeNbOfNeurons: 4 nbOfWeights: 3 using: random.
	nl1 nextLayer: nl2.

	self deny: nl1 isOutputLayer.
	self assert: nl2 isOutputLayer.

	result := nl1 feed: { 1 . 2 . 3 . 4 }.
	"Since nl2 has 4 neurons, we will obtain 4 outputs"
	self assert: result size equals: 4.
 	self assert: result closeTo: #(0.030894022895187584 0.9220488835263312 0.5200462953493653 0.20276557516858304)
```

We can now wrap a chain of layers into a neural network.

## Neural network

We will represent a neural network as an instance of the class `NNetwork`:

```Smalltalk
Object subclass: #NNetwork
	instanceVariableNames: 'layers errors'
	classVariableNames: ''
	package: 'NeuralNetwork'
```

We define a neural network simply as a container of layers. We also add an `errors` instance variable that will be useful to trace the evolution of error during the learning phase.

The initialization of a network is done through the method `initialize`:
```Smalltalk
NNetwork>>initialize
	super initialize.
	layers := OrderedCollection new.
	errors := OrderedCollection new
```

Both the `layers` and `errors` instance variables are initialized with an empty collection. The variable `layers` will contains instances of the class `NeuronLayer` and `errors` will contains numerical values, representing the errors during the training process. 

Adding a layer is simply done through the method `addLayer:`, which takes a layer as argument:
```Smalltalk
NNetwork>>addLayer: aNeuronLayer
	"Add a neural layer. The added layer is linked to the already added layers."
	layers ifNotEmpty: [ 
		aNeuronLayer previousLayer: layers last.
		layers last nextLayer: aNeuronLayer ].
	layers add: aNeuronLayer.
```

Layers are linked to each other. When a layer is added, it is linked to the previous layer and that layer is linked to the added layer.

Feeding a neural network is simply feeding the first hidden layer:

```Smalltalk
NNetwork>>feed: someInputValues
    "Feed the first layer with the provided inputs"
    ^ layers first feed: someInputValues
```

We need a way to easily create a neural network. In case we wish to build a network with one hidden layer and one output layer, we can define the following method:

```Smalltalk
NNetwork>>configure: nbOfInputs hidden: nbOfNeurons nbOfOutputs: nbOfOutput
	"Configure the network with the given parameters
	The network has only one hidden layer"
	| random |
	random := Random seed: 42.
	self addLayer: (NeuronLayer new initializeNbOfNeurons: nbOfNeurons nbOfWeights: nbOfInputs using: random).
	self addLayer: (NeuronLayer new initializeNbOfNeurons: nbOfOutput nbOfWeights: nbOfNeurons using: random).
```

In case we wish to have two hidden layers and one output layer:
```Smalltalk
NNetwork>>configure: nbOfInputs hidden: nbOfNeurons1 hidden: nbOfNeurons2 nbOfOutputs: nbOfOutput
	"Configure the network with the given parameters
	The network has only one hidden layer"
	| random |
	random := Random seed: 42.
	self addLayer: (NeuronLayer new initializeNbOfNeurons: nbOfNeurons1 nbOfWeights: nbOfInputs using: random).
	self addLayer: (NeuronLayer new initializeNbOfNeurons: nbOfNeurons2 nbOfWeights: nbOfNeurons1 using: random).
	self addLayer: (NeuronLayer new initializeNbOfNeurons: nbOfOutput nbOfWeights: nbOfNeurons2 using: random).
```


We can now tests our network implementation:

```Smalltalk
TestCase subclass: #NNetworkTest
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'NeuralNetwork'
```


```Smalltalk
NNetworkTest>>testBasic
    | n |
    n := NNetwork new.
    n configure: 2 hidden: 2 nbOfOutputs: 1.
    self assert: (n feed: { 1 . 3 }) closeTo: #(0.6745388083637035)
```

As you can see, `testBasic` is rather simplistic. It builds a simple network with two inputs, one hidden layer made of 2 neurons, and an output layer with only one neuron, and run the forward feeding. 

So far, our network is pretty useless. The next section covers the learning mechanism for neural networks.

## Backpropagation

Backpropagation is an algorithm commonly employed to train neural networks. By training, we mean to make the network identify some patterns.

So far, we built a network as a set of neurons, each being initialized with random weights and random biases. Conceptually, backpropagation is an algorithm for supervised learning of gradient descent (next chapters will cover this terminology). In practice, this algorithm will find adequate weights and biases to identify patterns from the input values. This section focuses on informally presenting the algorithm and providing an implementation of it. Subsequent chapters will provide a theoretical foundation of the algorithm.

The backpropagation algorithm is composed of three steps:

1. _Forward feeding the inputs_. We first activate each neurons of our network to make the network produce an output.
1. _Backward propagating the errors through the network_. The output produced in the previous step has to be contrasted with the actual training dataset. We can therefore compute an error, which indicates how far our network is from correctly predicting the training set. 
1. _Updating the neurons weights and biases_. From the error computed in the previous step, we adequately adjust each neuron weights and bias to, hopefully, reduce the error made by the network.

The first step is mostly implemented by the method `NNetwork>>feed:`, however, we need to slightly improve the class `Neuron` to actually remember the produced output. During the forward feeding (_i.e.,_ when the method `feed:` is called), an output is produced by each neuron. This output has to be compared with an expected output. Making the network learn is based on the difference between the actual output of a neuron and the expected output. Each neuron has therefore to keep a reference to the output. 

We add two variables, `delta` and `output`, to the class `Neuron`. So, our new definition of `Neuron` is:

```Smalltalk
Object subclass: #Neuron
	instanceVariableNames: 'weights bias delta output activationFunction'
	classVariableNames: ''
	package: 'NeuralNetwork'
```

The delta value has to be accessible:

```Smalltalk
Neuron>>delta
	"Return the delta value computed when propagating the error"
	^ delta
```

We also need to rewrite the method `feed:` in the class `Neuron` to remember the output value:
```Smalltalk
Neuron>>feed: inputs
	| z |
	z := (inputs with: weights collect: [ :x :w | x * w ]) sum + bias.
	output := activationFunction eval: z.
	^ output    
```

We also need to access the output value for a given neuron:
```Smalltalk
Neuron>>output
	"Return the output value, previous computed when doing a feed:"
	^ output
```


At that stage, it is important to run the unit tests we have previously defined. In particular, we need to make sure that the small changes we have defined on the class `Neuron` does not break any invariant. We are now done with the first phase of the backpropagation. 

The second step of the backpropagation consists in propagating the errors computed at the output layer back in the network. We define the following method:

```Smalltalk
NNetwork>>backwardPropagateError: expected
	self outputLayer backwardPropagateError: expected
```

And the following helper method:
```Smalltalk
NNetwork>>outputLayer
	"Return the output layer, which is also the last layer"
	^ layers last
```

We add the method `backwardPropagateError:` to back propagate the error from the output layer:

```Smalltalk
NeuronLayer>>backwardPropagateError: expected
	"This is a recursive method. The back propagation begins with the output layer (i.e., the last layer)"
	"We are in the output layer"
	neurons with: expected do: [ :neuron :exp | 
		| theError |
		theError := exp - neuron output.
		neuron adjustDeltaWith: theError ].

	"We iterate"
	self previousLayer notNil
		ifTrue: [
			 self previousLayer backwardPropagateError ].
```

The method `backwardPropagateError:` takes as argument the expected output values. It then computes the error for each neuron in the output layers and call the method `adjustDeltaWith:`. We will soon see this method.

Once the neuron in the output layer have their delta value adjusted, previous layers have to be recursively updated. The method `backwardPropagateError` exactly implements this behavior:

```Smalltalk
NeuronLayer>>backwardPropagateError
	"This is a recursive method. The back propagation begins with the output layer (i.e., the last layer)"

	"We are in an hidden layer"
	neurons doWithIndex: [ :neuron :j |
		| theError |
		theError := 0.0.
		self nextLayer neurons do: [ :nextNeuron |
			theError := theError + ((nextNeuron weights at: j) * nextNeuron delta)
		].
		neuron adjustDeltaWith: theError
	].

	self previousLayer notNil
		ifTrue: [
			 self previousLayer backwardPropagateError ].
```

The recursion end on the first hidden layer, which is the layer with no previous layer. Note that we do not explicitly model the input layer. 

We also need the following helper method on the class `Neuron`:
```Smalltalk
Neuron>>adjustDeltaWith: anError
	delta := anError * (activationFunction derivative: output)
```

We are now done with the second phase. Only the third phase remains to be implemented. 

Luckily, the third phase is rather simple. We recursively update the weights and biases based on the delta computed in the previous step. The main method is `updateWeight:`:

```Smalltalk
NNetwork>>updateWeight: initialInputs
	"Update the weights of the neurons using the initial inputs"
	layers first updateWeight: initialInputs
```

This method simply invokes `updateWeight:` on each first hidden layer: 

```Smalltalk
NeuronLayer>>updateWeight: initialInputs
	"Update the weights of the neuron based on the set of initial input. This method assumes that the receiver of the message invoking that method is the first hidden layer."
	| inputs |
	inputs := initialInputs.
		
	neurons do: [ :n |
		n adjustWeightWithInput: inputs learningRate: learningRate.
		n adjustBiasUsingLearningRate: learningRate ].
	
	self nextLayer ifNotNil: [ 
		self nextLayer updateWeight ]
```

The recursion happens in the method `updateWeight`:

```Smalltalk
NeuronLayer>>updateWeight
	"Update the weights of the neuron based on the set of initial input. This method assumes that the receiver of the message invoking that method is the first hidden layer.
	We are now in the second hidden layers or in the output layer" 
	| inputs |
	inputs := self previousLayer neurons collect: #output.
		
	self updateWeight: inputs
```

And we need the following methods to update a neuron's weights:
```Smalltalk
Neuron>>adjustWeightWithInput: inputs learningRate: learningRate
	inputs withIndexDo: [ :anInput :index | 
		weights at: index put: ((weights at: index) + (learningRate * delta * anInput)) ]
```

We also need to update the bias:
```Smalltalk
Neuron>>adjustBiasUsingLearningRate: learningRate
	bias := bias + (learningRate * delta)
```

This ends the third and last phase of the backpropagation algorithm. We are now ready to hook the backpropagation phases all together:

```Smalltalk
NNetwork>>train: someInputs desiredOutputs: desiredOutputs
	"Train the neural network with a set of inputs and some expected output"
	| realOutputs t |
	realOutputs := self feed: someInputs.
	t := (1 to: desiredOutputs size) collect: 
			[ :i | ((desiredOutputs at: i) - (realOutputs at: i)) raisedTo: 2 ].
	self backwardPropagateError: desiredOutputs.
	self updateWeight: someInputs.
```

And Voila! We have implemented the necessary to train a neural network. 

We can now test our network with the XOR example:
```Smalltalk
NNetworkTest>>testXOR
	| n |
	n := NNetwork new.
	n configure: 2 hidden: 3 nbOfOutputs: 1.

	10000 timesRepeat: [ 
		n train: { 0 . 0 } desiredOutputs: { 0 }.	
		n train: { 0 . 1 } desiredOutputs: { 1 }.
		n train: { 1 . 0 } desiredOutputs: { 1 }.
		n train: { 1 . 1 } desiredOutputs: { 0 }.
	].

	self assert: (n feed: { 0 . 0 }) first < 0.1.
	self assert: (n feed: { 0 . 1 }) first > 0.9.
	self assert: (n feed: { 1 . 0 }) first > 0.9.
	self assert: (n feed: { 1 . 1 }) first < 0.1.
```

If you try to decrease the `10000` to a low value, `10` for example, the network does not receive enough training and the test ultimately fails.

## What have we seen in this chapter

This chapter covers the following topics:

- _Presented the general architecture of a fully-connected network_. This architecture drove our implementation effort.
- _Implemented a neural network library_. We build a small API to build neural networks. 
- _Implemented the backpropagation algorithm_. Making neural network learn is a fundamental operation to give values to a network. A properly trained network is able to identify patterns. This chapter ends with a trivial example, the XOR logical gate. The coming chapters will see real and representative examples.