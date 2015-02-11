---
layout: post
comments: true
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

####The cell-free DNA phenomenon

Much of the work for my PhD was rooted in an observation made by two French scientists in 1947. They reported that human blood contains free-floating "cell-free" DNA molecules. These molecules are the debris of dead cells that continually get washed into circulation. Blood is packed full of them: there are ~100 billion of these DNA fragments per mililiter.

In certain contexts, these molecules can come from interesting sources. For example, during pregnancy up to 10% of cell-free DNA comes from the fetus. In the 1990s, researchers began to develop non-invasive genetic tests based upon this fact. Chromosonal disorders, such as Down Syndrome, were an obvious target; non-invasive tests have limited reliability and invasive testing has a risk of miscarriage.

Down Syndrome is caused by the duplication of a chromosome, which could lead to a slightly elevated signal from this chromosome in maternal blood. But, the high background of maternal DNA made this difference challenging to detect with statistical confidence. In the mid-2000s, Next-Generation Sequencing (NGS), a very high-throughput DNA sequeucning technology, made it possible sample cell-free DNA at very high depth. 

In 2008, Steve Quake (my advisor) showed that deep molecular couting of these fragments using NGS can non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0). Due to its high reliability and superiroity to invasive methods, sequencing cell-free DNA has been rapidly commercialized. Over one million women are now tested per year.

####Taste for problems

