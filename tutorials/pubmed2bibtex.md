---
layout: post
title: Getting BibTeX bibliography from PubMed with R, rentrez, and glue
---

{% include toc.html %}

I've spent a few days trying to find a tool that would take a list of publications from PubMed (formatted as either XML or JSON) and convert it to BibTeX, so I could use it for our lab website, built with Jekyll at Github Pages. Our site template uses jekyll-scholar plugin, which can use a BibTeX file to auto-generate list of publications - neat!

I couldn't find any such tool though, or at least not any tool still working in 2025. But I know the `rentrez` library for R could get me part of the way to what I need, and then it's just a matter of some R coding to get the final file, hopefully. In the end, the `glue` library was surprisingly helpful with this task, and turned my greatest worry - BibTeX output - into a trivial excercise. Keep reading to learn how I solved the problem.

We start by loading the necessary libraries:


```R
library(rentrez)
library(tidyverse)
library(magrittr)
library(glue)
```

    ── Attaching core tidyverse packages ──────────────────────────────────────────────────────────────── tidyverse 2.0.0 ──
    ✔ dplyr     1.1.4     ✔ readr     2.1.5
    ✔ forcats   1.0.0     ✔ stringr   1.5.1
    ✔ ggplot2   3.5.2     ✔ tibble    3.3.0
    ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
    ✔ purrr     1.1.0
    ── Conflicts ────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter() masks stats::filter()
    ✖ dplyr::lag()    masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

    Attaching package: ‘magrittr’


    The following object is masked from ‘package:purrr’:

        set_names


    The following object is masked from ‘package:tidyr’:

        extract





## rentrez queries
The `rentrez` library allows an R programmer to query the NCBI databases, including PubMed, directly from R code. This means I can always get the latest results, rather than just converting a data file obtained by other means.

Now, we need a search term:


```R
search_term = "\"flegontov p\"[au] OR \"flegontov pn\"[au]"
```

