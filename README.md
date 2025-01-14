# Geolocation-function
Explanation of how the the geolocation functions works step by step

## Summary
The geolocation function uses 2 out of 3 APIs that utilize the forward/inverse geocoding services to locate an address based on coordinates. Each API locates the same
address and the two points with the minimum square distance between them are then used to calculate the coordinates of the mid-point. Next, it analyzes if the mid-point was calculated based on three edge cases that could indicate that the coordinates are not entirely correct. The first flag indicates if the minimum distance between the two selected points is greater than 20 km, the second flag indicates if the address is made of two or fewer administrative levels (ex. country, province), and the third flag indicates if the coordinates of the calculated mid-point are outside the country's boundary. Finally, for the first flag, the rgeoboudaries package is used to correct the mid-point coordinates by uploading the 2nd administrative level of the country in question and getting the coordinates of the city's centroid from the respective address.

The order of administrative levels is always country-province-city-block (please see the figure below).
![admin_level](https://github.com/user-attachments/assets/05314504-0c1f-4e9d-9b20-e65c1b0c246c)

```
two_point<- function(data_base, #Database that contains the geographical information that needs to be geolocated 
                     country = TRUE, #A boolean input that indicates if you have the "country" variable and want to use it to geolocate an address
                     province = TRUE, #A boolean input that indicates if you have the "province" variable and want to use it to geolocate an address
                     city = TRUE, #A boolean input that indicates if you have the "city" variable and want to use it to geolocate an address
                     block = FALSE, #A boolean input that indicates if you have the "block" variable and want to use it to geolocate an address
                     filter_NA = TRUE){ #A boolean input that indicates if addresses that are not found should be filter out or not
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
Please check that you have the correct variables names in your data frame depending on the administrative levels you chose: country, province, city, block
```

### Step 2: Getting the administrative level fields to create the address
Here the function determines if the inputs to be used as administrative levels are type string or factor: province, city and block (country should always be a string). In case they are a factor, it extracts the label from them, if not, the value is maintained. The function will always use by default the administrative levels of country, province, city. But they can be switch off or you can even add the administrative level of block.

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
Please note that by default, the addresses that pass to the next step of measuring the distance between points are only the ones that are found in all three APIs, therefore, a message will always appear (unless `filter_NA = FALSE`) with the number of common adresses.
```
__ common addresses were located
```
Each API will then locate the address and using the `st_distnace` function from the `sf` package, it will calculate the distance between each point. Then, with that distance we will calculate the mean square distance and select the two points who have the minimum value. By selecting these two points, we will next get the mid-point between them and obtain the coordinates.

![mid-point](https://github.com/user-attachments/assets/378e672a-c6b5-4103-8e85-2b38cad64121)

### Step 4: Indicating flags
With the final location indicated by the coordinates of the mid-point, we will now analyze if these were located correctly through three flags. The first flag will let us know if the two points used to obtain the final coordinates had a flat distance greater than 20 km. The second flag, indicates if the final coordinates were located by using only two administrative levels, for example, country-province. The final flag will tell us if the final coordinates and the two points used to obtain it are outside the boundary of the country using the geoboundaries country data. The first two flags are binary variables where 1 indicates the condition is true and 0 if not. The final flag is numeric with a value range between 0-3.

![two_point](https://github.com/user-attachments/assets/6002ea11-239e-4e96-8ca1-5a4d4ce7d6c0)

### Step 5: Correcting 20 km flag
Since some locations have similar names even within the country, the APIs will no always locate correctly. Therefore, with the 20km flag it is easy to notice those cases and correct them if possible. For example, if in country A there is a province named B, and within that province we want to locate city F, but the city next to it has a street also named F, then some APIs could locate the wrong address. As a result, in this step of the function rows who only had a 20 km flag raised will be corrected using the [rgeoboundaries](https://github.com/wmgeolab/rgeoboundaries) package that uses open data from geoboundaries. The flowchart below will demosntrate each step of the correction process.
![flow_chart_geo3](https://github.com/user-attachments/assets/3a82515d-ccaa-4109-85d5-7b6649f27ede)

Following this process, the following warning messages will appear depending on the case:

**_No 20 km flags or block = TRUE_**
```
Warning: If you indicated block to be taken into account, the 20 km flag cannot be corrected with level-2 geoboundaries global data
```
**_ISO code was not located_**
```
Warning: the country does not have an ISO code thus, no correction for the 20 km flag was possible, please check the name
```
**_No matches found with 2-level boundary data_**
```
Warning: the geoboundaries correction is not possible for this country because of the 2nd administrative level it uses
```
The cases that were corrected can be seen in the _flag_20km_ column with a 2, in case it was not corrected it would remain a 1 and if there was no flag to begin with, then it will be a 0. We can see that from our example that even though we started with four flagged cases of over 20 km, only two of them were corrected.
![final_two_point](https://github.com/user-attachments/assets/e2f265d5-0dfb-4a91-b15f-f3bb086ec00d)

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
>* Please note that `province` could also be interpret as a State and `block` as a neighborhood
>* The CRS used throughout the function is 4326 (WGS 84)
>* The function may stop for cases when `filter_NA = FALSE`, because the `st_as_sf` function from the `sf` package cannot change NAs into coordinates since the API may not find the address
>* Locations near the coast may have a value in _flag_box_ different from 0
>* If your internet is not stable, then the function will stop and not work correctly 



