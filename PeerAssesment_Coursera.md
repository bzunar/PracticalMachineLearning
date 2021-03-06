---
title: "Machine Learning for Correct Weight Lifting"
author: "Bojan Zunar"
date: '4 August 2018 '
output: 
    html_document:
      toc: true
      toc_float: true
      number_sections: true
      code_folding: hide
      fig.cap: ""
---



# Summary

Devices like *Jawbone Up*, *Nike FuelBand*, and *Fitbit* inexpensively collect data about a personal activity. These data can be used to improve user health and find patterns in behaviour. Today, they are regularly used to quantify how much particular activity users do, but rarely how well they do it. As a proof of concept, we develop a prediction model that correctly labels the type of weightlifting in more than 99% of cases while avoiding user-specific bias. 

# Dataset processing

Dataset measures six users which lifted barbell in one correct way (level A) and four incorrect ways (levels B-E). Data was captured from accelerometers on the belt, forearm, arm, and dumbbell. Extraction of 153 features is detailed in Velloso et al. (2013) [2].

At the beginning of the assignment, the original dataset was already partitioned into the `training` set (19,622 observations) and the `test` set (20 observations). We used 75% of the training set for the `model training` (14,715 observations) and the remaining 25% for `cross-validation` (4,904 observations).


```r
url_train = c("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv")
dest_train = c("training.csv")
download.file(url_train, dest_train)
url_test = c("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv")
dest_test = c("testing.csv")
download.file(url_test, dest_test)
rm(url_train, url_test, dest_train, dest_test)

train <- read.csv("training.csv",  na.strings = c("NA", "", "#DIV/0!"))
test <- read.csv("testing.csv",  na.strings = c("NA", "", "#DIV/0!"))

set.seed(12345)
inTrain <- createDataPartition(train$classe, p = 3/4)[[1]]
training <- train[ inTrain,]
crossValidation <- train[-inTrain,]
rm(train)
```

Preliminary analysis showed that NA values represented over 60% of the `model training` dataset as most of the columns (features) were empty.


```r
library(dplyr); library(reshape2); library(tibble); library(ggplot2); library(ggsci); library(GGally); library(ggrepel); library(caret); library(lemon); library(gridExtra)

## Calculate proportion of NA values in the dataset
training %>%
    is.na %>% table() %>% prop.table()
```

```
## .
##     FALSE      TRUE 
## 0.3870261 0.6129739
```

# Exploratory data analysis

Exploration of a subset of features showed their complex dependence on the class (the way in which the weight was lifted) and user which performed the activity. As the goal was to develop a model which recognises types of weightlifting based on accelerometer-measured features, we omitted the first seven columns of the dataset, i.e. those that identify users and timestamps. We also excluded features with near zero-variance (function `caret::nearZeroVar`), thus continuing with only 51 features. Using them, we performed the last part of the exploratory analysis, principal component analysis (PCA).

First seven principal components explained 91.13% of the variance in the dataset.


```r
## Perform PCA and calculate total variance explained
training %>% 
    .[, -c(1:7, 160)] %>%
    replace(., is.na(.), 0) %>%
    .[, -nearZeroVar(.)] %>%
    prcomp() %>% 
    summary() %>% 
    .$importance %>% 
    .[2,] %>%
    cumsum() %>%
    .[1:9]
```

```
##     PC1     PC2     PC3     PC4     PC5     PC6     PC7     PC8     PC9 
## 0.26272 0.47229 0.63613 0.74242 0.83478 0.88186 0.91133 0.93352 0.95176
```

Among them, no combination of two principal components separated types of weightlifting.


```r
## Function that plots only lower triangle in ggpairs (thanks to Richard Telford)
## https://stackoverflow.com/questions/42654928/how-to-show-only-the-lower-triangle-in-ggpairs

gpairs_lower <- function(g){
    g$plots <- g$plots[-(1:g$nrow)]
    g$yAxisLabels <- g$yAxisLabels[-1]
    g$nrow <- g$nrow -1
    
    g$plots <- g$plots[-(seq(g$ncol, length(g$plots), by = g$ncol))]
    g$xAxisLabels <- g$xAxisLabels[-g$ncol]
    g$ncol <- g$ncol - 1
    
    g
}
```


