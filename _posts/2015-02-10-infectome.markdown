---
layout: post
comments: true
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

PhDs can seem like strange endevours. This is my attempt to explain what I did and why.

####The cell-free DNA phenomenon

Much of the work for my PhD was rooted in an observation made by two French scientists in 1947. They reported that human blood contains free-floating DNA molecules. These molecules are simply the debris of dead cells that continually get washed into circulation. Little attention was paid to this "cell-free DNA" phenomenon for decades, largely because there wasn't much to do with it. Most of these fragments of DNA are simply "normal", identical to parts of the genome found in nearly every cell in your body.

But, blood is very information-rich. There are around 100 billion of these DNA fragments per mililiter. Though most are normal, some may be dervied from pathogenic (e.g., Cancer) sources or foreign cells (e.g., the fetus in a pregnant woman). The advent of Next-Generation Sequencing (NGS), a very high-throughput DNA seqeucning technology, made it possible to count these molecules at far greater depth than ever before. Deeper NGS-enabled sampling opened the door for new diagnostic applications.

Steve Quake, my advisor, showed that NGS-enabled molecular couting of these fragments can be used to non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0) during pregnancy. Similarly, cancer-associated mutations in cell-free DNA can be used to non-invasively monitor the progress of [tumors](http://www.genome.gov/27556716) and donor-organ derived DNA fragments can be monitored as a marker of rejection following [transplantation](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28).

####Taste for problems

All of the applications were united by a common thread: they use deep sampling to obtain statistically significant detection of low-abundance foregin genomes. Around the time I started my PhD, a [rising tide](http://www.ucsf.edu/news/2014/06/114946/faster-dna-sleuthing-saves-critically-ill-boy) of studies made made a strong case for NGS in infectious disease scenarios and [much work](http://www.hmpdacc.org/) was focused on NGS as a tool to measure the human microbiome. It naturally follows that cell-free DNA presents an opportunity for non-invasive detection of infectious diseases. Four appealing features made this an appealing problem to tackle:

* Zero cost data: We already had deep sequencing of blood for thousands of samples, which could be re-mined for non-human sequences.
* Fundemental: The debris from dead cells is detectable in blood. This should include micro-organisms.
* Easily testable: For our patient cohorts, we had indepndent clinical tests for infection against which our measurments could be validated.
* Not obvious: If it were obvious, then it would have been done. This required etablishing meaninful correlations from "exhust" data.

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
