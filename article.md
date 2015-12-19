Dynamically autoscaling a jenkins slave pool on EC2
===================================================

At $lastjob, I configured a cluster of [Jenkins](https://wiki.jenkins-ci.org/display/JENKINS/Meet+Jenkins) slaves on [EC2](https://aws.amazon.com/ec2/) to dynamically autoscale to meet demand during the working day, while scaling down at night and over weekends to cut down on costs.

There is a [plugin](https://wiki.jenkins-ci.org/display/JENKINS/Amazon+EC2+Plugin) that has very similar functionality, but I wasn't aware of its existence at the time and it's not as flexible or as much fun.

There were a few fun and interesting ingredients in this recipe:

* Generating AMIs for the slaves and rolling out slave instances in an autoscaling group (ASG) with [Atlas](https://atlas.hashicorp.com/), [Packer](https://www.packer.io/intro), [Terraform](https://terraform.io/), and [Puppet](https://puppetlabs.com/puppet/what-is-puppet)
* Setting up [Netflix-Skunkworks/dynaslave-plugin](https://github.com/Netflix-Skunkworks/dynaslave-plugin) so slaves register _themselves_ with the Jenkins master
* After regenerating slave AMIs with new configuration, rolling through the ASG so slaves terminate and get replaced with the new configuration while avoiding terminating Jenkins jobs that may be running on the slaves
* Pulling metrics from Jenkins about build executors and job queuing, then pushing them to CloudWatch
* Using Autoscale Lifecycle Hooks to make Autoscaling wait for any Jenkins jobs to finish running on a terminating slave, instead of terminating it immediately
* Tuning the autoscaling to ensure it's cost effective

To anyone unfamiliar with how [EC2 AutoScaling](https://aws.amazon.com/autoscaling/) works, here's a rough and possibly oversimplified explanation: An AutoScaling Group manages a pool of instances on EC2. It will replace unhealthy instances, distribute instances evenly across availability zones (AZ), allow you to use spot instances and of course dynamically grow or shrink the pool. Some of the key configuration parameters on an ASG are: MinSize (minimum instances in the group), MaxSize (maximum instances in the group), DesiredCapacity (how many instances should be running in the group, this is an optional parameter), LaunchConfigurationName (the AMI and configuration of the instances in the pool). These parameters can be changed at any time and AWS will adjust the number of instances appropriately. _Dynamic Autoscaling_ is the automatic manipulation of these parameters by CloudWatch alarms and Autoscaling policies triggered by CloudWatch metrics.

And for anyone unfamiliar with Jenkins and Jenkins slaves. Slaves are simply servers that the master Jenkins instance can execute jobs on. Slaves have "labels" which are just arbitrary user defined tags, jobs can be configured to "execute only on slaves with label _foo_".

Setting the Scene
-----------------

The Jenkins cluster I worked on needed at most about 25 slaves running to avoid queuing and waiting. There were several types of slaves with different labels for different uses, but ~20 of that 25 ran most of the jobs. This is the pool that needed autoscaling; we'll say these slaves have the label *workhorse*. One thing special about these slaves is that they are small instances and they are configured to have only a single build executor each. That is, they only run one job at a time. This is a bit unusual but it simplifies some of these problems so I'll use it in any examples and explanations here.

We had Puppet code for installing and configuring a system to be used as a slave but needed to automate the rest of the process: bringing up an EC2 instance, running [masterless] puppet and some shell scripts on it, generating the AMI, creating a new launch configuration that points to the AMI, updating the Autoscaling group to use this new launch configuration for new instances, then finally terminating all the running instances so that they get replaced by new instances.

Building the Jenkins slaves
---------------------------

Generating an AMI by running Puppet on an EC2 instance is basically a textbook use case for Packer, automating this step was a snap.

Creating a launch configuration and updating the ASG to point at it? Hi Terraform! Once I had the _create_before_destroy_ bits figured out, this was also easy.

I used Atlas to glue it together and fully automate the pipeline for delivering new Jenkins slave AMIs. I could push a button and Atlas would run Packer to generate an AMI, push metadata about the AMI to Atlas's artifact store then kick a terraform _plan_ run to handle the launch config and ASG, using the latest AMI ID fetched from the artifact store. A notification would be sent to our chat asking for the output of the plan to be acknowledged and applied.

Registering newly-build slaves
------------------------------

Not going to go into too much detail here, but an important piece of this puzzle is a mechanism to allow slaves to register themselves with the Jenkins master. We started off using the [Swarm plugin](https://wiki.jenkins-ci.org/display/JENKINS/Swarm+Plugin) but had some problems and switched to [Netflix-Skunkworks/dynaslave-plugin](https://github.com/Netflix-Skunkworks/dynaslave-plugin). Batteries aren't really included with this plugin, and the scripts that come with it are almost pseudocode, they look like shell scripts but they are non-functional examples and maybe even a bit misleading. But it was smooth sailing once I had the scripts squared away. The slaves had a cron job that would check if they were registered with the master, and if not, register themselves.

Next step was getting metrics into CloudWatch that we can use to trigger scaling events. I whipped up a script that polled the Jenkins build queue and counted total/busy/idle executors for each label, and pushed the resulting counts into custom CloudWatch metrics. I also enabled CloudWatch metrics on the ASG so I could track Total/Max/Min/Desired instances in the group. Since CloudWatch only stores metrics at a minimum resolution of 1 minute, the script can run from cron without missing anything.

Upgrading and tearing down slaves
---------------------------------

When we wanted to upgrade the Jenkins slaves, we needed to kill off all the instances in the ASG so they would be replaced by new instances that used the new AMI. I came up with kind of a stupid party trick involving queuing, which took advantage of these slaves happening to have only a single build executor slot each. This needs the [NodeLabel](https://wiki.jenkins-ci.org/display/JENKINS/NodeLabel+Parameter+Plugin) and [Matrix Project](https://wiki.jenkins-ci.org/display/JENKINS/Matrix+Project+Plugin) plugins installed. I created a "suicide" Matrix job that when run, executes on every slave matching the *workhorse* label; it was not much more than a shell build step that executes:

```aws ec2 terminate-instances --instance-ids $(ec2metadata --instance-id)```

Because these slaves do not execute jobs in parallel, it's safe to dispatch this 'suicide' job.

If the slave instances had been configured with >1 build executor, this job could make a call to the Jenkins API on the master asking it to set the slaves build executors to 1 so we could still take advantage of this pattern. If multiple jobs are already running on the slave, they will continue to run in parallel, until the queue is empty, then the suicide job will run safely. In practice, we should probably also update the slave's label to prevent a race condition where a terminating slave begins running a new job.

At this point I had just about everything in place to dynamically autoscale, but there was one tricky problem left- When scaling in/down, AWS doesn't give you really any control over which instances in the ASG will be terminated, you can associate a policy that will make it favor the instance closest to the billing hour, the one with the oldest launch configuration, the best one to terminate to balance instances across AZs, etc. So, how do we stop it from killing a slave that's still running a job?

Autoscaling lifecycle hooks is part of the solution. This allows you to register hooks to run at startup or termination. With the termination hook registered, what happens when autoscaling terminates an instance is:

1. target instance state is set to `terminate:pending` but does not shut down
2. a [Simple Notification Service](https://aws.amazon.com/sns/) (SNS) notification is sent, whose payload contains some data like the instance id that will be terminated
3. the instance will keep running until either a timeout expires or its state is changed to `terminate:continue`, at which point it will actually terminate.

So, I registered the hook, set a timeout to around the maximum time a job on one of these slaves should ever take to run, subscribed a [Simple Queue Service](https://aws.amazon.com/sqs/) (SQS) queue to receive these SNS messages, then I wrote a script. The script runs on the Jenkins master and polls the SQS queue, when it receives a notification, it does a variation of the suicide job: it changes the target slave's label so it will stop accepting new jobs, sets the number of build executors to 1 if it is not already, then it *queues* a job _on the slave by using the NodeLabel plugin_ which changes the slave's state to `terminate:continue` and the slaves get terminated safely.

Finally, we can dynamically autoscale this thing!

Tuning autoscaling
------------------

Getting dynamic autoscaling right is a bit of a nuanced art. Amazon bills by the instance hour and still charges you for an hour if your instance runs for one minute, you can get burned badly and end up with a huge AWS bill if you're not careful. You always want to scale up fast, but scale down slow. What I found to be effective in this situation was scaling up by increasing DesiredInstances if there are jobs queued. And beginning to scale in once IdleExecutors is >3 for 50 minutes.

I set up a CloudWatch graph that had all the metrics for the ASG and the Jenkins slaves in one place--another pleasant side effect of having 1 executor per slave was the effect it had on this graph, it was very easy to visualize and reason about what was going on. For example, I could spot and aim to minimize fluctuations in DesiredInstances that added up to waste, and I could see that someone started up a slave outside of autoscaling because TotalExecutors was 1 higher than DesiredInstances.

Conclusion and lessons learned
------------------------------

Getting these Jenkins slaves to autoscale was quite a bit of work but it was also a great excuse to learn and experiment. It was a success and saved money immediately, it cut the instance-hours on this group of servers in half and settled the debate about the right statically-set number of slaves to use. It was also surprisingly robust despite the number of moving parts, and the repercussions of any part of this autoscaling scheme being broken weren't terrible.

I learned a ton while working on this and will surely be applying that knowledge to other projects in the future. For example, the next time I'm designing a service that consumes from a message queue I'll add a feature: a "suicide" message can be queued so the service can deal with lifecycle hooks. I've already built other "immutable infrastructure" delivery pipelines for other projects with what I learned.
