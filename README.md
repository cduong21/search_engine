# HW: Search Engine

In this assignment you will create a highly scalable web search engine.

**Due Date:** Sunday, 9 May

**Learning Objectives:**
1. Learn to work with a moderate large software project
1. Learn to parallelize data analysis work off the database
1. Learn to work with WARC files and the multi-petabyte common crawl dataset
1. Increase familiarity with indexes and rollup tables for speeding up queries

## Task 0: project setup

1. Fork this github repo, and clone your fork onto the lambda server

1. Ensure that you'll have enough free disk space by:
    1. bring down any running docker containers
    1. run the command
       ```
       $ docker system prune
       ```

## Task 1: getting the system running

In this first task, you will bring up all the docker containers and verify that everything works.

There are three docker-compose files in this repo:
1. `docker-compose.yml` defines the database and pg_bouncer services
1. `docker-compose.override.yml` defines the development flask web app
1. `docker-compose.prod.yml` defines the production flask web app served by nginx

Your tasks are to:

1. Modify the `docker-compose.override.yml` file so that the port exposed by the flask service is different.

1. Run the script `scripts/create_passwords.sh` to generate a new production password for the database.

1. Build and bring up the docker containers.

1. Enable ssh port forwarding so that your local computer can connect to the running flask app.

1. Use firefox on your local computer to connect to the running flask webpage.
   If you've done the previous steps correctly,
   all the buttons on the webpage should work without giving you any error messages,
   but there won't be any data displayed when you search.

1. Run the script
   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   to perform automated checks that the system is running correctly.
   All tests should report `[pass]`.

## Task 2: loading data

There are two services for loading data:
1. `downloader_warc` loads an entire WARC file into the database; typically, this will be about 100,000 urls from many different hosts. 
1. `downloader_host` searches the all WARC entries in either the common crawl or internet archive that match a particular pattern, and adds all of them into the database

### Task 2a

We'll start with the `downloader_warc` service.
There are two important files in this service:
1. `services/downloader_warc/downloader_warc.py` contains the python code that actually does the insertion
1. `downloader_warc.sh` is a bash script that starts up a new docker container connected to the database, then runs the `downloader_warc.py` file inside that container

Next follow these steps:
1. Visit https://commoncrawl.org/the-data/get-started/
1. Find the url of a WARC file.
   On the common crawl website, the paths to WARC files are referenced from the Amazon S3 bucket.
   In order to get a valid HTTP url, you'll need to prepend `https://commoncrawl.s3.amazonaws.com/` to the front of the path.
1. Then, run the command
   ```
   $ ./download_warc.sh $URL
   ```
   where `$URL` is the url to your selected WARC file.
1. Run the command
   ```
   $ docker ps
   ```
   to verify that the docker container is running.
1. Repeat these steps to download at least 5 different WARC files, each from different years.
   Each of these downloads will spawn its own docker container and can happen in parallel.

You can verify that your system is working with the following tasks.
(Note that they are listed in order of how soon you will start seeing results for them.)
1. Running `docker logs` on your `download_warc` containers.
1. Run the query
   ```
   SELECT count(*) FROM metahtml;
   ```
   in psql.
1. Visit your webpage in firefox and verify that search terms are now getting returned.

### Task 2b

The `download_warc` service above downloads many urls quickly, but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, and so their contents will not be reflected in the ngrams graph.
In this task, you will implement and run the `download_host` service for downloading high quality urls.

