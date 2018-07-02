# Terraform Redshift Provider

Manage Redshift users, groups, privileges, databases and schemas. It runs the SQL queries necessary to manage these (CREATE USER, DELETE DATABASE etc)
in transactions, and also reads the state from the tables that store this state, eg pg_user_info, pg_group etc. The underlying tables are more or less equivalent to the postgres tables, 
but some tables are not accessible in Redshift. 

Currently only supports users, groups and databases. Privileges and schemas coming soon! 

## Limitations
For authoritative limitations, please see the Redshift documentations. 
1) You cannot delete the database you are currently connected to. 
2) You cannot set table specific privileges since this provider is table agnostic (for now, if you think it would be feasible to manage tables let me know)
3) On importing a user, it is impossible to read the password (or even the md hash of the password, since Redshift restricts access to pg_shadow)

## Example: 

Provider configuration
```
provider redshift {
  "url" = "localhost",
  user = "testroot",
  password = "Rootpass123",
  database = "dev"
}
```

Creating an admin user who is in a group and who owns a new database, with a password that expires
```
# Create a user
resource "redshift_user" "testuser"{
  "username" = "testusernew" # User name are not immutable. 
  # Terraform can't read passwords, so if the user changes their password it will not be picked up. One caveat is that when the user name is changed, the password is reset to this value
  "password" = "Testpass123" 
  "valid_until" = "2018-10-30" # See below for an example with 'password_disabled'
  "connection_limit" = "4"
  "createdb" = true
  "syslog_access" = "UNRESTICTED"
  "superuser" = true
}

# Add the user to a new group
resource "redshift_group" "testgroup" {
  "group_name" = "testgroup" # Group names are not immutable
  "users" = ["${redshift_user.testuser.id}"] # A list of user ids as output by terraform (from the pg_user_info table), not a list of usernames (they are not immnutable)
}

# Create a schema
resource "redshift_schema" "testschema" {
  "schema_name" = "testschema", # Schema names are not immutable
  "owner" ="${redshift_user.testuser.id}", # This defaults to the current user (eg as specified in the provider config) if empty
  "cascade_on_delete" = true
}

# Give that group select, insert and references privileges on that schema
resource "redshift_group_schema_privilege" "testgroup_testchema_privileges" {
  "schema_id" = "${redshift_schema.testschema.id}" # Id rather than group name
  "group_id" = "${redshift_group.testgroup.id}" # Id rather than group name
  "select" = true
  "insert" = true
  "update" = false
  "references" = true
  "delete" = false # False values are optional
}
```

You can only create resources in the db configured in the provider block. Since you cannot configure providers with 
the output of resources, if you want to create a db and configure resources you will need to configure it through a `terraform_remote_state` data provider. 
Even if you specifiy the name directly rather than as a variable, since providers are configured before resources you will need to have them in separate projects. 

```
# First file:

resource "redshift_database" "testdb" {
  "database_name" = "testdb", # This isn't immutable
  "owner" ="${redshift_user.testuser.id}",
  "connection_limit" = "4"
}

output "testdb_name" {
  value = "${redshift_database.testdb.database_name}"
}

# Second file: 

data "terraform_remote_state" "redshift" {
  backend = "s3"
  config {
    bucket = "somebucket"
    key = "somekey"
    region = "us-east-1"
  }
}

provider redshift {
  "url" = "localhost",
  user = "testroot",
  password = "Rootpass123",
  database = "${data.terraform_remote_state.redshift.testdb_name}"
}

```

Creating a user who can only connect using IAM Credentials as described [here](https://docs.aws.amazon.com/redshift/latest/mgmt/generating-user-credentials.html)

```
resource "redshift_user" "testuser"{
  "username" = "testusernew",
  "password_disabled" = true
  "connection_limit" = "1"
}
```

## Prequisites to development
1. Go installed
2. Terraform installed locally

## Building: 
1. Run `go build -o terraform-provider-redshift_v0.0.1_x4.exe`. You will need to tweak this with GOOS and GOARCH if you are planning to build it for different OSes and architectures
2. Add to terraform plugins directory: https://www.terraform.io/docs/configuration/providers.html#third-party-plugins

You can debug crudely by setting the TF_LOG env variable to DEBUG. Eg 
```
$ TF_LOG=DEBUG terraform apply
```

## I usually connect through an ssh tunnel, what do I do?
The easiest thing is probably to update your hosts file so that the url resolves to localhost

## TODO 
1. Database property for Schema
2. Schema privileges on a per user basis
3. Add privileges for languages and functions