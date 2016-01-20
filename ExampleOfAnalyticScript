##################################################################################################################################
## This program loads observation and permit data from the fish-specific data entry form and exports a permit-compliance report ##
## for the 2015 Carp Study Permit.                                                                                              ##
## SStrand                                                                                                                      ##
## City of Portland, Oregon                                                                                                     ##
## June 2015                                                                                                                    ##
##################################################################################################################################

library(openxlsx)
library(stringi)
library(plyr)
library(XLConnect)
library(stringr)
library(reshape2)
library(lubridate)

##########################
####### Functions ########
##########################

#' Load data entry tables
#' 
#' This function loads a data-entry worksheet from Excel. 
#' 
#' This function contains parameters designed to standardize variations in data entry and set desired variable types.
#' @param filePath the path to the workbook as a string
#' @param sheetNumber the sheet number within the workbook to load
#' @param headerRow the row number to be used as a header (anything above it won't load.)
#' @param allCaps the name of a column or set of columns in which all the letters will be capitalized. Designed for site ID standardization.
#' @param dateCol the name of a column or set of columns containing values recognized by Excel as dates. 
#' @param timeCol the name of a column or set of columns containing four-digit times occuring within the date in dateCol. If there is more than one variable in dateCol, the first will be joined to the times listed here.
#' @param timeZone the tz that timeCol times reflect. Default is "America/Los_Angeles".
#' 
DataEntryLoader <- function(filePath, sheetNumber, headerRow, allCaps, dateCol, timeCol, timeZone){
  ret <- openxlsx::read.xlsx(filePath, sheetNumber, startRow = headerRow, colNames = TRUE, rowNames = FALSE)
  if(!(missing(dateCol))){
    for(i in dateCol){
      ret[[i]] <- as.numeric(ret[[i]])
      ret[[i]] <- as.Date(ret[[i]], origin = "1899-12-30")
    }
  }
  if(!(missing(allCaps))){
    for(i in allCaps){
      ret[[i]] <- stri_trans_toupper(ret[[i]])
    }
  } 
  if(!(missing(timeCol))){
    date <- dateCol[[1]]
    for(i in timeCol){
      ret[[i]] <- paste0(ret[[date]], " ", ret[[i]])
      ret[[i]] <- strptime(ret[[i]], format = "%Y-%m-%d %H%M", tz = timeZone)
    }
  }
  return(ret)
}

DateCompliant <- function(date, start, end){
  range <- new_interval(start = start, end = end, tzone = "America/Los_Angeles")
  ret <- date %within% range
  ret
}

##########################
####### Constants ########
##########################

# Object substitutions
entryWorkbookPath <- "//BESFILE1/besinfo/emap/data/fish/FishDataEntry_2015.xlsx"
exportReportPath <- "//besfile1/besinfo/emap/data/fish/fishreportoutput/"
observationVars <- c("DateCompliant", "common_name", "CarpStudyPermitName", "DefaultESU", "clipped", "is_dead", "lifestage", "take", "gear", "StreamCode")
permitVars <- c("SPECIES", "LISTING.UNIT/STOCK", "PRODUCTION/ORIGIN", "LIFESTAGE", "TAKE.ACTION", "OBSERVE/COLLECT.METHOD", "StreamCode")
problemFishVars <- c("common_name", "SPECIES", "LISTING.UNIT/STOCK", "PRODUCTION/ORIGIN", 
                     "LIFESTAGE", "TAKE.ACTION", "OBSERVE/COLLECT.METHOD", "StreamCode", 
                     "ActualTake", "ActualMortality")
reportExportVars <- c("LINE", "VERSION", "SPECIES", "common_name", "LISTING.UNIT/STOCK", "PRODUCTION/ORIGIN", "LIFESTAGE", "TAKE.ACTION", "OBSERVE/COLLECT.METHOD", "PROCEDURES", 
                      "StreamCode", "EXPECTED.TAKE", "ActualTake", "EXPECTED.MORTALITY", "ActualMortality", "PctMortalityPerTake")
whatFishIsPermit <- c("SPECIES", "LISTING.UNIT/STOCK", "PRODUCTION/ORIGIN", "LIFESTAGE", "StreamCode")
whatFishIsDataEntry <- c("PawmapPermitName", "DefaultESU", "clipped", "lifestage", "StreamCode")

# Name maps
takeMap <- c("Capture, Handle, Release" = "Capture/Handle/Release Fish", 
             "Mark/Tag/Sample Tissue, Release" = "Capture/Mark, Tag, Sample Tissue/Release Live Animal",
             "Whole Fish Tissue Sample" = "Intentional (Directed) Mortality", 
             "Euthanize" = "Intentional (Directed) Mortality")
gearMap <- c("Boat Electrofisher" = "Electrofishing, Boat", "Backpack Electrofisher" = "Electrofishing, Backpack",
             "Modified Boat Electrofisher" = "Electrofishing, Boat Modified", "Modified Backpack Electrofisher" = "Electrofishing, Backpack Modified",
             "Seine Net" = "Net, Seine", "Gill Net" = "Net, Gill")
reportRename <- c("CarpStudyPermitName" = "SPECIES", "take" = "TAKE.ACTION", "gear" = "OBSERVE/COLLECT.METHOD", "clipped" = "PRODUCTION/ORIGIN",
                  "DefaultESU" = "LISTING.UNIT/STOCK", "FALSE" = "ActualTake", "TRUE" = "ActualMortality", "lifestage" = "LIFESTAGE")

##########################################
######## Loads and joins tables ##########
##########################################

fish <- DataEntryLoader(entryWorkbookPath, 4, 2, "site_ID", "date")
gear <- DataEntryLoader(entryWorkbookPath, 2, 2, "site_ID", "date", c("start_time", "end_time"), "America/Los_Angeles")
permit <- DataEntryLoader(entryWorkbookPath, sheet = 10, headerRow = TRUE, dateCol = c("BEGIN.DATE", "END.DATE"))