```r
## Perform PCA and plot all combinations of two principal components (by types of lifting)

gg <- training %>% 
    .[, -c(1:7, 160)] %>%
    replace(., is.na(.), 0) %>%
    .[, -nearZeroVar(.)] %>%
    prcomp() %>% 
    .$x %>% 
    as.data.frame %>% 
    select(1:7) %>%
    mutate(user = training$user_name, classe = training$classe) %>%
    ggpairs(aes(colour = classe), columns = 1:7, 
            title = "Combinations of two principal components: colored by class", 
                 lower = list(continuous = wrap("points", alpha = 0.3, size = 0.1), 
                              combo = wrap("points", alpha = 0.3, size = 0.1), 
                              discrete = wrap("facetbar", alpha = 0.3, size = 0.1), 
                              na = "na"),
                 upper = list(continuous = "blank", 
                              combo = "box_no_facet", 
                              discrete = "facetbar", 
                              na = "na"), 
                 diag = list(continuous = wrap("blank", alpha = 0.3), 
                             discrete = "barDiag", 
                             na = "naDiag"),
                 legend = 8) +
    theme_bw() +
    theme(legend.position = "bottom")

## Change the color of points as well as the alpha and size of markers in the legend
for(i in 1:gg$nrow) {
    for(j in 1:gg$ncol){
        gg[i,j] <- gg[i,j] + 
            scale_color_npg() +
            scale_fill_npg() +
            guides(colour = guide_legend(override.aes = 
                                             list(alpha = 1, size = 5), title = "Class")) +
            scale_y_continuous(breaks = c(-500, 500)) +
            scale_x_continuous(breaks = c(-1000, 0, 1000))
    }
}
gpairs_lower(gg)
```

<img src="figure/unnamed-chunk-5-1.png" title="plot of chunk unnamed-chunk-5" alt="plot of chunk unnamed-chunk-5" width="100%" style="display: block; margin: auto;" />

# Building prediction models

Fifteen prediction models were built with `caret::train` function, each using one algorithm (tp = number of tuning parameters algorithm allows):

* Bagged CART (0 tp, `method = 'treebag'`)
* Classification and Regression Trees: CART (1 tp, `method = 'rpart'`)
* k-Nearest Neighbors (1 tp, `method = 'knn'`)
* Linear Discriminant Analysis (0 tp, `method = 'lda'`)
* Maximum Uncertainty Linear Discriminant Analysis (0 tp, `method = 'Mlda'`)
* Naive Bayes (3 tp, `method = 'nb'`)
* Quadratic Discriminant Analysis (0 tp, `method = 'qda'`)
* Random Forest (1 tp, `method = 'rf'`)
* Robust Quadratic Discriminant Analysis (0 tp, `method = 'QdaCov'`)
* Robust SIMCA (0 tp, `method = 'RSimca'`)
* SIMCA (0 tp, `method = 'CSimca'`)
* Single C5.0 Ruleset (0 tp, `method = 'C5.0Rules'`)
* Single C5.0 Tree (0 tp, `method = 'C5.0Tree'`)
* Single Rule Classification (0 tp, `method = 'OneR'`)
* Support Vector Machines with Radial Basis Function Kernel (2 tp, `method = 'svmRadial'`)

Models were optimised through four resampling iterations, each consisting of 5-fold cross-validation, as specified by the `caret::trainControl` function. The function `caret::train` built the models, selecting the most accurate one while checking 50 levels for each tuning parameter. We estimated the sample error with `predict` and `caret::confusionMatrix` function on the `crossValidation` dataset, which was not part of the `model training` dataset.


