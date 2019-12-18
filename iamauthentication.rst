Enabling IAM Authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~~
New Aurora clusters created in DSTerraform have IAM authentication enabled on a cluster by default.
This means that owners of a cluster should make the following client changes on their side to make full use of the feature.
Source material for this guide can be found `here`_.

**Preqrequisites**:

If you have a cluster in production that does not have clients connecting with IAM Authentication, it's important
to run testing on staging first to verify end to end functionality. Additionally, IAM Authentication works well for the
majority of use cases, but you should keep in mind the following limitations:

* Use IAM database authentication only for workloads that can be easily retried.
* Don't use IAM database authentication if your application requires more than 256 new connections per second.

**Step 1:** Verify that IAM authentication is enabled on the cluster

DSTerraform provides this by default, but to double check the setting, you can run the following using the CLI:

``aws rds describe-db-clusters --db-cluster-identifier mydbcluster``

The IAMDatabaseAuthenticationEnabled field should be set to True.

**Step 2:** Update the service.sls file in the repo of the service accessing your Aurora cluster to make sure that your
service's role adds a policy allowing IAM access to your cluster. It should look something like the following:

.. code::

    {
       "Version": "2012-10-17",
       "Statement": [
          {
             "Effect": "Allow",
             "Action": [
                 "rds-db:connect"
             ],
             "Resource": [
                 "arn:aws:rds-db:us-east-1:1234567890:dbuser:cluster-ABCDEFGHIJKL01234/db_user"
             ]
          }
       ]
    }


Keep in mind that the piece of the stanza above pertaining to ``cluster-ABCDEFGHIJKL01234`` will be the DbClusterResourceId
field in the metadata given from step 1. If you are **not** using Aurora, instead of cluster-<ID> it would be db-<ID>.

**Step 3:** Download a `AWS Intermediate certificate`_ (pertaining to the region in which your cluster resides) on the machine accessing your cluster.

**Step 4:** Update your client to `generate an authentication token`_ and use this value as a password for future connections.

**Step 5:** Connect to the cluster by generating a token and connecting with all the parameters. The example below uses psycopg2 (one of the SQLAlchemy `dialects`_):


.. code::

    client = get_aws_client('rds')
    token = client.generate_db_auth_token(db_hostname, DB_PORT, DB_USERNAME)
    return psycopg2.connect(
         host=db_hostname,
         port=DB_PORT,
         dbname=DB_NAME,
         user=DB_USERNAME,
         password=token,
         sslmode='verify-full',
         sslrootcert=SSL_CERT_PATH,
     )


**Miscellaneous:**
If you're creating additional users, it's also important to make sure they're created with the ability to connect with IAM Authentication.
View `this page`_ for user creation details.


.. _here: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.html
.. _generate an authentication token: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds.html#RDS.Client.generate_db_auth_token
.. _this page: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.DBAccounts.html
.. _AWS Intermediate certificate: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.IntermediateCertificates
.. _dialects: https://docs.sqlalchemy.org/en/13/dialects/postgresql.html
