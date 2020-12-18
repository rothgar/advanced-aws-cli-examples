# A collection of advanced AWS CLI commands and queries

This repo is a collection of hard won [AWS CLI](https://aws.amazon.com/cli/) commands that can be used as starting points for your own use cases.

If you've spent 2 hours crafting the perfect command there's a good chance it could help someone else too!
Consider [submitting a PR](https://github.com/rothgar/advanced-aws-cli-examples/pulls) to help the next person.

## General shell formatting

There are lots of different ways to format text in a shell.
`tr`, `sed`, and `awk` are very versatile and usable in many situations.
I will try to stick with the most portable and simple option in these examples.

### Making a multi-line list into a comma separated list

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
alias -g CS='| tr '[:space:]' ',' | sed 's/,$//')'
```

Now you can do

```
aws ... CS
```

## Advanced Queries

### Print ec2 instances based on tag

If you want to print a all ec2 instances that have a specific tag you can use a query like the following.

```bash
aws ec2 describe-instances --query 'Instances[*].Tag[Key]
```

You can further reduce the output to the `InstanceId` with the following command.

```bash

```

### Selectivly match based on field value

If you want to print a field based on the value of another field in the same resource you can use the `[?]` operator.

Let's say you want to print out a VPC based on a security group `GroupName`.

Your json looks like this for `aws ec2 describe-security-groups`

```json

```

You can print the VpcId for the `super-secure` security group with

```bash
aws ec2 describe-security-groups --query 'SecurityGroups[?GroupName==`super-secure`].VpcId'
```

### Print multiple fields

If you would like to only print select fields you can separate the fields with commas.

```bash
aws ec2 describe-instances --query 'Instances[*].
```

### Print multiple named fields

If you want to print multiple fields but also want to set the name of what the field should be called (e.g. for table output) you can used
`[name: key]` syntax and also comma separate multiple fields like `[name: key,interfaces: TODO]`

```bash

```

### Nesting command output

You often want to look up values for something that requires information from another command output.
In this case you can use standard shell value substitution with `$()`.

Make sure your subshell command is only printing the text you want (without headers and in the correct text or json format).

```bash
aws TODO
```

Now you can simply plug in that command into a larger command like so.

> If the correct subshell command was the _last_ command you ran you can use `$(!!)` as a shorthand for replacing `!!` for the last command you ran. You can see more advanced command history usage in my [mastering zsh workshop](https://github.com/rothgar/mastering-zsh/blob/master/docs/config/history.md#using-words-from-the-last-command)

```bash

```

## Helpful tools and scripts

- [awsls](https://github.com/jckuester/awsls)
- [bash-my-aws](https://github.com/bash-my-aws/bash-my-aws)