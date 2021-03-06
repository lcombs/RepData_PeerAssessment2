# Mapping the Effects of Weather Events in U.S. Counties

## Synopsis

This document investigates the effect of weather events throughout counties in the United States. Three graphs are made to show which events are most frequent, most dangerous to human health, and most consequential for the economy in each county. Throughout most of the country, thunderstorms and hail tend to occur the most, tornadoes and hail tend to cause the most injuries and fatalities, and tornadoes and flooding tend to cause the most property and crop damage.

*****

## Data Processing

The steps taken for data processing and understanding the overall strucure of the data are below:

1. Import the libraries we will need to complete the assignment.


```r
library(magrittr) # pipe
library(data.table) # faster for large datasets
library(datasets) # states
library(ggplot2)
library(maps)
```

2. Read in the raw csv file.


```r
# The raw csv file is read in as a data table.
storm_data <- 
        read.csv("repdata-data-StormData.csv") %>% 
        as.data.table()
```

3. Reformat the data so that we can create maps to visualize the effects of events in the contiguous United States at the county level.


```r
# we reformat the data so we can plot the events that occur most by county.

# reformat county names so that they do not include "County"
storm_data[, county := 
                   tolower(gsub(" County, [A-Z]{2}", 
                                "",
                                storm_data$COUNTYNAME))] 

# reformat states so that they are the same abbreviations
storm_data[, state := 
        gsub("^.*([A-Z]{2}).*$", 
             "\\1",
             storm_data$STATE)]

# we will merge storm_data with counties to get lat and long
counties <- 
        map_data("county") %>% 
        as.data.table()

names(counties) <- 
        c("long", "lat", "group", "order", "state_name", "county")

# need abbreviations to merge with counties
counties$state <- 
        state.abb[match(counties$state_name,
                        tolower(state.name))]

counties[, state_name := 
                 NULL]

# to be used later to make white space borders for states on the map 
state_df <- 
        map_data("state") 

# combine counties and storm data for mapping for more accurate latitude and longitude
choropleth <- 
        merge(counties, 
              storm_data, 
              by = c("state", "county"), 
              allow.cartesian = TRUE) 

# combine relevant events together:
# not all the events seem to be unique.
levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("BLACK ICE", "ICE", "EXCESSIVE SNOW", "WINTER STORM", "ICY ROADS", 
                                    "MIXED PRECIP", "HYPOTHERMIA/EXPOSURE", "HEAVY SNOW", "GLAZE", 
                                    "HEAVY SNOW/WINTER STORM")] <-
        "WINTER"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("WINDS", "HIGH WINDS", "WIND", "HIGH WIND")] <-
        "WIND"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("EXTREME HEAT", "HEAT", "HEAT WAVE", "RECORD HEAT")] <- 
        "HEAT"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("RIP CURRENT", "RIP CURRENTS")] <- 
        "RIP CURRENT"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("FLASH FLOOD", "FLOOD", "URBAN/SML STREAM FLD", "FLOODING", 
                                    "FLASH FLOODING", "FLOOD/FLASH FLOOD", "HIGH WATER", 
                                    "HEAVY SURF", "HIGH SURF", "STORM SURGE", "COASTAL FLOODING")] <- 
        "FLOODING"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("FUNNEL CLOUD", "TORNADO", "WATERSPOUT/TORNADO", "WATERSPOUT")] <- 
        "TORNADO"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("THUNDERSTORM WIND", "THUNDERSTORM WINDS", "TSTRM WIND", 
                                    "TSTM WIND", "THUNDERSTORMW", "LIGHTNING", "TSTORM", "HEAVY RAIN")] <- 
        "TSTORM"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("HAIL", "TSTORM WIND/HAIL", "TSTM WIND/HAIL", "SMALL HAIL")] <- 
        "HAIL"

levels(choropleth$EVTYPE)[levels(choropleth$EVTYPE) %in% 
                                  c("WILDFIRE", "WILD/FOREST FIRE")] <-
        "WILDFIRE"

# order the states for mapping
choropleth <- 
        choropleth[order(choropleth$order), ] 
```

