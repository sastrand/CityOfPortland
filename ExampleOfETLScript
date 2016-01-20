######################################################################################################################################
## This program reads fish observation data from the FishDataEntry_2014.xls workbook and joins all result data together.            ##
## It then derives missing values and normalizes the resultant data set for inclusion in the PAWMAP database.                       ##
## SStrand                                                                                                                          ##
## City of Portland, Oregon                                                                                                         ##
## June 2015                                                                                                                        ##
######################################################################################################################################

library(xlsx)
library(plyr)
library(uuid)
library(reshape2)
library(stringr)
library(lubridate)
library(openxlsx)

# JLaw
# Creates a vector of UUIDs of length n
# requires library(uuid) 
uuid <- function(n, use.time = NA){
  ret <- replicate(n, UUIDgenerate(use.time))
  stopifnot(identical(length(unique(ret)), as.integer(n)))
  return(ret)
}

# JLaw
# Creates a template for the PAWMAP load file
# requires list "envMonFields" (below)
envMonTemplate <- function(x){
  envMonFields <- c("FeatureNamespace", "FeatureIdentifier", "ObservationUUID", "ObservationTime", "Procedure", "Observer",
                    "ObservationComment", "PhenomenonTimeStart", "PhenomenonTimeEnd", "ObservedProperty", "ComponentProperty", "ResultTimeUTC",
                    "ResultTimeLocal", "ResultTimeLocalOffset", "Result", "Unit", "Qualifier", "QualityControlLevel", "ObservationStatus")
  add <- setdiff(envMonFields, names(x))
  for (i in add){
    x[i] <- rep(NA, nrow(x))
  }
  x[, envMonFields]
}


templatePath <- "//besfile1/besinfo/emap/dataManagement/pawmapDatabase/PawmapTemplate_v1.2.5_loadsToR.xlsx"

writeToTemplate <- function(dataFrame, templatePath, destinationPath, worksheetName){
  destinationFile <- openxlsx::loadWorkbook(templatePath)
  openxlsx::writeData(destinationFile, sheet = worksheetName, dataFrame, startCol = 1, startRow = 3, colNames = FALSE, rowNames = FALSE)
  openxlsx::saveWorkbook(destinationFile, file = destinationPath)
}


###########################
######## Constants ########
###########################

# Data sets extracted from Field Ops data entry spreadsheets
pawmapFish <- xlsx::read.xlsx("//besfile1/BESINFO/EMAP/Data/Fish/FishDataEntry_2014.xls", sheetName = "PAWMAP_Fish_Occurrence", header = TRUE, colIndex = 1:7)
pawmapSites <- xlsx::read.xlsx("//besfile1/BESINFO/EMAP/Data/Fish/FishDataEntry_2014.xls", sheetName = "PAWMAP_Gear_Sheet", header = TRUE, colIndex = 1:6)
lwraFish <- xlsx::read.xlsx("//besfile1/BESINFO/EMAP/Data/Fish/FishDataEntry_2014.xls", sheetName = "LWRA_Fish_Occurrence", header = TRUE, colIndex = 1:7)
lwraSites <- xlsx::read.xlsx("//besfile1/BESINFO/EMAP/Data/Fish/FishDataEntry_2014.xls", sheetName = "LWRA_Gear_Sheet", header = TRUE, colIndex = 1:6)

# Name maps
nameMapCurrent <- c("X.location" = "FeatureIdentifier", "date" = "ObservationTime", "species" = "taxon_identity",
             "length" = "individual_length", "has_anomaly" = "presence_anomaly", "comment" = "ObservationComment",
             "start.time" = "PhenomenonTimeStart", "end.time" = "PhenomenonTimeEnd")

# Variables for subsetting
measureVars <- c("taxon_identity", "individual_length", "presence_anomaly", "is_dead")

# Object substitutions 
jointObs <- "City of Portland, Oregon: Bureau of Environmental Services: Field Operations and the Science, Fish and Wildlife Division"
FO.Obs <- "City of Portland, Oregon: Bureau of Environmental Services: Field Operations"
SciFiWi.Obs <- "City of Portland, Oregon: Bureau of Environmental Serivces: Science, Fish and Wildlife Division"
winterQtr <- 1:3
springQtr <- 4:6
summerQtr <- 7:9
fallQtr <- 10:12
fishFrom2014NameMap <- c("LampetraÂ richardsoni" = "Lampetra richardsoni", "Cottus Spp." = "Cottus")


##########################################################################
######## Joins Datasets and Assigns Procedures based on Namespace ########
##########################################################################

