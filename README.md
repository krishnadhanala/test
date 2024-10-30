1. Introduction
This documentation describes a group-based access control system that integrates:

Okta for user authentication and session handling.
Redis for caching group memberships and group permissions.
DynamoDB as the source of truth for persistent user-to-group mappings and group permissions.
Each user can belong to multiple groups, with permissions defined at the group level. The system aggregates permissions from all groups a user belongs to and constructs policies accordingly.

2. System Workflow
User logs in via Okta and provides an authorization token.
Lambda function validates the token and extracts user information.
Retrieve user groups from Redis (if cached) or DynamoDB (if not cached) and store in Redis for future use.
Fetch permissions for each group the user belongs to, aggregate them, and generate a policy based on the combined permissions.
3. Key Components
3.1 Redis for Caching Group Memberships and Permissions
Redis is used to cache both user group memberships and group permissions to avoid repeated lookups in DynamoDB, thus improving performance.

3.2 DynamoDB for Persistent Group-Based Permissions
DynamoDB stores:

User-to-group mappings: Defines which groups each user belongs to.
Group permissions: Defines which actions each group can perform on different resources.
3.3 Okta for Authentication
Okta handles user authentication and provides JWT tokens that include user information.

4. Detailed Implementation
4.1 Redis Utility for Caching
This function retrieves the user’s groups from Redis (or DynamoDB if not cached) and then caches them for future requests.

python
Copy code
import redis
import boto3
import json
import os
from okta.jwt import validate_token

# Initialize Redis and DynamoDB clients
redis_client = redis.Redis(host=os.environ['REDIS_HOST'], port=os.environ['REDIS_PORT'])
dynamodb = boto3.resource('dynamodb')
permissions_table = dynamodb.Table('Permissions')

def get_user_groups_from_cache_or_db(username):
    """
    Checks Redis cache for the user's group memberships.
    If not present, fetches from DynamoDB and caches it.
    """
    cached_groups = redis_client.get(f"user_groups:{username}")
    
    if cached_groups:
        return json.loads(cached_groups)
    else:
        # Fetch user groups from DynamoDB if not cached
        response = permissions_table.get_item(Key={'group_or_user_id': username, 'type': 'user'})
        if 'Item' in response:
            groups = response['Item'].get('groups', [])
            # Cache in Redis
            redis_client.set(f"user_groups:{username}", json.dumps(groups), ex=3600)  # Cache for 1 hour
            return groups
        else:
            return []
4.2 Fetch Group Permissions
This function retrieves and caches the permissions for each group a user belongs to. It aggregates permissions for all the groups and returns a combined permissions dictionary.

python
Copy code
def get_group_permissions_from_cache_or_db(group_name):
    """
    Checks Redis cache for the group's permissions.
    If not present, fetches from DynamoDB and caches it.
    """
    cached_permissions = redis_client.get(f"group_permissions:{group_name}")
    
    if cached_permissions:
        return json.loads(cached_permissions)
    else:
        # Fetch group permissions from DynamoDB if not cached
        response = permissions_table.get_item(Key={'group_or_user_id': group_name, 'type': 'group'})
        if 'Item' in response:
            permissions = response['Item'].get('permissions', {})
            # Cache in Redis
            redis_client.set(f"group_permissions:{group_name}", json.dumps(permissions), ex=3600)  # Cache for 1 hour
            return permissions
        else:
            return {}

def aggregate_permissions_for_user(groups):
    """
    Aggregates permissions from multiple groups.
    """
    aggregated_permissions = {}
    for group in groups:
        group_permissions = get_group_permissions_from_cache_or_db(group)
        for resource, actions in group_permissions.items():
            if resource not in aggregated_permissions:
                aggregated_permissions[resource] = set(actions)
            else:
                aggregated_permissions[resource].update(actions)
    # Convert sets back to lists for JSON serialization
    return {resource: list(actions) for resource, actions in aggregated_permissions.items()}
4.3 Token Validation with Okta
The validate_authorization_token function authenticates the user via Okta, validating the JWT token.

