.. _peds-client:

###############################################
USPTO Patent Examination Data System API client
###############################################

*****
About
*****
The USPTO Patent Examination Data System (PEDS) contains bibliographic, published document and patent term extension data
in Public PAIR from 1981 to present. There is also some data dating back to 1935.

The PEDS system provides additional information concerning the transaction activity that has occurred for each patent.
The transaction history includes the transaction date, transaction code and transaction description for each transaction activity.

Please also refer to the `API documentation`_ of this service at USPTO.

.. _API documentation: https://ped.uspto.gov/peds/#/apiDocumentation


************
Introduction
************
This software library implements a client to access the PEDS API through the one-stop
``UsptoPatentExaminationDataSystemClient`` class and also provides a respective command line program ``uspto-peds``.

Both can be used for searching and for downloading packages of bundled artefacts.


********
Synopsis
********

Search interface
================
Simple usage::

    from uspto.peds.client import UsptoPatentExaminationDataSystemClient
    client = UsptoPatentExaminationDataSystemClient()

    expression = 'firstNamedApplicant:(nasa)'
    result     = client.search(expression)

Advanced usage::

    expression = 'firstNamedApplicant:(*grohe*)'
    filter     = 'appFilingDate:[2000-01-01T00:00:00Z TO 2015-12-31T23:59:59Z]'
    result     = client.search(expression, filter=filter, sort='applId asc', start=0, rows=20)

The query syntax follows the standard `Apache Solr`_ search syntax,
the JSON documents returned also follow the Solr response formats.

Please note the maximum number of records available per request is limited to 20,
so you have to use the argument options ``start`` and ``rows`` to iterate
through result pages.

.. _Apache Solr: https://lucene.apache.org/solr/


Download interface
==================
The download interface uses the search interface and adds automation for
requesting and downloading package bundles for search results as outlined
in the »API Tutorial« section of the API documentation.
It has two different modes, synchronous and asynchronous.


Synchronous mode
----------------
Simple usage::

    from uspto.peds.client import UsptoPatentExaminationDataSystemClient
    client = UsptoPatentExaminationDataSystemClient()

    # Download document by document number
    # Automatically guesses the document type (application, publication, patent) from the document number schema
    # Will acquire both XML and JSON formats
    result = client.download_document('PP28532')

Advanced usage::

    # Explicitly request a document by application number, acquire XML format only
    result = client.download_document(type='application', number='15431686', format='xml')

    # Download published application by early publication number, acquire JSON format only
    result = client.download_document(type='publication', number='US20170293197A1', format='json')

    # Download granted patent by patent number, acquire both XML and JSON formats
    result = client.download_document(type='patent', number='PP28532', format=['xml', 'json'])


Asynchronous mode
-----------------
The software can also operate asynchronously by using Celery_
as a task queue for scheduling downloads in the background.
Please refer to the `taskqueue documentation`_.

To make the background task scheduler operate it in a simple manner, just start Redis_ and Celery_ like that::

    redis-server
    celery worker --app uspto.tasks --loglevel=info

See also: `taskqueue documentation`_.

.. _Redis: https://redis.io/
.. _Celery: https://celery.readthedocs.io/
.. _taskqueue documentation: taskqueue.rst


Simple usage::

    from uspto.peds.tasks import UsptoPatentExaminationDataSystemDownloader
    downloader = UsptoPatentExaminationDataSystemDownloader()

    # Start downloading single document
    # Automatically guesses the document type
    # Will acquire both XML and JSON formats
    downloader.run('PP28532')

    # Start downloading multiple documents, with document type autoguessing and dual format acquisition
    downloader.run(['PP28532', 'US20170293197A1'])

    # Wait until results arrived
    result = downloader.poll()

Advanced usage::

    # Start downloading single document, explicitly requesting a patent document in XML format
    downloader.run({'type': 'patent', 'number': 'PP28532', 'format': 'xml'})

    # Start downloading multiple documents, each with both XML and JSON formats
    downloader.run([{'type': 'publication', 'number': 'US20170293197A1'}, {'type': 'patent', 'number': 'PP28532'}])

    # Save multiple documents to designated directory using the application identifier as filename
    # The first document should be loaded in XML format, the second one in both XML and JSON formats
    job = downloader.run(
        [
            {'type': 'publication', 'number': 'US20170293197A1', 'format': 'xml'},
            {'type': 'patent',      'number': 'PP28532',         'format': ['xml', 'json']},
        ],
        options = {
            'save': True,
            'directory': './tmp',
            'overwrite': False,
            'use-application-id': True,
        }
    )

    # Is the job ready?
    job.ready()
    False

    # Iterate the results after finishing
    results = list(map(lambda task: task.result, job.results))


Utilities
---------
The ``UsptoPatentExaminationDataSystemDocument`` class can be used to inquire information about the downloaded document::

    from uspto.peds.client import UsptoPatentExaminationDataSystemClient
    from uspto.peds.document import UsptoPatentExaminationDataSystemDocument
    client = UsptoPatentExaminationDataSystemClient()

    # Download document
    result = client.download_document('PP28532')

    # Get document identifiers
    document = UsptoPatentExaminationDataSystemDocument(result)
    document.get_identifiers()
    {'patent': u'PP28532', 'application': u'14999644'}

