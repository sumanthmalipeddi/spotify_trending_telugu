# Spotify - Trending Telugu Songs List
## Story Line:
**Client:** I need a list of trending Telugu songs and to collect the album, artists and song details in the database for future analytics purposes.
<br>**Sumanth:** Is there any particular platform you regularly use to listen to the songs?
<br> **Client:** Yeah I do use a lot of platforms like Wynk Music, JioSaavn, YouTube Music and presently **Spotify**.
<br> **Sumanth:** We shall take Spotify for the project and collect the trending song details along with albums and artists.
# Architecture
![Main](https://github.com/sumanthmalipeddi/spotify_trending_telugu/blob/main/dfdf.drawio.png)
# Technology Used
* Programming Language - Python
* Amazon Web Services Cloud Platform
  * AWS CloudWatch: Monitor AWS resources and applications in real-time to gain insights into system performance.
  * AWS Lambda: Run code without provisioning or managing servers, responding to events across AWS services.
  * AWS S3: Store and retrieve data securely at any scale, with high availability and durability.
  * AWS Triggers: Automate actions in response to events across various AWS services for streamlined workflows.
  * AWS Glue (Crawler): Automatically discover, catalogue, and transform your data to make it readily available for analysis.
  * AWS Athena: Query data in Amazon S3 using standard SQL without the need for managing infrastructure, enabling quick and cost-effective analysis.
# Data Base Used
Spotify - [Trending Telugu Songs](https://open.spotify.com/playlist/37i9dQZF1DWTt3gMo0DLxA)
* Here we can build the database by utilising **spotipy** API the gateway to seamless integration with Spotify's vast music catalogue and powerful features. 

```
#For Jupyter Notebook Code Implementation
pip install spotipy
```
# Budget Allocation
* With the service of AWS Cloud Watch we have allocated budget limits so resources will be under watch
# Data Extraction
* Amazon Lambda services do not have a pip function so we can add layers by manually uploading spotipy layer and extracting the data from the API to the Amazon S3 (to_processed) bucket **in raw folder**.

### Important Note: Never Disclose your cliend_id and cliend_secret ( In the configurations we can add environmental variables for secured usage)
```
import json
import os
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials
import boto3
from datetime import datetime

def lambda_handler(event, context):
    client_id = os.environ.get('client_id')
    client_secret = os.environ.get('client_secret')
    client_credentials_manager = SpotifyClientCredentials(client_id=client_id, client_secret=client_secret)
    sp = spotipy.Spotify(client_credentials_manager= client_credentials_manager)
    playlists = sp.user_playlists('spotipy')
    
    playlist_link = "https://open.spotify.com/playlist/37i9dQZF1DWTt3gMo0DLxA"
    playlist_URI = playlist_link.split('/')[-1]
    
    spotify_data = sp.playlist_tracks(playlist_URI)
    
    client = boto3.client('s3')
    
    filename = "spotify_raw_" + str(datetime.now()) + ".json"
    
    client.put_object(
        Bucket = "spotify-telugu-etl-project",
        Key = "raw_data/to_processed/" + filename,
        Body = json.dumps(spotify_data)
        )
```
# Data Transformation
```
#For Jupyter Notebook - code transformations
pip install pandas
```
```
import pandas as pd
```
* In the Amazon Lambda pip install does not work so we can add the layer of pandas library in their create layers section.
```
import json
import boto3
from datetime import datetime
from io import StringIO
import pandas as pd

def album(data):
    
    album_list = []
    for row in data['items']:
        
        album_id = row['track']['album']['id']
        album_name = row['track']['album']['name']
        album_release_date = row['track']['album']['release_date']
        album_total_tracks = row['track']['album']['total_tracks']
        album_url = row['track']['album']['external_urls']['spotify']
        album_element = {'album_id' : album_id,'album_name':album_name,'release_date':album_release_date,
                                 'total_tracks':album_total_tracks,'url':album_url}
        album_list.append(album_element)
        
    return album_list

def artist(data):
    
    artist_list = []
    
    for row in data['items']:
        for key, value in row.items():
            if key == 'track':
                for artist in value['artists']:
                    artist_dict = {'artist_id' : artist['id'],'artist_name' : artist['name'], 
                                       'external_url' : artist['external_urls']['spotify']}
                    artist_list.append(artist_dict)
    
    return artist_list
    
def song(data):
    song_list = []
    for row in data['items']:
        song_id = row['track']['id']
        song_name = row['track']['name']
        song_duration = row['track']['duration_ms']
        song_url = row['track']['external_urls']['spotify']
        song_popularity = row['track']['popularity']
        song_added = row['added_at']
        #album code importing
        album_id = row['track']['album']['id']
        artist_id = row['track']['album']['artists'][0]['id']
        song_element = {'song_id':song_id,'song_name':song_name,'duration_ms':song_duration,'url':song_url,
                        'popularity':song_popularity,'song_added':song_added,'album_id':album_id,
                        'artist_id':artist_id
                       }
        song_list.append(song_element)
    
    return song_list
    

def lambda_handler(event, context):
    
    s3 = boto3.client('s3')
    Bucket = "spotify-telugu-etl-project"
    Key = "raw_data/to_processed/"
    
    spotify_data = []
    spotify_keys = []
    
    for file in s3.list_objects(Bucket = Bucket, Prefix = Key)['Contents']:
        file_key = file['Key']
        if file_key.split('.')[-1] == 'json':
            response = s3.get_object(Bucket = Bucket, Key = file_key)
            content = response['Body']
            jsonObject = json.loads(content.read())
            spotify_data.append(jsonObject)
            spotify_keys.append(file_key)
            
    for data in spotify_data:
        album_list = album(data)
        artist_list = artist(data)
        song_list = song(data)
        
        #album
        album_df = pd.DataFrame.from_dict(album_list)
        album_df = album_df.drop_duplicates(subset=['album_id'])
        
        #artist
        artist_df = pd.DataFrame.from_dict(artist_list)
        artist_df = artist_df.drop_duplicates(subset=['artist_id'])
        
        #song
        song_df = pd.DataFrame.from_dict(song_list)

        # Transform 'release_date' column in album_df to datetime
        album_df['release_date'] = pd.to_datetime(album_df['release_date'],format='mixed')
        # Transform 'song_added' column in song_df to datetime
        song_df['song_added'] = pd.to_datetime(song_df['song_added'],format='mixed')
```
* After the Data Frames are correctly cleansed we need to import them into the Amazon S3 transformed folder with subfolders of songs, albums and artists folders.
```
        #putting the songs data into s3 bucket
        songs_key = "transformed_data/songs_data/songs_transformed_" + str(datetime.now()) + ".csv"
        song_buffer=StringIO()
        song_df.to_csv(song_buffer, index=False)
        song_content = song_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=songs_key, Body=song_content)
        
        album_key = "transformed_data/album_data/album_transformed_" + str(datetime.now()) + ".csv"
        album_buffer=StringIO()
        album_df.to_csv(album_buffer, index=False)
        album_content = album_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=album_key, Body=album_content)
        
        artist_key = "transformed_data/artist_data/artist_transformed_" + str(datetime.now()) + ".csv"
        artist_buffer=StringIO()
        artist_df.to_csv(artist_buffer, index=False)
        artist_content = artist_buffer.getvalue()
        s3.put_object(Bucket=Bucket, Key=artist_key, Body=artist_content)
        
```
* We also need to remove the file from the to_processed folder to the processed folder **in raw** and delete the older files
```
 s3_resource = boto3.resource('s3')
    for key in spotify_keys:
        copy_source = {
            'Bucket': Bucket,
            'Key': key
        }
        s3_resource.meta.client.copy(copy_source, Bucket, 'raw_data/processed/' + key.split("/")[-1])    
        s3_resource.Object(Bucket, key).delete()
```
# Data Load
* After the successful creation of subfolders we are now loading data into Amazon Athena by Amazon Glue-Crawler by naming the new database **spotify_db** and tabling **songs, albums** and **artists**.
```
#MySQL
SELECT * FROM "spoitfy_db"."songs" LIMIT 10; #Execute Each Line
SELECT * FROM "spoitfy_db"."albums" LIMIT 10; #Execute Each Line
SELECT * FROM "spoitfy_db"."artists" LIMIT 10; #Execute Each Line
```
### Here, the Successful retrieval of Data from the tables closes the Data Engineering Part of analytics.
