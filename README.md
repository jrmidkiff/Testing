---
title: "PolitiFact - 10+ Years of Fact Checking"
author: "James Midkiff"
output: html_document
---

```{r Setup, include=FALSE, echo=FALSE, message = FALSE}
knitr::opts_chunk$set(echo = TRUE)
library(tidyverse)
library(lubridate)
library(stringr)
library(tidytext)
library(scales)
library(RColorBrewer)
library(rvest)
library(forcats)
library(knitr)
library(BSDA)
load("Final_Workspace.RData")
```
This analysis was conducted at `r Sys.time()` and only captures statement ratings PolitiFact published before that time. This project should auto-update each time that I re-run the R script. 

##Background
```{r Missing November 2008, message=F, echo=FALSE, warning=FALSE}
page_series <- tibble(contains = str_detect(date, "December 1st, 2008"), 
                      page_no = 1:length(date))
page_series <- page_series %>% 
  filter(contains == TRUE)

see_this_nov_2008 <- max(page_series$page_no)
```
According to its website, [PolitiFact](www.politifact.com) is a "fact-checking website that rates the accuracy of claims by elected officials and others who speak up in American politics." The goal of this project is to analyze all of the ratings that PolitiFact has issued since it began in 2007 and to look for trends and other curiosities in the data. Besides strengthening my data analysis and mark-up skills, my motivations for this project have been my admiration of PolitiFact's non-partisan fact-checking and my own simple curiosity.  

```{r Total Ratings, echo = FALSE, message = FALSE, tidy = TRUE}
V_Total_Ratings <- comma_format()(as.integer
                                  (politifact_orig %>% 
                                      summarise(Total_Ratings = n())))
``` 

