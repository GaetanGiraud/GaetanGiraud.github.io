# How to interpret classificiations test results

## Some reminders on how to interpret classifications algorithms test results

Imagine the following scenario:
An additive is added to a shampo to make your hair glow in the dark, called the cosmic shampoo. A labelling is performed to 
indicate if this shampo will give you the halloween look, or if it should be labelled as a regular shampo. This process is stochastic, 
sometimes it gets the labelling wrong.
Some tests are performed to make sure that the label are correctly identified. In this context we can define intuitively 
the following measures:

**True Positives (TP)**: These are the correctly predicted positive values. All the cosmic shampoo correctly identified as cosmic shampoo.

**True Negatives (TN)**: These are the correctly predicted negative values. All the regular shampoo correctly identified as regular shampoo.

**False positives and false negatives, these values occur when your actual class contradicts with the predicted class.

**False Positives (FP)**: When actual class is no and predicted class is yes. Regular shampoo identified as being cosmic shampoo.

**False Negatives (FN)**: When actual class is yes but predicted class in no. cosmic shampoo identified as being regular shampoo.

### Some more metrics

**Accuracy** : In total, how many shampo did the labelling algorithm got right? 

**Precision** : Of all the shampo identified as cosmic glow, how many actually have it? How many of the customers buying the cosmic glow will 
actually have a very basic haircut for halloween?

**Recall (True positive Rate / Sensitivity)** : Of all the shampoo that are actually of the cosmic glow sort, how many did we identify 
as actually being of the cosmic glow? If we were to sell the cosmic glow for a higher price, how many would we sell 
for the lower regular price and thus miss revenue.

Another graphical representation
![Precision & Recall](https://upload.wikimedia.org/wikipedia/commons/2/26/Precisionrecall.svg)

By <a href="//commons.wikimedia.org/wiki/User:Walber" title="User:Walber">Walber</a> - <span class="int-own-work" lang="en">Own work</span>, <a href="https://creativecommons.org/licenses/by-sa/4.0" title="Creative Commons Attribution-Share Alike 4.0">CC BY-SA 4.0</a>, <a href="https://commons.wikimedia.org/w/index.php?curid=36926283">Link</a>

## My interpretation of these measures
For me, accuracy represent the most straigforward metric. It is a measure of the *raw* power of the algorithm, how does it perform doing 
its main task of classifying (whatever it is it's actually classifying). The first thing to look at, intuitively makes the most sense.
Any model will need to have a relatevely high level of accuracy to be able to perform. 

TO Be continued


