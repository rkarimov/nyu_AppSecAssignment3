FROM mysql:latest

#RUN mkdir /data
COPY ./products.csv /products.csv
COPY ./users.csv /users.csv
#COPY ./setup.sql /docker-entrypoint-initdb.d/setup.sql
COPY ./seedingjob.sql /docker-entrypoint-initdb.d/seedingjob.sql

ENTRYPOINT ["/entrypoint.sh"]
CMD ["mysqld", "--secure-file-priv=/"]
#CMD ["--secure-file-priv=/"]
