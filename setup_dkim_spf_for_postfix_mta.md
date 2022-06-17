<!--
This article attempts to provide steps to ubuntu users to setup DKIM and SPF to be used with postfix MTA.
-->

# How to Set up SPF and DKIM with Postfix on Ubuntu 18.04?

## Introduction to DKIM
Domain Key Identified Mail (DKIM) is a method of signing electronic emails using public-private key pair. DKIM is used by receiving mail server for identifying authenticity, that they are sent by authorized mail servers on behalf of the domain. It also minimizes the possibility of emails getting marked as SPAM.

## Introduction to SPF
Sender Policy Framework (SPF) record specifies which hosts or IP addresses are allowed to send emails on behalf of a domain. You should allow only your own email server or your ISP’s server to send emails for your domain.

For the user, DKIM and SPF attempts to protect email senders and receivers from spam, spoofing and phishing.

Basically, SPF & DKIM are TXT records of the DNS file but internally they do a lot more!

## Prerequisites
1. Ubuntu Server 16.04 and above.
1. Advanced Packaging Tool (APT)
1. Postfix server installed and configured, as shown in [How to install postfix on Ubuntu](https://github.com/subodh4ujjainkar/pepithon-WTC/blob/master/question_8.md)
1. A non-root user with sudo privileges
1. A registered domain with access to DNS management console.

By default ubuntu provides a powerful command-line tool __apt__ Which downloads and installs required packages from debian repositories.

`Note: We have used example.com as the domain name in this article. You should replace example.com with your registered domain name.`

## Step 1: Create a SPF record in your DNS
In your DNS management interface, create a new TXT record like below. Login to your domain provider like Godaddy, wherever you have registered your domain.
> TXT  @   v=spf1 mx ~all

Here,
1. TXT indicates the record type.
1. @ is entered in the name field
1. v=spf1 stands for SPF record version 1
1. mx ~all means that all hosts listed in mx record are allowed to send email on behalf of example.com and all other hosts are disallowed from sending. Alternatively, __+all__, __-all__,__?all__ can also be used. 

## Step 2: Setup SPF policy agent
Inorder to secure our mail users from forged and spam emails, we need to check the authenticity of all incoming emails on our server. This can be done by checking the SPF record of the incoming emails. 

Install the required packages using the following command.
> sudo apt-get install postfix-policyd-spf-python

On invoking the above command you will receive a confimation prompt, which will start the installation on responding with `y` and dependencies will also be resolved. Finally you will get a success confirmation with the version of the installed package as below.

```
Setting up postfix-policyd-spf-python (1.3.2-1) ...
```
Then, edit the Postfix master process config file in a text editor.
> sudo vim /etc/postfix/master.cf

Adding the following lines at the end of the file tells Postfix to start the SPF policy daemon whenever postfix starts.

```
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf
```

Save and exit the above file. Next, we have to edit the Postfix config file as below.
> sudo vim /etc/postfix/main.cf

Add the following lines to the Postfix config file.
```shell
policyd-spf_time_limit = 3600
smtpd_recipient_restrictions =
   permit_mynetworks,
   permit_sasl_authenticated,
   reject_unauth_destination,
   check_policy_service unix:private/policyd-spf
```

Save and exit the above file. Next, we restart postfix server with the below command.

> sudo service postfix restart

## Step 3: Testing our configuration
When receiving email, the raw headers will have a `Received-SPF: Pass` result. This means, the sending server is authorized and our setup is working.

When sending email from this server you will be able to check if the email passes the SPF check on the receiving end. In Gmail, there is a small arrow at the end of the *to* line inside the mail. If we click the arrow, we can see a callout where two rows are important for identifying the authenticity.
1. "Mailed by" and
1. "Signed by"

![GMAIL SPF Authentication](https://i.imgur.com/1CQiQ3y.png)

## Step 4: DKIM Installation
To setup DKIM we will need the following packages to be installed on our ubuntu system.
1. opendkim
1. opendkim-tools

The two packages can be installed with a single command as below. 
> sudo apt-get install opendkim opendkim-tools

You will be prompted to confirm by entering y/n and if you enter `y` and hit enter you get the below message.
```
The following NEW packages will be installed:
  opendkim opendkim-tools
0 upgraded, 2 newly installed, 0 to remove and 152 not upgraded.
Need to get 0 B/263 kB of archives.
After this operation, 815 kB of additional disk space will be used.
Selecting previously unselected package opendkim.
(Reading database ... 144269 files and directories currently installed.)
Preparing to unpack .../opendkim_2.10.3-3build1_amd64.deb ...
Unpacking opendkim (2.10.3-3build1) ...
Selecting previously unselected package opendkim-tools.
Preparing to unpack .../opendkim-tools_2.10.3-3build1_amd64.deb ...
Unpacking opendkim-tools (2.10.3-3build1) ...
Processing triggers for man-db (2.7.5-1) ...
Processing triggers for systemd (229-4ubuntu21.21) ...
Processing triggers for ureadahead (0.100.0-19) ...
Setting up opendkim (2.10.3-3build1) ...
Setting up opendkim-tools (2.10.3-3build1) ...
Processing triggers for systemd (229-4ubuntu21.21) ...
Processing triggers for ureadahead (0.100.0-19) ...
```
At the end we can see that both the packages are installed and their respective versions.

## Step 5: DKIM Configuration
1. Generate key pair
	We can create DKIM key-pair using the *opendkim-genkey* command. Execute the following commands in order to create the proper folder structure for the keys to be stored.

	> sudo mkdir -p /etc/opendkim/keys/example.com
	>
	> cd /etc/opendkim/keys/example.com
	>
	> sudo opendkim-genkey -t -s mail -d example.com

	This will generate two files, viz., mail.private & mail.txt

1. Add postfix user to opendkim group with the following command

	> sudo gpasswd -a postfix opendkim
	>

   	On success, we get the below response.
	```
	Adding user postfix to group opendkim
	```

1. Edit OpenDKIM main config file /etc/opendkim.conf and make the following changes.

	> sudo vim /etc/opendkim.conf

   1. Uncomment the following lines. If the below lines are not present, add them.
      ```shell
		Canonicalization   simple
		Mode               sv
		SubDomains         no
      ```
   1. Replace _simple_ with _relaxed/simple_
   1. Add the following lines if not present otherwise uncomment them and update values as given below.

      ```shell
		AutoRestart         yes
		AutoRestartRate     10/1M
		Background          yes
		DNSTimeout          5
		SignatureAlgorithm  rsa-sha256
      ```

   1. Append the following lines to the end of the file.

      ```shell
		#Remember to add user postfix to group opendkim
		UserID             opendkim

		#Map domains in From addresses to keys used to sign messages
		KeyTable           refile:/etc/opendkim/key.table
		SigningTable       refile:/etc/opendkim/signing.table

		#Hosts to ignore when verifying signatures
		ExternalIgnoreList  /etc/opendkim/trusted.hosts

		#A set of internal hosts whose mail should be signed
		InternalHosts       /etc/opendkim/trusted.hosts
      ```

1. Create key table, signing table & trusted host file as mentioned in the config above.
	> sudo mkdir /etc/opendkim
	>
	> sudo mkdir /etc/opendkim/keys
	
	Change owner for these files from root to opendkim so that opendkim user can read/write to this directory

	> sudo chown -R opendkim:opendkim /etc/opendkim
	>
	> sudo chmod go-rw /etc/opendkim/keys

	Create the signing table

	> sudo vim /etc/opendkim/signing.table

	Add the below line to this file.
	```shell
	*@example.com		default._domainkey.example.com
	```
	Create the key table file with the following command.
	> sudo vim /etc/opendkim/key.table
	
	Add this line to the file, then save and exit.
	```shell
	default._domainkey.example.com    example.com:default:/etc/opendkim/keys/example.com/mail.private
	```
	Next, create the trusted host file with the following command.
	> sudo vim /etc/opendkim/trusted.hosts

	Add below lines to this file, then save and exit.
	```shell
	127.0.0.1 
	localhost 
	*.example.com
	```
	This specifes that message coming from the above hosts will be signed and trusted.

## Step 6: Add public key to you DNS record
Open the below file and copy the public key from there to your
> sudo cat /etc/opendkim/keys/example.com/mail.txt
>
You will see key value pairs there as below.
```nginx
mail._domainkey	IN	TXT	( "v=DKIM1; k=rsa; t=y; ""p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDoZSIL9CKpJgpPB5LMYLOY+h3kptrwLXn89CAc8pUeMgaeTcsw6b4nUB4t79AIA2hES6k2tX3s/zYIAt2aUnpMciFeKfNcqwtRHUiVgcvykcePNpsv8ERGuCKrE4ferQPrTRRONGyJUprDBvasjolqclAYKEOZQk+NFT7xcNyFEQIDAQAB" )  ; ----- DKIM key mail for example.com
```

The string in `p` parameter without the quotes is the public key. Add this string to the DNS record of example.com by creating a TXT record

## Step 7: Test your configuration
Enter the following command to test your key.
> sudo opendkim-testkey -d example.com -s default -vvv
>
If everything is ok, you will get the below message.
```shell
key OK
```

## Step 8: Connecting Postfix to OpenDKIM
If the above test is successful, then we can connect postfix server to opendkim for seamlessly signing outgoing email with the private key.
Postfix can talk to OpenDKIM via a Unix socket file. 
1. Create a directory to hold the OpenDKIM socket file and only allow opendkim user and postfix group to access it.
> sudo mkdir /var/spool/postfix/opendkim
> sudo chown opendkim:postfix /var/spool/postfix/opendkim

Then edit the socket configuration file.
> sudo vim /etc/default/opendkim
>

1. Find the following line:
	```
	SOCKET="local:/var/run/opendkim/opendkim.sock"
	```
1. and replace it with
	```
	SOCKET="local:/var/spool/postfix/opendkim/opendkim.sock"
	```
1. Next, we need to edit Postfix main configuration file /etc/postfix/main.cf and add the following lines smtpd_recipient_restriction section.
	```shell
	#Milter configuration
	milter_default_action = accept
	milter_protocol = 6
	smtpd_milters = local:/opendkim/opendkim.sock
	non_smtpd_milters = $smtpd_milters
	```
1. Finally, restart postfix and opendkim
	> sudo service postfix restart
	>
	> sudo service opendkim restart

## Conclusion
In this article you configured SPF & DKIM on the email server and added the required authentication records in the DNS manager. Now, the emails sent from this server on behalf of example.com should be authenticated successfully by the recepient servers. This reduces the chances of the email being marked as spam.
