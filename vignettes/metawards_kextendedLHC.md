---
title: "Example Metawards Design"
author: "Danny Williamson"
date: "15/05/2020"
output: html_document
references:
- id: willi15
  title: Exploratory designs for computer experiments using k-extended Latin Hypercubes
  author:
  - family: Williamson
    given: Daniel
  container-title: Environmetrics
  volume: 26
  URL: 'https://doi.org/10.1002/env.2335'
  DOI: 10.1002/env.2335
  issue: 4
  page: 268-283
  type: article-journal
  issued:
    year: 2015
layout: default
---



## Designing Metawards Ensembles for UQ

This vignette will demonstrate a UQ design for Metawards. For now, we will be designing on $[-1,1]$ and other scripts will convert these designs, what I am terming "UQ designs" to the right scales for Metawards. Later, the parameters on $[-1,1]$ will be combined with the data for UQ.

You can use any design that you like, so what comes in the next section is merely a suggestion (but at least one with theoretically attractive properties). To design for Metawards, currently the parameter names are:


```r
Param_Names <- c("incubation_time", "infectious_time", 
                 "r_zero", "lock_1_restrict", "lock_2_release")
Param_Names
```

```
## [1] "incubation_time" "infectious_time" "r_zero"          "lock_1_restrict"
## [5] "lock_2_release"
```

and it is important that your final design has an additional column called "Repeats". If you don't wish to read about designing for a stochastic simulator or will use your own method regardless, please skip to the **Completing your design** section.

### Designing with repeats for a stochastic simulator

For stochastic simulators, you cannot run the model just once for a parameter choice as the answer for any run is only really a draw from the distribution of the model output induced at the given input choice. The defaults for most simulators, including Metawards, is to repeat every run a fixed number of times (say 10 for argument sake). But this idea is incredibly wasteful and limits the ability of the user to properly explore parameter space. 

Much more in the spirit of clever designs for this type of problem is to have a design that has a sub-design with repeats, but that also uses computer power to explore more parameter space. Of course, there are many ways one could think of to achive that. I will suggest a method here that has some attractive features.

First, what would make a good design for this type of problem? The main desirable features for exploratory designs are space fillingness and orthogonality. Space filling designs are obviously important: we want to explore as much of parameter space as possible ready for emulation. Orthogonality is also critical as it ensures that any effects in each parameter are identifiable.

The twist in these problems is that you need repeats to understand the variability in parameter space and how it changes with the parameter space. So the runs that you repeat should have all of the above attractive properties. Also, you want to fill space as much as possible to learn about the mean effects, so your non-repeating design should also fill space. Furthermore, it should fill space (and be orthogonal) in relation to the repeats design (which also tells you about the mean). So you have a design where you want a space filling and orthogonal subdesign and where you want the composite to also be space filling and orthogonal. k-extended Latin Hypercubes [@willi15], were designed with exactly that purpose in mind.

### k-extended Latin Hypercubes

Far more than you could ever want to know about k-extended Latin Hypercubes is discussed in [@willi15]. The basic idea is this: Suppose your input dimension is $d$. First you produce a space filling and orthogonal LHC of size $N$. This will be the LHC where you repeat all of the runs. This LHC is then extended using $k-1$ LHCs of size $N$ so that the composite LHC is $kN \times d$. A score function that trades of space-filling and orthogonality is optimised at each extension so that: Each extension is optimal with respect to the currently generated sub-cubes and the final design is optimal with respect to the construction and the given critera (methods and proofs are in [@willi15]).