```r
## Reshaping the `model training` dataset for `caret::train` function
training.ml <- training %>% 
    .[, -c(1:7)] %>%
    replace(., is.na(.), 0) %>%
    .[, -nearZeroVar(.)]

## Specifying the `caret::trainControl` function
control <- trainControl(method = "repeatedcv",
                        number = 4,
                        repeats = 5,
                        summaryFunction = multiClassSummary,
                        classProbs = TRUE,
                        verboseIter = TRUE)

## Example of a code used to build and cross-validate a prediction model
set.seed(56789)
sys.time.qda <- system.time(
    fit.qda <- train(classe ~ .,
                     data = training.ml,
                     method = "qda",
                     tuneLength = 50, 
                     metric = "Accuracy",
                     trControl = control)
)
pred.qda <- predict(fit.qda, crossValidation)
confusion.qda <- confusionMatrix(pred.qda, crossValidation$classe)
```

# Testing prediction models

Models exceedingly differed in accuracy and in the time needed to build them. Those with more tuning parameters took longer to build, but their complexity did not guarantee accuracy. Random forest `rf` produced the most accurate model (99.47% accuracy), but it also took 9.12 h to generate it. Algorithms that offer interesting trade-offs between the time of model building and accuracy are bagged CART (`treebag`, accuracy = 98.94, time = 3.42 min) and quadratic discriminant analysis (`qda`,  accuracy = 89.03, time = 6.84 s).


```r
## Function that extracts needed data
ta <- function(...) {
    method <- list(...)
    m <- data.frame()
    for (i in 1:length(method)) {
        m[i, 1] <- method[i]
        m[i, 2] <- eval(parse(text = paste0("sys.time.", method[i], "[[3]]")))
        m[i, 3] <- eval(parse(text = paste0("confusion.", method[i], "$overall[[1]] *100")))
    }
    colnames(m) <- c("method", "elapsedTime", "accuracy")
    m
}

## Plot model's accuracy vs time to build it
ta("knn", "lda", "nb", "rf", "rpart", 
           "treebag", "svmRadial", "qda", "QdaCov", "C5.0Tree", 
           "C5.0Rules", "Mlda", "CSimca", "OneR", "RSimca") %>% 
    cbind(tp = c(1, 0, 3, 1, 1, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0)) %>% 
    ggplot(aes(x = log10(elapsedTime), y = accuracy, color = factor(tp))) + 
    geom_rug() + 
    geom_text_repel(aes(label = method),
                    box.padding   = 0.3,
                    nudge_x = 0.15) +
    geom_point(size = 2.5) + 
    theme_bw() + 
    guides(colour = guide_legend(override.aes = list(linetype = 0, size = 5), 
                                 title = "tuning parameters")) +
    theme(legend.position = "bottom") +
    labs(x = expression(Log[10]*"(time to build a model) [s]"), y = "Accuracy [%]",
         title = "Model building vs accuracy") +
    scale_color_npg() +
    scale_fill_npg() +
    ylim(30, 100)
```

<img src="figure/unnamed-chunk-7-1.png" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" width="80%" style="display: block; margin: auto;" />

We plotted confusion matrices and ROC curves to see if models make systematic errors in predicting the type of weightlifting. Model's accuracy determined their order.


