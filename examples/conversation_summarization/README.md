# Fine-tuning Language Models with UpTrain: A Simple Guide to Enhancing Models for Custom Use-cases

<h2>Run the example on Google Colab <a href="https://colab.research.google.com/drive/1svJoDdbwe-jSeGuQOhtrRMhxwIqP7nMM?usp=sharing">here</a>.</h1> 

The era of large language models (LLMs) taking the world by storm has come and gone. Today, the debate between proponents of bigger models and smaller models has intensified. While the debate continues, one thing is clear: not everyone needs to run large models for their specific use-cases. In such situations, it's more practical to collect high-quality datasets to fine-tune smaller models for the task at hand.



## Enhancing a Conversation Summarization Model with UpTrain
In this blog post, we'll walk you through a tutorial to collect a dataset to fine-tune a model that can summarize human conversations. The model we're working with is based on the `facebook/bart-large-xsum model`, which is already fine-tuned on the SAMSum dataset, a collection of conversations and their summaries. This model is one of the best open-source models for conversation summarization.

Our goal is to further improve this model so that it performs better on a similar dataset called DialogSum, which also contains conversations and their summaries. To do this, we'll use UpTrain, a powerful tool for creating fine-tuning datasets.

## The Process
In this blog, the process of collecting a fine-tuning dataset involves several steps:

1. Visualizing UMAP/t-SNE for low-performing clusters
2. Finding clusters around data-points where accuracy is low
3. Edge-case Collection (user defines the edge-case parameters based on heuristics/observations)
4. Building Custom Monitor (that checks out-of-vocabulary cases) 

Note: GPU is not required to run this tutorial. <br>
We understand that running the bart-large-xsum can be time consuming on some machines. Hence, we have pre-generated the model outputs and their corresponding sentence BERT embeddings to remote for both the SAMSum and DialogSUM datasets. Due to this, running this entire script does not take too much time (e.g., it runs in 3 minutes on my Macbook Air).

First, let's see (literally) what we are dealing with. We plot the sentence BERT embeddings with UMAP dimensionality reduction. We apply dimensionality reduction on 3 types of datasets: SAMSum train (aka reference dataset), SAMSum test, and DialogSum train.

## Analyzing Performance and Visualizing with UMAP
We'll also use a visualization technique called UMAP to see how different datasets are related in terms of content. Datasets marked `reference` (i.e., SAMSum training) and `samsum` (i.e., SAMSum test) are close in the UMAP space. Most point from the DialogSum dataset are further than the data on which the model was finetuned on (i.e., reference aka SAMSum train).  

<p align="center">
<img width="700" alt="concept_drift_avg_acc" src="https://uptrain-demo.s3.us-west-1.amazonaws.com/conversation_summarization/umap_conv_summ.png">
</p>

Next, we identify poorly performing points and find clusters around them.

## Defining a monitor to catch data-points close to outliers

In this section, we first define a performance metric: Rogue-L similarity. You can choose any metric relevant to your use case. We'll select data points with Rogue-L scores equal to 0.0 as outliers. Our objective is to find data-points that lie around these outliers because they are more likely to perform worse (and as we show later, they do!). 

To do this, we'll use a technique called sentence BERT embeddings to represent our text data. We'll compare these embeddings from the DialogSum dataset to the ones from our reference dataset (SAMSum training).

**RESULT**: The overall performance (Rogue-L score) on the DialogSum dataset was 0.305. However, on the data-points identified by the method above, the performance score dropped to 0.237.

While analyzing the model outputs above, we made a few observations on cases where model does not perform well. Note that these are not statistical ways of finding edge cases but are more inspired by our intuition in dealing with the above data.

## Catching Edge-cases for Finetuning the Model Later
A basic step in fine-tuning the model is to analyze its performance and identify situations where it doesn't perform well to catch appropriate edge-cases. We do this by making a few observations on the model performance.

**Observation**:
Our first observation is that the model has trouble summarizing very long conversations, often producing incomplete summaries. Some examples of these incomplete summaries are below.