Around the time I started my PhD, a [rising tide](http://www.hmpdacc.org/) of studies were using NGS to study the human microbiome and infectious disease. It was also clear that cell-free DNA is broadly useful for non-invasive detection of foreign genomes. For example, Quake lab had completed a large study on organ transplants, showing that donor-organ derived cell-free DNA can be resolved using donor-specific DNA mutations. The level of this signal correlates with [organ damage and rejection](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28).

We thought that it should be possible to merge these two threads and build a non-invasive diagnostic for infectious disease by applying NGS to cell-free DNA. At least three features made this a good problem to tackle:

* ***Fundemental:*** We knew that DNA from dead human cells is shed into blood. Micro-organisms should be subject to this same process. The question made physical sense.

* ***Zero cost data:*** We already had cell-free DNA sequencing data for thousands of transplant samples. We simply had to mine these and extract the "exhuast" data (non-human cell-free DNA sequences) that was normally discarded. 

* ***Easily testable:*** We had indepndent clinical tests for infection disease for most of the samples. The non-human cell-free DNA signal could be correlated with these tests in order to determine whether it is clinically informative.

####Re-mining 

<div class="imgcap">
<img src="/assets/Infectome_1.jpg" width="100%">
</div>

We had thousands of cell-free DNA samples sequenced from organ transplant patients. The majority of cell-free DNA fragments in each sample were human-derived (black, above) and a small fraction come from micro-organisms (red). In most of our samples, ~30 million cell-free DNA fragments were sequenced and captured in file (typically a few GB) of short (100-200) character strings that indicate the DNA base composition (A,C,T, or G) of each molecule. 

Previously, the human-derived sequences in these datasets were isolated and analyzed. This process uses the human reference, a ["representitive"](http://en.wikipedia.org/wiki/Reference_genome) human genome sequence. The pool of strings are matched to this reference using alignment algoriths and the un-aligned strings are discarded. We inverted this: we discarded all human-derived sequences. We then [aligned](http://en.wikipedia.org/wiki/BLAST) the trace remainder to databases of micro-organism genomes, yielding a dictionary (a count of strings associated with each identified micro-organism). 

####Data management

This process resulted in thousands of count dictionaries for our samples, each with up to thousands of unique identified micro-organisms. To further complicate this, each micro-organism exists within a taxonomic tree. For example, a sample may contain dozens of different E. coli species. For analysis, we want the ability to aggregate these counts at different level of taxonomic resolution, such as genus or class.

In addition, we collected patient -metadata, thousands of de-identified clinical test results, and pipeline log files required for normalization of counts (e.g., to account for differences in sampling depth). All of this information had to be assocaited sample counts. To collate all of this information, we used `Postgres` with two excellent libraries, [`pandas`](http://pandas.pydata.org/) and [`SQLAlchemy`](http://www.sqlalchemy.org/). 

Coupling `Postgres` to `pandas` via `SQLAlchemy` was very convient. All the SQL logic is simply passed to `Postgres` (along with specified paramters in the python script) and `pandas` dataframes are returned. For example, below is a common query required to consolodate data from the various sources: we grab counts for all micro-organisms within a specified patient at a specified `level` of taxonomic resolution.

```python

# Connection object
from sqlalchemy import create_engine
engine = create_engine('postgresql://lmartin@localhost:5432/infectome_db')

# Pull data for patient 
p=sql.read_sql("select Count_Table.Sample_ID,Tax_Table.%s,SUM(Count_Table.Counts) "
                 "from Tax_Table "
                 "INNER JOIN Count_Table "
                 "ON Tax_Table.Tax_ID=Count_Table.Tax_ID "
                 "WHERE Count_Table.Patient_ID = '%s' "
                 "GROUP BY Count_Table.Sample_ID,Tax_Table.%s "%(level,patient,level),engine)

```

####Why is "big data" relevant here?

Though we had a suitable means of data organization, it is worth asking why a lot of data is valuable for this application. **Significance**. For each micro-organism detected in a new sequenced sample, we compared the count against all prior measurments of that same micro-organism. This results in a percentile (relative to the cohort), which is an intutive measure of abundance. As more samples are collected, this can be helpful for distinguishing whether a measurment is anomolous. 

**Unbiased**. With deep and un-biased sampling, we found that this approach can detect a broad range of micro-organisms, many of which are potential pathogens that escape conventional testing. To highlight this, we evaluated the incidence of infection (number of samples in which a given virus is detected via sequencing) relative to the clinical screening frequency for a cohort of lung transplant patients. The most commonly clinically screened virus (HHV-5, tested in 335 samples) was detected (based upon our data) in 22 samples. This incidence was similar to that of several other pathogens that were not routinely screened, including adenovirus and polyomavirus (clinically tested on four occasions and one occasion, respectively). We also detect many un-tested pathogens using this approach.

<div class="imgcap">
<img src="/assets/Infectome_2.jpg" width="100%">
</div>

####But, is it real?

It is reasonable to ask whether these measuments are real! Time-series data was an early indication that the level of infection-derived cell-free DNA may reflect bona-fide infections within the body. For many cases, we observed a sustained signal in blood prior to detection with a clincal test (after which treatment was administered and the signal drops, shown below left). The study was also designed such that we could easily validate our measurments using indepdent clinical tests performed on the same patients (clinical correlations on HHV-5, below right). Both provided strong evidence that the measurments were real. 

<div class="imgcap">
<img src="/assets/Infectome_3.jpg" width="70%">
</div>

####How to navigate the data?

We quickly found that navigating the data was very cumbersome, in part because it had several layers of organization: each cohort was comprised of patients, which in turn may have many samples. In each sample, there may be thousands of unique infections identified in the cell-free DNA sequencing. Furthermore, infections may be viewed at different levels of taxonomic complexity, such as genus or species level resolution. 

To address this, we built a `Django` application such that the views were organized according to these different layers of organization. Each view (page)in the application sources from a common `.html` base template. The base templates sources `bootstrap` javascript and css templates from the  project's `static` directory; these do most of the nice layout work!

Each view `.html` template is associated with a particular url and a corresponding function in the `views.py` script. The funtion recieves parameters from the url string (e.g., `p1` and `p2` below) and return various objects to the `.html`. Conviently, these objects can be Pandas dataframes using `to_html`. By tagging the dataframes with relevant .css tags, they can be rendered as attractive tables on the page. ALso links to other views can easily be embedded based upon table entries (below). 

```python
def my_view(request,p1,p2):

	### Pull relevant data to make table, t ###

	# Add a link to another view in the table
	t['Name']=t['Name'].apply(lambda x:"<a href='%s/?level=%s'>%s</a>"% (x,level,getName(x)))
	# Write to table to html
	t_html=t.to_html(classes=['table','table-striped','table-condensed'],index=False)
	# Pass it the html template for this view
	context_dict = {'my_table' : t} 
	# Render the response and return to the client
	return render_to_response('infectome_app/cohort.html', context_dict, context)
```

Similarly, plots can be passed back to the `.html` by funtions in `views.py`. 

```python
def my_view_plot(request,p1,p2):

	### Define a figure, fig ###

	canvas=FigureCanvas(fig)
	response=HttpResponse(content_type='image/png')
	canvas.print_png(response,dpi=400)
	plt.close('all')
	return response
```

With these basic tools, each view can easily present any Pandas table or plot. For the latter, we took advantage of python visualization libraries, including `Matplotlib` and a very nice statistical plotting library called `Seaborn`. This simply organization allowed us to present data in view that was reponsive to questions relevat to that layer of organization. For example, the patient view simply presented a sortable and searable table of all infections detected in that patient, using reasonably intutive percentile measurments. 

<div class="imgcap">
<img src="/assets/Infectome_4.jpg" width="70%">
</div>

With these basics in place, it was also easy to design views with more complex organization. For example, the below view is accessed from the patient table. In this case, we are curious to learn more information about a particular infection in a patient. The view presents timeseries data (below) as well as coverage, which indicates the actual distribution of raw measurments across the organism genome.

<div class="imgcap">
<img src="/assets/Infectome_5.jpg" width="70%">
</div>

####Summary


