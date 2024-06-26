---
title: "BI 3083/8082 - Animal model"
author: "Debora Goedert"
date: "11/1/2022"
output: github_document
---
**Disclaimers**: 

- NTNU has not provided resources for the development of this activity and, therefore, this material does not fall under regulations stipulated by the "Intellectual Property Rights (IPR) and Physical Material and Results generated at NTNU". As per Norwegian's Individual Intellectual Property Rights, intellectual work cannot be commercialized or distributed without the owner's permission. Please do not share/distribute this activity.  
- This script is based on the supporting material accompanying the package MCMCglmm: Hadfield 2021. MCMCglmm Course Notes. Accessed on 2021-10-18 and on years of gathering information from various other sources.  
- The data used here is implemented in the package, and originates from the study: Hadfield et al 2007.  

# OBJECTIVES:  

For this activity, we'll learn how to estimate the additive genetic variance considering the phenotypic values and the genetic relationship taken from a pedigree. We'll then use the additive genetic variance obtained from the analyses to estimate various quantitative genetic parameters of interest, such as the narrow-sense heritability and the evolvability of a trait.  

## STEP 1: Load necessary R-packages and datasets  
```{r}
library(MCMCglmm)

data(BTdata)
data(BTped)
```
### STEP 2: Run an animal model.  

The simplest structure that can be fitted by the MCMCglmm is in the format shown below:   
```{r}
set.seed(3083)
back.model.inter<-MCMCglmm(back~1, random=~animal, data=BTdata, pedigree=BTped, nitt=65000, thin=50, burnin=5000)
```

Here are a few things you need to understand from this structure:  
	 - the trait that is being fitted is "back" or, as you saw from the description of the dataset, the back coloration for the birds.  
	 - we saw before that the argument "random" specifies the random effects of the model, in this case the individual ID. When the argument "pedigree" is used, however, this random effect changes into a matrix of relationships between the individuals in the dataset.  
	- the notation "~1" specifies to the model that we expect it to fit an intercept. In this case, the intercept is the mean value of the fitted trait when accounting for the genetic relatedness across individuals within the dataset.  

Keep in mind that we always need to inspect the run before proceeding with the conclusions. Since these models are computationally intensive, however, we might need to have the model run for various minutes and, sometimes, many hours. For the purposes of this activity, we'll assume the model is ok. Note the codes below but skip this step  
```{r}
# plot(back.model.inter$Sol)
# plot(back.model.inter$VCV)

# effectiveSize(back.model.inter$Sol)
# effectiveSize(back.model.inter$VCV)

# autocorr(back.model.inter$Sol)
# autocorr(back.model.inter$VCV) # This output shows you that there is high correlation at the start of the chain. This suggests we would need to run the model for longer and increase the burnin period.

# heidel.diag(back.model.inter$Sol)
# heidel.diag(back.model.inter$VCV)
```

Let's look at the output:  
```{r}
summary(back.model.inter)
```

## STEP 3: Estimating quantitative genetics parameters:  
In the animal models, our additive genetic variance is stored in the component "animal". You may think of that as:  
	 - since the random effect of the model is, in the case of the added pedigree, a matrix of the genetic relationship between individuals in our dataset, the variance estimated by this random effect is the variance in phenotype that is due to the heritable, additive genetic component.   
	 - since breeding values are the statistics that express the amount of phenotypic variance that is passed on across generations due to additive effects of alleles, the phenotypic variance that is due to additive genetic effects has to be the variance of breeding values in the population.  

The total phenotypic variance is estimated by adding all of the variance components. In the case of this simple format, the only variance besides the additive genetic variance is the residual variance stored in the component "units".  

### STEP 3.a: Narrow-sense heritability (h2)
Remember that the narrow-sense heritability is the proportion of total phenotypic variance due to the additive component (i.e. Va/Vp)  
```{r}
back.model.inter.Va<-back.model.inter$VCV[,'animal']
back.model.inter.Vp<-back.model.inter$VCV[,'animal']+back.model.inter$VCV[,'units']
back.model.inter.h2<-back.model.inter.Va/back.model.inter.Vp
```

