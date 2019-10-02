# YouTube-Analysis
Analyzing popular YouTube Channel ChuChuTV using an R code

I recently came across the tuber package for analyzing YouTube data (https://cran.r-project.org/web/packages/tuber/tuber.pdf) and I thought it would be a good idea to look at the popular ChuChuTV Channel

The code is written in R, data is outputted to Google Sheets and final dashboard is built using Tableau Public

Packages
#install.packages("tuber")
library(tuber)   #YouTube API

library(magrittr)
library(tidyverse)
library(purrr)
library(dplyr)
library(tidyr)
library(sqldf)

#install.packages("googlesheets")
library(googlesheets)