The most recent statements evaluated by PolitiFact appear at this page, (http://www.politifact.com/truth-o-meter/statements/). As of the date and time I ran this code, **PolitiFact has evaluated a total of `r V_Total_Ratings` ratings**. [^1]


### Truth-O-Meter vs. Flip-O-Meter
On its site, PolitiFact evaluates statements using either its Truth-O-Meter or its Flip-O-Meter system. In short, the Truth-O-Meter is used to evaluate statements for their **_accuracy_** (did the speaker say the truth, a falsehood, or something in between?), while the Flip-O-Meter is used to evaluate an official's **_consistency_** on an issue (did they maintain their position, or partly/completely change their stance on a topic?).  

```{r Truth-o-Meter vs. Flip-O-Meter, echo = FALSE, message = FALSE, tidy = TRUE}
temp <- politifact_orig %>% 
  group_by(Type) %>% 
  summarise(Subtotal = n()) %>%
  mutate(Proportion = round((Subtotal / sum(Subtotal)) * 100, 1),
         Subtotal = comma_format()(Subtotal))

V_Flip_O_Meter_n <- temp %>% 
  filter(Type == "Flip-O-Meter") %>% 
  select(Subtotal)
V_Flip_O_Meter_prop <- temp %>% 
  filter(Type == "Flip-O-Meter") %>% 
  select(Proportion)

V_Truth_O_Meter_n <- temp %>% 
  filter(Type == "Truth-O-Meter") %>% 
  select(Subtotal)
V_Truth_O_Meter_prop <- temp %>% 
  filter(Type == "Truth-O-Meter") %>% 
  select(Proportion)
```
From here on, my analysis will focus only on those statements evaluated by the Truth-O-Meter since they are much more numerous. There are **`r V_Truth_O_Meter_n` (`r V_Truth_O_Meter_prop`%) Truth-O-Meter Ratings** and `r V_Flip_O_Meter_n` (`r V_Flip_O_Meter_prop`%) Flip-O-Meter Ratings.



PolitiFact assigns statements one of six possible ratings to Truth-O-Meter Ratings:  

* **TRUE** - The statement is accurate and there's nothing significant missing.

* **MOSTLY TRUE** - The statement is accurate but needs clarification or additional information.

* **HALF TRUE** - The statement is partially accurate but leaves out important details or takes things out of context.

* **MOSTLY FALSE** - The statement contains an element of truth but ignores critical facts that would give a different impression.

* **FALSE** - The statement is not accurate.

* **PANTS ON FIRE** - The statement is not accurate and makes a ridiculous claim.  

For more information on how PolitiFact selects and evaluates statements, see [here](http://www.politifact.com/truth-o-meter/article/2013/nov/01/principles-politifact-punditfact-and-truth-o-meter/).  




##Truth-O-Meter Summary Statistics
```{r Total Truth-O-Meter Ratings, echo=FALSE, message=FALSE, fig.align='center'}
truth_o_meter_ratings <- politifact %>% 
  rename(Rating = rating) %>%
  group_by(Rating) %>%
  summarise(Total_Ratings = n()) %>%
  mutate(Proportion = Total_Ratings / sum(Total_Ratings))

truth_o_meter_colors <- c("#E40602", "#E71F28", "#EE9022", "#FFD503", "#C3D52D", "#71BF44")

ggplot(truth_o_meter_ratings, aes(x = Rating, y = Proportion, fill = Rating)) +
  labs(title = "PolitiFact Truth-O-Meter Ratings Since 2017") +
  geom_col(color = "black", size = 1) +
  geom_label(label = comma_format()(truth_o_meter_ratings$Total_Ratings)) +
  scale_fill_manual(values = truth_o_meter_colors) +
  scale_y_continuous(labels = percent) +
  theme(legend.position = "none")  

```

```{r Total Truthhoods and Falsehoods, echo = FALSE, message = FALSE, tidy = TRUE}
V_truthful <- truth_o_meter_ratings %>%
  filter(Rating == c("Mostly True", "True")) %>%
  summarise(Proportion = sum(Proportion)) %>%
  mutate(Proportion = percent_format()(Proportion))
V_truthful <- as.vector(V_truthful$Proportion)

V_false <- truth_o_meter_ratings %>%
  filter(Rating == c("Pants on Fire!", "False", "Mostly False")) %>%
  summarise(Proportion = sum(Proportion)) %>%
  mutate(Proportion = percent_format()(Proportion))
V_false <- as.vector(V_false$Proportion)

V_half_true <- truth_o_meter_ratings %>%
  mutate(Proportion = percent_format()(Proportion)) %>%
  filter(Rating == "Half-True")
V_half_true <- as.vector(V_half_true$Proportion)  
```
If we consider "truths" to be those statements PolitiFact rated as _Mostly True_ or _True_ and "falsehoods" to be those rated as _Pants on Fire!_, _False_, or _Mostly False_, then **`r V_truthful` of the total statements rated were _truths_**,  **`r V_false` of the total statements rated were _falsehoods_**, and the remaining **`r V_half_true`** statements were **_Half-True_** . As such, PolitiFact has rated more statements as falsehoods than as truths; whether this represents a selection bias on PolitiFact's part or the nature of American political rhetoric is something not possible to say with this data alone. 



###Rating Issuer
```{r PolitiFact Editions, echo = FALSE, message = FALSE, warning = FALSE}
locations <- politifact %>%
  group_by(source) %>% 
  mutate(Founded = min(year(date))) %>%
  group_by(source, Founded, politifact_location) %>%
  summarise(Total_Ratings = n()) %>% 
  arrange(-Total_Ratings) %>%
  ungroup() %>%
  mutate(Proportion = percent_format()(Total_Ratings / sum(Total_Ratings))) %>%
  rename(Issuer = source, Type = politifact_location)

locations <- locations %>%
  add_column(Rank = 1:nrow(locations)) %>%
  select(Rank, everything())

#Merging in the electoral votes for each state, from https://state.1keydata.com/state-electoral-votes.php
#don't forget DC
electoral_votes <- tribble(
~US_State,	~`Electoral Votes`,	
"Alabama", 9,
"Montana", 3,
"Alaska", 3,
"Nebraska", 5,
"Arizona", 11,
"Nevada", 6,
"Arkansas", 6,
"New Hampshire", 4,
"California", 55,
"New Jersey", 14,
"Colorado", 9,
"New Mexico", 5,
"Connecticut", 7,
"New York", 29,
"Delaware", 3,
"North Carolina", 15,
"Florida", 29,
"North Dakota", 3,
"Georgia", 16,
"Ohio", 18,
"Hawaii", 4,
"Oklahoma", 7,
"Idaho", 4,
"Oregon", 7,
"Illinois", 20,
"Pennsylvania", 20,
"Indiana", 11,
"Rhode Island", 4,
"Iowa", 6,
"South Carolina", 9,
"Kansas", 6,
"South Dakota", 3,
"Kentucky", 8,
"Tennessee", 11,
"Louisiana", 8,
"Texas", 38,
"Maine", 4,
"Utah", 6,
"Maryland", 10,
"Vermont", 3,
"Massachusetts", 11,
"Virginia", 13,
"Michigan", 16,
"Washington", 12,
"Minnesota", 10,
"West Virginia", 5,
"Mississippi", 6,
"Wisconsin", 10,
"Missouri", 10,
"Wyoming", 3,
"Washington D.C.", 3)

locations <- 
  locations %>% 
  separate(Issuer, c("A", "State"), sep = 11, remove = FALSE)

locations <- left_join(locations, electoral_votes, by = c("State" = "US_State"))%>%
  select(-A, -State) %>%
  rename(`Total Ratings` = Total_Ratings) %>%
  select(everything(), `Electoral Votes`)
  
  
```

```{r Most Impt. Editions, message=F, warning=F, echo=F}
top_x_issuers <- locations %>% 
  filter(Proportion >= 1) %>% 
  mutate(Cumulative_Sum = percent_format()(cumsum(
    as.numeric(
      str_replace_all(Proportion, "[[:punct:]]", "")) / 1000)))

values <- tibble(rating = levels(politifact$rating)[1:6], value = seq(-3, 2, 1)) %>%
  mutate(rating = as_factor(rating),
         rating = fct_relevel(rating, levels(politifact$rating)))

locations_rating <- politifact %>%
  group_by(source, rating) %>% 
  summarise(Total_Ratings = n()) 

locations_rating <- left_join(locations_rating, values, by = "rating") %>%
  mutate(score = Total_Ratings * value) %>%
  summarise(Score = round((sum(score) / sum(Total_Ratings)), 3))

one_percent_issuers <- left_join(top_x_issuers, locations_rating, by = c("Issuer" = "source")) %>%
  mutate(Issuer = factor(Issuer), 
         Issuer = fct_reorder(Issuer, Score)) %>%
  arrange(Score)

one_percent_issuers <- one_percent_issuers %>%
  mutate(cut = cut_interval(one_percent_issuers$Score, 12))
```
PolitiFact is made up of various "Editions" that focus on different sources for the statements that they will rate. There are a total of `r nrow(locations)` editions, which I have grouped into two main categories: "State" Editions that focus on statements made by officials or people in a certain U.S. state (`r nrow(locations %>% filter(Type == "State"))` such editions) and "Non-State" Editions that do not focus on a a single state (the remaining `r nrow(locations %>% filter(Type != "State"))` editions).  These `r nrow(locations %>% filter(Type == "State"))` State Editions cover states with a total of `r sum(locations$'Electoral Votes', na.rm = TRUE)` electoral votes, equal to `r percent_format()(sum(locations$'Electoral Votes', na.rm = TRUE) / 538)` of the total electoral votes available for presidential elections. 

As we see from this table, PolitiFact National was the source for over one-third of the total statements evaluated. This is likely due in part to the fact that it was the earliest edition founded. The other "Not State" editions include PunditFact, which evaluates statements made by political pundits, PolitiFact Global News Service, which evaluates statements made about Health and Development, and PolitiFact NBC, which is a partnership between PolitiFact and NBC. 

```{r All PolitiFact Locations, echo = FALSE, tidy = TRUE}
kable(locations, align = c("l", "l", "r", "r", "r", "r", "r"), 
      caption = "The Various PolitiFact Editions")
```


Given that only a subset of all of the PolitiFact Editions are responsible for the vast majority of the statement ratings, I am going to focus on those Editions which individually were responsible for more than 1% of the Truth-O-Meter ratings. These top `r nrow(top_x_issuers)` PolitiFact Editions were cumulatively responsible for `r max(top_x_issuers$Cumulative_Sum)` of the total statements rated. 

Looking at the most important PolitiFact Editions, I have set up a metric to measure the average truthfulness of the statements that they have rated. Any statement that is rated as _True_, I have assigned a Truthfulness Score of +2. Likewise, any statement that PolitiFact has rated _False_, I have assigned a Truthfulness value of -2. _Mostly True_, _Half-True_, and _Mostly False_ statements thus correspond to scores of +1, 0, and -1 respectively. _Pants on Fire!_ claims are "False" statements that are especially ridiculous, so I have given them a Truthfulness Score of -3 (see Table below). 
```{r Table of Values, echo = FALSE, fig.align='center', fig.width=3}
kable(values %>% 
        rename(`Statement Rating` = rating, `Truthfulness Score` = value) %>%
        arrange(desc(`Truthfulness Score`)), 
      caption = "Truthfulness Score per Statement Rating")
```

Using then the top `r nrow(top_x_issuers)` PolitiFact Editions, I have generated a Mean Truthfulness Score for all of the statements PolitiFact has examined. This number takes the total number of statements by rating (e.g. _True_, _Mostly True_, etc.), the Truthfulness Score I have assigned each rating, and then divides the sum of those Truthfulness values by the total number of ratings that each Edition has made. In other words, this metric says "What is the average truthfulness of a statement evaluated by each PolitiFact Edition?". 

As we see from the graph below, **PunditFact has the lowest average Truthfulness score at `r min(one_percent_issuers$Score)`**, which means that the _average_ statement that PunditFact rates is **_Mostly False_**. Rounding out the top three, `r one_percent_issuers$Issuer[[2]]` and `r one_percent_issuers$Issuer[[3]]` were the next Editions with the lowest average Truthfulness Scores. On the other end of the spectrum **`r one_percent_issuers$Issuer[[12]]` and `r one_percent_issuers$Issuer[[11]]` were the only Editions who had a positive truthfulness rating**, meaning that the _average_ statement they rated was at least **_Half-True_**. 

```{r Mean Truthfulness Score, echo=FALSE, fig.align='center', fig.width=10.5, tidy=TRUE}
ggplot(one_percent_issuers, aes(x = Issuer, y = Score, fill = cut)) +
  labs(x = "PolitiFact Edition (i.e. who rated the statement?)", 
       y = "Average Truthfulness Rating", 
       title = "Mean Truthfulness Score by PolitiFact Edition") +
  geom_col(color = "black", size = 1) +
  geom_label(label = one_percent_issuers$Score) +
  scale_x_discrete(labels = c(
    "PolitiFact Wisconsin" = "PolitiFact\n Wisconsin", 
    "PolitiFact National" = "PolitiFact\n National",
    "PolitiFact Rhode Island" = "PolitiFact\n Rhode Island",
    "PolitiFact New Hampshire" = "PolitiFact\n New Hampshire",
    "PolitiFact Virginia" = "PolitiFact\n Virginia",
    "PolitiFact Texas" = "PolitiFact\n Texas",
    "PolitiFact Florida" = "PolitiFact\n Florida",
    "PolitiFact Oregon" = "PolitiFact\n Oregon",
    "PolitiFact New Jersey" = "PolitiFact\n New Jersey",
    "PolitiFact Ohio" = "PolitiFact\n Ohio",
    "PolitiFact Georgia" = "PolitiFact\n Georgia")) +
  scale_fill_manual(values = c("#E40602", "#D0240D", "#BD4318", "#AA6223", "#97812E", "#84A039", "#71BF44")) +
  theme(legend.position = "none")
```



###Ratings Over Time
Given that PolitiFact's provides the date that it has rated each statement, it may be interesting to explore if the relative truthfulness of the statements rated has changed over time. Since mid-2010, PolitiFact has generally issued about 3-4 statement ratings per day. In the below graph, I have taken the Mean Truthfulness Score for all of the statements that PolitiFact has evaluated each month (excluding months where PolitiFact rated fewer than 30 statements) and a smooth line to estimate the average Mean Truthfulness Score over time. 

Some interesting trends present themselves.  

1. The Mean Truthfulness Score for each month is generally negative (the overall rating for all statements PolitiFact has rated was `r round(avg_score_all$avg_value, 2)`). There were only two occasions where the Mean Truthfulness Score was notably positive for a few months in a row: September and October of 2007 (early in PolitiFact's rating history) and around December 2012 / January 2013. To a smaller extent, this same positive blip also occurred in December 2014 / January 2015. Maybe the PolitiFact staff around the holiday season join in the spirit of giving by becoming more generous with their ratings?  
2. I had thought that the presidential campaign season, which for the sake of argument I have set as one-year prior to the presidential election, would have seen several months with particularly low Mean Truthfulness Scores. My thinking was that close to the presidential election season, we would see politicians lying a lot more at the last minute to get elected. At least from this graph, these factors appear to be uncorrelated, though that may be due to the fact that there are not many presidential candidates per election and the number of them decreases as the election date approaches.  

```{r Worst Month, echo=F, message=F, warning=F}
lowest_score_ever <- min(year_score_all$Score, na.rm = TRUE)
worst_month <- year_score_all %>% 
  filter(Score == lowest_score_ever) %>%
  mutate(Label = str_replace_all(Label, c("Jan" = "January", "Feb" = "February", "Mar" = "March",
                           "Apr" = "April", "May" = "May", "Jun" = "June", "Jul" = "July",
                           "Aug" = "August","Sep" = "September", "Oct" = "October", 
                           "Nov" = "November", "Dec" = "December")))
```

3. The Mean Truthfulness Score for each month has become remarkably negative since April 2017, with the **record low in `r worst_month$Label` at `r worst_month$Score`** as you can see in the graph below. Each month since April (except October) has broken the previously low record set in August 2009. Why is this so? Some possible ideas may be:  
    i) [Increased political polarization](http://www.people-press.org/interactives/political-polarization-1994-2017/) whereby politics has gotten to the point where candidates will say anything to win  
    ii) An artefact of the [post-truth](https://www.economist.com/news/briefing/21706498-dishonesty-politics-nothing-new-manner-which-some-politicians-now-lie-and) era some pundits think we have entered into?  
    iii) Donald Trump and his notoriety for telling falsehoods  
    iv) Or is there some other explanation?  
  
  

```{r Overall Average Score, echo = FALSE, message = FALSE, tidy = TRUE, warning = FALSE, fig.align="center", fig.width=10.5}
avg_score_all <- politifact %>%
  group_by(rating) %>%
  summarise(n = n())

avg_score_all <- left_join(avg_score_all, values, by = c("rating" = "rating")) %>%
  mutate(score = value * n) %>%
  summarise(avg_value = sum(score) / sum(n)) #The average score for PolitiFact Truth-O-Meter's history
```

```{r Avg. Score by Month, echo = FALSE, message = FALSE, tidy = TRUE, warning = FALSE, fig.align="center", fig.width=12}
year_score_all_pre_na <- politifact %>% 
  mutate(revised_date = date) 
day(year_score_all_pre_na$revised_date) <- 1 #Setting all the days to day = 1 so that way they're grouped by month

year_score_all_pre_na <- year_score_all_pre_na %>%
  group_by(revised_date, rating) %>%
  summarise(n = n())

year_score_all_pre_na <- left_join(year_score_all_pre_na, values, by = c("rating" = "rating")) %>%
  ungroup() %>%
  mutate(Score = n * value) %>%
  group_by(revised_date) %>%
  summarise(Score = round((sum(Score) / sum(n)), 3),
            Total_Ratings = sum(n))

year_score_all <- year_score_all_pre_na %>%
  mutate(Score = ifelse(Total_Ratings < 30, NA, Score)) %>% #Only dealing with those with more than 30+ ratings a month
  separate(revised_date, into = c("Year", "Month", "Junk"), remove = FALSE) %>%
  mutate(Month = str_replace_all(Month, c("01" = "Jan", "02" = "Feb", "03" = "Mar",
                                          "04" = "Apr", "05" = "May", "06" = "Jun",
                                          "07" = "Jul", "08" = "Aug", "09" = "Sep",
                                          "10" = "Oct", "11" = "Nov", "12" = "Dec"))) %>%
  unite(Label, Month, Year, sep = " ") %>%
  select(-Junk)
  
                                         
ggplot(year_score_all %>% slice(5:nrow(year_score_all)), #Lop off the first 4 rows because they're NA
       aes(x = revised_date + 15, y = Score, fill = Score)) + #Add 15 days to each month so they align better
  geom_rect(xmin = ymd("2016-Nov-8") - 365, xmax = ymd("2016-Nov-8"),
            ymin = -3, ymax = 2, fill = "gray75", color = "black", linetype = "dashed", size = .8) +
  geom_rect(xmin = ymd("2012-Nov-6") - 365, xmax = ymd("2012-Nov-6"),
            ymin = -3, ymax = 2, fill = "gray75", color = "black", linetype = "dashed", size = .8) +
  geom_rect(xmin = ymd("2008-Nov-4") - 365, xmax = ymd("2008-Nov-4"),
            ymin = -3, ymax = 2, fill = "gray75", color = "black", linetype = "dashed", size = .8) +
  geom_col(color = "black", size = 1) +
  geom_smooth(size = 1.7, color = "royalblue4", se = FALSE) +
  scale_x_date(date_breaks = "1 year", date_labels = "%Y") +
  scale_y_continuous(breaks = seq(-1.5, 0.5, 0.25)) +
  scale_fill_continuous(low =  "#E40602", high = "#71BF44") +
  theme(legend.position = "none",
        axis.text.x = element_text(size = 12), 
        axis.text.y = element_text(size = 10)) +
  labs(x = "", 
       title = "PolitiFact Mean Truthfulness Score per Month",
       subtitle = "Only months with 30 or more ratings are included. Shaded areas indicate the 12 months prior to a U.S. presidential election") +
  geom_point(data = worst_month, aes(x = revised_date + 15, y = Score - .10),
             size = 5, color = "black", fill = "red", shape = 24, stroke = 2,
             inherit.aes = FALSE) +
  geom_text(data = worst_month, aes(x = revised_date + 15, y = Score - .2),
            label = "Minimum", inherit.aes = FALSE, color = "black", fontface = "bold")
  

```
Given how quickly PolitiFact's monthly Mean Truthfulness Score becomes very negative after the 2016 presidenital election, I decided to separate the ratings into three mutually exclusive categories to figure out why:  

1. I flagged as **Website** any rating made by an entity ending in _.com_, _.net_, _.org_, _.us_, _.edu_, _.gov_ or that had the word _email_, _facebook_, _blog_, _tweet_, or _image_ in its name.  
2. I flagged as **Trump** any rating made by _Donald Trump_.  
3. I flagged all other ratings as **Other**.  


I took the _subtotal_ Mean Truthfulness values for these three statement categories and divided by the total number of ratings issued each month by PolitiFact. These steps allows me to determine what each of the three categories contributed to PolitiFact's total Mean Truthfulness Score for each month. 

```{r Trump/Websites/All Other Data, echo = F, message = F, warning=F, fig.align="center", fig.width=10.5}
trump_web_other <- politifact %>% 
  mutate(analysis = NA)
trump_web_other <- politifact %>% 
  mutate(analysis = ifelse(website == TRUE, "Website", NA), 
         analysis = ifelse(name == "Donald Trump", "Trump", analysis),
         analysis = ifelse(is.na(analysis), "Other", analysis))

trump_web_analysis <- trump_web_other %>% 
  mutate(revised_date = date) 
day(trump_web_analysis$revised_date) <- 1 #Setting all the days to day = 1 so that way they're grouped by month

trump_web_analysis <- trump_web_analysis %>%
  group_by(revised_date, rating, analysis) %>%
  summarise(n = n()) 

trump_web_analysis <- left_join(trump_web_analysis, values, by = c("rating" = "rating")) %>%
  ungroup() %>%
  mutate(Score = n * value) %>%
  group_by(revised_date, analysis) %>%
  summarise(subtotal = sum(Score),
            Subtotal_Ratings = sum(n))

trump_web_analysis <- left_join(trump_web_analysis, year_score_all_pre_na) %>%
  mutate(Sub_Score = round(subtotal / Total_Ratings, 3)) 
trump_web_analysis <- trump_web_analysis[c(1:4, 7, 5:6)] #Reordering columns

trump_web_analysis <- trump_web_analysis %>%
  mutate(Score = ifelse(Total_Ratings < 5, NA, Score)) %>% #Only dealing with those with more than 30+ ratings a month
  separate(revised_date, into = c("Year", "Month", "Junk"), remove = FALSE) %>%
  mutate(Month = str_replace_all(Month, c("01" = "Jan", "02" = "Feb", "03" = "Mar",
                                          "04" = "Apr", "05" = "May", "06" = "Jun",
                                          "07" = "Jul", "08" = "Aug", "09" = "Sep",
                                          "10" = "Oct", "11" = "Nov", "12" = "Dec"))) %>%
  unite(Label, Month, Year, sep = " ") %>%
  select(-Junk)

website_contribution_worst <- trump_web_analysis %>% 
  filter(revised_date %in% worst_month$revised_date, 
         analysis == "Website") %>%
  mutate(percentage = Sub_Score / Score)
trump_contribution_worst <- trump_web_analysis %>% 
  filter(revised_date %in% worst_month$revised_date, 
         analysis == "Trump") %>%
  mutate(percentage = Sub_Score / Score)
other_contribution_worst <- trump_web_analysis %>% 
  filter(revised_date %in% worst_month$revised_date, 
         analysis == "Other") %>%
  mutate(percentage = Sub_Score / Score)
```

For example, in `r worst_month$Label` for all of PolitiFact there were `r worst_month$Total_Ratings` total statements rated with a Mean Truthfulness Score  of **`r worst_month$Score`**. This value correlates to the average statement being at least _Mostly False_, and this month has the lowest Mean Truthfulness Score since PolitiFact was founded as you can again see in the graph below. Of that `r worst_month$Score` Mean Truthfulness Score:  

* With `r website_contribution_worst$Subtotal_Ratings` statements rated, **Websites** contributed `r website_contribution_worst$Sub_Score` to this month's Mean Truthfulness Score, equal to **`r percent_format()(website_contribution_worst$percentage)`** of it  
* With `r trump_contribution_worst$Subtotal_Ratings` statements rated, **Donald Trump** contributed `r trump_contribution_worst$Sub_Score` to this month's Mean Truthfulness Score, equal to **`r percent_format()(trump_contribution_worst$percentage)`** of it   
* And the remaining `r other_contribution_worst$Subtotal_Ratings` statements rated contributed `r other_contribution_worst$Sub_Score` to this month's Mean Truthfulness Score, equal to the remaining **`r percent_format()(other_contribution_worst$percentage)`** of it  
```{r Websites, echo=F}
websites_by_month <- trump_web_analysis %>% 
  filter(analysis == "Website", 
         revised_date >= first_website$date[[1]]) %>% 
  mutate(pre_dec_2016 = ifelse(revised_date < "2016-12-01", "PRE", "POST")) %>% 
  group_by(pre_dec_2016) %>% 
  summarise(avg_per_month = round(sum(Subtotal_Ratings) / n(), 1), 
            months = n(), 
            avg_rating = round(sum(subtotal) / sum(Subtotal_Ratings), 3))
```
Since December 2016, ratings of statements made by websites have been the single largest contributor to the highly negative Mean Truthfulness Scores of late. This coincides with the [founding of PolitiFact's partnership with Facebook](http://www.politifact.com/truth-o-meter/article/2017/dec/15/we-started-fact-checking-partnership-facebook-year/) to fact-check claims made on the social media site, and likely represents a push overall to rate the accuracy of more claims made online. [^2] Since December 2016, PolitiFact has rated an average of `r websites_by_month$avg_per_month[[1]]` statements made online per month and these statements have an average rating of `r websites_by_month$avg_rating[[1]]`, which is closest to **_Pants on Fire!_**. Prior to December 2016, PolitiFact only rated an average of `r websites_by_month$avg_per_month[[2]]` statements from websites per month, and with an average rating of `r websites_by_month$avg_rating[[2]]` these ratings were closer to **_False_**. 

```{r Trump/Websites/All Other Plot, echo = F, message = F, warning=F, fig.align="center", fig.width=10.5}
ggplot(trump_web_analysis %>% filter(revised_date >= mdy("January 1, 2014")), 
       aes(x = revised_date + 15, y = Sub_Score, fill = analysis, 
           color = analysis, group = analysis)) + #Add 15 days to each month so they align better
  geom_rect(xmin = ymd("2016-Nov-8") - 365, xmax = ymd("2016-Nov-8"),
            ymin = -3, ymax = 2, fill = "gray75", color = "black", linetype = "dashed", size = .8) +
  geom_col(color = "black", size = 1) +
  scale_x_date(date_breaks = "1 year", date_labels = "%Y") +
  scale_fill_manual("", values = c("Website" = "#c118ec", "Trump" = "#ffb00a", "Other" = "#808080"),
                    position = "bottom") +
  scale_y_continuous(breaks = seq(-1.25, 0.5, 0.25)) +
  labs(title = "PolitiFact Mean Truthfulness Score per Month, 2014 - Present",
       x = "", y = "Monthly Average", 
       subtitle = "Shaded area indicates 12 months prior to the November 8, 2016 presidential election. Mean Truthfulness Scores are additive for Donald Trump, Websites, and Other.") +
  theme(legend.position = "bottom", 
        legend.box.margin = margin(t = -20), 
        axis.text.x = element_text(size = 12), 
        axis.text.y = element_text(size = 10)) +
  guides(fill = guide_legend(reverse = TRUE)) +
  geom_point(data = worst_month, aes(x = revised_date + 15, y = Score - .10),
             size = 5, color = "black", fill = "red", shape = 24, stroke = 2,
             inherit.aes = FALSE) +
  geom_text(data = worst_month, aes(x = revised_date + 15, y = Score - .2),
            label = "Minimum", inherit.aes = FALSE, color = "black", fontface = "bold")

```

##Subject
```{r Topics, echo = F, message=F, warning = F, fig.align="center", fig.width=10.5}
total_topics <- total_topics %>%
  mutate(scorecard = str_replace_all(scorecard, c("Half True" = "Half-True", "Pants on Fire" = "Pants on Fire!")))

total_topic_ratings <- left_join(total_topics, values, by = c("scorecard" = "rating")) %>%
  mutate(Score = value * count) %>%
  group_by(topic)

total_topic_summary <- total_topic_ratings %>%
  summarise(total_ratings = sum(count),
            total_score = sum(Score)) %>%
  mutate(Avg_Score = total_score / total_ratings) %>% 
  mutate(Pos_Neg = ifelse(Avg_Score >= 0, "Positive", "Negative"))

table_of_topics <- total_topic_summary %>%
  filter(rank(desc(total_ratings)) < 11) %>% 
  arrange(desc(total_ratings)) %>%
  mutate(`Mean Truthfulness Score` = round(Avg_Score, 3),
         `Total Ratings` = comma_format()(total_ratings)) %>%
  rename(Subject = topic) %>%
  mutate(Rank = 1:10) %>% 
  select(Rank, Subject, `Total Ratings`, `Mean Truthfulness Score`) 

```  
  
PolitiFact groups together many of the statements it rates by the subject that statement addresses (e.g. Abortion, Patriotism, Obama Birth Certificate, etc.) [here](http://www.politifact.com/subjects/). PolitiFact also notes the frequency with which it assigns its six ratings (i.e. *True*, *Mostly True*, *Half-True*, *Mostly False*, *False*, and *Pants on Fire!*) to statements in each subject. Currently there are `r nrow(as_tibble(unique(total_topics$topic)))` different subjects. The most frequently discussed subjects and their Mean Truthfulness Scores are below:

```{r Topics Table, echo = F, message=F, warning = F}
kable(table_of_topics, align = c("c", "l", "r", "r"), 
      caption = "Most Discussed Subjects in PolitiFact Ratings")
```  

Looking at the most truthfully discussed and most falsely discussed subjects, it comes as no surprise that topics such as "Fake news" and [Obama's Birth Certificate](https://obamawhitehouse.archives.gov/sites/default/files/rss_viewer/birth-certificate-long-form.pdf) have very negative Mean Truthfulness Scores. I leave it to others to ponder why topics such as "Population", "Redistricting", and "Gambling" are more truthfully discussed. I believe part of it has to do with uneven statement sampling.  

```{r Best/Worst Topics, echo = F, message=F, fig.align="center", fig.width=10.5}
top_10 <- total_topic_summary %>% 
  filter(total_ratings >= 30) %>%
  filter(rank(desc(Avg_Score)) < 11) %>% 
  arrange(desc(Avg_Score)) 

bottom_10 <- total_topic_summary %>% 
  filter(total_ratings >= 30) %>%
  filter(rank(Avg_Score) < 11) %>% 
  arrange(Avg_Score) 

both_10s <- bind_rows(top_10, bottom_10) %>%
  mutate(topic = as_factor(topic),
         topic = fct_reorder(topic, Avg_Score))

ggplot(both_10s, aes(x = topic, y = Avg_Score, fill = Pos_Neg)) +
  geom_col(size = 1, color = "black") +
  geom_vline(xintercept = 10.5, size = 1.5, linetype = "twodash") +
  coord_flip() +
  scale_fill_manual(values = c("Positive" = "#71BF44", "Negative" = "#E40602")) +
  theme(legend.position = "none",
        axis.text = element_text(size = 12),
        axis.title = element_text(size = 12)) +
  labs(x = "", y = "Mean Truthfulness Score", 
       title = "Top 10 Most Truthfully Discussed and Most Falsely Discussed Truth-O-Meter Subjects",
       subtitle = "Only subjects with at least 30 rated statements are included.") +
  scale_y_continuous(limits = c(-3, 0.5), breaks = c(-3:0, 0.5))
```

##Individuals
```{r  Democrats and Republicans, echo=F, message=F, warning=F}
number_of_politicians <- verified_people_speakers %>% 
  group_by(Party) %>% 
  distinct(name) %>% 
  summarise(`Number of Politicians Rated` = n())

party_statements <- verified_people_speakers %>% 
  group_by(Party, rating) %>% 
  summarise(Subtotal_Ratings = sum(n)) %>%
  ungroup() %>%
  group_by(Party) %>%
  mutate(`Total Ratings` = comma_format()(sum(Subtotal_Ratings))) %>%
  spread(rating, Subtotal_Ratings) %>%
  select(-`Total Ratings`, everything(), `Total Ratings`)

nominal_party_statements <- left_join(party_statements, number_of_politicians, by = c("Party" = "Party")) %>%
  select(Party, `Number of Politicians Rated`, everything())

number_party_statements <- verified_people_speakers %>% 
  group_by(Party, rating) %>% 
  summarise(Subtotal_Ratings = sum(n)) %>%
  ungroup() %>%
  group_by(Party) %>%
  mutate(Total_Ratings = sum(Subtotal_Ratings)) %>%
  spread(rating, Subtotal_Ratings) %>%
  select(-Total_Ratings, everything(), Total_Ratings)
```
And finally, the last part of my analysis looks at various U.S. elected politicians' individual records with the Truth-O-Meter. My intent has been to capture the most important U.S. politicians, whom I have defined to include current and recently former Presidents, Vice-Presidents, Presidential Candidates, Senators, Representatives, and Governors. [^3] In total, I will focus on the `r nrow(verified_people_speakers %>% group_by(name) %>% summarise(n = n()))` Democratic and Republican U.S. politicians from these groups who have made statements that PolitiFact has rated. 

Interestingly, PolitiFact has rated statements by `r percent_format()(as.numeric(nominal_party_statements[2,2]/nominal_party_statements[1,2] - 1))` more Republican politicians than Democratic ones (`r nominal_party_statements[2,2]` vs. `r nominal_party_statements[1,2]`) but it has rated `r percent_format()(number_party_statements$Total_Ratings[2]/number_party_statements$Total_Ratings[1] - 1)` more statements by Republican politicians than statements by Democratic ones (`r nominal_party_statements[2,9]` vs. `r nominal_party_statements[1,9]`). See the table below:

```{r Nominal Party, echo=FALSE}
kable(nominal_party_statements, align = c("l", "r", "r", "r", "r", "r", "r", "r", "r"),
      caption = "Number of Statements by Major U.S. Politicians by Party")
```

```{r Text Top Bottom, echo=F, message=F, warning= F}
peoples_scores <- left_join(verified_people_speakers, values, by = c("rating" = "rating")) %>%
  mutate(sub_score = n * value) %>%
  ungroup() %>%
  group_by(name, Party) %>%
  summarise(total_ratings = sum(n),
         sum = sum(sub_score),
         score = sum / total_ratings) %>%
  filter(total_ratings > 30)

top_5_peoples_scores <- peoples_scores %>%
  group_by(Party) %>%
  filter(rank(desc(score)) < 6) %>% 
  arrange(desc(score)) %>%
  filter(score >= 0)

bottom_5_peoples_scores <- peoples_scores %>%
  group_by(Party) %>%
  filter(rank(score) < 6) %>% 
  arrange(score) %>%
  filter(score <= 0)

top_bot_5_both <- bind_rows(top_5_peoples_scores, bottom_5_peoples_scores) %>%
  ungroup() %>%
  mutate(name = as_factor(name),
         name = fct_reorder(name, score),
         color = ifelse(Party == "Democratic", "#1A80C4", "#CC3D3D"))


dems_repubs_function <- function(number, party) {
  the_tibble <- top_bot_5_both %>% 
    top_n(number, score) %>% 
    filter(Party == party)
  
  str_c(the_tibble$name, collapse = ", ") %>%
    str_replace_all(", (?=\\w+\\s\\w+$)", ", and ")
}
top_dems <- dems_repubs_function(10, "Democratic")
bottom_dems <- dems_repubs_function(-10, "Democratic")
top_repubs <- dems_repubs_function(10, "Republican")
bottom_repubs <- dems_repubs_function(-10, "Republican")
```
Next, I will look at which politicians have the highest Mean Truthfulness score by party. The five Democratic politicians with the highest Truthfulness Score are `r top_dems` while the five Democrats with the lowest Truthfulness Score are `r bottom_dems`. The five Republican politicians with the highest Truthfulness Score are `r top_repubs` while the five Republicans with the lowest truthfulness score are `r bottom_repubs`. **Notably, the five least truthful Democrats all have a higher Mean Truthfulness Score than the five least truthful Republicans.** 


```{r Individual Graphs, echo=F, message=F, warning=F, fig.align="center", fig.width=10.5}
colors <- top_bot_5_both$color[order(top_bot_5_both$name)] #Had to post on stackoverflow to figure this one out

ggplot(top_bot_5_both, aes(x = name, y = score, fill = Party)) +
  geom_col(size = 1, color = "black") +
  coord_flip() +
  geom_vline(xintercept = 10.5, size = 1.5, linetype = "twodash") +
  scale_fill_manual(values = c("Democratic" = "#1A80C4", "Republican" = "#CC3D3D")) +
  labs(y = "Mean Truthfulness Score", x = "",
       title = "Top 5 Individual Highest and Lowest Mean Truthfulness Scores by Party",
       subtitle = "Only major U.S. politicians with at least 30 rated statements are included") +
  theme(axis.text = element_text(size = 12),
        axis.text.y = element_text(color = colors),
        axis.title = element_text(size = 14),
        legend.position = "bottom", 
        legend.title = element_blank())
```
```{r All Politicians Lie, echo=F}
Those_Who_Lie <- verified_people_speakers %>%
  filter(name %in% peoples_scores$name) %>%
  group_by(name) %>%
  filter(rating == "Pants on Fire!" | rating == "False")

Those_Who_Dont <- peoples_scores %>%
  filter(!name %in% Those_Who_Lie$name)

```
Currently, there are **`r nrow(Those_Who_Dont)` major U.S. Politicians who have not made a _False_ or _Pants on Fire!_ statement**, among those with 30 or more statements rated by PolitiFact. The maxim that "All politicians lie" seems to be accurate, but it should be followed up with "but some lie a lot more than others."

###Overall Party Mean Truthfulness Score
```{r Democrats and Republicans Total, echo=F, message=F, warning=F, fig.align="center", fig.width=8}
politifact <- politifact %>% 
  mutate(name = str_replace_all(name, "  ", " ")) %>%
  mutate(rating = fct_drop(rating))

politifact <- left_join(politifact, values, by = "rating")

party_statistical_test <- inner_join(politifact, 
                                      verified_people_speakers %>% 
                                        filter(!duplicated(name)) %>%
                                        select(name, Party), 
                                      by = c("name" = "name"))
  
democrats <- party_statistical_test %>%
  filter(Party == "Democratic") 
republicans <- party_statistical_test %>%
  filter(Party == "Republican")
  
```
The final part of my analysis includes grouping all of the major U.S. politicians who have received ratings from PolitiFact by party. This may be an accurate reflection of which party is more "truthful" or it may reflect selection-bias by me or by PolitiFact. Nonetheless, after performing a z-test on statements by major U.S. politicians from the Democratic and Republican party, I can say with greater than 99% confidence that the Mean Truthfulness Score for statements made by major U.S. Democratic politicians is statistically higher than that for major U.S. Republican politicians. [^4]  

```{r Democrats and Republicans Total Graph, echo=F, message=F, warning=F, fig.align="center", fig.width=8}
party_politicians <- verified_people_speakers %>%
  group_by(Party, rating) %>%
  summarise(`Total Statements` = sum(n))

party_politicians <- left_join(party_politicians, values, by = c("rating" = "rating")) %>%
  mutate(score = value * `Total Statements`) %>%
  group_by(Party) %>%
  summarise(Total_Statements = sum(`Total Statements`),
            Score = sum(score)) %>%
  mutate(Final_Ratings = Score / Total_Statements)

added_row <- tibble(Party = "Average Major U.S. Politician", 
                    Total_Statements = sum(party_politicians$Total_Statements), 
                    Score = sum(party_politicians$Score), 
                    Final_Ratings = Score / Total_Statements)

party_politicians <- party_politicians %>%
  bind_rows(added_row) %>%
  mutate(Party = as_factor(Party),
         Party = fct_relevel(Party, "Average Major U.S. Politician", after = 2))

ggplot(party_politicians, aes(x = Party, y = Final_Ratings, fill = Party, 
                              label = round(Final_Ratings, 3))) +
  geom_col(size = 1, color = "black") +
  scale_fill_manual(values = c("Democratic" = "#1A80C4", "Republican" = "#CC3D3D", "Average Major U.S. Politician" = "#735F81"), guide = "none") +
  labs(x = "", y = "", 
       title = "Mean Truthfulness Score for Major Politicians' Statements by Party",
       subtitle = str_c("Includes ", 
                        comma_format()(sum(party_politicians %>% 
                                             filter(Party == c("Democratic", "Republican")) %>% 
                                             select(Total_Statements))), 
                        " statements rated by PolitiFact from ", 
                        nrow(verified_people_speakers %>% 
                               group_by(name) %>% 
                               summarise(n = n())), 
                        " current and recent major U.S. politicians, including:\nPresidents, Vice-Presidents, Presidential Candidates, Senators, Representatives, and Governors.")) +
  geom_hline(yintercept = 0, size = 1.5, linetype = "twodash") +
  geom_vline(xintercept = 2.5, size = 1.5, linetype = "solid") +
  theme(axis.text = element_text(size = 12),
        axis.text.x = element_text(color = c("#1A80C4", "#CC3D3D", "#735F81"), face = "bold"),
        panel.grid.major.x = element_blank()) +
  geom_label()
```

##Final Words
This project is simply an effort to improve my data wrangling and analysis skills as well to create a personal and demonstrable product of my abilities. While I have no connection to PolitiFact, I deeply appreciate the work they do and I encourage others to do the same. I apologize for using up their server space while scraping their site, but hopefully any exposure of their work and financial contributions from myself and others can more than repay that.  

For information on joining PolitiFact, see [here](http://www.politifact.com/truth-o-meter/blog/2011/oct/06/who-pays-for-politifact/). 

You are free to share this github site (attributing me as the author) or to use any of the R code I have written for your own private, non-commercial use. Simply put, please respect the time and effort I put into this project.

###Footnotes
[^1] Not until I had done a lot data wrangling did I realize that PolitiFact seems to be missing from its site any ratings issued in November 2008. While it has ratings through October 31, 2008 and beginning again on December 1, 2008, there are no ratings at all for November 2008.  Visit this page and nearby ones to verify (http://www.politifact.com/truth-o-meter/statements/?page=`r see_this_nov_2008`)  

[^2] It appears that PolitiFact's efforts to fact-check claims appearing on Facebook [do not appear in its Truth-O-Meter ratings](http://www.politifact.com/truth-o-meter/article/2017/dec/15/we-started-fact-checking-partnership-facebook-year/), which puts them beyond the scope of my analysis. When I discuss PolitiFact's ratings of statements made by websites, I only analyze those that appear in PolitiFact's Truth-O-Meter ratings.  

[^3] PolitiFact has rated statements by `r comma_format()(nrow(politifact %>% group_by(name) %>% summarise(n = n())))` different entities. In my analysis of statements by major politicians by their political party, I used the following lists:  

1) U.S. Senators and Representatives from the 109th to the 115th Congressional Sessions, available [here](https://www.congress.gov/members?q=%7B%22congress%22%3A%5B%22110%22%2C%22111%22%2C%22112%22%2C%22113%22%2C%22114%22%2C%22115%22%5D%7D). I included all of the sessions that have overlapped with PolitiFact's existence, which began in 2007. 
2) Current U.S. State Governors, available [here](https://en.wikipedia.org/wiki/List_of_living_former_United_States_Governors)
3) Currently living former U.S. State Governors, available [here](https://www.nga.org/files/live/sites/NGA/files/pdf/directories/GovernorsList.pdf)
4) List of [2008](https://www.nytimes.com/elections/2008/primaries/candidates/index.html), [2012](https://www.nytimes.com/elections/2012/primaries/candidates.html), and [2016](https://www.nytimes.com/interactive/2016/us/elections/2016-presidential-candidates.html?_r=0) presidential candidates

Because the matching process is difficult, I cannot guarantee that every individual who falls into the following categories has been accounted for but this should include enough politicians to make my conclusions and analysis valid. I furthermore sought to ensure that every valid U.S. politician who has had at least 10 statements rated by PolitiFact was included. The political party designation used comes from the above mentioned sources. 

The remaining entities whose individual records I have intentionally not chosen to analyze include:  

* Organizations, civic initiatives, PACs, campaigns, party committees and the like
* Websites  
* Pundits such as Rush Limbaugh, Sean Hannity, Rachel Maddow, etc.  
* And Politicians whose only positions thus far are:  
    * At the state level  
    * In a non-elected appointment (i.e. to the cabinet level, ambassadorship, or the Supreme Court)  
    * Unsuccessful campaigns for a political seat (other than U.S. president)  
    * Not-yet assumed office (i.e. Governor-elect, Senator-elect, etc., but I do include Presidents-elect)
    * Or not clearly associated with either the Democratic or Republican parties (e.g. I have excluded Gary Johnson, Ralph Nader, and Jesse Ventura, but not Bernie Sanders or Joe Lieberman). 
        
Keep in mind that many politicians who did fit my criteria may at some point have held a position on the above list. When I refer to 'Democrats' and 'Republicans' in my analysis, I am referring to politicians who I have been able to merge 
```{r, echo = F}
final_test <- z.test(x = democrats$value, y = republicans$value, 
       sigma.x = sd(democrats$value), sigma.y = sd(republicans$value), 
       alternative = "two.sided", conf.level = 0.99)
```

[^4] A z-test is used when the population variance and standard-deviation are known. Because I am considering the population only to be the statements that PolitiFact has rated (rather than _all_ statements _all_ politicians have made), I believe a z-test is more appropriate than a t-test. For Democrats, the standard deviation of their Mean Truthfulness Score is `r round(sd(democrats$value), 3)` and for Republicans it is `r round(sd(republicans$value), 3)` (all numbers in this footnote are rounded to three digits). In the z-test, the null hypothesis is that the true difference between the Mean Truthfulness Score of Democrats and Republicans is not equal to zero; currently the difference between those means is `r round(mean(democrats$value) - mean(republicans$value), 3)`. With 99 percent confidence, the true difference between these means is estimated to be between `r round(final_test$conf.int[[1]], 3)` and `r round(final_test$conf.int[[2]], 3)`. 