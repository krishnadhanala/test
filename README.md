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
        
        # Pass the user groups to the next step (in the wrapper, not the function)
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
