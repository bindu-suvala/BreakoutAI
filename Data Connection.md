import pandas as pd

from google.oauth2 import service_account

from googleapiclient.discovery import build

import os

# Function to read data from CSV file
def read_csv(file_path):
    """
    Reads the CSV file and returns the data as a pandas DataFrame.
    
    :param file_path: Path to the CSV file
    :return: pandas DataFrame containing the CSV data
    """
    try:
        # Using pandas to read the CSV file
        data = pd.read_csv(file_path)
        print(f"CSV file read successfully. Shape of data: {data.shape}")
        return data
    except Exception as e:
        print(f"Error reading CSV file: {e}")
        return None


# Function to authenticate with Google Sheets API
def authenticate_google_sheets(credentials_file):
    """
    Authenticates with Google Sheets API using service account credentials.
    
    :param credentials_file: Path to the service account JSON file
    :return: Google Sheets API service object
    """
    try:
        # Using service account credentials
        credentials = service_account.Credentials.from_service_account_file(credentials_file)
        service = build('sheets', 'v4', credentials=credentials)
        print("Google Sheets API authenticated successfully.")
        return service
    except Exception as e:
        print(f"Error during authentication: {e}")
        return None


# Function to read data from Google Sheets
def read_google_sheet(service, sheet_id, range_name):
    """
    Reads data from a specific range in the Google Sheets document.
    
    :param service: The authenticated Google Sheets service object
    :param sheet_id: Google Sheets document ID
    :param range_name: The range of cells to read (e.g., 'Sheet1!A1:D100')
    :return: List of rows from the sheet (as a list of lists)
    """
    try:
        sheet = service.spreadsheets()
        result = sheet.values().get(spreadsheetId=sheet_id, range=range_name).execute()
        values = result.get('values', [])
        
        if not values:
            print("No data found in the specified range.")
        else:
            print(f"Data from Google Sheet fetched successfully. Found {len(values)} rows.")
        
        return values
    except Exception as e:
        print(f"Error reading Google Sheets data: {e}")
        return None


# Example usage of the functions
if __name__ == "__main__":
    # Read CSV Example
    csv_file_path = "emails.csv"  # Provide your CSV file path here
    email_data_csv = read_csv(csv_file_path)
    if email_data_csv is not None:
        print(email_data_csv.head())  # Print first few rows of CSV data

    # Read Google Sheets Example
    google_sheet_id = "your-google-sheet-id"  # Replace with your Google Sheets document ID
    sheet_range = "Sheet1!A1:E100"  # Define the range of data you want to fetch
    
    # Path to your service account credentials JSON file
    credentials_json = "path_to_service_account.json"
    
    google_service = authenticate_google_sheets(credentials_json)
    if google_service is not None:
        email_data_google = read_google_sheet(google_service, google_sheet_id, sheet_range)
        if email_data_google is not None:
            for row in email_data_google:
                print(row)  # Print the rows fetched from the Google Sheet
