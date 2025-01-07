import os
import json
from datetime import datetime
import requests
import boto3

class SecretManager:
    def __init__(self, secret_name, region_name):
        self.secret_name = secret_name
        self.region_name = region_name
        self.client = boto3.client('secretsmanager', region_name=self.region_name)

    def get_secret(self):
        response = self.client.get_secret_value(SecretId=self.secret_name)
        secret = json.loads(response['SecretString'])
        return secret['api_key']

class APIClient:
    def __init__(self, api_key, endpoint):
        self.api_key = api_key
        self.endpoint = endpoint
        self.headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }

    def fetch_data(self):
        response = requests.get(self.endpoint, headers=self.headers)
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Failed to fetch data. Status code: {response.status_code}, Response: {response.text}")

class RecordProcessor:
    @staticmethod
    def filter_and_sum_records(data):
        today = datetime.now().strftime("%Y-%m-%d")
        total_count = 0

        for record in data:
            timestamp = record.get("timestamp", "")
            status = record.get("status", "")
            record_count = record.get("record_count", 0)

            if today in timestamp and status == "successful":
                total_count += record_count

        return total_count

if __name__ == "__main__":
    SECRET_NAME = "YOUR_SECRET_NAME"
    REGION_NAME = "YOUR_REGION"
    API_ENDPOINT = "https://your-api-endpoint.com/data"

    try:
        # Retrieve API key
        secret_manager = SecretManager(SECRET_NAME, REGION_NAME)
        api_key = secret_manager.get_secret()

        # Fetch data from API
        api_client = APIClient(api_key, API_ENDPOINT)
        data = api_client.fetch_data()

        # Process records
        processor = RecordProcessor()
        total_count = processor.filter_and_sum_records(data)

        print(f"Total count of records with status 'successful' for today: {total_count}")

    except Exception as e:
        print(f"An error occurred: {e}")
