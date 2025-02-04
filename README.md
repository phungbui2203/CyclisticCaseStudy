# Understanding Cyclistic Riders: Insights to Drive Membership Growth

As a junior data analyst on Cyclistic’s marketing analysis team, my role is to uncover actionable insights that will inform strategies to increase the company’s annual memberships. Cyclistic, a prominent bike-share company in Chicago, recognizes the critical role that annual members play in driving its long-term success. To support this objective, this report explores the differences in usage patterns between casual riders and annual members. By leveraging data-driven insights and professional visualizations, we aim to provide a solid foundation for designing targeted marketing strategies that convert casual riders into loyal members.

## Opening Question

#### Question we need to answer
- How do annual members and casual riders use Cyclistic bikes differently?
- Why would casual riders buy Cyclistic annual memberships?
- How can Cyclistic use digital media to influence casual riders to become members?

#### Questions to gain more insights
- How are the start and end points of riders distributed across different membership groups?
- What is the distribution of ride frequency across the months of the year for different membership groups?
- What is the distribution of ride frequency across the days of the week for different membership groups?
- What is the distribution of ride frequency across the hours of the day for different membership groups?
- What is the distribution of ride type for different membership groups?
- What is the distribution of travel distance for each membership group?
- What is the distribution of ride duration for each membership group?

## Prepare data for analysis
Currently, the data is stored in different files, so we will consolidate them into a single table. We will use PostgreSQL for data aggregation and analysis. For large datasets with millions of rows, PostgreSQL performs better than Excel in terms of data analysis and manipulation.

### Create table for data store
Create a table to store raw data in preparation for future analysis.

```sql
create table tripdata
(
    ride_id            varchar(100) not null
        constraint tripdata_pk
            primary key,
    rideable_type      varchar(100),
    started_at         timestamp,
    ended_at           timestamp,
    start_station_name varchar(100),
    start_station_id   varchar(100),
    end_station_name   varchar(100),
    end_station_id     varchar(100),
    start_lat          double precision,
    start_lng          double precision,
    end_lat            double precision,
    end_lng            double precision,
    member_casual      varchar(50)
);
```



## Create a view to prepare for analysis.
### In the view, include the following information: ride_id, rideable_type, started_at, ended_at, start_station_name, end_station_name, start_lat, start_lng, end_lat, end_lng, member_casual, month, day, hour, distance, and duration_minutes.
#### To calculate the distance between the start and end points using the columns start_lat, start_lng, end_lat, and end_lng, we need to install the extension cube and the extension earthdistance.

```sql
CREATE EXTENSION IF NOT EXISTS cube;
CREATE EXTENSION IF NOT EXISTS earthdistance;
```
#### Adding columns beyond the existing data in the table helps us:
- We add the month column to determine how the ride frequency is distributed across the months of the year for different member groups.
- We add the day column to determine how the ride frequency is distributed across the days of the week for different member groups.
- We add the hour column to determine how the ride frequency is distributed across the hours of the day for different member groups.
- We add the distance column to determine the average distance traveled by each member group.
- We add the duration_minutes column to determine the average ride duration for each member group.
##### We remove the start_station_id and end_station_id columns because they are unnecessary and not used in the analysis.
```sql
create view tripdata_view
            (ride_id, rideable_type, started_at, ended_at, start_station_name, end_station_name, start_lat, start_lng,
             end_lat, end_lng, member_casual, month, day, hour, distance, duration_minutes)
as
SELECT ride_id,
       rideable_type,
       started_at,
       ended_at,
       start_station_name,
       end_station_name,
       start_lat,
       start_lng,
       end_lat,
       end_lng,
       member_casual,
       EXTRACT(month FROM started_at)                                                   AS month,
       EXTRACT(dow FROM started_at)                                                     AS day,
       EXTRACT(hour FROM started_at)                                                    AS hour,
       earth_distance(ll_to_earth(start_lat, start_lng), ll_to_earth(end_lat, end_lng)) AS distance,
       EXTRACT(epoch FROM ended_at - started_at) / 60::numeric                          AS duration_minutes
FROM tripdata;
```

## Insert data to database
I will use two libraries, psycopg2 and pandas, to read data from a CSV file and save it to the database.

Firstly, we install psycopg2 and pandas:
```bash
pip install psycopg2 pandas
```

Next, let start insert data to database:

Some lat lng data is empty, we will remove these row

We got key duplicate error, so we will create a temp table to store duplicate data first named tripdata_temp

