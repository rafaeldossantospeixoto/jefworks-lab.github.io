---
layout: post
comments: true
tags: [probability, statistics, single cell]
---

# Barcode doublets in single cell RNA-seq data

When we perform single cell RNA-seq, we expect that the derived gene expression quantifications come from a single cell. Publications have shown that this is not always the case, as two cells may be physically captured together, resulting in a physical 'doublet' or 'multiplet' capture that is estimated to occur at a rate of 1.6%. Thus the derived gene expression quantifications would be a reflection of two cells, perhaps even vastly different cells or cell types, rather than a single cell. This of course presents analytical challenges if such doublets are misinterpreted as intermediate cell types. While these physical doublets have received much attention, what I will call "barcode doublets" seem to have raised fewer concerns. In this blog posts, we will use some math to explore why. 

## Definition of barcode doublets

In newer droplet-based single cell RNA-seq techniques, thousands of single cell are captured with barcoded oligo beads in droplets using microfluidic compartmentalization. Then, inside each given droplet, an individual cell is lysed and its mRNA are tagged by the barcoded bead. Droplets are then broken and the single cell barcoded mRNAs are all pooled together, reverse transcribed, amplified, and sequenced. Because of these barcodes, the resulting sequenced reads can be demultiplexed back into single cells. 

However, this techniques assumes that each cell receives a unique barcode. In the event two cells receive the same barcode, all their mRNAs will also be tagged by the same barcode. Thus, after sequencing, there is no way to resolve which cell each mRNA came from.

A barcode doublet is thus when two cells receive the same barcoded oligo bead. 

## Do barcode doublets exist in single cell RNA-seq data?

To address whether barcode doublets exist in single cell RNA-seq data, let's first think about a related problem: the birthday paradox. Have you ever heard the statistic that in a room of just 23 people, there’s a 50-50 chance of two people having the same birthday yet in a room of 75 people, there’s a 99.9% chance of two people matching? This is the birthday paradox and it is related to our barcode doublets problem. This is because the birthday paradox boils down to there being only 365 days in the year, so the probability that any two people will have the same birthday is given by the Taylor series approximation (there are many Wikipedia articles you can reference for deriving this equation):

```
p = 1-e(-n^2/(2*b))
```

Where n is the number of people, b is the number of birthdays, and p is the probability that there will be at least one "collision" ie. shared birthday.

So let's check our fun fact real quick:

```
1-e^(-23^2/(2*365)) = 0.5
1-e^(-75^2/(2*365)) = 0.999
```

So indeed, if we are in a room with 75 people, there will surely be two people with the same birthday (p=0.999)!

Where this equation relates to our single cell problem is that we have potentially thousands of single cells (just like people) and a limited number of barcodes (just like birthdays). Luckily, unlike birthdays where we can't create more days in the year, we can create more barcodes. But how many barcodes will we need to minimize these barcode doublets?

For the 10X system, approximately 750000 such barcodes are used. Let's assume that each 10X run will consist of 2500 cells. Using our former equation with `n=2500`, and `b=750000`:

```
1-e^(-2500^2/(2*750000)) = 0.984
```  

That means almost surely there will be a barcode doublet!

However, keep in mind that the probability we are calculating is whether there will be any barcode doublets (just like we were calculating the probability any two people will share the same birthday). What we are more interested in perhaps is the fraction or rate of barcode doublets ie. how many of such barcode doublets we expect to see as a function of the number of cells.

## Are barcode doublets a serious issue in single cell RNA-seq?

To address whether barcodes doublets present a serious issue in single cell RNA-seq data, let's reformulate the problem: If we consider each barcode as a bucket and randomly throw cells into these buckets, how many buckets will have more than 1 cell? Since the number of buckets is quite large, this is starting to sound like a Poisson distribution.

So the number of balls per bucket should follow a Poisson distribution with lambda = number of balls / number of buckets. The fraction of buckets with K balls is:

```
lambda^K*exp(-lambda)/K!
```  

Let's assume the number of balls is equal to the number of buckets so lambda = 1. When lambda = 1, then the fraction of buckets with 0 balls is 0.368. Likewise, the fraction of buckets with 1 ball is 0.184. So the fraction of buckets with more than 1 ball is `1 - 0.184 - 0.369 = 0.447`. That's pretty high! But this is why we have way more barcodes than we have cells!

More realistically, lambda is on the order of 2500/750000, at least for 10X experiments. Now, the fraction of barcodes with no cells is `exp(-2500/750000)/0! = 0.997` as expected from the more naive calculation of `1-(2500/750000)`. And likewise, the fraction of barcodes with 1 cell is `(2500/750000)*exp(-2500/750000)/1! = 0.003` as expected from the more naive calculation of `2500/750000`. And thus the fraction of barcodes with more than 1 cell is `1 - 0.997 - 0.003`, which is essentially 0. 

So it looks like for most single cell RNA-seq experiments, barcode doublets, while they most surely occur, do not occur at a sufficiently high rate to cause concern.

# But what happens when we increase the number of cells per run?

[A recent publication suggested increasing the number of cells per 10X run to upwards of 100,000 cells from many different patients](https://www.nature.com/articles/nbt.4042). For the sake of dramatization, let's consider what happens if we smoosh a whopping 200000 cells into one 10X run without increasing the number of barcodes. Now `lambda = 200000/750000`. So the fraction of barcodes with more than 1 cells is `1 - exp(-200000/750000)/0! - (200000/750000)*exp(-200000/750000)/1! = 0.03` or approximately 3%. 

While this 3% may seem high, when we start smooshing 200000 cells into one 10X run, [physical doublets will start becoming a much bigger issue](http://core-genomics.blogspot.com/2016/07/10x-genomics-single-cell-3mrna-seq.html) that will effectively dwarf concerns raised by these barcode doublets. In addition, synthesis errors of barcodes and sequencing error of barcodes also adds another layer of complexity. In the grand scheme of other technical errors that will surely occur and propagate, barcode doublets seem like a rather minor issue and probabilistically unlikely to cause much concern. 

---

# References:
- https://www.ncbi.nlm.nih.gov/pubmed/26000487
- http://macoskolab.com/wp-content/uploads/2016/10/Macosko2015-1.pdf
- https://www.dolomite-bio.com/how-it-works/drop-seq/
- https://en.wikipedia.org/wiki/Birthday_problem
