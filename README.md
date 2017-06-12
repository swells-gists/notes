# Python Client Library Package `mldeploy`


## Collaboration Goals

1. Agree upon a clean API contract for operationalization
2. Do not force an opinionated workflow
3. Provide flexibility that does not disrupt existing efforts (other than the agreed API)

> Only successful if both implementations are  successful


## Full API Overview (recap)

```python
from mldeploy import MLDeploy
from sklearn import datasets
import numpy as np

# --- Create authenticated session via LDAP/AD to the  MLServer ---
ml = MLDeploy('http://localhost:12800', auth=('username', 'password'), use='mldeploy.mls.MLServer')

# --- Create authenticated session via AAD-or-other-ctx to Azure  ---
ml = MLDeploy('http://localhost:8000', auth='AZURE_CUSTOM_AUTH_CTX', use='mldeploy.aml.Azure')

## --- Prepare for publishing ---
local_obj = 'Local Object'
model = 'Model'

def init:
   # Any objects defined here are evaluated first and are accessible in `code`
   init_obj = 'Initialized Object'

def add_one(x):
   print init_obj
   print local_obj
   print model
   return x + 1

# --- Publish service via fluent API ---
ml.service('add-one')
   .version('1.0.1')
   .code_fn(add_one, init)
   .inputs({ 'x': 'float' }),
   .outputs({ 'answer': 'float' }),
   .models([ model ]),
   .objects([ local_obj ])
   .packages([ 'pandas==0.18.0', 'sklearn', np ])
   .artifacts([ 'histogram.png' ])
   .description('The Description of the `add-one` service, accepts _markdown_.')
   .deploy() 

# --- Or Publish service via `kwargs` dictionary (alternative equivalent) ---
kwargs = { 
    'version': '1.0.1', 
    'code_fn': [ add_one, init ]    
    'objects': [ local_obj ],
    'models': [ model ],
    'inputs': { 'x': 'float' },
    'outputs': { 'answer': 'float' },
    'packages': [ 'pandas==0.18.0', 'sklearn', np ],
    'artifacts' : [ 'histogram.png' ],
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




## Requirments

1. Operationalization subclasses or modules need to only implement and extend the base `Operationalization` abstract class. The framework will manage the lifecycle.

```python
# -*- coding: utf-8; tab-width: 4; indent-tabs-mode: t; python-indent: 4 -*-

"""
Role
====

`Operationalization` is designed to be a low-level abstract foundation class 
from which other service operationalization attribute classes in the `mldeploy` 
package can be derived. It provides a standard template for creating
attribute-based operationalization lifecycle phases providing a consistent 
`__init()__`, `__del()__` sequence that chains initialization (initializer),
authentication (authentication), and destruction (destructor) methods for the 
class hierarchy.
"""

import six

# python 2 and python 3 compatibility library
from six import add_metaclass
from abc import ABCMeta, abstractmethod


@add_metaclass(ABCMeta)
class Operationalization(object):

    def initializer(self, api_client, config):
        """
        Init lifecycle method, invoked during construction. Sets up attributes 
        and invokes initializers for the class hierarchy.

        An optional _noonp_ method where subclass implementers MAY provide this 
        method definition by overriding.

        Object with configuration property name/value pairs
	"""        
        pass
        
    def destructor(self):
        """
        Destroy lifecycle method. Invokes destructors for the class hierarchy.

        An optional _noonp_ method where subclass implementers MAY provide this 
        method definition by overriding.
        """        
        pass
        
    def authentication(self, context):
        """
        Authentication lifecycle method. Invokes the authentication enterypoint
        for the class hierarchy.        

        An optional _noonp_ method where subclass implementers MAY provide this 
        method definition by overriding.
        """         
        pass
        
    @abstractmethod
    def get_service(self, name):
        """Retrieve service meta-data from the name source and return an new 
        service instance.
        """

    @abstractmethod
    def list_services(self, name=None, version=None):
        """
        return a list of service meta-data
        """

    @abstractmethod
    def delete_service(self, name):
        """
        return True|False
        """     

    @abstractmethod
    def deploy_service(self, name, **kwargs):
        """
        return a new service instance.        
        """

    @abstractmethod
    def redeploy_service(self, name, **kwargs):
        """
        return a new service instance.        
        """
        
    def service(self, name):
        """
        return a `OperationalizationDefinition` for fluent API.
        """        
        return OperationalizationDefinition(name, self)   


```

## Example (mlserver)


```python
# -*- coding: utf-8; tab-width: 4; indent-tabs-mode: t; python-indent: 4 -*-

"""

Overview
========

MLServer operationalization impl.

"""

import logging
from ..operationalization import Operationalization

log = logging.getLogger(__name__)

class MLServer(Operationalization):
        
    def initializer(self, api_client, config):
        """
        Override
        """
        log.debug('MLServer.initializer(): \n %s', config.host)
        self._api_client = api_client        

    def destructor(self):
        """
        Override
        """
        log.debug('MLServer.destructor()')        
        
    def authentication(self, context):
        """
        Override
        """
        log.debug('MLServer.authentication()')
           
    def get_service(self, name):
        log.debug('MLServer.get_service(): \n   %s', name)
        return name + "-MLServer"

    def list_services(self, name=None, version=None):
        log.debug('MLServer.list_services(): \n   %s\n   %s', name, version)
        pass
        
    def delete_service(self, name, **kwargs):
        log.debug('MLServer.deploy_service(): \n   %s\n   %s', name, kwargs)
        pass

    def deploy_service(self, name, **kwargs):
        log.debug('MLServer.deploy_service(): \n   %s\n   %s', name, kwargs)
        pass

    def redeploy_service(self, name, **kwargs):
        log.debug('MLServer.redeploy_service(): \n   %s\n   %s', name, kwargs)        
        pass


```