```sql
create table tripdata_temp
(
    ride_id            varchar(100) not null,
    rideable_type      varchar(100),
    started_at         timestamp,
    ended_at           timestamp,
    start_station_name varchar(100),
    start_station_id   varchar(100),
    end_station_name   varchar(100),
    end_station_id     varchar(100),
    start_lat          double precision,
    start_lng          double precision,
    end_lat            double precision,
    end_lng            double precision,
    member_casual      varchar(50)
);
```


```python
import psycopg2
import pandas as pd
import os

# Config PostgreSQL
db_config = {
    "dbname": "cyclist",
    "user": "postgres",
    "password": "admin",
    "host": "localhost",
    "port": 5432        
}

# path
csv_dir = "./data"

try:
    # connect db
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()        
    for filename in os.listdir(csv_dir):
        # open with pandas
        df = pd.read_csv("./data/" + filename)
        # clean null data
        df.dropna(subset=['start_lat', 'start_lng', 'end_lng', 'end_lat'], inplace=True)
        # save csv
        df.to_csv("./data/" + filename, index=False)
        # open csv file
        with open(csv_dir + '/' + filename, 'r') as f:
            next(f) # ignore header
            # copy from cvs to database table
            cursor.copy_from(f, 'tripdata_temp', sep=',', columns=(
    'ride_id', 'rideable_type', 'started_at', 'ended_at', 'start_station_name', 'start_station_id', 'end_station_name', 'end_station_id', 'start_lat', 'start_lng', 'end_lat', 'end_lng', 'member_casual'))
    # save the change
    conn.commit()
    print("Data insert successful")

except Exception as e:
    print(f"Error: {e}")

finally:
    if conn:
        cursor.close()
        conn.close()

```

## Insert data to tripdata table

We insert data to tripdata table and ignore duplicate data by ride_id

```SQl
INSERT INTO tripdata
SELECT *
FROM tripdata_temp
ON CONFLICT (ride_id) DO NOTHING;
```

## Start analyse data

Let start answer question using view table

### How are the start and end points of riders distributed across different membership groups?

To answer the question above, we will extract 10 data points for each membership group, calculated from the starting points to the endpoints with the highest number of trips.

First, we will process the data for casual customers.
```sql
with visited_location as (
  (Select start_station_name as location from tripdata_view 
                                         where start_station_name != '' 
                                           and member_casual = 'casual')
  union all
  (Select end_station_name as location from tripdata_view 
                                       where end_station_name != '' 
                                         and  member_casual = 'casual')
)

Select location, count(location) as total from visited_location
group by location order by total desc limit 10
```

Result:

| Location                                 | Total  |
|------------------------------------------|--------|
| Streeter Dr & Grand Ave                  | 105361 |
| DuSable Lake Shore Dr & Monroe St        | 65695  |
| Michigan Ave & Oak St                    | 50778  |
| DuSable Lake Shore Dr & North Blvd       | 49558  |
| Millennium Park                          | 46536  |
| Shedd Aquarium                           | 39818  |
| Dusable Harbor                           | 35049  |
| Theater on the Lake                      | 34848  |
| Michigan Ave & 8th St                    | 25733  |
| Adler Planetarium                        | 23870  |


Next, we will process the data for member customers.
```sql
with visited_location as (
  (Select start_station_name as location from tripdata_view 
                                         where start_station_name != '' 
                                           and member_casual = 'member')
  union all
  (Select end_station_name as location from tripdata_view 
                                       where end_station_name != '' 
                                         and  member_casual = 'member')
)

Select location, count(location) as total from visited_location
group by location order by total desc limit 10
```

Result:

| Location                      | Total |
|-------------------------------|-------|
| Kingsbury St & Kinzie St      | 56715 |
| Clinton St & Washington Blvd  | 53279 |
| Clinton St & Madison St       | 48378 |
| Clark St & Elm St             | 48032 |
| Wells St & Concord Ln         | 40471 |
| Wells St & Elm St             | 39405 |
| Clinton St & Jackson Blvd     | 39189 |
| Canal St & Madison St         | 37453 |
| State St & Chicago Ave        | 37225 |
| University Ave & 57th St      | 36776 |

### Conclusion:

The locations in the upper section of casual riders are mainly focused on tourism, entertainment, culture, and nature, often near Lake Michigan. These areas, such as Millennium Park and Shedd Aquarium, attract tourists and provide outdoor spaces, parks, and cultural events. They are ideal for recreation and relaxation, with a vibrant atmosphere that caters to those looking to explore and enjoy the city's beauty.