python
Copy code
def validate_authorization_token(auth_header):
    try:
        parts = auth_header.split()
        if len(parts) != 2 or parts[0] != 'Bearer':
            raise Exception("Invalid authorization token")
        
        issuer = os.environ['OKTA_ORG_URL'] + "/oauth2/default"
        audience = os.environ['OKTA_AUD']
        client_id = os.environ['OKTA_CLIENT_ID']
        
        access_token = parts[1]
        jwt = validate_token(access_token, issuer, audience, client_id)
        return jwt
    except Exception as ex:
        print(f"Exception: {ex}")
        print("Could not validate authorization token")
        return None
4.4 lambda_handler for Policy Generation Based on Aggregated Group Permissions
The lambda_handler function fetches user groups, aggregates permissions, and generates a policy based on the user’s allowed actions across resources.

python
Copy code
def lambda_handler(event, context):
    try:
        # Validate the Okta JWT token
        jwt = validate_authorization_token(event['authorizationToken'])
        if not jwt:
            raise Exception('Unauthorized')
        
        username = jwt['sub']
        aws_account_id = event['methodArn'].split(":")[4]
        api_gateway_arn_tmp = event['methodArn'].split(":")[5].split("/")
        
        # Initialize AuthPolicy
        principal_id = f"user|{username}"
        policy = AuthPolicy(principal_id, aws_account_id)
        policy.restApiId = api_gateway_arn_tmp[0]
        policy.region = event['methodArn'].split(":")[3]
        policy.stage = api_gateway_arn_tmp[1]
        
        # Fetch user groups from Redis or DynamoDB
        user_groups = get_user_groups_from_cache_or_db(username)
        
        # Aggregate permissions for all user groups
        aggregated_permissions = aggregate_permissions_for_user(user_groups)
        
        # Allow or deny methods based on aggregated permissions
        for resource, actions in aggregated_permissions.items():
            if 'read' in actions:
                policy.allowMethod(HttpVerb.GET, resource)
            if 'write' in actions:
                policy.allowMethod(HttpVerb.POST, resource)
                policy.allowMethod(HttpVerb.PUT, resource)
            if 'delete' in actions:
                policy.allowMethod(HttpVerb.DELETE, resource)
        
        # Build and return the authorization response
        auth_response = policy.build()
        context['userGroups'] = user_groups  # Save groups for auditing/logging
        
        return auth_response
    except Exception as ex:
        print(f"Exception happened: {ex}")
        raise Exception('Unauthorized')
5. Policy Management with AuthPolicy
The AuthPolicy class builds policies based on allowed and denied methods. Each resource can have specific HTTP verbs enabled based on the group permissions.

6. Example Workflow
User logs in via Okta:

Okta issues a JWT token that the backend validates.
Retrieve user groups:

The function checks Redis for user groups. If not found, it fetches the groups from DynamoDB and caches the result.
Aggregate permissions:

For each group, permissions are retrieved from Redis (or DynamoDB if not cached). The permissions are aggregated to determine allowed actions for the user.
Policy generation:

Based on the aggregated permissions, a policy is generated using AuthPolicy, allowing or denying specific HTTP verbs for each resource.
7. Final Notes on Caching and Policy Management
Caching Strategy:

Redis caches both user group memberships and group permissions to minimize calls to DynamoDB. Cached items are set to expire after 1 hour to ensure data freshness.
Dynamic Aggregation of Permissions:

Permissions from multiple groups are aggregated to provide the user with comprehensive access across all resources their groups have permissions for.
Role-Based UI Control:

The user’s permissions are dynamically constructed based on group memberships, allowing for more granular access control in the UI.
This setup effectively combines Okta for authentication, Redis for caching, and DynamoDB for role and permission storage to create a scalable, dynamic group-based access control system.






You said:
can you combine code 
ChatGPT said:
ChatGPT
Here's the combined code integrating Redis, DynamoDB, and Okta for a dynamic, group-based access control system. The code includes functions to authenticate via Okta, retrieve and cache group memberships and permissions from Redis (with DynamoDB as the primary data store), aggregate permissions across groups, and generate policies based on those permissions.

