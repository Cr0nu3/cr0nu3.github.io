---
title: Text Summarization via Machine Learning
description: >-
  Machine Learning Project
author: Cronus
date: 2024-01-02 10:41:00 
categories: [Project, Edu Helper]
tags: [Machine Learning]
pin: false
image:
  path: /assets/img/ML/extract.png
  width: 300
  height: 300
---

The basic information about Text Summarization is described in [here](https://cronuse.tistory.com/238). There are many documents and description about Text Summarization, but no appropriate code so I made my own.

Many documents that I found by googling take articles or journals and run text summarization in abstractive way. In my testing of various articles, I found that abstractive mode made the article easier to understand and better captured the context and meaning of the original article.

But for my project, there is a basic difference from normal text summarization codes. The differences are as follows

1.  Existing codes take "complete" articles and summarize it. ("complete" means a post with proper punctuation including commas, periods, exclamination points, and double quotes. However,  in my project, the posts to be summarized have no punctuation by default.)
2.  The length of aritcles that existing code targets is significantly shorter than the length of texts that my project needs to summarize. This means, I have to develop a new algorithm that can't found by googling to summarize long aritcle.

In my testing, the Abstractive mode seems to be able to extract good summaries for longer posts, although the summarization algorithm needs to be more sophisticated. I decided that it would be more productive to use a Extractive mode that to develop a sophisticated algorithms, so I created an algorithm using that approach.

( I describe the way that adding proper punctation to original text in [Converting Audio into Text(eng)](https://cronuse.tistory.com/246) )

#### **1\. Get contents I want to summarize**

```python
file = open(path, "r", encoding='UTF-8')
core_text = file.read().split('===========================================================================')[-1]
```

#### **2\. Text Summarization**

```python
from collections import Counter
from nltk.corpus import stopwords

nltk.download("stopwords", quiet=True)
stopwords_english = stopwords.words("english")
word_frequencies = {}
for word in nltk.word_tokenize(formatted_text):
    if word not in stopwords_english:
        if word not in word_frequencies.keys():
            word_frequencies[word] = 1
        else:
            word_frequencies[word] += 1
```

This algorithm works like this. Import "stopwords list" from NLTK library. The list of stop\_words are like this.

```python
['i', 'me', 'my', 'myself', 'we', 'our', 'ours', 'ourselves', 'you', "you're"]
```

 Don't count these words because they don't have a significant meaning in article. Whenever a word in this list does not appear, the value of word in word\_frequencies is incremented by 1 if the value of word is non-zero. Now that I have counted the frequency of each word, I divide the frequency of each word by the frequency of the most frequent word.

```python
maximum_frequency = max(word_frequencies.values())

for word in word_frequencies.keys():
    word_frequencies[word] = (word_frequencies[word]/maximum_frequency)

print(Counter(word_frequencies).most_common(20))
```

```
[('course', 1.0), ('G', 0.92), ('three', 0.92), ('requirements', 0.88), ('education', 0.8), 
('The', 0.8), ('courses', 0.8), ('units', 0.76), ('general', 0.76), ('E', 0.64), ('one', 0.64),
('A', 0.6), ('see', 0.6), ('If', 0.56), ('division', 0.56), ('area', 0.56), ('transfer', 0.52),
('Area', 0.52), ('D', 0.44), ('pattern', 0.4)]
```

I can summarize text with extractive mode based on this result. This model choose words with the highest score. I caculated by using each word's frequency and result value, and  made algorithm based on sentences of 50 words or more. ( because I must summarize significantly long aritlces )

```
sentence_scores = {}
for sent in sentence_list:
    for word in nltk.word_tokenize(sent):
        if(word in word_frequencies.keys() and len(sent.split(' ')) > 50):
            if sent not in sentence_scores.keys():
                sentence_scores[sent] = word_frequencies[word]
            else:
                sentence_scores[sent] += word_frequencies[word]
```

The algorithm that calcuate importance of sentences is done! Let's get summarized text.

```
summary_sentences = heapq.nlargest(8, sentence_scores, key=sentence_scores.get)

summary = ' '.join(summary_sentences)
```

#### **Full Code :**

```python
import nltk
from nltk.corpus import stopwords
import heapq

def generate_summary(path, output_filename, mode):
    nltk.download('punkt')
    nltk.download("stopwords", quiet=True)

    file = open(path, "r", encoding='UTF-8')
    core_text = file.read().split('===========================================================================')[-1]

    # Removing Square Brackets and Extra Spaces
    core_text = re.sub(r'\[[0-9]*\]', ' ', core_text)
    core_text = re.sub(r'\s+', ' ', core_text)

    # Removing special characters and digits
    formatted_text = re.sub('[^a-zA-Z]', ' ', core_text )
    formatted_text = re.sub(r'\s+', ' ', formatted_text)

    sentence_list = nltk.sent_tokenize(core_text)
    stopwords_english = stopwords.words("english")

    word_frequencies = {}
    for word in nltk.word_tokenize(formatted_text):
        if word not in stopwords_english:
            if word not in word_frequencies.keys():
                word_frequencies[word] = 1
            else:
                word_frequencies[word] += 1

    maximum_frequency = max(word_frequencies.values())

    for word in word_frequencies.keys():
        word_frequencies[word] = (word_frequencies[word]/maximum_frequency)

    sentence_scores = {}
    for sent in sentence_list:
        for word in nltk.word_tokenize(sent):
            if(word in word_frequencies.keys() and len(sent.split(' ')) > 50):
                    if sent not in sentence_scores.keys():
                        sentence_scores[sent] = word_frequencies[word]
                    else:
                        sentence_scores[sent] += word_frequencies[word]

    summary_sentences = heapq.nlargest(8, sentence_scores, key=sentence_scores.get)

    summary = ' '.join(summary_sentences)
    path = "./summary/" + output_filename
    try:
        with open(path, mode=mode, encoding='UTF-8') as file:
            file.write('Summarized Content: \n')
            file.write(str(summary))
            file.write("\n========================================================\n")
        print("+======================+")
        print("|   Summary Complete   |")
        print("+======================+")
    except Exception as e:
        error("Error occured during summarizing text!")
```

#### **Result :**

```
Summarized Content: 
So if you look at recent results from several different leading speech groups, Microsoft showed that this kind of deep neural network when used to see coasting model and speech system would use the or right from twenty seven point four percent, eighteen point five percent, or alternatively, you can view it as reducing the amount of training that you needed from two thousand hours time to three hundred hours to get comparable performance i b m which has the best system for one of the standard speech recognition tasks for large recovery speech recognition showed that even it's very highly tuned system that was getting eighteen point eight percent can be beaten by one of these deep neural networks. That was still much less than that train think i see mixture model on but even with much less data, it did a lot better than the technology they had before said reduce the or right from sixteen percent trump on three percent and the area it is still falling and in the latest android if you do voice search is using one of these deep neural networks in order to very good speech recognition. So they look at this little window and they say in the middle of this window, what do I think the phony me some which part of the phone you miss it and a good speech recognition system will have many alternative models for phony and each model and might have three different parts. Students on the freshmen pattern must complete a minimum of three units in the one Social Sciences, three units, indeed to US History and three units in D, Three US Government and California Government students on the transfer pattern need nine units of Area D Social Science coursework If you have none of your Area D completed, we recommend you still include a D to and D three course when completing this section because you still need to complete the U. S. History and the U. S. Government and California government requirements. Students must complete a minimum of nine units in area, see, including three units, and see one arts, three units, and see to Humanities, and then three final units from either see one arts or see to humanities after area, see his area, d Social Sciences, which has met through completion of a minimum of nine units. This consists of a three unit course in each of the three domains of knowledge, physical in life Science, You D DB, Arts and Humanities, U. D. C and Social Sciences, You D. D. Note that to take an Upper division G. E. Course, one must meet all prerequisites for the courses which minimally include completion of lower division, G. E. A one A to a three and before freshmen.
========================================================
```