In contrast, the locations in the lower section of member riders are situated in the city’s commercial and transportation hubs. These areas, like Clinton St & Washington Blvd and State St & Chicago Ave, are centers of business and transit, primarily serving residents and commuters. They focus on economic activity, with many office buildings, retail spaces, and major transit connections.

The key difference lies in their purpose: the upper section caters to leisure, tourism, and outdoor activities, while the lower section supports commerce, transportation, and daily commuting.

### What is the distribution of ride frequency across the months of the year for different membership groups?

We will summarize the trip data by month for each customer group.

First, with casual group:
```sql
Select month, count(ride_id) from tripdata_view 
where member_casual = 'casual' group by month order by month
```

| Month | Total   |
|-------|---------|
| 1     | 2,216   |
| 2     | 46,963  |
| 3     | 82,268  |
| 4     | 131,431 |
| 5     | 230,370 |
| 6     | 300,195 |
| 7     | 319,583 |
| 8     | 317,563 |
| 9     | 345,879 |
| 10    | 215,956 |
| 11    | 92,848  |
| 12    | 38,272  |

Next, with member group:
```sql
Select month, count(ride_id) from tripdata_view 
where member_casual = 'member' group by month order by month
```

| Month | Total  |
|-------|--------|
| 1     | 13168  |
| 2     | 175883 |
| 3     | 219023 |
| 4     | 283115 |
| 5     | 378339 |
| 6     | 409411 |
| 7     | 428284 |
| 8     | 437263 |
| 9     | 474285 |
| 10    | 399759 |
| 11    | 241929 |
| 12    | 139961 |

Next, we visualize the data

First, we install the matplotlib library

```bash
pip install matplotlib
```



```python
import matplotlib.pyplot as plt

# Data
months = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
totals = [2216, 46963, 82268, 131431, 230370, 300195, 319583, 317563, 345879, 215956, 92848, 38272
]

# Plotting the bar chart
plt.figure(figsize=(10,6))
plt.bar(months, totals, color='skyblue')
plt.xlabel('Month')
plt.ylabel('Total')
plt.title('Total by Month Of Casual Riders')
plt.xticks(months)
plt.show()
```


    
![png](analysis_files/analysis_9_0.png)
    



```python
import matplotlib.pyplot as plt

# Data
months = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
totals = [13168, 175883, 219023, 283115, 378339, 409411, 428284, 437263, 474285, 399759, 241929, 139961
]

# Plotting the bar chart
plt.figure(figsize=(10,6))
plt.bar(months, totals, color='skyblue')
plt.xlabel('Month')
plt.ylabel('Total')
plt.title('Total by Month Of Member Riders')
plt.xticks(months)
plt.show()
```


    
![png](analysis_files/analysis_10_0.png)
    


### Conclusion

Looking at the two charts, we can see that the months from June to September are the busiest in terms of the number of visitors.

From June to September, the number of bicycle riders in Chicago is significantly higher compared to other months. This trend is evident for both casual riders and member riders, with the peak usage occurring in these months.

The primary reason for this increase is the weather. Chicago experiences warm and pleasant temperatures during summer (around 20–30°C), making it ideal for outdoor activities such as cycling. In contrast, winter months (December to February) are extremely cold, with frequent snow and temperatures below freezing, which discourages cycling.

Additionally, summer is the peak tourist season in Chicago. Visitors often use bicycles to explore popular attractions like Lake Michigan, Millennium Park, and bike-friendly routes. The city also hosts numerous outdoor events and festivals during this time, further boosting bicycle usage.

Conversely, in the colder months, shorter daylight hours and busy work or school schedules reduce the frequency of bike usage. The challenging weather conditions make other modes of transportation more favorable.

In summary, the combination of favorable weather, high tourism activity, and summer events makes June to September the most active period for bicycle riders in Chicago, while the winter months see a significant decline.

Data shows that from December to February, member riders account for 11.77% of the total yearly member rides, while casual riders only account for 5.1% of the total yearly casual rides. The higher number of rides by member riders compared to casual riders during the winter months can be explained by several factors: Member riders use bikes as a daily mode of transport, while casual riders use bikes for leisure, making them less likely to ride in cold weather. Member riders are also committed to their membership, whereas casual riders are not. Additionally, the harsh winter climate reduces outdoor activities, especially for casual riders.