Combined Code Implementation
python
Copy code
from __future__ import print_function
import redis
import boto3
import json
import os
import re
from okta.jwt import validate_token

# Initialize Redis and DynamoDB clients
redis_client = redis.Redis(host=os.environ['REDIS_HOST'], port=os.environ['REDIS_PORT'])
dynamodb = boto3.resource('dynamodb')
permissions_table = dynamodb.Table('Permissions')

def validate_authorization_token(auth_header):
    """
    Validates the Okta JWT token.
    """
    try:
        parts = auth_header.split()
        if len(parts) != 2 or parts[0] != 'Bearer':
            raise Exception("Invalid authorization token")
        
        issuer = os.environ['OKTA_ORG_URL'] + "/oauth2/default"
        audience = os.environ['OKTA_AUD']
        client_id = os.environ['OKTA_CLIENT_ID']
        
        access_token = parts[1]
        jwt = validate_token(access_token, issuer, audience, client_id)
        return jwt
    except Exception as ex:
        print(f"Exception: {ex}")
        print("Could not validate authorization token")
        return None

def get_user_groups_from_cache_or_db(username):
    """
    Checks Redis cache for the user's group memberships.
    If not present, fetches from DynamoDB and caches it.
    """
    cached_groups = redis_client.get(f"user_groups:{username}")
    
    if cached_groups:
        return json.loads(cached_groups)
    else:
        # Fetch user groups from DynamoDB if not cached
        response = permissions_table.get_item(Key={'group_or_user_id': username, 'type': 'user'})
        if 'Item' in response:
            groups = response['Item'].get('groups', [])
            # Cache in Redis
            redis_client.set(f"user_groups:{username}", json.dumps(groups), ex=3600)  # Cache for 1 hour
            return groups
        else:
            return []

def get_group_permissions_from_cache_or_db(group_name):
    """
    Checks Redis cache for the group's permissions.
    If not present, fetches from DynamoDB and caches it.
    """
    cached_permissions = redis_client.get(f"group_permissions:{group_name}")
    
    if cached_permissions:
        return json.loads(cached_permissions)
    else:
        # Fetch group permissions from DynamoDB if not cached
        response = permissions_table.get_item(Key={'group_or_user_id': group_name, 'type': 'group'})
        if 'Item' in response:
            permissions = response['Item'].get('permissions', {})
            # Cache in Redis
            redis_client.set(f"group_permissions:{group_name}", json.dumps(permissions), ex=3600)  # Cache for 1 hour
            return permissions
        else:
            return {}

def aggregate_permissions_for_user(groups):
    """
    Aggregates permissions from multiple groups.
    """
    aggregated_permissions = {}
    for group in groups:
        group_permissions = get_group_permissions_from_cache_or_db(group)
        for resource, actions in group_permissions.items():
            if resource not in aggregated_permissions:
                aggregated_permissions[resource] = set(actions)
            else:
                aggregated_permissions[resource].update(actions)
    # Convert sets back to lists for JSON serialization
    return {resource: list(actions) for resource, actions in aggregated_permissions.items()}

