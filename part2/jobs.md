**note to grader, please see "Assignment 3 - full write up" file in home directory with screenshots. This file will have same info, just w/o screenshots**

In order to run migrations, I created a separate migrationjob.yaml file with environment variables that I want to migrate. Additionally, I made sure the commands `['python3', 'manage.py', 'migrate']` were included which would trigger the django migration job itself via manage.py file 

The completed YAML File looks as follows

```
apiVersion: batch/v1
kind: Job
metadata:
  name: migrationjob
spec:
  template:
    spec:
      containers: 
      - name: migrationjob
        image: nyuappsec/assign3:v0
        command: ['python3', 'manage.py', 'migrate']
        env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: securedsecrets
                  key: password

            - name: MYSQL_DB
              value: GiftcardSiteDB

            - name: MYSQL_HOST
              value: mysql-service

            - name: ALLOWED_HOSTS
              value: "*,"

            - name: ADMIN_UNAME
              valueFrom:
                secretKeyRef:
                    name: securedsecrets
                    key: username

            - name: ADMIN_PASS
              valueFrom:
                secretKeyRef:
                    name: securedsecrets
                    key: password

            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                    name: securedsecrets
                    key: secret_key

      restartPolicy: Never
  backoffLimit: 7
  ```

Afterward, I simply ran `kubectl apply -f migrationjob.yaml` which applied the new file and triggered the migration flow. Then, I was able to confirm that migration was completed by running `kubectl get jobs`

The seeding job took me a while to figure out as it involved populating the records from one table into another. However, similar to the migration job, for the seeding job, I started off with creating a new seedjob.yaml file which would make a call to the database with credentials stored in secrets. Most importantly though, I added the argument to make a call to a separate sql job through the `[args]` parameter. Furthermore, I made sure to pass the username and password along with the seedjob.sql script so that they can authenticate against secrets that were created in step 1

The completed YAML File looks as follows
```
apiVersion: batch/v1
kind: Job

metadata:
  name: seedjob
spec:
  template:
    spec:
      containers:
      - name: seedjob
        image: 'nyuappsec/assign3-db:v0'
        command: ["/bin/sh"]
        args: ["-c","mysql --user=root --password=${MYSQL_ROOT_PASSWORD} --database=${MYSQL_DATABASE} --host=mysql-service -f < /docker-entrypoint-initdb.d/seedjob.sql"]
        env:

          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: securedsecrets
                key: password

          - name:  MYSQL_DATABASE
            value: GiftcardSiteDB
            
      restartPolicy: Never
  backoffLimit: 7
```

As to avoid duplication in records being added, I simply adjusted the docker file contained in DB folder to comment out the data folder and initial setup.sql command as this was already run on setup.  
The updated docker file than reads as follows: 
```
FROM mysql:latest

#RUN mkdir /data
COPY ./products.csv /products.csv
COPY ./users.csv /users.csv
#COPY ./setup.sql /docker-entrypoint-initdb.d/setup.sql
COPY ./seedingjob.sql /docker-entrypoint-initdb.d/seedingjob.sql

ENTRYPOINT ["/entrypoint.sh"]
CMD ["mysqld", "--secure-file-priv=/"]
#CMD ["--secure-file-priv=/"]
```
In continuity to earlier step, I adjusted the setup.sql file to comment out the extra data load mentioned here as to avoid duplication with the earlier job.
The following lines were commented out:
```
---- Commenting out to complete step 2 which conducts the seeding job
--
-----LOAD DATA INFILE '/data/products.csv' INTO TABLE LegacySite_product FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' LINES TERMINATED BY '\r\n';
--
-- Put user into table.
--
----LOAD DATA INFILE '/data/users.csv' INTO TABLE LegacySite_user FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' LINES TERMINATED BY '\r\n';
```

As we commented out these lines, I simply re-added them those lines ‘separately’ into the file ‘seedjob.sql’ which would run along with username and password arguments when seedjob.yaml file is applied 
This file than reads as follows:
```
LOAD DATA INFILE '/products.csv' INTO TABLE LegacySite_product FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' LINES TERMINATED BY '\r\n';
LOAD DATA INFILE '/users.csv' INTO TABLE LegacySite_user FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"' LINES TERMINATED BY '\r\n';
```
Finally, similarly to the migration job, I ran `kubectl apply -f seedjob.yaml` to trigger the seeding job. 
The confirm completion of seedjob, I ran kubectl get jobs which showed both migrationjob and seedjob complete 


References: 
https://stackoverflow.com/questions/51577441/how-to-seed-django-project-insert-a-bunch-of-data-into-the-project-for-initi
https://docs.djangoproject.com/en/4.1/howto/initial-data/
https://stackoverflow.com/questions/65974627/clearing-and-seeding-database-from-endpoint-in-django
https://medium.com/@ardho/migration-and-seeding-in-django-3ae322952111
https://stackoverflow.com/questions/60141107/how-do-i-run-django-seed-data-in-my-mysql-docker-image
https://stackoverflow.com/questions/59940160/is-there-a-way-in-a-seed-data-yaml-file-to-autogenerate-models-on-which-first-mo
https://github.com/Brobin/django-seed
https://www.coursehero.com/tutors-problems/Computer-Science/30588605-How-to-create-a-Kubernetes-migrations-job-using-a-YAML-file-for-a-MySQ/
