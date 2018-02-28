MIIMETIQ Lite
=============

MIIMETIQ lite is a software build upon MIIMETIQ Framework such as a vertical project that implements a multi tenant solution with
some fixed constraints to set up projects fater than using MIIMETIQ by it self. Each tenat belongs one model and one user to handle
the instances of this model.

The tenants are implemented as assets and they have the `__billable__` keyword enabled, therefore the number of maxium tenants
available for one installation can be restricted by the license setting the `max billable assets` concept.

The main features of the MIIMETIQ Lite are:

    * Multi tenant
    * Assets CRUD, schema free
    * Dasboards
    * Automations
    * Alarms
    * Reports
    * Connection Layer

Deployment
==========


Setup
-----

The project root directory is expected at `/opt/miimetiq-lite` and the content of this git project at `/opt/miimetiq-lite/src`.
For development machines that run with vagrant, the next line suffice to synchronize your local git clone into the virtual machine at the correct path:

    config.vm.synced_folder "<path_to_the_local_git_clone>", "/opt/miimetiq-lite/src"

MIIMETIQ Lite needs also an environ where the the third party libraries will be installed isolated to the system libraries, to get that
we will create the environ using the next command

    $ virtualenv /opt/miimetiq-lite/env

Once you have configured the VM, you can configure miimetiq-lite with the file located at `/opt/miimetiq-lite/src/settings.conf`. There there are a set of variable to check:

* *REST_SERVER* This defines the URL to acces the miimetiq api.
* *API_USER* and *API_PASSWORD* sets the user and password to access the rest api as admin
* *SCHEMA_TENANT_DIRECTORY* This is the directory where the tenant schemas are stored. Make sure it exists.
* *BASE_TENANT_FILE* This is a very basic template to be used in the construction of the tenants schemas
* *DEVICE_TYPE* Refers to the name of the model used to store the information of the tenants
* *SEED* The information of the tenants include the encrypted password of the users. They are set on its instances applying a sha1 mixed with this variable


FlexibleDS
-----------

See the README.md in `services/flexible_ds`.


Dependencies
------------

Activate virtualenv and install Python dependencies:

    $ source /opt/miimetiq-lite/env/bin/activate
    $ cd /opt/miimetiq-lite/src
    $ pip install -r requirements.txt

MIIMETIQ configuration
----------------------

Create the folder where the schemas will be saved

    mkdir /opt/miimetiq-lite/tenant-schemas
Specific configuration for MIIMETIQ:

    SCHEMA_SOURCES = json:/opt/miimetiq-lite/src/data/schemas, json:/opt/miimetiq-lite/tenant-schemas
    MONGO_DBNAME = miimetiq-lite

    CUSTOM_RPCS_AUTHENTICATED = tenants/create, tenants/update, tenants/delete, schemas/read, schemas/create, schemas/update, schemas/delete, duplicate_asset, node_manager, node_manager/stop, node_manager/start, node_manager/status, node_manager/update, node_manager/create

    IDM = miimetiq
    GENERATE_LONLAT_INDEXES = False

    PROFILE_KEYS_INTEGRATOR = "units_system",

    CELERY_RPC_TIMEOUT = 120


Supervisor
----------

Create a dedicated supervisor configuration's directory for this project

    mkdir /etc/supervisor/conf.d/mqlite

Configure supervisor to be aware of this new configurations directory. Append this line at the end of the ``/etc/supervisor/supervisord.conf`` file:

   files = /etc/supervisor/conf.d/\*.conf /etc/supervisor/conf.d/miimetiq/\*.conf /etc/supervisor/conf.d/mqlite/\*.conf

