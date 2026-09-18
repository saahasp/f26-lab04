# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

### Milestone 1 (healthy deploy)

```
-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-041eadd877bd9be2d                                     |
|  ServiceUrl|  http://ec2-34-229-96-198.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+
```

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-34-229-96-198.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 3. What the template created

The template creates a single `t3.micro` EC2 instance running Amazon Linux 2023, with a security group that opens port 8080 (the service port) and port 22 (SSH fallback) to all inbound traffic (0.0.0.0/0), while leaving outbound unrestricted so the instance can install packages and pull images. A UserData bash script acts as the glue: on first boot it installs Docker via `dnf`, enables the Docker daemon, then runs `docker run` to pull the `ghcr.io/cmu-17-214/lab04-service:latest` container image and start it with a host-to-container port mapping of 8080:8080. The instance is also assigned the `LabInstanceProfile` IAM role, which grants SSM permissions for remote shell access without needing a key pair.

## 4. Scenario 2 diagnosis

### Scenario 2 deploy outputs

```
--------------------------------------------------------------------------
|                             DescribeStacks                             |
+------------+-----------------------------------------------------------+
|  InstanceId|  i-029c9e615af50b5e3                                      |
|  ServiceUrl|  http://ec2-54-242-231-218.compute-1.amazonaws.com:8080   |
+------------+-----------------------------------------------------------+
```

**The failing curl** (command and output):

```
$ curl http://ec2-54-242-231-218.compute-1.amazonaws.com:8080/api/health
curl: (7) Failed to connect to ec2-54-242-231-218.compute-1.amazonaws.com port 8080 after 2183 ms: Could not connect to server
```

**The log line that told you what was wrong:**

```
$ docker ps
CONTAINER ID   IMAGE                                     ...   PORTS                                       NAMES
9ac1e09bf0de   ghcr.io/cmu-17-214/lab04-service:latest   ...   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

$ docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

The `params-scenario2.json` sets `PortOverride` to `9090`, which makes the container's `PORT` environment variable `9090`, so the service binds to port 9090 inside the container. However, the Docker port mapping (`-p 8080:8080`) only forwards host port 8080 to container port 8080. Since nothing is listening on container port 8080, incoming traffic is refused. The `docker ps` output shows `8080->8080` but `docker logs` shows `listening on 9090` — the mismatch is the root cause. The fix is to delete the broken stack and redeploy with `params-healthy.json`, which leaves `PortOverride` empty so the service binds to the default port 8080, matching the port mapping.

**The healthy curl after the fix:**

```
$ curl http://ec2-52-90-0-217.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```
$ aws cloudformation describe-stacks --stack-name lab04-service
An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```
