# Tidyverse Tutorial

## The traditional approaches:
```{r}
set.seed(69)
sample <- rnorm(10)
mean(sample)
     
set.seed(69)
mean(rnorm(10))
```

## The pipe function:

```{r}
library(magrittr) #Ceci ne pas une pipe
set.seed(69)
rnorm(10) %>% mean
```

![Ceci ne pas une pipe](https://saciart.files.wordpress.com/2014/10/magritte_pipe.jpg?w=710&h=496)

## Select columns
```{r}
library(RCurl)
library(dplyr)
SchoolData<-read.csv(text=getURL('https://raw.githubusercontent.com/MixedModels/LearningMLwinN/master/tutorial.txt'), sep="\t")

head(SchoolData[,c('standlrt', 'normexam')])
select(SchoolData, standlrt, normexam) %>% head
```

## Select rows
```{r}
head(subset(SchoolData, standlrt< -1))
filter(SchoolData, standlrt< -1) %>% head
```

## Sort
```{r}
head(SchoolData[order(SchoolData$standlrt),])
arrange(SchoolData, standlrt) %>% head
```

## Calculate group means
```{r}
head(aggregate(normexam ~ school, data=SchoolData, FUN=function(x) av_score=mean(x)))
#I can't figure out how to rename the output
SchoolData %>% group_by(school) %>% summarise(av_score=mean(normexam)) %>% head
```

## Calculate multiple stats
```{r}

head(aggregate(normexam ~ girl+schgend, data=SchoolData, FUN=function(x) c(mean=mean(x), var=var(x))))
SchoolData %>% group_by(girl, schgend) %>% summarise(mean=mean(normexam), var=var(normexam)) %>% head
```

## Count occurences per group
```{r}
head(aggregate(normexam ~ school, data=SchoolData,  FUN=function(x) num_students=length(x)))
SchoolData %>% group_by(school) %>% summarise(num_students=n()) %>% head
```

## Compute new values
```{r}
SchoolData$improvement <- SchoolData$normexam - SchoolData$standlrt
head(SchoolData)
SchoolData %>% mutate(improvement=normexam-standlrt) %>% head
```

## Pipe results to ggplot
```{r}
library(ggplot2)
library(scales)
library(RColorBrewer)
require(grid)
source("~/Documents/R/FormatGGplot.R")

SchoolData %>% 
  group_by(school) %>% 
  summarise(mean=mean(normexam), se=sd(normexam)/sqrt(n())) %>% 
  ggplot(aes(x=school, y=mean)) +
    geom_bar(stat="identity", fill="royalblue4", alpha=1/2) +
    geom_errorbar(aes(ymin=mean-se, ymax=mean+se), colour="royalblue4", alpha=1/2) +
    fte_theme()
```

## Incorporate other datasets
```{r}
SchoolData %>% 
  group_by(school) %>% 
  summarise(mean=mean(normexam), se=sd(normexam)/sqrt(n())) %>% 
  ggplot(aes(x=school, y=mean)) +
    geom_jitter(aes(x=school, y=normexam), alpha=1/10, 
                position = position_jitter(width = 0.2), 
                colour="royalblue4", data=SchoolData) +
    geom_point(stat="identity", alpha=2/3, shape=5, size=2, colour="royalblue4") +
    geom_errorbar(aes(ymin=mean-se, ymax=mean+se), colour="royalblue4", alpha=2/3) +
    fte_theme()

```

## Select top student in each shool 
use row_number() if you want a single student per school
```{r}
SchoolData %>% group_by(school) %>% filter(min_rank(desc(normexam)) == 1)

```

## Calculate schav from chapter 6
dense_rank() assigns the same rank to ties but assigns consecutive ranks to different numbers. This gives consecutive ranks for each school.
```{r}
SchoolData %>% 
  select(school, student, normexam, standlrt, schav) %>% 
  group_by(school) %>% 
  mutate(mean = mean(standlrt)) %>% 
  ungroup() %>% 
  mutate(rank=dense_rank(desc(mean))) %>% 
  mutate(schav2 = ifelse(rank < max(rank)/4, "high", ifelse(rank > max(rank) *3/4, "low", "mid")))

```

## Plot minimum differences between students vs. scores
```{r}
SchoolData %>% 
  group_by(school) %>% 
  arrange(desc(normexam)) %>% 
  mutate(gap =  lag(normexam)- normexam) %>% 
  ggplot(aes(y=gap, x=normexam)) + 
  geom_point() +
  fte_theme()

```

## Plot cumulative mean vs. number of students
```{r}
SchoolData %>% 
  select(school, student, normexam) %>% 
  group_by(school) %>% 
  mutate(mean = cummean(normexam)) %>% 
  ggplot(aes(x=student, y=mean, colour=factor(school))) + 
    geom_point() + 
    fte_theme()

```

## Reshaping data using tidyr
```{r}
library(tidyr)
FakeData <- data.frame('Feature' = c('A', 'A', 'B', 'B', 'C', 'C', 'D', 'D'), 
            'variable' = rep(c("PercentDone", "Planned"), 4),
            "value"=c(35, 50, 10, 80, 50, 75, 35, 40))
FakeData
FakeData %>%
   spread(variable, value) %>% 
   mutate(colour=ifelse(Planned-PercentDone <= 15, "Within 15%", ">15%")) %>% 
   gather("variable", "value", 2:3) %>% 
   mutate(colour = ifelse(variable == "Planned", "Plan", colour)) %>%
   ggplot(aes(x=Feature, y=value, fill=relevel(factor(colour), ref="Plan"))) +
     geom_bar(stat="identity", position="dodge") + 
     fte_theme()

```

## Combine dataframes
works the same as merge, but a lot faster
```{r}
(df1 = data.frame(CustomerId = c(1:6), Product = c(rep("Toaster", 3), rep("Radio", 3))))
(df2 = data.frame(CustomerId = c(2, 2, 4, 6, 7), State = c("Ohio", rep("Alabama", 2), rep("Ohio", 2))))

#Keep all rows in df1
df1 %>% left_join(df2)
merge(x=df1, y=df2, all.x=T)

#Keep all rows in df2
merge(x=df1, y=df2, all.y=T)
df1 %>% right_join(df2)

#Keep all rows in either
merge(df1, df2, all=T)
df1 %>% full_join(df2)

#Keep only rows in both
merge(df1, df2)
df1 %>% inner_join(df2)
```

## Working with Databases
By storing your data in a MySQL database, it's possible to work with extremely large datasets
- 21.7 million blast results (1.2 Gb on file)
```{r warning=FALSE}
db <- src_mysql("test", user = "root", password = "", host='localhost')
db %>% tbl("Blast") %>% filter(qseqid == 'KRAUScomp174846_c0_seq1_0374')
db %>% tbl("Blast") %>% filter(hitlen > 2000) %>% select(qseqid, sseqid, pident)
```

*see the [Rstudio data wrangling cheat sheet](https://www.rstudio.com/wp-content/uploads/2015/02/data-wrangling-cheatsheet.pdf)

## Alternatives to dplyr
- ddply{plyr} (slow)
- data.table (confusing)


