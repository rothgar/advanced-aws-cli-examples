# Advanced AWS CLI usage examples

This repo is a collection of hard won [AWS CLI](https://aws.amazon.com/cli/) commands that can be used as starting points for your own use cases.

If you've spent 2 hours crafting the perfect command there's a good chance it could help someone else too!
Consider [submitting a PR](https://github.com/rothgar/advanced-aws-cli-examples/pulls) to help the next person.

## Advanced Queries

### Print ec2 instances based on tag

If you want to print a all ec2 instances with Instance ID, Instance Type, and Public IP address you can use a query like the following.

```bash
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress]' \
  --output text
```
> example output

```bash
i-00cea6e9c6ffeb6ef     m5.large        34.208.319.189
i-09b019e2af1ddca36     m5.large        54.203.900.191
i-0b4a835ac6c46f716     m5.large        34.220.260.226
i-0f2944fd98a0e6cee     m5.large        34.216.734.203
i-01c57eb8bf279abe3     m5.large        52.88.441.205
i-072890691849f7c36     m5.large        34.292.102.24
i-0709516028af86309     m5.large        34.220.301.165
```

If you want to add a field for a tag (e.g. the `Name` tag) you will need to use a `?` query search.
Notice you need to add `[ ]` grouping around the `Tags` and then use `[0][0]` to print the value.

```bash
aws ec2 describe-instances \
--query 'Reservations[].Instances[].[ [Tags[?Key==`Name`].Value][0][0],InstanceId]' \
--output text
```

> example output
```bash
stage-ng-655ee1b7-Node  i-00cea6e9c6ffeb6ef
stage-ng-655ee1b7-Node  i-09b019e2af1ddca36
stage-ng-655ee1b7-Node  i-0b4a835ac6c46f716
stage-ng-655ee1b7-Node  i-0f2944fd98a0e6cee
stage-ng-655ee1b7-Node  i-01c57eb8bf279abe3
prod-ng-30b6388f-Node   i-072890691849f7c36
prod-ng-30b6388f-Node   i-0709516028af86309
stage-ng-655ee1b7-Node  i-04642a47773962ac7
```

You can further reduce the output to only running instances by using `--filter`

> Note: It's always better to use `--filter` rather than `--query` if you have a large list because `--filter` will be done server side and make smaller page sizes and less data sent to the client.

```bash
aws ec2 describe-instances \
  --filter "Name=instance-state-name,Values=running"
  --query 'Reservations[].Instances[].[ [Tags[?Key==`Name`].Value][0][0],InstanceId]' \
  --output text
```

To use a filter for if a specific tag exists on a resource you can use the following.

```bash
aws ec2 describe-instances \
  --filter "Name=tag-key,Values=alpha.eksctl.io/nodegroup-name"
  --query 'Reservations[].Instances[].[ [Tags[?Key==`Name`].Value][0][0],InstanceId,[Tags[?Key==`alpha.eksctl.io/nodegroup-name`].Value][0][0]]'' \
  --output text
```
> example output
```bash
stage-ng-655ee1b7-Node  i-00cea6e9c6ffeb6ef     ng-655ee1b7
stage-ng-655ee1b7-Node  i-09b019e2af1ddca36     ng-655ee1b7
stage-ng-655ee1b7-Node  i-0b4a835ac6c46f716     ng-655ee1b7
stage-ng-655ee1b7-Node  i-0f2944fd98a0e6cee     ng-655ee1b7
stage-ng-655ee1b7-Node  i-01c57eb8bf279abe3     ng-655ee1b7
prod-ng-30b6388f-Node   i-072890691849f7c36     ng-30b6388f
prod-ng-30b6388f-Node   i-0709516028af86309     ng-30b6388f
stage-ng-655ee1b7-Node  i-04642a47773962ac7     ng-655ee1b7
```

If you want to further reduce your `--filter` based on a tag value you can use the following.

```bash
aws ec2 describe-instances \                                                                                                                        
  --filter "Name=tag:alpha.eksctl.io/cluster-name,Values=stage" \  
  --query 'Reservations[].Instances[].[ [Tags[?Key==`Name`].Value][0][0],InstanceId,[Tags[?Key==`alpha.eksctl.io/cluster-name`].Value][0][0]]' \
  --output text
```
> example output
```bash
stage-ng-655ee1b7-Node  i-00cea6e9c6ffeb6ef     stage
stage-ng-655ee1b7-Node  i-09b019e2af1ddca36     stage
stage-ng-655ee1b7-Node  i-0b4a835ac6c46f716     stage
stage-ng-655ee1b7-Node  i-0f2944fd98a0e6cee     stage
stage-ng-655ee1b7-Node  i-01c57eb8bf279abe3     stage
stage-ng-655ee1b7-Node  i-04642a47773962ac7     stage
stage-ng-655ee1b7-Node  i-00247ff71444822f8     stage
```

Make sure you put the `?` operator at the right level of the object.
You can print the VpcId for the `eks-cluster-sg-stage-109757182` security group with

```bash
aws ec2 describe-security-groups --query 'SecurityGroups[?GroupName==`eks-cluster-sg-stage-109757182`].VpcId'
```
> example output
```bash
vpc-0dcbebaad295821ce
```

### Nesting command output

You often want to look up values for something that requires information from another command output.
In this case you can use standard shell value substitution with `$()`.

Make sure your subshell command is only printing the text you want (without headers and in the correct text or json format).

```bash
aws ecs list-container-instances --cluster cftc --query 'containerInstanceArns[*]' --out text
```

Now you can simply plug in that command into a larger command like so.

> If the correct subshell command was the _last_ command you ran you can use `$(!!)` as a shorthand for replacing `!!` for the last command you ran. You can see more advanced command history usage in my [mastering zsh workshop](https://github.com/rothgar/mastering-zsh/blob/master/docs/config/history.md#using-words-from-the-last-command)

```bash
aws ecs describe-container-instances --container-instances $(aws ecs list-container-instances --cluster cftc --query 'containerInstanceArns[*]' --out text) --cluster cftc --query 'containerInstances[*].{status: status,id: ec2InstanceId,agent: versionInfo.agentVersion,docker: versionInfo.dockerVersion,tasks: runningTasksCount,arch: attributes[?name==`ecs.cpu-architecture`] | [0].value,cpu: remainingResources[?name==`CPU`] | [0].integerValue,memory: remainingResources[?name==`MEMORY`] | [0].integerValue,hostname: attributes[?name==`hostname`] | [0].value,datacenter: attributes[?name==`datacenter`] | [0].value}' --out table
```
> example output
```bash
------------------------------------------------------------------------------------------------------------------------------------
|                                                    DescribeContainerInstances                                                    |
+--------+-----------------------+---------+-------------------------+----+---------+-------+--------+------------------+----------+
|  ACTIVE|  mi-098882def3d5812b4 |  1.45.0 |  DockerVersion: 19.03.6 |  1 |  arm64  |  3840 |  7555  |  raspberry-pi-3  |  nathan  |
|  ACTIVE|  mi-0560dc180a9c56d9f |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  arm64  |  4096 |  7811  |  None            |  None    |
|  ACTIVE|  mi-01b5cfa868a532fa3 |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  x86_64 |  8192 |  15675 |  ecs2            |  justin  |
|  ACTIVE|  mi-02844112aad62b81f |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  arm64  |  4096 |  7811  |  raspberry-pi-4  |  nathan  |
|  ACTIVE|  mi-096b1b81d8db7b3be |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  arm64  |  4096 |  7811  |  raspberry-pi-1  |  nathan  |
|  ACTIVE|  mi-0fafefe8e517bb56d |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  x86_64 |  8192 |  15675 |  ecs1            |  justin  |
|  ACTIVE|  mi-0bf67a4032bc6eaec |  1.45.0 |  DockerVersion: 19.03.6 |  0 |  arm64  |  4096 |  7811  |  raspberry-pi-2  |  nathan  |
+--------+-----------------------+---------+-------------------------+----+---------+-------+--------+------------------+----------+
```

## General formatting

### Disable pager

If you would like to disable the paged output you can do that with `--no-cli-pager` option, the `AWS_PAGER` environment variable, or the `PAGER` variable.

A command like this will likely be paged locally.
```bash
aws ec2 describe-instances
```

You can get all of the output at once by piping it to another command.
```bash
aws ec2 describe-instances | jq .
```
or disabling the pager
```bash
PAGER='' aws ec2 describe-instances
```

### Making a multi-line list into a comma separated list

There are lots of different ways to format text in a shell.
`tr`, `sed`, and `awk` are very versatile and usable in many situations.
I will try to stick with the most portable and simple option in these examples.

If you have a list that looks like this from some command output

```bash
i-abcdefg
i-1234567
i-zyxwvut
```

> Some `aws` commands require comma separated list and some request space separated lists. If the list is supposed to be space separated you don't need to remove the newlines.

You can transform it into a comma separated list by piping it to

```bash
#TODO this is a bad command
aws describe-instances --instance-ids $(aws ... | tr '[:space:]' ',' | sed 's/,$//')
```

The `tr` command will replace all newlines with commas and the `sed` portion will remove the trailing comma.

If you're using zsh you can make a global alias for this to simplify using it.

```
alias -g CS='| tr -s "[:blank:]" "," | sed "s/,$//"'
```

Now you can save typing with

```
aws ec2 describe-instances \
  --filter "Name=tag:alpha.eksctl.io/cluster-name,Values=stage" \
  --query 'Reservations[].Instances[].InstanceId' \                                                          
  --output text CS
```
> example output
```bash
i-00cea6e9c6ffeb6ef,i-09b019e2af1ddca36,i-0b4a835ac6c46f716,i-0f2944fd98a0e6cee
```

## Helpful tools and scripts

- [awsls](https://github.com/jckuester/awsls)
- [bash-my-aws](https://github.com/bash-my-aws/bash-my-aws)
