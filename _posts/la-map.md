# Famished on the Freeway: Visualizing, mapping LA restaurant inspections

**Use Python, Matplotlib, Plotly to analyze, view graded restaurants on map**

![](https://cdn-images-1.medium.com/max/800/0*uOEUYRp12RYI9eJe)
<span class="figcaption_hack">“pizza and nachos chip” by [Roberto Nickson
(@g)](https://unsplash.com/@rpnickson?utm_source=medium&utm_medium=referral) on
[Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)</span>

Hey there! I used Python, matplotlib, Plotly to analyze LA county, California
restaurant inspections results dataset and visualize by mapping out the
restaurants with a C grade. Perhaps you can peruse this information after
zooming down the freeway (or more likely, sitting in traffic :)). Here’s an
overview:

* Import the
[dataset](https://www.kaggle.com/cityofLA/la-restaurant-market-health-data) from
kaggle — inspections, violations files
* Create dataframes
* Analyze, sort by number of violations, types etc.
* Group, count by grade, risk and display graphically
* Extract restaurants with low grade (C )and their addresses
* Get their geographic coordinates (latitude, longitude) to use in a map from
Google maps, store them to [create] map
* Plot those restaurants’ locations on a map

Before you begin:

* Ensure that you have Python editor or Notebook available for analysis
* Have a [Google maps
API](https://www.wpgmaps.com/documentation/creating-a-google-maps-api-key/)
token (to get coordinates for restaurant locations)
* Have a [Mapbox token](https://www.mapbox.com/help/define-access-token/) (to map
coordinates onto a map)

    Start a new notebook file:

    from mpl_toolkits.mplot3d import Axes3D
    from sklearn.preprocessing import StandardScaler
    import matplotlib.pyplot as plt
    import numpy as np 
    import os
    import pandas as pd 
    import requests
    import logging
    import time
    import seaborn as sns

Read the two dataset files:

* **Inspections** file is the list of all restaurants, food markets inspected
* **Violations** file is the list of all the violations found and corresponding
facilities (a facility could have multiple violations)

    df1 = pd.read_csv('../input/restaurant-and-market-health-inspections.csv', delimiter=',')
    nRow, nCol = df1.shape
    print({nRow}, {nCol})

    df2 = pd.read_csv('../input/restaurant-and-market-health-violations.csv')

![](https://cdn-images-1.medium.com/max/800/1*Ey5vU4-NBNydP9TFLpS7gA.png)
<span class="figcaption_hack">Output of file rows x columns</span>

Create a dataframe with the raw inspection scores and plot on a histogram:


    from matplotlib import reload
    reload(plt)
    %matplotlib notebook
    df2['score'].hist().plot()

<span class="figcaption_hack">Histogram for score distribution — x-axis is score value bins, y-axis is count
for each value bin</span>

Type the following for the label description:

    df1[~df1['grade'].isin(['A','B','C'])]
    df1.loc[49492,['grade']] = 'C'

    grade_distribution = df1.groupby('grade').size()
    pd.DataFrame({'Count of restaurant grade totals':grade_distribution.values}, index=grade_distribution.index)

#### Note that the totals above add up to 58,872, which is the total # of rows in the
file, which means there are no rows without a grade.

Group by restaurant type (seating type) and count and graph them

    temp = df1.groupby('pe_description').size()
    description_distribution = pd.DataFrame({'Count':temp.values}, index=temp.index)
    description_distribution = description_distribution.sort_values(by=['Count'], ascending=True)
    df2['pe_description'].hist().plot()

<span class="figcaption_hack">Grouped by pe_description field (ex: “Restaurant (seats 31–60, high risk”)</span>

Create a dataframe with the raw inspection scores and plot on a histogram:


    def sub_risk(x):
        return x.split(' ')[-2]
    df2['risk'] = df2['pe_description'].astype(str).apply(sub_risk)

    temp = df2.groupby('risk').size()                                       
    df2['risk'].hist().plot()

<span class="figcaption_hack">Count by Risk type (High, Medium, Low) , extracted from restaurant type field
string</span>

Create a dataframe with the raw inspection scores and plot on a histogram:

    df2.head(10)
    violation_description = df2.groupby('violation_description').size()
    pd.DataFrame({'Count':violation_description.values},index = violation_description.index).sort_values(by = 'Count',ascending=False)

<span class="figcaption_hack">Screen shot — Counts for each type of violation</span>

Create a function to convert string defined values to numbers and make a
heatmap:

    def convert_risk_value(x):
        if x == 'LOW':
            return 10
        elif x == 'MODERATE':
            return 5
        else:
            return 0
       
    def convert_grade_value(x):
        if x == 'A':
            return 10
        elif x == 'B':
            return 5
        else:
            return 0
    df2['risk_value']=df2['risk'].apply(convert_risk_value)
    df2['grade_value']=df2['grade'].apply(convert_grade_value)
    df3 = df2.loc[:,['score', 'grade_value', 'risk_value']]
    corr = df3.corr()
    corr = (corr)
    reload(plt)
    %matplotlib notebook
    sns.heatmap(corr, xticklabels = corr.columns.values, yticklabels=corr.columns.values, cmap="Purples", center=0)

<span class="figcaption_hack">Screen shot — Heat map — score, grade_value and risk_value</span>

Show the violation counts, in descending order from the most common violation:

    violation_desc=df2.groupby(['violation_description','violation_code']).size()
    pd.DataFrame({'Count':violation_desc.values}, index=violation_desc.index).sort_values(by = 'Count', ascending=False)

<span class="figcaption_hack">Screen shot — Violation count by description and code</span>

Show facilities with the most violations:

    violation_desc2 = df2.groupby(['facility_name','violation_description']).size()
    pd.DataFrame({'Count':violation_desc2.values}, index=violation_desc2.index).sort_values(by='Count', ascending=False)

<span class="figcaption_hack">Screen shot — Facilities with the most violations and type of violation</span>

Separate and extract restaurants with grade ‘C’ for mapping. Use the loc
function, then drop duplicates so that each restaurant name only appears once.
We will use this later for the map.

    df4 = df2.loc[(df2['grade'] == 'C'),['facility_name','facility_address','facility_zip']]

    df4=df4.drop_duplicates(['facility_name'])

    df4

<span class="figcaption_hack">Screen shot — list of restaurants with grade C</span>

Below, we list the addresses from above (C grade restaurants) and add the city
and state (which are the same for all addresses) to use as a full address
(location #, street, city, state) this is required as there could be streets and
towns with the same name in other areas around the country and the world) as an
argument to call the google maps api, which will give us the geographic
coordinates (latitude, longitude) which we will then overlay on a map of LA
county, CA.

    addresses_to_avoid = df4['facility_address'].tolist()
    addresses_to_avoid = (df4['facility_name'] + ',' + df4['facility_address'] + ',' + 'LOS ANGELES' + ',CA').tolist()

Add in some console handling, some constants:

    logger = logging.getLogger("root")
    logger.setLevel(logging.DEBUG)
    ch = logging.StreamHandler()      #console handler
    ch.setLevel(logging.DEBUG)
    logger.addHandler(ch)

    address_column_name = 'facility_address'
    restaurant_name = 'facility_name'
    RETURN_FULL_RESULTS = False
    BACKOFF_TIME = 30                 


    API_KEY = 'your api key here'
    output_filename = '../output-2018.csv'     

Create function that calls each address from our list above, and passes a call
to google maps to get the coordinates that will be used to overlay on a map of
LA, California.

    logger = logging.getLogger("root")
    logger.setLevel(logging.DEBUG)
    ch = logging.StreamHandler()      #console handler
    ch.setLevel(logging.DEBUG)
    logger.addHandler(ch)

    address_column_name = 'facility_address'
    restaurant_name = 'facility_name'
    RETURN_FULL_RESULTS = False
    BACKOFF_TIME = 30

    API_KEY = 'Your API key here'
    output_filename = '../output-2018.csv'
    #print(addresses)

    def get_google_results(address, api_key=None, return_full_response=False):
        geocode_url = "
    }".format(address)
        if api_key is not None:
            geocode_url = geocode_url + "&key={}".format(api_key)
            
            #ping google for the results:
            results = requests.get(geocode_url)
            results = results.json()             
            
            if len(results['results']) == 0:
                output = {
                    "formatted_address" : None,
                    "latitude": None,
                    "longitude": None,
                    "accuracy": None,
                    "google_place_id": None,
                    "type": None,
                    "postcode": None
                }
            else:
                answer = results['results'][0]
                output = {
                    "formatted_address" : answer.get('formatted_address'),
                    "latitude": answer.get('geometry').get('location').get('lat'),
                    "longitude": answer.get('geometry').get('location').get('lng'),
                    "accuracy": answer.get('geometry').get('location_type'),
                    "google_place_id": answer.get("place_id"),
                    "type": ",".join(answer.get('types')),
                    "postcode": ",".join([x['long_name'] for x in answer.get('address_components') 
                                      if 'postal_code' in x.get('types')])
                }
                
            #append some other details
            output['input_string'] = address
            output['number_of_results'] = len(results['results'])
            output['status'] = results.get('status')
            if return_full_response is True:
                output['response'] = results
            
            return output

Call the function, and if all goes well, it will write the results out to a
specified file. Note: Every so often, the call will hit a “Query limit” bump
imposed by Google, so you will either have to call those manually (after
filtering from the file) or delete them from your output file.

    results2=[]
    for address in addresses_to_avoid:
        geocode_result = get_google_results(address, API_KEY, return_full_response=RETURN_FULL_RESULTS)
        results2.append(geocode_result)


    pd.DataFrame(results2).to_csv('/Users/hsantanam/datascience-projects/LA-restaurant-analysis/restaurants_to_avoid.csv', encoding='utf8')

We are almost done. I could have created a function in the same code, but for
readibility and understanding, did it in a new program, which sets the
configuration for the map, calls the data from the output file with the address
coordinates that we just created above, and draws it on a browser tab. Here it
is, along with the output:

And here, finally, is the result — visual map output of the code above.

<span class="figcaption_hack">Restaurants with C grade — mapped out for LA county, CA — opens in default
browser, new tab</span>

<span class="figcaption_hack">Popups with name, coordinates, address appear when you hover over any location
dot</span>

Thank you for patiently reading through this :). You are probably hungry by now
— if you happen to be this area, find a restaurant and eat, just not in the ones
with the bad grade!

The github link to this is
[here](https://github.com/HariSan1/LA-restaurant-inspections-analysis).

Check out some of my other articles if it piques your interest:

* [Data Science](https://towardsdatascience.com/tagged/data-science?source=post)
* [Python](https://towardsdatascience.com/tagged/python?source=post)
* [Matplotlib](https://towardsdatascience.com/tagged/matplotlib?source=post)
* [Mapbox](https://towardsdatascience.com/tagged/mapbox?source=post)
* [Data Analysis](https://towardsdatascience.com/tagged/data-analysis?source=post)

From a quick cheer to a standing ovation, clap to show how much you enjoyed this
story.

### [Hari Santanam](https://towardsdatascience.com/@hari.santanam)

I am interested in AI, Machine Learning and its value to business and to people.
I occasionally write about life in general as well.

### [Towards Data Science](https://towardsdatascience.com/?source=footer_card)

Sharing concepts, ideas, and codes.
