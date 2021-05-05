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
	 count  
	--------
 	 278429
	```
    1. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
  	```
	| hll_count |  url  | hostpathquery | hostpath | host  |      insert_hour      | 
	|-----------|-------|---------------|----------|-------|------------------------|
	|	 6 | 32119 |         33960 |    23988 |     6 | 2021-05-05 14:00:00+00|
	|	 7 | 76966 |         77933 |    65045 | 15545 | 2021-05-05 13:00:00+00|
	|	 7 | 41920 |         40758 |    34071 | 11333 | 2021-05-05 12:00:00+00|
	|	 4 | 82351 |         82422 |    80674 | 72990 | 2021-05-04 02:00:00+00|
	|	 4 | 39144 |         39281 |    37117 | 33587 | 2021-05-04 01:00:00+00|
	
	
    1. This query shows the hostnames that you have downloaded the most webpages from:
    
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
       ```
 	| url  | hostpathquery | hostpath |            host            |
	|-------|---------------|----------|---------------------------|
	| 13350 |         13475 |    13363 | com,apnews)|
	| 13355 |         13601 |    13345 | com,pinterest)|
	| 12375 |         12354 |    12199 | com,fivethirtyeight)|
	| 10941 |         11189 |    11185 | com,deadspin)|
	| 10769 |         10722 |     6134 | com,nytimes)|
	| 14960 |         15277 |      794 | com,bing)|
	|    82 |            82 |       58 | com,mlb)|
	|    57 |            57 |       57 | org,wikipedia,en)|
	|    54 |            54 |       54 | com,popsugar)|
	|    51 |            51 |       51 | com,agoda)|
	|    81 |            81 |       48 | com,atgstores)|
	|    47 |            47 |       47 | com,stackoverflow)|
	|    46 |            46 |       46 | com,tripadvisor)|
	|   46 |            46 |       46 | com,gamefaqs)|
	|    46 |            46 |       46 | com,society6)|
	|    46 |            46 |       46 | com,dollartree)|
	|    45 |            45 |       45 | edu,cornell,law)|
	|    44 |            44 |       44 | net,sourceforge)|
	|    43 |            43 |       43 | org,worldcat)|
	|    42 |            42 |       42 | jp,tripadvisor)|
	|    42 |            42 |       42 | com,grouprecipes)|
	|    44 |            44 |       41 | org,marylandpublicschools)|
	|    40 |            40 |       40 | com,pandora)|
	|    40 |            40 |       40 | fr,tripadvisor)|
	|    40 |            40 |       40 | com,packersproshop)|
	|    40 |            40 |       40 | com,orientaltrading)|
	|    40 |            40 |       40 | com,theguardian)|
	|    40 |            40 |       40 | com,ajmadison)|
	|    40 |            40 |       40 | com,6pm)|
	|    40 |            40 |       40 | com,vimeo)|
	|    60 |            60 |       39 | com,cnet)|
	|    41 |            41 |       38 | com,go,espn)|
	|    37 |            37 |       37 | com,imdb)|
	|    39 |            39 |       37 | com,barnesandnoble)|
	|    37 |            37 |       37 | com,utsandiego)|
	|    36 |            36 |       36 | net,worldcosplay)|
	|    35 |            35 |       35 | tw,com,tripadvisor)|
	|    35 |            35 |       35 | com,landsend)|
	|    35 |            35 |       35 | com,scribdassets,imgv2-3)|
	|    35 |            35 |       35 | kr,co,tripadvisor)|
	|    39 |            39 |       34 | com,motorsport)|
	|    34 |            34 |       34 | com,washingtontimes)|
	|    34 |            34 |       34 | org,wikipedia,de)|
	|    50 |            50 |       34 | com,bloomberg)|
	|    34 |            34 |       34 | br,com,tripadvisor)|
	|    42 |            42 |       33 | com,snagajob)|
	|    33 |            33 |       33 | org,eol)|
	|    33 |            33 |       33 | com,zappos)|
	|    33 |            33 |       33 | com,scribdassets,imgv2-4)|
	|    33 |            33 |       33 | com,eastbay)|
	|    35 |            35 |       33 | com,walmart)|
	|    33 |            33 |       32 | com,oxforddictionaries)|
	|    32 |            32 |       32 | net,slideshare)|
	|    32 |            32 |       32 | org,apache,mail-archives)|
	|    32 |            32 |       32 | com,upi)|
	|    32 |            32 |       32 | com,aceshowbiz)|
	|    32 |            32 |       32 | com,imgur)|
	|    32 |            32 |       32 | com,modelmayhem)|
	|    32 |            32 |       32 | com,twopeasinabucket)|
	|    32 |            32 |       32 | com,terrysvillage)|
	|    32 |            32 |       32 | com,justjared)|
	|    31 |            31 |       31 | com,oracle,docs)|
	|    32 |            32 |       31 | com,reverbnation)|
	|    34 |            34 |       31 | com,craftsy)|
	|    32 |            32 |       31 | com,dpreview)|
	|    31 |            31 |       31 | com,cnn)|
	|    31 |            31 |       31 | com,bigstockphoto)|
	|    31 |            31 |       31 | com,scribdassets,imgv2-2)|
	|    36 |            36 |       31 | com,ocregister)|
	|    31 |            31 |       31 | id,co,tripadvisor)|
	|    31 |            31 |       31 | com,opticsplanet)|
	|    31 |            31 |       31 | com,cricketarchive)|
	|    33 |            33 |       31 | com,cnbc)|
	|    30 |            30 |       30 | com,tennisplaza)|
	|    30 |            30 |       30 | es,tripadvisor)|
	|    30 |            30 |       30 | se,tripadvisor)|
	|    36 |            36 |       30 | org,summitpost)|
	|    30 |            30 |       30 | gr,com,tripadvisor)|
	|    30 |            30 |       30 | com,gilt)|
	|    30 |            30 |       30 | com,basspro)|
	|    30 |            30 |       30 | de,tripadvisor)|
	|    30 |            30 |       30 | org,wikipedia,es)|
	|    29 |            29 |       29 | ve,com,tripadvisor)|
	|    29 |            29 |       29 | com,pbase)|
	|    35 |            35 |       29 | com,nordstrom,shop)| 
	|    29 |            29 |       29 | com,tmz)|
	|    39 |            39 |       29 | com,musicnotes)|
	|    29 |            29 |       29 | com,flightaware)|
	|    29 |            29 |       29 | ru,tripadvisor)|
	|    29 |            29 |       29 | gov,epa,yosemite)|
	|    28 |            28 |       28 | com,iheart)|
	|    28 |            28 |       28 | cl,tripadvisor)|
	|    30 |            30 |       28 | com,tvguide)|
	|    28 |            28 |       28 | com,grainger)|
	|    28 |            28 |       28 | com,oyster)|
	|    28 |            28 |       28 | com,sporcle)|
	|    28 |            28 |       28 | com,ebaumsworld)|
	|    31 |            31 |       28 | com,cars)|
	|    28 |            28 |       28 | com,dnb)|
	|    31 |            31 |       28 | com,godlikeproductions)|
	    
	```

1. Take a screenshot of an interesting search result.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='covid-ngrams.png' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