The code for generating k-extended LHCs can be found at
[github.com/UQ4covid/uq4covid/tools/make_design/kExtendedLHC.R](https://github.com/UQ4covid/uq4covid/blob/master/tools/make_design/kExtendedLHCs.R) and local copy is also saved [here](kExtendedLHCs.R)


```r
source("kExtendedLHCs.R")
```

```
## Loading required package: lhs
```

```
## Loading required package: parallel
```

I will now generate a design for Metawards as a k-extended LHC, discussing the options as we go (so that if you want to use this design method, you can tweak the parameters). Of course, you can use your own design code if you are used to emulating stochastic simulators, but if not, this can save you time (and it's also pretty good).

First we need to choose how big the LHC with the repeats will be and how many repeats. My choices are "for example" and bigger ensembles will be run when we are ready.


```r
Rep_Ens_Size <- 25
Nreps <- 5
```
So we will have a LHC with $25$ runs and each run is repeated $5$ times. If when looking at the data we see huge variation in variability across parameter space it might be wise to revisit the number of repeats and we may need $50$ members.

Next, how big is the final LHC going to be? I am going to have a $100$ member LHC so I will want a 4-extended LHC.


```r
HowManyCubes <- 4
```

Now we construct the k-extended Latin Hypercube. The parameters `w` and `FAC_t` control the optimisation of the LHC. `w` is the weight given to orthogonality vs space filling properties and `FAC_t` controls the cooling in the simluated annealing algorithm. 


```r
New_cube <- MakeRankExtensionCubes(n=Rep_Ens_Size, m=length(Param_Names), 
                                   k=HowManyCubes, w=0.2, FAC_t=0.5)
```

I've hidden the output here as there are a lot of print outs giving the status of the optimisation. When the lone numbers reach order $10^{-08}$ you can normally expect that the algorithm has almost converged. If it is too slow, adjust `FAC_t`.

The LHC is then completed with 


```r
newExtended <- NewExtendingLHS(New_cube)
colnames(newExtended) <- Param_Names
```

We can plot the design colouring the points by whether they are in the repeat design or not:


```r
pairs(newExtended, 
      col=c(rep(2, Rep_Ens_Size), rep(3, Rep_Ens_Size*(HowManyCubes-1))), 
      pch=16)
```

![plot of chunk unnamed-chunk-7](figure/metawards_kextendedLHC.Rmd//unnamed-chunk-7-1.png)

We can also plot the design with each sub-cube coloured individually:


```r
tcols <- rep(2, Rep_Ens_Size)
for (j in 1:(HowManyCubes-1)) {
  tcols <- c(tcols, rep(2+j, Rep_Ens_Size))
}
pairs(newExtended, col=tcols, pch=16)
```

![plot of chunk unnamed-chunk-8](figure/metawards_kextendedLHC.Rmd//unnamed-chunk-8-1.png)

Note how each subdesign is space filling whilst ensuring a quality composite space filling design.

We now ensure that this example design is fit for running with Metawards.

### Completing your design

First ensure your design is on $[-1,1]$. The design `newExtended` generated above is on $[0,1]$, so first we map to the right space:


```r
newExtended <- 2*newExtended - 1
```

Now we ensure the columns are named and we cast into a data frame. You may cast into your own R object ready for analysis later. The important thing is that you get to a `.csv` at the end...


```r
colnames(newExtended) <- Param_Names
EnsembleDesign <- as.data.frame(newExtended)
head(EnsembleDesign)
```

```
##   incubation_time infectious_time     r_zero lock_1_restrict lock_2_release
## 1      -0.5588436      0.72659085  0.8286644       0.5528692    -0.04646315
## 2      -0.6971701     -0.01592854  0.5644243      -0.5651912     0.43394167
## 3       0.4297743     -0.75674184 -0.5462618      -0.6895794    -0.75504154
## 4       0.5468641      0.47361914 -0.7788059       0.1592936     0.59271992
## 5       0.9952420      0.33934277  0.8827025       0.3557198    -0.32253715
## 6       0.6061667      0.90287160  0.4641745      -0.2931559     0.29380206
```

We now need to add the repeats column. In the previous section we used k-extended Latin Hypercubes to construct a special sub design that would have repeats with the remainder of the design not repeated (and also space filling etc), so we add this repeat structure here, if using another method you will need to think carefully about this repeat structure.


```r
EnsembleDesign <- cbind(EnsembleDesign, 
                        Repeats=c(rep(Nreps, Rep_Ens_Size), 
                                  rep(1, Rep_Ens_Size*(HowManyCubes-1))))
head(EnsembleDesign)
```

```
##   incubation_time infectious_time     r_zero lock_1_restrict lock_2_release
## 1      -0.5588436      0.72659085  0.8286644       0.5528692    -0.04646315
## 2      -0.6971701     -0.01592854  0.5644243      -0.5651912     0.43394167
## 3       0.4297743     -0.75674184 -0.5462618      -0.6895794    -0.75504154
## 4       0.5468641      0.47361914 -0.7788059       0.1592936     0.59271992
## 5       0.9952420      0.33934277  0.8827025       0.3557198    -0.32253715
## 6       0.6061667      0.90287160  0.4641745      -0.2931559     0.29380206
##   Repeats
## 1       5
## 2       5
## 3       5
## 4       5
## 5       5
## 6       5
```

Finally, all we have to do is write out to `.csv`. I prefer to also save a copy of the design, but that is a matter for your own taste.


```r
save(EnsembleDesign, file = "EnsembleDesign.RData")
write.csv(EnsembleDesign, file = "EnsembleDesign.csv", row.names = FALSE)
```

The `csv` file containing the final design is available [here](EnsembleDesign.csv) and at [github.com/UQ4covid/uq4covid/tools/make_design/ExampleEnsembleDesign.csv](https://github.com/UQ4covid/uq4covid/blob/master/tools/make_design/ExampleEnsembleDesign.csv). 

# References

