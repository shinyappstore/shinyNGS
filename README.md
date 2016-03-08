<!-- README.md is generated from README.Rmd. Please edit that file -->
Synopsis
========

This package will construct Shiny dashboards for a variety of next-generation sequencing and other applications. But I'm currently porting a large script for RNA-seq type downstream analyses, so for now all it does is produce a heatmap builder, 3D PCA plot, boxplot or dendrogram, as toy examples.

Features
--------

-   A variety of single and multiple-panel Shiny applications- currently heatmap, pca, boxplot, a simple table and an RNA-seq app currently being ported from non-modularised code.
-   Leveraging of libraries such as [DataTables](https://rstudio.github.io/DT/) and [Plotly](https://plot.ly/) for rich interactivity.
-   Takes input in the commonly used `SummarizedExperiment` format.
-   Interface kept simple where possible, with complexity automatically added where required:
    -   Input field clutter reduced with the use of collapses from [shinyBS](https://ebailey78.github.io/shinyBS/index.html) (when installed).
    -   If a list of `SummarizedExperiment`s is supplied (useful in situiations where the features are different beween matrices - e.g. from transcript- and gene- level analyses), a selection field will be provided.
    -   If a selected experiment contains more than one assay, a selector will again be provided.
-   For me: leveraging of [Shiny modules](http://shiny.rstudio.com/articles/modules.html). This makes re-using complex UI components much easier, and maintaining application code is orders of magnitude simpler as a result.

Code Example
============

A basic heatmap builder
-----------------------

To produce a simple heat map using some example data you'd do the following:

``` r
require(airway)
library(shinyngs)
library(shiny)
library(GenomicRanges)

# Get some example data in the form of a StructuredExperiment object

data(airway, package="airway")
se <- airway

# Use Biomart to retrieve some annotation, and add it to the object

library(biomaRt)
attributes <- c(
  "ensembl_gene_id", # The sort of ID your results are keyed by
  "entrezgene", # Will be used mostly for gene set based stuff
  "external_gene_name" # Used to annotate gene names on the plot
)
mart <- useMart(biomart = "ENSEMBL_MART_ENSEMBL", dataset = 'hsapiens_gene_ensembl', host='www.ensembl.org')
annotation <- getBM(attributes = attributes, mart = mart)
annotation <- annotation[order(annotation$entrezgene),]

mcols(se) <- annotation[match(rownames(se), annotation$ensembl_gene_id),]

# Specify some parameters to store in the objects's metadata slot. 

metadata(se)$title = 'An example application'
metadata(se)$transcriptfield = "ensembl_gene_id" 
metadata(se)$entrezgenefield = "entrezgene"
metadata(se)$genefield = "external_gene_name"
metadata(se)$group_vars = c('cell', 'dex', 'albut') 
metadata(se)$default_groupvar = 'dex'

# Prepare the UI and server parts of the Shiny app

app <- prepareApp("heatmap", se)

# Run the Shiny app

shinyApp(app$ui, app$server)
```

An interactive 3D PCA plot
--------------------------

Using the same input data a PCA plot can be produced as follows:

``` r

# Prepare the UI and server parts of the Shiny app

app <- prepareApp("pca", se)

# Run the Shiny app

shinyApp(app$ui, app$server)
```

... and so on for other plotting modules.

A multi-panel RNA-seq app
-------------------------

Multiple modules can be combined to create a more comprehensive application. This is what's been done for the RNA-seq app (work in progress- new panels being added all the time):

``` r
app <- prepareApp('rnaseq', se)
shinyApp(app$ui, app$server)
```

### Adding a gene set filter

It's quite handy to see heat maps based on known gene sets. Assuming you have a bunch of .gmt format gene set files from MSigDB keyed by Entrez ID, you can add a gene set filter to the heatmap controls like:

``` r
gene_sets = list(
  'KEGG' =  "/path/to/MSigDB/c2.cp.kegg.v5.0.entrez.gmt",
  'MSigDB canonical pathway' = "/path/to/MSigDB/c2.cp.v5.0.entrez.gmt",
  'GO biological process' = "/path/to/MSigDB/c5.bp.v5.0.entrez.gmt",
  'GO cellular component' = "/path/to/MSigDB/c5.cc.v5.0.entrez.gmt",
  'GO molecular function' = "/path/to/MSigDB/c5.mf.v5.0.entrez.gmt",
  'MSigDB hallmark'= "/path/to/MSigDB/h.all.v5.0.entrez.gmt"
)

metadata(se)$geneset_files = gene_sets

app <- prepareApp("heatmap", se, params)
shinyApp(app$ui, app$server)
```

This will read in the gene sets (which will take a while first time), and use them to add a filter which will allow users to make heat maps based on known sets of genes. Of course you could make your own .gmt files with custom gene sets.

To make your own Summarized experiment objects
----------------------------------------------

Assuming you have:

-   an expression matrix
-   a data frame of experimental variables with rows matching the columns of the expression matrix
-   a data frame containing annotation, one row for each of the expression matrix

... you can make a StructuredExperiment like:

``` r
se <- SummarizedExperiment(
  assays=SimpleList(expression=expression),
  colData=DataFrame(experiment)
)
mcols(se) <- annotation
```

Running on a shiny server
-------------------------

Just use the commands sets above with `shinyApp()` in a file called app.R in a directory of its own on your Shiny server.

Motivation
==========

Shiny apps are great for NGS and bioinformatics applications in general. But apps can get monstrous when their complexity increases, and it's not always easy to re-use components. This is an effort to create modularised components (e.g. a heatmap with controls), re-used to produce multiple shiny apps.

For example this package currently contains five Shiny modules:

-   `heatmap` - provides controls and a display for making heat maps based on user criteria.
-   `pca` - provides controls and display for an interactive PCA plot.
-   `boxplot` - provides controls and display for an interactive boxplot.
-   `dendro` - a clustering of samples in dendrogram plotted with `ggdendro`}.
-   `simpletable` - a simple display using datatables (via the `DT` package) to show a table and a download button. More complex table displays (with further controls, for example) can build on this module.
-   `selectmatrix` - provides controls and output for subsetting the profided assay data prior to plotting. Called by many of the plotting modules.
-   `sampleselect` - provides a UI element for selecting the columns of the matrix based on sample name or group. Called by the `selectmatrix` module.
-   `geneselect` - provides a UI element for selecing the rows of a matrix based on criteria such as variance. Called by the `selectmatrix` module.
-   `genesets` - provides UI element for selecting gene sets. Called by the `geneselect` module when a user chooses to filter by gene set.
-   `plotdownload` - provides download button to non-Plotly plots (Plotly-driven plots have their own export button)

So for example `heatmap` uses `selectmatrix` to provide the UI controls to subselect the supplied matrices as well as the code which reads the output of those controls to actually derive the subsetted matrix. Shiny modules make this recycling of code much, much simpler than it would be otherwise.

I intend to provide modules for a number of things I currently use (boxplots, PCA, scatterplots), which can then be simply plugged into many different applications.

Installation
============

``` r
library(devtools)
install_github('pinin4fjords/shinyngs')
```

Contributors
============

This is an experimental embryonic project, but I can be reached on @pinin4fjords with any queries. Other contributors welcome.

License
=======

MIT
