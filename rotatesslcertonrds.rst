Rotate TLS/SSL Certificate on RDS/Aurora
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
AWS provides CA certificates for connecting to RDS instance using SSL/TLS. If you application connects to SSL or TLS layer you must follow below steps before current certificate expires.

Source material for this guide can be found `here`_.

**Prerequisites**:

Search code repository and connection string to validate which client .pem version is being used. Below are possible client-side certification.

* rds-ca-2015-root.pem
* rds-ca-2019-root.pem
* rds-combined-ca-bundle.pem

It is advisable to search using *.pem. Based on findings you will know what version of client certificate is used.

If ``rds-combined-ca-bundle.pem`` is used, you can avoid or reduce outage compare to using specific root cert or intermediate cert. The certificate bundle contains certificates for both the old and new CA, so you can upgrade your application safely and maintain connectivity during the transition period. If you are using ``rds-ca-2015-root.pem`` or ``rds-ca-2019-root.pem`` or any other intermediate certificates, it is strongly recommanded that you upgrade your client certificates to ``rds-combined-ca-bundle.pem``.
More information on SSL/TLS client side certificates can be found `sslcertinfo`_.


**Step 1:** Validate current version of CA certificate authority

Validate “Certificate authority version” for given individual instance using AWS CLI or AWS console.

node with rds-ca-2015:

.. code::

aws rds describe-db-instances --db-instance-identifier  <instance_name> | grep CACertificateIdentifier
"CACertificateIdentifier": "rds-ca-2015",

node with rds-ca-2019:

.. code::

aws rds describe-db-instances --db-instance-identifier  <instance_name> | grep CACertificateIdentifier
"CACertificateIdentifier": "rds-ca-2019",

If current version of CA certificate is rds-ca-2015 version, it must be upgraded to rds-ca-2019 version before March 05 2020.



**Step 2:** SSL/TLS rotation in reader instances

RDS Instance will reboot as part of TLS/SSL cert rotation. It is advisable to perform cert upgrade in reader instance first. If your cluster is with multiple reader nodes, take one reader node at time using below steps:

In the navigation pane, choose Databases, and then choose the DB instance that you want to modify.
Choose Modify.

The Modify DB Instance page appears.

In the Network & Security section, choose rds-ca-2019.

Choose Continue and check the summary of modifications.
To apply the changes immediately, choose Apply immediately.
Choosing this option causes an outage/reboot of this specific instance.
Click on modify db instance.
It takes around 1 minute on average for RDS instance to reboot. Watch for reader traffic and other statistics on this reader instance using various monitoring tools you normally use.

wait for traffic is fully balanced between all reader instances.

follow same steps for all reader instances.

Validate “Certificate authority version” for given individual instance using AWS CLI or AWS console. it should show ``rds-ca-2019`` version.

.. code::


aws rds describe-db-instances --db-instance-identifier  <instance_name> | grep CACertificateIdentifier
"CACertificateIdentifier": "rds-ca-2019",


**Step 3:** Perform failover and choose new writer

Once all readers are upgraded with ``rds-ca-2019``, perform cluster failover. to perform failover, go to action choose failover.
This will choose new writer instance and old writer instance will become reader.
Watch for writer traffic and other statistics on new writer instance using various monitoring tools you normally use.


**Step 4:** SSL/TLS rotation in last reader instance
follow steps described in step 2 to perform SSL/TLS rotation and perform validation.



**Miscellaneous:**
We encourage you to test the steps listed here in a development or staging environment before taking them for your production environments.


.. _here: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.SSL-certificate-rotation.html
.. _sslcertinfo: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html
