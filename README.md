1. Introduction
This documentation describes a group-based access control system where system users (administrators) are responsible for assigning users to groups and configuring permissions for groups. The system ensures that:
1.	Users belong to specific groups, and their access to resources is controlled through group membership.
2.	Groups have specific permissions to perform actions (e.g., read, write, delete) on components (e.g., datasets, projects, pipelines, views).
To validate user access, two separate decorators are used:
•	The first decorator checks whether the user belongs to a valid group.
•	The second decorator checks whether any of the user’s groups have the necessary permissions to perform the action on the component.
This design promotes modularity and separation of concerns.
 
2. System Overview
In this system:
1.	System users manage groups and assign permissions to each group for various components.
2.	System users assign users to groups.
3.	When a user attempts to perform an action, the system validates:
o	User group membership: Ensures the user belongs to a valid group.
o	Group permission validation: Checks if any of the user’s groups have the necessary permission for the action on the component.
The system leverages caching to optimize performance by storing user group memberships and group permissions in-memory to minimize database queries.
 
3. Key Components
3.1 DynamoDB Table
A single Permissions Table is used to store both group permissions and user group memberships.
Field Name	Type	Description
group_or_user_id	String	Primary key that uniquely identifies groups or users.
type	String	Specifies whether the entry is for a group or user (group or user).
permissions	Map	For group entries, stores a map of components to their allowed actions.
groups	List	For user entries, stores a list of groups the user belongs to.
Example DynamoDB Table:
group_or_user_id	type	permissions	groups
group_1	group	{ "datasets": ["read", "write"], "projects": ["read"] }	
group_2	group	{ "projects": ["read", "delete"], "pipelines": ["read"] }	
user_1	user		["group_1", "group_2"]
user_2	user		["group_1"]
 
4. Workflow and Process
4.1 Workflow Diagram
1.	System user configures group permissions for components (e.g., granting read and write permissions to group_1on datasets).
2.	System user assigns users to groups (e.g., adding user_1 to group_1 and group_2).
3.	When a user attempts to perform an action (e.g., write on a dataset):
o	The system first checks group membership to ensure the user belongs to a valid group.
o	Then, the system checks group permissions to verify whether any of the user’s groups have the required permission for that action on the component.
4.	Caching is used to optimize the lookup of group permissions and user group memberships, minimizing database queries.
 
5. Two-Decorator Approach
This system uses two separate decorators:
1.	check_group_membership: Validates that the user belongs to one or more groups.
2.	check_group_permission: Verifies that the user’s group has permission to perform the requested action.
This approach separates concerns between checking group membership and verifying group permissions, resulting in a cleaner and more modular design.
5.1 check_group_membership Decorator
The check_group_membership decorator checks whether a user belongs to any group. It fetches the user’s group memberships from the cache or DynamoDB, and if the user is not part of any group, access is denied immediately.
python
Copy code
def check_group_membership(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        username = kwargs.get('username')

        # Fetch user's groups from cache or DynamoDB
        user_groups = get_user_groups(username)
        if not user_groups:
            return f"Access Denied: User {username} does not belong to any group."

        # Pass the user groups to the next decorator
        kwargs['user_groups'] = user_groups
        return func(*args, **kwargs)
    return wrapper
5.2 check_group_permission Decorator
The check_group_permission decorator checks if any of the user's groups have the necessary permissions to perform a specific action on a component. It uses cached permissions or fetches the data from DynamoDB.
python
Copy code
def check_group_permission(action):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user_groups = kwargs.get('user_groups')
            component_name = kwargs.get('component_name')

            # Check if any group has the required permission
            for group in user_groups:
                if check_group_permission_for_action(group, component_name, action):
                    return func(*args, **kwargs)

            return f"Access Denied: No group has {action} permission on {component_name}."
        return wrapper
    return decorator
5.3 Helper Functions
Here are the helper functions that fetch user groups and check group permissions. These functions interact with DynamoDB and cache results for subsequent requests.
python
Copy code
def get_user_groups(username):
    """
    Fetches and caches the groups that a user belongs to.
    """
    if username in session_cache:
        return session_cache[username]['groups']

    try:
        response = permissions_table.get_item(Key={'group_or_user_id': username})
        if 'Item' in response:
            user_groups = response['Item']['groups']
            session_cache[username] = {
                'groups': user_groups,
                'group_permissions': {},  # Cache for group permissions
                'timestamp': time.time()
            }
            return user_groups
        else:
            raise Exception(f"User {username} not found.")
    except Exception as e:
        print(f"Error fetching user groups: {e}")
        return None

def check_group_permission_for_action(group_name, component_name, action):
    """
    Checks if the group has permission to perform a certain action on a component.
    Uses cache if available, otherwise queries DynamoDB.
    """
    permissions = get_group_permissions(group_name)

    if component_name in permissions:
        allowed_actions = permissions[component_name]
        return action in allowed_actions
    else:
        return False

def get_group_permissions(group_name):
    """
    Fetches and caches the permissions of a group for various components.
    """
    if group_name in session_cache and 'group_permissions' in session_cache[group_name]:
        return session_cache[group_name]['group_permissions']

    try:
        response = permissions_table.get_item(Key={'group_or_user_id': group_name})
        if 'Item' in response:
            permissions = response['Item']['permissions']
            session_cache[group_name] = {
                'group_permissions': permissions,
                'timestamp': time.time()
            }
            return permissions
        else:
            raise Exception(f"Group {group_name} not found.")
    except Exception as e:
        print(f"Error fetching group permissions: {e}")
        return None
 
6. Example Usage
By using two separate decorators, you can modularly handle both group membership and permission checks. Below are examples of how these decorators can be applied for different actions:
6.1 Creating a Component
The create_component function requires a write permission for the component. The first decorator checks if the user belongs to a valid group, and the second verifies if the group has the necessary permission.

@check_group_membership
@check_group_permission('write')
def create_component(username, component_name):
    return f"Component '{component_name}' created successfully by {username}."
6.2 Viewing a Component
The view_component function requires a read permission for the component. The same decorators are applied to ensure group membership and permissions are checked.

@check_group_membership
@check_group_permission('read')
def view_component(username, component_name):
    return f"Component '{component_name}' viewed by {username}."
6.3 Deleting a Component
The delete_component function requires a delete permission for the component. Both decorators ensure group membership and the required permission.

@check_group_membership
@check_group_permission('delete')
def delete_component(username, component_name):
    return f"Component '{component_name}' deleted by {username}."
 
7. Caching and Performance
To ensure efficient performance, both user group memberships and group permissions are cached. Cached entries have an expiration time to ensure they remain fresh and reflect any recent changes made by system users.

def check_cache_expiry(username):
    expiry_time = 900  # 15 minutes
    current_time = time.time()
    if username in session_cache and 'timestamp' in session_cache[username]:

