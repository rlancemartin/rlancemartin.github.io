---
layout: post
comments: true
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

PhDs can seem like strange endevours. This is my attempt to explain what I did and why.

####The cell-free DNA phenomenon

Much of the work for my PhD was rooted in an observation made by two French scientists in 1947. They reported that human blood contains free-floating DNA molecules. These molecules are the debris of dead cells that continually get washed into circulation. Little attention was paid to this "cell-free DNA" phenomenon for decades, largely because it didn't seem interesting.

But, blood turns out to be information-rich. There are ~100 billion of these DNA fragments per mililiter. Though most are normal, a small fraction may be abnormal in interesting and useful ways. The advent of Next-Generation Sequencing (NGS), a very high-throughput DNA seqeucning technology, made it possible to count these molecules at far greater depth than ever before. 

Steve Quake, my advisor, showed that deep molecular couting of these fragments can be used to non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0) during pregnancy. With sufficiently deep sampling, it is possible to detect an elevation in certain fetal-derived DNA fragments specific to Down Syndrome pregnancies. 

####Taste for problems

Around the time I started my PhD, a [rising tide](http://www.hmpdacc.org/) of studies were using NGS to study the human microbiome. NGS applied to infectious disease was equally appealing. We reasoned that it should possible to build a non-invasive diagnostic for infectious disease by applying NGS to cell-free DNA. At least three specific features made this a good problem to tackle:

* ***Zero cost data:*** We had cell-free DNA sequencing data for thousands of samples.
* ***Fundemental:*** Micro-organisms should be subject to the same process as human cells. 
* ***Easily testable:*** We had indepndent clinical tests for infection for our sampled cohorts.

In addition, this was non-obvious. Our strategy required re-analyzing cell-free DNA sequencing from organ [transplant](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28) patients.Though these data had been analyzed for a different study, the non-human fraction wasn't clinically analyzed. We reasoned that it may be possible to establishe clinical correlations for infectious disease using this "exhust" data.

####The pipeline

<div class="imgcap">
<img src="/assets/Infectome_1.jpg" width="100%">
</div>

This is data problem, because the biochemistry and technology needed to isolate and sequence cell-free DNA are well worked out; I barely touched a pipet during my PhD. The challenge (and opportunity) is that human blood is very information rich: We typically sequence around 30 million of these at a time, resulting in a file of digitized DNA fragments (typically a few GB) as short (100-200) character strings. Each string indicates the nucleic acid composition (A,C,T, or G) of a single DNA molecule in the sample. 

Once all DNA molecules in the sample are digitized, we remove all human-derived sequences. This uses the human reference, a ["representitive"](http://en.wikipedia.org/wiki/Reference_genome) human genome sequence. Our pool of strings can be matched to the reference using alignment tools that take advantage of algorithms such as [the Burrows–Wheeler transformation](http://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform). Strings that fail to match the human reference can then be [aligned](http://en.wikipedia.org/wiki/BLAST) to databases of micro-organism genomes. Ultimatly, this process yields a dictionary for each sample, a count of strings associated with each entry in the micro-organism database. 

####Why is "big data" relevant here?

We performed the basic processing described above to thousads of blood samples collected from patients undergoing procedures at Stanford hospital. The dictionary resturned from each sample had up to thousands of unique micro-organisms. Along with this, we collected patient -metadata, including thousands of de-identified clinical test results. These data, along with pipeline log files required to normalization of counts based upon parametrs such as sampling depth, we written to `Postgres`. For this, we took advantage of two excellent libraries, [`pandas`](http://pandas.pydata.org/) and [`SQLAlchemy`](http://www.sqlalchemy.org/). 

This resulted in a large amount of data to organize and understand. The value of data at scale is two-fold. **Significance**. Every count recorded for a sample can be compared against every other observed count for that same micro-organism across all other samples. As more samples are collected, we gain greater ability to detect anomolies. **Unbiased**. With un-biased sampling, we can detect 

<div class="imgcap">
<img src="/assets/Infectome_2.jpg" width="100%">
</div>

#### Bullet points work!

* Start a line with a star
* Profit!

<div class="imgcap">
<img src="/assets/Infectome_3.jpg" width="70%">
</div>

And to make other words *italic* with Markdown. 

<div class="imgcap">
<img src="/assets/Infectome_4.jpg" width="70%">
</div>

#### Code

<div class="imgcap">
<img src="/assets/Infectome_5.jpg" width="70%">
</div>

Test `time.sleep`

```python
import codecs
codecs.open(filename, 'w', 'utf-8').write(tweet_texts)
```
