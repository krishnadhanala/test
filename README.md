import redis
import boto3
from functools import wraps

# Initialize Redis and DynamoDB clients
redis_cache = redis.Redis(host='localhost', port=6379, db=0)
dynamodb = boto3.resource('dynamodb')
users_groups_table = dynamodb.Table('UserGroups')
groups_table = dynamodb.Table('Groups')

# Fetch user groups from DynamoDB or Redis cache
def get_user_groups(username):
    cached_groups = redis_cache.get(f"user_groups:{username}")
    if cached_groups:
        return eval(cached_groups)  # Convert cached string to list
    
    # If not cached, fetch from DynamoDB
    try:
        response = users_groups_table.get_item(Key={'username': username})
        if 'Item' in response:
            user_groups = response['Item']['groups']
            # Cache the user groups in Redis with a TTL of 1 hour (3600 seconds)
            redis_cache.setex(f"user_groups:{username}", 3600, str(user_groups))
            return user_groups
        else:
            return []
    except Exception as e:
        print(f"Error fetching user groups: {e}")
        return []

# Fetch group permissions from DynamoDB or Redis cache
def get_group_permissions(group):
    cached_permissions = redis_cache.get(f"group_permissions:{group}")
    if cached_permissions:
        return eval(cached_permissions)  # Convert cached string to dict
    
    # If not cached, fetch from DynamoDB
    try:
        response = groups_table.get_item(Key={'group_name': group})
        if 'Item' in response:
            group_permissions = response['Item']['permissions']
            # Cache the group permissions in Redis with a TTL of 1 hour
            redis_cache.setex(f"group_permissions:{group}", 3600, str(group_permissions))
            return group_permissions
        else:
            return {}
    except Exception as e:
        print(f"Error fetching group permissions: {e}")
        return {}

# Decorator to check if the user belongs to any group
def check_group_membership(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        username = kwargs.get('username')

        # Fetch the user's groups
        user_groups = get_user_groups(username)
        if not user_groups:
            return f"Access Denied: User {username} does not belong to any group."
        
        # Store the groups in kwargs to pass to the next decorator
        kwargs['user_groups'] = user_groups
        return func(*args, **kwargs)
    
    return wrapper

# Decorator to check if the user's groups have the required permission for the dataset
def check_group_permission(required_permission):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            username = kwargs.get('username')
            dataset_name = kwargs.get('dataset_name')
            user_groups = kwargs.get('user_groups')  # Passed from check_group_membership

            # Check if any of the user's groups have the required permission for this dataset
            for group in user_groups:
                permissions = get_group_permissions(group)
                if dataset_name in permissions and permissions[dataset_name] == required_permission:
                    return func(*args, **kwargs)

            return f"Access Denied: User {username} does not have {required_permission} access to {dataset_name}."
        return wrapper
    return decorator

# Example actions with group-based access control using two decorators

@check_group_membership
@check_group_permission('Write')
def create_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' created successfully by {username}."

@check_group_membership
@check_group_permission('Read')
def view_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' viewed by {username}."

@check_group_membership
@check_group_permission('Admin')
def delete_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' deleted by {username}."

# Main application simulation
if __name__ == "__main__":
    # Simulate actions by different users with group-based permissions
    print(create_dataset(username="user1", dataset_name="Project_Data"))
    print(view_dataset(username="user2", dataset_name="Project_Data"))
    print(delete_dataset(username="admin_user", dataset_name="Project_Data"))
    
    # Simulate reusing cached data
    print(create_dataset(username="user1", dataset_name="Project_Data"))  # Uses cached access and role
    print(view_dataset(username="user2", dataset_name="Project_Data"))  # Uses cached access and role


from functools import wraps

# Mock Data (simulating DynamoDB data)
MOCK_USER_GROUPS = {
    'user1': ['group1', 'group2'],
    'user2': ['group2'],
    'admin_user': ['admin_group'],
}

MOCK_GROUP_PERMISSIONS = {
    'group1': {
        'Project_Data': 'Write',
        'Public_Data': 'Read',
    },
    'group2': {
        'Project_Data': 'Read',
        'Public_Data': 'Read',
    },
    'admin_group': {
        'Project_Data': 'Admin',
        'Public_Data': 'Admin',
    },
}

# Fetch user groups (simulated function instead of DynamoDB)
def get_user_groups(username):
    return MOCK_USER_GROUPS.get(username, [])

# Fetch group permissions (simulated function instead of DynamoDB)
def get_group_permissions(group):
    return MOCK_GROUP_PERMISSIONS.get(group, {})

# Decorator to check if the user belongs to any group
def check_group_membership(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        username = kwargs.get('username')

        # Fetch the user's groups
        user_groups = get_user_groups(username)
        if not user_groups:
            return f"Access Denied: User {username} does not belong to any group."
        
        # Store the groups in kwargs to pass to the next decorator
        kwargs['user_groups'] = user_groups
        return func(*args, **kwargs)
    
    return wrapper

# Decorator to check if the user's groups have the required permission for the dataset
def check_group_permission(required_permission):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            username = kwargs.get('username')
            dataset_name = kwargs.get('dataset_name')
            user_groups = kwargs.get('user_groups')  # Passed from check_group_membership

            # Check if any of the user's groups have the required permission for this dataset
            for group in user_groups:
                permissions = get_group_permissions(group)
                if dataset_name in permissions and permissions[dataset_name] == required_permission:
                    return func(*args, **kwargs)

            return f"Access Denied: User {username} does not have {required_permission} access to {dataset_name}."
        return wrapper
    return decorator

# Example actions with group-based access control using two decorators

@check_group_membership
@check_group_permission('Write')
def create_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' created successfully by {username}."

@check_group_membership
@check_group_permission('Read')
def view_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' viewed by {username}."

@check_group_membership
@check_group_permission('Admin')
def delete_dataset(username, dataset_name):
    return f"Dataset '{dataset_name}' deleted by {username}."

# Simulating mock actions
def simulate_actions():
    print(create_dataset(username="user1", dataset_name="Project_Data"))  # Success
    print(view_dataset(username="user2", dataset_name="Project_Data"))  # Success
    print(delete_dataset(username="admin_user", dataset_name="Project_Data"))  # Success
    print(create_dataset(username="user2", dataset_name="Project_Data"))  # Access Denied
    print(delete_dataset(username="user1", dataset_name="Public_Data"))  # Access Denied

# Run the simulation
if __name__ == "__main__":
    simulate_actions()