```r
## Function that calculates and plots confusion matrices and ROC curves
## ggML()    # ggMachineLearning = ggplot of (multiple) ROC curves and confusion matrices
### model = calculated prediction model
### newdata = testing dataset
### truth = column of the testing dataset with true labels
### method = caret's train method (as would be specified for the caret::train())
### fullName = machine learning algorithm's full names
### legend = position of the legend

ggML <- function(model, newdata, truth,
                      method = "Method", fullName = "Algorithm's name", legend = "hidden") {
    require(dplyr)
    require(dummies)
    require(ggplot2)
    require(multiROC)
    require(caret)
    
    # A way to force caret to load packages needed for predict function
    data(iris)
    try(train(Species ~ .,
              data = iris,
              method = method,
              tuneLength = 1, 
              metric = "Accuracy",
              trControl = trainControl(number = 1,
                                       summaryFunction = multiClassSummary,
                                       classProbs = TRUE)))
    
    # For some algorithm's ROC curve cannot be easily calculated: this code creates empty graph
    mock <- data.frame(Specificity = c(0, 0, 0, 0, 0), 
                       Sensitivity = c(1, 1, 1, 1, 1), 
                       Group = c("A", "B", "C", "D", "E"), 
                       AUC = c(0.9, 0.9, 0.9, 0.9, 0.9), 
                       Method = c("A", "A", "A", "A", "A"))
    
    ggROC <- mock %>% 
        ggplot(aes(x = Specificity, y = Sensitivity, color = Group)) +
        geom_path(aes(alpha = 0.3), size = 1.5) +
        geom_segment(aes(x = 0, y = 0, xend = 1, yend = 1), 
                     colour = 'grey', linetype = 'dotdash') +
        scale_colour_manual(values = c("#E41A1C", "#377EB8", "#4DAF4A", "#984EA3",
                                       "#FF7F00", "#FFFF33", "#A65628", "#F781BF")) +
        guides(alpha = FALSE, 
               colour = guide_legend(override.aes = list(alpha = 0.7))) +
        labs(x = "1 - Specificity", y = "Sensitivity",
             title = method, subtitle = fullName) +
        theme_bw() +
        theme(plot.title = element_text(face = "bold"))

    # Calculate and crete ggROC curve
    try(pred <- model %>%
            predict(newdata = newdata, type = 'prob') %>% 
            data.frame() %>%
            setNames(paste(names(.), "_pred_A")))
    
    true <- truth %>% 
        dummies::dummy(sep = ".") %>% 
        data.frame() %>%
        setNames(gsub(".*?\\.", "", names(.))) %>%
        setNames(paste(names(.), "_true"))
    
    try(ggROC <- cbind(pred, true) %>%
            multi_roc(., force_diag = TRUE) %>%
            plot_roc_data() %>%
            filter(!Group %in% c("Micro", "Macro")) %>%
            ggplot(aes(x = 1 - Specificity, y = Sensitivity, color = Group)) +
            geom_path(aes(alpha = 0.3), size = 1.5) +
            geom_segment(aes(x = 0, y = 0, xend = 1, yend = 1), 
                         colour = 'grey', linetype = 'dotdash') +
            scale_color_npg() +
            guides(alpha = FALSE, 
                   colour = guide_legend(override.aes = list(alpha = 0.7))) +
            labs(title = method, subtitle = fullName) +
            theme_bw() +
            theme(plot.title = element_text(face = "bold")))
    
    # Define colors for ggconfusion matrix (from RColorBrewer, brewer.pal(9, "OrRd") %>% .[1:8])
    brew.orrd <- c("#FFF7EC", "#FEE8C8", "#FDD49E", "#FDBB84", 
                   "#FC8D59", "#EF6548", "#D7301F", "#B30000") %>% colorRampPalette
    brew.orrd <- brew.orrd(100)
    brew.orrd[1:20] <- "#FFFFFF"
    
    # Calculte confusion matrix
    try(confusion <- model %>%
        predict(newdata = newdata) %>%
        confusionMatrix(truth))
    
    # Crete ggconfusion matrix
    ggconfusion <- confusion$table %>%
        data.frame() %>% 
        rename(prediction = Prediction, reference = Reference, number = Freq) %>%
        group_by(reference) %>% 
        mutate(total = sum(number), fraction = number/total) %>%
        ggplot(aes(x = prediction, y = reorder(reference, desc(reference)), fill = fraction)) +
        geom_tile() +
        geom_text(aes(label = number)) +
        scale_fill_gradientn(colors = brew.orrd[20:90]) +
        scale_x_discrete(position = "top") + 
        geom_tile(color = "black", fill = "black", alpha = 0) +
        labs(x = "Prediction", y = "Reference", fill = "Fraction per row", title = method,
             subtitle = paste0(fullName, "\nAccuracy: ", percent_format()(confusion$overall[1]),
                               " [", percent_format()(confusion$overall[3]), "-",
                               percent_format()(confusion$overall[4]), "]")) +
        theme_minimal() +
        theme(legend.position = legend, plot.title = element_text(face = "bold"))
    list(ggROC, ggconfusion)
}
```


