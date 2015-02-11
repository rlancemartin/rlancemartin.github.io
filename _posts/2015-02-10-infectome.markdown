---
layout: post
comments: true
title:  "Building a counter for infectious disease"
excerpt: "High throughput measurment of infection-derived DNA in human blood."
date:   2015-02-10 10:19:00
---

####The cell-free DNA phenomenon

Much of the work for my PhD was rooted in an observation made by two French scientists in 1947. They reported that human blood contains free-floating "cell-free" DNA molecules. These molecules are the debris of dead cells that continually get washed into circulation throughout the body. Blood is packed full of them: there are ~100 billion of these DNA fragments per mililiter.

Sometimes these molecules can come from interesting sources. For example, during pregnancy up to 10% of cell-free DNA comes from the fetus. In the 1990s, researchers began to consider whether this can be used to detect chromosonal disorders, such as Down Syndrome. Existing non-invasive tests have limited reliability and invasive testing has a reasonable risk of miscarriage.

In 2008, Steve Quake (my advisor) showed that deep molecular couting of these fragments using Next-Generation DNA Sequencing (NGS) can non-invasively [detect Down Syndrome](http://well.blogs.nytimes.com/2014/02/26/new-dna-test-better-at-predicting-some-disorders-in-babies-study-finds/?_r=0) with high statistical confidence. This was commercialized and has become a [multi-billion dollar market](http://www.xconomy.com/san-francisco/2014/12/02/roche-enters-noninvasive-prenatal-test-market-with-ariosa-purchase/).

####Taste: Why work on a probem?

Around the time I started my PhD, a [several](http://www.hmpdacc.org/) studies were applying NGS to the human microbiome and infectious disease. It was also clear that NGS applied to cell-free DNA can be used for non-invasive detection of foreign genomes: the Quake lab was completing a study on organ transplants, which showed that the level of donor-organ derived cell-free DNA correlates with [transplant rejection](http://www.marketwatch.com/story/caredx-presents-cell-free-dna-biomarker-results-in-heart-and-kidney-transplant-recipients-at-the-world-transplant-congress-2014-07-28).

We thought merging these two threads made sense. We planned to build a universal non-invasive diagnostic for infectious disease by applying NGS to cell-free DNA for at least three reasons:

* ***Fundemental:*** We knew that DNA from dead human cells is shed into blood. Micro-organisms should be subject to this same process. The question made physical sense.

* ***Zero cost data:*** We had cell-free DNA sequencing data for thousands of transplant samples. We could mine these and extract the non-human sequencess that were not previously analyzed. 

* ***Easily testable:*** We had indepndent clinical tests for infection disease for most of the samples. The non-human cell-free DNA signal could be correlated with these tests.

####Re-mining 

<div class="imgcap">
<img src="/assets/Infectome_1.jpg" width="100%">
</div>

We started with thousands of cell-free DNA datasets from from organ transplant patients. The majority of cell-free DNA fragments in each sample were human-derived (black, above). Only a small fraction came from micro-organisms (red). In most samples, ~30 million DNA fragments were sequenced, resulting in files of short (100-200) character strings (A,C,T, or G). 

Human-derived sequences were previously isolated from these samples using string matching algorithms. These align each strings to a human ["reference genome"](http://en.wikipedia.org/wiki/Reference_genome) and keep those with a certain alignment quality (deemed human sequences). Everything else was discarded.

We simply re-designed the pipeline to capture the "junk" that is normally discarded. These we then [aligned](http://en.wikipedia.org/wiki/BLAST) these trace remaining fragments to a availabile databases of micro-organism genomes, yielding a dictionary (a count of strings associated with each identified micro-organism). 

####Data management

Across all samples, this yielded thousands of count dictionaries with up to thousands of unique identified micro-organisms in each. To further complicate this, each micro-organism exists within a taxonomic tree. For example, a sample may contain dozens of different E. coli species. 

For analysis, we wanted the ability to aggregate these counts at different level of taxonomic resolution. We also needed to couple each count with assocaited patient -metadata, thousands of clinical test results, and pipeline log files for steps such as count normalization.

To deal with this, we used `Postgres` along with two excellent libraries, [`Pandas`](http://pandas.pydata.org/) and [`SQLAlchemy`](http://www.sqlalchemy.org/). These three tools alone made it possible to easily weave data sources together via `SQL` queries and immediatly have the output available as a `Pandas` dataframe.

For example, below is a common query required to consolodate data from the various sources: we grab counts for all micro-organisms within a specified patient at a specified taxonomic level of resolution (e.g., species), group by infection ID and sample ID, and sum all counts per group. 

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

####Why is a lot of data useful?

Infections are interesting because there are many categories: there are many different pathogens and subtle differences in the genome sequence of each one may confer clinically relevant traits (e.g., drug resistance). Unlike existing tests, NGS can detect all possible categories. 

**Significance**. Because no "healthy" baseline yet exists for this kind of measurment, it is challenging to interpet absolute count, particularly for micro-organisms that are not familiar pathogens. One way to address this is simply to build out the distribution by acquiring more samples. To aid interpretation, each count can be re-cast as a percentile relative all prior processed samples. As more samples are collected, the percentile become increacingly effective at identifying anomolies.

**Unbiased**. To examine the benefit of unbiased measurment, we looked at the number of samples in which a each virus was detected in our data relative to the number of samples in which that same virus was tested clinically (for lung transplants). The most commonly clinically screened virus (HHV-5, tested in 335 samples) was detected (based upon our data) in 22 samples, an incidence similar to several pathogens that were not routinely screened, such as Adenovirus and Polyomavirus (clinically tested on four occasions and one occasion, respectively). We also detect many un-tested pathogens.

<div class="imgcap">
<img src="/assets/Infectome_2.jpg" width="100%">
</div>

####But, is it real?

Many blood samples were collected longitudinally for the same patient, allowing us to record time-series for each measured infection. Sustained, high abundance in the time-series data provided an early indication that the measurments may correlated with infections within the body. By overlaying independent clinical data, we often observed a sustained signal in blood prior to clinical detection of the specified pathogen (below left). We also could validate our measurments using indepdent clinical tests performed on the same patients (clinical correlations on HHV-5, below right). 

<div class="imgcap">
<img src="/assets/Infectome_3.jpg" width="70%">
</div>

####Building a data product

**Defining the organization**. After validating that the signal is clinically meaningful for infections that had matched clinical tests, we addressed the practical challenge of building a tool that clinicians and resarchers could use. We noted that the data has natural structure: each cohort was comprised of patients, which may have many samples. In each sample, there may be thousands of unique infections identified. 

**Encoding the questions**. Each layer of organization had associated questions. We built a `Django` application that split the data according to these different layers of organization and designed objects (tables and visualizations) that address relevant questions at that layer of organization. 

**Designing the views**. In terms of basic design, each view (page) in the application sources a common `.html` base template, which sources `bootstrap` javascript and css templates. These take care of the basic layout. Each view is also coupled to a particular `.html` template, url, and function in the `views.py` script. 

Each function in `views.py` recieves parameters from the url string (e.g., `p1` and `p2` below), generates objects (e.g., tables or graphs) needed to address the relevant questions, and returns objects to the `.html` template, which handles layout. 

Conviently, these objects can be Pandas dataframes using the `to_html()` method. By tagging a dataframe with relevant .css tags, it will be rendered nicely on the page. In many cases, we easily be embedded links in table entries (below) and use `Dynatable.js`, interactive table plugin using jQuery. 

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

With these basic tools, each view presented our choice of table or plots, directly pulled from the Python code. For the latter, we took advantage of python visualization libraries, including `Matplotlib` and `Seaborn`. As a simple example, the patient view presented a sortable and searable table of all infections detected in that patient, using intutive percentile measurments. 

<div class="imgcap">
<img src="/assets/Infectome_4.jpg" width="70%">
</div>

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



With these basics in place, it was also easy to design views with more complex organization. For example, the below view is accessed from the patient table. In this case, we are curious to learn more information about a particular infection in a patient. The view presents timeseries data (below) as well as coverage, which indicates the actual distribution of raw measurments across the organism genome.

<div class="imgcap">
<img src="/assets/Infectome_5.jpg" width="70%">
</div>

####Summary


