### Disclaimer

This is not an official Google product.

# Dockerflow

Dockerflow makes it easy to run a multi-step workflow of Docker tasks using
[Google Cloud Dataflow](https://cloud.google.com/dataflow) for orchestration.
Docker steps are run using the [Pipelines API](https://cloud.google.com/genomics/v1alpha2/pipelines).

You can run Dockerflow from a shell on your laptop, and the job will run in 
Google Cloud Platform using Dataflow's fully managed service and web UI.

Dockerflow workflows can be defined in [YAML](http://yaml.org) files, or by writing
Java code. Examples of workflows defined in YAML can be found in

*   [src/test/resources](src/test/resources)

Examples of workflows defined in Java can be found in

*   [examples](examples)
*   [src/test/java/com/google/cloud/genomics/dockerflow/examples](src/test/java/com/google/cloud/genomics/dockerflow/examples)

You can run a batch of workflows at once by providing a csv file with one row per
workflow to define the parameters.

## Why Dockerflow?

This project was created as a proof-of-concept that Dataflow can be used
for monitoring and management of directed acyclic graphs of command-line tools.

Dataflow and Docker complement each other nicely:

*   Dataflow provides graph optimization, a nice monitoring interface, retries,
    and other niceties.
*   Docker provides portability of the tools themselves, and there's a large
    library of packaged tools already available as Docker images.

While Dockerflow supports a simple YAML workflow definition, a similar approach
could be taken to implement a runner for one of the open standards like [Common
Workflow Language]
(https://github.com/common-workflow-language/common-workflow-language) or
[Workflow Definition Language](github.com/broadinstitute/wdl).

## Table of contents

*   [Prerequisites](#prerequisites)
*   [Getting started](#getting-started)
*   [Docker + Dataflow vs custom scripts](#docker-and-dataflow-vs-custom-scripts)
*   [Creating your own workflows](#creating-your-own-workflows)
    *   [Sequential workflows](#sequential-workflows)
    *   [Parallel workflows](#parallel-workflows)
    *   [Branching workflows](#branching-workflows)
*   [Testing](#testing)
*   [What next](#what-next)

## Prerequisites

1.  Sign up for a Google Cloud Platform account and [create a project]
    (https://console.cloud.google.com/project?).

2.  [Enable the APIs]
    (https://console.cloud.google.com/flows/enableapi?apiid=genomics,dataflow,storage_component,compute_component&redirect=https://console.cloud.google.com)
    for Cloud Dataflow, Google Genomics, Compute Engine and Cloud Storage.
3.  [Install the Google Cloud SDK](https://cloud.google.com/sdk/) and run:

        gcloud init
        gcloud auth login
        gcloud beta auth application-default login
        gcloud components update

## Getting started

Run the following steps on your laptop or local workstation:

1.  git clone this repository.

        git clone https://github.com/googlegenomics/dockerflow

2.  Build it with Maven.

        cd dockerflow
        mvn package -DskipTests

3.  Set up the DOCKERFLOW_HOME environment.

        export DOCKERFLOW_HOME="$(pwd)"
        export PATH="${PATH}":"${DOCKERFLOW_HOME}/bin"
        chmod +x bin/*

4.  Set up your local project environmental variables.

        export PROJECT_NAME=$(gcloud config list project | grep = | cut -d ' ' -f 3)
        export WORKSPACE_BUCKET=gs://MY-BUCKET
        gsutil mb $WORKSPACE_BUCKET
        export WORKSPACE_PATH=$WORKSPACE_BUCKET/MY-PATH

    Set `MY-BUCKET` and `MY-PATH` to your cloud bucket and folder.  A `gsutil mb` command
    is included above to ensure that bucket `MY-BUCKET` exists.  Subdirectory `MY-PATH`
    does not need to exist, as it will be created by running the workflow.

5.  Create a workflow

        echo "
        version: v1alpha2
        defn:
          name: LinearGraph
          description: A simple linear graph.
        args:
          inputs:
            BASE_DIR: $WORKSPACE_PATH
            stepOne.inputFile: \${BASE_DIR}/input-one.txt
            stepTwo.inputFile: \${stepOne.outputFile}
          outputs:
            stepOne.outputFile: output-one.txt
            stepTwo.outputFile: output-two.txt
        steps: 
        - defn:
            name: stepOne
          defnFile: $DOCKERFLOW_HOME/src/test/resources/task-one.yaml
        - defn:
            name: stepTwo
          defnFile: $DOCKERFLOW_HOME/src/test/resources/task-two.yaml
        " > ./linear-graph.yaml

        echo "puppy dog" | gsutil cp - $WORKSPACE_PATH/input-one.txt

    The commands above
    * Create a new file in the current working directory called `linear-graph.yaml`.  It's based on the dockerflow unit test yaml file [here](https://github.com/allenday/dockerflow/blob/master/src/test/resources/linear-graph.yaml).  Note that it references two additional yaml files for steps `stepOne` and `stepTwo`.
    * Create an input file for the workflow.  Contents are `puppy dog`.

6.  Run a sample workflow:

    There are two ways to do this:

    * Locally and blocking:

      ```
      dockerflow --project=$PROJECT_NAME \
        --workflow-file=./linear-graph.yaml \
        --workspace=$WORKSPACE_PATH \
        --runner=DirectPipelineRunner
      ```

      This will not return control to the shell until the job completes.


    * Remotely and non-blocking, with visualization.

      ```
      dockerflow --project=$PROJECT_NAME \
        --workflow-file=./linear-graph.yaml \
        --workspace=$WORKSPACE_PATH
      ```

      Here we simply remove the parameter `--runner=DirectPipelineRunner`.  The effect
      is that the workflow is launched and control is returned to the current shell.
      With the non-blocking version, it's possible to [monitor and visualize Workflow progress](https://console.cloud.google.com/dataflow).


    `$PROJECT_NAME` and `$WORKSPACE_PATH` are defined in step 4 above.

## Docker and Dataflow vs custom scripts

How is Dataflow better than a shell script?

Dataflow provides:

*   **Complex workflow orchestration**: Dataflow supports arbitrary directed
acyclic graphs. The logic of branching, merging, parallelizing, and monitoring is
all handled automatically.
*   **Monitoring**:
[Dataflow's monitoring UI](https://cloud.google.com/dataflow/pipelines/dataflow-monitoring-intf)
shows you what jobs you've run and shows an execution graph with nice details.
*   **Debugging**: Dataflow keeps logs at each step, and you can view them directly
in the UI.
*   **Task retries**: Dataflow automatically retries failed steps.
Dockerflow adds support for preemptible VMs, rerunning failures on standard VM
instances.
*   **Parallelization**: Dataflow can run 100 tasks on 100 files and
keep track of them all for you, retrying any steps that failed.
*   **Optimization**: Dataflow optimizes the execution graph for your workflow.

Docker provides:

*   **Portability**: Tools packaged in Docker images can be run
anywhere Docker is supported.
*   **A library of pre-packaged tools**: The community has contributed a growing
library of popular tools.

## Creating your own workflows

The Dockerflow command-line expects a static workflow graph definition in YAML,
or the Java class name of a Java definition.

If you'd rather define workflows in code, you'll use the Java SDK. See

*   [src/test/java/com/google/cloud/genomics/dockerflow/examples](src/test/java/com/google/cloud/genomics/dockerflow/examples)

Everything that can be done with YAML can also be done (and more compactly) in
Java code. Java provides greater flexibility too.

The documentation below provides details for defining workflows in YAML. To
create a workflow, you define the tasks and execution graph. You can
define the tasks and execution graph in a single file, or your graph can
reference tasks that are defined in separate YAML files.

A workflow is a recursive format, meaning that a workflow can contain multiple
steps, and each of the steps can be a workflow.

### Hello, world

The best way to get started with Dockerflow is to look at a real example.

    defn:
      name: HelloWorkflow
    steps:
    - defn:
        name: Hello
        docker:
          imageName: ubuntu
          cmd: echo “Hello, world”

You can save this as `hello.yaml` and run it as above.

Let's dissect the pieces -- even if there aren't that many. The overall workflow
has a name: `HelloWorkflow`. It contains one step, which runs an `echo` command
in a stock Ubuntu Docker image.

The `defn` section for the workflow `steps` is a superset of the
[Pipeline object](https://cloud.google.com/genomics/reference/rest/v1alpha2/pipelines#Pipeline)
used by the [Pipelines API](https://cloud.google.com/genomics/v1alpha2/pipelines).

### Sequential workflows

The simplest multi-step workflow consists of a series of steps in linear sequence.
For example, save this text as `two-steps.yaml`:

    defn:
      name: TwoSteps
    steps:
    - defnFile: step-one.yaml
    - defnFile: step-two.yaml

This workflow contains two steps. The steps will be loaded from files, located
relative to the `workspace` path provided on the command-line.

The two steps will run in sequence. One common use is to pass the output file
from the first step as the input file to the second step.

Here's an equivalent workflow that's a little more verbose. The extra `graph` section
lets you run the steps in a different order.

    defn:
      name: TwoSteps
    graph:
    - taskTwo
    - taskOne
    steps:
    - defn:
        name: taskOne
      defnFile: step-one.yaml
    - defn:
        name: taskTwo
      defnFile: step-two.yaml

Now `taskTwo` will run before `taskOne`.

Note that we've also given the subtasks explicit names. These can be different
from whatever's mentioned in the `defnFile` definition; the values in the
workflow definition will override those contained in the task file.

### Parallel workflows

Sometimes you want to run a step in parallel with different values. One example
is running the same task on multiple input files. Another is passing a series of
variable values for a parameter sweep.

The `scatterBy` attribute lets you choose an input or output parameter for that
purpose. The values will be split on new lines. For example:

    defn:
      name: FileParallel
    args:
      inputs:
        parallelTask.inputFile: |
          file1.txt
          file2.txt
    steps:
    - defn:
        name: parallelTask
      defnFile: step-one.yaml
      scatterBy: inputFile

The task will be run twice in parallel, once with `file1.txt` and once with
`file2.txt`. Each task's outputs will be stored in a subdirectory,
`parallelTask/1` and `parallelTask/2`.

If you have a large number of values you'd like to run at once, you can pass
them in from a file, using the `--inputs-from-file` option.

    dockerflow \
        --project=$PROJECT_NAME \
        --workflow-file=src/test/resources/parallel-graph.yaml \
        --workspace=$WORKSPACE_PATH \
        --inputs-from-file=parallelTask.inputFile=many_files.txt

The file `many_files.txt` can contain many file paths. All will run, with
retries, each on a separate VM, until complete. Note that to set the value of
the `inputFile` for subtask named `parallelTask`, you pass the input variable
name `parallelTask.inputFile` on the command-line, because the task definition
includes the task name, `parallelTask`.

See the section on [passing parameters](#passing-parameters) for more details.

There's also a corresponding `gatherBy` attribute so that you can gather the
outputs of multiple parallel tasks into arrays of file paths. For an example, see:

*   [src/test/resources/gather-graph.yaml](src/test/resources/gather-graph.yaml)

### Branching workflows

If file-parallel workflow steps aren't complex enough, you can write a branching
workflow. Each branch will execute in parallel.

    defn:
      name: Branching
    graph:
    - firstStep
    - BRANCH:
      - branchA
      - branchB
    steps:
    - defn:
        name: firstStep
      defnFile: task-one.yaml
    - defn:
        name: branchA
      defnFile: task-two.yaml
    - defn:
        name: branchB
      defnFile: task-three.yaml

You'll notice that there's a new keyword, `BRANCH`. It indicates that tasks
should be executed in parallel. In this example, `firstStep` runs, then
`branchA` and `branchB` run in parallel.

## Passing parameters

Now how do you pass parameters to the tasks? You can certainly set them in the
individual task definition files, as described in the [Pipelines API Guide]
(https://cloud.google.com/genomics/v1alpha2/pipelines-api-guide).

You can also set them in the workflow definition file.

    defn:
      name: TwoSteps
    steps:
    - defn:
        name: stepOne
      defnFile: step-one.yaml
      args:
        inputs:
          inputFile: input-one.txt
        outputs:
          outputFile: output-one.txt
    - defn:
        name: stepTwo
      defnFile: step-two.yaml
      args:
        inputs:
          inputFile: output-one.txt
        outputs:
          outputFile: output-two.txt

If it's clearer, you can set the parameters at the top of the workflow
definition:

    defn:
      name: TwoSteps
    args:
      inputs:
        stepOne.inputFile: input-one.txt
        stepTwo.inputFile: output-one.txt
      outputs:
        stepOne.outputFile: output-one.txt
        stepTwo.outputFile: output-two.txt
    steps:
    - defnFile: step-one.yaml
    - defnFile: step-two.yaml

In this case, the input and output names must be prefixed by the name of the
step. Since the definition (defn) of each task step is loaded from file, the
task names in the included files will be used.

Finally, you can set parameters from the command-line:

    dockerflow --project=$PROJECT_NAME \
        --workflow-file=src/test/resources/parallel-graph.yaml \
        --workspace=$WORKSPACE_PATH \
        --outputs=stepOne.outputFile=output-one.txt,\
            stepTwo.outputFile=output-two.txt \
        --inputs=stepTwo.inputFile=output-one.txt

To set the value of the `inputFile` for subtask named `stepOne`, you pass the
input variable name `stepOne.inputFile` on the command-line, because the task
name specified is `stepOne`.

## Testing

Workflows can be tricky to test and debug. Dataflow has a local runner that
makes it easy to fix the obvious bugs before running in your Google Cloud
Platform project.

To test locally, set `--runner=DirectPipelineRunner`. Now Dataflow will run on
your local computer rather than in the cloud. You'll be able to see all of the
log messages.

Two other flags are really useful for testing: `--test=true` and
`--resume=true`.

When you set `test` to true, you'll get a dry run of the pipeline. No calls to
the Pipelines API will be made. Instead, the code will print a log message and
continue. That lets you do a first sanity check before submitting and running on
the cloud. You can catch many errors, mismatched parameters, etc.

When you use the `resume` flag, Dockerflow will try to resume a failed pipeline
run. For example, suppose you're trying to get your 10-step pipeline to work. It
fails on step 6. You go into your YAML definition file, or edit your Java code.
Now you want to re-run the pipeline. However, it takes 1 hour to run steps 1-5.
That's a long time to wait. With `--resume=true`, Dockerflow will look to see if
the outputs of each step exist already, and if they do, it will print a log
message and proceed to the next step. That means it takes only seconds to skip
ahead to the failed step and try to rerun it.

## What next?

*   See the YAML examples in the [src/test/resources](src/test/resources) directory.
*   See the Java code examples in
    *    [examples](examples)
    *    [src/test/java/com/google/cloud/genomics/dockerflow/examples](src/test/java/com/google/cloud/genomics/dockerflow/examples)
*   Learn about the [Pipelines API]
    (https://cloud.google.com/genomics/v1alpha2/pipelines).
*   Read about [Dataflow](https://cloud.google.com/dataflow).
*   Write your own workflows!

## FAQ and Troubleshooting

### What if I want to run large batch jobs?

Google Cloud Platform has various quotas that affect how many VMs and IP addresses, and how much disk space you can get. Some tips:

* [Check and potentially increase quotas](https://console.cloud.google.com/compute/quotas)
* Consider listing all zones in the region or geography where your Cloud Storage bucket is (e.g., for standard buckets in the EU, use "eu-"; for regional buckets in US central, use "us-central-")
* The pipeline system will queue jobs until resources are available if quotas are exceeded
* Dockerflow will abort if any job fails. Use the '--abort=false' flag for different behavior.
