Database.com FDW for PostgreSQL
===============================

This Python module implements the `multicorn.ForeignDataWrapper` interface to allow you to create foreign tables in PostgreSQL 9.1+ that map to sobjects in database.com/Force.com. Column names and qualifiers (e.g. `Name LIKE 'P%'`) are passed to database.com to minimize the amount of data on the wire.

* Version 0.0.7 fix a bug that prevented querying using null values
* Version 0.0.6 allow to set api_version in the options
* Version 0.0.5 fix utf8 encoding error, when querying with non ascii characters
* Version 0.0.4 updates yajl-py refs to prevent YajlContentHandler is not defined issue with latest yajl and yajl-py
* Version 0.0.3 removes the requirement for column names to be a case-sensitive match for the database.com field names.
* Version 0.0.2 switched to using the yajl-py streaming JSON parser to avoid reading the entire response from database.com into memory at once.

Pre-requisites
--------------

* [PostgreSQL 9.1+](http://www.postgresql.org/)
* [Python](http://python.org/)
* [Multicorn](http://multicorn.org)
* [yajl-py](http://pykler.github.com/yajl-py/)
* [YAJL](http://lloyd.github.com/yajl/)

Installation
------------

1. [Create a Remote Access Application](http://wiki.developerforce.com/page/Getting_Started_with_the_Force.com_REST_API#Setup), since you will need a client ID and client secret so that that the FDW can login via OAuth and use the REST API.
2. [Install Multicorn](http://multicorn.org/#installation)
3. [Build and install YAJL](http://lloyd.github.com/yajl/)
4. [Build and install yajl-py](http://pykler.github.com/yajl-py/)
5. Build the FDW module:

        $ cd Database.com-FDW-for-PostgreSQL
        $ python setup.py sdist
        $ sudo python setup.py install

    or, with easy_install:

        $ cd Database.com-FDW-for-PostgreSQL
        $ sudo easy_install .

6. In the PostgreSQL client, create an extension and foreign server:


        CREATE EXTENSION multicorn;
        CREATE SERVER multicorn_force FOREIGN DATA WRAPPER multicorn
        OPTIONS (
            wrapper 'forcefdw.DatabaseDotComForeignDataWrapper',
            api_version 'v35.0',
            login_server 'https://login.salesforce.com'
        );

Default version is v23.0 and default login server is `https://login.salesforce.com`

7. Create a foreign table. You can use any subset of fields from the sobject, and column/field name matching is not case-sensitive:

        CREATE FOREIGN TABLE contacts (
            firstname character varying,
            lastname character varying,
            email character varying
        ) SERVER multicorn_force OPTIONS (
            obj_type 'Contact',
            client_id 'CONSUMER_KEY_FROM_REMOTE_ACCESS_APP,
            client_secret 'CONSUMER_SECRET_FROM_REMOTE_ACCESS_APP',
            username 'user@domain.com',
            password '********'
        );

8. Query the foreign table as if it were any other table. You will see some diagnostics as the FDW interacts with database.com/Force.com. Here are some examples:

    `SELECT *`

        SELECT * FROM contacts;
        NOTICE:  Logged in to https://login.salesforce.com as pat@superpat.com
        NOTICE:  SOQL query is SELECT lastname,email,firstname FROM Contact
         firstname |              lastname               |           email           
        -----------+-------------------------------------+---------------------------
         Rose      | Gonzalez                            | rose@edge.com
         Sean      | Forbes                              | sean@edge.com
         Jack      | Rogers                              | jrogers@burlington.com
         Pat       | Stumuller                           | pat@pyramid.net
         Andy      | Young                               | a_young@dickenson.com
         Tim       | Barr                                | barr_tim@grandhotels.com
        ...etc...

    `SELECT` a column with a condition

        postgres=# SELECT email FROM contacts WHERE lastname LIKE 'G%';
        NOTICE:  SOQL query is SELECT lastname,email FROM Contact WHERE lastname LIKE 'G%'
               email       
        -------------------
         rose@edge.com
         jane_gray@uoa.edu
         agreen@uog.com
        (3 rows)

    Aggregator

        postgres=# SELECT COUNT(*) FROM contacts WHERE lastname LIKE 'G%';
        NOTICE:  SOQL query is SELECT lastname,email,firstname FROM Contact WHERE lastname LIKE 'G%'
         count
        -------
             3
        (1 row)s

    `JOIN`

        postgres=# CREATE TABLE example (
        postgres(#     email varchar PRIMARY KEY,
        postgres(#     favorite_color varchar NOT NULL
        postgres(# );
        NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "example_pkey" for table "example"
        CREATE TABLE
        postgres=# INSERT INTO example VALUES('rose@edge.com', 'Red');
        INSERT 0 1
        postgres=# INSERT INTO example VALUES('jane_gray@uoa.edu', 'Green');
        INSERT 0 1
        postgres=# INSERT INTO example VALUES('agreen@uog.com', 'Blue');
        INSERT 0 1
        postgres=# SELECT favorite_color FROM example JOIN contacts ON example.email=contacts.email;
        NOTICE:  SOQL query is SELECT lastname,email,firstname FROM Contact
         favorite_color
        ----------------
         Red
         Green
         Blue
        (3 rows)
        postgres=# SELECT favorite_color FROM example JOIN contacts ON example.email=contacts.email WHERE contacts.firstname = 'Rose';
        NOTICE:  SOQL query is SELECT lastname,email,firstname FROM Contact WHERE firstname = 'Rose'
         favorite_color
        ----------------
         Red
        (1 row)

    Token refresh

        postgres=# SELECT DISTINCT email FROM contacts LIMIT 1;
        NOTICE:  SOQL query is SELECT email FROM Contact
        NOTICE:  Invalid token 00D50000000IZ3Z!AQ0AQBwEiMxpN5VhLER2PKlifISWxln8ztl2V0cw3BPUAf3IxiD6ZG8Ei5PBcJoCKHDZRmp8lGnFDPQl7kaYgKL73vHHkqbG - trying refresh
        NOTICE:  Logged in to https://login.salesforce.com as pat@superpat.com
        NOTICE:  SOQL query is SELECT email FROM Contact
                 email          
        ------------------------
         jrogers@burlington.com
        (1 row)

    `EXPLAIN`

        postgres=# EXPLAIN ANALYZE SELECT * FROM contacts ORDER BY lastname ASC LIMIT 3;
        NOTICE:  SOQL query is SELECT lastname,email,firstname FROM Contact
                                                                   QUERY PLAN                                                           
        --------------------------------------------------------------------------------------------------------------------------------
         Limit  (cost=129263.11..129263.12 rows=3 width=96) (actual time=431.883..431.887 rows=3 loops=1)
           ->  Sort  (cost=129263.11..154263.11 rows=9999999 width=96) (actual time=431.880..431.880 rows=3 loops=1)
                 Sort Key: lastname
                 Sort Method: top-N heapsort  Memory: 17kB
                 ->  Foreign Scan on contacts  (cost=10.00..15.00 rows=9999999 width=96) (actual time=429.914..431.726 rows=69 loops=1)
                       Foreign multicorn: multicorn
                       Foreign multicorn cost: 10
         Total runtime: 431.941 ms
        (8 rows)

License
-------

3-clause BSD. See license file.
