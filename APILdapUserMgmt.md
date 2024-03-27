### Create LDAP users and assign roles using UAA/CF API calls

Create an Admin client to do all CRUD calls
```
# Target TAS UAA
uaac target https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com --skip-ssl-validation
# Login with Admin Client Credentials
uaac token client get admin -s "uT6-ercAfivheelsc7I72tMlUNKfvT4t"
# Create New Admin Client
uaac client add superclient -s "Passsss!"  --authorized_grant_types client_credentials --scope cloud_controller.admin,clients.read,password.write,clients.secret,clients.write,uaa.admin,scim.write,scim.read --authorities cloud_controller.admin,clients.read,password.write,clients.secret,clients.write,uaa.admin,scim.write,scim.read
```
Create LDAP user via cf CLI
(to illustrate how it's done from that end and to see results)
```
cf api https://api.sys.tas01.tas-aws-lab.hyrulelab.com --skip-ssl-validation
cf login (use a real admin user, not client)
cf create-user john-ldap --origin ldap # needs no password                                                                           
cf set-org-role john-ldap dev OrgAuditor
```
Query UAA for user created via cf CLI
```
# Get UAA token with my new client via UAA CLI
uaac token client get superclient -s 'Passsss!'
# Get user via UAA CLI (the guid is included in the response)
uaac user get john-ldap
# (the user guid is included in the response)

# Get token via UAA API
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/oauth/token' -u 'superclient:Passsss!' -d grant_type=client_credentials
# Get user via UAA API (using token)
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/Users/508b7098-4395-4ca3-ac31-b5d761eed33c' -i -X GET \
    -H 'Authorization: Bearer aaa'
```
Query CF API  for user created via cf CLI - [/v3/users/[guid] GET](https://v3-apidocs.cloudfoundry.org/version/3.159.0/#get-a-user)
```
curl -k 'https://api.sys.tas01.tas-aws-lab.hyrulelab.com/v3/users/508b7098-4395-4ca3-ac31-b5d761eed33c' \
  -X GET \
  -H 'Authorization: Bearer aaa'
```
Create LDAP User via UAA API - [/Users POST](https://docs.cloudfoundry.org/api/uaa/version/77.3.0/index.html#create-2)
```
# Get token via UAA API
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/oauth/token' -u 'superclient:Passsss!' -d grant_type=client_credentials
# Create user via UAA API (with token)
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/Users' -i -X POST \
   -H 'Accept: application/json' \
   -H 'Authorization: Bearer aaa' \
   -H 'Content-Type: application/json' \
   -d '{
 "externalId" : "",
 "userName" : "jaime-ldap",
 "name" : {
   "formatted" : "given name family name",
   "familyName" : "LDAP",
   "givenName" : "Jaime"
 },
 "emails" : [ {
   "value" : "jaime-ldap@mydomain.com",
   "primary" : true
 } ],
 "active" : true,
 "verified" : true,
 "origin" : "ldap",
 "schemas" : [ "urn:scim:schemas:core:1.0" ]
}'
# (the user guid is included in the response)
# Get user via UAA API (using token)
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/Users/32423849-84af-485d-94bf-0df229d27dcf' -i -X GET \
    -H 'Authorization: Bearer aaa'
```
Create (map user) in CC via CF API - [/v3/users POST](https://v3-apidocs.cloudfoundry.org/version/3.159.0/#create-a-user)
- Confirmed this is not needed: we already have the user guid, and that's all we need for the permissions, and this record in the CC DB is created regardless automatically when we create the roles
- Confirmed also that this CC DB user has username: null both when created from API calls or cf create-user --origin ldap
```
# Get token via UAA API
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/oauth/token' -u 'superclient:Passsss!' -d grant_type=client_credentials
# Get user via CF API (using token)
curl -k 'https://api.sys.tas01.tas-aws-lab.hyrulelab.com/v3/users/0528e9ef-ef58-4427-8da3-c9fd0191c867' \
  -X GET \
  -H 'Authorization: Bearer aaa'
# response should look like this
{"errors":[{"detail":"User not found","title":"CF-ResourceNotFound","code":10010}]}

# You can't set the username or presentation_name with this endpoint (neither POST not PATCH). Those are update when the user logs in and the information is retrieved from the UAA DB
curl -k 'https://api.sys.tas01.tas-aws-lab.hyrulelab.com/v3/users' \
  -X POST \
  -H 'Authorization: bearer aaa' \
  -H 'Content-type: application/json' \
  -d '{
    "guid": "0528e9ef-ef58-4427-8da3-c9fd0191c867"
  }'
```
List Organizations - [/v3/organizations GET](https://v3-apidocs.cloudfoundry.org/version/3.159.0/#list-organizations)
```
# Get token via UAA API
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/oauth/token' -u 'superclient:Passsss!' -d grant_type=client_credentials
# Get org guid
curl -k 'https://api.sys.tas01.tas-aws-lab.hyrulelab.com/v3/organizations' \
  -X GET \
  -H 'Authorization: bearer aaa'
# (the guid of all orgs is included in the response)
```
Configure our LDAP user as `organization_manager` of `dev` org - [/v3/roles POST](https://v3-apidocs.cloudfoundry.org/version/3.159.0/#create-a-role)
```
# Get token via UAA API
curl -k 'https://uaa.sys.tas01.tas-aws-lab.hyrulelab.com/oauth/token' -u 'superclient:Passsss!' -d grant_type=client_credentials
# Configure relationship between user and org
curl -k 'https://api.sys.tas01.tas-aws-lab.hyrulelab.com/v3/roles' \
  -X POST \
  -H 'Authorization: bearer aaa' \
  -H 'Content-type: application/json' \
  -d '{
      "type": "organization_manager",
      "relationships": {
        "user": {
          "data": {
            "guid": "7fe79e3d-29b0-4ae8-9a62-00880ec4922e"
          }
        },
        "organization": {
          "data": {
            "guid": "af1a5229-cf9a-4064-9740-145c2d456a0e"
          }
        }
      }
    }'

```
After this, the user is displayed in AppsManager with the guid for some time.
- Confirmed this happens both when created from API calls or `cf create-user --origin ldap`

Eventually, the CC DB gets the username from the UAA DB and displays that instead of the guid
- When another role is added to a new user
- Probably when the user logs in

From this point on you can use all other CF API calls to get spaces, more orgs and assign more roles to the user.