Note that this statistics is being calculated for each iteration of the model and, therefore, we create a [posterior] distribution of h2 values. You can see that with a plot:  
```{r}
plot(back.model.inter.h2)
```

We can then output the mean of this distribution but, more commonly, we report the mode and the credible intervals:
```{r}
posterior.mode(back.model.inter.h2)

## Credible intervals for the 95% HPD of the distribution:
HPDinterval(back.model.inter.h2)
```

### STEP 3.b: Evolvability (Ia)  

Although heritabilities have the immediate appeal of indicating the ability of a trait to respond to selection, heritabilities are not comparable across traits or taxa. This is due to its nature as a proportion or a ratio of two other estimates. Any ratio can change if the nominator OR if the denominator of the equation changes, and two taxa/traits with the same heritability values could present much different Va and Vp. Therefore, in addition to the heritability estimates, it is good practice to present the interested reader with an additional estimate - the evolvability (sensu Houle 1992). Ia = Va / (mean z)^2, where z is the trait of interest
```{r}
back.mean<-mean(BTdata$back)
back.model.inter.Ia<-back.model.inter.Va/(back.mean^2)

plot(back.model.inter.Ia)
posterior.mode(back.model.inter.Ia)
HPDinterval(back.model.inter.Ia)
```

Note that these values are extremely large here. This is unusual and an artifact of the distribution of the trait "back". The scaling of this variable causes the mean to be ~0, while the range of the values is proportionally huge.  
```{r}
plot(BTdata$back)
abline(h=0, col="red", lty="dashed", lwd=2)

back.mean
range(BTdata$back) # this shows the lowest and largest values
```

Traits in nature, however are not normally centered around 0 with such [relatively] large ranges - see Extra section #1 for a "for instance"".

## STEP 4: Adding more variables.  
Just like in a regular mixed effect model, we can add extra fixed effect variables that we think might explain the variation in the trait of focus. For example, body size and sex are common ones. As before, we'll skip the checks for the purpose of the course.  

**ATTENTION: WE'LL PLOT ALL OF THE ESTIMATES IN THE END, SO MOVE QUICKLY THROUGH THE CALCULATIONS IN SECTIONS 4a-c (DON'T SPEND TIME GOING BACK AND FORTH ON THE OUTPUTS)**  

### STEP 4.a: Add a continuous fixed effect.   
Here we'll use tarsus length, which is a commonly used proxy for body size in birds, as a continuous fixed effect.  
```{r}
set.seed(3083)
back.model.size<-MCMCglmm(back~tarsus, random=~animal, data=BTdata, pedigree=BTped, nitt=65000, thin=50, burnin=5000, verbose=F)
summary(back.model.size)

# plot(back.model.size$Sol)
# plot(back.model.size$VCV)
# effectiveSize(back.model.size$Sol)
# effectiveSize(back.model.size$VCV)
# autocorr(back.model.size$Sol)
# autocorr(back.model.size$VCV) # We again have some problems with autocorrelation and the need to increase the length of the chain and burnin period
# heidel.diag(back.model.size$Sol)
# heidel.diag(back.model.size$VCV)
```

Now let's see how adding this variable influenced the heritability estimates:
```{r}
back.model.size.Va<-back.model.size$VCV[,'animal']
back.model.size.Vp<-back.model.size$VCV[,'animal']+back.model.size$VCV[,'units']
back.model.size.h2<-back.model.size.Va/back.model.size.Vp

# plot(back.model.size.h2)
posterior.mode(back.model.size.h2)
HPDinterval(back.model.size.h2)
```