```
"Benjamin, Elliot, Daniel and Hilary are going to have lunch with French"
"Jesse, Lee, Melvin and Maxine are going to chip in for the"
"Jayden doesn't want to have children now, but maybe in the future when"
"Leah met a creepy guy at the poetry reading last night. He asked her"
"Jen wants to break up with her boyfriend. He hasn't paid her back the"
```

We used the above information to define an edge case in UpTrain.
To collect a dataset that can help the model improve in these edge cases, we need to find conversations that are longer than a certain threshold. 
To find an appropriate threshold, we generated a histogram of length of input dialogues on the training dataset (i.e., SAMSum train). From here, we noted that a length of 1700 can be a good cut-off to collect large conversation data-points. The histogram is shown below.

<p align="center">
<img width="550" alt="concept_drift_avg_acc" src="https://uptrain-demo.s3.us-west-1.amazonaws.com/conversation_summarization/hist_num_words_samsum.png">
</p>

### Edge Case Type 1: Long dialogues

Following is how the edge-case check is defined in UpTrain. It checks if the length of the input dialogue is greater 
than 1700 characters.
```python
def length_check_func(inputs, outputs, gts=None, extra_args={}):
    this_batch_dialog = inputs['dialog']
    return np.array([len(x) for x in this_batch_dialog]) > 1700

edge_case_length = {
    'type': uptrain.Monitor.EDGE_CASE,
    'signal_formulae': uptrain.Signal("Length_dialog", length_check_func)
} 
```

**Observation**:
Next, we will discuss another type of edge case that affects the performance of our language model. In this case, the model directly copies one or two sentences from the input conversation. This works well in many cases but fails when the conversation is about refuting those one or two sentences which model produced as summary.
This can lead to the generation of summaries that are not accurate or do not capture the true essence of the conversation.

For example, the output summary of the model in the following cases is not appropriate:
```
Input:
Janice: my son has been asking me to get him a hamster for his birthday. Janice: Should I? Martina: NO! NO! NO! NO! NO! Martina: I got one for my son and it stank up the whole house. Martina: So don't do it!!!
Output: Janice's son wants her to get him a hamster for his birthday.

Input:
Person1: Hello, I'm looking for a shop that sells inexpensive cashmere sweaters. Person2: Have you tried an outlet? Person1: Why didn't I think of that? Person2: Many of my friends shop at outlets. Person1: Thanks. That is a good suggestion. Person2: I'm only too happy to help.
Output: Person1 is looking for a shop that sells inexpensive cashmere sweaters.
```

### Edge Case Type 2: Copied sentences with negation
In order to catch this heuristic as an edge case in UpTrain, we define two functions: `rogueL_check_func` and `negation_func`.

The `rogueL_check_func` function checks whether sentences from the input are copied directly using the Rouge-L metric. This metric calculates the longest common subsequence of characters in the input and output texts.

The `negation_func` function, on the other hand, checks if there's a negation in the input by searching for common negation words such as "no," "not," "can't," "couldn't," "won't," "didn't," and "don't."

Finally, we combine these two functions to create an edge case definition called `edge_case_negation`. This definition will help us identify and collect data points where the model directly copies sentences and where negation is present in the input.

By identifying and addressing these edge cases, we can further improve the performance of our language model, making it more accurate and reliable in generating summaries of conversations.

## Custom Monitor for Vocabulary Coverage: Ensuring High-Quality Summarization
In this section, we will discuss how to create a custom monitor to check the vocabulary coverage of the new dataset (DialogSum) on the old dataset (SAMSum). By evaluating the vocabulary coverage, we can determine whether there is a significant shift in the vocabulary used between the two datasets. This will help us understand how well the model can generalize and produce accurate summaries for different datasets.

#### Creating a Custom Monitor
To create a custom monitor, we need to define two functions:

1. `vocab_init`: Initializes the state of the monitor, which includes the training vocabulary and a counter for out-of-vocab words.
2. `vocab_drift`: Checks the vocabulary coverage of the production dataset in the training dataset, updates the out-of-vocab words counter, and logs the results to the UpTrain dashboard.