A quick [refresher on using the search](https://janxkoci.github.io/tutorials/rentrez.html#search--links) - we can use e.g. the `str` function to have a quick look at a structure of the information returned from a search:


```R
entrez_search(db = "pubmed", term = search_term) %>% str
```

    List of 5
     $ ids             : chr [1:20] "40604287" "40454862" "40169722" "39979458" ...
     $ count           : int 57
     $ retmax          : int 20
     $ QueryTranslation: chr "\"flegontov p\"[Author] OR \"flegontov pn\"[Author]"
     $ file            :Classes 'XMLInternalDocument', 'XMLAbstractDocument' <externalptr>
     - attr(*, "class")= chr [1:2] "esearch" "list"


We see the total number of records found under `$count`, while the NCBI IDs themselves are under `$ids`. See also that `retmax` is by default set to 20, so we will save the `count` info to take all the records instead.

Using search, we pull the data into a list. NCBI supports two modes (set with `retmode`) - XML and JSON. XML is the legacy mode, while JSON is the new way of obtaining information, because its structure fits better with data structures in programming languages (such as lists in R, or dictionaries in Python). I use `retmode = "json"` because the resulting list is easier to work with. For example, authors are provided in form of a `data.frame`, rather than a nested list as with XML.


```R
## number of publications
npubs = entrez_search(db = "pubmed", term = search_term)$count
## get ncbi uids
ncbi_ids = entrez_search(db = "pubmed", term = search_term, retmax = npubs)$ids
## get all publications as json, save into list object
esummary = entrez_summary(db = "pubmed", id = ncbi_ids, retmode = "json")
```

The result is a list of records, so getting a total count of publications is as simple as looking at the length of the list:


```R
esummary %>% length
```


57


Looking at the first record, we can see a complex nested list, with some items represented as vectors (e.g. `pubtype`) or even data frames:


```R
esummary[1] %>% str
```

    List of 1
     $ 40604287:List of 43
      ..$ uid              : chr "40604287"
      ..$ pubdate          : chr "2025 Aug"
      ..$ epubdate         : chr "2025 Jul 2"
      ..$ source           : chr "Nature"
      ..$ authors          :'data.frame':	71 obs. of  3 variables:
      .. ..$ name     : chr [1:71] "Zeng TC" "Vyazov LA" "Kim A" "Flegontov P" ...
      .. ..$ authtype : chr [1:71] "Author" "Author" "Author" "Author" ...
      .. ..$ clusterid: chr [1:71] "" "" "" "" ...
      ..$ lastauthor       : chr "Reich D"
      ..$ title            : chr "Ancient DNA reveals the prehistory of the Uralic and Yeniseian peoples."
      ..$ sorttitle        : chr "ancient dna reveals the prehistory of the uralic and yeniseian peoples"
      ..$ volume           : chr "644"
      ..$ issue            : chr "8075"
      ..$ pages            : chr "122-132"
      ..$ lang             : chr "eng"
      ..$ nlmuniqueid      : chr "0410462"
      ..$ issn             : chr "0028-0836"
      ..$ essn             : chr "1476-4687"
      ..$ pubtype          : chr [1:2] "Journal Article" "Historical Article"
      ..$ recordstatus     : chr "PubMed - indexed for MEDLINE"
      ..$ pubstatus        : chr "256"
      ..$ articleids       :'data.frame':	6 obs. of  3 variables:
      .. ..$ idtype : chr [1:6] "pubmed" "mid" "pmc" "pmcid" ...
      .. ..$ idtypen: int [1:6] 1 8 8 5 3 4
      .. ..$ value  : chr [1:6] "40604287" "NIHMS2095973" "PMC12342343" "pmc-id: PMC12342343;manuscript-id: NIHMS2095973;" ...
      ..$ history          :'data.frame':	6 obs. of  2 variables:
      .. ..$ pubstatus: chr [1:6] "received" "accepted" "medline" "pubmed" ...
      .. ..$ date     : chr [1:6] "2023/09/12 00:00" "2025/05/23 00:00" "2025/08/07 06:26" "2025/07/03 06:28" ...
      ..$ references       : list()
      ..$ attributes       : chr "Has Abstract"
      ..$ pmcrefcount      : int 83
      ..$ fulljournalname  : chr "Nature"
      ..$ elocationid      : chr "doi: 10.1038/s41586-025-09189-3"
      ..$ doctype          : chr "citation"
      ..$ srccontriblist   : list()
      ..$ booktitle        : chr ""
      ..$ medium           : chr ""
      ..$ edition          : chr ""
      ..$ publisherlocation: chr ""
      ..$ publishername    : chr ""
      ..$ srcdate          : chr ""
      ..$ reportnumber     : chr ""
      ..$ availablefromurl : chr ""
      ..$ locationlabel    : chr ""
      ..$ doccontriblist   : list()
      ..$ docdate          : chr ""
      ..$ bookname         : chr ""
      ..$ chapter          : chr ""
      ..$ sortpubdate      : chr "2025/08/01 00:00"
      ..$ sortfirstauthor  : chr "Zeng TC"
      ..$ vernaculartitle  : chr ""
      ..- attr(*, "class")= chr [1:2] "esummary" "list"


And here are names of all the items that can be obtained from a record:


```R
esummary[[1]] %>% names %>% print
```

     [1] "uid"               "pubdate"           "epubdate"
     [4] "source"            "authors"           "lastauthor"
     [7] "title"             "sorttitle"         "volume"
    [10] "issue"             "pages"             "lang"
    [13] "nlmuniqueid"       "issn"              "essn"
    [16] "pubtype"           "recordstatus"      "pubstatus"
    [19] "articleids"        "history"           "references"
    [22] "attributes"        "pmcrefcount"       "fulljournalname"
    [25] "elocationid"       "doctype"           "srccontriblist"
    [28] "booktitle"         "medium"            "edition"
    [31] "publisherlocation" "publishername"     "srcdate"
    [34] "reportnumber"      "availablefromurl"  "locationlabel"
    [37] "doccontriblist"    "docdate"           "bookname"
    [40] "chapter"           "sortpubdate"       "sortfirstauthor"
    [43] "vernaculartitle"


The items can be manipulated by names, as follows:


```R
esummary[[1]][c("fulljournalname","pubdate")]
```


    $fulljournalname
    [1] "Nature"

    $pubdate
    [1] "2025 Aug"




Later, I will instead use helper functions from the `magrittr` package, such as `use_series` (instead of `$`) or `extract` (instead of `[]`), as they fit better within complex pipelines. So the above example will look like this:


```R
esummary[[1]] %>% extract(c("fulljournalname","pubdate"))
```


    $fulljournalname
    [1] "Nature"

    $pubdate
    [1] "2025 Aug"



We can have a look at a few summaries:


```R
esummary %>%
    extract_from_esummary("pubtype") %>%
    lapply(paste, collapse = " - ") %>%
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble %>% print
```

    # A tibble: 5 × 2
      .                                        n
      <chr>                                <int>
    1 Journal Article                         47
    2 Journal Article - Review                 4
    3 Historical Article - Journal Article     3
    4 Journal Article - Historical Article     2
    5 Published Erratum                        1


Here are journals, with a bit of format processing:


```R
esummary %>%
    extract_from_esummary("fulljournalname") %>%
    lapply(., function(x) gsub(":.*", "", x)) %>% # some journals seem to use sub-title too
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble %>% print
```

    # A tibble: 31 × 2
      .                                            n
      <chr>                                    <int>
    1 "Scientific reports"                         7
    2 "bioRxiv "                                   5
    3 "Nature"                                     4
    4 "Genome biology and evolution"               3
    5 "PLoS genetics"                              3
    6 "Current biology "                           2
    7 "eLife"                                      2
    8 "Environmental microbiology"                 2
    9 "Genetics"                                   2
    10 "Molecular and biochemical parasitology"     2
    # ℹ 21 more rows



The `source` item may be even more useful, as it's already formatted the way most people like:


```R
esummary %>%
    extract_from_esummary("source") %>%
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble %>% print
```

    # A tibble: 31 × 2
      .                         n
      <chr>                 <int>
    1 Sci Rep                   7
    2 bioRxiv                   5
    3 Nature                    4
    4 Genome Biol Evol          3
    5 PLoS Genet                3
    6 Curr Biol                 2
    7 Elife                     2
    8 Environ Microbiol         2
    9 Genetics                  2
    10 Mol Biochem Parasitol     2
    # ℹ 21 more rows



## Extracting data
As we have seen, the data returned by PubMed have structure of a nested list. Some items are at the top level, like `fulljournalname` or `title`, while other items are nested within the list, like `authors` or `articleids`.

Getting all items at once is not easy, so a better approach is to extract first the easy items into a data frame, and then add new columns with processed information from the more complex items.

For example, to extract DOI, we need to parse a data frame out of the list first:


```R
esummary[[1]] %>% use_series(articleids)
```


<table class="dataframe">
<caption>A data.frame: 6 × 3</caption>
<thead>
	<tr><th></th><th scope=col>idtype</th><th scope=col>idtypen</th><th scope=col>value</th></tr>
	<tr><th></th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;int&gt;</th><th scope=col>&lt;chr&gt;</th></tr>
</thead>
<tbody>
	<tr><th scope=row>1</th><td>pubmed</td><td>1</td><td>40604287                                        </td></tr>
	<tr><th scope=row>2</th><td>mid   </td><td>8</td><td>NIHMS2095973                                    </td></tr>
	<tr><th scope=row>3</th><td>pmc   </td><td>8</td><td>PMC12342343                                     </td></tr>
	<tr><th scope=row>4</th><td>pmcid </td><td>5</td><td>pmc-id: PMC12342343;manuscript-id: NIHMS2095973;</td></tr>
	<tr><th scope=row>5</th><td>doi   </td><td>3</td><td>10.1038/s41586-025-09189-3                      </td></tr>
	<tr><th scope=row>6</th><td>pii   </td><td>4</td><td>10.1038/s41586-025-09189-3                      </td></tr>
</tbody>
</table>



Extracting just the DOI then looks like this:


```R
esummary[[1]] %>% use_series(articleids) %>%
    subset(idtype == "doi", select = value) %>%
    pluck(1) # return scalar, not list
```


'10.1038/s41586-025-09189-3'


Similarly, to extract all authors as single string:


```R
esummary[[1]] %>% use_series(authors) %>% pull(name) %>% paste(collapse = ", ")
```


'Zeng TC, Vyazov LA, Kim A, Flegontov P, Sirak K, Maier R, Lazaridis I, Akbari A, Frachetti M, Tishkin AA, Ryabogina NE, Agapov SA, Agapov DS, Alekseev AN, Boeskorov GG, Derevianko AP, Dyakonov VM, Enshin DN, Fribus AV, Frolov YV, Grushin SP, Khokhlov AA, Kiryushin KY, Kiryushin YF, Kitov EP, Kosintsev P, Kovtun IV, Makarov NP, Morozov VV, Nikolaev EN, Rykun MP, Savenkova TM, Shchelchkova MV, Shirokov V, Skochina SN, Sherstobitova OS, Slepchenko SM, Solodovnikov KN, Solovyova EN, Stepanov AD, Timoshchenko AA, Vdovin AS, Vybornov AV, Balanovska EV, Dryomov S, Hellenthal G, Kidd K, Krause J, Starikovskaya E, Sukernik R, Tatarinova T, Thomas MG, Zhabagin M, Callan K, Cheronet O, Fernandes D, Keating D, Candilio F, Iliev L, Kearns A, Özdoğan KT, Mah M, Micco A, Michel M, Olalde I, Zalzala F, Mallick S, Rohland N, Pinhasi R, Narasimhan VM, Reich D'


We can use the above code within a `lapply` call, which we will later use to add columns to our table, like this:


```R
esummary %>% head %>%
    lapply(function(x) use_series(x, authors) %>% pull(name) %>% paste(collapse = ", ")) %>% str
```

    List of 6
     $ 40604287: chr "Zeng TC, Vyazov LA, Kim A, Flegontov P, Sirak K, Maier R, Lazaridis I, Akbari A, Frachetti M, Tishkin AA, Ryabo"| __truncated__
     $ 40454862: chr "Wiener P, Friedrich J, Marr MM, Simo G, Tanya VN, Ballingall KT, Flegontov P, Rosen BD, Sallé G, Spangler G, Va"| __truncated__
     $ 40169722: chr "Flegontova O, Işıldak U, Yüncü E, Williams MP, Huber CD, Kočí J, Vyazov LA, Changmai P, Flegontov P"
     $ 39979458: chr "Lazaridis I, Patterson N, Anthony D, Vyazov L, Fournier R, Ringbauer H, Olalde I, Khokhlov AA, Kitov EP, Shishl"| __truncated__
     $ 39910300: chr "Lazaridis I, Patterson N, Anthony D, Vyazov L, Fournier R, Ringbauer H, Olalde I, Khokhlov AA, Kitov EP, Shishl"| __truncated__
     $ 39091721: chr "Gyuris B, Vyazov L, Türk A, Flegontov P, Szeifert B, Langó P, Mende BG, Csáky V, Chizhevskiy AA, Gazimzyanov IR"| __truncated__


### Extract selected items into a table

Some useful-looking items include:
- authors
- pubdate or sortpubdate
- title
- fulljournalname or source
- issue
- volume
- pages
- articleids

I will start with the simple items first, puting them into a data frame. I will also tweak a few of them with `mutate`, e.g. to extract a year out of the publication date. I also noticed in our website template that BibTeX seems to use double dashes (`--`) to represent the longer m-dash for range of pages, so I'm tweaking that too.


```R
papers <- esummary %>%
    lapply(extract, c("uid", "title", "fulljournalname", "source","pubdate", "sortpubdate", "issue", "volume", "pages")) %>%
    bind_rows %>%
    mutate(
        journalname = stringr::str_remove(fulljournalname, ":.*"),
        year = stringr::str_extract(sortpubdate, "\\d{4}"),
        pages = stringr::str_replace(pages, "-","--")
    )

papers %>% print
```

    # A tibble: 57 × 11
      uid      title  fulljournalname source pubdate sortpubdate issue volume pages
      <chr>    <chr>  <chr>           <chr>  <chr>   <chr>       <chr> <chr>  <chr>
    1 40604287 Ancie… Nature          Nature 2025 A… 2025/08/01… "807… "644"  "122…
    2 40454862 Genom… Molecular ecol… Mol E… 2025 J… 2025/06/02… ""    ""     "e17…
    3 40169722 Perfo… Genetics        Genet… 2025 M… 2025/05/08… "1"   "230"  ""
    4 39979458 Autho… Nature          Nature 2025 M… 2025/03/01… "805… "639"  "E14"
    5 39910300 The g… Nature          Nature 2025 M… 2025/03/01… "805… "639"  "132…
    6 39091721 Long … bioRxiv : the … bioRx… 2024 J… 2024/07/23… ""    ""     ""
    7 39013011 Testi… Genetics        Genet… 2024 S… 2024/09/04… "1"   "228"  ""
    8 38659893 The G… bioRxiv : the … bioRx… 2024 A… 2024/04/18… ""    ""     ""
    9 38014190 Testi… bioRxiv : the … bioRx… 2023 N… 2023/11/15… ""    ""     ""
    10 37904998 Perfo… bioRxiv : the … bioRx… 2025 F… 2025/02/03… ""    ""     ""
    # ℹ 47 more rows
    # ℹ 2 more variables: journalname <chr>, year <chr>



We see some missing data, e.g. in columns issue, volume, or pages. We can handle them later in a few ways.

Now, we can add more columns with information parsed from the nested data frames:


```R
papers$doi <- esummary %>%
    lapply(function(x) use_series(x, articleids) %>% subset(idtype == "doi", select = value) %>% pluck(1))

papers$authors <- esummary %>%
    lapply(function(x) use_series(x, authors) %>% pull(name) %>% paste(collapse = ", ")) #%>% str

papers %>% glimpse
```

    Rows: 57
    Columns: 13
    $ uid             <chr> "40604287", "40454862", "40169722", "39979458", "39910…
    $ title           <chr> "Ancient DNA reveals the prehistory of the Uralic and …
    $ fulljournalname <chr> "Nature", "Molecular ecology", "Genetics", "Nature", "…
    $ source          <chr> "Nature", "Mol Ecol", "Genetics", "Nature", "Nature", …
    $ pubdate         <chr> "2025 Aug", "2025 Jun 2", "2025 May 8", "2025 Mar", "2…
    $ sortpubdate     <chr> "2025/08/01 00:00", "2025/06/02 00:00", "2025/05/08 00…
    $ issue           <chr> "8075", "", "1", "8054", "8053", "", "1", "", "", "", …
    $ volume          <chr> "644", "", "230", "639", "639", "", "228", "", "", "",…
    $ pages           <chr> "122--132", "e17796", "", "E14", "132--142", "", "", "…
    $ journalname     <chr> "Nature", "Molecular ecology", "Genetics", "Nature", "…
    $ year            <chr> "2025", "2025", "2025", "2025", "2025", "2024", "2024"…
    $ doi             <named list> "10.1038/s41586-025-09189-3", "10.1111/mec.1779…
    $ authors         <named list> "Zeng TC, Vyazov LA, Kim A, Flegontov P, Sirak …


## Converting to BibTeX with glue
The basic bibtex format for one record (in this case an article) looks as follows:

```
@article{einstein1905electrodynamics,
  title={On the electrodynamics of moving bodies},
  author={Einstein, A.},
  journal={Ann. Phys.},
  year={1905},
  number={17},
  volume={10},
  pages={891--921}
}
```

Before I started, this part looked the most difficult, as I couldn't find any easy-to-use library for doing the conversion to BibTeX. But `glue` is so powerful that outputting a well-formated BibTeX was the easiest part of the process - it took me about 20 minutes to figure out all the details. And I never used `glue` before!

> **Note:** When you load `tidyverse` (or just `stringr`) you already get access to `glue` functionality with `stringr::str_glue()`, which is an alias to `glue::glue()`. But I wanted to explicitly use the `glue` library to give it a well-deserved shout-out!

Note that both BibTeX and `glue` use braces `{}` in their syntax, which can lead to collision, so I need to escape the braces in my `glue` code. This is done by triplicating them, so we use e.g. `{{{uid}}}` for the article ID. This is because the innermost pair of braces is interpreted by `glue` as a variable to be interpolated, while the two outer pairs insert literal braces required by BibTeX.

Moreover, I use the function `glue_data()` here - which works on data frames, matrices, or lists - to process our data frame all at once:

{% highlight R %}
{% raw %}
bibtex <- papers %>%
    glue_data("
@article{{{uid},
    authors={{{authors}}},
    year={{{year}}},
    title={{{title}}},
    journal={{{source}}},
    number={{{issue}}},
    volume={{{volume}}},
    pages={{{pages}}},
    doi={{{doi}}}
}}
")
{% endraw %}
{% endhighlight %}

Finally, we can write the resulting bibtex string into a file.


```R
## write it into a file
writeLines(bibtex, "papers.bib")
```

Since some records have missing data for some items (such as pages for electronic-only journals), I pass the final file through the `sed` utility to remove these empty items from all records:


```R
## remove empty fields
system("sed -i '/{}/d' papers.bib")
```

This probably won't work on Windows though (unless you run R in WSL). There should be a way to remove these empty fields directly in R, but I thought I've done enough coding for this project already, so I went with a simple Unix one-liner that I know will do the job just fine. It may also be the case that the Jekyll-scholar plugin can handle empty fields without issues, but I haven't tested it.

One final corner case involves another detail - if any of the papers in my examples did not have a DOI, then the corresponding field would get deleted, which would leave a trailing comma at the end of the preceding line, which would lead to an error. This would be true for any item that you put as last, so be careful about missing data in this last item. I've checked that all the papers have DOI included, so it's not an issue here.

Finally, this page was made with [Jupyter notebook](https://github.com/janxkoci/janxkoci.github.io/blob/master/notebooks/pubmed2bibtex.ipynb), which you can use to run your own queries. I have also prepared a brief version of the notebook, skipping over the exploratory parts, along with a script to run directly (e.g. in Rstudio, if you prefer), which you can get [here](https://gist.github.com/janxkoci/f6538194706103447b9826b653e6d7db).

## Comments
