# Considerations for restoring DB instances<a name="rds-on-outposts.restoring"></a>

When you restore a DB instance in Amazon RDS on AWS Outposts, you can choose the storage location for automated backups and manual snapshots of the restored DB instance\.

Restoring from a manual DB snapshot, you can store backups either in the parent AWS Region or locally on your Outpost\.

Restoring from an automated backup \(point\-in\-time recovery\), you have fewer choices:
+ If restoring from the parent AWS Region, you can store backups either in the AWS Region or on your Outpost\.
+ If restoring from your Outpost, you can store backups only on your Outpost\.