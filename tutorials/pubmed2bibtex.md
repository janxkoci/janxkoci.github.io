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

    ── [1mAttaching core tidyverse packages[22m ──────────────────────────────────────────────────────────────── tidyverse 2.0.0 ──
    [32m✔[39m [34mdplyr    [39m 1.1.4     [32m✔[39m [34mreadr    [39m 2.1.5
    [32m✔[39m [34mforcats  [39m 1.0.0     [32m✔[39m [34mstringr  [39m 1.5.1
    [32m✔[39m [34mggplot2  [39m 3.5.2     [32m✔[39m [34mtibble   [39m 3.3.0
    [32m✔[39m [34mlubridate[39m 1.9.4     [32m✔[39m [34mtidyr    [39m 1.3.1
    [32m✔[39m [34mpurrr    [39m 1.1.0     
    ── [1mConflicts[22m ────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    [31m✖[39m [34mdplyr[39m::[32mfilter()[39m masks [34mstats[39m::filter()
    [31m✖[39m [34mdplyr[39m::[32mlag()[39m    masks [34mstats[39m::lag()
    [36mℹ[39m Use the conflicted package ([3m[34m<http://conflicted.r-lib.org/>[39m[23m) to force all conflicts to become errors
    
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


<dl>
	<dt>$fulljournalname</dt>
		<dd>'Nature'</dd>
	<dt>$pubdate</dt>
		<dd>'2025 Aug'</dd>
</dl>



Later, I will instead use helper functions from the `magrittr` package, such as `use_series` (instead of `$`) or `extract` (instead of `[]`), as they fit better within complex pipelines. So the above looks like this:


```R
esummary[[1]] %>% extract(c("fulljournalname","pubdate"))
```


<dl>
	<dt>$fulljournalname</dt>
		<dd>'Nature'</dd>
	<dt>$pubdate</dt>
		<dd>'2025 Aug'</dd>
</dl>



We can have a look at a few summaries:


```R
esummary %>% 
    extract_from_esummary("pubtype") %>% 
    lapply(paste, collapse = " - ") %>% 
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble
```


<table class="dataframe">
<caption>A tibble: 5 × 2</caption>
<thead>
	<tr><th scope=col>.</th><th scope=col>n</th></tr>
	<tr><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;int&gt;</th></tr>
</thead>
<tbody>
	<tr><td>Journal Article                     </td><td>47</td></tr>
	<tr><td>Journal Article - Review            </td><td> 4</td></tr>
	<tr><td>Historical Article - Journal Article</td><td> 3</td></tr>
	<tr><td>Journal Article - Historical Article</td><td> 2</td></tr>
	<tr><td>Published Erratum                   </td><td> 1</td></tr>
</tbody>
</table>



Here a journals, with a bit of format processing:


```R
esummary %>% 
    extract_from_esummary("fulljournalname") %>% 
    lapply(., function(x) gsub(":.*", "", x)) %>% # some journals seem to use sub-title too
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble %>% head(10)
```


<table class="dataframe">
<caption>A tibble: 10 × 2</caption>
<thead>
	<tr><th scope=col>.</th><th scope=col>n</th></tr>
	<tr><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;int&gt;</th></tr>
</thead>
<tbody>
	<tr><td>Scientific reports                    </td><td>7</td></tr>
	<tr><td>bioRxiv                               </td><td>5</td></tr>
	<tr><td>Nature                                </td><td>4</td></tr>
	<tr><td>Genome biology and evolution          </td><td>3</td></tr>
	<tr><td>PLoS genetics                         </td><td>3</td></tr>
	<tr><td>Current biology                       </td><td>2</td></tr>
	<tr><td>eLife                                 </td><td>2</td></tr>
	<tr><td>Environmental microbiology            </td><td>2</td></tr>
	<tr><td>Genetics                              </td><td>2</td></tr>
	<tr><td>Molecular and biochemical parasitology</td><td>2</td></tr>
</tbody>
</table>



The `source` item may be even more useful, as it's already formatted the way most people like:


```R
esummary %>% 
    extract_from_esummary("source") %>% 
    unlist %>% table %>% sort(decreasing = T) %>% as_tibble %>% head(10)
```


<table class="dataframe">
<caption>A tibble: 10 × 2</caption>
<thead>
	<tr><th scope=col>.</th><th scope=col>n</th></tr>
	<tr><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;int&gt;</th></tr>
</thead>
<tbody>
	<tr><td>Sci Rep              </td><td>7</td></tr>
	<tr><td>bioRxiv              </td><td>5</td></tr>
	<tr><td>Nature               </td><td>4</td></tr>
	<tr><td>Genome Biol Evol     </td><td>3</td></tr>
	<tr><td>PLoS Genet           </td><td>3</td></tr>
	<tr><td>Curr Biol            </td><td>2</td></tr>
	<tr><td>Elife                </td><td>2</td></tr>
	<tr><td>Environ Microbiol    </td><td>2</td></tr>
	<tr><td>Genetics             </td><td>2</td></tr>
	<tr><td>Mol Biochem Parasitol</td><td>2</td></tr>
</tbody>
</table>



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

papers %>% head
```


<table class="dataframe">
<caption>A tibble: 6 × 11</caption>
<thead>
	<tr><th scope=col>uid</th><th scope=col>title</th><th scope=col>fulljournalname</th><th scope=col>source</th><th scope=col>pubdate</th><th scope=col>sortpubdate</th><th scope=col>issue</th><th scope=col>volume</th><th scope=col>pages</th><th scope=col>journalname</th><th scope=col>year</th></tr>
	<tr><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th><th scope=col>&lt;chr&gt;</th></tr>
</thead>
<tbody>
	<tr><td>40604287</td><td>Ancient DNA reveals the prehistory of the Uralic and Yeniseian peoples.                                                                          </td><td>Nature                                   </td><td>Nature  </td><td>2025 Aug   </td><td>2025/08/01 00:00</td><td>8075</td><td>644</td><td>122--132</td><td>Nature           </td><td>2025</td></tr>
	<tr><td>40454862</td><td>Genomic Analysis of Hair Sheep From West/Central Africa Reveals Unique Genetic Diversity and Ancestral Links to Breed Formation in the Caribbean.</td><td>Molecular ecology                        </td><td>Mol Ecol</td><td>2025 Jun 2 </td><td>2025/06/02 00:00</td><td>    </td><td>   </td><td>e17796  </td><td>Molecular ecology</td><td>2025</td></tr>
	<tr><td>40169722</td><td>Performance of qpAdm-based screens for genetic admixture on graph-shaped histories and stepping stone landscapes.                                </td><td>Genetics                                 </td><td>Genetics</td><td>2025 May 8 </td><td>2025/05/08 00:00</td><td>1   </td><td>230</td><td>        </td><td>Genetics         </td><td>2025</td></tr>
	<tr><td>39979458</td><td>Author Correction: The genetic origin of the Indo-Europeans.                                                                                     </td><td>Nature                                   </td><td>Nature  </td><td>2025 Mar   </td><td>2025/03/01 00:00</td><td>8054</td><td>639</td><td>E14     </td><td>Nature           </td><td>2025</td></tr>
	<tr><td>39910300</td><td>The genetic origin of the Indo-Europeans.                                                                                                        </td><td>Nature                                   </td><td>Nature  </td><td>2025 Mar   </td><td>2025/03/01 00:00</td><td>8053</td><td>639</td><td>132--142</td><td>Nature           </td><td>2025</td></tr>
	<tr><td>39091721</td><td>Long shared haplotypes identify the Southern Urals as a primary source for the 10th century Hungarians.                                          </td><td>bioRxiv : the preprint server for biology</td><td>bioRxiv </td><td>2024 Jul 23</td><td>2024/07/23 00:00</td><td>    </td><td>   </td><td>        </td><td>bioRxiv          </td><td>2024</td></tr>
</tbody>
</table>



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
    $ uid             [3m[90m<chr>[39m[23m "40604287"[90m, [39m"40454862"[90m, [39m"40169722"[90m, [39m"39979458"[90m, [39m"39910…
    $ title           [3m[90m<chr>[39m[23m "Ancient DNA reveals the prehistory of the Uralic and …
    $ fulljournalname [3m[90m<chr>[39m[23m "Nature"[90m, [39m"Molecular ecology"[90m, [39m"Genetics"[90m, [39m"Nature"[90m, [39m"…
    $ source          [3m[90m<chr>[39m[23m "Nature"[90m, [39m"Mol Ecol"[90m, [39m"Genetics"[90m, [39m"Nature"[90m, [39m"Nature"[90m, [39m…
    $ pubdate         [3m[90m<chr>[39m[23m "2025 Aug"[90m, [39m"2025 Jun 2"[90m, [39m"2025 May 8"[90m, [39m"2025 Mar"[90m, [39m"2…
    $ sortpubdate     [3m[90m<chr>[39m[23m "2025/08/01 00:00"[90m, [39m"2025/06/02 00:00"[90m, [39m"2025/05/08 00…
    $ issue           [3m[90m<chr>[39m[23m "8075"[90m, [39m""[90m, [39m"1"[90m, [39m"8054"[90m, [39m"8053"[90m, [39m""[90m, [39m"1"[90m, [39m""[90m, [39m""[90m, [39m""[90m, [39m…
    $ volume          [3m[90m<chr>[39m[23m "644"[90m, [39m""[90m, [39m"230"[90m, [39m"639"[90m, [39m"639"[90m, [39m""[90m, [39m"228"[90m, [39m""[90m, [39m""[90m, [39m""[90m,[39m…
    $ pages           [3m[90m<chr>[39m[23m "122--132"[90m, [39m"e17796"[90m, [39m""[90m, [39m"E14"[90m, [39m"132--142"[90m, [39m""[90m, [39m""[90m, [39m"…
    $ journalname     [3m[90m<chr>[39m[23m "Nature"[90m, [39m"Molecular ecology"[90m, [39m"Genetics"[90m, [39m"Nature"[90m, [39m"…
    $ year            [3m[90m<chr>[39m[23m "2025"[90m, [39m"2025"[90m, [39m"2025"[90m, [39m"2025"[90m, [39m"2025"[90m, [39m"2024"[90m, [39m"2024"…
    $ doi             [3m[90m<named list>[39m[23m "10.1038/s41586-025-09189-3"[90m, [39m"10.1111/mec.1779…
    $ authors         [3m[90m<named list>[39m[23m "Zeng TC, Vyazov LA, Kim A, Flegontov P, Sirak …


## Converting to BibTeX with glue
Before I started, this part looked the most difficult, as I couldn't find any easy-to-use library for doing the conversion to BibTeX. But `glue` is so powerful that outputting a well-formated BibTeX was the easiest part of the process - it took me about 20 minutes to figure out all the details. And I never used `glue` before!

> **Note:** When you load `tidyverse` (or just `stringr`) you already get access to `glue` functionality with `stringr::str_glue()`, which is an alias to `glue::glue()`. But I wanted to explicitly use the `glue` library to give it a well-deserved shout-out!

Here, I use the function `glue_data()` - which works on data frames, matrices, or lists - to process our data frame all at once:


```R
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
```

Finally, we can write the resulting bibtex string into a file.


```R
## write it into a file
writeLines(bibtex, "papers.bib")
```

Since some records have missing data for some items (such as pages for electronic-only journals), I pass the final file through the `sed` utility to remove these empty items from all records. This probably won't work on Windows.


```R
## remove empty fields
system("sed -i '/{}/d' papers.bib")
```

## Comments