Copy the supervisor configurations available in the repo to this directory

    cp /opt/miimetiq-lite/src/data/supervisors/* /etc/supervisor/conf.d/mqlite

Reload supervisor:

    supervisorctl reread
    supervisorctl update

Graphite
--------

How to setup Graphite according to the needs of the project (if needed)

Nginx
-----

How to setup Nginx to serve the frontend application and other HTTP services. Example:

    cp /opt/miimetiq-lite/src/data/nginx/* /etc/nginx/conf.d/


Usage
=====

After all the setup steps, you can start the processes and reload depending ones by executing:

    supervisorctl restart uwsgi workers: mqlt: nginx

Describe here any procedure needed in order to start, use, and maintain the project

RPCs
----

### Tenants

* Create

    The create rpc is used with the next data:

    * URI:

            http://api.miimetiq.local/customrpcs/tenants/create

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "name": "tenant_name/->@.3",
                "username": "tenant_name",
                "max_devices": "10",
                "description": "tenant name description",
                "max_signals": "100",
                "contact_email": "contact_name@contact_name.com",
                "active": true,
                "contact_name": "contact_name",
                "password": "12345678"
            }

    As you can notice, the *name* field accept any kind of character, it will be slugified though.

    The expected output is this one:

        #!javascript
        {
            "_status": "OK"
        }

    In case of error it will be show with an output similar to this:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "get_existing_tenenat_instance": "There is not a tenant instance with the name tenant_name/->@.3"
            }
        }

* Update

    Update an already existing instance, in case of different username, the previous username/group will be deleted and the tenant instance will be updated.


    * URI:

            http://api.miimetiq.local/customrpcs/tenants/update

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "name": "tenant_name/->@.3",
                "username": "tenant_name",
                "max_devices": "10",
                "description": "tenant name description",
                "max_signals": "100",
                "contact_email": "contact_name@contact_name.com",
                "active": true,
                "contact_name": "contact_name",
                "password": "12345678"
            }

    NOTICE that the only mandatory field is `name` which can be the raw user name or the slugified name.

    In case of error it will be show with an output similar to this:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "get_existing_tenenat_instance": "There is not a tenant instance with the name tenant_name/->@.3"
            }
        }


* Delete

    This fully functional RPC works as follows:

    * URI:

            http://api.miimetiq.local/customrpcs/tenants/delete

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "name": "tenant_name/->@.3"
            }

    NOTICE that the only mandatory field is `name` which can be the raw user name or the slugified name.

    The expected output is this one:

        #!javascript
        {
            "_status": "OK"
        }

    In case of error it will be shoow with an output similar to this. Just be aware that this operation can provide more than one error message:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "schema_delete_tenant_file": "No such file or directory",
                "idm_delete_group": "Object not found",
                "idm_delete_user": "Object not found"
            }
        }


    As you can see more than one operation is tried, therefore multiple errors can appear.


This set of RPCs are aimed to CRUD a signal/attribute into a tenant schema. To run these RPCs you need to create a tenant with the use of the previous RPCs.

* Add attribute

    This RPC adds a signal/attribute to the *schema identified by the current user logged*.

    * URI:

            http://api.miimetiq.local/customrpcs/schemas/create

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "attribute_name": "foo.bar",
                "attribute_type": "string"
            }

     The expected output is this one:

        #!javascript
        {
            "_status": "OK"
        }

    In case of error an output similar to this will be shown:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "exception": "The attribute path foo.bar is not properly defined"
            }
        }


* Update attribute

    This RPC updates a signal/attribute to the *schema identified by the current user logged*.

    * URI:

            http://api.miimetiq.local/customrpcs/schemas/update

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "new_attribute_name": "foo.bar",
                "old_attribute_name": "string"
            }

     NOTICE that only the name is allowed to be changed.
     The expected output is this one:

        #!javascript
        {
            "_status": "OK"
        }

    In case of error an output similar to this will be shown:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "exception": "The attribute path foo.bar is not properly defined"
            }
        }



* Delete attribute

    This RPC deletes a signal/attribute to the *schema identified by the current user logged*.

    * URI:

            http://api.miimetiq.local/customrpcs/schemas/delete

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "attribute_name": "foo.bar"
            }

     The expected output is this one:

        #!javascript
        {
            "_status": "OK"
        }

    In case of error an output similar to this will be shown:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "exception": "The attribute path foo.bar is not properly defined"
            }
        }


### Assets

* Duplicate asset

    * URI:

        http://api.miimetiq.local/customrpcs/duplicate_asset

    * Headers:

            Accept: application/json
            Content-Type: application/json

    * Body:

            #!javascript
            {
                "asset_type": "foo_tenant",
                "original_asset_id": "568b89b4e7e4665b09041b14",
                "duplicated_name": "foo_tenant",
                "duplication_number": 2
            }

    The parameter `duplicated_name` and `duplication_number` are optional. The default values are:

        * `duplicated_name`: `orig-name_copy_X` where X is a number
        * `duplication_number`: 1

    The expected output is this one:

        #!javascript
        {
          "_status": "OK",
          "_items": [
            {
              "_updated": "2016-01-14T11:56:26Z",
              "_id": "56978cea6bf9e779abef2d2d",
              "name": "foo_tenant_copy_0",
              "_created": "2016-01-14T11:56:26Z",
              "_group": "test_mim_01",
              "_user": "test_mim_01",
              "_etag": "8070bc685d51eb16ab9111faf8f4bb42fd3f3a0b",
              "connection_instrument": {
                "connected_ts": {
                  "last_value_update": "2016-01-14T11:56:26Z",
                  "value": null
                },
                "connected": {
                  "last_value_update": "2016-01-14T11:56:26Z",
                  "value": false
                },
                "family": "5680fd0b6bf9e71158aa5fac",
                "manufacturer": "5680fd0b6bf9e71158aa5fab"
              }
            },
            {
              "_updated": "2016-01-14T11:56:27Z",
              "_id": "56978ceb6bf9e779abef2d2e",
              "name": "foo_tenant_copy_1",
              "_created": "2016-01-14T11:56:27Z",
              "_group": "test_mim_01",
              "_user": "test_mim_01",
              "_etag": "6850f6261a3138872050fc12fb860b5ad389ffa0",
              "connection_instrument": {
                "connected_ts": {
                  "last_value_update": "2016-01-14T11:56:27Z",
                  "value": null
                },
                "connected": {
                  "last_value_update": "2016-01-14T11:56:27Z",
                  "value": false
                },
                "family": "5680fd0b6bf9e71158aa5fac",
                "manufacturer": "5680fd0b6bf9e71158aa5fab"
              }
            }
          ]
        }


    In case of error an output similar to this will be shown:

        #!javascript
        {
            "_status": "ERR",
            "_issues": {
                "exception": "The attribute path foo.bar is not properly defined"
            }
        }

Tests
=====

Describe here how to execute the unit, functional and UI tests of the project.


Unit
----

The available unit tests are located in `tests/unit` directory. To execute them just ensure that you meet the requeriment imposed in the `requeriments_test.txt` file.

* Schema manager unit tests

    To execute the unit tests just execute these line. The output is shown as well:

        #!bash
        py.test -xv tests/unit/test_schema_manager.py
        ======================================== test session starts =========================================
        platform linux2 -- Python 2.7.3 -- py-1.4.30 -- pytest-2.7.2 -- /opt/miimetiq-lite/env/bin/python
        rootdir: /opt/miimetiq-lite/src/tests/unit, inifile:
        collected 28 items

        tests/unit/test_schema_manager.py::TestCreateTenantFile::test_create_tenant_file PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[float-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[float-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[float-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[integer-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[integer-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[integer-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[string-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[string-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[string-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[datetime-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[datetime-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[datetime-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[blob-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[blob-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[blob-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[json-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[json-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[json-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[boolean-None] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[boolean-reader] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_attribute[boolean-writer] PASSED
        tests/unit/test_schema_manager.py::TestAddAttribute::test_add_repeated_attribute PASSED
        tests/unit/test_schema_manager.py::TestUpdateAttribute::test_update_attribute PASSED
        tests/unit/test_schema_manager.py::TestDeleteAttribute::test_update_attribute PASSED
        tests/unit/test_schema_manager.py::TestAddDeleteDictNode::test_add_dict_node PASSED
        tests/unit/test_schema_manager.py::TestAddDeleteDictNode::test_delete_dict_node PASSED
        tests/unit/test_schema_manager.py::TestAddDeleteDictNode::test_delete_uncomplete_path_dict_node PASSED

        ===================================== 28 passed in 0.20 seconds ======================================

Functional
----------

The available unit tests are located in `tests/functional` directory. To execute them just ensure that you meet the requeriment imposed in the `requeriments_test.txt` file.

*Important information*
The functionals tests effectively modify the database and the filesystem. If you want to execute them in a safe environment remember to modify the functional test global variables and the next configurations:

* *settings.conf:SCHEMA_TENANT_DIRECTORY* This options sets the path where all the tenant schemas will be found. Change it to do not overwrite you current tenant schemas.

The available tests are:

* test_tenants_handler_create.py
* test_tenants_handler_update.py
* test_tenants_handler_delete.py

* test_rpc_schema_services.py

All of them can be executed with:

    py.test -v test/functionals/

UI
--

The available functional test for UI are located in `tests/UI`. All of them are programmed in JavaScript and are located in the folder `tests/UI/spec`.

The available test are:

* TenantsComponentSpec.js

*Requirements*
One of the requirements is that the MIIMETIQ services must be available to execute all the functional tests.

*Command line execution*
If you want to execute them from command line it is necessary to have installed `phantomJS`. The instructions to build the binary for a Linux distribution can be found here (http://phantomjs.org/build.html). In order to execute the test it is only necessary to execute the bash script `tests/UI/phantomjs/phantom-run-tests.sh`.

*Browser execution*
The tests can be runned manually by executing them in a browser. In order to execute them a see the results it is only necessary to open the html file `tests/UI/run-tests.html` in a web browser and it will be automatically executed.
