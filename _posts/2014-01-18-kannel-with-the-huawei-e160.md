---
layout: post_page
title: Kannel with The Huawei E160
---

I love [Kannel](http://www.kannel.org). Everybody who's used It, loves It. And why wouldn't you? It's libre, it's open and tt's powerful. However, Kannel is hard to configure. Especially if Its your first time using It (let alone it being your first time using Linux). There are just too many variables you can tweak.


### My Use Case

As I had mentioned in another post, I used Kannel to send and receive SMSs from and Android App I've been building for farmers.

> **Note:**

> - I haven't used Kannel in large scale. Therefore, I do not know how well it scales.
> - I only configured my instance of Kannel to send and receive SMSs and not MMSs. Although It is able to do that.


### Lego

Enough chitchat. Let's configure this badboy. I am assuming you have already installed Kannel (this shouldn't be hard).


#### 1. Lock it up

Enough chitchat. Let's configure this badboy. I am assuming you have already installed Kannel (this shouldn't be hard).

 1. Most modems, including the E160, have some sort of storage embedded in them (as an extra). Therefore, if you plug in such a modem it might either mount as a modem or a storage device.
 2. If the modem mounts in 'modem mode', it will probably mount as two (or more) devices in /dev/. In actual sense, one of the devices is the modem's control port and the other is its data port.
 3. It is not guaranteed the modem's data and control ports will be given the same names in /dev/.
 4. Kannel needs to have sufficient permissions to read and write to the modem.

We therefore have to find a way to make sure that; the modem always mounts in 'modem mode' and the the modem's ports are always given the same names in /dev/ every time they are mounted. We can achieve this using udev rules. We also have to make sure Kannel has sufficient permissions to write to the modem's data port.