## What is the distribution of ride frequency across the days of the week for different membership groups?

We will summarize data by days of week

We summarize the data:
```sql
Select CASE day
           WHEN 0 THEN 'Sunday'
           WHEN 1 THEN 'Monday'
           WHEN 2 THEN 'Tuesday'
           WHEN 3 THEN 'Wednesday'
           WHEN 4 THEN 'Thursday'
           WHEN 5 THEN 'Friday'
           WHEN 6 THEN 'Saturday'
       END AS day_name,
    count(case member_casual when 'casual' then 1 end) as total_casual,
    count(case member_casual when 'member' then 1 end) as total_member
from tripdata_view  group by day order by day
```

| Day       | Casual Total | Member Total |
|-----------|--------------|--------------|
| Sunday    | 366545       | 409083       |
| Monday    | 249296       | 516802       |
| Tuesday   | 228803       | 553605       |
| Wednesday | 264558       | 586519       |
| Thursday  | 260438       | 550100       |
| Friday    | 312068       | 513329       |
| Saturday  | 441836       | 470982       |



```python
import matplotlib.pyplot as plt
import numpy as np  

# Data
day_name = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday']
total_casual = [366545, 249296, 228803, 264558, 260438, 312068, 441836]
total_member = [409083, 516802, 553605, 586519, 550100, 513329, 470982]

X_axis = np.arange(len(day_name))

# Plotting the bar chart
plt.figure(figsize=(10,6))
plt.bar(X_axis - 0.2, total_casual, width=0.4, color='skyblue', label='Casual Riders')
plt.bar(X_axis + 0.2, total_member, width=0.4, color='orange', label='Member Riders')
plt.xlabel('Day')
plt.ylabel('Total')
plt.title('Total by day of weeks')
plt.legend()
plt.xticks(X_axis, day_name)
plt.show()
```


    
![png](analysis_files/analysis_13_0.png)
    


### Conclusion:

The charts illustrate the weekly trip counts for two groups of users in Chicago: casual users and members. Casual users show the highest trip counts on Saturday, followed by Sunday, indicating their preference for using the service on weekends. In contrast, members have the highest activity on Wednesday, followed by Tuesday and Thursday, which fall on weekdays. This difference reflects distinct user behaviors.

Casual users, likely tourists or occasional riders, prioritize leisure and recreational trips, making weekends the peak period for their service usage. These users are less consistent and tend to ride when they have more free time. On the other hand, members are more regular riders who rely on the service primarily for commuting purposes during the workweek. Their activity peaks midweek, likely aligning with their work schedules and daily routines.

The disparity between the two groups highlights their unique travel patterns. While casual users benefit from convenient weekend accessibility, members require reliable service during weekdays for consistent use. Understanding these differences allows service providers to tailor their offerings, ensuring adequate availability and resources during peak times for each group. This segmentation provides valuable insights for improving service efficiency and meeting user expectations.

## What is the distribution of ride frequency across the hours of the day for different membership groups?

We will summarize data by hours of day

First, with casual group:

```sql
select  hour, count(hour) as total from tripdata_view 
where member_casual='casual' group by  hour order by hour
```

| Hour | Total Casual | Total Member |
|------|--------------|--------------|
| 0    | 34914        | 31972        |
| 1    | 23214        | 19533        |
| 2    | 14395        | 11052        |
| 3    | 8023         | 7575         |
| 4    | 6326         | 8384         |
| 5    | 11901        | 33221        |
| 6    | 28276        | 100232       |
| 7    | 52601        | 193630       |
| 8    | 74491        | 246581       |
| 9    | 75574        | 168460       |
| 10   | 93301        | 149032       |
| 11   | 118889       | 174036       |
| 12   | 140669       | 198967       |
| 13   | 146748       | 198718       |
| 14   | 152850       | 198929       |
| 15   | 167963       | 243559       |
| 16   | 188957       | 332566       |
| 17   | 201503       | 381197       |
| 18   | 171968       | 298687       |
| 19   | 126838       | 209943       |
| 20   | 92711        | 147093       |
| 21   | 77128        | 113119       |
| 22   | 67292        | 82183        |
| 23   | 47012        | 51751        |




