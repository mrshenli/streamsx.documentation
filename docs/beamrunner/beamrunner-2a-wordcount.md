---
layout: docs
title:  WordCount sample application for IBM Streams Runner for Apache Beam
navtitle: WordCount sample app
description:  description
weight:  10
published: true
tag: beam
prev:
  file: beamrunner-2-install
  title: Installing
next:
  file: beamrunner-3-sample
  title: TemperatureSample sample app
---

You can use IBM® Streams Runner for Apache Beam to run the Apache Beam 2.0 Java SDK Quickstart WordCount sample application.

## Prerequisites

Before you run the WordCount sample application, you must must configure and run the following services on IBM Bluemix®:

- Streaming Analytics. For more information, see [Creating a Streaming Analytics service on Bluemix](http://127.0.0.1:4000/streamsx.documentation/docs/beamrunner/beamrunner-2-install/#creating-a-streaming-analytics-service-on-bluemix).
- IBM® Object Storage for Bluemix®. Make sure the environment variables are configured. For more information about the environment variables, see _Setting up the client_ in [Configuring the CLI to use Swift and Cloud Foundry commands](https://console.stage1.bluemix.net/docs/services/ObjectStorage/os_configuring.html).

In addtion, you must set up your development JDK and Maven environment. For more information, see [Set up your Development Environment](https://beam.apache.org/get-started/quickstart-java/#set-up-your-development-environment).

If you want to compile on Bluemix, you must unset the `STREAMS_INSTALL` variable before you submit the job/application/pipeline???.

## I/O options

Streams Runner offers two options to provide the input file, `streams://` and `swift://`. Applications can also use other Beam IO options to link the input.

### Using the `streams://` file system scheme:

An application can include a local file into the bundle by using the `--filesToStage` option. This option take a JSON string where keys are paths of local files to include and values are destination paths in the bundle. For example, with the option `--filesToStage={\"/local/file.txt\":\"data/input\"} --filesToStage={\"/local/\":\"\"}`, the runner copies the `/local/file.txt` file into the bundle under relative path `data/input`. Later, the pipeline can access the file using path `streams://data/input`.

### Using the `swift://` Object Storage file system scheme

The application can upload the input file to Object Storage, and use the path `swift://<container>/<object>` to access the file.

## Running the WordCount sample

1. Get and compile the wordcount code.  
    Note that the `-DarchetypeVersion` variable is set to 2.0.0. Why is this important???

   ```
$ mvn archetype:generate \
          -DarchetypeGroupId=org.apache.beam \
          -DarchetypeArtifactId=beam-sdks-java-maven-archetypes-examples \
          -DarchetypeVersion=2.0.0 \
          -DgroupId=org.example \
          -DartifactId=word-count-beam \
          -Dversion="0.1" \
          -Dpackage=org.apache.beam.examples \
          -DinteractiveMode=false
$ cd word-count-beam
$ mvn package
```

1. Choose one of the following options to run the WordCount sample application.

   - The following command uses `streams://` to provide the input, and writes the output to Object Storage.

    ```
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:target/word-count-beam-0.1.jar \
      org.apache.beam.examples.WordCount \
      --filesToStage="{\"./pom.xml\":\"pom.xml\"}" \
      --inputFile=streams://pom.xml \
      --output=swift://beam-container/quickstart.out- \
      --runner=StreamsRunner \
      --jarsToStage=target/original-word-count-beam-0.1.jar:$STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.sdk.jar \
      --contextType=STREAMING_ANALYTICS_SERVICE \
      --vcapServices=/path/to/vcap/file \
      --serviceName=yourSasName
```

  - The following command uses Object storage to host both input and output files. Make sure that the `pom.xml` input file has been uploaded to `beam-container` before you submit the pipeline. Will the user know what beam-container is???

    ```
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:target/word-count-beam-0.1.jar \
      org.apache.beam.examples.WordCount \
      --inputFile=swift://beam-container/pom.xml \
      --output=swift://beam-container/quickstart.out- \
      --runner=StreamsRunner \
      --jarsToStage=target/original-word-count-beam-0.1.jar:$STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.sdk.jar \
      --contextType=STREAMING_ANALYTICS_SERVICE \
      --vcapServices=/path/to/vcap/file \
      --serviceName=yourSasName
```

1. Inspect the results
    After the pipeline completes, you can download the output by using the Swift API. Each output file contains up to 50,000 lines. If the output goes beyond that limit, it will end up in multiple output files.

    ```
    $ swift download beam-container -p quickstart.out -D swift-output
    $ cat swift-output/*
```