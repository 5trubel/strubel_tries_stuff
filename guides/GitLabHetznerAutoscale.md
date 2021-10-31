# Gitlab Runner Hetzner AutoScale

## Table of Content
- [Preparation (Hetzner)](#preparation--hetzner-)
- [Preparation (Server/Broker)](#preparation--server-broker-)
- [Runner Setup](#runner-setup)
- [Creation of a Container](#creation-of-a-container)


## Preparation (Hetzner)

First you need to create an API Token, save it, you'll need it later. Give it a fitting description and read and write permissions. 

![APIToken](https://sx.5trubel.de/97rm5.png)

You'll find this Window under `Secruity` -> `API Tokens` -> `Generate API Token`

![API_TokenGIF](https://sx.5trubel.de/cvhhf.gif)

## Preparation (Server/Broker)

I will from now on call the Gitlab-Runner itself, so the part who orders more vServers when needed, Broker. First you need to get a cheap vServer or a use your already existing server for it, it must have Docker installed.  

Register a GitLab-Runner in your GitLab Instance. You can use any GitLab-Runner you like to, we just need GitLab to register the Runner. If you don't know how this works, you can follow [The Offical](https://docs.gitlab.com/runner/register/) GitLab Docs and check back here if you're finished. 

```
gitlab-runner register \
  --non-interactive \
  --url https://gitlab.example.com
  --registration-token <TOKEN> \
  --executor docker \
  --docker-image docker:dind \
  --paused
```

## Runner Setup

Create a Folder for the configs, in this example it is called `hetzner_config`, copy the Runner config file into the created folder. Edit the config to the following (if not already done in the registration step). Edit the config so it fits your purpose. 

```
  [[runners]]
  ...
  executor = "docker+machine"
```

```
  [runners.machine]
    IdleCount = 0
    IdleTime = 600
    MachineDriver = "hetzner"
    MachineName = "%s"
    MachineOptions = [
        "hetzner-api-token=<API Token from Step one>", 
        "hetzner-image=ubuntu-20.04",
        "hetzner-server-type=cx51-ceph"
        ]
    [[runners.machine.autoscaling]]
      Periods = ["* * * * * sun *"]
      Timezone = ""
      IdleCount = 0
      IdleTime = 21600
```
  In my Example here, it is set so that no Machines are ready for use, in this case, if a Job comes in a machine needs to be created first, which isn't that bad when you're not depending on a few minutes. 

  `IdleCount=0` -> Sets the Idle Count, how many Machines should always be available. As already mentioned if set to zero, a Machine needs to be created first.

  `IdleTime=600` -> If more than `IdleCount` Machines are available, the overhead machines get deleted after being `IdleTime` in Idle. 

  `MachineDriver` -> Tell the Runner to use Hetzner

   `MachineName` -> Sets the name of the vServer in the Hetzner Console. 

   `MachineOptions` (Not all but the most important):

* `hetzner-image` -> Sets the image that is provisioned on new machines. 
* `hetzner-image-id` -> Same as above, but with an ID - [API](https://docs.hetzner.cloud/#resources-images-get)
* `hetzner-server-type` -> Machine Type (CX11, CX51-CEPH, etc.) - [API](https://docs.hetzner.cloud/#resources-server-types-get)
* `hetzner-location` -> Force a Hetzner Location (Falkenstein, Nurnberg, Helsinki) - [API](https://docs.hetzner.cloud/#resources-locations-get)
* `hetzner-placement-group` -> Add the machine to a Placement Group


`runners.machine.autoscaling` -> Use this for "Peak usage", lets you set a different `IdleCount`/`IdleTime` at a certain Day or Time. `IdleCount` and `IdleTime` are the same as above. 

`Periods = ["* * 9-17 * * mon-fri *"]` -> More machines will be provisioned Monday till Friday, 9-17

`Timezone` -> Sets your Timezone

## Creation of a Container

Create a `docker-compose.yml` that looks like this and start it with `docker-compose up`, check if the Runner connects.

```yml
version: '2'
services:
  hetzner-runner:
    image: mawalu/hetzner-gitlab-runner:latest
    mem_limit: 128mb
    memswap_limit: 256mb
    volumes:
      - "./hetzner_config:/etc/gitlab-runner"
    restart: always
```

If everything works, you should get this messages

```log
Starting broker_hetzner-runner_1 ... done
Attaching to broker_hetzner-runner_1
hetzner-runner_1  | Runtime platform                                    arch=amd64 os=linux pid=8 revision=8925d9a0 version=14.1.0
hetzner-runner_1  | Starting multi-runner from /etc/gitlab-runner/config.toml...  builds=0
hetzner-runner_1  | Running in system-mode.
hetzner-runner_1  |
hetzner-runner_1  | Configuration loaded                                builds=0
hetzner-runner_1  | listen_address not defined, metrics & debug endpoints disabled  builds=0
hetzner-runner_1  | [session_server].listen_address not defined, session endpoints disabled  builds=
```

Create a Job and check if the runner works, if everything works correct the Output should look like this: 

```
hetzner-runner_1  | Checking for jobs... received                       job=724 repo_url= runner=L7mc7so7
hetzner-runner_1  | Running pre-create checks...                        driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Creating machine...                                 driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8) Creating SSH key...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8) SSH key not found in Hetzner. Uploading...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8) Creating Hetzner server...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8)  -> Creating server runner-l7mc7so7-1635714803-ed01fdb8[15658509] in create_server[299061970]  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8)  -> Server runner-l7mc7so7-1635714803-ed01fdb8[15658509]: Waiting to come up...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8) Using public network ...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | (runner-l7mc7so7-1635714803-ed01fdb8)  -> Server runner-l7mc7so7-1635714803-ed01fdb8[15658509] ready. Ip 65.21.181.221  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Waiting for machine to be running, this may take a few minutes...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Detecting operating system of created instance...   driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Waiting for SSH to be available...                  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Detecting the provisioner...                        driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Provisioning with ubuntu(systemd)...                driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Installing Docker...                                driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Copying certs to the local machine directory...     driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Copying certs to the remote machine...              driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
hetzner-runner_1  | Setting Docker configuration on the remote daemon...  driver=hetzner name=runner-l7mc7so7-1635714803-ed01fdb8 operation=create
```

If the Job succeeds, congratulations! You know have working autoscaled GitLab-Runner! You can now put the Container into the background by `CTRL+C` and entering `docker-compose up -d`.

## Disclaimer
If I missed something or explained something wrong/too complicated, feel free to create an Issue. I will update this Guide in the future when I test more and get something to work. At this point I'm working on getting Insecure Registries to be accepted, so if you can help me, Issues or PRs are always welcome! 