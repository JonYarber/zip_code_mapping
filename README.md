
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

This project is essential step 1 of of the outlined in the methodology.
A few challenges and questions arose during this process:

#### 1) How do we get a *free* list of all U.S. zip codes (approximately 41,600)?

- While it is relatively easy to obtain a list of residential and
  deliverable zip codes (around 33,800), acquiring the remaining types
  (such as special, PO Box, and military zip codes) typically requires a
  paid subscription. These lists can be quite expensive, and it wouldn’t
  be cost-effective to purchase them for a one-time use. Therefore, I
  needed to find an alternative way to generate this list at no cost.

#### 2) How do we generate geocoordinates for the zip codes?

- To determine if a zip code falls within the radius of an MTF, we need
  geocoordinates (latitude and longitude). The challenge is finding a
  way to obtain these coordinates **without requiring an API key**.
  Ensuring code repeatability is crucial for my contract, and it would
  be inefficient to ask users to generate an API key to execute the
  code.

#### 3) How do we *efficiently* find which of the approximately 41,600 zip codes lie within a radius of the MTF?

- We need a method to assess the proximity of these zip codes to the MTF
  without having to compare each one individually, which would be overly
  time-consuming and impractical.

## Solution

Here is how I went about addressing all these challenges and the code I
used to do it:

#### 1) Generate a list of every possible zip code combination (100,000)

