---
layout: post
title: "Text Compression with Neural Networks"
date: 2020-10-30 18:50:20 +0000
tags: programming python compression
---

Can we use AI to make compression better? Yes we can! In this post I outline a compression algorithm for English text built on top of GPT-2 that achieves an order of magnitude better compression ratio than gzip (and is several orders of magnitude slower to run 😉)


## Introduction: Compression and Language Models

At a very high level, compression works by identifying and exploiting patterns and redundancies in the source message. A common approach in compression is to use more compact codes for more common inputs - for example, you might compress English text by using fewer bits for common letters (like 'e'), and more bits for rare letters (like 'q'). Similarly, some character sequences are more common than others: for example 'th' is very common in English, while no words contain 'tq'. Most lossless compression algorithms like gzip work by exploiting these kinds of byte-level patterns[[^gzip]]. 

[^gzip]: The DEFLATE algorithm used in gzip has two parts: first it looks for and eliminates repetitions using LZ77, and _then_ it does bit reduction using Huffman coding.  

In addition to patterns at the character level, English has a lot of higher-order structure, at the word and sentence level. For example, at the word level, vowels in English carry little information.  Therefore: mst ppl hv n trbl rdng ths sntnc. At the sentence level, consider the phrase: "I was born in France, then moved to Germany when I was 15.  I still speak fluent _____." As humans, we can confidently guess the last word is most likely "French".

Generally speaking, a language model is a probability distribution over a sequence of words.  That is, given some previous words in a text, a language model tells us what is the probability the next word is _w_? A straightforward usecase for this is autocomplete on your smartphone keyboard and (more recently) when composing emails in Gmail. It is possible to build a simple language model using n-grams, but recently neural networks have become very good at generating convincing text. A famous, recent, successful example of this is [GPT-2](https://openai.com/blog/better-language-models/) from OpenAI (take a look at some of the samples in that post to see how convincing this model is)


## Using a language model for text compression

The fact that language models work means they can capture some of the structure in language. So, how can we exploit this structure for compression? A simple scheme might be the following: type out the input text on your smartphone keyboard, and every time the keyboard's next-word prediction is correct, replace that word with a special character (let's use underscores for now):

{% highlight text %}
Input:      It's a nice place to work with friends.
Compressed: It's a nice _ _ work with friends. 
{% endhighlight %}

On my phone, after typing "It's a nice ", SwiftKey suggests "place" as the main (middle) suggestion, so we can replace "place" with an underscore. Since the underscore is shorter than any word it might replace, the compressed message is shorter than the original. As long as I have access to the same keyboard (language model), and if it is deterministic, I can reconstruct the original text by replacing underscores with whatever the main suggestion is from the language model. 

But we can do even better: the phone keyboard actually has 3 suggestions, not just one. Therefore, we might be able to compress even better by encoding runner-ups as well: 

{% highlight text %}
Input:      It's a nice place to work with friends.
Compressed: It's a nice _0 _0 work _1 friends. 
{% endhighlight %}

