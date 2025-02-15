+++
title = "QWC Data Service"
menuTitle = "qwc-data-service"
weight = 9
+++


Edit spatial and unlocated features of datasets via GeoJSON.

**Note:** requires a PostGIS database for reading and writing features

Setup
-----

Uses PostgreSQL connection service or connection to a PostGIS database.
This connection's user requires read and write access to the configured tables.

### qwc_demo example

Uses PostgreSQL connection service `qwc_geodb` (GeoDB).
The user `qwc_service_write` requires read and write access to the configured tables
of the data layers from the QGIS project `qwc_demo.qgs`.

Setup PostgreSQL connection service file `~/.pg_service.conf`:

```
host=localhost
port=5439
dbname=qwc_demo
user=qwc_service_write
password=qwc_service_write
sslmode=disable
```


Configuration
-------------

The static config and permission files are stored as JSON files in `$CONFIG_PATH` with subdirectories for each tenant,
e.g. `$CONFIG_PATH/default/*.json`. The default tenant name is `default`.


### Data Service config

* [JSON schema](schemas/qwc-data-service.json)
* File location: `$CONFIG_PATH/<tenant>/dataConfig.json`

Example:
```json
{
  "$schema": "https://raw.githubusercontent.com/qwc-services/qwc-data-service/v2/schemas/qwc-data-service.json",
  "service": "data",
  "config": {
    "attachments_base_dir": "/tmp/qwc_attachments/",
    "allowed_attachment_extensions": ".bmp,.jpg,.pdf",
    "max_attachment_file_size": 10485760,
    "upload_user_field_suffix": "uploaduser",
    "edit_user_field": "edituser",
    "edit_timestamp_field": "edittimestamp"
  },
  "resources": {
    "datasets": [
      {
        "name": "qwc_demo.edit_points",
        "db_url": "postgresql:///?service=qwc_geodb",
        "schema": "qwc_geodb",
        "table_name": "edit_points",
        "primary_key": "id",
        "fields": [
          {
            "name": "id",
            "data_type": "integer",
            "constraints": {
              "min": -2147483648,
              "max": 2147483647
            }
          },
          {
            "name": "name",
            "data_type": "character varying",
            "constraints": {
              "maxlength": 32
            }
          },
          {
            "name": "description",
            "data_type": "text"
          },
          {
            "name": "num",
            "data_type": "integer",
            "constraints": {
              "min": -2147483648,
              "max": 2147483647
            }
          },
          {
            "name": "value",
            "data_type": "double precision",
            "constraints": {
              "pattern": "[0-9]+([\\.,][0-9]+)?"
            }
          },
          {
            "name": "type",
            "data_type": "smallint",
            "constraints": {
              "min": -32768,
              "max": 32767
            }
          },
          {
            "name": "amount",
            "data_type": "numeric",
            "constraints": {
              "numeric_precision": 5,
              "numeric_scale": 2,
              "min": -999.99,
              "max": 999.99,
              "step": 0.01
            }
          },
          {
            "name": "validated",
            "data_type": "boolean"
          },
          {
            "name": "datetime",
            "data_type": "timestamp without time zone"
          }
        ],
        "geometry": {
          "geometry_column": "geom",
          "geometry_type": "POINT",
          "srid": 3857
        }
      }
    ]
  }
}
```

### Permissions