1. The file `services/downloader_host/downloader_host.py` has 3 `FIXME` statements.
   You will have to complete the code in these statements to make the python script correctly insert WARC records into the database.

   HINT:
   The code will require that you use functions from the cdx_toolkit library.
   You can find the documentation [here](https://pypi.org/project/cdx-toolkit/).
   You can also reference the `download_warc` service for hints,
   since this service accomplishes a similar task.

1. Run the query
   ```
   SELECT * FROM metahtml_test_summary_host;
   ```
   to display all of the hosts for which the metahtml library has test cases proving it is able to extract publication dates.
   Note that the command above lists the hosts in key syntax form, and you'll have to convert the host into standard form.
1. Select 5 hostnames from the list above, then run the command
   ```
   $ ./downloader_host.sh "$HOST/*"
   ```
   to insert the urls from these 5 hostnames.

## ~~Task 3: speeding up the webpage~~

Since everyone seems pretty overworked right now,
I've done this step for you.

There are two steps:
1. create indexes for the fast text search
1. create materialized views for the `count(*)` queries

## Submission

1. Edit this README file with the results of the following queries in psql.
   The results of these queries will be used to determine if you've completed the previous steps correctly.

    1. This query shows the total number of webpages loaded:
       ```
       select count(*) from metahtml;
       ```
       ```
        count  
        --------
        383286
        (1 row)
       ```
       

    1. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
       ```
       ```
        hll_count |  url  | hostpathquery | hostpath | host  |      insert_hour       
        -----------+-------+---------------+----------+-------+------------------------
         1 |  5883 |          5861 |     5861 |     1 | 2021-05-14 00:00:00+00
         2 |  5213 |          5320 |     5277 |     2 | 2021-05-13 23:00:00+00
         1 |  2693 |          2648 |      728 |     2 | 2021-05-13 21:00:00+00
         2 |  3187 |          3309 |     2912 |     4 | 2021-05-13 20:00:00+00
         1 |  6436 |          6735 |     6807 |     1 | 2021-05-13 19:00:00+00
         2 |  3023 |          2980 |     2980 |     2 | 2021-05-13 18:00:00+00
         1 |  2167 |          2207 |     2149 |     1 | 2021-05-13 17:00:00+00
         1 |  2900 |          2780 |     2749 |     1 | 2021-05-13 16:00:00+00
         1 | 44133 |         43045 |    40105 | 29575 | 2021-05-13 09:00:00+00
         1 | 45363 |         44957 |    42975 | 30076 | 2021-05-13 08:00:00+00
         2 | 69078 |         68841 |    65211 | 41194 | 2021-05-13 07:00:00+00
         1 | 18555 |         18999 |    18646 | 15169 | 2021-05-13 06:00:00+00
         2 | 10312 |         10616 |    10283 |  7808 | 2021-05-13 04:00:00+00
         1 | 48258 |         47971 |    47302 | 42635 | 2021-05-13 03:00:00+00
         1 | 31343 |         31604 |    30140 | 28734 | 2021-05-13 02:00:00+00
         2 | 29320 |         29444 |    28756 | 25691 | 2021-05-13 01:00:00+00
         1 | 21948 |         21935 |    21467 | 19685 | 2021-05-13 00:00:00+00
         1 | 24342 |         24382 |    23998 | 22443 | 2021-05-01 03:00:00+00
        (18 rows)
       ```

    1. This query shows the hostnames that you have downloaded the most webpages from:
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
       ```
       ```
         url  | hostpathquery | hostpath |            host            
        -------+---------------+----------+----------------------------
         11623 |         11913 |    11814 | com,bbc)
         10655 |         10419 |    10419 | com,dailyreadlist)
          4859 |          5148 |     4859 | com,coronavirusnewslive)
          3860 |          3739 |     1562 | com,linkedin)
          3360 |          3380 |     2990 | com,wordpress)
            74 |            74 |       74 | com,mobygames)
            71 |            71 |       71 | com,ibegin)
            70 |            70 |       70 | is,bible)
            61 |            61 |       61 | com,uplike)
            60 |            60 |       60 | org,worldcat)
            69 |            69 |       59 | com,google)
            58 |            58 |       58 | org,wikipedia,en)
            58 |            58 |       58 | com,teacherspayteachers)
            55 |            55 |       55 | com,gamershell)
            55 |            55 |       55 | com,foxnews)
            55 |            55 |       55 | de,welt)
            55 |            55 |       55 | ru,mail,torg)
            52 |            52 |       52 | fr,tripadvisor)
            51 |            51 |       51 | com,bleacherreport)
            50 |            50 |       50 | com,imgur)
            49 |            49 |       49 | ru,tripadvisor)
            49 |            49 |       49 | com,beau-coup)
            49 |            49 |       49 | com,grabcad)
            49 |            49 |       49 | com,pbase)
            54 |            54 |       48 | com,indeed)
            47 |            47 |       47 | com,bookyogaretreats)
            47 |            47 |       47 | com,ecrater)
            47 |            47 |       46 | com,economist)
            46 |            46 |       46 | com,tripadvisor)
            46 |            46 |       46 | com,dx)
            51 |            51 |       45 | com,clearancejobs)
            44 |            44 |       44 | com,newgrounds)
            44 |            44 |       44 | com,cricketarchive)
            44 |            44 |       44 | ru,primanota)
            43 |            43 |       43 | org,wikipedia,es)
            43 |            43 |       43 | net,behance)
            43 |            43 |       43 | de,auswaertiges-amt)
            47 |            47 |       42 | com,starizona)
            42 |            42 |       42 | cz,docplayer)
            42 |            42 |       42 | org,ohiomemory)
            42 |            42 |       42 | tr,com,yandex)
            43 |            43 |       42 | net,blogmarks)
            42 |            42 |       42 | ru,elbase)
            41 |            41 |       41 | com,cafepress)
            41 |            41 |       41 | org,repec,ideas)
            41 |            41 |       41 | ru,academic,dic)
            44 |            44 |       41 | com,trekearth)
            40 |            40 |       40 | de,thomann)
            39 |            39 |       39 | uk,co,uk-local-search)
            39 |            39 |       39 | com,modelmayhem)
            39 |            39 |       39 | com,agoda)
            39 |            39 |       39 | com,kohls)
            39 |            39 |       39 | com,aceshowbiz)
            39 |            39 |       39 | com,baltimoresun,articles)
            39 |            39 |       39 | com,tripadvisor,pl)
            48 |            48 |       39 | com,care2)
            38 |            38 |       38 | com,vitals)
            38 |            38 |       38 | com,drownedinsound)
            38 |            38 |       38 | de,mydealz)
            43 |            43 |       38 | com,texasmotorspeedway)
            38 |            38 |       38 | hu,regikonyvek)
            38 |            38 |       38 | com,tampabay)
            70 |            70 |       37 | com,philly)
            38 |            38 |       37 | com,travellerspoint)
            39 |            39 |       37 | com,crestock)
            55 |            55 |       37 | com,tumblr)
            37 |            37 |       37 | ua,okino)
            36 |            36 |       36 | com,gizmodo)
            36 |            36 |       36 | cn,mafengwo)
            36 |            36 |       36 | com,drugs)
            36 |            36 |       36 | com,made-in-china,es)
            36 |            36 |       36 | jp,housecom)
            36 |            36 |       36 | org,wikidata)
            36 |            36 |       36 | com,911tabs)
            36 |            36 |       36 | jp,livedoor,blog)
            36 |            36 |       36 | org,apache,mail-archives)
            36 |            36 |       36 | nl,muziekweb)
            36 |            36 |       36 | com,reuters,lta)
            35 |            35 |       35 | com,pinterest,nl)
            35 |            35 |       35 | com,society6)
            35 |            35 |       35 | com,ibm)
            35 |            35 |       35 | com,aol)
            35 |            35 |       35 | com,funnyjunk)
            35 |            35 |       35 | net,tiexue,bbs)
            35 |            35 |       35 | com,thefreedictionary)
            34 |            34 |       34 | com,tv)
            34 |            34 |       34 | ge,place)
            34 |            34 |       34 | org,wikipedia,fr)
            34 |            34 |       34 | it,docplayer)
            34 |            34 |       34 | com,thestreet)
            34 |            34 |       34 | uk,co,telegraph)
            34 |            34 |       34 | nl,tripadvisor)
            35 |            35 |       34 | com,mcmelectronics)
            34 |            34 |       34 | com,magnumtuning)
            34 |            34 |       34 | com,upi)
            34 |            34 |       34 | fr,lexpress)
            33 |            33 |       33 | com,chron)
            33 |            33 |       33 | au,com,tripadvisor)
            33 |            33 |       33 | com,blogspot,althouse)
            33 |            33 |       33 | com,onthesnow)
        (100 rows)
       ```

1. Take a screenshot of an interesting search result.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='happy.png' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
