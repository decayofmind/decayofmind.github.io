---
title:  "Integrate Slack and Icinga2"
layout: post
date:   2015-09-01 10:00:00
tag:
- icinga2
- slack
- monitoring
blog: true
---

This was originally posted by me on [Glogster dev blog](https://glogster.github.io/posts/2015/09/01/slack-with-icinga2.html).

Since we've been using [Slack](http://slack.com) for communication inside our team, I've
started to play with different integrations. There are two ways to integrate
Slack with Nagios-based monitoring systems. The first one is to use general
connectors and write a [custom script](http://matthewcmcmillan.blogspot.cz/2013/12/simple-way-to-integrate-nagios-with.html).
The second one is to use the built-in Slack bot for Nagios. This option is better, because
it gives a better integration level.  In Slack, the documentation only describes how to setup this bot for Nagios or Icinga1 and there is nothing about Icinga2.

So, to setup Nagios integration follow these steps:

* For CentOS or RHEL system we first need to install some requirements:

```
yum install perl-libwww-perl perl-Crypt-SSLeay perl-LWP-Protocol-https
```

* Download this plugin:

{% highlight text %}
wget https://raw.github.com/tinyspeck/services-examples/master/nagios.pl -O /etc/icinga2/scripts/slack_nagios.pl
chmod 755 /etc/icinga2/scripts/slack_nagios.pl
{% endhighlight %}

* Edit `slack_nagios.pl` and add proper values to the `$opt_domain` and `$opt_token` variables.
* Edit `/etc/icinga2/conf.d/commands.conf`:

{% highlight text %}
object NotificationCommand "slack-host-notification" {
  import "plugin-notification-command"

  command = [ SysconfDir + "/icinga2/scripts/slack_nagios.pl" ]
  arguments = {
    "-field" = "$user.vars.host_fields$"

  }
}


object NotificationCommand "slack-service-notification" {
  import "plugin-notification-command"

  command = [ SysconfDir + "/icinga2/scripts/slack_nagios.pl" ]
  arguments = {
    "-field" = "$user.vars.service_fields$"

  }
}
{% endhighlight %}

If you pass env variables with this command, Icinga2 will not propagate it to
the script, so you must pass everything with `-field` key. If you only pass the
`slack_channel` value to the script, Nagios bot will post only empty messages
to the channel.

![](/assets/images/bot_empty.png?raw=true "Empty messages")

If you define multiple entries for `-field` key, Icinga2 will only use the last
one, suppressing the others.
Here you'll need to define an array with those fields.

{% highlight text %}
[2015-08-31 22:36:50 +0200] notice/Process: PID 19582 ('/etc/icinga2/scripts/slack_nagios.pl' '-field' 'NOTIFICATIONTYPE=RECOVERY') terminated with exit code 0
{% endhighlight %}

And surprisingly, you can't define an array in the `NotificationCommand` definition, so something like this
```"-field" = ["a", "b, "c"]```
will give you nothing but errors.

However, you can define an array in another object definition connected to the `NotificationCommand`.
We'll define this variables in User object.

* Edit `/etc/icinga2/conf.d/users.conf` and add this lines:

{% highlight text %}
object User "slack" {
  display_name = "Slack"
  import "generic-user"
  vars.host_fields = ["slack_channel=#ops", "HOSTALIAS=$host.display_name$", "HOSTSTATE=$host.state$", "HOSTOUTPUT=$host.output$", "NOTIFICATIONTYPE=$notification.type$"]
  vars.service_fields = ["slack_channel=#ops", "HOSTALIAS=$host.display_name$", "SERVICEDESC=$service.name$", "SERVICESTATE=$service.state$", "SERVICEOUTPUT=$service.output$", "NOTIFICATIONTYPE=$notification.type$"]
}
{% endhighlight %}

Replace `#ops` with the name of your channel.

* Add following lines to `/etc/icinga2/conf.d/notifications.conf`:

{% highlight text %}
apply Notification "slack-host" to Host {
  import "slack-host-notification"

  users = [ "slack"  ]

  assign where host.vars.sla == "24x7"

}

apply Notification "slack-service" to Service {
  import "slack-service-notification"

  users = [ "slack"  ]

  assign where service.vars.sla == "24x7"

}
{% endhighlight %}

and this to `/etc/icinga2/conf.d/templates.conf`:

{% highlight text %}
template Notification "slack-host-notification" {
  command = "slack-host-notification"

  states = [ Up, Down  ]
  types = [ Problem, Acknowledgement, Recovery, Custom,
            FlappingStart, FlappingEnd,
            DowntimeStart, DowntimeEnd, DowntimeRemoved  ]

  period = "24x7"

}

template Notification "slack-service-notification" {
  command = "slack-service-notification"

  states = [ OK, Warning, Critical, Unknown  ]
  types = [ Problem, Acknowledgement, Recovery, Custom,
            FlappingStart, FlappingEnd,
            DowntimeStart, DowntimeEnd, DowntimeRemoved  ]

  period = "24x7"

}
{% endhighlight %}

* Validate your configs with `icinga2 daemon -C` and restart/reload the service `systemctl reload icinga2`
* Check that your notifications are working.

{% highlight text %}
[2015-09-01 02:31:11 +0200] notice/Process: PID 12654 ('/etc/icinga2/scripts/slack_nagios.pl' '-field' 'slack_channel=#ops' '-field' 'HOSTALIAS=edu01' '-field' 'SERVICEDESC=load' '-field' 'SERVICESTATE=OK' '-field' 'SERVICEOUTPUT=OK - load average: 4.72, 5.71, 7.90' '-field' 'NOTIFICATIONTYPE=RECOVERY') terminated with exit code 0
{% endhighlight %}

![](/assets/images/bot_working.png?raw=true "Correct bot output")

* Don't forget to enable the Nagios bot in Slack for your team.

Enjoy your teamwork!
