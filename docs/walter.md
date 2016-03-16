---
layout: page
title: Walter - Docs
---
# Walter

Walter has one configuration file, which specifies a set of tasks needed to build or deploy target an application or service.
More specifically, the configuration file specifies the order of the tasks for the deployment.

In Walter, each task is called as **stage**, and the overall flow is called a **pipeline**.

## Pipelines

Walter's configuration file are written in yaml format. The file needs to contain at least one pipeline block that has at least one stage element.

The following is a sample configuration for Walter.

```yaml
pipeline:
  - name: command_stage_1
    type: command
    command: echo "hello, world"
  - name: command_stage_2
    type: command
    command: echo "hello, world, command_stage_2"
  - name: command_stage_3
    type: command
    command: echo "hello, world, command_stage_3"
```

As we can see, the pipeline block has three stages of type _command_, each of which executes the **echo** command.
Each stage is named (eg **command_stage_1**). You can arbitrarily name each stage.
The commands are executed in the order they are listed in the pipeline configuration.

### Stages

Each stage in a Walter pipeline has three elements: **name**, **type** and **parameters**.
A stage's parameters depend on the stage's type.
For example, a stage of type _command_ can include the parameter **command**, which specifies the shell command to run when the stage is executed.
The following tables describe each stage type and its parameters.

#### Command stage

The Command stage executes one command. This is enabled by setting the stage type to **command**.

The following are valid parameters for the Command stage.

|  Parameter Name   |   Optional   | Description                                                                 |
|:-----------------:|:------------:|:----------------------------------------------------------------------------|
|   command         | false        | shell command to run in the stage                                           |
|   only_if         | true         | run specified command only when the condition following only_if is true     |
|   directory       | true         | the directory where walter runs the specified command                       |

#### Shell script stage

The Shell script stage executes the specified shell script file. This is enabled by setting the stage type to **shell**.

The following are valid parameters for the Shell script stage.

|  Parameter Name   | Optional   | Description                                   |
|:-----------------:|:----------:|:---------------------------------------------:|
|   file            | false      | the shell script file to run in the stage     |

## Parallel stages

You can define child stages, and then run these stages in parallel like this.

```yaml
pipeline:
  - name: parallel stages
    parallel:
      - name: parallel command 1
        type: command
        command: parallel command 1
      - name: parallel command 2
        type: command
        command: parallel command 2
      - name: parallel command 3
        type: command
        command: parallel command 3
```

For the above pipeline, `parallel command 1`, `parallel command 2` and `parallel command 3` will be executed in parallel when the stage 'parallel stage' executes.

## Import predefined stages

Walter provides the facility to import predefined stages from other files.
To import stages to a pipeline configuration file, use the **require** block and
add the list of file names to the block.

For example, the following example imports the stages defined in **conf/mystage.yml**

```yaml
require:
    - conf/mystages.yml

pipeline:
  - call: mypackage::hello
  - call: mypackage::goodbye

```

In the above configuration, the stages ("mypackage::hello" and "mypackage::goodbye"), which are defined in "mystage.yml", are specified.

The files specified in pipeline configuration file need to have two blocks: **namespace** and **stages**.
In **namespace**, we add the package name. This is required to avoid collisions of stage names in multiple required files.
The **stages** block contains the list of stage definitions. These are specified in the same way as the stages in a pipeline configuration.

For example, the following is the contents of conf/mystages.yml referred to in the above pipeline configuration file.

```yaml
namespace: mypackage

stages:
  - def:
      name: hello
      command: echo "May I help you majesty!"
  - def:
      name: goodbye
      command: echo "Goobye majesty."
```

As we can see, stages **hello** and **goodbye** are defined in the file, and the file declares the namespace **mypackage**.

## Cleanup pipeline

Each Walter configuration can have one **cleanup** block; cleanup is another pipeline which will always be executed after a pipeline has either failed or passed.
In the cleanup block, we can add command or shell script stages.
The example below creates a log file in the pipeline, and then removes the log file in the cleanup step.


```yaml
pipeline:
  - name: start pipeline
    command: echo “pipeline” > log/log.txt
cleanup:
  - name: cleanup
    command:  rm log/*
```


## Reporting

Walter supports the submission of progress messages to external messaging services.

### Report configuration

To submit a message, you need to add a **messenger** block to the configuration file.
The following is a sample of a yaml block used to push messages to HipChat.

