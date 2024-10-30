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