Another example::

    UsptoPatentExaminationDataSystemDocument(client.download_document('US20170293197A1')).get_identifiers()
    {'application': u'15431686', 'publication': u'US20170293197A1'}


Command line
============
::

    $ uspto-peds --help

    Usage:
      uspto-peds get  <document-number> --format=xml [--type=publication] [--pretty] [--background] [--wait] [--debug]
      uspto-peds save <document-number> --format=xml [--type=publication] [--pretty] [--directory=/var/spool/uspto] [--use-application-id] [--overwrite] [--background] [--wait] [--debug]
      uspto-peds bulk get  --numberfile=numbers.txt --format=xml,json [--pretty] [--use-application-id] [--wait] [--debug]
      uspto-peds bulk save --numberfile=numbers.txt --format=xml,json [--pretty] --directory=/var/spool/uspto [--use-application-id] [--overwrite] [--wait] [--debug]
      uspto-peds search [<expression>] [--filter=filter] [--start=0] [--rows=20] [--download] [--format=xml,json] [--directory=/var/spool/uspto] [--debug]
      uspto-peds info
      uspto-peds --version
      uspto-peds (-h | --help)

    Document acquisition options:
      <document-number>         Document number, e.g. 2017/0293197, US20170293197A1, PP28532, 15431686.
                                Format depends on data source.
      --type=<type>             Document type, one of "publication", "application", "patent" or "auto".
                                When using "auto", the program tries to to guess the document number type
                                (application, publication, patent) from the document number itself.
      --format=<target>         Data format, one of "xml" or "json".
                                In bulk mode, it can also be "--type=xml,json".

    Search options:
      <expression>              Search expression for generic querying.
                                Examples:

                                - firstNamedApplicant:(nasa)
                                - patentTitle:(network AND security) AND appStatus_txt:(patented)
                                - appCls:(701) AND appStatus_txt:(patented)

      --filter=<filter>         Filter expression.
                                Example:

                                - appFilingDate:[2000-01-01T00:00:00Z TO 2005-12-31T23:59:59Z]

      --start=<start>           Start record. Default: 0
      --rows=<rows>             Number of records returned. Default: 20 (which is also the limit).


    Output options:
      --pretty                  Pretty-print output data. This currently applies to "--format=json" only.

    Save options:
      --directory=<directory>   Save downloaded documents to designated target directory.
      --use-application-id      Use the application identifier as filename.
      --overwrite               Overwrite already existing documents.

    Background mode:
      --background              Run the download process in the background.
      --wait                    Wait for the background download job to finish.

    Bulk options:
      --numberfile=<numberfile> Read document numbers from file. Implicitly uses "--background" mode.
                                Guess document number type by implicitly using "--type=auto".
                                Download multiple formats by specifying "--format=xml,json".

    Miscellaneous options:
      --debug                   Enable debug messages
      --version                 Show version information
      -h --help                 Show this screen


    Operation modes:

        "uspto-peds get"             Download one document and print the result to STDOUT.

        "uspto-peds save"            Download one document and save it to the target directory,
                                     defaulting to the current working directory.


        "uspto-peds bulk get"        Submit task for downloading multiple documents to the background job machinery.
                                     After finishing, print the results to STDOUT when using the "--wait" option.

        "uspto-peds bulk save"       Submit task for downloading multiple documents to the background job machinery.
                                     While doing so, progressively save documents to the target directory.
                                     After finishing, print the full file names to STDOUT when using the "--wait" option.


    Examples:

        # Display published application by publication number in XML format
        uspto-peds get "US20170293197A1" --type=publication --format=xml

        # ... same in JSON format, with pretty-printing
        uspto-peds get "US20170293197A1" --type=publication --format=json --pretty

        # Display published application by application number
        uspto-peds get "15431686" --type=application --format=xml

        # Display granted patent by patent number
        uspto-peds get "PP28532" --type=patent --format=xml

        # Display granted patent by automatically guessing document type
        uspto-peds get "PP28532" --format=xml

        # Download granted patent by patent number and save to /var/spool/uspto/PP28532.peds.xml
        uspto-peds save "PP28532" --type=patent --format=xml --directory=/var/spool/uspto

    Bulk example:

        # Download all documents from numbers.txt and save them /var/spool/uspto/$number.peds.(xml|json)
        uspto-peds bulk save --numberfile=numbers.txt --format=xml,json --pretty --directory=/var/spool/uspto --wait

    Search examples:

        # Search for documents matching "applicant=nasa" and display polished JSON response
        uspto-peds search 'firstNamedApplicant:(nasa)'

        # Search for documents matching "applicant=grohe" filed between 2010 and 2017
        uspto-peds search 'firstNamedApplicant:(*grohe*)' --filter='appFilingDate:[2010-01-01T00:00:00Z TO 2017-12-31T23:59:59Z]'

        # Search for documents matching "applicant=nasa" and download zip archives containing bundles in XML and JSON formats
        uspto-peds search 'firstNamedApplicant:(nasa)' --download --format=xml,json --directory=/tmp

