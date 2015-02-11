---
layout: post
comments: true
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

####The cell-free DNA phenomenon

In 1947, Mandel and Metais first observed that human blood contains circulating cell-free DNA molecules. These fragments enter blood as the detritus of dead cells and circulate with short (15 minute) half-life before being degraded by enzymes. Recent technologies for molecular counting, such as Next-Generation Sequencing (NGS), can read these molecules at very high throughput. This has lead to a surge of data-centric and non-invasive diagnostic tests, which are all rooted in the observation that the abundance of particular cell-free DNA molecules correlates with human health.

The greatest value of these tests has been the ability to measure the proportion of foreign genomes in an individual. Cancer-associated mutations in cell-free DNA can be used to non-invasively monitor the progress of [cancer](http://www.genome.gov/27556716), fetal DNA assayed by molecular counting can be used to non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0), and donor-organ derived DNA fragments can be monitored as a marker of rejection following [transplantation](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28). My PhD was focused on extending this idea to new problems.

####Infectious diease and cell-free DNA

The ability to resolve foreign genomes in blood presents an opportunity for non-invasive infectious disease monitoring. Already, a [rising tide](http://www.ucsf.edu/news/2014/06/114946/faster-dna-sleuthing-saves-critically-ill-boy) of studies have made a strong case for NGS in infectious disease scenarios, such as resistance and outbreak monitoring. In parallel, [much work](http://www.hmpdacc.org/) has focused on NGS as a tool to measure the human microbiome. We extended these studies by developing and validating a way to measure the infections or the human microbiome directly from blood. 

<div class="imgcap">
<img src="/assets/Infectome_1.jpg" width="1000%">
</div>

This is data problem, because the biochemistry and technology needed to isolate and sequence cell-free DNA are well worked out; I barely touched a pipet during my PhD. The challenge (and opportunity) is that human blood is very information rich: there are around 100 billion cell-free DNA fragments per mililiter of blood! We typically sequence around 30 million of these at a time, resulting in a file of digitized DNA fragments (typically a few GB) as short (100-200) character strings. Each string indicates the nucleic acid composition (A,C,T, or G) of a single DNA molecule in the sample. 

Once all DNA molecules in the sample are digitized, we remove all human-derived sequences. This uses the human reference, a ["representitive"](http://en.wikipedia.org/wiki/Reference_genome) human genome sequence. Our pool of strings can be matched to the reference using alignment tools that take advantage of algorithms such as [the Burrows–Wheeler transformation](http://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform). Strings that fail to match the human reference can then be [aligned](http://en.wikipedia.org/wiki/BLAST) to databases of micro-organism genomes. Ultimatly, this process yields a dictionary for each sample, a count of strings associated with each entry in the micro-organism database. 

####Why is "big data" relevant here?

We performed the basic processing described above to thousads of blood samples collected from patients undergoing procedures at Stanford hospital. The dictionary resturned from each sample had up to thousands of unique micro-organisms. Along with this, we collected patient -metadata, including thousands of de-identified clinical test results. These data, along with pipeline log files required to normalization of counts based upon parametrs such as sampling depth, we written to `Postgres`. For this, we took advantage of two excellent libraries, [`pandas`](http://pandas.pydata.org/) and [`SQLAlchemy`](http://www.sqlalchemy.org/). 

This resulted in a large amount of data to organize and understand. The value of data at scale is two-fold. ** Benchmarking. ** Every count recorded for a sample can be compared against every other observed count for that same micro-organism across all other samples. As more samples are collected, we gain greater ability to detect anomolies. ** Unbiased. ** With un-biased sampling, we can detect 

<div class="imgcap">
<img src="/assets/Infectome_2.jpg" width="1000%">
</div>

ability to perform such measurments non-invasively from blood adds to the power of NGS, as blood presents a window into the microbiome of deep tissues that are otherwise inaccessible.

Genome fragments in human blood can be isolated and sequenced. The digitized genome fragments are simply strings of letters, which can be further filtered based upon similarity to known microbial sequencings. Strings that match to pathogens can be counted.



#### Bullet points work!

* Start a line with a star
* Profit!


It's very easy to make some words **bold**.

And to make other words *italic* with Markdown. 



#### Code

Test `time.sleep`

```python
import codecs
codecs.open(filename, 'w', 'utf-8').write(tweet_texts)
```
