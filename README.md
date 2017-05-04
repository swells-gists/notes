# The Operationalization Client Library `mldeploy` 

## Introduction

July 2017, we will be working to introduce a Client Library for model operationalization written in Python. Focus will be given to produce a light-weight progressive set of APIs that are crafted for flexibility, readability, and a low learning curve.

At its core, `mldeploy` contains an extensive and powerful **plugin** system that allows you to very easily break your "operationalization application" up into isolated pieces of business logic, and reusable utilities. These `plugins` are responsible for their respective implementation concerns regarding: authentication, deploying, re-deploying, and service for consumption. 

## Authentication 

Authentication is completely handled by the `plugin` and not by the operationalization framework. Since Python is duck typed we can exploit whatever is assigned to the `auth` property without introducing an new APIS/objects/class. It will be the plugin that will be in charge of defining the shape of what gets assigned to `auth` and what to do with it.

For example:

```python
from mldeploy import MLDeploy

ml = MLDeploy('env-url', auth=('username', 'password'))
ml = MLDeploy('env-url', auth=('authuri', 'tenant', 'resource', 'clientid')

# It does not matter, perhaps some other authentication strategy (Basic Auth)from requests.auth import HTTPBasicAuth
ml = MLDeploy('env-url', auth=HTTPBasicAuth('user', 'pass'))
```

## Plugin Registration

To register the `plugin` we first import the plugin and then call:

```python
from mldeploy import MLDeploy
from somemlplugin import SomeMLPlugin

# --- Plugin SomeMLPlugin operationalization implementation ---

ml = MLDeploy('env-url', auth=('username', 'password'))
ml.register(SomeMLPlugin)

# Alternative as an argument
ml = MLDeploy('env-url', auth=('username', 'password'), register=SomeMLPlugin)
```

## API Overview

> **Note**  I am not advocating the `revoml` name or a separate python package outside of `mldeploy` I am just using it as a distinction between `azureml` to better articulate the example.

## Full API Overview

```python
from mldeploy import MlDeploy, RevoML, AzureML
from sklearn import datasets
import numpy as np

# --- Or Plugin AzureML's operationalization implementation ---
ml = MlDeploy('url', auth=('username', 'password'))
ml.register(RevoML)


# --- Or Plugin AzureML's operationalization implementation ---
ml = MLDeploy('env-url', auth=('authuri', 'tenant', 'resource', 'clientid'))
ml.register(AzureML)

## -------------------------------------------------
## --- Prepare for publishing                    ---
## -------------------------------------------------

local_obj = 'Local Object'

def init:
   # Any objects defined here are evaluated first and are accessible in `code`
   init_obj = 'Initialized Object'

def add_one(x):
   print init_obj
   print local_obj
   return x + 1

# --- Publish service via fluent API ---
ml.service('add-one')
   .version('1.0.1')
   .code_fn(add_one, init)
   .inputs({ 'x', 'float' }),
   .outputs({ 'answer', 'float' }),
   .objects([ local_obj ])
   .packages([ 'pandas==0.18.0', 'sklearn', np ])
   .description('The Description of the `add-one` service, accepts _markdown_.')
   .deploy() 

# --- Or Publish service via `kwargs` dictionary (alternative equivalent) ---
kwargs = { 
    'version': '1.0.1', 
    'code_fn': [ add_one, init ]    
    'objects': [ local_obj ], 
    'inputs': { 'x', 'float' },
    'outputs': { 'answer', 'float' },
    'packages': [ 'pandas==0.18.0', 'sklearn', np ],
    'description': 'The Description of the `add-one` service, accepts _markdown_.' 
}

ml.deploy_service('add-one', **kwargs)

# --- Discover service by `name` or `name` and optional `version` ---
service = ml.get_service('add-one')

# consume/test the service (raw response)
res = service.add_one(5)

print res.output('answer')

# --- service management ---
services = ml.list_services('add-one') # all versions of service `add-one`
service = ml.get_service('add-one', version = '1.0.1') # get `v1.0.1` otherwise latest
ml.delete_service('add-one')

# --- service update and re-deploy ---
ml.service('add-one')
   .version('1.0.1')
   .description('Update the description field.')
   .redeploy()

# --- Or service update via `kwargs` dictionary (alternative equivalent) ---
kwargs = { 
    'version': '1.0.1', 
    'description': 'Update the description field.' 
}
ml.redeploy_service('add-one', **kwargs)
```

## Highlevel API to Swagger mapping (MRS)

How **optional** properties map to a service's dynamic swagger definition:

```python
ml.service('add-one')
  .version('1.0.1')
  .inputs({ 'x', 'float' }),
  .outputs({ 'answer', 'float' }),
  .description('The Description of the `add-one` service, accepts _markdown_.')
  .deploy()
```

### Produces a service:

- API: `/api/add-one/1.0.1`
- swagger.json: `/api/add-one/1.0.1/swagger.json`

### `name`

`swagger.info.title`

### `version`

`swagger.info.version`  

### `description`

`swagger.info.description`

### `code_fn` or `alias`

The function name or `alias` map to the swagger's `operationId`

### `inputs` and `outputs`

```json
"definitions":{
   "InputParameters":{
       "type":"object",
          "properties":{
              "x":{
                  "type":"float"
               }
            }
   },
   "OutputParameters":{
      "type":"object",
      "properties":{
           "answer":{
              "type":"float"
            }
       }
   }
}
```

## Full swagger.json upon publishing

> Note: things left out for brevity

```json
{
   "swagger":"2.0",
   "info":{
      "description":"The service description in text or `markdown`",
      "title":"add-one",
      "version":"1.0.0"
   },
   "schemes":[
      "http",
      "https"
   ],
   "securityDefinitions":{
      "Bearer":{
         "type":"apiKey",
         "name":"Authorization",
         "in":"header"
      }
   },
   "tags":[
      {
         "name":"add-one"         
      }
   ],
   "consumes":[
      "application/json"
   ],
   "produces":[
      "application/json"
   ],
   "/api/add-one/1.0.0":{
      "post":{
         "tags":[
            "add-one"
         ],
         "description":"Consume the add-one web service.",
         "operationId":"add_one",
         "parameters":[
            {
               "name":"WebServiceParameters",
               "in":"body",
               "required":true,
               "description":"Input parameters to the web service.",
               "schema":{
                  "$ref":"#/definitions/InputParameters"
               }
            }
         ],
         "responses":{
            "200":{
               "description":"OK",
               "schema":{
                  "$ref":"#/definitions/WebServiceResult"
               }
            }
         }
      },
      "definitions":{
         "InputParameters":{
            "type":"object",
            "properties":{
               "x":{
                  "type":"float"
               }
            }
         },
         "OutputParameters":{
            "type":"object",
            "properties":{
               "answer":{
                  "type":"float"
               }
            }
         }
      }
   }
```
