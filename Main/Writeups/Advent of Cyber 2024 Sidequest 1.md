# Advent of Cyber 2024 Sidequest 1

**Tags**: [[Cybersecurity]] [[Tech]] [[Writeup]]

# Summary

Completed a **Hard room** in [[TryHackMe]] by analyzing a network packet capture file, figuring out the attacker's methodology and providing the necessary answers for the room completion.

# Context

 [The Advent of Cyber 2024 Sidequest](https://tryhackme.com/r/room/adventofcyber24sidequest) is a [THM](https://tryhackme.com/) hidden challenge related to the [The Advent of Cyber 2024](https://tryhackme.com/r/room/adventofcyber24). that range from hard to insane difficulty in [THM](https://tryhackme.com/) standards. 

# Tasks

## Main Task

The main task was to analyze a .pcap file that recorded traffic of an attacker gaining access to a victim server and exfiltrating credentials. Then, figure out what credentials the attacker stole and use it to answer the questions.

# What I Did

## Analyze the .pcap file

Using [[Wireshark]] I opened the .pcap file. I did a quick overview of the requests and the protocols then familiarized myself with the attacker's and the victim's ips.

I noticed that most of the traffic was encrypted.

## Filter for HTTP traffic

I Found a lot of HTTP traffic. For most of these traffic the server sent a 404. Taking a closer look at the 404 packets the [[User-Agent]] header was [[gobuster]]. This meant two things:

1. These requests were made by the attacker trying to find directories.
2. These requests would not provide any information to me.

## Filter out 404 responses

```
http.response.code != 404
```

The attacker first created a normal user account. Exploited some vulnerability (couldn't figure what) and gained the credentials to the **admin** account for the web server. Then they gained a [[shell]] somehow, I couldn't figure this out either but this was out of scope for the task.

After the attacker gained a foothold they spun up a [[SimpleHTTPServer]] on their device and then made requests to it. These packets showed that the attacker transferred 2 binary files to the victim server using [[wget]], **ff** and **exp_file_credentials**. The attacker had also transferred something through port 9002. 

## Follow Port 9002's TCP stream

Following the [[TCP]] stream it was evident that the attacker transferred a **zip** file. 

## Export the raw bytes for this stream 

I tried unzipping the file but it was password protected.

## Try cracking the zip hash

I used [[zip2john]] before trying a dictionary attack with [[john]]. This failed for various wordlists, the zip file obviously had a strong and random password.

## Check other avenues

Next, I decided to check the binaries that the attacker transferred to the victim server.

I searched for the **exp_file_credentials** file credentials and I found an **exploit** but it was something unrelated to the tasks I had. 

Then, I downloaded the **ff** file and tried reverse engineering it but it felt like it was just a time sink. But, reverse engineering yielded a **secret key** that the file used. I saved this **key** cause it was interesting. Using the unique strings in the file (Usage help in this case) I searched github code for open source repos. The binary's src was easily found and it was a remote-shell. This repo simply provided a client and a server that had its own "protocol" for encrypting, decrypting and executing cmds. The repo also used a **secret key** for encryption which was what I found earlier.

This backdoor had to be connected to a port so I looked at the packets again and **9001** had a lot of back-and-forth traffic common to remote-shells that [[Wireshark]] didn't identify as encrypted but it was gibberish meaning it was encrypted. This was obviously the remote-shell's port.

## Decrypt the port 9001 traffic

I exported all the data the attacker sent to port 9001 and saved it in a file `packetdata`. Then, I modified the src code of the binary in a way that the server didn't challenge the client for verification and printed the buffer it received after decryption. 

Next I ran the server locally and wrote the `packetfile` into [[nc]] connected to the server.

### In terminal 1

`{shell icon} ./server -p 31337 -s "<SECRET>"`

Where secret is the **key** from earlier.

### In terminal 2

`{shell icon} cat packetfile | nc localhost 31337`

In `Terminal 1` all the data/cmds sent by the attacker was seen in plaintext. The attacker zipped the file through this so the password was clearly visible.

Using the password I opened the zip file and the task was completed as all the questions were answered.