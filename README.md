First install the packages if you haven't already done so:

    if (!require("tidyr")) install.packages("tidyr")
    if (!require("reshape2")) install.packages("reshape2")
    if (!require("plyr")) install.packages("plyr")

## Load the packages

    library("tidyr")
    library("reshape2")
    library("plyr")

## Loading the data
Load the data, we're using the inflammation data from the [Software Carpentry novice R lessons](https://github.com/swcarpentry/r-novice-inflammation)

    data <- read.csv("inflammation-01.csv", header=FALSE)

Each row in this data represents a patient and columns represent measurements of inflammation over time.

    dim(data)

Lets put some more informative labels on this dataframe:

    names(data) <- c(paste0("day_", 1:40))
    head(data)

Lets add patient IDs. Yes the method below is a bit more complicated than it needs to be for now, but the patientID's position will be important later

    patientID <- c(1:60)
    data <- cbind(patientID, data)
    head(data)

Now the dataframe is human-friendly, but what happens if we need to run an analysis with all the inflammation values in 1 column? (e.g. repeated measures ANOVA)
Another motivation is to have data that meets the basic tenants of well organized "tidy" data where:
- each column is a variable
- each row is an observation

To meet these criteria, our dataframe should have 3 columns (variables): patientID, day, and inflammation
We could use some "brute-force" coding to re-organize the data, but that's encouraging mistakes
With the reshape2 or tidyr packages, we have the tools to easily re-organize our data

## reshape2::melt()
First, lets try melt() from the reshape2 package:

    reshaped_data <- melt(data)
    head(reshaped_data)
    tail(reshaped_data)

This is not quite what we needed, melt() took all the columns and treated them equally because we didn't tell it that patientID is an important id variable
Lets try that again but this time specify the id variables

    reshaped_data <- melt(data, id.vars = "patientID")
    head(reshaped_data)

What if we had some other information about the patients that we think may be important?
Let's assume that each patient was given 1 of 3 different drugs. Again, this is a bit more complicated than it needs to be for now, but the drug's position will be important later

    drugs <- rep(c("A", "B", "C"), each = 20)
    data <- as.data.frame(append(data, list(drug = drugs), after = 1))
    head(data)

We now need to specify both patientID and drug as id variables

    reshaped_data <- melt(data, id.vars = c("patientID", "drug"))
    head(reshaped_data)
    tail(reshaped_data)

The variables named patientID and drug are descriptive, but variable and value are not. We can fix that in melt() 

    reshaped_data <- melt(data, id.vars = c("patientID", "drug"), variable.name = "day", value.name = "inflammation")

You can also manipulate data that is in 1 variable. In this data, we might want to seperate the numeric 'day' value from 'day_'. This would be particularly useful if the time units were not consistent (i.e. some as 'day_x' and some as 'week_x')

    day_splits <- colsplit(reshaped_data$day, "_", c("unit_time", "time"))
    head(day_splits)

## reshape2::dcast()
Now if for some reason, you did not keep the original data (e.g. hard-drive failure?) and you need the data with the original formatting back (e.g. if your colleague will only work with it in that format), we can easily re-organize the data back to the original format

    unreshaped_data <- dcast(reshaped_data, patientID+drug~day, value.var='inflammation')

A brief note on the syntax here; the value variable that gets broken up is inflammation (the column with all the inflammation values). The new column names for the 'inflammation' values comes from the 'day' column (on the right side of the ~) and the id variables (patientID+drug) are on the right side of the '~' seperated by '+'

Let's double-check that we didn't mess anything up along the way:

    identical(data,unreshaped_data)

## tidyr::gather()
The tidyr package does the same kinds of things albeit with different syntax. This is similar to melt(), it will reorganize your data in the 'long' format. Note that in this case, the ID variables (patientID and drug are left out)

tidied_data <- gather(data, day, inflammation, day_1:day_40)
head(tidied_data)
identical(tidied_data, reshaped_data)

Gather also allows the alternative syntax of using the '-' symbol to identify which variables are not to be gathered (i.e. ID variables)

    tidied_data <- gather(data, day, inflammation, -patientID, -drug)
    head(tidied_data)
    identical(tidied_data, reshaped_data)


## tidyr::separate()
Similar to `colsplit()`, `separate()` will split a variable into multiple

    day_splits <- separate(tidied_data, day, c("unit_time","time"), sep="_")

## tidyr::spread()
The tidyr package does the same kinds of things albeit with different syntax

    untidied_data <- spread(tidied_data,day, inflammation)
    head(untidied_data)
    
    identical(untidied_data, data)
    identical(untidied_data, unreshaped_data)

## Side by side comparison
Tidyr is designed specifically to tidy dataframes while reshape2 is a bit more for general reshaping

They seem to be roughly equally efficient computationally

    data_big <- rbind.fill(replicate(10000, data, simplify = FALSE))
    
    system.time(melt(data_big, id.vars=c("patientID","drug"), variable.name="day", value.name="inflammation"))
    system.time(gather(data_big, day, inflammation, day_1:day_40))

Reshape allows functions which is useful when id variables are not unique. Here we calculate the mean inflammation for each drug

    means_drug <- dcast( reshaped_data, drug ~ day, value.var="inflammation", fun.aggregate = mean)
    head(means_drug)

You can also use dcast's sybling acast() to get an array instead of a dataframe

    unreshaped_data_array <- acast(reshaped_data, patientID+drug~day, value.var='inflammation')
