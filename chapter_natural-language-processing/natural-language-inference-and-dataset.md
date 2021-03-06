# Natural Language Inference and the Dataset

In :numref:`sec_sentiment`, we discussed the problem of sentiment analysis.
This task aims to classify a single text sequence into predefined categories,
such as a set of sentiment polarities.
However, when there is a need to decide whether one sentence can be inferred form another, 
or eliminate redundancy by identifying sentences that are semantically equivalent,
the capability of analyzing one text sequence seems rather insufficient.
Instead, we need to be able to reason over pairs of text sequences.


## Natural Language Inference

*Natural language inference* (NLI) studies whether a *hypothesis*
can be inferred from a *premise*, where both are a text sequence.
In other words, NLI determines the logical relationship between a pair of text sequences.
Such relationships usually fall into three types:

* *Entailment*: the hypothesis can be inferred from the premise.
* *Contradiction*: the negation of the hypothesis can be inferred from the premise.
* *Neutral*: all the other cases.

NLI is also known as the recognizing textual entailment task.
For example, the following pair will be labeled as *entailment* because "showing affection" in the hypothesis can be inferred from "hugging one another" in the premise.

> Premise：Two women are hugging each other.

> Hypothesis：Two women are showing affection.

The following is an example of *contradiction* as "running the coding example" indicates "not sleeping" rather than "sleeping".

> Premise：A man is running the coding example from Dive into Deep Learning.

> Hypothesis：The man is sleeping.

The third example shows a *neutrality* relationship because neither "famous" nor "not famous" can be inferred from the fact that "are performing for us". 

> Premise：The musicians are performing for us.

> Hypothesis：The musicians are famous.

NLI has been a central topic for understanding natural language.
It enjoys wide applications ranging from
information retrieval to open-domain question answering.
To study this problem, we will begin by investigating a popular NLI benchmark dataset.


## The Stanford Natural Language Inference (SNLI) Dataset

Stanford Natural Language Inference (SNLI) Corpus is a collection of over $500,000$ labeled English sentence pairs :cite:`Bowman.Angeli.Potts.ea.2015`.
We download and store the extracted SNLI dataset in the path `../data/snli_1.0`.

```{.python .input  n=28}
import collections
import d2l
import os
from mxnet import gluon, np, npx
import re
import zipfile

npx.set_np()

# Saved in the d2l package for later use
d2l.DATA_HUB['SNLI'] = (
    'https://nlp.stanford.edu/projects/snli/snli_1.0.zip',
    '9fcde07509c7e87ec61c640c1b2753d9041758e4')

data_dir = d2l.download_extract('SNLI')
```

### Reading the Dataset

The original SNLI dataset contains much richer information than what we really need in our experiments. Thus, we define a function `read_snli` to only extract part of the dataset, then return lists of premises, hypotheses, and their labels.

```{.python .input  n=66}
# Saved in the d2l package for later use
def read_snli(data_dir, is_train):
    """Read the SNLI dataset into premises, hypotheses, and labels."""
    def extract_text(s):
        # Remove information that will not be used by us
        s = re.sub('\(', '', s) 
        s = re.sub('\)', '', s)
        # Substitute two or more consecutive whitespace with space
        s = re.sub('\s{2,}', ' ', s)
        return s.strip()
    label_set = {'entailment': 0, 'contradiction': 1, 'neutral': 2}
    file_name = (data_dir + 'snli_1.0_'+ ('train' if is_train else 'test')
                 + '.txt')
    with open(file_name, 'r') as f:
        rows = [row.split('\t') for row in f.readlines()[1:]]
    premises = [extract_text(row[1]) for row in rows if row[0] in label_set]
    hypotheses = [extract_text(row[2]) for row in rows if row[0] in label_set]
    labels = [label_set[row[0]] for row in rows if row[0] in label_set]
    return premises, hypotheses, labels
```

Now let's print the first $3$ pairs of premise and hypothesis, as well as their labels ("0", "1", and "2" correspond to "entailment", "contradiction", and "neutral", respectively ).