```r
## Call ggML() for each model
g1 <- ggML(fit.knn, crossValidation, crossValidation$classe, "knn", "k-Nearest Neighbours")
g2 <- ggML(fit.lda, crossValidation, crossValidation$classe, "lda", "Linear Discriminant Analysis")
g3 <- ggML(fit.nb, crossValidation, crossValidation$classe, "nb", "Naive Bayes")
g4 <- ggML(fit.rf, crossValidation, crossValidation$classe, "rf", "Random Forest")
g5 <- ggML(fit.rpart, crossValidation, crossValidation$classe, "rpart", "CART: Recursive Partitioning")
g6 <- ggML(fit.treebag, crossValidation, crossValidation$classe, "treebag", "Bagged CART")
g7 <- ggML(fit.svmRadial, crossValidation, crossValidation$classe, "svmRadial", "Support Vector Machines")
g9 <- ggML(fit.qda, crossValidation, crossValidation$classe, "qda", "Quadratic Discriminant Analysis")
g10 <- ggML(fit.QdaCov, crossValidation, crossValidation$classe, "QdaCov", "Robust QDA")
g12 <- ggML(fit.C5.0Tree, crossValidation, crossValidation$classe, "C5.0Tree", "Single C5.0 Tree")
g13 <- ggML(fit.C5.0Rules, crossValidation, crossValidation$classe, "C5.0Rules", "Single C5.0 Ruleset")

# Error: Error in predict.train(., newdata = newdata, type = "prob") : only classification models that produce probabilities are allowed
# Just ignore the errors
g8 <- ggML(fit.Mlda, crossValidation, crossValidation$classe, "Mlda", "Maximum Uncertainty LDA")
g11 <- ggML(fit.CSimca, crossValidation, crossValidation$classe, "CSimca", "SIMCA")
g14 <- ggML(fit.OneR, crossValidation, crossValidation$classe, "OneR", "Single Rule Classification")
g15 <- ggML(fit.RSimca, crossValidation, crossValidation$classe, "RSimca", "Robust SIMCA")
```

Both performance measures correspond to the models' accuracy and generally do not detect clear bias. Models flagged as biased are `QdaCov` (which often misclassifies other classes under type E) and `svmRadial` (which unusually accurately classified type A). Thus, we conclude that random forest `rf` built the best model. Notably, this model correctly assigned labels to all 20 test cases in the `test` dataset, as verified by Coursera.


```r
# Plot confusion matrices
grid_arrange_shared_legend(g15[[2]], g11[[2]], g8[[2]], g14[[2]], g10[[2]], 
                           g2[[2]], g3[[2]], g7[[2]], g9[[2]], g5[[2]],
                           g1[[2]], g12[[2]], g13[[2]], g6[[2]], g4[[2]], 
                           nrow = 4, ncol = 4, position = "bottom")
```

<img src="figure/unnamed-chunk-10-1.png" title="plot of chunk unnamed-chunk-10" alt="plot of chunk unnamed-chunk-10" width="100%" style="display: block; margin: auto;" />


```r
# Plot ROC curves
grid_arrange_shared_legend(g15[[1]], g11[[1]], g8[[1]], g14[[1]], g10[[1]], 
                           g2[[1]], g3[[1]], g7[[1]], g9[[1]], g5[[1]],
                           g1[[1]], g12[[1]], g13[[1]], g6[[1]], g4[[1]], 
                           nrow = 4, ncol = 4, position = "bottom")
```