The following is how the custom monitor is defined in UpTrain.
```python
custom_monitor_check = {
    'type': uptrain.Monitor.CUSTOM_MONITOR,
    'initialize_func': vocab_init,
    'check_func': vocab_drift,
    # To tell the framework ground truth is not needed
    'need_gt': False,
}
```

By using this custom monitor, we can analyze the vocabulary coverage of our model and identify any vocabulary drift that might be affecting the model's performance. This allows us to make the necessary adjustments to improve the accuracy and reliability of our language model in generating summaries for various datasets.

#### Defining UpTrain Config and Framework
In this section, we set up the UpTrain configuration and framework to analyze our model's performance and collect the necessary data. This helps us ensure that our language model is generating accurate summaries for various datasets.

We create a configuration dictionary that includes the edge cases and custom monitor we defined earlier. Additionally, we specify the folder where the smart data will be stored.

```python
config = {
    "checks": [edge_case_negation, edge_case_length, custom_monitor_check],
    "logging_args": {"st_logging": True},
    "retraining_folder": "smart_data_edge_case_and_custom_monitor",
}
framework = uptrain.Framework(cfg_dict=config)
```

#### Vocabulary Coverage
Using the UpTrain dashboard, we can visualize the vocabulary coverage of our model on the production data. The coverage starts at around 98% for the SAMSum test dataset but decreases to about 95% for the DialogSum dataset (that is, after ~800 SAMSum test points are logged).

<p align="center">
<img width="550" alt="hist_num_characters_samsum" src="https://uptrain-demo.s3.us-west-1.amazonaws.com/conversation_summarization/vocab_coverage.gif">
</p>

By inspecting the collected edge cases, we can confirm that our edge case detector is effectively identifying the appropriate cases and collects 554 edge-cases of the 13200 data-points logged into the framework.

#### Out-of-Vocabulary Words
We can also examine the out-of-vocabulary words to gain insights into the differences between the datasets. For example, a significant number of out-of-vocabulary words are related to Asia, such as "yuan", "li", "wang", "taiwan", "zhang", "liu", "chinas", "sichuan", "singapore", and others. This indicates that many conversations in the DialogSum dataset focus on the Asia region.

By understanding the out-of-vocabulary words and their context, we can take steps to improve our model's performance on specific topics or regions, ensuring it generates accurate and relevant summaries for various datasets, as shown next.

## Identifying Edge Cases with Asia-related Words

We've found that many conversations in the DialogSum dataset are focused on the Asia region which were not originally present in the SAMSum dataset. So, we'll create a check to catch cases containing specific Asian words. The list `asian_words` contains some common Asian words like names, places, and currencies, and the check goes on to define cases where Asia-related words are found in the input dialogue.

```python
asian_words = ['yuan', 'li', 'wang', 'taiwan', 'zhang', 'liu', 'chinas', 'sichaun', 'singapore']
def asian_words_check(inputs, outputs, gts=None, extra_args={}):
    has_asian_word = [False]*len(inputs['dialog'])
    for i,text in enumerate(inputs['dialog']):
        all_words = clean_string(text).split()
        if len(set(asian_words).intersection(set(all_words))):
            has_asian_word[i] = True
    return has_asian_word

edge_case_asian_word = {
    'type': uptrain.Monitor.EDGE_CASE,
    'signal_formulae': uptrain.Signal("asian_word", asian_words_check)
}
```

Finally, we can add the above edge-case check to UpTrain config to catch all data-points with Asia-related words.


## Putting It All Together
By following these steps, we can collect a dataset that targets specific weaknesses in the model's performance. This dataset can then be used to fine-tune the model, making it better at handling custom use-cases like summarizing long conversations in the DialogSum dataset.

In summary, fine-tuning smaller language models using tools like UpTrain is an effective way to create tailored solutions for specific tasks. By identifying edge cases, building custom monitors, and analyzing the model's performance, we can collect high-quality datasets that help improve the model's performance for our particular use-case.
