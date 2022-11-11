# Azure DevOps Self-Agent

<br><br>
## Sign up for [Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-sign-up?view=azure-devops)<br><br>

1. Log into your Microsoft account.

   ![Microsoft Account](/images/microsoft_account.png)

2. Enter a name for your organization, select a host location from the drop-down menu, enter the characters you see, and then select Continue.

   ![Create ADO Organization](/images/ado-organization.png)

   > Use the following URL to sign in to your organization at any time: https://dev.azure.com/{yourorganization}

3. Enter a name for your project, select the visibility, and optionally provide a description. Then choose Create project.

   ![Create ADO Project](/images/ado-create_project.png)













Create a self-hosted agent pool

https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser#create-agent-pools


Docker

1. Save the following content to a file named **```Dockerfile```**.

```console
FROM ubuntu:20.04
RUN DEBIAN_FRONTEND=noninteractive apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get upgrade -y

RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
    apt-transport-https \
    apt-utils \
    ca-certificates \
    curl \
    git \
    iputils-ping \
    jq \
    lsb-release \
    software-properties-common

RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# Can be 'linux-x64', 'linux-arm64', 'linux-arm', 'rhel.6-x64'.
ENV TARGETARCH=linux-x64

WORKDIR /azp

COPY ./start.sh .
RUN chmod +x start.sh

ENTRYPOINT [ "./start.sh" ]
```

2. Save the following content to file **```start.sh```**.

```console
#!/bin/bash
set -e

if [ -z "$AZP_URL" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

if [ -z "$AZP_TOKEN_FILE" ]; then
  if [ -z "$AZP_TOKEN" ]; then
    echo 1>&2 "error: missing AZP_TOKEN environment variable"
    exit 1
  fi

  AZP_TOKEN_FILE=/azp/.token
  echo -n $AZP_TOKEN > "$AZP_TOKEN_FILE"
fi

unset AZP_TOKEN

if [ -n "$AZP_WORK" ]; then
  mkdir -p "$AZP_WORK"
fi

export AGENT_ALLOW_RUNASROOT="1"

cleanup() {
  if [ -e config.sh ]; then
    print_header "Cleanup. Removing Azure Pipelines agent..."

    # If the agent has some running jobs, the configuration removal process will fail.
    # So, give it some time to finish the job.
    while true; do
      ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break

      echo "Retrying in 30 seconds..."
      sleep 30
    done
  fi
}

print_header() {
  lightcyan='\033[1;36m'
  nocolor='\033[0m'
  echo -e "${lightcyan}$1${nocolor}"
}

# Let the agent ignore the token env variables
export VSO_AGENT_IGNORE=AZP_TOKEN,AZP_TOKEN_FILE

print_header "1. Determining matching Azure Pipelines agent..."

AZP_AGENT_PACKAGES=$(curl -LsS \
    -u user:$(cat "$AZP_TOKEN_FILE") \
    -H 'Accept:application/json;' \
    "$AZP_URL/_apis/distributedtask/packages/agent?platform=$TARGETARCH&top=1")

AZP_AGENT_PACKAGE_LATEST_URL=$(echo "$AZP_AGENT_PACKAGES" | jq -r '.value[0].downloadUrl')

if [ -z "$AZP_AGENT_PACKAGE_LATEST_URL" -o "$AZP_AGENT_PACKAGE_LATEST_URL" == "null" ]; then
  echo 1>&2 "error: could not determine a matching Azure Pipelines agent"
  echo 1>&2 "check that account '$AZP_URL' is correct and the token is valid for that account"
  exit 1
fi

print_header "2. Downloading and extracting Azure Pipelines agent..."

curl -LsS $AZP_AGENT_PACKAGE_LATEST_URL | tar -xz & wait $!

source ./env.sh

print_header "3. Configuring Azure Pipelines agent..."

./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "$AZP_URL" \
  --auth PAT \
  --token $(cat "$AZP_TOKEN_FILE") \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula & wait $!

print_header "4. Running Azure Pipelines agent..."

trap 'cleanup; exit 0' EXIT
trap 'cleanup; exit 130' INT
trap 'cleanup; exit 143' TERM

chmod +x ./run-docker.sh

# To be aware of TERM and INT signals call run.sh
# Running it with the --once flag at the end will shut down the agent after the build is executed
./run-docker.sh "$@" & wait $!
```

3. Run the following command.

docker build -t dockeragent:latest .

```console
Example:
ubuntu@DESKTOP-3UU8691:~/source/azure-ado-selfagent/Docker$ docker build -t adoagent-docker:latest .
[+] Building 6.7s (14/14) FINISHED
 => [internal] load build definition from Dockerfile                                                     0.0s
 => => transferring dockerfile: 667B                                                                     0.0s
 => [internal] load .dockerignore                                                                        0.0s
 => => transferring context: 2B                                                                          0.0s
 => [internal] load metadata for docker.io/library/ubuntu:20.04                                          6.6s
 => [auth] library/ubuntu:pull token for registry-1.docker.io                                            0.0s
 => [1/8] FROM docker.io/library/ubuntu:20.04@sha256:450e066588f42ebe1551f3b1a535034b6aa46cd936fe7f2c6b  0.0s
 => [internal] load build context                                                                        0.0s
 => => transferring context: 2.52kB                                                                      0.0s
 => CACHED [2/8] RUN DEBIAN_FRONTEND=noninteractive apt-get update                                       0.0s
 => CACHED [3/8] RUN DEBIAN_FRONTEND=noninteractive apt-get upgrade -y                                   0.0s
 => CACHED [4/8] RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends       0.0s
 => CACHED [5/8] RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash                                   0.0s
 => CACHED [6/8] WORKDIR /azp                                                                            0.0s
 => CACHED [7/8] COPY ./start.sh .                                                                       0.0s
 => CACHED [8/8] RUN chmod +x start.sh                                                                   0.0s
 => exporting to image                                                                                   0.0s
 => => exporting layers                                                                                  0.0s
 => => writing image sha256:21b2316c0bf90b0a7b6a8b78588ce586a092e20da3614d2b0a9baabc97246ae3             0.0s
 => => naming to docker.io/library/adoagent-docker:latest                                                0.0s
ubuntu@DESKTOP-3UU8691:~/source/azure-ado-selfagent/Docker$
```

4. Run the container.

This installs the latest version of the agent, configures it, and runs the agent.

It targets the Docker pool created earlier.

docker run \
-e AZP_URL=https://dev.azure.com/csa-kledsonbasso \
-e AZP_TOKEN=bxxqrz2tmerbod6mpuhxqpvisj52w6aimjmw55samf6bvgbbtcsq \
-e AZP_POOL=Docker \
adoagent-docker:latest

You can control the pool and agent work directory by using environment variables.

https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&amp%3Btabs=Windows&tabs=Windows

Environment variables
Environment variable	Description
AZP_URL	The URL of the Azure DevOps or Azure DevOps Server instance.
AZP_TOKEN	Personal Access Token (PAT) with Agent Pools (read, manage) scope, created by a user who has permission to configure agents, at AZP_URL.
AZP_AGENT_NAME	Agent name (default value: the container hostname).
AZP_POOL	Agent pool name (default value: Default).
AZP_WORK	Work directory (default value: _work).