# Azure DevOps Self-Agent

An agent that you set up and manage on your own to run jobs is a self-hosted agent.

You can use self-hosted agents in Azure Pipelines.

Self-hosted agents give you more control to install dependent software needed for your builds and deployments.

<br><br>
## Sign up for [Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-sign-up?view=azure-devops)

Sign up for an Azure DevOps organization and Azure Pipelines to begin managing CI/CD to deploy your code.<br><br>

1. Log into your Microsoft account.

   ![Microsoft Account](/images/microsoft_account.png)

2. Enter a name for your organization, select a host location from the drop-down menu, enter the characters you see, and then select **Continue**.

   ![Create ADO Organization](/images/ado-organization.png)

   > Use the following URL to sign in to your organization at any time: https://dev.azure.com/{yourorganization}

3. Enter a name for your project, select the visibility, and optionally provide a description. Then choose **Create project**.

   ![Create ADO Project](/images/ado-create_project.png)


<br><br>
## Create and manage [agent pools](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser#create-agent-pools)

An agent pool is a collection of agents.

Instead of managing each agent individually, you organize agents into agent pools.

When you configure an agent, it is registered with a single pool, and when you create a pipeline, you specify the pool in which the pipeline runs.

When you run the pipeline, it runs on an agent from that pool that meets the demands of the pipeline.

In Azure Pipelines, pools are scoped to the entire organization, so you can share the agent machines across projects.

This article describes how to [Create the Azure DevOps Agent](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser#create-agent-pools) (ADO) on different targets.

- [Self-hosted agents running on Docker](#self-hosted-agents-running-on-docker)
- [Self-hosted agents running on Kubernetes](#self-hosted-agents-running-on-kubernetes)


<br><br>
### Self-hosted agents running on **Docker**

To be successful and get the most of this section, you are encouraged to have the [Docker Runtime](https://docs.docker.com/docker-for-windows/install/) ready.
<details>
<summary>Expand for instructions</summary>

1. Go to your organization and select **Organization settings**.

   ![ADO Organization Settings](/images/ado-organization_settings.png)

2. Select **Agent pools** in the left panel under **Pipelines**.

   ![ADO Organization Settings Agent pools](/images/ado-organization_settings_agent_pools.png)

3. Select **Add pool**.

4. Select **Self-hosted** for **Pool type**, type **Docker** as the **Name** of the agent pool and select **Create**.

   ![ADO Organization Settings Add Agent pools](/images/ado-organization_settings_agent_pools-add.png)

5. See the agent pool **Docker**.

   ![ADO Agent Pool Docker](/images/ado-agent_pool-docker.png)

6. Create a directory of your choice and navigate into it.

   ![Doker dir](/images/ado-agent_pool-docker-dir.png)

7. Save the following content to file **```Dockerfile```**.

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

8. Save the following content to file **```start.sh```**.

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

9. Build the container with the ADO agent software.


    ```console
    docker build -t dockeragent:latest .
    ```

10. Run the container.

    ```console
    docker run \
    -e AZP_URL=$AZP_URL \
    -e AZP_TOKEN=$AZP_TOKEN \
    -e AZP_POOL=Docker \
    adoagent-docker:latest
    ```

    > **Warning**

    > You need to control your ADO url and token using environment variables.

    > Command above is for example only.

    | Env Var | Description |
    |----------|---------------|
    | `AZP_URL` | The URL of the Azure DevOps or Azure DevOps Server instance. |
    | `AZP_TOKEN` | [Personal Access Token](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&amp%3Btabs=Windows&tabs=Windows) (PAT) with Agent Pools (read, manage) scope, created by a user who has permission to configure agents, at AZP_URL. |

17. See the container agent connected to the agent pool.

    ![ADO agent listening](/images/ado-agent_pool-docker-run-listening.png)

18. Go to your **Organization settings**, select **Agent pools** and select the **Docker** agent pool.

19. You should now see your container agent connected.

    ![ADO agent pool with connected agent](/images/ado-agent_pool-docker-connected_agent.png)

20. You can start more container agents as needed.

</details>