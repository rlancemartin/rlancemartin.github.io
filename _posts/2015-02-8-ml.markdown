---
layout: post
comments: false
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

####The cell-free DNA phenomenon

Much of my PhD work was rooted in an observation made by two French scientists in 1947. They reported that human blood contains free-floating "cell-free" DNA molecules. These molecules are the debris of dead cells that continually get washed into circulation throughout the body. Blood is packed full of them: there are ~100 billion of these DNA fragments per mililiter of plasma!

Sometimes these molecules come from interesting sources. For example, during pregnancy up to 10% of cell-free DNA actually comes from the fetus. In the 1990s, researchers began to consider whether this phenomenon could be used to detect chromosomal disorders, such as Down Syndrome; existing non-invasive tests had limited reliability and invasive testing had a reasonable risk of miscarriage.

In 2008, Steve Quake (my advisor) showed that deep molecular counting of these fragments using Next-Generation DNA Sequencing (NGS) can non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0) with high statistical confidence. This was commercialized and has become a [multi-billion dollar market](http://www.xconomy.com/san-francisco/2014/12/02/roche-enters-noninvasive-prenatal-test-market-with-ariosa-purchase/).

####Taste: Why work on this probem?

Around the time I started my PhD, [several](http://www.hmpdacc.org/) studies were applying NGS to the human microbiome and infectious disease. It was also clear that NGS applied to cell-free DNA can be used for non-invasive detection of foreign genomes: the Quake lab was completing a study on organ transplants, which showed that the level of donor-organ derived cell-free DNA correlates with [transplant rejection](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28).

We thought merging these two threads made sense. We planned to build a universal non-invasive diagnostic for infectious disease by applying NGS to cell-free DNA for at least three reasons:

* ***Fundamental:*** We knew that DNA from dead human cells is shed into blood. Micro-organisms should be subject to this same process. The question made physical sense.

* ***Zero cost data:*** We had cell-free DNA sequencing data for thousands of transplant samples. We could mine these and extract the non-human sequences that were not previously analyzed. 

* ***Testable:*** We had independent clinical tests for infection disease for most of the samples. The non-human cell-free DNA signal could be correlated with these tests.

####Mining

<figure>
<img src="/assets/Infectome_1.jpg" width="100%">
<figcaption>
</figcaption>
</figure>

We started with thousands of cell-free DNA datasets from from organ transplant patients. The majority of cell-free DNA fragments in each sample were human-derived (black, above). Only a small fraction came from micro-organisms (red). In most samples, ~30 million DNA fragments were sequenced, resulting in large files of short (100-200) character strings (A,C,T, or G). 

Human-derived sequences were previously isolated from these samples using string matching algorithms. These align each strings to a human ["reference genome"](http://en.wikipedia.org/wiki/Reference_genome) and keep those with a certain alignment quality (deemed human sequences). Everything else was discarded.

We simply re-designed the pipeline to capture the "junk" that is normally discarded. We then [aligned](http://en.wikipedia.org/wiki/BLAST) these remaining fragments to a databases of micro-organism genomes, yielding a dictionary: we recorded a count of strings (DNA fragments) associated with each micro-organism. 

####Organizing one million measurments

Our mining resulted in a large nunber of measurments. Below is a sub-set, with counts discretized (0 or 1). Each column is a unique micro-organism species and each row is a unique sample. Both are sorted by sum. In total, there are ~1 million measurments shown. The challenge was to examine whether the data is useful and, if so, organize it so that insights can be easily extracted.

<div class="imgcap">
<img src="/assets/Infectome_1_1.jpg" width="50%">
</div>

####Data management

We collected millions of measurements associated with thousands of unique micro-organisms across hundreds of patients. To further complicate this, each micro-organism exists within a taxonomic tree. For example, a sample may contain dozens of different E. coli species. 

For analysis, we wanted the ability to aggregate these counts at different level of taxonomic resolution. We also needed to couple each count with associated patient -metadata, thousands of clinical test results, and pipeline log files (e.g., for count normalization).

To deal with this, we used `Postgres` along with two excellent libraries, [`Pandas`](http://pandas.pydata.org/) and [`SQLAlchemy`](http://www.sqlalchemy.org/). These three tools alone made it possible to easily weave data sources together via `SQL` queries and immediately have the output available as a `Pandas` dataframe.

For example, below is a common query required to consolidate data from the various sources: we grab counts for all micro-organisms within a specified patient at a specified taxonomic level of resolution (e.g., species), group by infection ID and sample ID, and sum all counts per group. 

```python
def do_something(l,p):
	''' Pull all patient (p) data at tax level (l).'''

	# Connection object
	from sqlalchemy import create_engine
	engine = create_engine('postgresql://lmartin@localhost:5432/infectome_db')

	# Pull data for patient 
	df=sql.read_sql("select Count_Table.Sample_ID,Tax_Table.%s,SUM(Count_Table.Counts) "
	                 "from Tax_Table "
	                 "INNER JOIN Count_Table "
	                 "ON Tax_Table.Tax_ID=Count_Table.Tax_ID "
	                 "WHERE Count_Table.Patient_ID = '%s' "
	                 "GROUP BY Count_Table.Sample_ID,Tax_Table.%s "%(l,p,l),engine)
	
	# Use df ...
```

As we process more samples, we simply write the outputs and logs to `Postgres`. All downstream analysis can easily pull from `Postgres`, with the output dataframe available for further analysis.

####Why is big data useful here?

Infections are interesting because there are many categories: there are many different pathogens and subtle differences in the genome sequence of each one may confer clinically relevant traits (e.g., drug resistance). Unlike conventional clinical tests, NGS can detect all possible categories. 

**Significance**. Because no "healthy" baseline yet exists for this kind of measurement, it is challenging to interpret absolute counts, particularly for micro-organisms that are not familiar pathogens. One way to address this is simply to build out the distribution by acquiring more samples. To aid interpretation, each count can be re-cast as a percentile relative all prior processed samples. As more samples are collected, the percentile become increasingly effective at identifying anomalies.

**Unbiased**. To examine the benefit of unbiased measurement, we looked at the number of samples in which a each virus was detected in our data relative to the number of samples in which that same virus was tested clinically (for lung transplants). The most commonly clinically screened virus (HHV-5, tested in 335 samples) was detected (based upon our data) in 22 samples, an incidence similar to several pathogens that were not routinely screened, such as Adenovirus and Polyomavirus (clinically tested on four occasions and one occasion, respectively). We also detect many un-tested pathogens.

<div class="imgcap">
<img src="/assets/Infectome_2.jpg" width="100%">
</div>

####But, is it real?

Many blood samples were collected longitudinally for the same patient, allowing us to record a time-series for each infection. We saw sustained, high abundance signal for many infections. By overlaying independent clinical tests, we often observed a sustained signal in blood prior to clinical detection of the specified pathogen (below left, with decrease in signal after clinical detection and intervention). We also could validate our measurements using independent clinical tests performed on the same patients (clinical correlations on HHV-5, below right). 

<div class="imgcap">
<img src="/assets/Infectome_3.jpg" width="70%">
</div>

####Building an application around the data

**Data organization**. After validating that the measurements could be clinically meaningful, we addressed the practical challenge of managing this large volume of data. To do this, we built an application that would serve the data such that clinicians and researchers could easily extract insights from it. We designed the application around the basic organization of the data, with natural segmentation of cohorts, patients, samples, and micro-organisms detected within each sample.

**Encoding the questions**. Each layer of organization had associated questions. In addition to splitting the data according to these layers of organization, we designed the application views (tables and visualizations) in order to be responsive to questions at each layer of organization.

**Designing the views**. Each view uses the `bootstrap` framework, which takes care of the basic layout. Each view was also coupled to an associated `.html` template, url, and function in the `views.py` script. Each function receives parameters from the url string for that view (e.g., `p1` and `p2` below), generates objects (e.g., tables or graphs), and returns the objects to the `.html` template.

The objects returned by `views.py` present the data. Conveniently, Pandas dataframes can be re-cast as tables using the `to_html()` method, which adds relevant `css` tags. We embedded links to other views in the table entries (below) and used `Dynatable.js`, an interactive table plugin.

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

With these basic tools, each view could be easily populated with any dataframe or plot directly pulled from python. As a simple example, the patient view presented a sortable and searchable table of all infections detected in that patient (displaying percentile measurments).

<div class="imgcap">
<img src="/assets/Infectome_4.jpg" width="100%">
</div>

For plotting, we took advantage of `Matplotlib` and [`Seaborn`](http://stanford.edu/~mwaskom/software/seaborn/). Conveniently, plots could be passed to any template from `views.py`. 

```python
def my_view_plot(request,p1,p2):

	### Define a figure, fig ###

	canvas=FigureCanvas(fig)
	response=HttpResponse(content_type='image/png')
	canvas.print_png(response,dpi=400)
	plt.close('all')
	return response
```

With these basics in place, it was also easy to design views with more complex organization. For example, the below view provides detailed information about an infection and is accessed directly from the patient table. This view presents timeseries data (below) with overlaid clinical test results. It also displays sequencing statistics from `Postgres`, presenting the distribution of sequencing reads across the microbial genome to evaluate uniformity of sequencing coverage.

<div class="imgcap">
<img src="/assets/Infectome_5.jpg" width="100%">
</div>

####Summary: "Found money"

We built a universal non-invasive diagnostic for infectious disease by applying NGS to cell-free DNA. We mined existing datasets that were generated for a different application. Because of this, the results were "found money," un-claimed value that we were able to validate by correlating our results with existing clinical test results. We built an application to manage this data, which split the data according to an intuitive hierarchy that clinicians and researchers can use to extract insights.