```yaml
messenger:
  type: hipchat2
  base_url: BASE_URL
  room_id: ROOM_ID
  token: TOKEN
  from: USER_NAME
```

To publish the full output of a stage's execution to the specified messenger service,
add the **report_full_output** attribute with the value **true** to the stage.

```yaml
pipeline:
  - name: command_stage_1
    type: command
    command: echo "hello, world"
    report_full_output: true # If you include this option, the messenger sends the command output "hello, world"
  - name: command_stage_2
    type: command
    command: echo "hello, world, command_stage_2"
    # By default, report_full_output is false
```

### Report types

Walter supports HipChat API v1 and v2, and Slack messenger types.

|  Messenger Type  | Description                                                                                   |
|:----------------:|:-----------------------------------------------------------------------------------------:|
|   hipchat        |  [HipChat (API v1)](https://www.hipchat.com/docs/api)                                     |
|   hipchat2       |  [HipChat (API v2)](https://www.hipchat.com/docs/apiv2)                                   |
|   slack          |  [Slack Incoming Webhook integration](https://my.slack.com/services/new/incoming-webhook) |

### Report configuration

To activate the report function, we need to configure the relveant properties of each messenger type.

#### hipchat and hipchat2

|  Property Name   | Description                                                                                   |
|:----------------:|:----------------------------------|
|   room_id        |  Room name                        |
|   token          |  HipChat token                    |
|   from           |  Account name                     |
|   base_url       |  Base API URL (for private servers; hipchat2 only) |

#### slack

|  Property Name   | Description                                                                                   |
|:----------------:|:----------------------------------|
|   channel        |  Channel name                     |
|   username       |  User name                        |

## Service coordination

Walter provides a coordination function to the project hosting service GitHub.
Specifically, this function provides two roles:

- Check if a new commit or pull request has occured for a repository
- Run the pipeline for the latest commit and pull request if it is newer than the last update

### Service configuration

To activate the service coordination function, we add a "service" block to the Walter configuration file.
The "service" block contains the following elements.

```yaml
service:
  type: github
  token: ADD_YOUR_KEY
  repo: YOUR_REPOSITORY_NAME
  from: YOUR_ACCOUNT_OR_GROUP_NAME
  update: UPDATE_FILE_NAME
```

The following table describes each element.

|  Element  | description                                                                                                                          |
|:---------:|:-------------------------------------------------------------------------------------------------------------------------------------|
|   type    |  Service type (currently Walter supports github only)                                                                                |
|   token   |  [GitHub token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/)                                     |
|   repo    |  Repository name                                                                                                                     |
|   from    |  Account or organization name (if the repository is owned by an organization)                                                           |
|   update  |  Update file which contains the result and time of the last execution  (default **.walter**)                                         |
|   branch  |  Branch name pattern (When this value is supplied, Walter only checks the branches whose name matches the supplied regex pattern)  |

## Environment Variables

You can use environment variables in your Walter configuration files.
Each environment variable name is replaced with the its value from the environment.
Environment variables in Walter configuration files are valuable when you need to use sensitive information
such as messenger service tokens or passwords for external systems.

The following is the format for environment variables used in Walter configuration files.

```
$ENV_NAME
```

We write the envrionment variable into **ENV_NAME**. The following configuration file specify the GitHub Token by embedding the environment variable, GITHUB_TOKEN.

```yaml
service:
  type: github
  token: $GITHUB_TOKEN
  repo: my-service-repository
  from: service-group
  update: .walter
```

## Reusing stage results

Walter stores the results of preceding stages. Subsequent stages can make use of the results of finished stages by using the
four special variables (__OUT, __ERR, __COMBINED and __RESULT) in Walter configuration files.

* **__OUT** -  output flushed to standard output
* **__ERR** - output flushed to standard error
* **__COMBINED** - combined output of stdout and stderr
* **__RESULT** - execution result (true or false)

The four variables are maps whose keys are stage names and whose values are the results of a particular stage.
For example, we can refer to the standard output result of a stage named "stage1" as __OUT["stage1"].

The following is a sample configuration that references one of these special variables.

```yaml
pipeline:
  - name: stage_1
    command: echo "hello world"
  - name: stage_2
    command: echo __OUT["stage_1"]
```

With the above configuration, Walter will output "hello world" twice, since the second stage (stage_2) echoes the standard output result of first stage (stage_1).

## Waiting for preconditions

Typically, a Walter stage starts imidiately after the previous stage finishes.
However, some stages might need to wait for a particular condition, such as a port being ready or the existance of a file.
Walter's ```wait_for``` feature provides support for preconditions which need to be ready before a stage begins.

wait_for is defined as a property of stage.

```yaml
pipeline:
  - name: launch solr
    command: bin/solr start
  - name: post data to solr index
    command: bin/post -d ~/tmp/foobar.js
    wait_for: host=localhost port=8983 state=ready
```

The **wait_for** property takes **key** **value** pairs.
The key has several variations, and the value depends on the key.
The following table describes the supported key value pairs.


| Key     | Value (value type)  | Description                                         |
|:--------|:--------------------|:----------------------------------------------------|
| delay   | second (float)      | The number of seconds to wait after the previous stage finishes |
| port    | port number (int)   | A port number |
| file    | file name (string)  | A file name          |
| host    | host (string)       | An IP address or host name                 |
| state   | state of the other key (string) | Four types of states are supported. The value is dependent on the other Keys. |

State values
--------------

There are several **state** values that depend on which other keys are used.

| State value       | Description                                         |
|:------------------|:----------------------------------------------------|
| present / ready   | The named file is present, or the specified port is ready   |
| absent  / unready | The name file is absent, or the port is not active   |


# Server & Agent

## What is this ?

* Store and display results from walter.
* Provide webhook endpoint and enqueue jobs into walter-agent (walter-agent does not exist yet).

----

## API

### POST /api/v1/reports

Report a build result.

#### request

```json
POST /api/v1/reports
Content-Type: application/json

{
  "project": {
    "name": "walter-server",
    "repo": "https://github.com/walter-cd/walter-server"
  },
  "status": "fail",
  "branch": "master",
  "commits": [
    {
      "revision": "xxxxxxxxx",
      "author": "mizzy",
      "message": "xxxxxxx"
    },
    {
      "revision": "xxxxxxxxx",
      "author": "mizzy",
      "message": "xxxxxx"
    }
  ],
  "stages": [
    {
      "name": "stage 1",
      "status": "success",
      "out": "foo",
      "err": "bar",
      "start": 1449062903,
      "end": 1449062940,
    },
    {
      "name": "stage 2",
      "status": "success",
      "out": "foo",
      "err": "bar",
      "start": 1449062903,
      "end": 1449062940,
      "stages": [
        {
          "name": "child stage 1",
          "status": "fail",
          "out": "foo",
          "err": "bar",
          "start": 1449062903,
          "end": 1449062940,
        }
      ]
    }
  ],
  "compareUrl": "https://xxxxxx/", # Optional
  "start": 1449062903,
  "end": 1449062940,
  "triggeredBy": {
    "name": "mizzy",
    "url":  "https://github.com/mizzy",
    "avatarUrl": "https://avatars1.githubusercontent.com/u/3620",
  } # Optional
}
```

#### response

```json
  Status: 201
  Location:
```

----

### GET /api/v1/reports?count=XXXXX&until=YYYYY

#### Request

```
GET /api/v1/reports?count=XXXXX&until=YYYYY
```

| Parameter | Description |
|-----------|-------------|
|count | API returns this number of reports at most.  |
|since|  API returns reports that start time of which are after this parameter (Unix time by seconds) . |
|until| API returns reports that start time of which are before this parameter (Unix time by seconds) . |
|status| API returns reports that the status of which are this parameter(Passed, Failed, Running or Pending) . |
|projectId | API returns reports of the project with this project id. |

#### Reponse

```json
{
  "Reports": [
    {
      "Id": 2,
      "Project": {
        "Name": "walter-cd/walter",
        "Repo": "https://github.com/walter-cd/walter",
      },
      "Status": "Failed",

      "Branch": "alpha",
      "Commits": [
        {
          "Revision": "8ecb42",
          "Author": "mizzy",
          "Message": "Some changes"
        },
      ],
      "Stages": [
        {
          "Name": "command_stage_1",
          "Status": "Passed",
          "Log": "12:24:00  INFO Pipeline file path: \"./pipeline.yml\"\\n12:24:00  WARN failed to read the configuration file\\n12:24:00  INFO kettle places on heating element\\n12:24:00  INFO a bit of steam visible\\n12:24:00  WARN getting a little hot now\\n12:24:00  WARN loud whistling sounds\\n12:24:00 ERROR open ./pipeline.yml: no such file or directory\\n12:24:00 ERROR failed to create Walter\\n",
          "Stages": null,
          "Start": 1450268347,
          "End": 1450269085
        },
      ],
      "CompareUrl": "http://compare.url/",
      "Start": 1450268347,
      "End": 1450269106,
      "TriggeredBy": {
        "Name": "gerryhocks",
        "Url": "https://github.com/gerryhocks",
        "AvatarUrl": "https://avatars3.githubusercontent.com/u/1311758?v=3&s=460"
      }
    },
    {
      "Id": 1,
      "Project": {
        "Name": "walter-cd/walter-server",
        "Repo": "https://github.com/walter-cd/walter-server"
      },
      "Status": "Passed",
      "Branch": "alpha",
      "Commits": [
        {
          "Revision": "6a3c35",
          "Author": "gerryhocks",
          "Message": "Updated documentation"
        },
      ],
      "Stages": [
        {
          "Name": "command_stage_1",
          "Status": "Passed",
          "Log": "12:24:00  INFO Pipeline file path: \"./pipeline.yml\"\\n12:24:00  WARN failed to read the configuration file\\n12:24:00  INFO kettle places on heating element\\n12:24:00  INFO a bit of steam visible\\n12:24:00  WARN getting a little hot now\\n12:24:00  WARN loud whistling sounds\\n12:24:00 ERROR open ./pipeline.yml: no such file or directory\\n12:24:00 ERROR failed to create Walter\\n",
          "Stages": null,
          "Start": 1450247583,
          "End": 1450248050
        },
      ],
      "CompareUrl": "http://compare.url/",
      "Start": 1450247583,
      "End": 1450248205,
      "TriggeredBy": {
        "Name": "mizzy",
        "Url": "https://github.com/mizzy",
        "AvatarUrl": "https://avatars0.githubusercontent.com/u/3620?v=3&s=460"
      }
    }
  ],
  "NextStart": 1440247583
}
```

---

### GET /api/v1/reports/:reportId:

#### Request

```
GET /api/v1/reports/1
```

#### Reponse

```json
{
  "Report": {
    "Id": 1,
    "Project": {
      "Id": 1,
      "Name": "walter-cd/walter",
      "Repo": "https://github.com/walter-cd/walter"
    },
    "Status": "Passed",
    "Branch": "alpha",
    "Commits": [
      {
        "Revision": "3f54ea",
        "Author": "cmoen",
        "Message": "Fixing previous errors",
        "Url": ""
      },
      {
        "Revision": "3308c8",
        "Author": "gerryhocks",
        "Message": "Fixing previous errors",
        "Url": ""
      }
    ],
    "Stages": [
      {
        "Name": "command_stage_1",
        "Status": "Passed",
        "Log": "12:24:00  INFO Pipeline file path: \"./pipeline.yml\"\\n12:24:00  WARN failed to read the configuration file\\n12:24:00  INFO kettle places on heating element\\n12:24:00  INFO a bit of steam visible\\n12:24:00  WARN getting a little hot now\\n12:24:00  WARN loud whistling sounds\\n12:24:00 ERROR open ./pipeline.yml: no such file or directory\\n12:24:00 ERROR failed to create Walter\\n",
        "Start": 1452688790,
        "End": 1452688960
      },
      {
        "Name": "build_thing",
        "Status": "Passed",
        "Log": "12:24:00  INFO this info message is to test the INFO message level\"\\n",
        "Stages": [
          {
            "Name": "build_thing_substage_1",
            "Status": "Passed",
            "Log": "bash: foobar: command not found...",
            "Start": 1452688960,
            "End": 1452689047
          }
        ],
        "Start": 1452688960,
        "End": 1452689047
      },
      {
        "Name": "package_product",
        "Status": "Pending",
        "Log": "",
        "Start": 0,
        "End": 0
      }
    ],
    "CompareUrl": "http://compare.url/",
    "Start": 1452688790,
    "End": 1452689047,
    "TriggeredBy": {
      "Name": "takahi-i",
      "Url": "https://github.com/takahi-i",
      "AvatarUrl": "https://avatars2.githubusercontent.com/u/339436?v=3&s=460"
    }
  }
}

```

## Web GUI

## Example URLS

* `http://localhost:8080`
  * Show default UI interface with the Activity tab focussed
* `http://localhost:8080?projects`
  * Show the Projects tab
* `http://localhost:8080?project=walter-cd/walter`
  * Show the build history for a specific project
* `http://localhost:8080?project=walter-cd/walter&report=102`
  * Show the build history for a specific project
