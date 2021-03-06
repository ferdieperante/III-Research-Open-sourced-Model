# 📊 Data Analysis

## Initializing the Data and Libraries
```{r}
library(caret)
library(rfPermute)
library(rfUtilities)
library(readr)
library(car)
library(MASS)
library(tseries)
library(lmtest)
library(tidyverse)
library(GA)
library(mice)
library(caTools)
library(rsample)
library(gbm)
library(glmnet)
library(randomForest)

LFS2015 <- read_csv("./LFS2015.csv")
LFS2015 <- as.data.frame(LFS2015)
summary(LFS2015)
str(LFS2015)
```

## Creating the First Model

```{r}
## Create untuned model
LFS2015.rfP <- rfPermute(
  C40_WKS ~ ., 
  data = LFS2015,
  ntree = 1000, ## ntree maximized to minimize OOB sample error
  na.action = na.omit,
  nrep = 100,
  num.cores = 6
)

## Feature selection: check for unimportant variables
plot(
  rp.importance(LFS2015.rfP, scale = FALSE)
  )

## Create tuned model
LFS2015.rfPtuned <- rfPermute(
  C40_WKS ~ ., 
  data = LFS2015[c(-7, -10, -8, -2)], ## Exclude unimportant variables
  ntree = 1000,
  na.action = na.omit,
  nrep = 100, 
  num.cores = 6
)

```

## Selecting the two other seperately tuned model for model comparison
```{r}
## Preparing: creating training and test sets for the genetic algorithm's objective function

LFS2015 <- na.omit(LFS2015)
md.pattern(LFS2015)

set.seed(123)

split=initial_split(LFS2015,prop=0.7)
data.train=training(split)
data.test=testing(split)

## Using the best model, selected from a genetic algorithm, for the second

fit_rf=function(x)
{
  ntree=binary2decimal(x[1:8])
  mtry=binary2decimal(x[8:9])
  nrep=binary2decimal(x[9:13])
  if(ntree==0)
  {
    ntree=1
  }
  if(mtry==0)
  {
    mtry=1
  }
  if(nrep==0)
  {
    nrep=1
  }
  model=rfPermute(C40_WKS ~ .,data=data.train, mtry=mtry,
            ntree=ntree,nrep=nrep)
  predictions=predict(model,data.test)
  rmse=sqrt(mean((data.test$C40_WKS-predictions)^2))
  return(-rmse) 
}

GA3_a=ga(type='binary',fitness=fit_rf,nBits=15,
       maxiter=30,popSize=50,seed=1234,keepBest=TRUE)

ga_rf_fit_a=as.data.frame(GA3_a@solution)

ga_ntree_a=apply(ga_rf_fit_a[, 1:8],1,binary2decimal)
ga_mtry_a=apply(ga_rf_fit_a[, 8:10],1,binary2decimal)
ga_nrep_a=apply(ga_rf_fit_a[, 10:14],1,binary2decimal)

ga_ntree_a
ga_mtry_a
ga_nrep_a

## Select the third from the best feature-engineered model of a genetic algorithm.

## Feature selection: check for unimportant variables

model=rfPermute(C40_WKS ~ .,data=data.train, mtry=2, ntree=50, nrep=5)

plot(
  rp.importance(LFS2015.rfP, scale = FALSE)
  )

## Selecting the best feature-engineered model from the genetic algorithm

fit_rf=function(x)
{
  ntree=binary2decimal(x[1:8])
  mtry=binary2decimal(x[8:9])
  nrep=binary2decimal(x[9:13])
  if(ntree==0)
  {
    ntree=1
  }
  if(mtry==0)
  {
    mtry=1
  }
  if(nrep==0)
  {
    nrep=1
  }
  model=rfPermute(C40_WKS ~ .,data=data.train[c(-7, -10, -3, -2)],mtry=mtry,
            ntree=ntree, nrep=nrep)
  predictions=predict(model,data.test)
  rmse=sqrt(mean((data.test$C40_WKS-predictions)^2))
  return(-rmse) 
}

GA3_b=ga(type='binary',fitness=fit_rf,nBits=15,
       maxiter=30,popSize=50,seed=1234,keepBest=TRUE)

ga_rf_fit_b=as.data.frame(GA3_b@solution)

ga_ntree_b=apply(ga_rf_fit_b[, 1:8],1,binary2decimal)
ga_mtry_b=apply(ga_rf_fit_b[, 8:10],1,binary2decimal)
ga_nrep_b=apply(ga_rf_fit_b[, 10:14],1,binary2decimal)

ga_ntree_b
ga_mtry_b
ga_nrep_b



## model comparison
print(LFS2015.rfPtuned) ## first model
summary(GA3_a) ## second model
summary(GA3_b) ## third model

```

## Model Interpretation
```{r}
## Feature Importance and Significance
plot(
  rp.importance(LFS2015.rfPtuned, scale = FALSE),
  alpha = 0.05
  )

print(rp.importance(LFS2015.rfPtunedfeng, scale = FALSE))

## Effect Size
rf.effectSize(LFS2015.rfPtuned, y = LFS2015$C07_AGE, pred.data = LFS2015, x.var = C07_AGE)

```

## Model Export
```{r}
saveRDS(LFS2015.rfPtuned, "./LFS2015_rfP_Model.rds")

```