In this case, "with" was the second suggestion, so we can compress it as _1 (we'll keep things zero-indexed). If the on-screen keyboard offered even more autocomplete suggestions (say, the top 10 runner-ups), we would have even more opportunities for compression.  


## Trying this out with a language model

Instead of a phone keyboard, I'll use GPT-2, because (1) it's a much more powerful model, (2) it's easier to work with, and (3) it's open source.  

Broadly, this is what I did: 

1. Start with some English text (I used the full text of [Alice in Wonderland](https://github.com/google/snappy/blob/master/testdata/alice29.txt))
2. Split the text into words, and assign each word a number. GPT-2 has a dictionary of 50,000 words, so we'll end up with a sequence of numbers between 0 and 50,000. [[^tokenization]]
3. For each word in the text:
   1.  Show the language model the last _k_ words, and ask it to predict the next word. It will output a probability for each of the 50,000 possible words in our dictionary. 
   2.  Sort the output by probability, and figure out the rank of our word.  In our previous example, "place" would have rank 0 because it was the top prediction, and "with" would have rank 1 because it was the runner-up. 

   ![Converting tokens to ranks](/assets/2020-compression/tokens_to_ranks.png)
4. Instead of compressing the IDs of each word, compress the IDs of each word's _rank_. 

[^tokenization]: GPT-2 actually uses Byte Pair Encoding with a vocabulary size of 50,000 words, so the tokens are not necessarily complete words. In practice, though, most tokens are complete words.  


By doing steps 1 and 2, we converted our original _n_-word text into a sequence of _n_ integers. Step 3 still gives us _n_ integers, so on the surface it's not very helpful.  However, the hope is that the latter sequence is somehow more _compressible_. 


### Token Frequency
Let's take a look at the distribution of the 20 most common words (more precisely, tokens). As expected, a lot of it is punctuation and stopwords. In total, the top-20 most frequent tokens account for 42.6% of all the tokens in the text. 

![Distribution of words](/assets/2020-compression/frequencies-words.png)

### Rank frequency
The distribution of the word ranks drops off much more quickly, and the top-20 ranks account for 82.8% of all the tokens. 

![Distribution of ranks](/assets/2020-compression/frequencies-ranks.png)

For a good langauge model, we expect lower-ranks (i.e. higher-probability tokens) to occur more frequently. This is mostly true here, though there are some exceptions - for example, the actual next token is the 12th likeliest slightly more often than the 11th likeliest. 

### Comparing entropy

We can also compare the entropy for these two distributions.  Intuitively, a distribution has more entropy if sampling from it is more informative.  For example, tossing a fair coin ({H: 0.5, T: 0.5}) gives us 1 bit of information, whereas tossing a completely biased coin ({H: 1.0, T: 0.0}) gives us 0 bits -- it tells us absolutely nothing new. Another interpretation is that we need 1 bit to store the result of one fair coin toss. 

Using `scipy.stats.entropy`: the distribution of tokens has 8.44 bits of entropy, and the distribution of ranks has only 4.37 bits. In other words, because the distribution of ranks is less informative, it should take fewer bits (less space) to encode each token. 


## Compressing ranks from the language model

The Alice in Wonderland text is 148,481 characters.  The tokenizer for GPT-2 has a 50,000-token vocabulary, so we can represent each of the 42,579 tokens with 2 bytes. Thus, just by encoding the text into tokens, we can compress it to `42,579 * 2 bytes = 85,158 bytes`.

As we've seen before, the 50,000 tokens are not all equally likely, so we can do bit reduction to compress it further.  In Python, we could use the builtin zlib function: 

{% highlight python %}
import zlib
import struct

def compress_short_ints(shorts): 
    byte_parts = [struct.pack('>H', short) for short in shorts]
    bytearr = b''.join(byte_parts)
    return zlib.compress(bytearr)
{% endhighlight %}

This reduces the tokens sequence to 48,138 bytes, and the ranks sequence to 33,377 bytes. This is a nice result, as it confirms that the ranks sequence is more compressible than the tokens sequence. However, the performance still seems suboptimal: theoretically, we expect every token in the ranks sequence to take up ~4 bits or 0.5 bytes, so the ranks sequence should compress to just over 21,000 bytes. 

One source of inefficiency is that zlib does bit reduction at the byte level, but we work with tokens of two bytes. Just to check if this is right, we can build our own compressor.  I used Huffman coding because the `dahuffman` library is available on pip, but it should be possible to get even better results with arithmetic coding. In Python: 

{% highlight python %}
from dahuffman import HuffmanCodec
from collections import Counter

def huffman_compress_short_ints(shorts): 
    freqs = Counter(shorts)
    codec = HuffmanCodec.from_frequencies(freqs)
    return codec.encode(shorts)
{% endhighlight %}

The tokens sequences now compresses to 45,143 bytes, and the ranks sequence to 23,487. Much better!


### What about the frequency table? 
Note that we're cheating a little bit above, because we also need access to the frequency table to decompress the message. There are three approaches we can take:  

   1. Use the same frequency table for all messages. For example, we could obtain this by computing frequencies on a large corpus. This works well when the token distributions don't change a lot between messages. 
   2. Prepend the frequency table to the compressed message. This makes the message longer but might provide a better encoding if the token distribution varies between messages.
   3. A hybrid approach. For example, the top-k most common tokens have a fixed distribution, and compressed messages prepend the infrequent tokens. 

Almost immediately, we can see that approach (2) is not appropriate for short texts because it would take 50,000 integers just to store the frequencies (by comparison, a short book like Alice in Wonderland has only 42,579 tokens). To compare these approaches in a quick and dirty way, I split my dataset into two halves, using the first to generate a frequency table and the second to compress (this approximation is only accurate if you believe the two parts are independent). 

In my tests the second half compresses to: 
  - 12,342 bytes if we "cheat" and use its own frequency table to do compression (baseline). 
  - 13,390 (+8.49%) bytes if we use the frequency table from the other half of the dataset. 
  - 13,427 (+8.79%) bytes if we use the hybrid approach, keeping the top 1000 most common tokens. This results from an encoding size of 12,497 bytes, plus 930 bytes to indicate the 465 new tokens. 

So, it seems that using a static frequency table is the best approach. If our corpus is small, we can probably generate a much better static frequency table by smoothing it. However, I didn't experiment with this. 


### Conclusion

Using the tricks described in this article, we managed to compress Alice in Wonderland from 148,481 to 25,600 bytes. This is 2x better than gzip (53,654 bytes) and xz (47,876 bytes). 

That compression and building ML models are related is not a new idea. In fact, the website for the Hutter Prize, an ongoing competition for effective compression of text data, recognizes as much: they are motivated by the observation that "being able to compress well is closely related to acting intelligently." This specific approach wouldn't qualify for the Hutter Prize, because the GPT-2 model is too big. There is some [academic work](https://paperswithcode.com/sota/language-modelling-on-hutter-prize) on using language models to compress text, as well as a [very easy-to-use implementation](https://bellard.org/textsynth/sms.html) (which unfortunately isn't open-sourced). 



---
Footnotes:
