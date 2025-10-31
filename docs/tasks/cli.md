# Task: Manage Actuated via CLI

Monitor Actuated runners and jobs from the command line.

## Installation

Download and installation instruction are available via the [actuated-dashboard](https://dashboard.actuated.com/cli)

You'll need to run `actuated-cli auth` first, so that you can get a Personal Access Token with the appropriate scopes from GitHub.

## View queued jobs

```bash
actuated-cli jobs \
    actuated-samples
```

## View runners for organization

```bash
actuated-cli runners \
    actuated-samples
```

## View SSH sessions available:

```bash
actuated-cli ssh ls
```

Hosts are ordered by the connected time.

```bash
| NO  |   ACTOR   |                 HOSTNAME                 |  RX   |  TX   | CONNECTED |
|-----|-----------|------------------------------------------|-------|-------|-----------|
|   1 | alexellis | 6aafd53144e2f00ef5cd2c16681eeab4712561a6 | 13679 | 10371 | 6m4s      |
|   2 | alexellis | fv-az268-245                             | 23124 | 13828 | 12m2s     |
```

## Connect to an SSH session

Connect to the first available session from your account:

```bash
actuated-cli ssh connect
```

Connected to the second session in the list:

```bash
actuated-cli ssh connect 2
```

Connect to a specific session by hostname:

```bash
actuated-cli ssh connect runner1
```

Connect to a specific session with a host prefix:

```bash
actuated-cli ssh connect 6aafd
```

## Check the logs of VMs

View the serial console and systemd output of the VMs launched on a specific server.

* Check for timeouts with GitHub's control-plane
* View output from the GitHub runner binary
* See boot-up messages
* Check for errors if the GitHub Runner binary is out of date

```bash
actuated-cli logs \
    --owner actuated-samples \
    --age 15m \
    runner1
```

The age is specified as a Go duration i.e. `60m` or `24h`.

You can also get the logs for a specific runner by using the `--id` flag.

```bash
actuated-cli logs \
    --owner actuated-samples \
    --id ea5c285282620927689d90af3cfa3be2d5e2d004
    runner1
```

## Check the logs of the actuated agent service

Show the logs of the actuated agent binary running on your server.

View VM launch times, etc.

```bash
actuated-cli agent-logs \
    --owner actuated-samples \
    --age 60m \
    runner1
```

## Schedule a repair to re-queue jobs

If a job has been retried for 30 minutes, without a runner to take it, it'll be taken off the queue. This command will re-queue all jobs that are in a "queued" state.

Run sparingly because it will launch one VM per job queued.

Do not run it multiple times in succession. And then if you are still in doubt where jobs are stuck - first check the logs of the runners via `actuated-cli agent-logs`, finally, reach out to support.

```bash
actuated-cli repair \
    actuated-samples
```

In the above command `actuated-samples` is the GitHub organization.

## Rescue a remote server

Restart the agent by sending a `kill -9` signal:

```bash
actuated-cli restart \
    --owner actuated-samples \
    runner1
```

Any inflight VMs will be killed, see also: `actuated-cli update --force`

Reboot the machine, if in an unrecoverable position:

```bash
actuated-cli restart \
    --owner actuated-samples \
    --reboot
    runner1
```

Use with caution, since this may not perform a safe and clean shutdown.

## JSON mode

Add `--json` to any command to get JSON output for scripting.

API rate limits apply, so do not run the CLI within a loop or `watch` command.


## Help & support

Reach out to our team [on Slack](https://self-actuated.slack.com)