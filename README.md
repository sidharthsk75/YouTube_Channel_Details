import streamlit as st
from PIL import Image
from googleapiclient.discovery import build
import requests
from googleapiclient.errors import HttpError
import isodate


def convert_iso8601_to_hhmmss(duration):
    # Parse the duration using the built-in Python libraries
    duration_obj = isodate.parse_duration(duration)

    # Extract hours, minutes, and seconds
    hours = duration_obj.days * 24 + duration_obj.seconds // 3600
    minutes = (duration_obj.seconds % 3600) // 60
    seconds = duration_obj.seconds % 60

    # Format the duration in HH:MM:SS
    formatted_duration = '{:02d}:{:02d}:{:02d}'.format(hours, minutes, seconds)
    return formatted_duration


def channel(channel_id):
    api_key = 'AIzaSyCOE_Ofg9G6GBqH9uIEcZVMqTmTV0Grvhs'
    youtube = build('youtube', 'v3', developerKey=api_key)

    request = youtube.channels().list(
        part="snippet,contentDetails,statistics",
        id=channel_id
    )
    response = request.execute()

    channel_data = {
        'Channel_Name': response['items'][0]['snippet']['title'],
        'Channel_Id': channel_id,
        'Subscription_Count': int(response['items'][0]['statistics']['subscriberCount']),
        'Channel_Views': int(response['items'][0]['statistics']['viewCount']),
        'Channel_Description': response['items'][0]['snippet']['description'],
        'channel_playlist': response['items'][0]['contentDetails']['relatedPlaylists']['uploads']

    }

    return channel_data


def get_video_ids(channel_id):
    api_key = 'AIzaSyCOE_Ofg9G6GBqH9uIEcZVMqTmTV0Grvhs'
    youtube = build('youtube', 'v3', developerKey=api_key)

    video_IDs = []
    video_request = youtube.channels().list(id=channel_id, part='contentDetails').execute()
    playlist_id = video_request['items'][0]['contentDetails']['relatedPlaylists']['uploads']
    next_page_token = None

    while True:
        video_response = youtube.playlistItems().list(playlistId=playlist_id,
                                                      part='snippet',
                                                      maxResults=10,
                                                      pageToken=next_page_token).execute()
        for item in video_response['items']:
            video_IDs.append(item['snippet']['resourceId']['videoId'])

        next_page_token = video_response.get('nextPageToken')
        if not next_page_token:
            break

    return video_IDs


def video(video_id):
    api_key = "AIzaSyCOE_Ofg9G6GBqH9uIEcZVMqTmTV0Grvhs"
    youtube = build('youtube', 'v3', developerKey=api_key)

    request = youtube.videos().list(
        part="snippet,contentDetails,statistics",
        id=video_id
    )
    response = request.execute()

    video_data = {
        'Video_Id': response['items'][0]['id'],
        'Video_Name': response['items'][0]['snippet']['title'],
        'Video_Description': response['items'][0]['snippet']['description'],
        'PublishedAt': response['items'][0]['snippet']['publishedAt'],
        'View_Count': int(response['items'][0]['statistics']['viewCount']),
        'Like_count': int(response['items'][0]['statistics']['likeCount']),
        'Favorite_count': int(response['items'][0]['statistics']['likeCount']),
        'Comment_Count': int(response['items'][0]['statistics']['commentCount']),
        'Duration': convert_iso8601_to_hhmmss(response['items'][0]['contentDetails']['duration']),
        'Thumbnail': response['items'][0]['snippet']['thumbnails']['default']['url']
    }

    # Check for the existence of the 'tags' key
    if 'tags' in response['items'][0]['snippet']:
        video_data['Tags'] = response['items'][0]['snippet']['tags']
    else:
        video_data['Tags'] = []

    return video_data


def get_comment_ids(video_id):
    api_key = "AIzaSyCOE_Ofg9G6GBqH9uIEcZVMqTmTV0Grvhs"
    youtube = build('youtube', 'v3', developerKey=api_key)

    comment_ids = []
    request = youtube.commentThreads().list(
        part='snippet',
        videoId=video_id,
        maxResults=10  # Adjust as needed
    )

    try:
        response = request.execute()
    except HttpError as err:
        # Handle the 403 error and skip processing comments for this video
        if err.resp.status == 403:
            print(f"Comments for video {video_id} are disabled.")
            return []

    for item in response['items']:
        comment_ids.append(item['id'])

    return comment_ids


def comment(comment_id):
    api_key = "AIzaSyCOE_Ofg9G6GBqH9uIEcZVMqTmTV0Grvhs"
    youtube = build('youtube', 'v3', developerKey=api_key)

    try:
        request = youtube.comments().list(
            part="snippet",
            parentId=comment_id
        )
        response = request.execute()

        if 'items' in response and len(response['items']) > 0:
            comment_data = {
                'comment_id': response['items'][0]['id'],
                'Comment_Author': response['items'][0]['snippet']['authorChannelId'],
                'Comment_PublishedAt': response['items'][0]['snippet']['updatedAt'],
                'Comment_Text': response['items'][0]['snippet']['textOriginal']
            }
        else:
            # Handle the case where no comment data is found
            comment_data = {}

    except IndexError:
        # Handle the IndexError and continue processing
        comment_data = {}

    return comment_data


def main():
    channel_id = input("Enter YouTube channel ID: ")
    channel_data = channel(channel_id)
    video_ids = get_video_ids(channel_id)[:10]  # Retrieve the first 10 video IDs

    video_details = []
    comment_data_all = []  # Collect all comment data from all videos

    for vid_id in video_ids:
        print(f"Processing video ID: {vid_id}")

        # Step 1: Get video data for the 10 video IDs
        vid_data = video(vid_id)
        if vid_data:
            video_details.append(vid_data)

            # Step 2: Get comment data for each video
            comment_ids = get_comment_ids(vid_id)
            comments_data = []
            if comment_ids:
                for comment_id in comment_ids:
                    comment_data = comment(comment_id)
                    if comment_data:
                        comments_data.append(comment_data)
                        comment_data_all.append(comment_data)

            vid_data['Comments'] = comments_data

    data = {
        'Channel_Data': channel_data,
        'Video_Details': video_details,
        'Comment_Data_All_Videos': comment_data_all
    }

    return data


# Function to display channel details
def display_channel_details(channel_id):
    channel_data = channel(channel_id)
    st.write("Channel Details:")
    st.write(channel_data)


# Function to extract and transform YouTube data
def extract_and_transform(channel_id):
    data = main()
    st.write("Extracted Data:")
    st.write(data)


# Streamlit app configuration
st.set_page_config(page_title="YouTube Data App", layout="wide")

# Sidebar navigation
with st.sidebar:
    selected = st.selectbox("Navigation", ["Home", "Extract and Transform"])

# Main content based on user selection
if selected == "Home":
    st.title("Welcome to YouTube Data App")
    st.write("Enter YouTube Channel ID on the sidebar to get started.")

elif selected == "Extract and Transform":
    st.title("Extract and Transform YouTube Data")
    channel_id = st.text_input("Enter YouTube Channel ID")

    if st.button("Get Channel Details"):
        if channel_id:
            display_channel_details(channel_id)

        else:
            st.warning("Please enter a YouTube Channel ID")

    if st.button("Extract and Transform"):
        if channel_id:
            extract_and_transform(channel_id)
        else:
            st.warning("Please enter a YouTube Channel ID to extract data.")
