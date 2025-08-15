
<!--Set environment and load needed functions-->

# ZIP Code Mapping

## Context

I previously worked as a contractor for the Air Force Medical Command’s
(AFMEDCOM) Studies and Analysis Division (A9), providing analytical
support for healthcare-related inquiries concerning the Air Force. One
of our key projects involved examining the types of care that active
duty service members seek off base, referred to as Private Sector Care.
While we had access to Private Sector Care data, there was no efficient
way to filter this data to include only the records for care performed
within a certain radius of any given installation or military treatment
facility (MTF). With millions of records generated each fiscal year and
a focus on five years of data for this project, it was critical to
filter the results to include only the specific records we needed during
our initial processing.

One piece of information in the data was the ZIP code of the healthcare
professional from whom care was sought. We determined that an effective
way to filter the data is to identify all ZIP codes within a certain
radius of a given MTF and only examine records where the healthcare
provider’s ZIP code matched one of those identified. While this method
may not be perfect, it successfully reduced the data set to a more
manageable number of records.

This provided a clear outline of our methodology:

1)  Identify all ZIP codes within a specified radius of an MTF.
2)  Isolate all healthcare providers whose ZIP codes match those
    identified in Step 1.
3)  Extract the relevant records for these providers from the data set
    and document all associated ICD-10 codes for visits by active duty
    service members.

## Challenges

This project focuses on Step 1 of the outlined methodology—produce a
list of all U.S. ZIP codes within a specified radius of our MTFs of
interest. A few challenges arose during this process:

#### 1) Acquiring a <i>free</i> list of all ~41,600 U.S. ZIP codes

• While it is relatively easy to obtain a list of residential and
deliverable ZIP codes (~33,800), acquiring the remaining types such as
special, PO Box, and military ZIP codes typically requires a paid
subscription. These lists can be quite expensive, and it wasn’t
cost-effective to purchase them for a one-time use. Therefore, I needed
to find an alternative way to generate the complete list at no cost.

#### 2) Generating geographic coordinates for the ZIP codes

• To determine if a ZIP code falls within the radius of an MTF, I needed
geographic coordinates. The challenge was finding a way to obtain these
coordinates **without requiring an API key**. Code repeatability was
critical, and it would be inefficient to ask future users to create API
keys before running the process.

#### 3) <i>Efficiently</i> finding ZIP codes within a given radius

• I needed a method of assessing the proximity of ZIP codes to an MTF
without comparing each one individually, which would have been
time-consuming and impractical.

#### 4) Dealing with outdated or incompatible packages

• Many R packages for this task were incompatible with my environment,
and the ones I could install contained outdated ZIP code data. I recall
one package using a ZIP code list from as far back as 2008. Since ZIP
codes change frequently, it was important that this process always use
the most up-to-date information when run.

## Solution

Here’s how I addressed all these challenges, achieved the required
results, and the code I used to accomplish it:

#### 1) Generate a list of every possible ZIP code combination.

``` r
library(tidyverse)

# Produce a list from 00000 to 99999
zip_codes <- data.frame(zip_code = sapply(0:99999, 
                                          \(x) str_pad(x, 
                                                       side = 'left', 
                                                       pad = '0', 
                                                       width = 5)))
```

#### 2) Use <code>tidygeocoder</code> package to determine which of the ZIP code combinations are valid U.S. ZIP codes.

