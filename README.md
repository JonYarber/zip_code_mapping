
# Zip Code Mapping

## Context

I work as a contractor for the Air Force Medical Command’s (AFMEDCOM)
Studies and Analysis Division (A9), where we provide analytical support
for healthcare-related inquiries concerning the Air Force. One of our
key projects involves examining the types of care that active duty
servicemembers seek off base, which we refer to as Private Sector Care.
While we have access to Private Sector Care data, there is no efficient
way to filter this data to include only the records for care performed
within a certain radius of any given installation or military treatment
facility (MTF). With millions of records generated each fiscal year and
a focus on five years of data for this project, it is critical that we
filter the results to include only the specific records we need during
our initial processing.

One piece of information included in the data is the zip code of the
healthcare professional from whom care was sought. We determined that an
effective way to filter the data is to identify all zip codes within a
certain radius of a given MTF and only pull records where the healthcare
provider’s zip code matches one of those identified. While this method
may not be perfect, it successfully reduces the dataset to a more
manageable number of records.

This provided a clear outline of our methodology:

1)  Identify all zip codes within a specified radius of the MTF.
2)  Isolate all healthcare providers whose zip codes match those
    identified in step one.
3)  Extract the relevant records for these providers from the dataset
    and document all associated ICD-10 codes for visits by active duty
    servicemembers.

## Challenge

The entire project is primarily focused on step 1 of the methodology.
Several challenges and questions arose during this process:

1)  How do we get a *free* list of all U.S. zip codes (approximately
    43,000)?
    - While it is relatively easy to obtain a list of residential and
      deliverable zip codes (around 33,000), acquiring the remaining
      types (such as special, PO Box, and military zip codes) typically
      requires a paid subscription. These lists can be quite expensive,
      and it wouldn’t be cost-effective to purchase them for a one-time
      use. Therefore, I needed to find an alternative way to generate
      this list at no cost.
2)  How do we generate geocoordinates for the zip codes?
    - To determine if a zip code falls within the radius of an MTF, we
      need geocoordinates (latitude and longitude). The challenge is
      finding a way to obtain these coordinates **without requiring an
      API key**. Ensuring code repeatability is crucial for my contract,
      and it would be inefficient to ask users to generate an API key to
      execute the code.
3)  How do we *efficiently* find which of the approximately 43,000 zip
    codes lie within a radius of the MTF?
    - We need a method to assess the proximity of these zip codes to the
      MTF without having to compare each one individually, which would
      be overly time-consuming and impractical.

## Solution

Here is how I went about addressing all these challenges and the code I
used to do it:

#### Generate a list of every possible zip code combination (100,000)

For this solution, we will utilize the
[tidygeocoder](https://jessecambon.github.io/tidygeocoder/) package,
which I cannot recommend highly enough. It provides access to several
geocoders without requiring an API key. I was able to use three
different geocoders to obtain my final results, but the one that yielded
the best outcomes and for which I’ll be providin an example was the
ArcGIS geocoder.

Each geocoder has specific formatting requirements for addresses to
accurately determine latitude and longitude. The ArcGIS geocoder, for
instance, preferred the inclusion of “US” in the street address whereas
the Nominatim geocoder necessitated that the <code>country</code>
parameter be specified. As a result, when constructing the dataframe, I
appended “, US” to the zip code and included a country column.

``` r
suppressPackageStartupMessages(library(tidyverse))

zips <- 
  data.frame(zip_code = sapply(0:99999, function(x) str_pad(x, side = 'left', pad = '0', width = 5))) %>%
  # Need address column with U.S. designation for geocoding
  mutate(address = paste0(zip_code, ', US'),
         country = 'US')

# View result
head(zips)
```

      zip_code   address country
    1    00000 00000, US      US
    2    00001 00001, US      US
    3    00002 00002, US      US
    4    00003 00003, US      US
    5    00004 00004, US      US
    6    00005 00005, US      US

#### Use tidygeocoder to determine which of the zip code combination are valid U.S. zip codes.

With a comprehensive list of all possible zip code combinations, the
next step was to determine which ones are actually valid.

I discovered that by enabling the <code>full_results</code> parameter of
the ArcGIS geocoder, I could access a **score** variable. This score
indicates how confident ArcGIS is in the accuracy of the geocoordinates
it provides. After checking several results with a score of 100, I found
them to be accurate. Consequently, I filtered the geocoder results to
include only those with a score of 100. This resulted in the approximate
number of valid zip codes I needed: around 41,701.

*This block of code takes several hours to run*

``` r
zips_arcgis <-
        zips %>%
          tidygeocoder::geocode(method = 'arcgis',
                                # 'address' from dataframe
                                address = 'address',
                                # Returns longitude as 'lon' instead of 'long'
                                long = 'lon',
                                full_results = T) %>%
            filter(score == 100) %>%
            select(zip_code, lat, lon)

# View results
head(zips_arcgis, 10)
```

       zip_code      lat       lon
    1     00208 33.02149 -97.28233
    2     00209 36.88192 -76.20038
    3     00501 40.81680 -73.04507
    4     00544 40.81723 -73.04515
    5     00601 18.15860 -66.71886
    6     00602 18.38064 -67.19000
    7     00603 18.44903 -67.13793
    8     00604 18.49386 -67.14747
    9     00605 18.44508 -67.14133
    10    00606 18.18126 -66.98013

3)  Given an address, subset the zip codes to find those within 1
    latitude and 1 longitude of the address.
4)  Find the distances from the address to zip codes.
5)  Filter distances to desired radius.
