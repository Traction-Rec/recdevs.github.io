---
layout: post
title:  "Start Debug Trace From Shell"
date:   2020-06-03
categories: SFDX
author: john
---

Some of us are trying out VSCode + Salesforce Extension pack as an alternative to Intellij + Illuminated Cloud. VSCode is more responsive than Intellij, and, while IC is more feature-rich, the Salesforce Extension pack gets the job done. I only find myself back in IJ + IC when working on a 1st generation package. 

That being said, when running apex tests VSC+SE pales in comparison to IJ+IC. The biggest features missing are:

1. being able to rerun failed tests
2. being able to easily start a trace & get the log for a failed test

IJ+IC handles these requirements seemlessly. VSC+SE needs a little help. Here's a bash script to start a 10 hour debug trace for your user. Once the trace is started use the Get Debug Logs command within VSCode.

    #!/bin/bash

    # get user ID
    USER_ID=$(sfdx force:user:display | awk '/^Id/ {print $2}')
    echo "user ID $USER_ID retrieved"

    # create debug level
    NEW_UUID=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
    LEVEL_NAME="t_$NEW_UUID"
    echo "creating log level $LEVEL_NAME"
    DEBUG_LEVEL_ID=$(sfdx force:data:record:create -s DebugLevel -t -v "DeveloperName=$LEVEL_NAME MasterLabel=From_CLI ApexCode=DEBUG ApexProfiling=DEBUG Callout=DEBUG Database=Debug System=NONE Validation=DEBUG Visualforce=DEBUG Workflow=DEBUG"\
    | egrep -o '[a-zA-Z0-9]{18}')
    echo "debug level $DEBUG_LEVEL_ID created!"

    # create 10hr trace
    START_TIME=$(date --utc +%FT%T.%3NZ)
    END_TIME=$(date -d '+10 hour' --utc +%FT%T.%3NZ)
    sfdx force:data:record:create -sTraceFlag -t -v "DebugLevelId=$DEBUG_LEVEL_ID TracedEntityId=$USER_ID StartDate=$START_TIME ExpirationDate=$END_TIME LogType=DEVELOPER_LOG ApexCode=DEBUG ApexProfiling=DEBUG Callout=DEBUG Database=Debug System=NONE Validation=DEBUG Visualforce=DEBUG Workflow=DEBUG"
