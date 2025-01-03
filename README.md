# Geolocation-function
Explanation of how the the geolocation functions works step by step

## Summary
The geolocation function uses 2 out of 3 APIs that utilize the forward/inverse geocoding services to locate an address based on coordinates. Each API locates the same
address and the two points with the minimum square distance are then used to calculate the coordinates of the mid-point. Next, it analyzes if the mid-point was 
calculated based on three edge cases that could indicate that the coordinates are not entirely correct. The first flag indicates if the minimum distance between the two
selected points is greater than 20 km, the second flag indicates if the address is made of two or fewer administrative levels (ex. country, province), and the third
flag indicates if the coordinates of the calculated mid-point are outside the country's boundary. Finally, for the first flag, the rgeoboudaries package is used to
correct the mid-point coordinates by uploading the 2nd administrative level of the country in question and getting the coordinates of the city's centroid from the
respective address. 

## Step 1: Getting the administrative level fields to create the address
Here the function determines if the class of the inputs to be used as administrative levels are string or factor: province, city and district. In case they are factor,
it extract the label from them and when they are type string, they value is maintain. The function will always use by default the administrative levels of country, 
province, city. But they can be switch off or even add the administrative level of district.

## Step 2: Obtaining the coordinates from the APIs for each address
Once the address is created for each row, it selects the unique values (a same address will not be located more than once) and passes them to the ArcGIS, MapBox
and TomTom APIs. Each API will locate the address and using the st_distnace function from the sf package it will calculate the distance between each point.

Then, with the calculate distance we will calculate the mean square distance and select the two points who have the minimum value. By selecting these two points,
we will next get the mid-point between them and obtain the coordinates.

## Step 3: Indicating flags
With the final location indicated by the coordinates of the mid-point, we will now analyze if these were loacted correctly through three flags. The fisr flag
will let us know if thw two points used to obtain the final coordinates had a distance greater than 20 km. The second flag, indicates if the final coordinates
were located by using only two administrative levels, for example, country-province. The final flaf will tell us if the final coordinates are most likely out
of bounds from the country's boundary (over 3000 km away from the centroid of the country).

# Step 4: Correcting 20 km flag
In this last step of the function, rows who only had a 20 km flag raised will be corrected using the rgeoboundaries package that uses open data of geoboundaries.
First, it will filter the cases that only have a 20 km flag, then it will modify any special characters from the address and obtain the last administrative level
(by default it will be the city). Next, it will obtain the ISO code for the country, upload the 2-administrative level global data, filter the respective country
and, adjust the name of the 2-administartive level by removing any special character. Then, it will try to match (10% or less of the number of characters in the
string are different) the names of 2-administrative level boundaries with the name of the flagged cities. After finding the matches it will filter them, calculate
the centroid of that 2-administrative level boundary and obtain its coordinates. With these, we use the ArcGIS API reverse geocoding service and obtain the address
in order to match it with the actual address. In case they match (40% or less of the number of characters in the string are different) then the mid-point coordinate
will now be replaced with the centroid's coordinates of the geoboudaries, if not, then the mid-point coordinates wil remain. In case there was a replacement the
20 km flaf variable will change from 1 to 2, meaning that if the location was corrected it will be 2, if it was not then 1 and, if there was no flag to begin with
it will remain 0.

In case the 2-administartive levels from the geoboundarie sdata does not match any city no corrcetion will be done and the folloing warning will appear: XXX

## Step 5: Final result
The final result that the funtion will return is a list of the folloiwng variables:
* addr (address)
* p1 (point #1 used to calculate the mid-point)
* p2 (point #2 used to calculate the mid-point)
* latitude
* longitude
* flag_20km
* flag_2_level
* flag_box
 
### FALTA AGREGAR IMAGENES, CODIGO BLOQUE, LINKS y PASO DE REVISAR QUE LOS INPUTS SEAN LOS CORRECTOS 

> [!IMPORTANT]
>* Before applying this function, please make sure you have created an API key from the following applications: MapBox and TomTom
>* The CRS used throughout the function is 4326 (WGS 84)
>* The function may stop for cases when any of the API do not find an address, becase the st_distance function does not allow any empty inputs



