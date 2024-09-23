# Getting Started

## Introduction
The Orkid.io API allows you to programmatically interact with your organization data, at minimum it allows you to:  

- Invite users to your organization (memberships).
- Create projects within your organization.
- Create processing artifacts inside projects, i.e. document categories, classifiers, extractors.
- Upload documents (flow or step modes).
- Upload big document files by pre-signed URL.
- Detect text and layout info within documents (OCR).
- Create documents data extraction schemas.
- Classify documents.
- Extract information from documents.

For a complete reference of the API operations check the [API Reference](/api-reference)

## Quick start

### Install curl
``curl`` should already be installed by default in you OS if you use MacOS or any Linux distribution, if that's not the case you can grab it from [curl.haxx.se](https://curl.haxx.se/download.html).

### Use the API from Windows

### Create an account
Contact [support@orkid.io](mailto:support@orkid.io) to get a registered in MIDP. Once registration is confirmed, you will receive an email asking to reset your password, after which we'll send you a ````client id````. At this time you are all set to authenticate and start using the MIDP API.
!!! note
    Self-service account creation is a work in progress, we'll update the docs once it becomes available.

### Authenticate
```js
curl --location 'https://cognito-idp.eu-west-1.amazonaws.com/' \
--header 'Content-Type: application/x-amz-json-1.1' \
--header 'X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth' \
--data-raw '{
    "AuthParameters" : {
        "USERNAME" : "{{username}}",
        "PASSWORD" : "{{password}}"
    },
    "AuthFlow" : "USER_PASSWORD_AUTH",
    "ClientId" : "{{clientId}}"
}'
```
#### Auth Parameters  

````json
{
    "AuthParameters" : {
        "USERNAME" : "{{username}}", // email used to register in Orkid.io
        "PASSWORD" : "{{password}}" // password used to register in Orkid.io
    },
    "AuthFlow" : "USER_PASSWORD_AUTH", // authentication flow
    "ClientId" : "{{clientId}}" // clientId received after the registration in Orkid.io
}
````
#### Response
```json
{
    "AuthenticationResult": {
        "AccessToken": "(...)", // access token to send as a header in consecutive API calls
        "ExpiresIn": 3600, // by default token expires in 1h
        "IdToken": "(...)",
        "RefreshToken": "(...)", // refresh token
        "TokenType": "Bearer"
    },
    "ChallengeParameters": {}
}
```
#### Token use
All MIDP API endpoints require authentication prior using them. In order to access API endpoints, you must send the header ```'Authorization: Bearer <AccessToken>'```. 

### Base Url
Orkid.io API is exposed at [https://api.midp-dev.taiger.io](https://api.midp-dev.taiger.io), from now on ```baseUrl``` variable in ```curl``` commands.

### Create a project
A project is a container for documents, extraction schemas, classifiers and extractors configuration. The first step to use the MIDP API after getting registered in an Organisation is to create a new project. You can create a project through the API with:

````js
curl -X 'POST' \
  '{{baseUrl}}/projects' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "string"
}'
````

### Create a document category
Document categories reprsent the different types of documents a project can process, e.g. invoices, contracts, etc. The categories will be used by the project's classifiers and extractors to properly select the underlying LLM to process the incoming documents. You can create a category through the API with:
````js
curl -X 'POST' \
  '{{baseUrl}}/projects/{{project_id}}/categories' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "string",
  "description": "string"
}'
````


### Create an extractor
````js
curl -X 'POST' \
  '{{baseUrl}}/projects/{{project_id}}/extractors' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "string",
  "ocr_mode": "text"
}'
````

### Create a datapoint extraction schema
The schema of the data to be extracted will determine the output value parsing of each datapoint as well as the number of possible instances of each.

- **d_cardinality**: How many values do you expect to extract from this datapoint? Cardinality can be ````one```` or ````many````.
- **d_type**: Which type do you want the datapoint returns its value in? Types can be:  
    - *object*: JSON like object, e.g. 
    ```json
    {"name": "John", "age": 25} // Person object
    ```
    - *number*: A numeric value e.g. ```32``` or a string like ```"forty-four"``` will produce the numeric outputs: ````32```` and ````44```` respectively. Currency values applies as well.
    - *bool*: Binary statements, e.g. the statement: ```"the invoice is due"``` will return ```True``` if the condition is met, ```False``` otherwise. 
    - *text*: Any text value, e.g. ```"the name of the person"``` will return ```"John Doe"``` text value.

- **extractor_id**: The extractor you want to associate the new datapoint with.

````js
curl -X 'POST' \
  '{{baseUrl}}/projects/{{project_id}}/datapoints?d_cardinality=one&d_type=text&extractor_id={{extractor_id}}' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "ID number",
  "description": "Identification number",
  "required": true
}'
````

### Upload a document
Uploading a document requieres the definition of certain params:

- **collection**: Document collection you want to upload the document to, choose between (dev, valid, test).
- **classifier_id**: (Optional) The id of the classifier you want to use to classify the uploaded document.
- **channel**: The channel param indicates the workflow the document is going to follow after uploading.  
Step channel will upload the document, detect its OCR content and stop.  
The Flow channel, aditionally to OCR, will classify (if applicable) and extract the information from a the document if there's a live extractor configured for this project. 
- **category_id**: The id of the category the uploaded document belongs to. Optional if no classifier id is provided.

````js
curl -X 'POST' \
  '{{baseUrl}}/projects/{{project_id}}/documents/uploaded?collection=dev&channel=FLOW&category_id={{category_id}}' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}' \
  -H 'Content-Type: multipart/form-data' \
  -F 'file=@Document.pdf;type=application/pdf'
````

### Extract information from uploaded document
````js
curl -X 'PUT' \
  '{{baseUrl}}/projects/{{project_id}}/documents/{{document_id}}/extracted?extractor_id={{extractor_id}}' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}'
````

### Retrieve extracted data
````js
curl -X 'GET' \
  '{{baseUrl}}/projects/{{project_id}}/documents/{{document_id}}' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer <AccessToken>' \
  -H 'x-organization-id: {{organization_id}}'
````