### STEP 4.b: Add a factorial fixed effect.  
Here we'll add the variable sex.  
```{r}
set.seed(3083)
back.model.sex<-MCMCglmm(back~sex-1, random=~animal, data=BTdata, pedigree=BTped, nitt=65000, thin=50, burnin=5000, verbose=F)
summary(back.model.sex)
# plot(back.model.sex$Sol)
# plot(back.model.sex$VCV)
# effectiveSize(back.model.sex$Sol)
# effectiveSize(back.model.sex$VCV)
# autocorr(back.model.sex$Sol)
# autocorr(back.model.sex$VCV) # We again have some problems with autocorrelation and the need to increase the length of the chain and burnin period
# heidel.diag(back.model.sex$Sol)
# heidel.diag(back.model.sex$VCV)
```

Now let's see how adding this variable influenced the heritability estimates:  
```{r}
back.model.sex.Va<-back.model.sex$VCV[,'animal']
back.model.sex.Vp<-back.model.sex$VCV[,'animal']+back.model.sex$VCV[,'units']
back.model.sex.h2<-back.model.sex.Va/back.model.sex.Vp

# plot(back.model.sex.h2)
posterior.mode(back.model.sex.h2)
HPDinterval(back.model.sex.h2)
```

### STEP 4.c: Add the interaction between sex and size:
```{r}
set.seed(3083)
back.model.sizesex<-MCMCglmm(back~tarsus*sex-1, random=~animal, data=BTdata, pedigree=BTped, nitt=65000, thin=50, burnin=5000, verbose=F)
summary(back.model.sizesex)
# plot(back.model.sizesex$Sol)
# plot(back.model.sizesex$VCV)
# effectiveSize(back.model.sizesex$Sol)
# effectiveSize(back.model.sizesex$VCV)
# autocorr(back.model.sizesex$Sol)
# autocorr(back.model.sizesex$VCV) # We again have some problems with autocorrelation and the need to increase the length of the chain and burnin period
# heidel.diag(back.model.sizesex$Sol)
# heidel.diag(back.model.sizesex$VCV)
```

Now let's see how adding these variables influenced the heritability estimates:
```{r}
back.model.sizesex.Va<-back.model.sizesex$VCV[,'animal']
back.model.sizesex.Vp<-back.model.sizesex$VCV[,'animal']+back.model.sizesex$VCV[,'units']
back.model.sizesex.h2<-back.model.sizesex.Va/back.model.sizesex.Vp

# plot(back.model.sizesex.h2)
posterior.mode(back.model.sizesex.h2)
HPDinterval(back.model.sizesex.h2)
```

### STEP 4.d: Compare by plotting:  
We'll plot all of the estimates of heritability we calculated here. Don't worry about the details for now (but to understand it, you should try breaking down in small parts and check what each piece of the code does!), just run the code:  
```{r}
library(ggplot2) 

allmyestimates<-data.frame(model=as.factor(c("Intercept", "Size", "Sex", "SizeSex")), myh2=c(posterior.mode(back.model.inter.h2), posterior.mode(back.model.size.h2), posterior.mode(back.model.sex.h2), posterior.mode(back.model.sizesex.h2)), LowerCi=c(HPDinterval(back.model.inter.h2)[1],HPDinterval(back.model.size.h2)[1],HPDinterval(back.model.sex.h2)[1],HPDinterval(back.model.sizesex.h2)[1]), UpperCi=c(HPDinterval(back.model.inter.h2)[2],HPDinterval(back.model.size.h2)[2],HPDinterval(back.model.sex.h2)[2],HPDinterval(back.model.sizesex.h2)[2]),
myVa=c(posterior.mode(back.model.inter.Va), posterior.mode(back.model.size.Va), posterior.mode(back.model.sex.Va), posterior.mode(back.model.sizesex.Va)),
myVp=c(posterior.mode(back.model.inter.Vp), posterior.mode(back.model.size.Vp), posterior.mode(back.model.sex.Vp), posterior.mode(back.model.sizesex.Vp)))

allmyestimates$model<-factor(allmyestimates$model, levels=c("SizeSex", "Sex", "Size", "Intercept"))
ggplot(data=allmyestimates, aes(x=myh2, y=model))+
	geom_point(size=5)+xlim(0.15, 0.55)+
	xlab("Heritability of Dorsal Coloration")+ ylab("Model")+
	geom_segment(aes(x=UpperCi, xend=LowerCi, y=model, yend=model))+
	geom_vline(xintercept=posterior.mode(back.model.inter.h2), linetype="dotted", color="red")+
	theme_bw()
```