I cannot recommend the
[tidygeocoder](https://jessecambon.github.io/tidygeocoder/) package
highly enough. It provided access to several geocoders which do not
require an API key, along with various methods to validate the results.
To obtain my final result, I used two geocoders: ArcGIS and Nominatim.
ArcGIS yielded the majority of the results, so I will use this one as an
example.

Each geocoder has specific formatting requirements to accurately
determine latitude and longitude. The ArcGIS geocoder, for instance,
produced better results when “US” was added to the input
<code>address</code> field. In contrast, the Nominatim geocoder allows
the use of <code>postalcode</code> and <code>country</code> parameters.
Therefore, when constructing the data frame, I created an
<code>address</code> column for the ArcGIS geocoder and a
<code>country</code> column for the Nominatim geocoder.

``` r
zip_codes <- 
    zip_codes %>%
        mutate(address = paste0(zip_code, ', US'),
               country = 'US')

# View result
head(zip_codes)
```

| zip_code |  address  | country |
|:--------:|:---------:|:-------:|
|  00000   | 00000, US |   US    |
|  00001   | 00001, US |   US    |
|  00002   | 00002, US |   US    |
|  00003   | 00003, US |   US    |
|  00004   | 00004, US |   US    |
|  00005   | 00005, US |   US    |

I discovered that by enabling the <code>full_results</code> parameter of
the ArcGIS geocoder, I could access a <code>score</code> variable. This
score indicates how confident ArcGIS is in the accuracy of the
coordinates it generated. After checking several results with a score of
100, I found them to be accurate. Consequently, I filtered the results
to include only those with a score of 100. This resulted in the
approximate number of valid ZIP codes I was looking for: 41,071.

*Note that this block of code takes several hours to run*

``` r
zips_gis <-
    zips %>%
        tidygeocoder::geocode(method = 'arcgis',
                              # 'address' from data frame
                              address = 'address',
                              full_results = T) %>%
        filter(score == 100) %>%
        select(zip_code,
               zip_lat = truncate_digits(lat), 
               zip_lon = truncate_digits(long))

# View results
head(zips_gis, 10)
```

| zip_code | zip_lat |  zip_lon  |
|:--------:|:-------:|:---------:|
|  00103   | 34.2386 | -118.5552 |
|  00208   | 33.0214 | -97.2823  |
|  00209   | 36.8819 | -76.2003  |
|  00501   | 40.8167 | -73.0450  |
|  00544   | 40.8172 | -73.0451  |
|  00586   | 18.3436 | -67.0284  |
|  00601   | 18.1585 | -66.7188  |
|  00602   | 18.3806 | -67.1899  |
|  00603   | 18.4490 | -67.1379  |
|  00604   | 18.4938 | -67.1474  |

    Number of ZIP codes geocoded: 43,003

It was not essential for me to obtain *every* single ZIP code; however,
after processing those ZIP codes that either returned a score less than
100 or no results through the Nominatim geocoder, I ended up with 43,003
geocoded ZIP codes. As mentioned previously, ZIP codes are constantly
changing, so having more ZIP codes than needed is both reassuring and
necessary as there are old ZIP codes in the data as well.

I have provided a CSV of the 43,003 geocoded ZIP codes in this
repository. You can download them
[here](https://www.github.com/JonYarber/zip_code_mapping/tree/main/data/us_zips_with_gis_master.csv).
Having this list from the beginning would have saved me a lot of time
and effort; I hope it does the same for anybody who requires it.

#### 3) Geocode the MTF addresses.

I also used <code>tidygeocoder</code> and the ArcGIS geocoder to
generate the coordinates for the over 100 MTFs of interest.

Instead of using actual MTF addresses, I will be using the addresses of
some iconic U.S. landmarks from the <code>sample_addresses</code> in the
<code>tidygeocoder</code> package.

``` r
# Generate data frame
landmarks <- as.data.frame(tidygeocoder::sample_addresses)

# View data frame
landmarks
```

| name                 | addr                                            |
|:---------------------|:------------------------------------------------|
| White House          | 1600 Pennsylvania Ave NW Washington, DC         |
| Transamerica Pyramid | 600 Montgomery St, San Francisco, CA 94111      |
| NY Stock Exchange    | 11 Wall Street, New York, New York              |
| Willis Tower         | 233 S Wacker Dr, Chicago, IL 60606              |
| Chateau Frontenac    | 1 Rue des Carrieres, Quebec, QC G1R 4P5, Canada |
| Nashville            | Nashville, TN                                   |
| Nairobi              | Nairobi, Kenya                                  |
| Istanbul             | Istanbul, Turkey                                |
| Tokyo                | Tokyo, Japan                                    |

Since we’re only looking at United States ZIP codes, only the first 4
addresses need to be geocoded.

``` r
# Geocode landmarks
landmarks_gis <- 
    landmarks %>%
        # Use only the first 4 rows
        slice(1:4) %>%
        tidygeocoder::geocode(method = 'arcgis', 
                              address = 'addr') %>%
        select(landmark = name,
               address = addr, 
               address_lat = lat, 
               address_lon = long)

# View result
landmarks_gis
```

| landmark             | address                                    | address_lat | address_lon |
|:---------------------|:-------------------------------------------|------------:|------------:|
| White House          | 1600 Pennsylvania Ave NW Washington, DC    |     38.8976 |    -77.0365 |
| Transamerica Pyramid | 600 Montgomery St, San Francisco, CA 94111 |     37.7951 |   -122.4027 |
| NY Stock Exchange    | 11 Wall Street, New York, New York         |     40.7071 |    -74.0108 |
| Willis Tower         | 233 S Wacker Dr, Chicago, IL 60606         |     41.8786 |    -87.6358 |

#### 4) Find all ZIP codes within a square area an MTF.

I determined that the best way to filter the list of 43,000 ZIP codes
before calculating the distances from the MTF to the ZIP codes was to
create a square around the MTF by adding and subtracting latitude and
longitude values from the MTF coordinates. To ensure I captured as many
ZIP codes as possible, I constructed a larger square than necessary. My
target radius was 40 miles; since a degree of latitude in the United
States is approximately 69 miles and a degree of longitude is
approximately 55 miles, I could effectively encapsulate all the relevant
ZIP codes—plus some extra—by adjusting the latitude and longitude
accordingly.

For each of the addresses (MTFs), I then subset the ZIP code data frame
to include only those ZIP codes whose centroids lie within the specified
latitude and longitude ranges.

``` r
# Initiate an empty data frame to concatenate results
# This will be the final data frame
radius_df <- data.frame(landmark = character(0),
                        address = character(0),
                        address_lat = numeric(0),
                        address_lon = numeric(0),
                        zip_code = character(0),
                        zip_lat = numeric(0),
                        zip_lon = numeric(0))

# For each address in the landmarks_gis data frame
for(row in 1:nrow(landmarks_gis)){
    
    # Pull the coordinates
    address_lat <- landmarks_gis$address_lat[row]
    address_lon <- landmarks_gis$address_lon[row]
    
    # Filter the zips_gis data frame to those within 1 latitude and longitude of the address
    radius_zips <-
        zips_gis %>%
            filter(zip_lat < (address_lat + 1) & zip_lat > (address_lat - 1),
                   zip_lon < (address_lon + 1) & zip_lon > (address_lon - 1))
    
    # Add the address and coordinates to radius_zips data frame
    radius_zips <-
        radius_zips %>%
            mutate(landmark = landmarks_gis$landmark[row],
                   address = landmarks_gis$address[row],
                   address_lat = address_lat,
                   address_lon = address_lon)
  
    # Append results
    radius_df <- bind_rows(radius_df, radius_zips)
  
}

# How many ZIP codes did we return per landmark?
radius_df %>%
  group_by(landmark) %>%
  summarize(num_zips = n()) %>%
  arrange(desc(num_zips))
```

| landmark             | num_zips |
|:---------------------|---------:|
| NY Stock Exchange    |     1457 |
| White House          |     1179 |
| Transamerica Pyramid |      603 |
| Willis Tower         |      555 |

<!--Cleanup-->

Let’s examine 5 random results from the data frame.

| landmark             | address_lat | address_lon | zip_code | zip_lat |   zip_lon |
|:---------------------|------------:|------------:|:---------|--------:|----------:|
| Willis Tower         |     41.8786 |    -87.6358 | 60602    | 41.8829 |  -87.6293 |
| Transamerica Pyramid |     37.7951 |   -122.4027 | 95013    | 37.2145 | -121.7371 |
| White House          |     38.8976 |    -77.0365 | 20422    | 38.9250 |  -77.0121 |
| NY Stock Exchange    |     40.7071 |    -74.0108 | 11702    | 40.6955 |  -73.3267 |
| White House          |     38.8976 |    -77.0365 | 20879    | 39.1698 |  -77.1871 |

#### 5) Find the distances from the address to ZIP codes

For the distances, we will use the geodesic distance (shortest distance
between two points on a sphere) with the <code>distGeo</code> function
from the
[geosphere](https://cran.r-project.org/web/packages/geosphere/geosphere.pdf)
package.

``` r
library(geosphere)

# Initiate an empty array to store the distances
distances <- c()

# Iterate through the radius_df data frame and calculate the distance between address and ZIP code
for(row in 1:nrow(radius_df)){
  
    #' Store the address coordinates  
    #' !IMPORTANT: While we typically describe coordinates as (latitude, longitude) the distGeo 
    #'             function requires them to be passed as (longitude, latitude)!!!
    coord1 <- radius_df %>% slice(row) %>% select(address_lon, address_lat)
    
    # Store ZIP code coordinates
    coord2 <- radius_df %>% slice(row) %>% select(zip_lon, zip_lat)
          
    # Find distance between two coordinates
    # Distance generated is in meters and requires conversion to miles
    distance <- distGeo(coord1, coord2) / 1609.3445
          
    # Add to distances array
    distances <- append(distances, distance)
  
}

# Add distances to the data frame
radius_df$dist_in_miles <- round(distances, 2)

# View 5 random results
radius_df %>%
    select(-address) %>%
    slice_sample(n = 5)
```

| landmark             | address_lat | address_lon | zip_code | zip_lat |   zip_lon | dist_in_miles |
|:---------------------|------------:|------------:|:---------|--------:|----------:|--------------:|
| Transamerica Pyramid |     37.7951 |   -122.4027 | 95121    | 37.3041 | -121.8115 |         46.91 |
| White House          |     38.8976 |    -77.0365 | 20230    | 38.8921 |  -77.0331 |          0.42 |
| NY Stock Exchange    |     40.7071 |    -74.0108 | 07086    | 40.7709 |  -74.0194 |          4.43 |
| NY Stock Exchange    |     40.7071 |    -74.0108 | 12548    | 41.6726 |  -74.1040 |         66.80 |
| Willis Tower         |     41.8786 |    -87.6358 | 53125    | 42.5421 |  -88.5668 |         66.17 |

<!--Cleanup-->

#### 6) Filter distances to desired radius.

All that was left was to apply my filter! Let’s see how many zip codes
lie within a 20 mile radius of each of these monuments.

``` r
radius_df %>%
    filter(dist_in_miles <= 20) %>%
    group_by(landmark) %>%
    summarize(num_zips = n()) %>%
    arrange(desc(num_zips))
```

| landmark             | num_zips |
|:---------------------|---------:|
| NY Stock Exchange    |      562 |
| White House          |      554 |
| Willis Tower         |      189 |
| Transamerica Pyramid |      177 |

## Conclusion

In the end, this project delivered exactly what was needed and
established a process that future users could run with ease. We updated
ZIP code data that had gone unchanged for years and created a method to
keep it current whenever necessary. I hope this helps with your own
geospatial projects as well!