# Creates one data frame with all PAWMAP fish data 
pawmapFish$SiteID <- str_extract(pawmapFish$X.location, ".*[0-9]{2,4}")
pawmapSites <- rename(pawmapSites, replace = c("X.location" = "SiteID"))
pawmapAll <- merge(pawmapFish, pawmapSites, by = intersect(names(pawmapFish), names(pawmapSites)), all.x = TRUE)
pawmapAll$FeatureNamespace <- "PAWMAP"
pawmapAll$Procedure[!(pawmapAll$is_wadeable)] <- "NRSA Non-wadeable electrofishing protocol (2013, section 10)"
pawmapAll$Procedure[pawmapAll$is_wadeable] <- "NRSA Wadeable electrofishing protocol (2013, section 10)"

# Creates one data frame with all LWRA fish data
lwraFish$SiteID <- str_extract(lwraFish$X.location, "Will-[0-9]{2}")
lwraSites <- rename(lwraSites, replace = c("location" = "SiteID"))
lwraAll <- merge(lwraFish, lwraSites, by = intersect(names(lwraFish), names(lwraSites)), all.x = TRUE)
lwraAll$FeatureNamespace <- "LWRA"
lwraAll$Procedure <- "Internal Memo Regarding Application of PAWMAP to Mainstem Willamette"

# creates combined dataframe of 2014 LWRA and PAWMAP data
current <- rbind.fill(pawmapAll, lwraAll)
current <- rename(current, replace = nameMapCurrent)


####################################################################################
######## Normalizes Timestamps and FeatureIdentifiers and Assigns Observers ########
####################################################################################

# Performs dataset-wide modifications
current$QualityControlLevel <- "Quality Controlled Data"
current$ObservationStatus <- "Accepted"
current$ObservationUUID <- uuid(nrow(current))
current$taxon_identity <- revalue(current$taxon_identity, fishFrom2014NameMap)

# Normalizes timestamps
current$PhenomenonTimeStart[is.na(current$PhenomenonTimeStart)] <- "1200"
current$PhenomenonTimeEnd[is.na(current$PhenomenonTimeEnd)] <- "1200"
current$PhenomenonTimeStart[str_length(current$PhenomenonTimeStart) == 3] <- paste0("0", current$PhenomenonTimeStart[str_length(current$PhenomenonTimeStart) == 3])
current$PhenomenonTimeEnd[str_length(current$PhenomenonTimeEnd) == 3] <- paste0("0", current$PhenomenonTimeEnd[str_length(current$PhenomenonTimeEnd) == 3])
current$PhenomenonTimeStart <- paste0(current$ObservationTime, " ", current$PhenomenonTimeStart)
current$PhenomenonTimeEnd <- paste0(current$ObservationTime, " ", current$PhenomenonTimeEnd)
current$ObservationTime <- as.POSIXct(current$ObservationTime, tz = "America/Los_Angeles")
hour(current$ObservationTime) <- 12

# Normalizes FeatureIdentifiers
current$Transect <- str_extract(current$FeatureIdentifier, pattern = "[ABCDEFGHIJK]{2}$")
current$Transect <- str_replace(current$Transect, pattern = "[ABCDEFGHIJK]{1}$", replace = "")
current$FeatureIdentifier <- str_replace(current$FeatureIdentifier, pattern = "[ABCDEFGHIJK]{2}$", replace = "")
current$FeatureIdentifier[!(is.na(current$Transect))] <- 
  paste0(current$FeatureIdentifier[!(is.na(current$Transect))], ".T", current$Transect[!(is.na(current$Transect))])

# Assigns Observer
current$Observer[current$FeatureNamespace == "LWRA"] <- jointObs
current$Observer[current$FeatureNamespace == "PAWMAP" & month(current$ObservationTime) %in% c(summerQtr)] <- FO.Obs
current$Observer[current$FeatureNamespace == "PAWMAP" & !(current$is_wadeable) & month(current$ObservationTime) %in% c(winterQtr, springQtr, fallQtr)] <- jointObs
current$Observer[current$FeatureNamespace == "PAWMAP" & current$is_wadeable & month(current$ObservationTime) %in% c(winterQtr, springQtr, fallQtr)] <- SciFiWi.Obs


#############################################################################################
######## Performs Melt of Observation Component Properties and Exports Result to csv ########
#############################################################################################

# Performs Melt
current$taxon_identity <- as.character(current$taxon_identity)
current$individual_length <- as.character(current$individual_length)
current <- melt(current, measure.vars = measureVars)
current <- rename(current, replace = c("variable" = "ComponentProperty", "value" = "Result"))
current$Unit <- NA
current$Unit[current$ComponentProperty == "individual_length"] <- "mm"
current$ObservedProperty <- "fish_occurrence"

# Formats and exports Observation table to CSV
current <- envMonTemplate(current)
writeToTemplate(current, "c:/users/sstrand/documents/r/output/fish2014ToTempalte3.xlsx", "Observations")
