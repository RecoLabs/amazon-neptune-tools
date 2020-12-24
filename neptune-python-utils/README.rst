neptune-python-utils
====================

_neptune-python-utils_ is a Python 3 library that simplifies using `Gremlin-Python](https://pypi.org/project/gremlinpython/) to connect to Amazon Neptune. The library makes it easy to configure your driver to support [IAM DB Authentication <https://docs.aws.amazon.com/neptune/latest/userguide/iam-auth.html>`_ to connect to Amazon Neptune. The library makes it easy to configure your driver to support `IAM DB Authentication <https://docs.aws.amazon.com/neptune/latest/userguide/iam-auth.html>`_, create sessioned interactions with Neptune, and write data to Amazon Neptune from AWS Glue jobs.

You can use *neptune-python-utils* in AWS Lambda functions, Jupyter notebooks, AWS Glue PySpark and Python shell jobs, and in your own Python applications.

With *neptune-python-utils* you can:

 - Connect to Neptune using `IAM DB Authentication <https://docs.aws.amazon.com/neptune/latest/userguide/iam-auth.html>`_

 - Trigger and monitor `bulk load operations <https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html>`_

 - Create a `sessioned client <https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-sessions.html>`_ for implicit transactions that span multiple requests

 - Get Neptune connection information from the Glue Data Catalog

 - Create label and node and edge ID columns in DynamicFrames, named in accordance with the Neptune CSV bulk load format for property graphs

 - Write from DynamicFrames directly to Neptune 
 
# Build
=======

`sh build.sh`

This creates a zip file: `target/neptune*python*utils.zip`. 

When using AWS Glue to write data to Neptune, copy the zip file to an S3 bucket. You can then refer to *neptune-python-utils* from your Glue Development Endpoint or Glue job. See `Using Python Libraries with AWS Glue <https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-python-libraries.html>`_. 

# Examples
==========

## Querying
===========

The following query uses `NEPTUNE*CLUSTER*ENDPOINT` and `NEPTUNE*CLUSTER*PORT` environment variables to create a connection to the Gremlin endpoint. It automatically uses the credntial provider chain to connect to the database if IAM DB Auth is enabled.

```

from neptune*python*utils.gremlin_utils import GremlinUtils

GremlinUtils.init_statics(globals())

gremlin_utils = GremlinUtils()

conn = gremlin*utils.remote*connection()

g = gremlin*utils.traversal*source(connection=conn)

print(g.V().limit(10).valueMap().toList())

conn.close()

```

If you want to supply your own endpoint information, you can use `neptune*endpoint` and `neptune*port` parameters to an `Endpoints` object:

```

from neptune*python*utils.gremlin_utils import GremlinUtils

from neptune*python*utils.endpoints import Endpoints

GremlinUtils.init_statics(globals())

endpoints = Endpoints(neptune_endpoint='demo.cluster-111222333.eu-west-2.neptune.amazonaws.com')

gremlin_utils = GremlinUtils(endpoints)

conn = gremlin*utils.remote*connection()

g = gremlin*utils.traversal*source(connection=conn)

print(g.V().limit(10).valueMap().toList())

conn.close()

```

If you want to supply your own credentials, you can supply a `Credentials` object to the `Endpoints` object. Here we're simply getting the credentials from the session (you don't normally have to do this – *neptune-python-utils* will get credentials from the provider chain automatically).

```

from neptune*python*utils.gremlin_utils import GremlinUtils

from neptune*python*utils.endpoints import Endpoints

import boto3

GremlinUtils.init_statics(globals())

session = boto3.session.Session()

credentials = session.get_credentials()

endpoints = Endpoints(

	neptune\_endpoint='demo.cluster\-111222333.eu\-west\-2.neptune.amazonaws.com',

	credentials=credentials)

gremlin_utils = GremlinUtils(endpoints)

conn = gremlin*utils.remote*connection()

g = gremlin*utils.traversal*source(connection=conn)

print(g.V().limit(10).valueMap().toList())

conn.close()

```

## Sessioned client
===================