For this solution, we will utilize the
[tidygeocoder](https://jessecambon.github.io/tidygeocoder/) package,
which I cannot recommend highly enough. It provides access to several
geocoders without requiring an API key. I was able to use three
different geocoders to obtain my final results, but the one that yielded
the best outcomes and for which I’ll be providing an example was the
ArcGIS geocoder.

Each geocoder has specific formatting requirements for addresses to
accurately determine latitude and longitude. The ArcGIS geocoder, for
instance, preferred the inclusion of “US” in the street address whereas
the Nominatim geocoder necessitated that the <code>country</code>
parameter be specified. As a result, when constructing the dataframe, I
appended “, US” to the zip code and included a *country* column.

``` r
suppressPackageStartupMessages(library(tidyverse))

zip_codes <- 
  data.frame(zip_code = sapply(0:99999, function(x) str_pad(x, side = 'left', pad = '0', width = 5))) %>%
  # Need address column with U.S. designation for geocoding
  mutate(address = paste0(zip_code, ', US'),
         country = 'US')

# View result
head(zip_codes)
```

      zip_code   address country
    1    00000 00000, US      US
    2    00001 00001, US      US
    3    00002 00002, US      US
    4    00003 00003, US      US
    5    00004 00004, US      US
    6    00005 00005, US      US

#### 2) Use tidygeocoder to determine which of the zip code combination are valid U.S. zip codes.

With a comprehensive list of all possible zip code combinations, the
next step was to determine which ones are actually valid.

I discovered that by enabling the <code>full_results</code> parameter of
the ArcGIS geocoder, I could access a **score** variable. This score
indicates how confident ArcGIS is in the accuracy of the geocoordinates
it provides. After checking several results with a score of 100, I found
them to be accurate. Consequently, I filtered the geocoder results to
include only those with a score of 100. This resulted in the approximate
number of valid zip codes I needed: 41,071.

*Note that this block of code takes several hours to run*

``` r
zips_gis <-
    zips %>%
        tidygeocoder::geocode(method = 'arcgis',
                              # 'address' from dataframe
                              address = 'address',
                              full_results = T) %>%
        filter(score == 100) %>%
        select(zip_code,
               zip_lat = truncate_digits(lat), 
               zip_lon = truncate_digits(long))

# View results
head(zips_gis, 10)
```

       zip_code zip_lat  zip_lon
    1     00208 33.0214 -97.2823
    2     00209 36.8819 -76.2003
    3     00501 40.8167 -73.0450
    4     00544 40.8172 -73.0451
    5     00601 18.1585 -66.7188
    6     00602 18.3806 -67.1899
    7     00603 18.4490 -67.1379
    8     00604 18.4938 -67.1474
    9     00605 18.4450 -67.1413
    10    00606 18.1812 -66.9801

    Number of zip codes geocoded: 41,071

It was not essential for me to obtain every single zip code; however,
after processing the zip codes that did not return a score of 100 or
returned no results through two additional geocoders (Nominatim and
Census), I ended up with 43,003 geocoded zip codes. Given that zip codes
are constantly changing, having more zip codes than needed is both
reassuring and necessary as there are old zip codes in the data as well.

I have provided provided this data in the repo for those who wish to
utilize it for their own projects. As mentioned, this is not typically
available for free.

#### 3) Find all zip codes within 1 latitude and 1 longitude of an address.

The next step was to find an efficient way to narrow down the zip codes
that are close to a given address, without having to calculate the exact
distances between each address and the 43,000 zip codes. To achieve
this, I simply found all zip codes within a latitude (approximately 69
miles) and a longitude (approximately 55 miles) of the given address.
This approach resulted in a manageable number of zip codes for each
address, allowing me to calculate distances and apply further filtering
later.

Tidygeocoder provides a list of some addresses we can use for this
exercise

``` r
addresses <- as.data.frame(tidygeocoder::sample_addresses)
```

                      name                                            addr
    1          White House         1600 Pennsylvania Ave NW Washington, DC
    2 Transamerica Pyramid      600 Montgomery St, San Francisco, CA 94111
    3    NY Stock Exchange              11 Wall Street, New York, New York
    4         Willis Tower              233 S Wacker Dr, Chicago, IL 60606
    5    Chateau Frontenac 1 Rue des Carrieres, Quebec, QC G1R 4P5, Canada
    6            Nashville                                   Nashville, TN
    7              Nairobi                                  Nairobi, Kenya
    8             Istanbul                                Istanbul, Turkey
    9                Tokyo                                    Tokyo, Japan

Since we’re only looking at United States zip codes, we only need to
geocode the first 4 address.

``` r
addresses_gis <- 
    addresses %>%
        slice(1:4) %>%
        tidygeocoder::geocode(method = 'arcgis', 
                              address = 'addr',
                              full_results = T) %>%
        filter(score == 100) %>%
        select(address = addr, 
               address_lat = lat, 
               address_lon = long)
```

    Passing 4 addresses to the ArcGIS single address geocoder

    Query completed in: 1.6 seconds

    # A tibble: 4 × 3
      address                                    address_lat address_lon
      <chr>                                            <dbl>       <dbl>
    1 1600 Pennsylvania Ave NW Washington, DC           38.9       -77.0
    2 600 Montgomery St, San Francisco, CA 94111        37.8      -122. 
    3 11 Wall Street, New York, New York                40.7       -74.0
    4 233 S Wacker Dr, Chicago, IL 60606                41.9       -87.6

For each of these coordinates, we now subset the zip code data frame to
those whose centroids lie within a latitude or longitude of the
coordinates.

``` r
# Initiate an empty dataframe to concatenate results
# This will be the final dataframe
radius_df <- data.frame(address = character(0),
                        address_lat = numeric(0),
                        address_lon = numeric(0),
                        zip_code = character(0),
                        zip_lat = numeric(0),
                        zip_lon = numeric(0))

# For each address in the addresses_gis dataframe
for(row in 1:nrow(addresses_gis)){
    
    # Save the address and coordinates
    address <- addresses_gis %>% slice(row) %>% pull(address)
    address_lat <- addresses_gis %>% slice(row) %>% pull(address_lat)
    address_lon <- addresses_gis %>% slice(row) %>% pull(address_lon)
    
    # Filter the zips_gis dataframe to those within 1 latitude and longitude of the address
    radius_zips <-
        zips_gis %>%
            filter(zip_lat < (address_lat + 1) & zip_lat > (address_lat - 1),
                   zip_lon < (address_lon + 1) & zip_lon > (address_lon - 1))
    
    # Add the address and coordinates to the radius_dataframe
    radius_zips$address <- address
    radius_zips$address_lat <- address_lat
    radius_zips$address_lon <- address_lon
  
    # Append results
    radius_df <- bind_rows(radius_df, radius_zips)
  
}

# How many results did we return per address?
radius_df %>%
  group_by(address) %>%
  summarize(num_zips = n())
```

    # A tibble: 4 × 2
      address                                    num_zips
      <chr>                                         <int>
    1 11 Wall Street, New York, New York             1411
    2 1600 Pennsylvania Ave NW Washington, DC        1140
    3 233 S Wacker Dr, Chicago, IL 60606              549
    4 600 Montgomery St, San Francisco, CA 94111      579

<!--Cleanup-->

``` r
# Let's look at 5 random results
radius_df[sample(nrow(radius_df), 5),] 
```

                                         address address_lat address_lon zip_code
    1075 1600 Pennsylvania Ave NW Washington, DC     38.8976    -77.0365    22714
    1928      11 Wall Street, New York, New York     40.7071    -74.0108    07096
    1773      11 Wall Street, New York, New York     40.7071    -74.0108    06763
    2445      11 Wall Street, New York, New York     40.7071    -74.0108    10040
    439  1600 Pennsylvania Ave NW Washington, DC     38.8976    -77.0365    20646
         zip_lat  zip_lon
    1075 38.5000 -77.8990
    1928 40.7742 -74.0632
    1773 41.6815 -73.2171
    2445 40.8572 -73.9300
    439  38.5248 -76.9872

#### 4) Find the distances from the address to zip codes

With 43,000 zip codes narrowed down to 3,700 we can quickly find the
distances from address to the zip code centroid. For the distances, we
will use the geodesic distance (shortest distance between two points on
a sphere) with the <code>distGeo</code> function from the
[geosphere](https://cran.r-project.org/web/packages/geosphere/geosphere.pdf)
package.

``` r
library(geosphere)

# Initiate an empty array to store the distances
distances <- c()

# Iterate through the radius_df dataframe and calculate the distance between address and zip code
for(row in 1:nrow(radius_df)){
  
  # Store the address coordinates  
  # IMPORTANT: While we typically describe coordinates as (latitude, longitude) the distGeo function
  #            requires them to be passed as (longitude, latitude)!!!
  coord1 <- radius_df %>% slice(row) %>% select(address_lon, address_lat)
  
  # Store zip code coordinates
  coord2 <- radius_df %>% slice(row) %>% select(zip_lon, zip_lat)
        
  # Find distance between two coordinates
  # Distance generated is in meters and requires conversion to miles
  distance <- distGeo(coord1, coord2) / 1609.3445
        
  # Add to distances array
  distances <- append(distances, distance)
  
}

# Add distances to the dataframe
radius_df$distance <- round(distances, 2)


radius_df[sample(nrow(radius_df), 5), ]
```

                                    address address_lat address_lon zip_code
    2317 11 Wall Street, New York, New York     40.7071    -74.0108    08723
    3532 233 S Wacker Dr, Chicago, IL 60606     41.8787    -87.6358    60540
    2550 11 Wall Street, New York, New York     40.7071    -74.0108    10285
    2076 11 Wall Street, New York, New York     40.7071    -74.0108    07701
    1827 11 Wall Street, New York, New York     40.7071    -74.0108    06896
         zip_lat  zip_lon distance
    2317 40.0440 -74.1278    46.17
    3532 41.7623 -88.1532    27.89
    2550 40.7138 -74.0140     0.49
    2076 40.3522 -74.0731    24.71
    1827 41.3097 -73.3977    52.50

<!--Cleanup-->

#### 5) Filter distances to desired radius.

``` r
cat(radius_df %>%
      filter(distance <= 20) %>%
      nrow())
```

    1457