```python
import matplotlib.pyplot as plt
import numpy as np 

# Data
hour = [0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23]
total_casual = [34914, 23214, 14395, 8023, 6326, 11901, 28276, 52601, 74491, 75574, 93301, 118889, 140669, 146748, 152850, 167963, 188957, 201503, 171968, 126838, 92711, 77128, 67292, 47012]
total_member = [31972, 19533, 11052, 7575, 8384, 33221, 100232, 193630, 246581, 168460, 149032, 174036, 198967, 198718, 198929, 243559, 332566, 381197, 298687, 209943, 147093, 113119, 82183, 51751]

X_axis = np.arange(len(hour))

# Plotting the bar chart
plt.figure(figsize=(10,6))
plt.bar(X_axis - 0.2, total_casual, width=0.4, color='skyblue', label='Casual Riders')
plt.bar(X_axis + 0.2, total_member, width=0.4, color='orange', label='Member Riders')
plt.xlabel('Hour')
plt.ylabel('Total')
plt.title('Total by hour of days')
plt.legend()
plt.xticks(X_axis, hour)
plt.show()
```


    
![png](analysis_files/analysis_16_0.png)
    



### Conclusion

Member riders' peak hours are between 7–10 AM and 4–7 PM, aligning with typical commuting times for work.  

On the other hand, casual riders peak between 3–8 PM, a time when people usually go out for entertainment and leisure activities.

## What is the distribution of ride type for different membership groups?

We query the rideable type and the number of rides for each type:

```sql
select  rideable_type,
        count(case member_casual when 'casual' then 1 end) as total_casual,
        count(case member_casual when 'member' then 1 end) as total_member
from tripdata_view
 group by rideable_type
```

| rideable_type    | total_casual | total_member |
|------------------|--------------|--------------|
| classic_bike     | 960028       | 1702348      |
| electric_bike    | 1078301      | 1838950      |
| electric_scooter | 85215        | 59122        |



```python
import matplotlib.pyplot as plt
import numpy as np 

rideable_type = ['classic_bike', 'electric_bike', 'electric_scooter']
total_casual = [960028, 1078301, 85215]
total_member = [1702348, 1838950, 59122]

X_axis = np.arange(len(rideable_type))

# Plotting the bar chart
plt.figure(figsize=(10,6))
plt.bar(X_axis - 0.2, total_casual, width=0.4, color='skyblue', label='Casual Riders')
plt.bar(X_axis + 0.2, total_member, width=0.4, color='orange', label='Member Riders')
plt.xlabel('Rideable Type')
plt.ylabel('Total')
plt.title('Total by rideable type')
plt.legend()
plt.xticks(X_axis, rideable_type)
plt.show()
```


    
![png](analysis_files/analysis_19_0.png)
    


### Conclusion:

Calculate the ratio of electric vehicles to regular vehicles for the two groups: casual and member.

```sql
select member_casual, round(count(case rideable_type when 'electric_bike' then 1 when 'electric_scooter' then 1 end)*100.0/count(rideable_type),2) as rate
from tripdata_view group by member_casual
```

| member_casual | rate  |
|---------------|-------|
| Casual        | 54.79 |
| Member        | 52.72 |

The casual group has an electric vehicle usage rate of 54.79%, higher than the Member group.
The member group’s electric vehicle usage rate is slightly lower, at 52.72%.
Overall, the difference between the two groups in electric vehicle usage is not significant (only about 2.07%). However, the casual group tends to use electric vehicles slightly more than the member group. This could be related to the behavior or purpose of vehicle use specific to each group.

## What is the distribution of travel distance for each membership group?

We will get the data from database and draw using seaborn lib and sqlalchemy

```bash
pip3 install seaborn
pip3 install sqlalchemy
```

Because the max distance is super high (5764613 meters), we will only cover 99 percents of the distance in this case

```sql
select percentile_cont(0.99) within group (order by distance) as p99 from tripdata_view
```

p99 = 9453 (meters)

We also observe some trip have distance very low(<10m) or equal zero. They must be invalid and should be removed.



```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from sqlalchemy import create_engine

# connect with dsn
engine = create_engine('postgresql://postgres:admin@localhost:5432/cyclist')

df = pd.read_sql_query('select member_casual, distance from tripdata_view where distance < 9453 and distance > 10',con=engine)

casual_data = df[df['member_casual'] == 'casual']['distance']
member_data = df[df['member_casual'] == 'member']['distance']

# Plotting overlapping histograms with KDE
sns.histplot(data=casual_data, bins=50, color='blue', alpha=0.5, kde=True, label='Casual')
sns.histplot(data=member_data, bins=50, color='orange', alpha=0.5, kde=True, label='Member')

# Adding labels and legend
plt.xlabel('Distance')
plt.ylabel('Density')
plt.legend()

plt.show()
```


    
![png](analysis_files/analysis_22_0.png)
    