In Linux, udev rules are in /etc/udev/rules.d/. I already created rules for the Huawei E610 and saved them as [15-huawei-e1550.rules](https://github.com/jasonrogena/kannel_config/blob/master/etc/udev/rules.d/15-huawei-e1550.rules).

The first line uses [usb_modswitch](https://www.archlinux.org/packages/?q=usb_modeswitch) to make sure that the when the modem mounts, it mounts in 'modem mode'. The second and third line give the modem's two ports determinant names. Kannel will be writing into the ttyUSB_utps_pcui device.

After creating the rules and saving the file in /etc/udev/rules.d/ you would want to (re)insert the modem onto your machine. The rules should take effect immediately. You can however confirm if the have by listing what is in /dev/ to see if ttyUSB_utps_pcui and ttyUSB_utps_modem are there.

We now need to add the Kannel user to the group that owns write permissions to ttyUSB_utps_pcui. To confirm if the Kannel user exists, I ran:
     
     cat /etc/passwd | grep kannel
     
Details of the Kannel user will show up if it actually exists.

We now need to find out which group owns write permissions to /dev/ttyUSB_utps_pcui. The following command should do the trick (the second line is the command's result):

    ls -l /dev/ | grep ttyUSB_utps_pcui
    >> crw-rw----  1 root dialout   188,   1 Jan 18 12:40 ttyUSB_utps_pcui

In my case, the 'dialout' group owns the ttyUSB_utps_pcui device. To add the Kannel user into the 'dialout' group, run (as root):

    usermod -a -G dialout kannel


#### 2. Actual Configuration
We can now start the actual configuration of Kannel. All Kannel configuration files are in /etc/kannel/. In my case I have two config files; kannel.conf and modems.conf.

All configurations in Kannel fall under groups. For instance there is a group where all the settings in regards to sending of SMSs fall under. The groups that I configured are:

 1. core 
 2. smsc 
 3. smsbox 
 4. sendsms-user 
 5. sms-service

You can read [Kannel's documentation](http://www.kannel.org/download/1.4.0/userguide-1.4.0/userguide.html) if you require a better explanation of this groups and other groups.


###### core Group

The core group contains all the Kannel admin configurations. My core group looks like this:

    group = core
    admin-port = 13000
	smsbox-port = 13001
	admin-password = p@ssw0rd
	admin-deny-ip = "*.*.*.*"
	admin-allow-ip = "127.0.0.1"
	#wapbox-port = 13002
	#wdp-interface-name = "*"
	log-file = "/var/log/kannel/kannel.log"
	log-level = 0
	box-deny-ip = "*.*.*.*"
	box-allow-ip = "127.0.0.1"

The configurations are pretty straight forward. The only things here that I think need explanation are the admin-port and the smsbox-port. At this point it is important to note that all interaction with the Kannel daemon occur using HTTP. You can now guess that the admin-port is the TCP port through which admin commands can be issued to Kannel. The smsbox-port is the TCP port through which instances of the Kannel module (called smsbox) responsible for sending SMSs connect to the core Kannel daemon (called bearerbox).

Don't worry, further down this post I explain how to interact with Kannel.


###### smsc Group

This group contains settings responsible for controlling the modem and Kannel's interaction with it. My smsc group looks like this:

	group = smsc
	smsc = at
	modemtype = auto
	device = /dev/ttyUSB_utps_pcui
	my-number = +254723572302
	sms-center = +254722500029
	connect-allow-ip = 127.0.0.1
	log-level = 0
	include = /etc/kannel/modems.conf

As you can see, here is where we point Kannel to the device in which it will be talking to (check line 5). We also set the modem's mobile number and [SMS center](http://en.wikipedia.org/wiki/Short_message_service_center) here. I also point Kannel to the second config file in /etc/kannel/ here. The second config file [modems.conf](https://github.com/jasonrogena/kannel_config/blob/master/etc/kannel/modems.conf) contains the different settings for the different well known modems that will make them work well with Kannel.

You can set the log level to 0 for 'debug', 1 for 'info', 2 for 'warning, 3 for 'error' and 4 for 'panic' log levels depending on how you want it.


###### smsbox Group

This group contains configurations for sending SMSs.

	group = smsbox
	bearerbox-host = 127.0.0.1
	sendsms-port = 13013
	global-sender = +254723572302
	log-level = 0

Pretty self-explanatory, huh!?


This group contains configurations that control how users (apps) that want to send SMSs through Kannel interact with It. Mine looks like this:

	#use http://127.0.0.1:13013/cgi-bin/sendsms?username=kannel&password=kannel&text=inserttexthere
	group = sendsms-user
	username = kannel
	password = kannel
	concatenation = true
	max-messages = 1000

The username and password set here are the ones to be used to authenticate a user that wants to send SMSs using Kannel. If concatenation is set to true, messages that are longer that the maximum SMS length, the message will be sent as a [concatenated SMS](http://en.wikipedia.org/wiki/Concatenated_SMS).


###### sms-service Group

This is a group of configurations that control how Kannel dispatches incoming SMSs to apps (users) that are expecting SMSs. You can set it that  all messages from a particular number will be redirected to a certain script. My sms-service group looks like this:

	group = sms-service
	keyword-regex = .*
	catch-all = yes
	max-messages = 0
	#sms-resend-retry = 0
	get-url = "http://localhost/~jason/ngombe_planner/WebServer/php/kannel/sms_router.php?phone=%p&text=%a"

In my configuration, all incoming SMSs are redirected to a PHP script that redirects the message to other PHP scripts depending on the contents of the message. As you can see in 'get-url' config, Kannel interacts with the PHP script by sending to it two GET variables; phone and text. The phone variable holds the sender's number while the text variable holds the message.


#### 3. Interacting with Kannel

If, for instance, I wanted to see the how many SMSs Kannel has sent so far I would do something like:

    lynx http://127.0.0.1:13000/status

As I had mentioned earlier, all interaction with Kannel is done using HTTP. Lynx is text base browser. You will get the same results if you visited the link in the above command in a normal browser like Chrome.

There are some commands to Kannel that might require authentication. For instance:

	lynx http://127.0.0.1:13000/restart?password=bacon

You also send SMSs using HTTP. This means that you can actually have a dedicated SMS server in the office/server room. All services running is other servers will be able to send SMSs by communicating with Kannel in the SMS server like this (here I'm assuming Kannel is running on the same machine, but you already knew that):

	lynx http://127.0.0.1:13013/cgi-bin/sendsms?username=kannel&password=kannel&text=I%20fucking%20love%20bacon

All the files I used to configure Kannel are in this [repo](https://github.com/jasonrogena/kannel_config).
