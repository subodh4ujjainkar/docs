# How to send emails from php mail function using a SMTP server.

## Introduction
The mail() function in PHP is a part of the core functionalities in PHP. It is very useful in sending emails directly from the script. 

## Prerequisites
1. PHP 4 and above installed
1. A working postfix or sendmail setup
1. A fully qualified domain name (FQDN) and server already authorized to send email on behalf of the domain.

The basic syntax of the mail function is as follows.
```php
	mail(to, subject, message, additional_headers, additional_parameters);
```
| Parameters | Type | Description |
|---|---|---|
| to | Required | Email address of the receiver(s).
| subject | Required | Subject line for the email
| message | Required | Text or html message to be sent |
| headers | Optional | Specifies headers like From, Cc, Bcc seperated by CRLF (\r\n) |
| parameters | Optional | Additional parameters to the sendmail program like envelope sender address and others. |

The __to__ parameter is always a string of a single email address or multiple email addresses seperated by a comma.
For example, all the below variations are acceptable.
1. mail@example.com
1. System Mailer<mail@example.com>
1. mail@example.com, phpmail@example.com
1. System Mailer<mail@example.com>, PHP Mailer<phpmail@example.com>

Here, the examples depict the format to be followed while writing the recipient email address(s). You are free to use any of the formats. Also we have used System Mailer & PHP Mailer as indicators for the name of the recipient which should be replaced by the name or full name of the receiver. 

The __subject__ parameter is also and always a string.

The __message__ parameter is a string of the email body. Each line needs to be seperated by CRLF (\r\n) and a single line should not be greater than 70 characters.

All the input parameters to the mail function are strings, but from PHP 7.2, the __additional_headers__ parameter now also accepts an array.

The __additional_parameters__ parameter is used to pass command-line parameters or flags to the underlying mail server while sending emails. 

To send a simple email we open a text file with __php__ extension as below.

> sudo vim php_mail_test.php

Write the below code in the file.
```php
<?php
	//Original message
	$message = "This is a test email.\nIt is sent using postfix MTA";
	//Lines greater than 70 characters are wrapped.
	$message = wordwrap($message, 70, "\r\n");
	//Send email
	mail("mail@example.com", "Test Subject", $message);
?>
```

Now, save and exit the file. Run the script with the below command
> php php_mail_test.php

Now you should receive an email on __mail@example.com__ mailbox with the message as stated in the above example.

You can also include HTML into message parameter by adding the following to the headers.
```php
$headers = 'MIME-Version: 1.0 '."\r\n";
$headers .= 'Content-type: text/html; charset=iso-8859-1'."\r\n";
```

## Sending mail routed through a SMTP server
PHP mailer uses Simple Mail Transmission Protocol (SMTP) to send mail.
The SMTP mail settings can be configured from php.ini file in the installation folder. Locate the php.ini file and open it for editing.

> sudo vim /etc/php/7.2/

Find the following in this file
```
mail function
```
Locate the parameter SMTP and assign it the name of the smtp server.
```
	SMTP=smtp.example.com
```
Then, uncomment the line smtp_port and assign it the port number used for SMTP on the server.

	smtp_port=25

If the server requires authentication, then add the following lines
	auth_username=mail@example.com
	auth_password=example_password
	
After saving the changes to the ini file, restart apache server for the changes to take effect.

## A simple php mail example is given below.
```php
<?php

	$to="user@example.com";
	$subject="First php mail";
	$mesage="This is a test email sent from php mail function."
	$headers="From: noreply@example.com";
	mail($to, $subject, $message, $headers);
?>
```