* [JSON schema](https://github.com/qwc-services/qwc-services-core/blob/master/schemas/qwc-services-permissions.json)
* File location: `$CONFIG_PATH/<tenant>/permissions.json`

Example:
```json
{
  "$schema": "https://raw.githubusercontent.com/qwc-services/qwc-services-core/master/schemas/qwc-services-permissions.json",
  "users": [
    {
      "name": "demo",
      "groups": ["demo"],
      "roles": []
    }
  ],
  "groups": [
    {
      "name": "demo",
      "roles": ["demo"]
    }
  ],
  "roles": [
    {
      "role": "public",
      "permissions": {
        "data_datasets": [
          {
            "name": "qwc_demo.edit_points",
            "attributes": [
              "id",
              "name",
              "description",
              "num",
              "value",
              "type",
              "amount",
              "validated",
              "datetime"
            ],
            "writable": true,
            "creatable": true,
            "readable": true,
            "updatable": true,
            "deletable": true
          }
        ]
      }
    }
  ]
}
```

### Overview

                                     QWC2 build:                                      Public:
    +--------------------+           +------------+
    | themesConfig.json  |           |            |
    |          ^         +-----------> yarn build +----------------------------------> (themes.json)
    |          |         |       +--->            |
    |          +         +-----+ |   +------------+       +---------------+
    |        (edit.json) |     | |                        |               |
    +--------------------+     | +------------------------+ QGIS Server   +----------> WMS/WFS
                               | |      Capabilities      |               |
    +--------------------+     | |                        +------^--------+
    | ui Files           +-----------------------------------------------------------> assets/*.ui
    +--------------------+     | |                               |
                               | |                               |                    +------------------+
    +--------------------+     | |                               +                    |                  +--> html/js/css/assets
    | qgs Files          +--+-------------------------------> *.qgs         +---------> qwc-map-viewer   |
    +--------------------+  |  | |                                          |         |                  +--> config.json/themes.json
                            |  | |                                          |         +------------------+
                            |  | |                                          |
                            |  | |                                          |         +------------------+
                            |  | |                                          |         |                  |  REST-API
        +---------+         |  | |                                          +---------> qwc-data-service <--------------->
        |         |         |  | |   +------------+                         |         |                  |
        | config- |         |  | +--->            |                         +         +------------------+
        |   DB    | +-------v--v-----> Config-    +------> [service].json+permissions.json
        |         |                  | Generator  |
        |         |                  |            |
        +---------+                  +------------+

Edit forms:

- Edit forms are automatically created from field information extracted from QGS files, according to the attribute form configuration:
  * For autogenerated attribute forms, a flat edit form is in QWC2
  * Drag and Drop designer forms are automatically translated to Qt Designer forms in the `assets/forms/autogen` and loaded by QWC2
  * UI file are copied to the `assets/forms/autogen` folder and loaded by QWC2

- You can also manually create Qt Designer forms to use exclusively with QWC2 as follows:
  * For the desired theme, add a block as follows in the theme block of the `tenantConfig.json`:

        "editConfig":{
          "<layername>":{
            "editDataset":"<mapprefix>.<datasetname>",
            "layerName":"<Display name>",
            "geomType":"<Point|LineString|Polygon>",
            "form":":/forms/form.ui"
          }
        }

    Note: `:/` in the `form` property is resolved to the assets directory of the viewer.
  * Create the designer form in Qt-Designer, using the dataset field names as edit widget names.
- *Note*: In general, for tables with an auto-incrementing primary key field, you'll want to set the attribute form widget type to "Hidden" in the QGIS layer properties. This way, the data-service won't block the commit if the feature is comitted with an empty PK field value.

Logging the user/timestamp of a mutation:

- If you set `edit_user_field` in the data service config, the username of the last user who performed a mutation to a record with be logged to the `<edit_user_field>` field of the record.
- If you set `edit_timestamp_field` in the data service config, the timestamp of the last mutation to a record will be logged to the `<edit_timestamp_field>` field of the record.

File uploads:

- For autogenerated edit forms, in QGIS create a virtual field named `<fieldname>__upload` for the desired layer. `<fieldname>` must be equal to an existing field of the dataset of type string. As field expression, you can set a string `'<extensions>'`, with `<extensions>` a comma separated list of suggested file extensions, i.e. `.png,.jpg`.
- For Drag and Drop designer forms, set the widget type to Attachment.
- For manually created Qt Designed Ui forms, use a `QLineEdit` widget named `<fieldname>__upload`, and optionally as the text value of the `QLineEdit` set a comma separated list of suggested file extensions.
- *Note*: Make sure the client uses the EditingInterface.js shipped with [QWC2 submodule f053fdc or newer](https://github.com/qgis/qwc2/commit/f053fdceba4a2d8112fb516c793702b9fd118609) to support file uploads.

If you set `upload_user_field_suffix` in the data service config, the username of the last user who performed an upload to `<fieldname>` will be logged to `<fieldname>__<upload_user_field_suffix>`. You can set the respective field types to hidden in the QGIS layer properties to avoid them showing up in the autogenerated edit forms.

1:N relations:

- *Note*: 1:N relations are currently only supported in manually created Qt-Designed Ui forms.
- In your Ui form, create a `QWidget` named according to the pattern `nrel__<reltablename>__<foreignkeyfield>`, where `<reltablename>` is the name of the relation table and `<foreignkeyfield>` the name of the foreign key field in the relation table.
- Inside this widget, add the edit widgets for the values of the relation table. Name the widgets `<reltablename>__<fieldname>`. These edit widgets will be replicated for each relation record.
- *Note:* The relation table needs to be added as a (geometryless) table to the QGIS Project. You also need to set appropriate permissions for the relation table dataset in the QWC admin backend.

Key-value relations:

- For Drag and Drop designer forms, use widgets of type Value Relation. In the generated designer form, the naming convention indicated below is used.
- In a manually created Qt-Designer Ui form, you can use key-value relations for combo box entries by naming the `QComboBox` widget according to the following pattern: `kvrel__<fieldname>__<kvtablename>__<kvtable_valuefield>__<kvtable_labelfield>`. `<kvtablename>` refers to a table containing a field called `<kvtable_valuefield>` for the value of the entry and a field `<kvtable_labelfield>` for the label of the entry. For key-value relations inside a 1:N relation, use `kvrel__<reltablename>__<fieldname>__<kvtablename>__<kvtable_valuefield>__<kvtable_labelfield>`. `<kvtablename>`
- *Note:* In any case, the relation table needs to be added as a (geometryless) table to the QGIS Project. You also need to set appropriate permissions for the relation table dataset in the QWC admin backend.

Data service configuration:

- DB connection information, table and column names and primary key information are extracted from QGS files
- Data contraints are extracted from QGS files
- Column types and additional constraints are read from the the geo-DB

Data read/write:

- QWC2 issues data-service API requests for reading und writing


Usage
-----

Set the `CONFIG_PATH` environment variable to the path containing the service config and permission files when starting this service (default: `config`).

Base URL:

    http://localhost:5012/

Service API:

    http://localhost:5012/api/

Sample requests:

    curl 'http://localhost:5012/qwc_demo.edit_points/'


Docker usage
------------

To run this docker image you will need a PostGIS database. For testing purposes you can use the demo DB.

The following steps explain how to download the demo DB docker image and how to run the `qwc-data-service` with `docker-compose`.

**Step 1: Clone qwc-docker**

    git clone https://github.com/qwc-services/qwc-docker
    cd qwc-docker

**Step 2: Create docker-compose.yml file**

    cp docker-compose-example.yml docker-compose.yml

**Step 3: Start docker containers**

    docker-compose up qwc-data-service

For more information please visit: https://github.com/qwc-services/qwc-docker

Development
-----------

Create a virtual environment:

    virtualenv --python=/usr/bin/python3 --system-site-packages .venv

Without system packages:

    virtualenv --python=/usr/bin/python3 .venv

Activate virtual environment:

    source .venv/bin/activate

Install requirements:

    pip install -r requirements.txt

Start local service:

    CONFIG_PATH=/PATH/TO/CONFIGS/ python server.py


Testing
-------

Run all tests:

    python test.py

Run single test module:

    python -m unittest tests.feature_validation_tests

Run single test case:

    python -m unittest tests.feature_validation_tests.FeatureValidationTestCase

Run single test method:

    python -m unittest tests.feature_validation_tests.FeatureValidationTestCase.test_field_constraints