The following code creates a sessioned client. All requests sent using this client will be executed in a single implicit transaction. The transaction will commit when the sessioned client is close (we're using a `with` block here to close the session). The transaction will be rolled back if an exception occurs:

```

from neptune*python*utils.gremlin_utils import GremlinUtils

GremlinUtils.init_statics(globals())

gremlin_utils = GremlinUtils()

try:

	with gremlin\_utils.sessioned\_client() as client:

		client.submit("g.addV('User').property(T.id, 'person\-x')").all().result()

		client.submit("g.addV('User').property(T.id, 'person\-y')").all().result()

		client.submit("g.V('person\-x').addE('KNOWS').to(V('person\-y'))").all().result()

except Exception as err:

	print('Error: {}'.format(err))

	print('Rolling back session')
	
g = gremlin*utils.traversal*source()

print(g.V('person-x').outE('KNOWS').count().next())

gremlin_utils.close()

``` 

## Bulk loading data into Neptune
=================================

`BulkLoad` automatically supports IAM DB Auth, just as `GremlinUtils` does. You can supply an `Endpoints` object with custom credentials to the `endpoints` parameter of the `BulkLoad` constructor if necessary.

To bulk load and block until the load is complete:

```

from neptune*python*utils.bulkload import BulkLoad

bulkload = BulkLoad(

	source='s3://...', 

	update\_single\_cardinality\_properties=True)

bulkload.load()

```

Alternatively you can invoke the load and check the status using the returned `BulkLoadStatus` object:

```

from neptune*python*utils.bulkload import BulkLoad

bulkload = BulkLoad(

	source='s3://ianrob\-neptune\-lhr/mysql\-to\-neptune/', 

	update\_single\_cardinality\_properties=True)

load*status = bulkload.load*async()

status, json = load_status.status(details=True, errors=True)

print(json)

load_status.wait()

```

# Using neptune-python-utils with AWS Glue
==========================================

## Connecting to Neptune from an AWS Glue job using IAM DB Auth
===============================================================

To connect to an IAM DB Auth-enabled Neptune database from an AWS Glue job, complete the following steps:

### 1. Create a Neptune access role that your AWS Glue job can assume
=====================================================================

Create a Neptune access IAM role with a policy that allows connections to your Neptune database using IAM database authentication. For example:

```

{

	"Version": "2012\-10\-17",

	"Statement": [

		{

			"Action": "neptune\-db:connect",

			"Resource": "arn:aws:neptune\-db:eu\-west\-1:111111111111:\*/\*",

			"Effect": "Allow"

		}

	]

}

```

Instead of `*/*` you should consider restricting access to a specific cluster. See `Creating and Using an IAM Policy for IAM Database Access <https://docs.aws.amazon.com/neptune/latest/userguide/iam-auth.html#iam-auth-policy>`_ for more details.

### 2. Create a trust relationship that allows the Neptune access role to be assumed by your Glue job's IAM role
================================================================================================================

If your AWS Glue job runs with the `MyGlueIAMRole` IAM role, then create a trust relationship attached to the Neptune access role created in Step 1. that looks like this:

```

{

  "Version": "2012-10-17",

  "Statement": [

	{

	  "Effect": "Allow",

	  "Principal": {

		"AWS": "arn:aws:iam::111111111111:role/MyGlueIAMRole"

	  },

	  "Action": "sts:AssumeRole"

	}

  ]
}

```

### 3. Attach a policy to your Glue job's IAM role allowing it to assume the Neptune access role
================================================================================================

If your Neptune access IAM role as created in Step 1. has the ARN `arn:aws:iam::111111111111:role/GlueConnectToNeptuneRole`, attach the following inline policy to your Glue job's IAM role:

```

{

	"Version": "2012\-10\-17",

	"Statement": [

		{

			"Action": "sts:AssumeRole",

			"Resource": "arn:aws:iam::111111111111:role/GlueConnectToNeptuneRole",

			"Effect": "Allow"

		}

	]

}

```

### 4. In your PySpark job or Python shell script, assume the access role and create a Credentials object
=========================================================================================================

Assuming your Neptune access IAM role as created in Step 1. has the ARN `arn:aws:iam::111111111111:role/GlueConnectToNeptuneRole`, you can assume the role like this:

```

import boto3, uuid

from botocore.credentials import Credentials

region = 'eu-west-1'

role_arn = 'arn:aws:iam::111111111111:role/GlueConnectToNeptuneRole'

sts = boto3.client('sts', region_name=region)

role = sts.assume_role(

	RoleArn=role\_arn,

	RoleSessionName=uuid.uuid4().hex,

	DurationSeconds=3600

)

credentials = Credentials(

	access\_key=role['Credentials']['AccessKeyId'], 

	secret\_key=role['Credentials']['SecretAccessKey'], 

	token=role['Credentials']['SessionToken'])

```

This `Credentials` object can then be passed to an `Endpoints` object:

```

from neptune*python*utils.gremlin_utils import GremlinUtils

from neptune*python*utils.endpoints import Endpoints

import boto3

session = boto3.session.Session()

credentials = session.get_credentials()

endpoints = Endpoints(credentials=credentials)

gremlin_utils = GremlinUtils(endpoints)

conn = gremlin*utils.remote*connection()

g = gremlin*utils.traversal*source(connection=conn)

print(g.V().limit(10).valueMap().toList())

conn.close()

```

Credentials generated via `sts.assume_role()` last an hour. If you have a long running Glue job, you may want to create a `RefreshableCredentials` object. See `this article <https://dev.to/li_chastina/auto-refresh-aws-tokens-using-iam-role-and-boto3-2cjf>`_ for more details.

If using a `GlueNeptuneConnectionInfo` object to get Neptune connection information from the Glue Data Catalog, simply pass the region and Neptune access IAM role ARN to the `GlueNeptuneConnectionInfo` constructor:

```

import sys

from awsglue.utils import getResolvedOptions

from pyspark.context import SparkContext

from awsglue.context import GlueContext

from awsglue.job import Job

from neptune*python*utils.glue*neptune*connection_info import GlueNeptuneConnectionInfo

from neptune*python*utils.endpoints import Endpoints

args = getResolvedOptions(sys.argv, ['JOB*NAME', 'AWS*REGION', 'CONNECT*TO*NEPTUNE*ROLE*ARN'])

sc = SparkContext()

glueContext = GlueContext(sc)
 
job = Job(glueContext)

job.init(args['JOB_NAME'], args)

region = args['AWS_REGION']

role*arn = args['CONNECT*TO*NEPTUNE*ROLE_ARN']

endpoints = GlueNeptuneConnectionInfo(region, role*arn).neptune*endpoints('neptune-db')

```

## Using neptune-python-utils to insert or upsert data from an AWS Glue job
===========================================================================

The code below, taken from the sample Glue job `export-from-mysql-to-neptune.py <https://github.com/aws-samples/amazon-neptune-samples/blob/master/gremlin/glue-neptune/glue-jobs/mysql-neptune/export-from-mysql-to-neptune.py>`_, shows extracting data from several tables in an RDBMS, formatting the dynamic frame columns according to the Neptune bulk load CSV column headings format, and then bulk loading direct into Neptune.

Parallel inserts and upserts can sometimes trigger a `ConcurrentModifcationException`. *neptune-python-utils* will attempt 5 retries for each batch should such exceptions occur. 

```

import sys, boto3, os

from awsglue.utils import getResolvedOptions

from pyspark.context import SparkContext

from awsglue.context import GlueContext

from awsglue.job import Job

from awsglue.transforms import ApplyMapping

from awsglue.transforms import RenameField

from awsglue.transforms import SelectFields

from awsglue.dynamicframe import DynamicFrame

from pyspark.sql.functions import lit

from pyspark.sql.functions import format_string

from gremlin_python import statics

from gremlin_python.structure.graph import Graph

from gremlin*python.process.graph*traversal import __

from gremlin_python.process.strategies import *

from gremlin*python.driver.driver*remote_connection import DriverRemoteConnection

from gremlin_python.process.traversal import *

from neptune*python*utils.glue*neptune*connection_info import GlueNeptuneConnectionInfo

from neptune*python*utils.glue*gremlin*client import GlueGremlinClient

from neptune*python*utils.glue*gremlin*csv_transforms import GlueGremlinCsvTransforms

from neptune*python*utils.endpoints import Endpoints

from neptune*python*utils.gremlin_utils import GremlinUtils

args = getResolvedOptions(sys.argv, ['JOB*NAME', 'DATABASE*NAME', 'NEPTUNE*CONNECTION*NAME', 'AWS*REGION', 'CONNECT*TO*NEPTUNE*ROLE_ARN'])

sc = SparkContext()

glueContext = GlueContext(sc)
 
job = Job(glueContext)

job.init(args['JOB_NAME'], args)

database = args['DATABASE_NAME']

product*table = 'salesdb*product'

product*category*table = 'salesdb*product*category'

supplier*table = 'salesdb*supplier'

Create Gremlin client
=====================

gremlin*endpoints = GlueNeptuneConnectionInfo(args['AWS*REGION'], args['CONNECT*TO*NEPTUNE*ROLE*ARN']).neptune*endpoints(args['NEPTUNE*CONNECTION_NAME'])

gremlin*client = GlueGremlinClient(gremlin*endpoints)

Create Product vertices
=======================

print("Creating Product vertices...")

1. Get data from source SQL database
====================================

datasource0 = glueContext.create*dynamic*frame.from*catalog(database = database, table*name = product*table, transformation*ctx = "datasource0")

datasource1 = glueContext.create*dynamic*frame.from*catalog(database = database, table*name = product*category*table, transformation_ctx = "datasource1")

datasource2 = datasource0.join( ["CATEGORY*ID"],["CATEGORY*ID"], datasource1, transformation_ctx = "join")

2. Map fields to bulk load CSV column headings format
=====================================================

applymapping1 = ApplyMapping.apply(frame = datasource2, mappings = [("NAME", "string", "name:String", "string"), ("UNIT*PRICE", "decimal(10,2)", "unitPrice", "string"), ("PRODUCT*ID", "int", "productId", "int"), ("QUANTITY*PER*UNIT", "int", "quantityPerUnit:Int", "int"), ("CATEGORY*ID", "int", "category*id", "int"), ("SUPPLIER*ID", "int", "supplierId", "int"), ("CATEGORY*NAME", "string", "category:String", "string"), ("DESCRIPTION", "string", "description:String", "string"), ("IMAGE*URL", "string", "imageUrl:String", "string")], transformation*ctx = "applymapping1")

3. Append prefixes to values in ID columns (ensures vertices for diffferent types have unique IDs across graph)
===============================================================================================================

applymapping1 = GlueGremlinCsvTransforms.create*prefixed*columns(applymapping1, [('~id', 'productId', 'p'),('~to', 'supplierId', 's')])

4. Select fields for upsert
===========================

selectfields1 = SelectFields.apply(frame = applymapping1, paths = ["~id", "name:String", "category:String", "description:String", "unitPrice", "quantityPerUnit:Int", "imageUrl:String"], transformation_ctx = "selectfields1")

5. Upsert batches of vertices
=============================

selectfields1.toDF().foreachPartition(gremlin*client.upsert*vertices('Product', batch_size=100))

Create Supplier vertices
========================

print("Creating Supplier vertices...")

1. Get data from source SQL database
====================================

datasource3 = glueContext.create*dynamic*frame.from*catalog(database = database, table*name = supplier*table, transformation*ctx = "datasource3")

2. Map fields to bulk load CSV column headings format
=====================================================

applymapping2 = ApplyMapping.apply(frame = datasource3, mappings = [("COUNTRY", "string", "country:String", "string"), ("ADDRESS", "string", "address:String", "string"), ("NAME", "string", "name:String", "string"), ("STATE", "string", "state:String", "string"), ("SUPPLIER*ID", "int", "supplierId", "int"), ("CITY", "string", "city:String", "string"), ("PHONE", "string", "phone:String", "string")], transformation*ctx = "applymapping1")

3. Append prefixes to values in ID columns (ensures vertices for diffferent types have unique IDs across graph)
===============================================================================================================

applymapping2 = GlueGremlinCsvTransforms.create*prefixed*columns(applymapping2, [('~id', 'supplierId', 's')])

4. Select fields for upsert
===========================

selectfields3 = SelectFields.apply(frame = applymapping2, paths = ["~id", "country:String", "address:String", "city:String", "phone:String", "name:String", "state:String"], transformation_ctx = "selectfields3")

5. Upsert batches of vertices
=============================

selectfields3.toDF().foreachPartition(gremlin*client.upsert*vertices('Supplier', batch_size=100))

SUPPLIER edges
==============

print("Creating SUPPLIER edges...")

1. Reuse existing DF, but rename ~id column to ~from
====================================================

applymapping1 = RenameField.apply(applymapping1, "~id", "~from")

2. Create unique edge IDs
=========================

applymapping1 = GlueGremlinCsvTransforms.create*edge*id_column(applymapping1, '~from', '~to')

3. Select fields for upsert
===========================

selectfields2 = SelectFields.apply(frame = applymapping1, paths = ["~id", "~from", "~to"], transformation_ctx = "selectfields2")

4. Upsert batches of edges
==========================

selectfields2.toDF().foreachPartition(gremlin*client.upsert*edges('SUPPLIER', batch_size=100))

End
===

job.commit()

print("Done")

```

## Further Examples
===================

See `Migrating from MySQL to Amazon Neptune using AWS Glue <https://github.com/aws-samples/amazon-neptune-samples/tree/master/gremlin/glue-neptune>`_.
 
## Cross Account/Region Datasources
===================================

If you have a datasource in a different region and/or different account from Glue and your Neptune database, you can follow the instructions in this `blog <https://aws.amazon.com/blogs/big-data/create-cross-account-and-cross-region-aws-glue-connections/>`_ to allow access.
 

 

