---
title: Text Summarization via Machine Learning
description: >-
  Machine Learning Project
author: Cronus
date: 2024-01-02 10:39:00 
categories: [Project, Edu Helper]
tags: [Machine Learning]
pin: false
image:
  path: /assets/img/ML/main.jpeg
  width: 300
  height: 300
---

#### **This document is related to my project named Edu\_Helper.**

I write this article with help from other's documents.

### **※ Impact**

Summarization System has additional evidence that they can utilize in order to specify the most import topics of documents. For example, summarizing journals or blogs, there are discussions and comments coming after the blog post which are good sources of information to determine which parts of blog are critical and important.

### ****※** How text summarization works**

There are two types of summarization. One is **abstractive summarization** and the other is **extractive summarization.** 

#### **1\. Abstractive Summarization :**

   Abstractive methods select words based on semantic understanding, even those words did not appear in the source documents. Its purpose is producing import material in a new way. They interpret and understand the text using advanced natrual language techniques in order to generate a new shorter text that conveys core information from original text.

It can be correlated to the way human reads a blog post or journal and then summarized in their own words.

**Give document -> understand context -> semantic -> create summary using their own words**

#### **2\. Extractive Summarization :**

    Extractive methods attempt to summarize articles by selecting a important words that retain high score.

This method weights the important part of sentences and form the summary. Different algorithm and techniques are used to define weights for the sentences and rank them based on importance and similarity among each other.

**Give document -> sentences similarity -> weight sentences -> select sentences via higher rank**

Automatic abstractive summaries usually give better results compared to extractive summaries. This is because of the fact that abstractive summarization methods cope with problems such as semantic representation, inference and natural language generation which is relatively harder than data-driven approaches such as sentence extraction.

This example uses [**unsupervised learning**](http://appier.com/ko-kr/blog/a-simple-guide-to-unsupervised-learning) approach to find similarity of the sentences and rank them. A benefit of this method is that I don't need gather data set and don't need to train.

Before keeping continue, you have to know [**Cosine Similarity**](https://wikidocs.net/24603) to understand this article better. Cosine Similarity is  a measure of similarity by using non-zero vectors taht measures the cosine of angle between them. By representing given sentences as the bunch of vectors, I can use it to find similarity among sentences. An easy example, if angle is zero, two sentences are similar.

Below is my code flow to generate summarize text.

**Give article -> split into setences by using given rule -> remove stop words if it exists**

**\->** **build a similarity matirx -> generate rank based on matrix -> pick high ranked sentences**

**1\. Import Libraries**

```
import nltk
from nltk.corpus import stopwords
from nltk.cluster.util import cosine_distance
import numpy as np
import networkx as nx
```

**2\. Organize sentences and read**

```
def read_article(file_name):
    file = open(file_name, "r", encoding='UTF-8')
    filedata = file.readlines()

    for i in range(0, len(filedata)):
        filedata[i] = filedata[i].replace("\n", " ")
    article = filedata[0].split(". ")
    sentences = []

    for sentence in article:
        print(sentence)
        sentences.append(sentence.replace("[^a-zA-Z]", " ").split(" "))
    sentences.pop() 
    print(filedata)
    return sentences
```

**3\. Similarity matrix**

This is where we will be using cosine similarity to get simailarity between sentences.

```
def build_similarity_matrix(sentences, stop_words):
    # Create an empty similarity matrix
    similarity_matrix = np.zeros((len(sentences), len(sentences)))
 
    for idx1 in range(len(sentences)):
        for idx2 in range(len(sentences)):
            if idx1 == idx2: #ignore if both are same sentences
                continue 
            similarity_matrix[idx1][idx2] = sentence_similarity(sentences[idx1], sentences[idx2], stop_words)

    return similarity_matrix
```

**4\. Generate summary method**

```
def generate_summary(file_name, top_n=5):
    nltk.download("stopwords")
    stop_words = stopwords.words('english')
    summarize_text = []

    # Step 1 - Read text anc split it
    sentences =  read_article(file_name)

    # Step 2 - Generate Similary Martix across sentences
    sentence_similarity_martix = build_similarity_matrix(sentences, stop_words)

    # Step 3 - Rank sentences in similarity martix
    sentence_similarity_graph = nx.from_numpy_array(sentence_similarity_martix)
    scores = nx.pagerank(sentence_similarity_graph)

    # Step 4 - Sort the rank and pick top sentences
    ranked_sentence = sorted(((scores[i],s) for i,s in enumerate(sentences)), reverse=True)    
    print("Indexes of top ranked_sentence order are ", ranked_sentence)    

    for i in range(top_n):
      summarize_text.append(" ".join(ranked_sentence[i][1]))

    # Step 5 - Offcourse, output the summarize text
    print("Summarize Text: \n", ". ".join(summarize_text))