# Loads lookup tables from named ranges
defaultESU <- XLConnect::readNamedRegionFromFile(entryWorkbookPath, name = "DefaultESU", header = TRUE)
permitTaxa <- XLConnect::readNamedRegionFromFile(entryWorkbookPath, name = "PermitTaxa", header = TRUE)
streamCode <- XLConnect::readNamedRegionFromFile(entryWorkbookPath, name = "StreamCode", header = TRUE)
streamCode <- streamCode[streamCode$PermitReport == "Columbia Slough Sed_Fish", ]

# Joins observations to named ranges and gear form
fish <- merge(fish, gear, by = c("site_ID", "date"), all.x = TRUE)
fish <- merge(fish, permitTaxa, by.x = "common_name", by.y = "CommonName", all.x = TRUE)
fish <- merge(fish, defaultESU, by.x = "common_name", by.y = "CommonName", all.x = TRUE)
fish <- merge(fish, streamCode, by = "site_ID")


###################################################
######## Assigns assumed values and names  ########
###################################################

# Assigns assumed values
fish$take[is.na(fish$take)] <- "Capture, Handle, Release"
fish$lifestage[!(fish$common_name %in% c("Chinook Salmon", "Steelhead", "Coho Salmon"))] <- "Adult"
fish$clipped[!(fish$common_name %in% c("Chinook Salmon", "Steelhead", "Coho Salmon"))] <- "Natural"
permit <- rename(permit, replace = c("INDIRECT.MORTALITY" = "EXPECTED.MORTALITY"))

# Revalues observations to match permit allocation tables
fish$gear <- revalue(fish$gear, replace = gearMap)
fish$take <- revalue(fish$take, replace = takeMap)


# adds date compliance variable to report
dateRanges <- permit[c(whatFishIsPermit, "BEGIN.DATE", "END.DATE")]
fish <- merge(fish, dateRanges, by.x = whatFishIsDataEntry, by.y = whatFishIsPermit, all.x = TRUE, all.y = FALSE)
fish$DateCompliant <- DateCompliant(date = fish$date, start = fish$BEGIN.DATE, end = fish$END.DATE)


################################################
######### Generates and exports reports ########
################################################

# generates and exports names of files containing observations of ESA listed taxa
ESAlisted <- fish[!(is.na(fish$DefaultESU)) & fish$DefaultESU != "Coastal", c("site_ID", "date")]
if(nrow(ESAlisted) == 0){ 
  ESAlisted <- "no files"
  } else 
    ESAlisted <- unique(paste0(ESAlisted, "_", ESAlisted$date, ".pdf"))
    ESAlisted <- str_replace_all(ESAlisted, "-", "")
write.table(ESAlisted, paste0(exportReportPath, "CarpPermit_ESAFilePaths.csv"), row.names = FALSE, col.names = FALSE)

# counts instances of actual take and actual mortality
report <- fish[, observationVars]
report <- ddply(report, .(DateCompliant, common_name, CarpStudyPermitName, DefaultESU, clipped, is_dead, lifestage, take, gear, StreamCode), nrow)
report <- dcast(report, ... ~ is_dead)
report <- rename(report, replace = reportRename)
report <- merge(report, permit, by = permitVars, all.x = TRUE, all.y = TRUE)

# generates and exports a list of "unbinned" fish, those that are observed but not listed as an instance on the permit. 
problemFish <- report[is.na(report$EXPECTED.TAKE) || !(report$DateCompliant), ]
problemFish <- problemFish[, problemFishVars]
problemFish$ActualMortality[is.na(problemFish$ActualMortality)] <- 0
if(nrow(problemFish) > 0) write.csv(problemFish, paste0(exportReportPath, "CarpPermit_UnbinnedFish.csv"), row.names = FALSE)
if(nrow(problemFish) == 0) {
  problemFish <- "All fish observations were matched to a permit category." 
  write.table(problemFish, paste0(exportReportPath, "CarpPermit_UnbinnedFish.csv"), row.names = FALSE, col.names = FALSE)
} 

# standardizes and exports compliance report
report <- report[!(is.na(report$EXPECTED.TAKE)), ]
report$ActualTake[is.na(report$ActualTake)] <- 0
report$ActualMortality[is.na(report$ActualMortality)] <- 0
report$ActualTake <- report$ActualTake + report$ActualMortality
report$PctMortalityPerTake <- round((report$ActualMortality/report$ActualTake) * 100)
report$PctMortalityPerTake[is.na(report$PctMortalityPerTake)] <- 0
report$TotalAllowedMortality[report$TAKE.ACTION != "Intentional (Directed) Mortality"] <- trunc(0.03 * report$ActualTake[report$TAKE.ACTION != "Intentional (Directed) Mortality"])
report$TotalAllowedMortality[report$TAKE.ACTION == "Intentional (Directed) Mortality"] <- report$ActualTake[report$TAKE.ACTION == "Intentional (Directed) Mortality"]
report$PctMortalityPerTake[report$PctMortalityPerTake == 0] <- ""
report <- report[, reportExportVars]
write.csv(report, paste0(exportReportPath, "CarpPermit_FishingCompliance.csv"), row.names = FALSE)

# generates list of all sites and panels sampled 
allSampled <- merge(gear, streamCode, by = "site_ID", all.x = TRUE, all.y = FALSE)
allSampled <- allSampled[c("site_ID", "date", "Panel", "StreamCode", "no_fish")]
write.csv(allSampled, paste0(exportReportPath, "CarpStudy_SitesVisited.csv"), row.names = FALSE)

#############################
######## Woot, done! ########
#############################
