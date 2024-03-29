Anaerobic CUE
================
Roey Angel
`2021-02-20`

-   [Differential abundance modelling of SIP
    gradients](#differential-abundance-modelling-of-sip-gradients)
    -   [Setting general parameters:](#setting-general-parameters)
    -   [Read phyloseq object](#read-phyloseq-object)
    -   [Subset the dataset](#subset-the-dataset)
    -   [Beta diversity analysis](#beta-diversity-analysis)
    -   [Differential abundance models](#differential-abundance-models)
        -   [Inspect results](#inspect-results)
        -   [Plot differential abundance
            models](#plot-differential-abundance-models)
    -   [Plot labelled ASVs](#plot-labelled-asvs)
        -   [Plot phylogenetic trees with
            heatmaps](#plot-phylogenetic-trees-with-heatmaps)
        -   [How abundant were the labelled
            ASVs?](#how-abundant-were-the-labelled-asvs)
        -   [Calculate NTI](#calculate-nti)
-   [References](#references)

## Differential abundance modelling of SIP gradients

Here we attempt to detect ASVs that were labelled with <sup>13</sup>C in
our soil incubations using differential abundance modelling. Using
DESeq2 ([Love, Huber and Anders 2014](#ref-love_moderated_2014)) we
compare the relative abundance of each ASV in the fractions where
<sup>13</sup>C-labelled RNA is expected to be found (\>1.795 g
ml<sup>-1</sup>; AKA ‘heavy’ fractions) to the fractions where
unlabelled RNA is expected to be found (\<1.795 g ml<sup>-1</sup>; AKA
‘light’ fractions). The method has been previously described in Angel et
al., ([2018](#ref-angel_application_2018)).

### Setting general parameters:

``` r
set.seed(2021)
alpha_thresh <- 0.05
LFC_thresh <- 0.26
samples_prep_path <- "./"
data_path <- "./DADA2_pseudo/"
# Metadata_table <- "./AnCUE_Metadata_decontam.csv"
# Seq_table <- "DADA2.seqtab_nochim_decontam.tsv"
# Seq_file <- "DADA2.Seqs_decontam.fa"
Ps_file <- "Ps_obj_decontam_filt3.Rds"
Tree_file <- "./Tree/DADA2.Seqs_decontam_filtered.filtered.align.treefile"
```

### Read phyloseq object

``` r
# Load phylogenetic tree
Tree <- read_tree(paste0(data_path, Tree_file))

# load and merge  phyloseq object
readRDS(paste0(data_path, Ps_file)) %>% 
  merge_phyloseq(.,
                 phy_tree(Tree)
  ) -> Ps_obj_SIP
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

### Subset the dataset

Because the DESeq2 models will be run on each gradient separately, we
need to subset This is easily done using `HTSSIP::phyloseq_subset`
([Youngblut, Barnett and Buckley 2018](#ref-youngblut_htssip_2018))

``` r
# split, ignore time points (for labelled ASV plots)
test_expr_1 <- "(Site == '${Site}' & Oxygen == '${Oxygen}' & Label..13C. == 'Unlabelled') | (Site == '${Site}'  & Oxygen == '${Oxygen}' & Label..13C. == '${Label..13C.}')"
params_1 <- get_treatment_params(Ps_obj_SIP, c("Site",
                                   "Oxygen",
                                   "Glucose",
                                   "Label..13C."),
                     "Label..13C. != 'Unlabelled'")
```

    ## Warning: `filter_()` was deprecated in dplyr 0.7.0.
    ## Please use `filter()` instead.
    ## See vignette('programming') for more help
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was generated.

``` r
Ps_obj_SIP_noTime_l <- phyloseq_subset(Ps_obj_SIP, params_1, test_expr_1) 
```

    ## Warning: `mutate_()` was deprecated in dplyr 0.7.0.
    ## Please use `mutate()` instead.
    ## See vignette('programming') for more help
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was generated.

``` r
# names(Ps_obj_SIP_noTime_l) %<>% 
#   map(., ~str_remove_all(.x, "\\s\\|\\s.*")) %>% 
#   map(., ~str_remove_all(.x, "\\(|\\)|Site == |Hours == |Oxygen == |Label..13C. == |'")) %>% 
#   map(., ~str_replace_all(.x, "([0-9]+)", "\\1 h")) 

# split, include time points (for DESeq2 modelling)
test_expr_2 <- "(Site == '${Site}' & Hours == '${Hours}' & Oxygen == '${Oxygen}' & Label..13C. == '${Label..13C.}') | (Site == '${Site}' & Hours == '${Hours}' & Oxygen == '${Oxygen}' & Label..13C. == '${Label..13C.}')"
params_2 <- get_treatment_params(Ps_obj_SIP, c("Site", 
                                   "Hours", 
                                   "Oxygen",
                                   "Glucose",
                                   "Label..13C."))

# Generate a list of subsetted phyloseq objects
Ps_obj_SIP_byTime_l <- phyloseq_subset(Ps_obj_SIP, params_2, test_expr_2) 
names(Ps_obj_SIP_byTime_l) %<>% 
  map(., ~str_remove_all(.x, "\\s\\|\\s.*")) %>% 
  map(., ~str_remove_all(.x, "\\(|\\)|Site == |Hours == |Oxygen == |Label..13C. == |'")) %>% 
  map(., ~str_replace_all(.x, "([0-9]+)", "\\1 h")) 
```

### Beta diversity analysis

Let us look first at the dissimilarity in community composition between
the different fractions. If the labelling was strong enough we should
see a deivation of (some of) the heavy fractions from the light ones.
However, a lack of a significant deviation does not mean unsuccesful
labelling because if only a small minority of the community was labelled
we might not see it here (but we will, hopefully, see it using DESeq2
modelling).

``` r
(mod1 <- adonis2(vegdist(otu_table(Ps_obj_SIP), method = "horn") ~ Site * Oxygen * Hours + Lib.size,
  data = as(sample_data(Ps_obj_SIP), "data.frame"),
  permutations = 999
))
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:right;">
Df
</th>
<th style="text-align:right;">
SumOfSqs
</th>
<th style="text-align:right;">
R2
</th>
<th style="text-align:right;">
F
</th>
<th style="text-align:right;">
Pr(\>F)
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Site
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
27.4431088
</td>
<td style="text-align:right;">
0.5392110
</td>
<td style="text-align:right;">
480.477524
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
5.2895923
</td>
<td style="text-align:right;">
0.1039316
</td>
<td style="text-align:right;">
92.610872
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1596437
</td>
<td style="text-align:right;">
0.0031367
</td>
<td style="text-align:right;">
2.795063
</td>
<td style="text-align:right;">
0.054
</td>
</tr>
<tr>
<td style="text-align:left;">
Lib.size
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0777490
</td>
<td style="text-align:right;">
0.0015276
</td>
<td style="text-align:right;">
1.361239
</td>
<td style="text-align:right;">
0.244
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.5404132
</td>
<td style="text-align:right;">
0.0106182
</td>
<td style="text-align:right;">
9.461624
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.4190994
</td>
<td style="text-align:right;">
0.0082346
</td>
<td style="text-align:right;">
7.337646
</td>
<td style="text-align:right;">
0.003
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1185437
</td>
<td style="text-align:right;">
0.0023292
</td>
<td style="text-align:right;">
2.075479
</td>
<td style="text-align:right;">
0.116
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.2259389
</td>
<td style="text-align:right;">
0.0044393
</td>
<td style="text-align:right;">
3.955768
</td>
<td style="text-align:right;">
0.021
</td>
</tr>
<tr>
<td style="text-align:left;">
Residual
</td>
<td style="text-align:right;">
291
</td>
<td style="text-align:right;">
16.6208496
</td>
<td style="text-align:right;">
0.3265718
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
<tr>
<td style="text-align:left;">
Total
</td>
<td style="text-align:right;">
299
</td>
<td style="text-align:right;">
50.8949386
</td>
<td style="text-align:right;">
1.0000000
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
</tbody>
</table>

``` r
plot_lib_dist(Ps_obj_SIP)
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-1.png)<!-- -->

``` r
Ps_obj_SIP %>%
  scale_libraries(round = "round") ->
  Ps_obj_SIP_scaled
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
plot_lib_dist(Ps_obj_SIP_scaled)
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-2.png)<!-- -->

``` r
(mod2 <- adonis2(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn") ~ Site * Oxygen * Hours + Lib.size,
  data = as(sample_data(Ps_obj_SIP_scaled), "data.frame"),
  permutations = 999
))
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:right;">
Df
</th>
<th style="text-align:right;">
SumOfSqs
</th>
<th style="text-align:right;">
R2
</th>
<th style="text-align:right;">
F
</th>
<th style="text-align:right;">
Pr(\>F)
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Site
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
27.6580326
</td>
<td style="text-align:right;">
0.5391944
</td>
<td style="text-align:right;">
482.574243
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
5.3811011
</td>
<td style="text-align:right;">
0.1049048
</td>
<td style="text-align:right;">
93.888847
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1610764
</td>
<td style="text-align:right;">
0.0031402
</td>
<td style="text-align:right;">
2.810443
</td>
<td style="text-align:right;">
0.054
</td>
</tr>
<tr>
<td style="text-align:left;">
Lib.size
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0690842
</td>
<td style="text-align:right;">
0.0013468
</td>
<td style="text-align:right;">
1.205374
</td>
<td style="text-align:right;">
0.304
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.5817234
</td>
<td style="text-align:right;">
0.0113407
</td>
<td style="text-align:right;">
10.149844
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.4264653
</td>
<td style="text-align:right;">
0.0083140
</td>
<td style="text-align:right;">
7.440919
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1115178
</td>
<td style="text-align:right;">
0.0021740
</td>
<td style="text-align:right;">
1.945750
</td>
<td style="text-align:right;">
0.139
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.2278696
</td>
<td style="text-align:right;">
0.0044423
</td>
<td style="text-align:right;">
3.975843
</td>
<td style="text-align:right;">
0.021
</td>
</tr>
<tr>
<td style="text-align:left;">
Residual
</td>
<td style="text-align:right;">
291
</td>
<td style="text-align:right;">
16.6782367
</td>
<td style="text-align:right;">
0.3251428
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
<tr>
<td style="text-align:left;">
Total
</td>
<td style="text-align:right;">
299
</td>
<td style="text-align:right;">
51.2951071
</td>
<td style="text-align:right;">
1.0000000
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
</tbody>
</table>

``` r
(mod3 <- adonis2(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn") ~ Site * Oxygen * Hours * Density.zone,
  data = as(sample_data(Ps_obj_SIP_scaled), "data.frame"),
  permutations = 999
))
```

<table>
<thead>
<tr>
<th style="text-align:left;">
</th>
<th style="text-align:right;">
Df
</th>
<th style="text-align:right;">
SumOfSqs
</th>
<th style="text-align:right;">
R2
</th>
<th style="text-align:right;">
F
</th>
<th style="text-align:right;">
Pr(\>F)
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Site
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
27.6580326
</td>
<td style="text-align:right;">
0.5391944
</td>
<td style="text-align:right;">
710.3918641
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
5.3811011
</td>
<td style="text-align:right;">
0.1049048
</td>
<td style="text-align:right;">
138.2126674
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1610764
</td>
<td style="text-align:right;">
0.0031402
</td>
<td style="text-align:right;">
4.1372204
</td>
<td style="text-align:right;">
0.026
</td>
</tr>
<tr>
<td style="text-align:left;">
Density.zone
</td>
<td style="text-align:right;">
2
</td>
<td style="text-align:right;">
2.4171422
</td>
<td style="text-align:right;">
0.0471223
</td>
<td style="text-align:right;">
31.0419431
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.5264587
</td>
<td style="text-align:right;">
0.0102633
</td>
<td style="text-align:right;">
13.5220022
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.4072149
</td>
<td style="text-align:right;">
0.0079387
</td>
<td style="text-align:right;">
10.4592446
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0884712
</td>
<td style="text-align:right;">
0.0017247
</td>
<td style="text-align:right;">
2.2723677
</td>
<td style="text-align:right;">
0.105
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1749105
</td>
<td style="text-align:right;">
0.0034099
</td>
<td style="text-align:right;">
4.4925470
</td>
<td style="text-align:right;">
0.010
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
2.7697312
</td>
<td style="text-align:right;">
0.0539960
</td>
<td style="text-align:right;">
71.1400749
</td>
<td style="text-align:right;">
0.001
</td>
</tr>
<tr>
<td style="text-align:left;">
Hours:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0296273
</td>
<td style="text-align:right;">
0.0005776
</td>
<td style="text-align:right;">
0.7609713
</td>
<td style="text-align:right;">
0.493
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen:Hours
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1426365
</td>
<td style="text-align:right;">
0.0027807
</td>
<td style="text-align:right;">
3.6635948
</td>
<td style="text-align:right;">
0.027
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0681521
</td>
<td style="text-align:right;">
0.0013286
</td>
<td style="text-align:right;">
1.7504743
</td>
<td style="text-align:right;">
0.185
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Hours:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.1071922
</td>
<td style="text-align:right;">
0.0020897
</td>
<td style="text-align:right;">
2.7532144
</td>
<td style="text-align:right;">
0.057
</td>
</tr>
<tr>
<td style="text-align:left;">
Oxygen:Hours:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.2601483
</td>
<td style="text-align:right;">
0.0050716
</td>
<td style="text-align:right;">
6.6818640
</td>
<td style="text-align:right;">
0.002
</td>
</tr>
<tr>
<td style="text-align:left;">
Site:Oxygen:Hours:Density.zone
</td>
<td style="text-align:right;">
1
</td>
<td style="text-align:right;">
0.0850351
</td>
<td style="text-align:right;">
0.0016578
</td>
<td style="text-align:right;">
2.1841127
</td>
<td style="text-align:right;">
0.134
</td>
</tr>
<tr>
<td style="text-align:left;">
Residual
</td>
<td style="text-align:right;">
283
</td>
<td style="text-align:right;">
11.0181769
</td>
<td style="text-align:right;">
0.2147998
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
<tr>
<td style="text-align:left;">
Total
</td>
<td style="text-align:right;">
299
</td>
<td style="text-align:right;">
51.2951071
</td>
<td style="text-align:right;">
1.0000000
</td>
<td style="text-align:right;">
NA
</td>
<td style="text-align:right;">
NA
</td>
</tr>
</tbody>
</table>

``` r
Site_disp <- betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), get_variable(Ps_obj_SIP_scaled, "Site"))
```

    ## Warning in betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), :
    ## some squared distances are negative and changed to zero

``` r
permutest(Site_disp)
```

    ## 
    ## Permutation test for homogeneity of multivariate dispersions
    ## Permutation: free
    ## Number of permutations: 999
    ## 
    ## Response: Distances
    ##            Df  Sum Sq  Mean Sq      F N.Perm Pr(>F)  
    ## Groups      1  0.2375 0.237546 5.5678    999  0.016 *
    ## Residuals 298 12.7139 0.042664                       
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
plot(Site_disp)
```

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-3.png)<!-- -->

``` r
Oxygen_disp <- betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), get_variable(Ps_obj_SIP_scaled, "Oxygen"))
```

    ## Warning in betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), :
    ## some squared distances are negative and changed to zero

``` r
permutest(Oxygen_disp)
```

    ## 
    ## Permutation test for homogeneity of multivariate dispersions
    ## Permutation: free
    ## Number of permutations: 999
    ## 
    ## Response: Distances
    ##            Df  Sum Sq   Mean Sq      F N.Perm Pr(>F)  
    ## Groups      1 0.03023 0.0302278 3.1681    999  0.087 .
    ## Residuals 298 2.84331 0.0095413                       
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
plot(Oxygen_disp)
```

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-4.png)<!-- -->

``` r
Hours_disp <- betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), get_variable(Ps_obj_SIP_scaled, "Hours"))
```

    ## Warning in betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), :
    ## some squared distances are negative and changed to zero

``` r
permutest(Hours_disp)
```

    ## 
    ## Permutation test for homogeneity of multivariate dispersions
    ## Permutation: free
    ## Number of permutations: 999
    ## 
    ## Response: Distances
    ##            Df Sum Sq  Mean Sq      F N.Perm Pr(>F)   
    ## Groups      4 0.2595 0.064877 4.7232    999  0.002 **
    ## Residuals 295 4.0520 0.013736                        
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
plot(Hours_disp)
```

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-5.png)<!-- -->

``` r
Density_disp <- betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), get_variable(Ps_obj_SIP_scaled, "Density.zone"))
```

    ## Warning in betadisper(vegdist(otu_table(Ps_obj_SIP_scaled), method = "horn"), :
    ## some squared distances are negative and changed to zero

``` r
permutest(Density_disp)
```

    ## 
    ## Permutation test for homogeneity of multivariate dispersions
    ## Permutation: free
    ## Number of permutations: 999
    ## 
    ## Response: Distances
    ##            Df  Sum Sq  Mean Sq      F N.Perm Pr(>F)    
    ## Groups      2 0.48515 0.242577 24.876    999  0.001 ***
    ## Residuals 297 2.89621 0.009752                         
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
plot(Density_disp)
```

![](05_Diff_abund_files/figure-gfm/beta%20div%20joint-6.png)<!-- -->

``` r
Ord <- ordinate(Ps_obj_SIP_scaled, "CAP", "horn", 
                formula =  ~ Site * Oxygen * Hours * Density.zone)
explained <- as.numeric(format(round(eigenvals(Ord)/sum(eigenvals(Ord)) * 100, 1), nsmall = 1))
Ord_plt <- plot_ordination(Ps_obj_SIP, Ord, type = "samples", color = "Label..13C.", justDF = TRUE)

p_ord_joint <- ggplot(Ord_plt) +
  geom_point(aes(
               x = CAP1,
               y = CAP2,
               color = Label..13C.,
               size = Density..g.ml.1.,
               shape = Oxygen
             ), alpha = 2 / 3) +
  guides(colour = guide_legend(title = "Labelling"), 
         size = guide_legend(title = "Density (g ml<sup>-1</sup>)"),
         shape = guide_legend(title = "Oxygen")) +
  scale_colour_locuszoom() +
  # scale_colour_manual(values = Gradient.colours) +
  # scale_fill_manual(values = Gradient.colours, guide = "none") +
  labs(x = sprintf("CAP1 (%s%%)", explained[1]),
  y = sprintf("CAP2 (%s%%)", explained[2])) +
  coord_fixed(ratio = sqrt(explained[2] / explained[1])) +
   theme(legend.justification = "top",
         legend.title = element_markdown(size = 11)
         ) +
  scale_size_continuous(breaks = round(c(seq(min(Ord_plt$Density..g.ml.1.), 
                                       max(Ord_plt$Density..g.ml.1.), 
                                       length.out = 5), 
                                   1), 4),
                        range = c(0.1, 5)) +
  facet_grid(Site ~ Hours) +
  # ggtitle("Joint analysis") +
  NULL

save_figure(paste0(fig.path, "Oridnation"), 
            p_ord_joint, 
            pwidth = 10, 
            pheight = 8,
            dpi = 600)
```

    ## Loading required package: ragg

``` r
knitr::include_graphics(paste0(fig.path, "Oridnation", ".png"))
```

<img src="05_Diff_abund_figures/Oridnation.png" width="6000" />

### Differential abundance models

Now run the differential abundance models using DESeq2. We then filter
the resutls to include only ASVs with Log_2\_ fold change \>`LFC_thresh`
and significant at P\<`alpha_thresh`. Lastly, we run ‘LFC-shrinking’
based on Stephens ([2016](#ref-stephens_fdr_2016)).

``` r
# generate a deseq object
DESeq_obj_SIP_byTime_l <- mclapply(Ps_obj_SIP_byTime_l, 
                                   function(x) {phyloseq_to_deseq2_safe(x, 
                                                                        test_condition = "Density.zone", 
                                                                        ref_level = "Light")}, 
                                   mc.cores = nrow(params_2))


# run dds pipeline
DESeq_obj_SIP_byTime_l %<>% mclapply(., 
                                     function(x) {DESeq(x, 
                                                        test = "Wald", 
                                                        fitType = "local")}, 
                                     mc.cores = nrow(params_2)) # run dds pipeline

# extract results from a DESeq analysis
DESeq_res_SIP_byTime_l  <- mclapply(DESeq_obj_SIP_byTime_l, 
                             function(x) {
                               results(x, 
                                       altHypothesis = "greater",
                                       alpha = alpha_thresh, 
                                       contrast = c("Density.zone", "Heavy", "Light"))}, # redundant if phyloseq_to_deseq2_safe() was used but doesn't hurt
                             mc.cores = nrow(params_2)) 

DESeq_res_SIP_byTime_LFC_l <- mclapply(DESeq_obj_SIP_byTime_l, 
                                     function(x) {
                                       results(x,
                                               lfcThreshold = LFC_thresh,
                                               altHypothesis = "greater",
                                               alpha = alpha_thresh,
                                               contrast = c("Density.zone", "Heavy", "Light"))}, # redundant if phyloseq_to_deseq2_safe() was used but doesn't hurt
                                     mc.cores = nrow(params_2)) # Extract results from a DESeq analysis


DESeq_res_SIP_byTime_LFC_shrink_l <- map(seq(length(DESeq_obj_SIP_byTime_l)), 
                                             ~lfcShrink(DESeq_obj_SIP_byTime_l[[.x]],
                                                         res = DESeq_res_SIP_byTime_LFC_l[[.x]],
                                                         coef = "Density.zone_Heavy_vs_Light",
                                                         type = "ashr"))
```

    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041
    ## using 'ashr' for LFC shrinkage. If used in published research, please cite:
    ##     Stephens, M. (2016) False discovery rates: a new deal. Biostatistics, 18:2.
    ##     https://doi.org/10.1093/biostatistics/kxw041

``` r
names(DESeq_res_SIP_byTime_LFC_shrink_l) <- names(DESeq_res_SIP_byTime_LFC_l)

# Compare
plotMA(DESeq_res_SIP_byTime_l[[2]], ylim = c(-2,2))
```

![](05_Diff_abund_files/figure-gfm/DESeq2%20models%20by%20time-1.png)<!-- -->

``` r
plotMA(DESeq_res_SIP_byTime_LFC_l[[2]], ylim = c(-2,2))
```

![](05_Diff_abund_files/figure-gfm/DESeq2%20models%20by%20time-2.png)<!-- -->

``` r
plotMA(DESeq_res_SIP_byTime_LFC_shrink_l[[2]], ylim = c(-2,2))
```

![](05_Diff_abund_files/figure-gfm/DESeq2%20models%20by%20time-3.png)<!-- -->

``` r
# summarise results (lfcShrink doesn't change the values)
# map2(DESeq_res_SIP_byTime_l, print(names(DESeq_res_SIP_byTime_l)), ~summary(.x)) # summarise results
# for (i in seq(1, length(DESeq_res_SIP_byTime_l))) { # didn't manage with map
#   print(names(DESeq_res_SIP_byTime_l[i]))
#   summary(DESeq_res_SIP_byTime_l[[i]])
# }

for (i in seq(1, length(DESeq_res_SIP_byTime_LFC_l))) { # didn't manage with map
  print(names(DESeq_res_SIP_byTime_LFC_l[i]))
  summary(DESeq_res_SIP_byTime_LFC_l[[i]])
}
```

    ## [1] "Certovo & 12 h & Anoxic & Labelled"
    ## 
    ## out of 3026 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 0, 0%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 3, 0.099%
    ## low counts [2]     : 7, 0.23%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 216 h & Anoxic & Labelled"
    ## 
    ## out of 2878 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 37, 1.3%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1484, 52%
    ## (mean count < 2)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 24 h & Anoxic & Labelled"
    ## 
    ## out of 2755 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 1, 0.036%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 4, 0.15%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 48 h & Anoxic & Labelled"
    ## 
    ## out of 2787 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 15, 0.54%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1222, 44%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 216 h & Anoxic & Unlabelled"
    ## 
    ## out of 2825 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 0, 0%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 5, 0.18%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 12 h & Oxic & Labelled"
    ## 
    ## out of 3053 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 7, 0.23%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 8, 0.26%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 24 h & Oxic & Labelled"
    ## 
    ## out of 2954 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 107, 3.6%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 905, 31%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 48 h & Oxic & Labelled"
    ## 
    ## out of 2882 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 122, 4.2%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 331, 11%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 72 h & Oxic & Labelled"
    ## 
    ## out of 2898 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 142, 4.9%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 1, 0.035%
    ## low counts [2]     : 1329, 46%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Certovo & 72 h & Oxic & Unlabelled"
    ## 
    ## out of 2979 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 33, 1.1%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 4, 0.13%
    ## low counts [2]     : 1995, 67%
    ## (mean count < 4)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 12 h & Anoxic & Labelled"
    ## 
    ## out of 3031 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 27, 0.89%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1678, 55%
    ## (mean count < 3)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 216 h & Anoxic & Labelled"
    ## 
    ## out of 3210 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 0, 0%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 116, 3.6%
    ## low counts [2]     : 3, 0.093%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 24 h & Anoxic & Labelled"
    ## 
    ## out of 2848 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 22, 0.77%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 2340, 82%
    ## (mean count < 15)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 48 h & Anoxic & Labelled"
    ## 
    ## out of 2758 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 12, 0.44%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1527, 55%
    ## (mean count < 2)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 216 h & Anoxic & Unlabelled"
    ## 
    ## out of 2862 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 0, 0%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 2, 0.07%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 12 h & Oxic & Labelled"
    ## 
    ## out of 2923 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 57, 2%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 1, 0.034%
    ## low counts [2]     : 1790, 61%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 24 h & Oxic & Labelled"
    ## 
    ## out of 2548 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 101, 4%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1061, 42%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 48 h & Oxic & Labelled"
    ## 
    ## out of 2531 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 13, 0.51%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1820, 72%
    ## (mean count < 3)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 72 h & Oxic & Labelled"
    ## 
    ## out of 2556 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 115, 4.5%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 0, 0%
    ## low counts [2]     : 1113, 44%
    ## (mean count < 1)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results
    ## 
    ## [1] "Plesne & 72 h & Oxic & Unlabelled"
    ## 
    ## out of 2798 with nonzero total read count
    ## adjusted p-value < 0.05
    ## LFC > 0.26 (up)    : 3, 0.11%
    ## LFC < -0.26 (down) : 0, 0%
    ## outliers [1]       : 7, 0.25%
    ## low counts [2]     : 0, 0%
    ## (mean count < 0)
    ## [1] see 'cooksCutoff' argument of ?results
    ## [2] see 'independentFiltering' argument of ?results

``` r
# Store labelled ASVs and save them to a file
# DESeq_res_SIP_byTime_l %>% 
#   map(., ~subset(.x, padj < alpha_thresh & log2FoldChange > LFC_thresh)) %>% 
#   map(., ~as.data.frame(.x)) %>% 
#   map(., ~rownames_to_column(.x, "ASV")) %>% 
#   bind_rows(., .id = "Comparison") %>% 
#   arrange(Comparison, desc(baseMean)) %T>% 
#   write_csv(., file = "DESeq2_byTime_a-0.05.txt") ->
#   DESeq_res_SIP_byTime_df

# Store labelled ASVs and save them to a file
DESeq_res_SIP_byTime_LFC_shrink_l %>% 
  map(., ~subset(.x, padj < alpha_thresh & log2FoldChange > LFC_thresh)) %>% 
  map(., ~as.data.frame(.x)) %>% 
  map(., ~rownames_to_column(.x, "ASV")) %>% 
  bind_rows(., .id = "Comparison") %>% 
  arrange(Comparison, desc(baseMean)) %>% 
  separate(., "Comparison" ,c("Site","Hours", "Oxygen", "Label"), sep = " & ") ->
  DESeq_res_SIP_byTime_LFC_sig_df

# grab the taxonomy of the ASVs
prune_taxa(DESeq_res_SIP_byTime_LFC_sig_df$ASV, Ps_obj_SIP) %>% 
  tax_table() %>% 
  as("data.frame") %>% 
  rownames_to_column("ASV") %>% 
  merge(., DESeq_res_SIP_byTime_LFC_sig_df, by = "ASV") %>%  # watch out: this merge recycles values!
  arrange(Site, Oxygen, Hours, baseMean) %>% 
  write_csv(., file = "DESeq2_byTime_a-0.05_LFC0-322.txt")
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

#### Inspect results

``` r
DESeq_res_SIP_byTime_LFC_sig_df %>% 
  get_variable() %>% 
  select_if(is.numeric) %>% 
  vis_value()
```

    ## Warning: `gather_()` was deprecated in tidyr 1.2.0.
    ## Please use `gather()` instead.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was generated.

![](05_Diff_abund_files/figure-gfm/vis%20DES%20res-1.png)<!-- -->

``` r
DESeq_res_SIP_byTime_LFC_sig_df %>% 
  get_variable() %>% 
  select_if(is.numeric) %>% 
  vis_cor()
```

![](05_Diff_abund_files/figure-gfm/vis%20DES%20res-2.png)<!-- -->

#### Plot differential abundance models

``` r
# DESeq_results <- DESeq_res_SIP_byTime_LFC_shrink_l[2]
# plot_DESeq(DESeq_results, Ps_obj_SIP, plot_title = names(DESeq_results))

DESeq_plots <- map(seq(length(DESeq_res_SIP_byTime_LFC_shrink_l)), 
                        ~plot_DESeq(DESeq_res_SIP_byTime_LFC_shrink_l[.x],  
                                                Ps_obj_SIP, plot_title = names(DESeq_res_SIP_byTime_LFC_shrink_l[.x])))
```

    ## Loading required package: ggrepel

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Note: Using an external vector in selections is ambiguous.
    ## ℹ Use `all_of(rank)` instead of `rank` to silence this message.
    ## ℹ See <https://tidyselect.r-lib.org/reference/faq-external-vector.html>.
    ## This message is displayed once per session.
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'
    ## 
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## 
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in speedyseq::psmelt(ps_obj_glom_rel): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'
    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

``` r
Certovo_DESeq <- ((DESeq_plots[[6]] + 
                     theme(legend.position = "none") +
                     theme(axis.text.x = element_blank())) +
                    (DESeq_plots[[1]] + 
                       theme(legend.position = "none", 
                             axis.text.x = element_blank(), 
                             axis.title.y = element_blank())) +
                    (DESeq_plots[[7]] + 
                       theme(legend.position = "none",
                             axis.text.x = element_blank())) +
                    (DESeq_plots[[3]] + 
                       theme(legend.position = "none", 
                             axis.text.x = element_blank(), 
                             axis.title.y = element_blank())) +
                    (DESeq_plots[[8]] + 
                       theme(legend.position = "none") +
                       theme(legend.position = "none",
                             axis.text.x = element_blank())) +
                    (DESeq_plots[[4]] + 
                       theme(legend.position = "none") +
                       theme(legend.position = "none", 
                             axis.text.x = element_blank(), 
                             axis.title.y = element_blank()) +
                       ylim(NA, 5)) +
                    (DESeq_plots[[9]] + 
                       theme(legend.position = "none",
                             axis.text.x = element_blank())) +
                    (DESeq_plots[[2]] + 
                       theme(legend.position = "none", 
                             axis.text.x = element_blank(), 
                             axis.title.y = element_blank())) +
                    (DESeq_plots[[10]] + 
                       theme(legend.position = "none") +
                       ylim(NA, 5)) +
                    (DESeq_plots[[5]] + 
                       theme(legend.position = "none", 
                             axis.title.y = element_blank())) + 
                    plot_layout(ncol = 2, guides = "collect") & 
                    theme(legend.position = 'bottom'))

save_figure(paste0(fig.path, "Certovo_DESeq2"), 
            Certovo_DESeq, 
            pwidth = 8, 
            pheight = 10,
            dpi = 600)
```

    ## Warning: Removed 3046 rows containing missing values (geom_segment).

    ## Warning: Removed 3026 rows containing missing values (geom_segment).

    ## Warning: Removed 2862 rows containing missing values (geom_segment).

    ## Warning: Removed 2755 rows containing missing values (geom_segment).

    ## Warning: Removed 2775 rows containing missing values (geom_segment).

    ## Warning: Removed 1 rows containing missing values (geom_point).

    ## Warning: Removed 2774 rows containing missing values (geom_segment).

    ## Warning: Removed 2785 rows containing missing values (geom_segment).

    ## Warning: Removed 2856 rows containing missing values (geom_segment).

    ## Warning: Removed 5 rows containing missing values (geom_point).

    ## Warning: Removed 2969 rows containing missing values (geom_segment).

    ## Warning: Removed 2825 rows containing missing values (geom_segment).

    ## Warning: Removed 3046 rows containing missing values (geom_segment).

    ## Warning: Removed 3026 rows containing missing values (geom_segment).

    ## Warning: Removed 2862 rows containing missing values (geom_segment).

    ## Warning: Removed 2755 rows containing missing values (geom_segment).

    ## Warning: Removed 2775 rows containing missing values (geom_segment).

    ## Warning: Removed 1 rows containing missing values (geom_point).

    ## Warning: Removed 2774 rows containing missing values (geom_segment).

    ## Warning: Removed 2785 rows containing missing values (geom_segment).

    ## Warning: Removed 2856 rows containing missing values (geom_segment).

    ## Warning: Removed 5 rows containing missing values (geom_point).

    ## Warning: Removed 2969 rows containing missing values (geom_segment).

    ## Warning: Removed 2825 rows containing missing values (geom_segment).

``` r
knitr::include_graphics(paste0(fig.path, "Certovo_DESeq2", ".png"))
```

<img src="05_Diff_abund_figures/Certovo_DESeq2.png" width="4800" />

``` r
Plesne_DESeq <- ((DESeq_plots[[16]] + 
                    theme(legend.position = "none") +
                    theme(axis.text.x = element_blank())) +
                   (DESeq_plots[[11]] + 
                      theme(legend.position = "none", 
                            axis.text.x = element_blank(), 
                            axis.title.y = element_blank())) +
                   (DESeq_plots[[17]] + 
                      theme(legend.position = "none",
                            axis.text.x = element_blank())) +
                   (DESeq_plots[[13]] + 
                      theme(legend.position = "none", 
                            axis.text.x = element_blank(), 
                            axis.title.y = element_blank())) +
                   (DESeq_plots[[18]] + 
                      theme(legend.position = "none") +
                      theme(legend.position = "none",
                            axis.text.x = element_blank())) +
                   (DESeq_plots[[14]] + 
                      theme(legend.position = "none") +
                      theme(legend.position = "none", 
                            axis.text.x = element_blank(), 
                            axis.title.y = element_blank())) +
                   (DESeq_plots[[19]] + 
                      theme(legend.position = "none",
                            axis.text.x = element_blank())) +
                   (DESeq_plots[[12]] + 
                      theme(legend.position = "none", 
                            axis.text.x = element_blank(), 
                            axis.title.y = element_blank()) +
                      ylim(-3, NA)) +
                   (DESeq_plots[[20]] + 
                      theme(legend.position = "none")) +
                   (DESeq_plots[[15]] + 
                      theme(legend.position = "none", 
                            axis.title.y = element_blank())) + 
                   plot_layout(ncol = 2, guides = "collect") & 
                   theme(legend.position = 'bottom'))

save_figure(paste0(fig.path, "Plesne_DESeq2"), 
            Plesne_DESeq, 
            pwidth = 8, 
            pheight = 10,
            dpi = 600)
```

    ## Warning: Removed 2867 rows containing missing values (geom_segment).

    ## Warning: Removed 3012 rows containing missing values (geom_segment).

    ## Warning: Removed 2463 rows containing missing values (geom_segment).

    ## Warning: Removed 2841 rows containing missing values (geom_segment).

    ## Warning: Removed 2518 rows containing missing values (geom_segment).

    ## Warning: Removed 2746 rows containing missing values (geom_segment).

    ## Warning: Removed 2460 rows containing missing values (geom_segment).

    ## Warning: Removed 11 rows containing missing values (geom_point).

    ## Warning: Removed 3210 rows containing missing values (geom_segment).

    ## Warning: Removed 2795 rows containing missing values (geom_segment).

    ## Warning: Removed 2862 rows containing missing values (geom_segment).

    ## Warning: Removed 2867 rows containing missing values (geom_segment).

    ## Warning: Removed 3012 rows containing missing values (geom_segment).

    ## Warning: Removed 2463 rows containing missing values (geom_segment).

    ## Warning: Removed 2841 rows containing missing values (geom_segment).

    ## Warning: Removed 2518 rows containing missing values (geom_segment).

    ## Warning: Removed 2746 rows containing missing values (geom_segment).

    ## Warning: Removed 2460 rows containing missing values (geom_segment).

    ## Warning: Removed 11 rows containing missing values (geom_point).

    ## Warning: Removed 3210 rows containing missing values (geom_segment).

    ## Warning: Removed 2795 rows containing missing values (geom_segment).

    ## Warning: Removed 2862 rows containing missing values (geom_segment).

``` r
knitr::include_graphics(paste0(fig.path, "Plesne_DESeq2", ".png"))
```

<img src="05_Diff_abund_figures/Plesne_DESeq2.png" width="4800" />

### Plot labelled ASVs

``` r
plot_combintions <- crossing(Site = c("Certovo", "Plesne"), 
         Oxygen = c("Oxic", "Anoxic"))

Labelled_ASVs <- map(seq(length(Ps_obj_SIP_noTime_l)), ~plot_otus_by_density(Ps_obj_SIP_noTime_l[[.x]], 
                     ASV2plot = filter(DESeq_res_SIP_byTime_LFC_sig_df, Site == plot_combintions$Site[.x], Oxygen == plot_combintions$Oxygen[.x])))
```

    ## Loading required package: ggpomological

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(.): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
map(seq(length(Ps_obj_SIP_noTime_l)), 
    ~save_figure(paste0(fig.path, "Labelled_ASVs_", paste(plot_combintions[.x, ], collapse = "_")), 
                 Labelled_ASVs[[.x]], 
                 pwidth = 16, 
                 pheight = 12,
                 dpi = 600))
```

    ## [[1]]
    ## [1] "05_Diff_abund_figures/Labelled_ASVs_Certovo_Anoxic.svgz"
    ## 
    ## [[2]]
    ## [1] "05_Diff_abund_figures/Labelled_ASVs_Certovo_Oxic.svgz"
    ## 
    ## [[3]]
    ## [1] "05_Diff_abund_figures/Labelled_ASVs_Plesne_Anoxic.svgz"
    ## 
    ## [[4]]
    ## [1] "05_Diff_abund_figures/Labelled_ASVs_Plesne_Oxic.svgz"

``` r
plots2display <- list.files(path = paste0(fig.path), 
                    pattern = "^Labelled_ASVs_(.*).png$",
                    full.names = TRUE)

knitr::include_graphics(plots2display)
```

<img src="05_Diff_abund_figures//Labelled_ASVs_Certovo_Anoxic.png" width="9600" /><img src="05_Diff_abund_figures//Labelled_ASVs_Certovo_Oxic.png" width="9600" /><img src="05_Diff_abund_figures//Labelled_ASVs_Plesne_Anoxic.png" width="9600" /><img src="05_Diff_abund_figures//Labelled_ASVs_Plesne_Oxic.png" width="9600" />

#### Plot phylogenetic trees with heatmaps

``` r
c("Certovo Oxic 12 h", "Certovo Oxic 24 h", "Certovo Oxic 48 h", "Certovo Oxic 72 h", "Certovo Anoxic 12 h", "Certovo Anoxic 24 h", "Certovo Anoxic 48 h", "Certovo Anoxic 216 h", "Plesne Oxic 12 h", "Plesne Oxic 24 h", "Plesne Oxic 48 h", "Plesne Oxic 72 h", "Plesne Anoxic 12 h",  "Plesne Anoxic 24 h",  "Plesne Anoxic 48 h",  "Plesne Anoxic 216 h" )  ->
  col_order

DESeq_res_SIP_byTime_LFC_l %>% 
  map(., ~as.data.frame(.x)) %>% 
  map(., ~rownames_to_column(.x, "ASV")) %>% 
  bind_rows(., .id = "Comparison") %>% 
  filter(str_detect(Comparison, "Labelled")) %>% # remove unlabelled samples [c(-5, -10, -15, -20)]
  mutate(Labelled = ifelse(padj < alpha_thresh & log2FoldChange > LFC_thresh, "Labelled", "Unlabelled")) %>% 
  # arrange(Comparison, desc(baseMean)) %>% 
  separate(., "Comparison" ,c("Site","Hours", "Oxygen", "Label"), sep = " & ") %>% 
  mutate(Site_Oxygen_Hours = paste(Site, Oxygen, Hours)) %>% 
  mutate(across(Site_Oxygen_Hours, ~factor(., levels = col_order))) %>% 
  mutate(Site_Oxygen = factor(paste0(Site, "-", Oxygen),
                              levels = c("Plesne-Oxic", "Plesne-Anoxic", "Certovo-Oxic", "Certovo-Anoxic"),
                              labels = c("Pl-Ox", "Pl-Anox", "Ct-Ox", "Ct-Anox"))) %>%
  mutate(across(c("Hours"), ~factor(., 
                                    levels = c("12 h", "24 h", "48 h", "72 h", "216 h"),
                                    labels = c("12", "24", "48", "72", "216")))) ->
  # mutate(Site_oxygen = paste(Site, Oxygen)) ->
  DESeq_res_SIP_byTime_all_df

# Summarise number of labelled and unlabelled ASVs
DESeq_res_SIP_byTime_all_df %>% 
  group_by(Labelled) %>% 
  summarise(n = n()) 
```

<div class="kable-table">

<table>
<thead>
<tr>
<th style="text-align:left;">
Labelled
</th>
<th style="text-align:right;">
n
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Labelled
</td>
<td style="text-align:right;">
778
</td>
</tr>
<tr>
<td style="text-align:left;">
Unlabelled
</td>
<td style="text-align:right;">
37162
</td>
</tr>
<tr>
<td style="text-align:left;">
NA
</td>
<td style="text-align:right;">
21148
</td>
</tr>
</tbody>
</table>

</div>

``` r
# detect taxa with NA from DESeq analysis
DESeq_res_SIP_byTime_all_df %<>% 
  filter(!is.na(Labelled)) #%>% 
  # pull(Labelled) -> 
  # bad_seqs

# remove NA taxa from PS obj
Ps_obj_SIP %>% 
  prune_taxa(setdiff(taxa_names(Ps_obj_SIP), "Seq_2375"), .) %>% # outlier
  prune_taxa(DESeq_res_SIP_byTime_all_df$ASV, .) ->
  Ps_obj_SIP4tree_plot
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
# Remove long name
tax_table(Ps_obj_SIP4tree_plot)[, "Order"] %<>%  str_replace_all(., "Gammaproteobacteria Incertae Sedis", "Incertae Sedis")
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
taxa2plot <- tibble(rank = c(rep("Class", 3), rep("Phylum", 4)), 
                    subrank = c(rep("Order", 3), rep("Class", 4)), 
                    Taxa2plot = c("Actinobacteria", 
                                  "Alphaproteobacteria", 
                                  "Gammaproteobacteria", 
                                  "Acidobacteriota",
                                  "Verrucomicrobiota",
                                  "Bacteroidota",
                                  "Firmicutes"),
                    l_rows = c(4, 5, 6, 3, 3, 3, 3),
                    pwidth = c(5, 6, 8, 3, 3, 3, 3), 
                    pheight = c(rep(10, 7)),)

tree_p_l <- map(seq(nrow(taxa2plot)), 
                ~wrap_ggtree_heatmap(Ps_obj_SIP4tree_plot,
                                     DESeq_res_SIP_byTime_all_df,
                                     rank = taxa2plot$rank[.x],
                                     subrank = taxa2plot$subrank[.x],
                                     Taxa2plot = taxa2plot$Taxa2plot[.x],
                                     l_rows = 8,
                                     pwidth = 4,
                                     pheight = 10))
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Warning in psmelt(model): The sample variables: 
    ## Sample
    ##  have been renamed to: 
    ## sample_Sample
    ## to avoid conflicts with special phyloseq plot attribute names.

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
trees2display <- list.files(path = paste0(fig.path), 
                    pattern = "^Tree_HM_(.*).png$",
                    full.names = TRUE)

knitr::include_graphics(trees2display)
```

<img src="05_Diff_abund_figures//Tree_HM_Acidobacteriota.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Actinobacteria.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Alphaproteobacteria.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Bacteroidota.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Firmicutes.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Gammaproteobacteria.png" width="2400" /><img src="05_Diff_abund_figures//Tree_HM_Verrucomicrobiota.png" width="2400" />

``` r
all_trees <- ((tree_p_l[[1]] | tree_p_l[[2]] + guides(fill = FALSE) | tree_p_l[[3]] + guides(fill = FALSE) | tree_p_l[[4]] + guides(fill = FALSE)) / (tree_p_l[[5]] + guides(fill = FALSE) | tree_p_l[[6]] + guides(fill = FALSE) | tree_p_l[[7]] + guides(fill = FALSE) | plot_spacer())) + plot_layout(heights = c(2, 1))
```

    ## Warning: `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> =
    ## "none")` instead.

    ## Warning: `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> = "none")` instead.
    ## `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> = "none")` instead.
    ## `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> = "none")` instead.
    ## `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> = "none")` instead.
    ## `guides(<scale> = FALSE)` is deprecated. Please use `guides(<scale> = "none")` instead.

``` r
save_figure(paste0(fig.path, "all_trees"), 
            all_trees, 
            pwidth = 16, 
            pheight = 18,
            dpi = 900)
```

#### How abundant were the labelled ASVs?

``` r
Ps_obj_SIP %>% 
  transform_sample_counts(function(x) {x/sum(x)}) %>% 
  prune_taxa(filter(DESeq_res_SIP_byTime_LFC_sig_df, Site == "Certovo" & Oxygen == "Oxic")$ASV, .) %>% 
  subset_samples(Site == "Certovo" & Oxygen == "Oxic") %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  rowSums() %>% 
  data.frame(Total = .) %>% 
  rownames_to_column() %>% 
  separate(col = rowname,
           into = c("Site", "Oxygen", "Glucose", "Label..13C.", "Hours", "Fraction.no."),
           sep = "_") %>% 
  filter(`Label..13C.` == "Labelled") %>% 
  group_by(Site, Oxygen) %>% 
  summarise_if(is.numeric, mean) ->
  Cert_Ox
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
Ps_obj_SIP %>% 
  transform_sample_counts(function(x) {x/sum(x)}) %>% 
  prune_taxa(filter(DESeq_res_SIP_byTime_LFC_sig_df, Site == "Certovo" & Oxygen == "Anoxic")$ASV, .) %>% 
  subset_samples(Site == "Certovo" & Oxygen == "Anoxic") %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  rowSums() %>% 
  data.frame(Total = .) %>% 
  rownames_to_column() %>% 
  separate(col = rowname,
           into = c("Site", "Oxygen", "Glucose", "Label..13C.", "Hours", "Fraction.no."),
           sep = "_") %>% 
  filter(`Label..13C.` == "Labelled") %>% 
  group_by(Site, Oxygen) %>% 
  summarise_if(is.numeric, mean) ->
  Cert_Anox
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
Ps_obj_SIP %>% 
  transform_sample_counts(function(x) {x/sum(x)}) %>% 
  prune_taxa(filter(DESeq_res_SIP_byTime_LFC_sig_df, Site == "Plesne" & Oxygen == "Oxic")$ASV, .) %>% 
  subset_samples(Site == "Plesne" & Oxygen == "Oxic") %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  rowSums() %>% 
  data.frame(Total = .) %>% 
  rownames_to_column() %>% 
  separate(col = rowname,
           into = c("Site", "Oxygen", "Glucose", "Label..13C.", "Hours", "Fraction.no."),
           sep = "_") %>% 
  filter(`Label..13C.` == "Labelled") %>% 
  group_by(Site, Oxygen) %>% 
  summarise_if(is.numeric, mean) ->
  Ples_Ox
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
Ps_obj_SIP %>% 
  transform_sample_counts(function(x) {x/sum(x)}) %>% 
  prune_taxa(filter(DESeq_res_SIP_byTime_LFC_sig_df, Site == "Plesne" & Oxygen == "Anoxic")$ASV, .) %>% 
  subset_samples(Site == "Plesne" & Oxygen == "Anoxic") %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  rowSums() %>% 
  data.frame(Total = .) %>% 
  rownames_to_column() %>% 
  separate(col = rowname,
           into = c("Site", "Oxygen", "Glucose", "Label..13C.", "Hours", "Fraction.no."),
           sep = "_") %>% 
  filter(`Label..13C.` == "Labelled") %>% 
  group_by(Site, Oxygen) %>% 
  summarise_if(is.numeric, mean) ->
  Ples_Anox
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
  bind_rows(Cert_Ox, Cert_Anox, Ples_Ox, Ples_Anox) %>% 
  kable(., digits = c(3)) %>%
  kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"), 
                full_width = F)
```

<table class="table table-striped table-hover table-condensed table-responsive" style="width: auto !important; margin-left: auto; margin-right: auto;">
<thead>
<tr>
<th style="text-align:left;">
Site
</th>
<th style="text-align:left;">
Oxygen
</th>
<th style="text-align:right;">
Total
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Certovo
</td>
<td style="text-align:left;">
Oxic
</td>
<td style="text-align:right;">
0.282
</td>
</tr>
<tr>
<td style="text-align:left;">
Certovo
</td>
<td style="text-align:left;">
Anoxic
</td>
<td style="text-align:right;">
0.036
</td>
</tr>
<tr>
<td style="text-align:left;">
Plesne
</td>
<td style="text-align:left;">
Oxic
</td>
<td style="text-align:right;">
0.317
</td>
</tr>
<tr>
<td style="text-align:left;">
Plesne
</td>
<td style="text-align:left;">
Anoxic
</td>
<td style="text-align:right;">
0.059
</td>
</tr>
</tbody>
</table>

``` r
# Labelled ASVs in all samples
Ps_obj_SIP %>% 
  transform_sample_counts(function(x) {x/sum(x)}) %>% 
  prune_taxa(DESeq_res_SIP_byTime_LFC_sig_df$ASV, .) %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  rowSums() %>% 
  data.frame(Total = .) %>% 
  rownames_to_column() %>% 
  separate(col = rowname,
           into = c("Site", "Oxygen", "Glucose", "Label..13C.", "Hours", "Fraction.no."),
           sep = "_") %>% 
  filter(`Label..13C.` == "Labelled") %>% 
  group_by(Site, Oxygen) %>% 
  summarise_if(is.numeric, mean) %>%   
  kable(., digits = c(3)) %>%
  kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"), 
                full_width = F)
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'
    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

<table class="table table-striped table-hover table-condensed table-responsive" style="width: auto !important; margin-left: auto; margin-right: auto;">
<thead>
<tr>
<th style="text-align:left;">
Site
</th>
<th style="text-align:left;">
Oxygen
</th>
<th style="text-align:right;">
Total
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:left;">
Certovo
</td>
<td style="text-align:left;">
Anoxic
</td>
<td style="text-align:right;">
0.212
</td>
</tr>
<tr>
<td style="text-align:left;">
Certovo
</td>
<td style="text-align:left;">
Oxic
</td>
<td style="text-align:right;">
0.329
</td>
</tr>
<tr>
<td style="text-align:left;">
Plesne
</td>
<td style="text-align:left;">
Anoxic
</td>
<td style="text-align:right;">
0.246
</td>
</tr>
<tr>
<td style="text-align:left;">
Plesne
</td>
<td style="text-align:left;">
Oxic
</td>
<td style="text-align:right;">
0.374
</td>
</tr>
</tbody>
</table>

#### Calculate NTI

``` r
# grab names of labelled ASVs
DESeq_res_SIP_byTime_all_df %>% 
  as.data.frame %>% 
  # rownames_to_column("ASV") %>% 
  filter(padj < alpha_thresh) %>% 
  filter(log2FoldChange > LFC_thresh) %>% 
  pull(ASV) %>% 
  unique() ->
  Labelled_ASVs_char

# keep only labelled gradients and heavy fractions  
Ps_obj_SIP4tree_plot %>% 
  subset_samples(Label..13C. == "Labelled") %>% 
  subset_samples(Density.zone == "Heavy") %>% 
  prune_taxa(Labelled_ASVs_char, .) ->
  Ps_obj_SIP4tree_plot_labelled 
```

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

    ## Found more than one class "phylo" in cache; using the first, from namespace 'phyloseq'

    ## Also defined by 'tidytree'

``` r
Ps_obj_SIP4tree_plot_labelled %>% 
  otu_table() %>% 
  as(., "matrix") %>% 
  {if(taxa_are_rows(Ps_obj_SIP4tree_plot_labelled)) t(.) else .}  %>% 
  {ifelse(. > 0, 1, 0)} ->
  presence_matrix

# Ps_tree <- phy_tree(Ps_obj_top)
all.equal(phy_tree(Ps_obj_SIP4tree_plot_labelled)$tip.label, colnames(otu_table(Ps_obj_SIP4tree_plot_labelled))) # just a test (should be TRUE)
```

    ## [1] TRUE

``` r
weights <- colSums(otu_table(Ps_obj_SIP4tree_plot_labelled)) / sum(colSums(otu_table(Ps_obj_SIP4tree_plot_labelled)))

# # Calculate MNTD using PhyloMeasures.
# MNTD_PM <- tibble(Identifier = rownames(otu_table(Ps_obj_SIP4tree_plot_labelled)), MNTD = mntd.query(tree = phy_tree(Ps_obj_SIP4tree_plot_labelled), matrix = presence_matrix, standardize = F, null.model = "frequency.by.richness", abundance.weights = weights)) # faster but results differ from picante 

# Using picante
MNTD_Pic <- tibble(Identifier = rownames(otu_table(Ps_obj_SIP4tree_plot_labelled)), MNTD = mntd(samp = as(otu_table(Ps_obj_SIP4tree_plot_labelled), "matrix"), dis = cophenetic(phy_tree(Ps_obj_SIP4tree_plot_labelled)), abundance.weighted = TRUE))

# # are the values equal?
# (MNTD_PM$MNTD == MNTD_Pic$MNTD)
# # Unfortunately not!

# I'll stick to picante's version
# add MNTD data to metadata 
Ps_obj_SIP4tree_plot_labelled %>%
  get_variable() %>% 
  left_join(., MNTD_Pic, "Identifier") %>% 
  mutate(Site_Oxygen = paste(Site, Oxygen)) %>% 
  mutate(Site_Oxygen_Hours = paste(Site, Oxygen, Hours)) ->
  MNTD_DF
```

``` r
# Calculate NTI using PhyloMeasures.
# NTI_PM <- tibble(Sample = rownames(otu_table(Ps_obj_top)), MNTD = -1 * mntd.query(tree = phy_tree(Ps_obj_top), matrix = otu_table(Ps_obj_top), standardize = T, null.model = "frequency.by.richness", abundance.weights = colSums(otu_table(Ps_obj_top)))) # only works on rooted trees

# using picante
NTI_Pic <- bind_cols(Identifier = rownames(otu_table(Ps_obj_SIP4tree_plot_labelled)), 
                     ses.mntd(samp = as(otu_table(Ps_obj_SIP4tree_plot_labelled), "matrix"), 
                              dis = cophenetic(phy_tree(Ps_obj_SIP4tree_plot_labelled)), 
                              null.model = "taxa.labels",
                              runs = 999, 
                              iterations = 999, 
                              abundance.weighted = TRUE))

# transform z-scores in NTI by multiplying by -1
NTI_Pic %<>% mutate(NTI = mntd.obs.z * -1)

save(NTI_Pic, file = "NTI.RData")
# load("NTI.RData")

# add MNTD data to metadata
MNTD_DF %<>%
  left_join(., NTI_Pic, "Identifier")
```

``` r
NTI_fig <- ggplot(data = MNTD_DF, 
                  aes(x = as.factor(Hours), 
                      y = NTI,  
           colour = Density..g.ml.1.)) +
  see::geom_point2(position = position_jitterdodge(), 
             size = 4, 
             alpha = 4/5)  +
  # geom_violinhalf(colour = "grey", scale = "area", trim = FALSE) +
  stat_summary(aes(group = as.factor(Hours)),
               fun.data = mean_cl_normal,
               fun.args = list(mult = 1),
               geom = "pointrange",
               colour = "black",
               alpha = 2/5,
               position = position_dodge(width = 0.5)) +
  facet_grid(Site ~ Oxygen) +
  scale_colour_viridis_c(name = "Density g ml^-1") +
  labs(x = "Incubation time (h)", 
       y = "NTI") +
  theme(legend.title = element_markdown(), 
        text = element_text(size = f_size + 2))

save_figure(paste0(fig.path, "NTI_fig"), 
            NTI_fig, 
            # pwidth = 16, 
            # pheight = 18,
            dpi = 300)
```

``` r
sessioninfo::session_info() %>%
  details::details(
    summary = 'Current session info',
    open    = TRUE
 )
```

<details open>
<summary>
<span title="Click to Expand"> Current session info </span>
</summary>

``` r
─ Session info ───────────────────────────────────────────────────────────────
 setting  value
 version  R version 4.2.1 (2022-06-23)
 os       Ubuntu 18.04.6 LTS
 system   x86_64, linux-gnu
 ui       X11
 language (EN)
 collate  en_US.UTF-8
 ctype    en_US.UTF-8
 tz       Europe/Prague
 date     2022-09-30
 pandoc   2.17.1.1 @ /usr/lib/rstudio-server/bin/quarto/bin/ (via rmarkdown)

─ Packages ───────────────────────────────────────────────────────────────────
 package              * version    date (UTC) lib source
 ade4                   1.7-19     2022-04-19 [1] CRAN (R 4.1.3)
 annotate               1.72.0     2021-10-26 [1] Bioconductor
 AnnotationDbi          1.56.2     2021-11-09 [1] Bioconductor
 ape                  * 5.6-2      2022-03-02 [1] CRAN (R 4.1.2)
 aplot                  0.1.7      2022-09-06 [1] CRAN (R 4.2.1)
 ashr                   2.2-54     2022-02-22 [1] CRAN (R 4.1.2)
 assertthat             0.2.1      2019-03-21 [1] CRAN (R 4.0.2)
 backports              1.4.1      2021-12-13 [1] CRAN (R 4.1.2)
 Biobase              * 2.54.0     2021-10-26 [1] Bioconductor
 BiocGenerics         * 0.40.0     2021-10-26 [1] Bioconductor
 BiocParallel           1.28.3     2021-12-09 [1] Bioconductor
 biomformat             1.22.0     2021-10-26 [1] Bioconductor
 Biostrings           * 2.62.0     2021-10-26 [1] Bioconductor
 bit                    4.0.4      2020-08-04 [1] CRAN (R 4.0.2)
 bit64                  4.0.5      2020-08-30 [1] CRAN (R 4.0.2)
 bitops                 1.0-7      2021-04-24 [1] CRAN (R 4.0.3)
 blob                   1.2.3      2022-04-10 [1] CRAN (R 4.1.3)
 broom                  1.0.1      2022-08-29 [1] CRAN (R 4.2.0)
 cachem                 1.0.6      2021-08-19 [1] CRAN (R 4.1.1)
 cellranger             1.1.0      2016-07-27 [1] CRAN (R 4.0.2)
 cli                    3.4.0      2022-09-08 [1] CRAN (R 4.2.1)
 clipr                  0.8.0      2022-02-22 [1] CRAN (R 4.1.2)
 cluster                2.1.4      2022-08-22 [1] CRAN (R 4.2.0)
 codetools              0.2-18     2020-11-04 [1] CRAN (R 4.0.2)
 colorspace             2.0-3      2022-02-21 [1] CRAN (R 4.1.2)
 crayon                 1.5.1      2022-03-26 [1] CRAN (R 4.1.2)
 data.table             1.14.2     2021-09-27 [1] CRAN (R 4.1.1)
 DBI                    1.1.3      2022-06-18 [1] CRAN (R 4.2.0)
 dbplyr                 2.2.1      2022-06-27 [1] CRAN (R 4.2.0)
 DelayedArray           0.20.0     2021-10-26 [1] Bioconductor
 desc                   1.4.2      2022-09-08 [1] CRAN (R 4.2.1)
 DESeq2               * 1.34.0     2021-10-26 [1] Bioconductor
 details                0.3.0      2022-03-27 [1] CRAN (R 4.1.3)
 digest                 0.6.29     2021-12-01 [1] CRAN (R 4.1.2)
 dplyr                * 1.0.10     2022-09-01 [1] CRAN (R 4.2.0)
 ellipsis               0.3.2      2021-04-29 [1] CRAN (R 4.0.3)
 evaluate               0.16       2022-08-09 [1] CRAN (R 4.2.0)
 extrafont            * 0.18       2022-04-12 [1] CRAN (R 4.1.3)
 extrafontdb            1.0        2012-06-11 [1] CRAN (R 4.0.2)
 fansi                  1.0.3      2022-03-24 [1] CRAN (R 4.1.2)
 farver                 2.1.1      2022-07-06 [1] CRAN (R 4.2.0)
 fastmap                1.1.0      2021-01-25 [1] CRAN (R 4.0.3)
 forcats              * 0.5.2      2022-08-19 [1] CRAN (R 4.2.0)
 foreach                1.5.2      2022-02-02 [1] CRAN (R 4.1.2)
 fs                     1.5.2      2021-12-08 [1] CRAN (R 4.1.2)
 gargle                 1.2.1      2022-09-08 [1] CRAN (R 4.2.1)
 genefilter             1.76.0     2021-10-26 [1] Bioconductor
 geneplotter            1.72.0     2021-10-26 [1] Bioconductor
 generics               0.1.3      2022-07-05 [1] CRAN (R 4.2.0)
 GenomeInfoDb         * 1.30.1     2022-01-30 [1] Bioconductor
 GenomeInfoDbData       1.2.7      2022-01-10 [1] Bioconductor
 GenomicRanges        * 1.46.1     2021-11-18 [1] Bioconductor
 ggfun                  0.0.7      2022-08-31 [1] CRAN (R 4.2.0)
 ggplot2              * 3.3.6      2022-05-03 [1] CRAN (R 4.1.3)
 ggplotify              0.1.0      2021-09-02 [1] CRAN (R 4.1.1)
 ggpomological        * 0.1.2      2020-08-13 [1] Github (gadenbuie/ggpomological@69f3815)
 ggrepel              * 0.9.1      2021-01-15 [1] CRAN (R 4.0.3)
 ggsci                * 2.9        2018-05-14 [1] CRAN (R 4.0.2)
 ggtext               * 0.1.2      2022-09-16 [1] CRAN (R 4.2.1)
 ggtree               * 3.2.1      2021-11-16 [1] Bioconductor
 glue                 * 1.6.2      2022-02-24 [1] CRAN (R 4.1.2)
 googledrive            2.0.0      2021-07-08 [1] CRAN (R 4.1.0)
 googlesheets4          1.0.1      2022-08-13 [1] CRAN (R 4.2.0)
 gridExtra              2.3        2017-09-09 [1] CRAN (R 4.0.2)
 gridGraphics           0.5-1      2020-12-13 [1] CRAN (R 4.0.2)
 gridtext               0.1.5      2022-09-16 [1] CRAN (R 4.2.1)
 gtable                 0.3.1      2022-09-01 [1] CRAN (R 4.2.0)
 haven                  2.5.1      2022-08-22 [1] CRAN (R 4.2.0)
 highr                  0.9        2021-04-16 [1] CRAN (R 4.0.3)
 hms                    1.1.2      2022-08-19 [1] CRAN (R 4.2.0)
 htmltools              0.5.3      2022-07-18 [1] CRAN (R 4.2.0)
 HTSSIP               * 1.4.1      2021-01-15 [1] Github (buckleylab/HTSSIP@29ec56b)
 httr                   1.4.4      2022-08-17 [1] CRAN (R 4.2.0)
 igraph                 1.3.5      2022-09-22 [1] CRAN (R 4.2.1)
 invgamma               1.1        2017-05-07 [1] CRAN (R 4.0.2)
 IRanges              * 2.28.0     2021-10-26 [1] Bioconductor
 irlba                  2.3.5      2021-12-06 [1] CRAN (R 4.1.2)
 iterators              1.0.14     2022-02-05 [1] CRAN (R 4.1.2)
 jsonlite               1.8.0      2022-02-22 [1] CRAN (R 4.1.2)
 kableExtra           * 1.3.4      2021-02-20 [1] CRAN (R 4.0.3)
 KEGGREST               1.34.0     2021-10-26 [1] Bioconductor
 knitr                  1.40       2022-08-24 [1] CRAN (R 4.2.0)
 labeling               0.4.2      2020-10-20 [1] CRAN (R 4.0.2)
 lattice              * 0.20-45    2021-09-22 [1] CRAN (R 4.1.1)
 lazyeval               0.2.2      2019-03-15 [1] CRAN (R 4.0.2)
 lifecycle              1.0.2      2022-09-09 [1] CRAN (R 4.2.1)
 locfit                 1.5-9.6    2022-07-11 [1] CRAN (R 4.2.0)
 lubridate              1.8.0      2021-10-07 [1] CRAN (R 4.1.1)
 magrittr             * 2.0.3      2022-03-30 [1] CRAN (R 4.1.3)
 MASS                   7.3-58.1   2022-08-03 [1] CRAN (R 4.2.0)
 Matrix                 1.5-1      2022-09-13 [1] CRAN (R 4.2.1)
 MatrixGenerics       * 1.6.0      2021-10-26 [1] Bioconductor
 matrixStats          * 0.62.0     2022-04-19 [1] CRAN (R 4.1.3)
 memoise                2.0.1      2021-11-26 [1] CRAN (R 4.1.2)
 mgcv                   1.8-40     2022-03-29 [1] CRAN (R 4.1.3)
 mixsqp                 0.3-43     2020-05-14 [1] CRAN (R 4.0.2)
 modelr                 0.1.9      2022-08-19 [1] CRAN (R 4.2.0)
 multtest               2.50.0     2021-10-26 [1] Bioconductor
 munsell                0.5.0      2018-06-12 [1] CRAN (R 4.0.2)
 nlme                 * 3.1-159    2022-08-09 [1] CRAN (R 4.2.0)
 patchwork            * 1.1.2      2022-08-19 [1] CRAN (R 4.2.0)
 permute              * 0.9-7      2022-01-27 [1] CRAN (R 4.1.2)
 phyloseq             * 1.38.0     2021-10-26 [1] Bioconductor
 picante              * 1.8.2      2020-06-10 [1] CRAN (R 4.0.2)
 pillar                 1.8.1      2022-08-19 [1] CRAN (R 4.2.0)
 pkgconfig              2.0.3      2019-09-22 [1] CRAN (R 4.0.2)
 plyr                   1.8.7      2022-03-24 [1] CRAN (R 4.1.3)
 png                    0.1-7      2013-12-03 [1] CRAN (R 4.0.2)
 purrr                * 0.3.4      2020-04-17 [1] CRAN (R 4.0.2)
 R6                     2.5.1      2021-08-19 [1] CRAN (R 4.1.1)
 ragg                 * 1.2.2      2022-02-21 [1] CRAN (R 4.2.0)
 RColorBrewer         * 1.1-3      2022-04-03 [1] CRAN (R 4.1.3)
 Rcpp                   1.0.9      2022-07-08 [1] CRAN (R 4.2.0)
 RCurl                  1.98-1.8   2022-07-30 [1] CRAN (R 4.2.0)
 readr                * 2.1.2      2022-01-30 [1] CRAN (R 4.1.2)
 readxl                 1.4.1      2022-08-17 [1] CRAN (R 4.2.0)
 reprex                 2.0.2      2022-08-17 [1] CRAN (R 4.2.0)
 reshape2               1.4.4      2020-04-09 [1] CRAN (R 4.0.2)
 rhdf5                  2.38.0     2021-10-26 [1] Bioconductor
 rhdf5filters           1.6.0      2021-10-26 [1] Bioconductor
 Rhdf5lib               1.16.0     2021-10-26 [1] Bioconductor
 rlang                  1.0.5      2022-08-31 [1] CRAN (R 4.2.0)
 rmarkdown              2.16       2022-08-24 [1] CRAN (R 4.2.0)
 rprojroot              2.0.3      2022-04-02 [1] CRAN (R 4.1.3)
 RSQLite                2.2.17     2022-09-10 [1] CRAN (R 4.2.1)
 rstudioapi             0.14       2022-08-22 [1] CRAN (R 4.2.0)
 Rttf2pt1               1.3.10     2022-02-07 [1] CRAN (R 4.1.2)
 rvest                  1.0.3      2022-08-19 [1] CRAN (R 4.2.0)
 S4Vectors            * 0.32.3     2021-11-21 [1] Bioconductor
 scales               * 1.2.1      2022-08-20 [1] CRAN (R 4.2.0)
 sessioninfo            1.2.2      2021-12-06 [1] CRAN (R 4.1.2)
 speedyseq            * 0.5.3.9018 2021-08-11 [1] Github (mikemc/speedyseq@ceb941f)
 SQUAREM                2021.1     2021-01-13 [1] CRAN (R 4.0.2)
 stringi                1.7.8      2022-07-11 [1] CRAN (R 4.2.0)
 stringr              * 1.4.1      2022-08-20 [1] CRAN (R 4.2.0)
 SummarizedExperiment * 1.24.0     2021-10-26 [1] Bioconductor
 survival               3.4-0      2022-08-09 [1] CRAN (R 4.2.0)
 svglite              * 2.1.0      2022-02-03 [1] CRAN (R 4.2.0)
 systemfonts            1.0.4      2022-02-11 [1] CRAN (R 4.1.2)
 textshaping            0.3.6      2021-10-13 [1] CRAN (R 4.1.1)
 tibble               * 3.1.8      2022-07-22 [1] CRAN (R 4.2.0)
 tidyr                * 1.2.1      2022-09-08 [1] CRAN (R 4.2.1)
 tidyselect             1.1.2      2022-02-21 [1] CRAN (R 4.1.2)
 tidytree               0.4.0      2022-08-13 [1] CRAN (R 4.2.0)
 tidyverse            * 1.3.2      2022-07-18 [1] CRAN (R 4.2.0)
 treeio                 1.18.1     2021-11-14 [1] Bioconductor
 truncnorm              1.0-8      2018-02-27 [1] CRAN (R 4.0.2)
 tzdb                   0.3.0      2022-03-28 [1] CRAN (R 4.1.3)
 utf8                   1.2.2      2021-07-24 [1] CRAN (R 4.1.0)
 vctrs                  0.4.1      2022-04-13 [1] CRAN (R 4.1.3)
 vegan                * 2.6-2      2022-04-17 [1] CRAN (R 4.1.3)
 viridis              * 0.6.2      2021-10-13 [1] CRAN (R 4.1.1)
 viridisLite          * 0.4.1      2022-08-22 [1] CRAN (R 4.2.0)
 visdat               * 0.6.0.9000 2022-02-18 [1] Github (ropensci/visdat@daa162f)
 vroom                  1.5.7      2021-11-30 [1] CRAN (R 4.1.2)
 webshot                0.5.3      2022-04-14 [1] CRAN (R 4.1.3)
 withr                  2.5.0      2022-03-03 [1] CRAN (R 4.1.2)
 xfun                   0.33       2022-09-12 [1] CRAN (R 4.2.1)
 XML                    3.99-0.10  2022-06-09 [1] CRAN (R 4.2.0)
 xml2                   1.3.3      2021-11-30 [1] CRAN (R 4.1.2)
 xtable                 1.8-4      2019-04-21 [1] CRAN (R 4.0.2)
 XVector              * 0.34.0     2021-10-26 [1] Bioconductor
 yaml                   2.3.5      2022-02-21 [1] CRAN (R 4.1.2)
 yulab.utils            0.0.5      2022-06-30 [1] CRAN (R 4.2.0)
 zlibbioc               1.40.0     2021-10-26 [1] Bioconductor

 [1] /home/angel/R/library
 [2] /usr/local/lib/R/site-library
 [3] /usr/lib/R/site-library
 [4] /usr/lib/R/library

──────────────────────────────────────────────────────────────────────────────
```

</details>

<br>

## References

<div id="refs" class="references csl-bib-body hanging-indent">

<div id="ref-angel_application_2018" class="csl-entry">

Angel R, Panhölzl C, Gabriel R *et al.* [Application of stable-isotope
labelling techniques for the detection of active
diazotrophs](https://doi.org/10.1111/1462-2920.13954). *Environ
Microbiol* 2018;**20**:44–61.

</div>

<div id="ref-love_moderated_2014" class="csl-entry">

Love MI, Huber W, Anders S. [Moderated estimation of fold change and
dispersion for RNA-seq data with
DESeq2](https://doi.org/10.1186/s13059-014-0550-8). *Genome Biol*
2014;**15**:550.

</div>

<div id="ref-stephens_fdr_2016" class="csl-entry">

Stephens M. [<span class="nocase">False discovery rates: a new
deal</span>](https://doi.org/10.1093/biostatistics/kxw041).
*Biostatistics* 2016;**18**:275–94.

</div>

<div id="ref-youngblut_htssip_2018" class="csl-entry">

Youngblut ND, Barnett SE, Buckley DH. [HTSSIP: An R package for analysis
of high throughput sequencing data from nucleic acid stable isotope
probing (SIP)
experiments](https://doi.org/10.1371/journal.pone.0189616). *PLOS ONE*
2018;**13**:e0189616.

</div>

</div>
