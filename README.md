**This repository has been moved to https://github.com/alphauslabs/oops.**

---

![main](https://github.com/flowerinthenight/oops/workflows/main/badge.svg)

## Overview

`oops` is an automation-friendly, highly-scalable, and scriptable API/generic testing tool built to run on [Kubernetes](https://kubernetes.io/). It accepts and runs file-based test cases, or 'scenarios' written in [YAML](https://yaml.org/).

## Running in a local environment

You can install the binary and run local scenario file(s). This is useful for testing your scenario file(s) before deployment.

```bash
# Install the binary.
$ brew tap flowerinthenight/tap
$ brew install oops

# Run a single scenario file.
$ oops -s ./examples/01-simple.yaml

# or multiple scenario files.
$ oops -s ./examples/01-simple.yaml -s ./examples/02-chaining.yaml

# For multiple scenario files in a directory (likely the case, eventually),
# you can provide the root directory as input. It will scan all yaml files
# recursively and execute them sequentially.
$ oops --dir ./examples/
```

## Deploying to Kubernetes

To scale the testing workload, this tool will attempt to distribute all scenario files to all worker pods using pub/sub messaging (currently supports SNS+SQS, and GCP PubSub). At the moment, it needs to be triggered first before the actual execution starts. The trigger payload is `{"code":"start"}`.

It is recommended that your scenario files are isolated at the file level. That means that as much as possible, a single scenario file is standalone. For integration tests, a single scenario file could contain multiple related test cases (i.e. create, inspect, delete type of tests) in it.

Although this tool was built to run on k8s, it will work just fine in any environment as long as the workload can be distributed properly using the currently supported pubsub services.

An example [`deployment.yaml`](https://github.com/flowerinthenight/oops/blob/master/deployment.yaml) for k8s using GCP PubSub is provided for reference. Make sure to update the relevant values for your own setup.

## Scenario file

The following is the specification of a valid scenario file. All scenario files must have a `.yaml` extension.

```yaml
tags:
  # Tag(s) for this scenario file. If these tag combinations match with what is
  # provided in the --tags flag (if any), this scenario file is allowed to run.
  key1: value1
  key2: value2

env:
  # These key/values will be added to the environment variables in your local
  # or in the pod oops will be running on.
  ENV_KEY1: value1
  ENV_KEY2: value2

# Any value that starts with '#!' (i.e. #!/bin/bash) will be written to disk as
# an executable script file and the resulting output combined from stdout & stderr
# will become the final evaluated value. This is useful if you chain http calls,
# such as, the url of the second test is from the payload response of the first
# test, etc.
#
# If supported, the filename of the script will be indicated below.
#
# At the moment, shell interpreters (that works with `bin -c scriptfile` command is
# supported. Also, Python is supported as well by using the shebang:
#   #!/path/to/python/binary

# A script to run before anything else. A non-zero return value indicates a failure.
# Filename: <tempdir>/<scenario-filename>.yaml_prepare
# Example: /tmp/scenario01.yaml_prepare
prepare: |
  #!/bin/bash
  echo "prepare"

# A list of http requests to perform sequentially. This tool will continue running
# all the list entries even if failure occurs during the execution.
run:
  - http:
      method: POST

      # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_url
      # Example: /tmp/scenario01.yaml_run0_url
      url: "https://service.alphaus.cloud/users"

      # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_hdr.<key>
      # Example: /tmp/scenario01.yaml_run0_hdr.Authorization
      headers:
        Authorization: |
          #!/bin/bash
          echo -n "Bearer $TOKEN"
        Content-Type: application/json

      # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_qparams.<key>
      # Example: /tmp/scenario01.yaml_run0_qparams.key2
      query_params:
        key1: value1
        key2: |
          #!/bin/bash
          echo -n "$KEY2"

      # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_forms.<key>
      # Example: /tmp/scenario01.yaml_run0_forms.key2
      forms:
        key1: value1
        key2: |
          #!/bin/bash
          echo -n "$KEY2"

      # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_payload
      # Example: /tmp/scenario01.yaml_run0_payload
      payload: |
        {"key1":"value1","key2":"value2"}

      # If response payload is not empty, its contents will be written in the
      # file below. Useful if you want to refer to it under 'asserts.script'
      # and/or 'check'.
      response_out: /tmp/out.json

      asserts:
        # The expected http status code. Indicates a failure if not equal.
        status_code: 200

        # JSON validation using https://github.com/xeipuuv/gojsonschema package.
        validate_json: |
          {
            "type": "object",
            "properties": {
              ...
            }
          }

        # A non-zero return value indicates a failure.
        # Filename: <tempdir>/<scenario-filename>.yaml_run<index>_assertscript
        # Example: /tmp/scenario01.yaml_run0_assertscript
        script: |
          #!/bin/bash
          if [[ "$(cat /tmp/out.json | jq -r .username)" != "user01" ]]; then
            echo "try fail"
            exit 1
          fi

# A script to run after 'run', if present. Useful also as a standalone script
# in itself, if 'run' is empty. A non-zero return value indicates a failure.
# Filename: <tempdir>/<scenario-filename>.yaml_check
# Example: /tmp/scenario01.yaml_check
check: |
  #!/bin/bash
  echo "check"
```

Example [scenario files](https://github.com/flowerinthenight/oops/tree/master/examples) are provided for reference as well. You can run them as is.

---

## TODO

PR's are welcome!

- [ ] Parsing and assertions for response JSON payloads
- [x] Labels/tags for filtering what tests to run
- [x] Support for other scripting engines other than `bash/sh`, i.e. Python
- [ ] Store reports to some storage, i.e. S3, GCS, etc.
- [ ] Support for AKS + Service Bus
- [x] Add GCP PubSub for reporting