4. Calculate the highest frequency events by county.


```r
# find the top event in each county:

# calculate the number of times events occurred in each county        
choropleth[, events_by_county := 
                   .N, 
           by = c("county", "state", "EVTYPE")]

# and find which event occurred the most in each county
choropleth[, top_events := 
                   max(events_by_county), 
           by = c("county", "state")]

# subset to get the unique county-event pairs
choropleth1 <- 
        choropleth[top_events == events_by_county]
```

5. Plot the events on a map of the Unites States.


```r
# visualize the top events in each county

ggplot(choropleth1, 
       aes(x = long, y = lat, group = group)) + 
        geom_polygon(aes(fill = EVTYPE)) +
        geom_polygon(data = state_df, colour = "white", fill = NA) +
        coord_equal() +
        ggtitle("Highest Frequency Events By County")
```

![](RepData_PeerAssessment2_files/figure-html/unnamed-chunk-5-1.png) 

Now, we have a visualization of the varied events that occur in the different regions of the country. For instance, most counties in the Midwest have a high frequency of hail, while many counties in the eastern part of the country have many thunderstorms. Additionally, there is flooding in the Southwest. There are also some counties that experience other events that could be more threatening like tornado and wild fires. Over the course of the investigation, we may expect these events to have a high impact on the health and economy of the counties where they occur even more frequently than some of the more minor events.

*****

## Results

### 1. Across the United States, which types of events are most harmful with respect to population health?


```r
choropleth[, health := 
                   sum(FATALITIES, INJURIES), 
           by = c("county", "state", "EVTYPE")]

choropleth2 <- 
        choropleth[health > 0]

choropleth2[, health_county := 
                    max(health), 
            by = c("county", "state")]

choropleth2 <- 
        choropleth2[health_county == health]

ggplot(choropleth2, 
       aes(x = long, y = lat, group = group)) + 
        geom_polygon(aes(fill = EVTYPE)) +
        geom_polygon(data = state_df, colour = "white", fill = NA) +
        coord_equal() +
        ggtitle("Worst Health Events By County")
```

![](RepData_PeerAssessment2_files/figure-html/unnamed-chunk-6-1.png) 

The graph above shows the type of event that caused the most injuries and fatalities each county. Injuries and fatalities are summed to create a proxy for the population health effects. Throughout most of the country tornadoes account for the highest injury and fatality rates. The next highest events impacting health is thunderstorms. In the west, wildfires and flooding also account for many of these health effects. 

*****

### 2. Across the United States, which types of events have the greatest economic consequences?


```r
choropleth[, damage := 
                   sum(PROPDMG, CROPDMG), 
           by = c("county", "state", "EVTYPE")]

choropleth3 <- 
        choropleth[damage > 0]

choropleth3[, damage_county := 
                    max(damage), 
            by = c("county", "state")]

choropleth3 <- 
        choropleth3[damage_county == damage]

ggplot(choropleth3, 
       aes(x = long, y = lat, group = group)) + 
        geom_polygon(aes(fill = EVTYPE)) +
        geom_polygon(data = state_df, colour = "white", fill = NA) +
        coord_equal() +
        ggtitle("Worst Damage Events By County")
```

![](RepData_PeerAssessment2_files/figure-html/unnamed-chunk-7-1.png) 

The graph above shows the type of event that caused the most property and crop damage each county. The damages are summed to create a proxy for the economic effects of the event on each county. There is less variation in the events that produce great economic consequences across the nation. Logically, the events are similar to those that cause the most injuries and fatalities. Throughout much of the U.S., these events are either flooding, tornado, hail, or thunderstorms. Particularly in the center of the country, hail causes a lot of damage. In the western United States, some counties experience the most damages from wildfires. 
