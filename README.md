# YouTube-Analysis
Analyzing popular YouTube Channel ChuChuTV using an R code

I recently came across the tuber package for analyzing YouTube data (https://cran.r-project.org/web/packages/tuber/tuber.pdf) and I thought it would be a good idea to look at the popular ChuChuTV Channel

The code is written in R, data is outputted to Google Sheets and final dashboard is built using Tableau Public

# Packages

install.packages("tuber")
library(tuber)   #YouTube API


library(magrittr)
library(tidyverse)
library(purrr)
library(dplyr)
library(tidyr)
library(sqldf)

install.packages("googlesheets")
library(googlesheets)

# Once the packages are installed, we can move on to the YouTube API key
To enable the APIs in Google, you have to go to the Google Developer Console (https://console.developers.google.com) and create a new project
Once you have the project created, click the "Enable APIS and Services" link and enable four APIs one at a time
1. YouTube Data API 
2. YouTube Analytics API
3. YouTube Reporting API
4. Freebase API

Then go to the credentials on the left side navigation menu and click "Create Credentials" and select the OAuth client ID option. Select 'other' as application type and give a descriptive name to the application and click Create. Click Confirm on the next screen(s) and you will get to the screen that outlines the Client ID and Client Secret. Note that down in notepad

client_id <- "XXXXXXXXX"
client_secret <- "XXXXXXXXX"

# Authenticate the application
# use the youtube oauth
yt_oauth(app_id = client_id,
app_secret = client_secret,
token = '')

On running this, a default browser window should open and ask you to sign into the Google Account you set everything in. Approve access to the application to continue

# An additional step is to authenticate access to Google Sheets as we are going to output data there 
# Authenticating and enabling the Google Sheets account for writing the final file
# Google Sheets R package usage is at https://datascienceplus.com/how-to-use-googlesheets-to-connect-r-to-google-sheets/
gs_auth(new_user = TRUE)

## Step 1 : Get all the Playlists associated with the channel
## Use https://commentpicker.com/youtube-channel-id.php to get the channel ID for ChuChuTV

PlayLists <- tuber::get_playlists(filter = 
                                            c(channel_id = "UCBnZ16ahKA2DZ_T5W0FPUXg"))

extraction_list <- list()
i <- 0


for (i in 0:29) {
  i <- i+1
  
  extraction_list[[i]]<- data.frame(playlistid = PlayLists$items[[i]]$id,
                                    playlisttitle = PlayLists$items[[i]]$snippet$title,
                                    stringsAsFactors = FALSE)
  
}

df <- data.table::rbindlist(extraction_list, fill = TRUE)


# Step 2 : Loop through the first data frame that has the playlist ids and 
#          get the video id associated with them



extraction_list <- list()
i <- 0

for (i in 1:nrow(df)) {
  
 extraction_list[[i]] <- data.frame(tuber::get_playlist_items(filter= 
                                                                 c(playlist_id = df$playlistid[[i]]),
                                                               max_results = 200),
                                     playlistid = df$playlistid[[i]]
  )
  
}

df1 <- data.table::rbindlist(extraction_list, fill = TRUE)


## Step 3 : Getting the details of the actual video

# Changing the column name for easier code reading
colnames(df1)[colnames(df1)=="contentDetails.videoId"] <- "MainVideoID"

# Converting factors to character for code loops
df1[] <- lapply(df1, as.character)

#Remove video ids that are null
df1 <- df1[complete.cases(df1$MainVideoID), ]


extraction_list <- list()
i <- 0

for (i in 1:nrow(df1)) {
  
  tryCatch({
    extraction_list[[i]] <- data.frame(tuber::get_video_details(video_id = df1$MainVideoID[[i]]),
                                       Videoid = df1$MainVideoID[[i]]
    )
  }, error = function(e) e)
  
}

df2 <- data.table::rbindlist(extraction_list, fill = TRUE)

colnames(df2)[colnames(df2)=="items.snippet.title"] <- "VideoTitle"

# Retain only required columns in df2
df2 <- select(df2, Videoid, VideoTitle)

# Converting factors to character for code loops
df2[] <- lapply(df2, as.character)


# Step 4 : Aggregate view data on videos
#Remove video ids that are null
df2 <- df2[complete.cases(df2$Videoid), ]


extraction_list <- list()
i <- 0

for (i in 1:nrow(df2)) {
  
  tryCatch({
    
    extraction_list[[i]] <- data.frame(tuber::get_stats(video_id = df2$Videoid[[i]])
    )
  }, error = function(e) e)
  
}

df3 <- data.table::rbindlist(extraction_list, fill = TRUE)

df3[] <- lapply(df3, as.character)
df3$viewCount <- as.numeric(as.character(df3$viewCount))
df3$likeCount <- as.numeric(as.character(df3$likeCount))
df3$dislikeCount <- as.numeric(as.character(df3$dislikeCount))
df3$favoriteCount <- as.numeric(as.character(df3$favoriteCount))
df3$commentCount <- as.numeric(as.character(df3$commentCount))

## Removing duplicates from data frames
df1 <- distinct(df1, MainVideoID, .keep_all = TRUE)
df2 <- distinct(df2, Videoid, .keep_all = TRUE)
df3 <- distinct(df3, id, .keep_all = TRUE)

#Renaming columns for readability of code
colnames(df1)[colnames(df1)=="contentDetails.videoPublishedAt"] <- "VideoPublishDate"

dffinal <- sqldf("Select df.*, 
                  df1.MainVideoID as VideoID,
                  df1.VideoPublishDate, 
                  df2.VideoTitle,
                 df3.viewCount, df3.likeCount, df3.dislikeCount, df3.favoriteCount, df3.commentCount
             from df left join df1 on df.playlistid = df1.playlistid
                         left join df2 on df1.MainVideoID = df2.Videoid
                         left join df3 on df1.MainVideoID = df3.id
                         ")

dffinal[is.na(dffinal)] <- 0

dffinal <- distinct(dffinal, VideoID, .keep_all = TRUE)

#Add when the data was last refreshed
dffinal$UpdateDate <- base::noquote(lubridate::today())

## Exporting to Google Sheets
ChuChuTV_Sheet <- gs_new(title = "ChuChuTVData", ws_title = "DataSheet", input = dffinal)

# Insert the rownnames vertically
gs_edit_cells(ChuChuTV_Sheet, ws = "DataSheet", 
              anchor = "L2", input = rownames(dffinal), 
              byrow = FALSE)

