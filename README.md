# [UBC Institute of Fisheries Field Records](https://open.library.ubc.ca/collections/fisheries)

Linked data web application providing a georeferenced visualization of UBC Open Collection Institute of Fisheries Field Records metadata, linked to [Encyclopedia of Life](http://eol.org/).

[Link for the application on CARTO.](https://carolamigo.carto.com/builder/df748d01-2fc8-424b-ae46-dfd834b200a0/embed)

![fisheries.gif](https://github.com/carolamigo/ubc_carto_fisheries/blob/master/fisheries.gif)

### How to do it:

#### Modeling

- Download collection metadata using the Open Collections Research API. A php script to batch download is provided at OC API Documentation page > Download Collection Data.. This script returns a folder containing one RDF file per collection item (or XML, JSON, any format preferred). We are going to use N-triples because the file is cleaner (no headers or footers), what makes the merging easier later. Edit the script following the instructions on the documentation page and run it using the command:

`$ php collection_downloader.php --cid fisheries --fmt ntriples`

- Merge files using the [file_merge_script.py](https://github.com/carolamigo/ubc_carto_fisheries/blob/master/file_merge_script.py). The folder containing the files to me merged has to be named “fisheries” and the script has to be run from the same folder the folder fisheries is in.

- Convert merged file obtained to a tabular format. Import project in Open Refine using the RDF/N3 files option. No character encoding selection is needed.

#### Cleaning

- The latitude values are in the “http://www.w3.org/2003/01/geo/wgs84_pos#lat” column. We have to change their formatting to numbers so CARTO can understand them:

    `"54 13"@en  >  54.13`

    `"0 30 S"@en  >  -0.3 (Latitudes south of the equator have negative values)`

- Create a new column “test” based on the column “http://www.w3.org/2003/01/geo/wgs84_pos#lat” using the following expression to remove any character and preserve only digits and blank spaces (keeping the spaces is important for placing the decimal points later):

`value.replace(/[^\ ,^\d]/, "")`

- We have now to transform those values in numbers, but it is important to insert the decimal point in the right spot first, so select on the column “test”, “Edit column” > “Split into several columns”, separating by a blank space and selecting splitting into 2 columns at most. You are going to get two columns, “test 1” and “test 2”. Create a new column “latitude” based on the “test 2” column using the following expression to concatenate the values with a decimal dot in between:

`cells["test 1"].value + "." + cells["test 2"].value`

- On “latitude” column, select “Edit cells” > “Transform” and write the following expression to remove any remaining blank spaces:

`value.replace(" ","")`

- We have now the values with the decimal point in the right position. Ensure all values are numbers by selecting on the column “latitude” > “Edit cells” > “Common transforms” > “To number”. Delete columns “test 1” and “test 2”.

- Filter column “http://www.w3.org/2003/01/geo/wgs84_pos#lat” to select only cells containing “S”, using “Text filter” and typing “S” in the box that appears in the left sidebar. On the “latitude” column, select “Edit cells” > “Transform” and write the following expression to make all south latitudes negative values:

`value*-1`

- Now we have all latitudes south with a negative sign before them. Close the “Text facet” window on the left sidebar to remove the filter.

- We have now to repeat the procedure to the “http://www.w3.org/2003/01/geo/wgs84_pos#long” column (longitudes).

- Create a new column “test” based on the column “http://www.w3.org/2003/01/geo/wgs84_pos#long” using the following expression to remove any character and preserve only digits and blank spaces:

`value.replace(/[^\ ,^\d]/, "")`

- We have to transform those values in numbers, but it is important to insert the decimal point in the right spot, so select on the column “test”, “Edit column” > “Split into several columns”, separating by an blank space and selecting splitting into 2 columns at most. You are going to get two columns, “test 1” and “test 2”. Create a new column “longitude” based on the “test 2” column using the following expression to concatenate the values with a decimal dot in between:

`cells["test 1"].value + "." + cells["test 2"].value`

- On “longitude” column, select “Edit cells” > “Transform” and write the following expression to remove any remaining blank spaces:

`value.replace(" ","")`

- We have now the values with the decimal point in the right position. Ensure all values are numbers by selecting on the column “longitude” > “Edit cells” > “Common transforms” > “To number”. Delete columns “test 1” and “test 2”.

- Filter column “http://www.w3.org/2003/01/geo/wgs84_pos#long” to select only cells containing “W”, using “Text filter” and typing “W” in the box that appears in the left sidebar. On the “longitude” column, select “Edit cells” > “Transform” and write the following expression to make all west longitudes negative values:

`value*-1`

- Now we have all longitudes west with a negative sign before them. Close the “Text facet” window on the left sidebar to remove the filter.

- Let’s verify if the values for latitude and longitude are within the correct ranges. Facet the “longitude” column by number (“Facet” > “Numeric facet”) to check values (you might need to increase the faceting limit number). Longitudes have a range of -180 to +180. Any value outside that range is incorrect. Slide the filter selector on the left sidebar to see values that are larger than 180. Uncheck box “blank”. Take a look on the rest of the metadata for inferring the correct value. Correct then manually by clicking on edit inside the wrong value cell, changing the value, the data type to “number” and selecting “Apply to all identical cells”.

- Now facet the “latitude” column by number (“Facet” > “Numeric facet”) to check values. Latitudes have a range of -90 to +90. Any value outside that range is incorrect. Slide the filter selector on the left sidebar to see values that are larger than 90. Uncheck box “blank”. We can see by examining column “longitude” that these values of latitude and longitude are swapped. Correct then manually by clicking on edit inside the wrong value cell, changing the value, the data type to “number” and selecting “Apply to all identical cells”. Changes to latitude and longitude are complete.

- The fish species are in the “http://purl.org/dc/terms/subject” column. To get better reconciliation results, we have to remove the double quotes, the “@en”, the “sp.” and keep just the species name inside the square brackets (when it exists):

  `"Agonus acipenerinus [Agonus accipenserinus]"@en  >  Agonus acipenerinus`
  
  `"Ambassis sp."@en  >  Ambassis`

- Create a new column “species” based on the column “http://purl.org/dc/terms/subject”, using the following expression:

`value.split('[')[-1].replace("\"", " ").replace("@en"," ").replace("sp.","").replace("]",'')`

- Trim leading and trailing whitespaces by selecting “Edit cells” > “Common transforms” > “Trim leading and trailing whitespaces”.

- Cluster species names in order to combine entries with typos and small inconsistencies under just one species name. On the “species” column, select “Facet” > “Text facet”, then select “Cluster” in the facet window. In the cluster window, experiment with different clustering methods. Start with “key collision” > fingerprint. Take a look at the results, and, if they are good enough, select the “Select all” button and then “Merge selected and Re-Cluster”. Iterate until there are no more cluster formed, then try another clustering method until your have formed all clusters possible.

- Fill down the following columns as we have several orphan cells resulting from the triple to tabular data format conversion. Go to each column, “Edit cells” > “Fill down”.
  - “subject”
  - “http://purl.org/dc/terms/title”
  - “http://purl.org/dc/elements/1.1/date”
  - “http://www.europeana.eu/schemas/edm/isShownAt”
  - “http://purl.org/dc/terms/coverage”
  - “http://purl.org/dc/terms/spatial”
  - latitude_number
  - longitude_number

#### Reconciling

- On the “species” column, select “Reconcile” > “Start Reconciling” > “Add standard service”. Paste the following URL* in the box, then click “Add Service”:

`http://iphylo.org/~rpage/phyloinformatics/services/reconciliation_eol.php`

The Encyclopedia of Life (EOL) taxonomy reconciliation service to Open Refine was developed by: http://iphylo.blogspot.ca/2012/02/using-google-refine-and-taxonomic.html

- The reconciliation service will appear under reconciliation services tab. Select it and click “Start reconciling”. The process will take a long time (one hour or two) since we have many entries. You have to wait until it is done to do any further work on the data.

- When reconciliation is finished, review the results. Use the reconciliation faceting “species: judgment” box on the left sidebar to review the “none” ones. Those need you input to pick the best match. Up to three options show up. Select the correct one by using the double checked box, which means your decision will be applied to all other cells that match this condition. If no correct option show up in the cell, click on the double checked box of “Create new topic”, meaning that no reconciliation value will be added to cells that match this condition (they are going under “new” facet and you will need to add values manually for those later).

- As there are too many unique values to assess, you can review a sample and then, with the “none” facet still on, select on the species column “Reconcile” > “Actions” > Match each cell to its best candidate.

- Extract reconciled data retrieving name and id, as a string separated by “|”. Select “species”, “Edit cells” > “Transform”, and entering the following expression:

`cell.recon.match.name + " | " + cell.recon.match.id`

- Split the values obtained in the reconciliation, that should be in this format:

`Isopsetta isolepis (Lockington, 1880) | 995111`

- Select “species”, “Edit column” > “Split into several columns”. Select the “|” separator and name the columns according after split: species_eol, eol_id.

- We have to build EOL links by creating a new column “eol_uri” based on “eol_id”, using the following expression:

`"http://eol.org/pages/"+value`

#### Building interface

- Prepare the data to interface. In order to have links on CARTO interface, we have to add html tags in the source dataset.

- Remove double quotes and language. Create a new column “title” based on the column “http://purl.org/dc/terms/title”, using the following expression:

`value.replace("\"", " ").replace("@en"," ")`

- Add html tags for title links. Create a new column “title_link” based on the column “subject”, using the following expression:

`"<a href=\""+value+"\">"+if(isBlank(cells["title"].value), " ", cells["title"].value)+"<\/a>"`

- Add html tags for EOL species links. Create a new column “eol_html” based on the column “eol_uri”, using the following expression:

`"<a href=\""+value+"\">"+if(isBlank(cells["species_eol"].value), " ", cells["species_eol"].value)+"<\/a>"`

- Export the dataset from OpenRefine in .csv format. Name the file “fisheries_cleaned”.

- Sign up or Log in to CARTO Builder: https://carto.com/signup/. Create a new map and import the Open Refine exported file to your map.

- Georeference your dataset following the instructions here: https://carto.com/learn/guides/analysis/georeference. Once in your map, click on the dataset on the left menu, then click on “Analysys” > “Add analysys” > Georeference. Select the corresponding column names in your dataset for latitude and longitude. Note that the application is plotting just one resource per location, so we will need to aggregate the results to have all the resources plotted.

- Export the georeferenced dataset from CARTO in csv format, in order to incorporate the “the_geom_webmercator” column (with georeferenced data) in your dataset. Name the file “fisheries_cleaned_geo”. Import the dataset back into CARTO map, and delete the previous dataset from your map. This step is necessary since CARTO does not allow georeference analysis and SQL manipulation (that we will need for aggregation) of the data concomitantly.

- Click on the dataset on the lateral menu and select the “Data” tab. At the bottom of the lateral panel, enable SQL option. Paste the following query in the editor and click “Apply”:

`SELECT string_agg(CONCAT(species, ' <br>', eol_html, ' <br>'),' <br>') as new_column_aggregated, title_link, the_geom_webmercator,
Min(cartodb_id) cartodb_id
FROM fisheries_cleaned_geo
group by title_link, the_geom_webmercator`

- Click on the “Pop-up” tab, and enter the following settings:

  - Style: color
  - Window size: 400
  - Header color: #a6e79a
  - Show item: check all boxes (make sure “title_link” is first on the list).

- To build the filters:

  - Click on “fisheries_cleaned_geo” dataset, “style” tab, and use the following settings:

    - Aggregation: none
    - Size 6, color #f6ff00
    - Stroke 1 color #FFFFFF, transparent
    - Blending: overlay

- Exit “fisheries_cleaned_geo” dataset and click on the “add” button to re import the same dataset. You will get copy named “fisheries_cleaned_geo_1”. Click on this new dataset, “style” tab, and use the following settings:

  - Aggregation: none
  - Size 12, color #ff0000
  - Stroke 1 color #FFFFFF, A:0.7
  - Blending: none

- Exit the dataset and make it the second one in the list showing on the lateral panel, dragging and dropping it. We want this new layer behind the one that has the pop-up with photos.

- Click on “Widget” tab to add filters. Add widgets following the instructions here: https://carto.com/learn/guides/widgets/exploring-widgets.

- Exit the datasets and change the basemap to “Here” > “Satellite Day”

- Publish your map using the “Publish” button on the left lateral panel, just under the map name.
