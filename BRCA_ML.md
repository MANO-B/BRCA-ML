---
title: "v3.4"
author: "Steven N. Hart, Ph.D"
date: '9/30/2019'
output:
  html_document:
    keep_md: true
---




```r
all_annotations = read.csv('sources/v3.4a.out',header=TRUE, stringsAsFactors = FALSE, sep="\t")

#Get BayesDel
bd = read.csv('sources/BayesDel_nsfp33a_noAF.tsv',header=TRUE, sep="\t", skip = 132)
bd$ID=NULL

names(bd) =c("Chr", "Pos", "Ref", "Alt", "BayesDel")
all_annotations = merge(all_annotations, bd, all.x=TRUE)
rm(bd)

#Get AlignGVGD
agvgd = read.csv('sources/alignGvgd.tsv', header=TRUE, sep="\t", stringsAsFactors = FALSE)
names(agvgd) =c("Chr", "Pos", "Ref", "Alt", "AlignGVGDPrior")
all_annotations = merge(all_annotations, agvgd, all.x=TRUE)
rm(agvgd)


for (j in names(all_annotations)[6:ncol(all_annotations)]){
    if (typeof(all_annotations[,j]) == 'character'){
      #all_annotations[,j] = vapply(strsplit(all_annotations[,j],"\\|"), `[`, 1, FUN.VALUE=character(1)) %>% as.numeric()
      # Get the maximum score
      all_annotations[,j] = suppressWarnings(sapply(strsplit(all_annotations[,j], "\\|"), function(x) (max(x,na.rm = TRUE))) %>% as.numeric())
      
    }
}

all_annotations = all_annotations %>%
  filter(AApos>0)

keep = names(all_annotations)[-grep("Rankscore",names(all_annotations))]
all_annotations = all_annotations[,keep]

all_annotations = all_annotations %>%
  mutate(Variant = paste(Chr,Pos,Ref,Alt,sep='_'))
rm(keep, j)
#glimpse(all_annotations)
```

Plot all predictions


Set some useful functions

```r
# Create a filter to remove nything not in the BRCT/RING domains of BRCA1 and the DNA binding domain of BRCA2
filter_to_target_regions = function(d, variant_search = FALSE, l = links)
{
  RING_prot = c(1, 109)
  BRCT_prot = c(1642, 1855)
  DNABD_prot = c(2479, 3192)
  
  if(variant_search == TRUE){
    d$Variant = as.character(d$Variant)
    d = merge(d, l)
  }
  
  #Filter BRCA1 variants
  d$X.CHROM = as.numeric(d$X.CHROM)
  d$AApos   = as.numeric(d$AApos)
  BRCA1_keep_idx = which(
    d$X.CHROM ==  17 & (
      dplyr::between(d$AApos, RING_prot[1], RING_prot[2]) | dplyr::between(d$AApos,BRCT_prot[1], BRCT_prot[2]) 
      )
  )
  
  BRCA2_keep_idx =  which(d$X.CHROM ==  13 & dplyr::between(d$AApos, DNABD_prot[1], DNABD_prot[2]))
  d = d[c(BRCA1_keep_idx,BRCA2_keep_idx),]
  d = distinct(d, Variant, .keep_all = TRUE)
  rm(RING_prot, BRCT_prot, DNABD_prot, BRCA1_keep_idx, BRCA2_keep_idx)
  return(d)
}


### Define the MCC function
mcc <- function(predicted, actual)
{
  TP <- sum(actual ==   "Damaging" & predicted ==   "Damaging")
  TN <- sum(actual ==   "Neutral"     & predicted ==   "Neutral")
  FP <- sum(actual ==   "Neutral"     & predicted ==   "Damaging")
  FN <- sum(actual ==   "Damaging" & predicted ==   "Neutral")
  mcc <- ((TP*TN) - (FP*FN)) / sqrt((as.numeric(TP + FP)*as.numeric(TP + FN)*as.numeric(TN + FP)*as.numeric(TN + FN)))
  rm(TP, TN, FP, FN)
  return(round(mcc, digits = 3))
}

  

links = all_annotations %>% 
  mutate(Variant = paste(Chr,Pos, Ref, Alt, sep = "_")) %>%
  mutate(X.CHROM=Chr, POS=Pos, REF=Ref, ALT=Alt) %>%
  select(X.CHROM, AApos, Variant )
```



Load all functional data

```r
#Read in our data
neutral_list = read.csv(paste0(wd, "/sources/neutral.list"),sep = "\t",header = FALSE, stringsAsFactors = FALSE)
damaging_list = read.csv(paste0(wd, "/sources/deleterious.list"),sep = "\t",header = FALSE, stringsAsFactors = FALSE)

Hart_df = rbind(neutral_list,damaging_list)
names(Hart_df) = c("Gene","Variant","Call", "Source")

Hart_df = filter_to_target_regions(Hart_df, variant_search = TRUE)

rm(neutral_list, damaging_list)
```


```r
#######################################################################################################################################
#Read in Starita
#Get data
Starita = read.csv(paste0(wd, '/sources/Starita.tsv'),header = TRUE,
                   sep = "\t", stringsAsFactors = FALSE, 
                   na.strings = c('.'))

HDR_damaging_max = 0.33
HDR_neutral_min = 0.77

Starita = Starita %>%
  filter(HDR <=   HDR_damaging_max | HDR >=  HDR_neutral_min)

Starita_df = data.frame(
  Variant = as.character(Starita$cpra),
  Gene = 'BRCA1',
  Call = ifelse(Starita$HDR <=  HDR_damaging_max, "Damaging", 'Neutral' )
)
Starita_df = na.omit(Starita_df)

Starita_df = Starita_df %>%
  select(Gene, Variant, Call) %>%
  mutate(Source = "Starita et al., 2015")

Starita_df    = filter_to_target_regions(Starita_df, variant_search = TRUE)

rm(Starita, HDR_damaging_max, HDR_neutral_min)
```



```r
#######################################################################################################################################
#Read in Fernandes
#Get data
Fernandes = read.csv(paste0(wd, "/sources/Fenandez.tsv"), header = TRUE, 
                     sep = "\t", stringsAsFactors = FALSE)
#Remove WT and splic variants
omit = Fernandes$variant[grep('del|\\+|WT|\\/|ins',Fernandes$variant)]

Fernandes = Fernandes %>%
  filter(!variant %in% omit) %>%
  filter(IARC %in% c(0,1,4,5)) %>%
  mutate(Call = ifelse(IARC > 2, "Damaging", "Neutral"), Gene = 'BRCA1') %>%
  select(Gene, Variant, Call) %>%
  mutate(Source = "Fernandes et al., 2019")

Fernandes_df    = filter_to_target_regions(Fernandes, variant_search = TRUE)

rm(Fernandes, omit)
```


```r
#######################################################################################################################################
#Read in Findlay
#Get data
Findaly = read.csv(paste0(wd, '/sources/Findlay.tsv'), header = TRUE,sep = "\t", 
                   stringsAsFactors = FALSE, na.strings = c('','NA','.'))
# Exclude splicing (1/28)
Findaly = Findaly %>%
  filter(Type %in% c('missense_variant'), Call %in% c('Deleterious','Neutral'))

Fin_df = data.frame(
  Gene = Findaly$Gene,
  Variant = Findaly$CPRA,
  Call = Findaly$func.class,
  Score = Findaly$function.score.mean,
  Seen = Findaly$AlreadySeen)

Findlay_df = Fin_df %>%
  filter(Seen ==  0)

Findlay_df$Call =  ifelse(Findlay_df$Call ==  'FUNC', 'Neutral', 'Damaging')

Findlay_df = Findlay_df %>%
  select(Gene, Variant, Call) %>%
  mutate(Source = "Findlay et al., 2018")

Findlay_df = filter_to_target_regions(Findlay_df, variant_search = TRUE)

rm(Fin_df, Findaly)
```


```r
#######################################################################################################################################
# Combine experimental datasets
experimental_data = rbind(Hart_df, Fernandes_df, Starita_df, Findlay_df)
d = as.data.frame(table(experimental_data$Gene, experimental_data$Call, experimental_data$Source))
names(d) = c('Gene', 'Call', 'Source', 'RawCount')

experimental_data = distinct(experimental_data, Gene, Variant, .keep_all = TRUE)
e = as.data.frame(table(experimental_data$Gene, experimental_data$Call, experimental_data$Source))
names(e) = c('Gene', 'Call', 'Source', 'FilteredCount')

Counts = cbind(d, e$FilteredCount)
names(Counts)[length(names(Counts))] = 'FilteredCount'


rm(Hart_df, Fernandes_df, Starita_df, Findlay_df)
table(experimental_data$Source, experimental_data$Call,experimental_data$Gene)
```

```
## , ,  = BRCA1
## 
##                         
##                          Damaging Neutral
##   Fernandes et al., 2019        0      16
##   Findlay et al., 2018         55     259
##   Guidugli et al., 2014         0       0
##   Guidugli et al., 2018         0       0
##   Hart et al, 2018              1       2
##   Lee et al., 2010             14      13
##   Lindor et al., 2012           0       0
##   Starita et al., 2015         11    1044
##   This paper                    0       0
##   Woods et al., 2016            3      36
## 
## , ,  = BRCA2
## 
##                         
##                          Damaging Neutral
##   Fernandes et al., 2019        0       0
##   Findlay et al., 2018          0       0
##   Guidugli et al., 2014         4       0
##   Guidugli et al., 2018        38      48
##   Hart et al, 2018             16      47
##   Lee et al., 2010              0       0
##   Lindor et al., 2012           9      18
##   Starita et al., 2015          0       0
##   This paper                    7      15
##   Woods et al., 2016            0       0
```

```r
table(experimental_data$Call,experimental_data$Gene)
```

```
##           
##            BRCA1 BRCA2
##   Damaging    84    74
##   Neutral   1370   128
```

More custom functions


```r
prepare_inputs = function(data = experimental_data, 
                          annotations = all_annotations, 
                          gene = 'BRCA1', 
                          sources = NULL){

  s = NULL

  if ('Hart' %in% sources){s = c("This paper", "Guidugli et al., 2018", 
                                 "Hart et al, 2018", "Lee et al., 2010",
                                 "Lindor et al., 2012", "Woods et al., 2016", 
                                  "Guidugli et al., 2014")}
  if ('Fernandes' %in% sources){s = c(s, 'Fernandes et al., 2019')}
  if ('Starita' %in% sources){s = c(s, 'Starita et al., 2015')}
  if ('Findlay' %in% sources){s = c(s, 'Findlay et al., 2018')}
  
  # if sources = PATH, then take all pathogenic variants from all studies and benign from our study
  if(is.null(s)){
    data = data %>%
      filter(Gene == gene)
  }else{
    # Filter to known results for a specific gene
    data = data %>%
      filter(Source %in% s, Gene == gene)
  }
  
  data = distinct(data, Gene, Variant, .keep_all = TRUE)
  
  #Merge to get annotations
  data = merge(data,
    annotations,
    all.x = TRUE,
    by = 'Variant')
  
  # Sort list randomly
  data = sample(data)
  if ('Gene.x' %in% names(data)){ data = data %>% mutate(Gene = Gene.x, Gene.x=NULL, Gene.y=NULL)}
  if ('AApos.x' %in% names(data)){ data = data %>% mutate(AApos = AApos.x, AApos.x=NULL, AApos.y=NULL)}
  if ('X.CHROM.x' %in% names(data)){ data = data %>% mutate(X.CHROM = X.CHROM.x, X.CHROM.x=NULL, X.CHROM.y=NULL)}
  return(data)
}

split_data = function(known_scores, validation_fraction = 0.2 ){
  set.seed(999)
  known_scores$Call = as.factor(known_scores$Call)
  num_predictions = nrow(known_scores)

  #Create training testing and validation
  Neutral_idx = which(known_scores$Call == 'Neutral')
  Damaging_idx = which(known_scores$Call == 'Damaging')
  
  n_train_idx = sample(Neutral_idx, size = length(Neutral_idx) * 0.8)
  d_train_idx = sample(Damaging_idx, size = length(Damaging_idx) * 0.8)

  # Subset to training and validation
  idx = c(n_train_idx, d_train_idx)
  #Shuffle randomly
  idx = sample(idx)
  
  train = known_scores[idx,]
  test = known_scores[-idx,]
  splits = NULL
  splits$test = NULL
  splits$train = NULL
  splits$train = train
  splits$test = test
  return(splits)
}



k_fold_split_data = function(known_scores, k=10 ){
  set.seed(999)
  known_scores$Call = as.factor(known_scores$Call)
  num_predictions = nrow(known_scores)
  
  #Create training testing and validation
  Neutral_idx = which(known_scores$Call == 'Neutral')
  Neutral_fold = createFolds(Neutral_idx, k = k, list = FALSE)
  Damaging_idx = which(known_scores$Call == 'Damaging')
  Damaging_fold = createFolds(Damaging_idx, k = k, list = FALSE)
  
  
  # Subset to training and validation
  idx_f = c(Neutral_fold, Damaging_fold)
  
  known_scores$Fold = idx_f
  
  splits = sample(known_scores)
  return(splits)
}


train_model = function(splits, k_=NULL, Gene = 'BRCA1'){
  knitr::opts_chunk$set(echo=FALSE)

  h2o.init(min_mem_size = "16g", max_mem_size = "32g", nthreads = 20)
  
  train = as.h2o(splits$train)
  
  all_h2o_models =  h2o.automl(x = COLUMNS, y = "Call",
                  training_frame = train,
                  keep_cross_validation_predictions = TRUE,
                  nfolds = 5,
                  max_runtime_secs = 3600,
                  sort_metric = 'mean_per_class_error',
                  stopping_metric = 'mean_per_class_error',
                  seed = 100)
  knitr::opts_chunk$set(echo=TRUE)
  
  #Save the model
  best_model = h2o.saveModel(object = all_h2o_models@leader,path = "results/",force = TRUE)
  best_model
  all_h2o_models

  # Load model
  model_out = h2o.loadModel(best_model)
  model_vi  = as.data.frame(all_h2o_models@leader@model$variable_importances)
  threshold = h2o.find_threshold_by_max_metric(h2o.performance(model_out), 'absolute_mcc')
  my_list   = list('all_h2o_models' = all_h2o_models, 'best_model' = best_model, 
                   'model' = model_out, 'vi' =  model_vi, 'threshold' = threshold)
  return(my_list)
  }


test_model = function(splits, model, threshold = NULL){

  knitr::opts_chunk$set(echo=FALSE)
  h2o.init(min_mem_size = "16g", max_mem_size = "32g", nthreads = 1)

  test = as.h2o(splits$test)
  
  pred = h2o.predict(model, 
                     newdata = test, 
                     threshold = threshold)
  knitr::opts_chunk$set(echo=TRUE)

  pred = as.data.frame(pred)

  #Edit prediction based on previous threshold
  final_result = cbind(splits$test, pred)

  cm = confusionMatrix(final_result$predict,final_result$Call, positive = 'Damaging')
  m = mcc(final_result$predict,final_result$Call)
  my_list = list('result' = final_result, 'cm' = cm, 'MCC' = m)
  return(my_list)
}

apply_model_to_gene = function(model = model, threshold = threshold,
                               gene = 'BRCA1', annotations = all_annotations){
  #Run model on each gene
  tmp = annotations %>%
    filter(Genename == gene)
  pred = h2o.predict(model, newdata = as.h2o(tmp), threshold = threshold)
  pred = as.data.frame(pred)
  tmp$Prediction = pred$predict
  tmp$`BRCA-ML` = pred$Damaging
  tmp$Neut = pred$Neutral
  return(tmp)
}


plot_full_gene = function(data, title, splits = splits, model_name='BRCA-ML', t_=NULL){
  
  #Pull out training, validation, & testing variants to plot
  
  if(length(splits$train)){
    variants = splits$train %>%
      select(AApos, Call)
    
  
    variants = rbind(variants,splits$test %>%
      select(AApos, Call)
    )  
  }else{
    variants = NULL
  }
  

  tmp = data[,c("Genename","BRCA-ML","AApos","Prediction")]
  tmp$Model = model_name
  names(tmp) = c("Gene","Score","AApos","Prediction","Model")
  tmp$Prediction = as.character(tmp$Prediction)
  if (!is.null(t_)){tmp = tmp %>% mutate(Prediction = ifelse(Score > t_, 'Damaging', 'Neutral'))}
  p = tmp %>%
    ggplot(aes(x = AApos,y = Score, colour = Prediction)) +
    geom_point(size = 0.2) +
    #facet_grid(Model ~ Gene, scales = 'free') +
    geom_smooth(aes(x = AApos,y = Score), inherit.aes = FALSE) +
    ggtitle(title) +
    theme(legend.position="none") 
  if(length(splits$train)){
    p = p + geom_rug(
        aes(x=AApos, colour=Call, alpha = 1/2), 
        inherit.aes = FALSE, 
        sides = "b", 
        data = variants)
  }
  return(p)
}
```

Define columns to TRAIN on

```r
models = read.csv(paste0(wd, '/sources/base_thresholds'), header = FALSE, sep="\t", stringsAsFactors = FALSE)
COLUMNS = models$V1
```

Run the training

```r
#BRCA1
b1_input = prepare_inputs(gene = 'BRCA1')
b1_DATA_SPLITS = split_data(b1_input)
b1_TRAINED_DATA = train_model(b1_DATA_SPLITS)
```

```
## 
## H2O is not running yet, starting it now...
## 
## Note:  In case of errors look at the following log files:
##     /tmp/RtmpnTasxs/h2o_m087494_started_from_r.out
##     /tmp/RtmpnTasxs/h2o_m087494_started_from_r.err
## 
## 
## Starting H2O JVM and connecting: . Connection successful!
## 
## R is connected to the H2O cluster: 
##     H2O cluster uptime:         1 seconds 385 milliseconds 
##     H2O cluster timezone:       CST6CDT 
##     H2O data parsing timezone:  UTC 
##     H2O cluster version:        3.22.1.1 
##     H2O cluster version age:    9 months and 2 days !!! 
##     H2O cluster name:           H2O_started_from_R_m087494_whe707 
##     H2O cluster total nodes:    1 
##     H2O cluster total memory:   28.44 GB 
##     H2O cluster total cores:    80 
##     H2O cluster allowed cores:  20 
##     H2O cluster healthy:        TRUE 
##     H2O Connection ip:          localhost 
##     H2O Connection port:        54321 
##     H2O Connection proxy:       NA 
##     H2O Internal Security:      FALSE 
##     H2O API Extensions:         XGBoost, Algos, AutoML, Core V3, Core V4 
##     R Version:                  R version 3.5.2 (2018-12-20) 
## 
## 
## 
```

```r
# GBM_grid_1_AutoML_20190920_074644_model_55
b1_TESTED_DATA = test_model(b1_DATA_SPLITS, b1_TRAINED_DATA$model, threshold = b1_TRAINED_DATA$threshold)
```

```
##  Connection successful!
## 
## R is connected to the H2O cluster: 
##     H2O cluster uptime:         53 minutes 35 seconds 
##     H2O cluster timezone:       CST6CDT 
##     H2O data parsing timezone:  UTC 
##     H2O cluster version:        3.22.1.1 
##     H2O cluster version age:    9 months and 2 days !!! 
##     H2O cluster name:           H2O_started_from_R_m087494_whe707 
##     H2O cluster total nodes:    1 
##     H2O cluster total memory:   27.84 GB 
##     H2O cluster total cores:    80 
##     H2O cluster allowed cores:  20 
##     H2O cluster healthy:        TRUE 
##     H2O Connection ip:          localhost 
##     H2O Connection port:        54321 
##     H2O Connection proxy:       NA 
##     H2O Internal Security:      FALSE 
##     H2O API Extensions:         XGBoost, Algos, AutoML, Core V3, Core V4 
##     R Version:                  R version 3.5.2 (2018-12-20) 
## 
## 
## 
```

```r
b1_PREDICTIONS = apply_model_to_gene(model = b1_TRAINED_DATA$model, 
                                          gene = 'BRCA1', 
                                          threshold = b1_TRAINED_DATA$threshold)
```

```
## 
## 
```

```r
b1_PLOTS = plot_full_gene(b1_PREDICTIONS, 
                               'BRCA1', 
                               splits = b1_DATA_SPLITS, 
                               t_ = b1_TRAINED_DATA$threshold )

#BRCA2
b2_input = prepare_inputs(gene = 'BRCA2')
b2_DATA_SPLITS = split_data(b2_input)
b2_TRAINED_DATA = train_model(b2_DATA_SPLITS)
```

```
##  Connection successful!
## 
## R is connected to the H2O cluster: 
##     H2O cluster uptime:         53 minutes 40 seconds 
##     H2O cluster timezone:       CST6CDT 
##     H2O data parsing timezone:  UTC 
##     H2O cluster version:        3.22.1.1 
##     H2O cluster version age:    9 months and 2 days !!! 
##     H2O cluster name:           H2O_started_from_R_m087494_whe707 
##     H2O cluster total nodes:    1 
##     H2O cluster total memory:   27.84 GB 
##     H2O cluster total cores:    80 
##     H2O cluster allowed cores:  20 
##     H2O cluster healthy:        TRUE 
##     H2O Connection ip:          localhost 
##     H2O Connection port:        54321 
##     H2O Connection proxy:       NA 
##     H2O Internal Security:      FALSE 
##     H2O API Extensions:         XGBoost, Algos, AutoML, Core V3, Core V4 
##     R Version:                  R version 3.5.2 (2018-12-20) 
## 
## 
## 
```

```r
# XGBoost_grid_1_AutoML_20190920_084013_model_2
b2_TESTED_DATA = test_model(b2_DATA_SPLITS, b2_TRAINED_DATA$model, threshold = b2_TRAINED_DATA$threshold)
```

```
##  Connection successful!
## 
## R is connected to the H2O cluster: 
##     H2O cluster uptime:         1 hours 34 minutes 
##     H2O cluster timezone:       CST6CDT 
##     H2O data parsing timezone:  UTC 
##     H2O cluster version:        3.22.1.1 
##     H2O cluster version age:    9 months and 2 days !!! 
##     H2O cluster name:           H2O_started_from_R_m087494_whe707 
##     H2O cluster total nodes:    1 
##     H2O cluster total memory:   27.69 GB 
##     H2O cluster total cores:    80 
##     H2O cluster allowed cores:  20 
##     H2O cluster healthy:        TRUE 
##     H2O Connection ip:          localhost 
##     H2O Connection port:        54321 
##     H2O Connection proxy:       NA 
##     H2O Internal Security:      FALSE 
##     H2O API Extensions:         XGBoost, Algos, AutoML, Core V3, Core V4 
##     R Version:                  R version 3.5.2 (2018-12-20) 
## 
## 
## 
```

```r
b2_PREDICTIONS = apply_model_to_gene(model = b2_TRAINED_DATA$model, 
                                          gene = 'BRCA2', 
```

```
## 

