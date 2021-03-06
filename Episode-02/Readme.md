# Decision Tree Algorithm

### Understanding the Algorithm: Simple Implementation Code Example 

The Python code for a Decision-Tree ([*decisiontreee.py*](/decisiontree.py?raw=true)) is a good example to learn how a basic machine learning algorithm works. The [*inputdata.py*](/inputdata.py?raw=true "Input Data") is used by the **createTree algorithm** to generate a simple decision tree that can be used for prediction purposes. The data and code presented here are a modified version of the [original code](https://github.com/pbharrin/machinelearninginaction3x/blob/master/Ch03/trees.py) given by Peter Harrington in Chapter 3 of his book: **Machine Learning in Action**.

In this discussion we shall take a deep dive into how the algorithm runs and try to understand how the [Python dict tree](/output.tree?raw=true "Decision Tree") structure depicted in the graph below is generated recursively. 




## Contents:
- Historical Note
- Running Create Tree Program
   - Program Input: Dataset Description
   - Program Output: Decision Tree dict
- Creating Decsion Tree: How machine learning algorithm works
   - Part 1: Calculating Entropy
   - Part 2: Choosing Best Feature To Branch Tree
   - Part 3: Looping and Tree Building
- Classification: Making Prediction

## Historical Note
Machine learning decision trees were first formalized by [John Ross Quinlan](https://en.wikipedia.org/wiki/Ross_Quinlan) during the years 1982-1985. Along linear and logistic regression, decision trees (with their modern version of random forests) are considered the easiest and the most commonly used machine learning algorithms. 

## Running Create Tree Program
To execute the main function you can just run the [*decisiontreee.py*](/decisiontree.py?raw=true "Decision Tree") program using a call to Python via the command line:

      $ python decisiontree.py
      
Or you can execute it from within the Python shell using the following commands: 

      $ python 
      >>> import decisiontree
      >>> import inputdata
      >>> dataset, features = inputdata.createDataset()
      >>> tree = decisiontree.createTree(dataset, features)
      >>> decisiontree.pprintTree(tree)
      
### Program Input: Dataset Description
The *createDataset* function generates sample records for 7 species: 2 fish, 3 not fish and 2 maybe. The dataset contains two input feature columns: *non-surfacing* and *flippers* and a 3rd prediction label column: *isfish*. (Note: *non-surfacing* means the specie can survive without coming to the surface of the water) 

     dataset = [[1, 1, 'yes'], [1, 1, 'yes'], 
                [1, 0, 'no'],  [0, 1, 'no'],  [0, 1, 'no'], 
                [1, 1, 'maybe'], [0, 0, 'maybe']]
     
     non-surfacing     flippers      isfish  
    ===============   ==========    ========
       True(1)         True(1)        yes
       True(1)         True(1)        yes
       
       True(1)         False(0)       no
       False(0)        True(1)        no
       False(0)        True(1)        no
       
       True(1)         True(1)        maybe
       False(1)        False(0)       maybe

### Program Output: Decision Tree dict
The machine learning program recursively builds a Python dictionary which represents the graph of a tree. If we run the **createTree** function with the input dataset we get the following pretty print taken from the [*output.txt*](/output.txt?raw=true) file, which is identical to the tree diagram shown above:

    # output as dict
    {'non-surfacing': {0: {'flippers': {0: 'maybe', 1: 'no'}}, 
                       1: {'flippers': {0: 'no', 1: 'yes'}}}} 
     
     non-surfacing: 
      | 0: 
      | | flippers: 
      | | | 0: maybe
      | | | 1: no
      | | 
      | 
      | 1: 
      | | flippers: 
      | | | 0: no
      | | | 1: yes
      | | 
      |    
  

## Creating Decsion Tree: How machine learning algorithm works
The **createTree algorithm** builds a decision tree recursively. The algorithm is composed of 3 main components:

 - Entropy test to compare information gain in a given data pattern
 - Dataset spliting performed according to the entropy test
 - Growing dict structure to represents the decision tree

In each recursive iteration of the *createTree* function, the algorithm searches for patterns in its given dataset by comparing information gain for each feature. It peforms an entropy test that discriminates between features and then chooses the one that can best split the given dataset into sub-datasets. The algorithm then calls itself recursively to do the pattern search, entropy test and spliting on the new sub-datasets. Recursion terminates and the tree branch is rolled when there are no more features to split in the sub-dataset or when all the prediction labels are the same.

The rest of the article will take a deeper look at the Python code that implements the algorithm. The code looks deceivingly simple but to understand how things actually work requires a deeper understanding of recursion, Python's list spliting, as well as understanding how Entropy works. 

## Part 1: Calculating Entropy
The code for calculating Entropy for the labels in a given dataset:

       def calculateEntropy(dataset):
           counter= defaultdict(int)   # number of unique labels and their frequency
     (1)   for record in dataset:      
               label = record[-1]      # always assuming last column is the label column 
               counter[label] += 1
           entropy = 0.0
     (2)   for key in counter:
               probability = counter[key]/len(dataset)       # len(dataSet) = total number of entries 
               entropy -= probability * log(probability,2)   # log base 2
           return entropy 

There are two main for-loops in the function. Loop (1) calculates the frequency of each label and Loop (2) calculates Entropy for those labels using the below formula:
 
&nbsp; &nbsp; &nbsp; H(X) = - &sum;<sub>i</sub> P<sub>X</sub>(x<sub>i</sub>) * log<sub>2</sub> (P<sub>X</sub>(x<sub>i</sub>))

To compute Entropy H(X) for a given variable X with possible values x<sub>i</sub> we take the negative sum of the product of probability P<sub>X</sub>(x<sub>i</sub>) with the log base 2 of that same probability value. 

Because Entropy uses probability in its formula, it measures **certain disorder** in the data, the greater the Entropy the higher is the disorder in the data. This means, that when a certain dataset includes a high-probability event (data patterns occuring frequently, that event carries less Entropy than when that data pattern occurs less often (low-probability event). For example, if we take the all seven records and measure baseEntropy for all labels we get: 
      
      # labels = [yes,yes,no,no,no,maybe,maybe]
      $ python
      >>> from decisiontree import *
      >>> dataset, features = createDataset()
      >>> baseEntropy = calculateEntropy(dataset)
      >>> print (baseEntropy)
      1.5566567074628228
      
If we drop the last two records and their corresponding *maybe* labels then Entropy decreases because the sample data has lost some variety and thus became more ordered.

      #  labels = [yes,yes,no,no,no]
      >>> entropy = calculateEntropy(dataset[:-2])
      >>> print (entropy)
      0.9709505944546686

If we compute Entropy for a any one record (drop all other 6 records) we get zero Entropy value because there is no variety at all in the data:

      # labels = [yes]
      >>> entropy = calculateEntropy(dataset[:-6])
      >>> print (entropy)
      0.0

## Part 2: Choosing Best Feature to Branch Tree
 
Although, the previous code is used to calculate Entropy for a list of labels, to build the tree top down we need to do more and discriminate between various features and test for their ability to reduce Entropy. 

      def chooseBestFeatureToSplit(dataset):
          baseEntropy = calculateEntropy(dataset)
          bestInfoGain = 0.0; bestFeature = -1

          numFeat = len(dataset[0]) - 1          # do not include last label column     
          for indx in range(numFeat):            # iterate over all the features index
              featValues = {record[indx] for record in dataset}     # put feature values into a set
              featEntropy = 0.0
              for value in featValues:
                  subDataset = splitDataset(dataset, indx, value)      # split on feature index and value
                  probability = len(subDataset)/float(len(dataset))
                  featEntropy += probability * calculateEntropy(subDataset) # sum Entropy for all feature vals
              infoGain = baseEntropy - featEntropy    # calculate the info gain; ie reduction in Entropy
              if infoGain > bestInfoGain:             # compare this to the best gain so far
                  bestInfoGain = infoGain             # if better than current best, set it to best
                  bestFeature = indx
          return bestFeature                          # return an best feature index


The code above is used to find the feature that can produce the highest information gain (across all its feature values) for the given set of labels. This means, that the choosen feature can split the data uniformly and produce the "purest" set of terminating nodes.

Initially the function calculates the baseEntropy for all labels of the dataset (which will be used to compare information gain). Then for each feature it calculates the featEntropy (feature Entropy) by dividing the dataset into various subgroup. It then calculates the sum of all subgroup Entropies for all feature values. In information theory, featEntropy is often referred to as the "information content of a message". In our case it is calculated using the following:

&nbsp; &nbsp; &nbsp; featEntrop = - Px<sub>0</sub> * log<sub>2</sub> (Px<sub>0</sub>) - Px<sub>1</sub> * log<sub>2</sub> (Px<sub>1</sub>)

### Choosing Root Node
The code listing below shows the featEntropy for each label group when given the full 7 data records. The calculation and spliting show how the node *non-surfacing* is choosen as the top root node of the tree. 

    Dataset: [[1, 1, 'yes'], [1, 1, 'yes'], 
              [1, 0, 'no'], [0, 1, 'no'], [0, 1, 'no'], 
              [1, 1, 'maybe'], [0, 0, 'maybe']]

    labels: ['yes', 'yes', 'no', 'no', 'no', 'maybe', 'maybe']
    => baseEntropy = 1.5566567074628228
    
    indx=0; feature=non-surfacing
    subDatasets: [[0,1,'no'],[0,1,'no'],[0,0,'maybe']]   [[1,1,'yes'],[1,1,'yes'],[1,0,'no'],[1,1,'maybe']]
    labels: [['no', 'no', 'maybe'],['yes', 'yes', 'no', 'maybe']] 
    => featEntropy = 1.2506982145947811
    => infoGain0 =  0.3059584928680417
    
    indx=1; feature=flippers
    subDatasets: [[0,0,'maybe'],[1,0,'no']]   [[0,1,'no'],[0,1,'no'],[1,1,'yes'],[1,1,'yes'],[1,1,'maybe']]
    labels:  [['no', 'maybe'],['yes', 'yes', 'no', 'no', 'maybe']] 
    => featEntropy = 1.3728057820624016
    => infoGain1 =  0.1838509254004212
    
    infoGain0 > infoGain1
    =>  bestFeat=0  (on-surfacing) is the best feature to split and create the root node

When we look at the two subDataset groups for indx=0 we see that *non-surfacing* splits the labels more uniformly with all *yes* values in one subgroup and two *no* values in the other. Whereas the flippers feature creates two subDatasets that are less pure, ie. the first one contains a *no* and a *maybe* label and the second subDataset contains the other 5 labels. As a result, *non-surfacing* was choosen as the root node in the tree.

It is important to note the the choosen feature is not the one that creates the highest Entropy but rather the one that creates the least because information gain (purity) increases when Entropy decreases (because of the minus sign between baseEntropy and featEntropy).

## Part 3: Looping and Tree Building
 
This part is the heart of the Decision Tree algorithm. In each recursive iteration, a new node is choosen to branch the tree and then a new set of sub-datasets are passed to the next iteration. The algorithm has 8 steps:

  1. Start the algorithm with the a given dataset and a feature list
  2. Check two recursion terminating conditions
  3. Choose best feature to split the given dataset using Entropy test
  4. Make a copy of fetures values of best dataset & copy features list without best feature column 
  5. Create an emptytree node
  6. Split given dataset based on the best feature and its values
  7. Grow subtree for the empty tree node and append it to the main tree
  8. Iterate to step 6 with for the all given values for best feature

Here is the Python code for the 8 steps:

      def createTree(dataset, features):
          labels = [record[-1] for record in dataset]
          
          # Terminating condition #1
          if labels.count(labels[0]) == len(labels):  # stop splitting when all of the labels are same 
              return labels[0]      
          # Terminating condition #2    
          if len(dataset[0]) == 1:                    # stop splitting when there are no more features in dataset
              mjcount = max(labels,key=labels.count)  # select majority count 
              return (mjcount) 

          bestFeat = chooseBestFeatureToSplit(dataset)
          bestFeatLabel = features[bestFeat]
          featValues = {record[bestFeat] for record in dataset}     # put feature values into a set
          subLabels = features[:]             # make a copy of features
          del(subLabels[bestFeat])            # remove bestFeature from labels list

          myTree = {bestFeatLabel:{}}         # value is empty dict
          for value in featValues:
              subDataset = splitDataset(dataset, bestFeat, value)
              subTree = createTree(subDataset, subLabels)
              myTree[bestFeatLabel].update({value: subTree})  # add (key,val) item into empty dict
          return myTree    
    
To understand this recursive function it is important to: (1) debug the sub-dataset splitting at each recursive call and (2) examine the two terminating conditions that stop recursion.

### Debugging Dataset Splitting and Leaf Node Selection
The following debugging output shows how at each level of the recursion a sub-dataset is passed to the function to generate the corresponding subtree branch or leaf node.  

Here we show how the 2 left leaf nodes 'maybe' and 'no' under non-surfacing = False are generated. 

      Dataset: [[1, 1, 'yes'], [1, 1, 'yes'], [1, 0, 'no'], [0, 1, 'no'], [0, 1, 'no'], [1, 1, 'maybe'], [0, 0, 'maybe']]
      labels: ['yes', 'yes', 'no', 'no', 'no', 'maybe', 'maybe']
      features:  ['non-surfacing', 'flippers']
      best Feature to split: non-surfacing (0)

      sub-dataset: [[1, 'no'], [1, 'no'], [0, 'maybe']]
      labels: ['no', 'no', 'maybe']
      features:  ['flippers']
      best Feature to split: flippers (0)

      suf-dataset: [['maybe']]
      labels: ['maybe']      
      features:  []
      leaf node: maybe
      ===========

      sub-dataset: [['no'], ['no']]
      labels: ['no', 'no']      
      features:  []
      leaf node: no
      ===========

Here we show how the 2 right leaf nodes 'no' and 'yes' under non-surfacing = True are generated. 

      Dataset: [[1, 1, 'yes'], [1, 1, 'yes'], [1, 0, 'no'], [0, 1, 'no'], [0, 1, 'no'], [1, 1, 'maybe'], [0, 0, 'maybe']]
      labels: ['yes', 'yes', 'no', 'no', 'no', 'maybe', 'maybe']
      features:  ['non-surfacing', 'flippers']
      2nd best Feature to split: non-surfacing (1)

      sub-dataset: [[1, 'yes'], [1, 'yes'], [0, 'no'], [1, 'maybe']]
      labels: ['yes', 'yes', 'no', 'maybe']
      features:  ['flippers']
      best Feature to split: flippers (0)

      sub-dataset: [['no']]
      labels: ['no']
      features:  []
      leaf node: no
      ===========

      sub-dataset: [['yes'], ['yes'], ['maybe']]
      labels: ['yes', 'yes', 'maybe']      
      features:  []
      leaf node: yes
      ===========

### Two Terminating Condition
When recursion terminates two things happen: (i) a leaf node is generated in the decision tree and (ii) the recursion start to backtrack and the subtree branch gets created.

In the *decisiontree* function there are 2 terminating conditions:
 
 - Condition #1: Check if all labels in the sub-dataset have same value
 - Condition #2: Check that the feature list is empty 

Simply, recursion terminates when all the given labels are the same or when there are no more features to split in the given sub-dataset. To understand, lets try first to correlate the printout from debugging the dataset spliting section with the code for the two condition. In there decision tree there are 4 leaf nodes \[maybe,no,no,yes] the 1st 3 of which are generated by terminating condition #1 and the yes leaf node is generated by condition #2. 

Here we relist the debugging output for two of the leaf nodes:

      sub-dataset: [['no'], ['no']]               # condition #1 applies
      labels: ['no', 'no']      
      features:  []
      leaf node: no
      ===========

      sub-dataset: [['yes'], ['yes'], ['maybe']]   # condition #2 applies
      labels: ['yes', 'yes', 'maybe']      
      features:  []
      leaf node: yes
      ===========

Generating the yes leaf node is a little tricky because as you see we have three labels: two yeses & one maybe. Natuarally in this case the algorithm choose yes because it occurs more often and represents the majority value in the labels list. This computation is done using the *max* Python function shown here.

    mjcount = max(labels,key=labels.count)      # select majority count

## Classification: Making Prediction
Here is the Python code for making predictions:

    def predict(inputTree, features, testVec):

       def classify (inputTree, testDict):
           (key, subtree), = inputTree.items()
           testValue = testDict.pop(key)
           if len(testDict) == 0:
               return subtree[testValue]
           else:
               return classify(subtree[testValue], testDict)

       testDict = dict(zip(features, testVec))
       return classify(inputTree, testDict)

The program is recursive with a nested function. Basically, the input features and test vector are transformed into a dict which is then used to traverse the given tree. At each recursion step the associated dict item is poped and the recursion continues into the sub-tree associated with the dict item that was poped.

To make predictions you can run the following:

      $ python 
      >>> import decisiontree
      >>> import inputdata
      >>> dataset, features = inputdata.createDataset()
      >>> tree = decisiontree.createTree(dataset, features)
      
      >>> decisiontree.predict(tree, features, [0,0])   
      maybe
      
      >>> decisiontree.predict(tree, features, [1,1])
      yes
     
### Traversing Decision Tree: Case examples
Here we try to explain how the decision tree relates to the input data. Looking at the tree diagram depicted at the start of the post, we clearly see that the root node first tests whether the specie is non-surfacing. Then it tests in each case (True or False) if the specie has flippers. There are 4 possible decision cases: maybe, no, no, yes each can be reached based on the given input data. 

For example, the two features of the 7th data records: 
 
    [0, 0, 'maybe']   # 7th data records
    non-surfacing = 0 ; flippers = 0 
    isFish = maybe
    
can be used to traverse the left most branch of the tree because both feature columns are False (0). In this case, the branch ends at the *maybe* leaf node. In another example:

    A specie isFish = Yes if and only if:
      - non-surfacing = 1
      - flippers = 1

the test runs along the right most branch of the tree and terminates at the yes node at the bottom. Eventhough the test for yes leaf node looks benign and simple, there is an important hidden issue here. Specifically, when we try to make predictions given  the following testVec:

    [1, 1]   # testVec for classification 
    
the decision tree returns yes despite the fact that our original data had a yes and a maybe with \[1,1].

    [1, 1, 'yes'], 
    [1, 1, 'maybe']

What I mean here is that the decision tree has decided to classify all \[1,1] as yes and a maybe is only returned when the data is \[0,0]. This issue is common in decision trees and can be traced back to using Entropy that averages probability values.

Overall using a decision tree to make predictions is simple, you take any data record and start traversing the tree based on the values of the feature columns. However, users should have an idea about how it does its computation in order to appreciate its limitations. 

## Concluding Remarks
We can clearly see now after this deep dive that the Decision Tree sample program looks deceivingly simple. The code is both compact and efficient. Understanding how things actually work requires a deeper knowledge of machine learning, Entropy, Python programming - especailly recursive programming.
