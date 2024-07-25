# Data Platform

## How to deploy
<details>
<summary>Expand for first run instructions.</summary>

### First run
To set up the databases you have the docker compose file `docker-compose_dtplt_firstrun.yaml`.

#### Secrets
For the first run you will need the following files.
In the `secrets/postgres` folder:
- `dtpltauth.sql` has the database credentials.
- `dtpltairflow.sql` creates the airflow database.  
    Is in secrets in order to avoid publishing the airflow database name and users.
- `dtpltpgfile` postgres password file used during the creation of the database.
In the `secrets/minio` folder:
- `.envminio_firstrun` has the envvars with the credentials for the root user and the buckets we want to create.  
    For instance:
    ```.env
    #first run settings
    MINIO_ROOT_USER=foo
    MINIO_ROOT_PASSWORD=bar
    MINIO_DEFAULT_BUCKETS=baz1;baz2
    ```
- `initial.sh` has the script to create buckets, policies, users, groups, and service accounts.
    For instance:
    ```bash
    ## Run the first time to set 
    # docker exec -it mini bash /var/run/secrets/minitial
    # bash /var/run/secrets/minitial

    mc alias set myminio http://localhost:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD

    mc mb myminio/foo
    mc mb myminio/bar

    cat > /tmp/foo-readonly.json <<EOF
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Action": [
            "s3:GetBucketLocation",
            "s3:ListBucket"
        ],
        "Effect": "Allow",
        "Resource": [
            "arn:aws:s3:::foo"
        ]
        },
        {
        "Action": [
            "s3:GetObject"
        ],
        "Effect": "Allow",
        "Resource": [
            "arn:aws:s3:::foo/*"
        ]
        }
    ]
    }
    EOF

    cat > /tmp/foo-readwrite.json <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:*"
                ],
                "Resource": [
                    "arn:aws:s3:::foo",
                    "arn:aws:s3:::foo/*"
                ]
            }
        ]
    }
    EOF

    mc admin policy create myminio foo-readonly /tmp/foo-readonly.json
    mc admin policy create myminio foo-readwrite /tmp/foo-readwrite.json

    mc admin user add myminio alfa yadayadayada
    mc admin user add myminio beta yadayadayada2

    mc admin group add myminio ad alfa
    mc admin group add myminio rer beta

    mc admin policy attach myminio foo-readwrite --group ad
    mc admin policy attach myminio foo-readonly --group rer

    echo "alfa service account. Save the output"
    mc admin user svcacct add myminio alfa --name alfasa --json
    echo "beta service account. Save the output"
    mc admin user svcacct add myminio beta --name betasa --json
    ```

#### Run
Four steps:
1. Run `docker compose -f docker-compose_dtplt_firstrun.yaml up -d`.
2. Wait for the database to be created and online. It should take seconds once the service is running. You can try connecting to it. 
3. Run `docker exec -it mini bash /var/run/secrets/minitial` and copy it's output, it has the credentials of the service account with write access and the one with read access..
4. Run `docker compose -f docker-compose_dtplt_firstrun.yaml down`. 

After this you could delete the secrets, since they won't be used by the platform.
</details>

### Running the platform
We have two compose files:
1. **docker-compose_dtpltbase.yaml** handles the database and starts the network.
2. **docker-compose_dtplt.yaml** handles airflow and the other services.

We have the database in a separate compose because one may want to restart airflow and the other services, stop them and so on more often. But we believe that the database service should be left alone as much as possible.

#### Secrets:
You will need the following files.  
In the `secrets/airflow` folder:
- `.env` with the sensible airflow envvars (or any you just prefer there).

In the `secrets/minio` folder:
- `.envminio` has the envvars for minio. We recommend setting `MINIO_BROWSER_REDIRECT_URL`.  
   ```
   MINIO_BROWSER_REDIRECT_URL=https://foo.example.com/minio/
   ```

In the `secrets/postgres` folder:
- `dtpltpgs` contains the postgres connection service file, with the credentials to the databases.  
    We use this to handle the connections to the database, e.g. from airflow.

### Run
1. Run `docker compose -f docker-compose_dtpltbase.yaml up -d`.
2. Run `docker compose -f docker-compose_dtplt.yaml up -d`.