**Q1. Did adding fixed effects change the estimates of heritability?**  
**Q2. Why or why not should we add these fixed effects in the model?**  

## STEP 5: Save the estimates  
We'll move on to more complex structures of the animal model in the next two classes, and it will be nice to compare the outputs there with what we have here. Let's save this output so we don't need to re-run the estimations:  
```{r}
# # this step is optional, and will create a subdirectory in your current working directory - change the path as necessary if you would like to create this directory elsewhere, skip it to simply save in the wd, etc.
system('mkdir ./1Estimates') 
# 
# # before saving the table, make sure to change the path to where you want this dataset saved.
write.csv(allmyestimates, "./1Estimates/Heritabilities_IntroAnimalModels.csv", row.names=F) 
```

## FINAL STEP: HOUSE KEEPING:  
Now that you are done with the main work, don't forget to save your R session info. Remember that this output indicates to you the versions of R and the packages you needed to load to run this script. In the future, if alterations are made to the packages, you can always consult back this script and explains differences, etc.   
```{r}
sessionInfo()
citation(package="MCMCglmm")
citation(package="ggplot2")
citation()
```

## EXTRA SECTION:

### EXTRA 1: How standardization affects estimates  
We saw that the estimates of evolvability were extremely large for the variable "back" and I told you that this was an artifact of the transformation used in the data. Here I demonstrate what I mean.  

First, let's create a dummy variable for "back" with no negative values. Note that I am not changing the relative phenotypic values across individuals.  
```{r}
back.positive<-BTdata$back+abs((min(BTdata$back))) 
## check that it worked
plot(back.positive) 
## check that we are not changing the relative values of the trait, just increasing the scale so values are positive
plot(back.positive~BTdata$back) 
```

Now, re-run the same model as the "back.model.inter", but with this new variable we created, and re-estimate Va, Vp and h2:  
```{r}
set.seed(3083) # same seed as before
backpos.model.inter<-MCMCglmm(back.positive~1, random=~animal, data=BTdata, pedigree=BTped, nitt=65000, thin=50, burnin=5000, verbose=F)

## Get estimates:
backpos.model.inter.Va<-backpos.model.inter$VCV[,'animal']
backpos.model.inter.Vp<-backpos.model.inter$VCV[,'animal']+backpos.model.inter$VCV[,'units']
backpos.model.inter.h2<-backpos.model.inter.Va/backpos.model.inter.Vp

## Let's compare these estimates with the ones we estimated before:
## Va:
mean(back.model.inter.Va) 
mean(backpos.model.inter.Va) 

## Vp:
mean(back.model.inter.Vp) 
mean(backpos.model.inter.Vp) 

## h2:
posterior.mode(back.model.inter.h2)
posterior.mode(backpos.model.inter.h2) 
```

Note that the estimates of Va, Vp and, therefore, the proportion of phenotypic variance due to additive effects (aka the narrow-sense heritability) haven't changed!  

What about the evolvability?
```{r}
(back.positive.mean<-mean(back.positive)) ## The mean changes a lot!!
backpos.model.inter.Ia<-backpos.model.inter.Va/(back.positive.mean^2)
mean(backpos.model.inter.Ia) ## Due to the change in mean, we see a huge difference in the estimate of Ia
plot(backpos.model.inter.Ia)
```