Conclusion:
## We will calculate the average travel distance of the two groups: casual and member.

```sql
select member_casual, avg(distance) as avg_distance from tripdata_view group by member_casual
```

| member_casual | avg_distance |
|---------------|--------------|
| Casual        | 2122.52      |
| Member        | 2190.84      |

The Member group has an average travel distance of 2190.84, which is higher than the Casual group's 2122.52, with a difference of approximately 68.32. This suggests that the Member group travels farther on average than the Casual group. This difference may be related to the purpose or frequency of vehicle use within each group, for example, the Member group may travel more due to factors such as work or a fixed travel schedule.

## What is the distribution of ride duration for each membership group?
Similar to above, we will plot the time distribution chart.  
We can see that the median is 9.7 minutes, p99 is 95.3, and the max is 1509.

```sql
select percentile_cont(0.99) within group (order by duration_minutes) as p99 from tripdata_view
```

We will discard some data with travel times less than 1 minute as it is invalid data.


```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from sqlalchemy import create_engine

# connect with dsn
engine = create_engine('postgresql://postgres:admin@localhost:5432/cyclist')

df = pd.read_sql_query('select member_casual, duration_minutes from tripdata_view where duration_minutes < 95.3 and duration_minutes > 1',con=engine)

casual_data = df[df['member_casual'] == 'casual']['duration_minutes']
member_data = df[df['member_casual'] == 'member']['duration_minutes']

# Plotting overlapping histograms with KDE
sns.histplot(data=casual_data, bins=50, color='blue', alpha=0.5, kde=True, label='Casual')
sns.histplot(data=member_data, bins=50, color='orange', alpha=0.5, kde=True, label='Member')

# Adding labels and legend
plt.xlabel('Duration minutes')
plt.ylabel('Density')
plt.legend()

plt.show()
```


    
![png](analysis_files/analysis_25_0.png)
    


Conclusion:
## We will calculate the average travel time for the two groups: casual and member.

```sql
select member_casual, avg(duration_minutes) as avg_duration from tripdata_view group by member_casual
```

| member_casual | avg_duration |
|---------------|--------------|
| Casual        | 21.17        |
| Member        | 12.25        |

The Casual group has an average travel time of 21.17 minutes.  
The Member group has an average travel time of 12.25 minutes.  
The Casual group may take longer but less frequent trips, such as traveling or engaging in recreational activities, which causes each trip to be longer. Additionally, the Casual group may choose less efficient routes or schedules, leading to more time spent on longer trips. Meanwhile, the Member group tends to make shorter and more efficient trips.

## Action
- How do annual members and casual riders use Cyclistic bikes differently?

Based on the analysis, we can see the differences between the two groups:

The starting and ending points of the member group tend to be shopping malls, transportation hubs, business centers, and public transit stations, primarily serving residents and commuters. In contrast, the casual group mainly focuses on tourism, entertainment, culture, and nature, often near Lake Michigan, attracting tourists.

The weekly trip patterns show that the casual group has the highest number of trips on Saturdays, followed by Sundays, indicating a preference for using the service on weekends. On the other hand, the member group is most active on Wednesdays, followed by Tuesdays and Thursdays, which fall on weekdays.

The proportion of electric bike usage in the casual group is slightly higher than in the member group, with a difference of only about 2.07%.

The average trip duration for the casual group is 21.17 minutes, while for the member group, it is 12.25 minutes.The casual group tends to take longer trips but less frequently, such as for travel or recreational activities, leading to extended trip durations. Additionally, casual riders may choose less efficient routes or schedules, resulting in longer travel times.

- Why would casual riders buy Cyclistic annual memberships?

Due to their frequent travel needs, their destinations are often business locations, workplaces, and shopping centers.  

The member group travels on all days of the week, especially on workdays.  

From these reasons, we can conclude that the member group subscribes to an annual plan because they have a high commuting demand and aim to optimize costs.

### Conclusion

How can Cyclistic use digital media to influence casual riders to become members?

To meet consumer needs, we will deliver our message to the public by:

- Advertising in office areas, shopping centers, and public transit stations.

- Promoting the availability of bike stations at busy locations in the city center.

- Highlighting that during peak hours, our service is always ready to meet user demand.

- Showing customers that the compact bike offers high speed, optimizing travel time within the city.
