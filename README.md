# Geolocation-function
Explanation of how the the geolocation functions works step by step

## Summary
The geolocation function uses 2 out of 3 APIs that utilize the forward/inverse geocoding services to locate an address based on coordinates. Each API locates the same
address and the two points with the minimum square distance between them are then used to calculate the coordinates of the mid-point. Next, it analyzes if the mid-point was calculated based on three edge cases that could indicate that the coordinates are not entirely correct. The first flag indicates if the minimum distance between the two selected points is greater than 20 km, the second flag indicates if the address is made of two or fewer administrative levels (ex. country, province), and the third flag indicates if the coordinates of the calculated mid-point are outside the country's boundary. Finally, for the first flag, the rgeoboudaries package is used to correct the mid-point coordinates by uploading the 2nd administrative level of the country in question and getting the coordinates of the city's centroid from the respective address.

```
two_point<- function(data_base, #Database that contains the geographical information that needs to be geolocated 
                     country = TRUE, #A boolean input that indicates if you have the "country" variable and want to use it to geolocate an address
                     province = TRUE, #A boolean input that indicates if you have the "province" variable and want to use it to geolocate an address
                     city = TRUE, #A boolean input that indicates if you have the "city" variable and want to use it to geolocate an address
                     district = FALSE) #A boolean input that indicates if you have the "district" variable and want to use it to geolocate an address
```

## Example
The following example will help us demonstrate step by step how the function works:

```
data<- data.frame(country = rep("Argentina", 10),
                  province = c("Buenos Aires", "Buenos Aires", "Mendoza", "Formosa", "Tucuman", "Misiones", NA, "Río Negro", "Chaco", "Corrientes"),
                  city = c("lobos", "bahía blanca", "la consulta", "", "amaicha", "obera", "gran parana", "general roca", "basail", "goya"))

```

![data_simple_fun](https://github.com/user-attachments/assets/c8aeb706-2492-4727-95e7-09a94ed1c752)

```
test<- two_point(data_base = data)
```
### Step 1: Input check point
This check point verifies that the inputs are correct, in case that they are not, the following message will appear:
```
Please check that you have the following variables in the database: country, province, city, and district (if you indicated)
```

### Step 2: Getting the administrative level fields to create the address
Here the function determines if the inputs to be used as administrative levels are type string or factor: province, city and district. In case they are a factor, it extracts the label from them, if not, the value is maintained. The function will always use by default the administrative levels of country, province, city. But they can be switch off or you can even add the administrative level of district.

![data_addr](https://github.com/user-attachments/assets/8a589438-50c9-43c4-a21a-819132f30bfa)

### Step 3: Obtaining the coordinates from the APIs for each address
Once the address is created for each row, it selects the unique values (the same address will not be located more than once) and passes them to the ArcGIS, MapBox and TomTom APIs. For our example, all addresses are unique, so we will have a total of 10 addresses and when they pass through the API, the console displays the following:
```
Passing 10 addresses to the ArcGIS single address geocoder
[===================================================] 10/10 (100%) Elapsed:  5s Remaining:  0s
Passing 10 addresses to the Mapbox single address geocoder
[===================================================] 10/10 (100%) Elapsed:  1s Remaining:  0s
Passing 10 addresses to the TomTom batch geocoder
Query completed in: 1.5 seconds
```
Each API will locate the address and using the `st_distnace` function from the `sf` package, it will calculate the distance between each point. Then, with that distance we will calculate the mean square distance and select the two points who have the minimum value. By selecting these two points, we will next get the mid-point between them and obtain the coordinates.

![mid-point](https://github.com/user-attachments/assets/378e672a-c6b5-4103-8e85-2b38cad64121)

### Step 4: Indicating flags
With the final location indicated by the coordinates of the mid-point, we will now analyze if these were located correctly through three flags. The first flag will let us know if thw two points used to obtain the final coordinates had a distance greater than 20 km. The second flag, indicates if the final coordinates were located by using only two administrative levels, for example, country-province. The final flag will tell us if the final coordinates are most likely out of bounds from the country's boundary (over 3000 km away from the centroid of the country). All of these three flags are binary variables where 1 indicates the condition is true and 0 if not.

![two_point](https://github.com/user-attachments/assets/871a13cd-9aa0-4da9-a2cf-bbae5e446ecd)

### Step 5: Correcting 20 km flag
In this last step of the function, rows who only had a 20 km flag raised will be corrected using the [rgeoboundaries](https://github.com/wmgeolab/rgeoboundaries) package that uses open data of geoboundaries. First, it will filter the cases that only have a 20 km flag, then it will modify any special characters from the address and obtain the last administrative level (by default it will be the city). Next, it will obtain the ISO code for the country, upload the 2-administrative level global data, filter the respective country and adjust the name of the 2-administartive level by removing any special character. Later, it will try to match the names of 2-administrative level boundaries with the names of the flagged cities (10% or less of the number of characters between the two strings can be different). After finding the matches, it will filter them, calculate the centroid of that 2-administrative level boundary and obtain its coordinates. With these, we use the ArcGIS API reverse geocoding service and obtain the address in order to match it with the actual address. In case they match (40% or less of the number of characters between the two strings can be different) then the mid-point's coordinates will now be replaced with the centroid's coordinates of the geoboudaries, if not, then the mid-point coordinates will remain. In case there was a replacement, the 20 km flag variable would change from 1 to 2, meaning that if the location was corrected it would be 2, if it was not then 1 and, if there was no flag to begin with it will remain 0.

![final two_point](https://github.com/user-attachments/assets/ea0c13c1-e0bd-42c3-bd98-072e8a04c10a)

We can see that from our example that even though we had three flagged cases of over 20 km, only two cases were corrected.

In case the 2-administartive levels from the geoboundaries data does not match any city, no corrcetion will be done and the following message will appear:

```
Warning: the geoboundaries correction is not possible for this country because of the 2nd level administrative level it uses
```

### Step 6: Final result
The final result that the function will return is a list of the following variables:
* addr (address)
* p1 (point #1 used to calculate the mid-point)
* p2 (point #2 used to calculate the mid-point)
* latitude (from the mid-point)
* longitude (from the mid-point)
* flag_20km
* flag_2_level
* flag_box

> [!IMPORTANT]
>* Before applying this function, please make sure you have created an API key from the following applications: [MapBox](https://www.mapbox.com/) and [TomTom](https://www.tomtom.com/en_gb/navigation/)
>* The CRS used throughout the function is 4326 (WGS 84)
>* The function may stop for cases when any of the API do not find an address, because the `st_as_sf` function from the `sf` package cannot change NAs into coordinates



