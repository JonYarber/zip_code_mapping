
# Zip Code Geocoding

## Context

One particular item we are interested in examining in my division is
what we call Private Sector Care, or care that comes from outside a
military treatment facility (MTF). One of our teams is tasked with
finding the predominant types of care being performed within a radius of
a given MTF. The data does not provide us with a level of detail to
ascertain this information, so we have to generate it.

## Methodology

We found that the best way to find this was as follows: 1. Find all zip
codes within a radius of the MTF 2. Isolate all providers with one of
those zip codes 3. Extract those providers from the data and indicate
all ICD-10 codes associated with TRICARE beneficiary visits.

## Challenge

There were actually quite a few challenges associated with this one: 1)
How do we get a *free* list of all U.S. zip codes (approximately 43K)?
2) To find zip codes within a radius, I will need to use geocoordinates
(latitude and longitude): - How do I find the geocoordinates of the zip
codes? - Given the geocoordinates of the MTF and zip codes, how do I
efficiently isolate the zip codes we need?

## Challenge