class AuthPolicy:
    pathRegex = r"^[/.a-zA-Z0-9-\*]+$"
    
    def __init__(self, principal_id, aws_account_id):
        self.awsAccountId = aws_account_id
        self.principalId = principal_id
        self.version = "2012-10-17"
        self.allowMethods = []
        self.denyMethods = []
        self.restApiId = "*"
        self.region = "*"
        self.stage = "*"

    def _addMethod(self, effect, verb, resource, conditions=None):
        if verb != "*" and not hasattr(HttpVerb, verb):
            raise NameError(f"Invalid HTTP verb {verb}. Allowed verbs in HttpVerb class")
        
        resourcePattern = re.compile(self.pathRegex)
        if not resourcePattern.match(resource):
            raise NameError(f"Invalid resource path: {resource}. Path should match {self.pathRegex}")
        
        resourceArn = f"arn:aws:execute-api:{self.region}:{self.awsAccountId}:{self.restApiId}/{self.stage}/{verb}/{resource}"
        method = {'resourceArn': resourceArn, 'conditions': conditions or {}}
        if effect.lower() == "allow":
            self.allowMethods.append(method)
        elif effect.lower() == "deny":
            self.denyMethods.append(method)

    def getStatementForEffect(self, effect, methods):
        statements = []
        if methods:
            statement = {
                "Action": "execute-api:Invoke",
                "Effect": effect.capitalize(),
                "Resource": [method['resourceArn'] for method in methods]
            }
            statements.append(statement)
        return statements

    def allowMethod(self, verb, resource):
        self._addMethod("Allow", verb, resource)

    def denyMethod(self, verb, resource):
        self._addMethod("Deny", verb, resource)

    def build(self):
        if not (self.allowMethods or self.denyMethods):
            raise NameError("No statements defined for the policy")
        
        policy = {
            'principalId': self.principalId,
            'policyDocument': {
                'Version': self.version,
                'Statement': []
            }
        }
        
        policy['policyDocument']['Statement'].extend(
            self.getStatementForEffect("Allow", self.allowMethods)
        )
        policy['policyDocument']['Statement'].extend(
            self.getStatementForEffect("Deny", self.denyMethods)
        )
        
        return policy

class HttpVerb:
    GET = "GET"
    POST = "POST"
    PUT = "PUT"
    PATCH = "PATCH"
    DELETE = "DELETE"
    OPTIONS = "OPTIONS"
    ALL = "*"

def lambda_handler(event, context):
    try:
        # Validate the Okta JWT token
        jwt = validate_authorization_token(event['authorizationToken'])
        if not jwt:
            raise Exception('Unauthorized')
        
        username = jwt['sub']
        aws_account_id = event['methodArn'].split(":")[4]
        api_gateway_arn_tmp = event['methodArn'].split(":")[5].split("/")
        
        # Initialize AuthPolicy
        principal_id = f"user|{username}"
        policy = AuthPolicy(principal_id, aws_account_id)
        policy.restApiId = api_gateway_arn_tmp[0]
        policy.region = event['methodArn'].split(":")[3]
        policy.stage = api_gateway_arn_tmp[1]
        
        # Fetch user groups from Redis or DynamoDB
        user_groups = get_user_groups_from_cache_or_db(username)
        
        # Aggregate permissions for all user groups
        aggregated_permissions = aggregate_permissions_for_user(user_groups)
        
        # Allow or deny methods based on aggregated permissions
        for resource, actions in aggregated_permissions.items():
            if 'read' in actions:
                policy.allowMethod(HttpVerb.GET, resource)
            if 'write' in actions:
                policy.allowMethod(HttpVerb.POST, resource)
                policy.allowMethod(HttpVerb.PUT, resource)
            if 'delete' in actions:
                policy.allowMethod(HttpVerb.DELETE, resource)
        
        # Build and return the authorization response
        auth_response = policy.build()
        context['userGroups'] = user_groups  # Save groups for auditing/logging
        
        return auth_response
    except Exception as ex:
        print(f"Exception happened: {ex}")
        raise Exception('Unauthorized')
Explanation of Key Components
validate_authorization_token: Validates the JWT token issued by Okta to authenticate the user.

get_user_groups_from_cache_or_db: Retrieves user group memberships from Redis. If not cached, fetches from DynamoDB, then caches the result.

get_group_permissions_from_cache_or_db: Retrieves group permissions from Redis or DynamoDB and caches the result.

aggregate_permissions_for_user: Aggregates permissions from all groups the user belongs to, consolidating actions like read, write, or delete for different resources.

AuthPolicy: Builds the authorization policy based on the aggregated permissions, defining allowed or denied HTTP methods per resource.

lambda_handler:

Validates the JWT token and retrieves user information.
Retrieves group memberships and permissions, aggregates them, and generates a policy based on the combined permissions.
Returns the generated policy as the authorization response.