```{.python .input  n=70}
train_data = read_snli(data_dir, is_train=True)
for x0, x1, y in zip(train_data[0][:3], train_data[1][:3], train_data[2][:3]):
    print('premise:', x0)
    print('hypothesis:', x1)
    print('label:', y)
```

The training set has about $550,000$ pairs,
and the testing set has about $10,000$ pairs.
The following shows that 
the three labels "entailment", "contradiction", and "neutral" are balanced in 
both the training set and the testing set.

```{.python .input}
test_data = read_snli(data_dir, is_train=False)
for data in [train_data, test_data]:
    print([[row for row in data[2]].count(i) for i in range(3)])
```

### Defining a Class for Loading the Dataset

By inheriting `Dataset` by Gluon, we self-defined a natural language inference dataset `SNLIDataset`. By realizing `__getitem__` function, we can get access to the sentence pairs with idx index and relevant categories.

```{.python .input  n=115}
# Saved in the d2l package for later use
class SNLIDataset(gluon.data.Dataset):
    """A customized dataset to load the SNLI dataset."""
    def __init__(self, dataset, vocab=None):
        self.num_steps = 50  # We fix the length of each sentence to 50.
        p_tokens = d2l.tokenize(dataset[0])
        h_tokens = d2l.tokenize(dataset[1])
        if vocab is None:
            self.vocab = d2l.Vocab(p_tokens + h_tokens, min_freq=5,
                                   reserved_tokens=['<pad>'])
        else:
            self.vocab = vocab
        self.premises = self.pad(p_tokens)
        self.hypotheses = self.pad(h_tokens)
        self.labels = np.array(dataset[2])
        print('read ' + str(len(self.premises)) + ' examples')

    def pad(self, lines):
        return np.array([d2l.trim_pad(self.vocab[line], self.num_steps, 
                                      self.vocab['<pad>']) for line in lines])

    def __getitem__(self, idx):
        return (self.premises[idx], self.hypotheses[idx]), self.labels[idx]

    def __len__(self):
        return len(self.premises)
```

### Putting All Things Together

Training set and testing set examples are respectively established on the basis of self-defining `SNLIDataset`. We define 50 as the maximum text length. Then, we can check the number of samples retained in training set and testing set.

```{.python .input  n=114}
# Saved in the d2l package for later use
def load_data_snli(batch_size, num_steps=50):
    """Download the SNLI dataset and return its Dataloader instances."""
    data_dir = d2l.download_extract('SNLI')
    train_data = read_snli(data_dir, True)
    test_data = read_snli(data_dir, False)
    train_set = SNLIDataset(train_data)
    test_set = SNLIDataset(test_data, train_set.vocab)
    train_iter = gluon.data.DataLoader(train_set, batch_size, shuffle=True)
    test_iter = gluon.data.DataLoader(test_set, batch_size, shuffle=False)
    return train_iter, test_iter, train_set.vocab
```

Assume batch size is 128, respectively define the iterators of training set and testing set.

```{.python .input  n=111}
batch_size = 128
train_iter, test_iter, vocab = load_data_snli(batch_size)
```

Output the size of the word list, showing 18677 valid words.

```{.python .input  n=112}
print('Vocab size:', len(vocab))
```

Print the form of the first small batch. What is different from text classification task is the data here consist of triples (Sentence 1, Sentence 2, Label)

```{.python .input  n=113}
for X, Y in train_iter:
    print(X[0].shape)
    print(X[1].shape)
    print(Y.shape)
    break
```

## Summary
- Natural language inference task aims to identify the inference relationship between premise and hypothesis.
- In natural language inference task, the sentences have three types of inference relationships: entailment, contradiction and neutral.
- An important dataset of natural language inference task is known as Stanford natural language inference (SNLI) dataset.


## Exercises

1. Machine translation has long been evaluated based on superficial $n$-gram matching between an output translation and a ground-truth translation. Can you design a measure for evaluating machine translation results by using NLI?





## [Discussions](https://discuss.mxnet.io/t/5517)

![](../img/qr_natural-language-inference-and-dataset.svg)