<img src="figure/unnamed-chunk-11-1.png" title="plot of chunk unnamed-chunk-11" alt="plot of chunk unnamed-chunk-11" width="100%" style="display: block; margin: auto;" />

Finally, we investigated whether some models show user bias. Indeed, some of the worse models misclassified up to 80% of labels for a specific user (e.g. `QdaCov` and `RSimca` for Eurico and Pedro).


```r
## Function that collects needed data
# predictions = output of predict() function
# method = algorithm's name

missedClasseByUser <- function(predictions, method) {
    require(dplyr)
    
    data <- predictions %>% 
        as.data.frame %>% 
        rename(classe = ".") %>% 
        mutate(truth = predictions == crossValidation$classe,
               user_name = crossValidation$user_name)
    
    data <- table(data$user_name, data$truth) %>% 
        prop.table(1) %>% 
        .[,1] %>% 
        as.data.frame()
    colnames(data) <- method
    data
}
```


```r
# Define colors (from RColorBrewer, brewer.pal(9, "OrRd") %>% .[1:8])
brew.orrd <- c("#FFF7EC", "#FEE8C8", "#FDD49E", "#FDBB84", 
               "#FC8D59", "#EF6548", "#D7301F", "#B30000") %>% colorRampPalette
brew.orrd <- brew.orrd(100)
brew.orrd[1:20] <- "#FFFFFF"

# Plot
cbind(missedClasseByUser(pred.RSimca, "RSimca"),
      missedClasseByUser(pred.CSimca, "CSimca"),
      missedClasseByUser(pred.Mlda, "Mlda"),
      missedClasseByUser(pred.OneR, "OneR"),
      missedClasseByUser(pred.QdaCov, "QdaCov"),
      missedClasseByUser(pred.lda, "lda"),
      missedClasseByUser(pred.nb, "nb"),
      missedClasseByUser(pred.svmRadial, "svmRadial"),
      missedClasseByUser(pred.qda, "qda"),
      missedClasseByUser(pred.rpart, "rpart"),
      missedClasseByUser(pred.knn, "knn"),
      missedClasseByUser(pred.C5.0Tree, "C5.0Tree"),
      missedClasseByUser(pred.C5.0Rules, "C5.0Rules"),
      missedClasseByUser(pred.treebag, "treebag"),       
      missedClasseByUser(pred.rf, "rf")) %>%
    rownames_to_column("user") %>%
    melt() %>%
    ggplot(aes(user, variable)) + 
    geom_tile(aes(fill = value), colour = "white") + 
    scale_fill_gradientn(colors = brew.orrd[20:90], name = "% misclassified") +
    scale_x_discrete(position = "top") +
    theme_minimal() + 
    labs(x = "", y = "Prediction model", title = "Misclassification of labels by user") + 
    theme(legend.position = "bottom")
```

<img src="figure/unnamed-chunk-13-1.png" title="plot of chunk unnamed-chunk-13" alt="plot of chunk unnamed-chunk-13" width="80%" style="display: block; margin: auto;" />

# Conclusions
We built a prediction model than on cross-validation dataset exceeded the accuracy of 99%. For this purpose, we used the random forest algorithm (`rf` implementation in `caret` package) which optimised the model through four resampling iterations, each one including 5-fold cross-validation. While the model correctly predicted 20/20 observations from the `test` set, it took over 9 h to build it. Such computational requirements will likely scale up even more for bigger datasets. We also identified `qda` as an algorithm which generates an acceptable model in less than 7 s (89% accuracy). Thus, in our experience, tuning parameters increase computational demands but do not guarantee higher accuracy. Interestingly, some of the underperforming models showed user-specific bias, thus demonstrating that common performance measures might miss essential characteristics of the model.

# References
[1] http://groupware.les.inf.puc-rio.br/har (section on the Weight Lifting Exercise Dataset) 

[2] Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) . Stuttgart, Germany: ACM SIGCHI, 2013. (http://groupware.les.inf.puc-rio.br/har#ixzz5N71xSZle)
