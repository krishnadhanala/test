import boto3
import time
from datetime import datetime, timedelta

class CloudWatchLogManager:
    def __init__(self, prefix, days_old):
        self.client = boto3.client('logs')
        self.prefix = prefix
        self.days_old = days_old
    
    def get_log_groups_with_prefix(self):
        log_groups = []
        paginator = self.client.get_paginator('describe_log_groups')
        for page in paginator.paginate(logGroupNamePrefix=self.prefix):
            for log_group in page.get('logGroups', []):
                log_groups.append(log_group['logGroupName'])
        return log_groups
    
    def delete_old_log_groups(self):
        log_groups = self.get_log_groups_with_prefix()
        if not log_groups:
            return {"message": "No log groups found with the specified prefix."}
        
        cutoff_time = int((datetime.utcnow() - timedelta(days=self.days_old)).timestamp() * 1000)
        deleted_logs = []
        
        for log_group in log_groups:
            response = self.client.describe_log_streams(logGroupName=log_group, orderBy='LastEventTime', descending=True, limit=1)
            if 'logStreams' in response and response['logStreams']:
                last_event_time = response['logStreams'][0].get('lastEventTimestamp', 0)
                if last_event_time < cutoff_time:
                    self.client.delete_log_group(logGroupName=log_group)
                    deleted_logs.append(log_group)
        
        if deleted_logs:
            return {"message": "Deleted log groups:", "log_groups": deleted_logs}
        return {"message": "No log groups older than specified days found."}
