## Double Opt-In Mechanism

#### TL;DR

*PHP project to verify email address with confirmation email before adding to blog newsletter campaign.*

### Description

While working as a Software Developer at FreshBooks, I was tasked with building a mechanism to capture email addresses for our blog newsletter drip. The solution I helped imagine involved an email confirmation step so that we would only be sending newsletter material to valid email addresses linked to a real human being.

In the interest of explaining how I approached this problem, I present here **simplified versions of the public methods** of the code, while describing its high-level functionality.

### User flow

1. The user enters his/her email address into a form and clicks submit.
2. The user receives a confirmation email with a link to validate that the email address belongs to a human (not spam).
3. Clicking on the link sends the user to the confirmation screen at which point they have been successfully added to the list of recipients for the email drip campaign.

### How it Works

##### CSV

A csv file stores rows containing email addresses, a unique secret value (random string), and a confirmation value (0 or 1).

```
email_address, 				secret_value, 		is_confirmed
somebody@yahoo.com, 		8xlj2xe, 			0
person2@hotmail.com, 		9zask24, 			1
...
```

The secret value is generated in step 1 of the user flow and stored in the csv, then sent as a query param in the email confirmation link to the user in step 2. If both secrets match, it means that the email used to signup is valid.

##### Public Methods

The code has two public methods. The first, `send()`, corresponds to step 1 and is called when the user submits their email to the system. It generates a secret, stores the secret in a row in a csv file, and sends out the confirmation email.

```php
public function send()
{
	// Get email & source from $_REQUEST
	$email = filter_var($_REQUEST['email'], FILTER_SANITIZE_EMAIL);

	// Generate secret
	$secret = uniqid('');

	// Add new row to CSV, leave 0 in is_confirmed column
	$this->saveSecretAndEmailToCSV($email, $secret);

	$this->sendConfirmationEmail($email, $secret);
}
```

The second method `validate()` is triggered when the user clicks on their confirmation email which includes the secret generated above as a query parameter in the confirmation URL. This is compared against the secret in the CSV file for that user's email, and if there's a match, that means a valid email was used to sign up!

```php
public function validate()
{
	// Get email from SERVER QUERY STRING as a workaround to include '+' character
	preg_match('/email=(.*)&secret=/', $_SERVER['QUERY_STRING'], $matches);
	$email = filter_var($matches[1], FILTER_SANITIZE_EMAIL);

	// Get request secret from SERVER REQUEST
	$request_secret = filter_var($_REQUEST['secret'], FILTER_SANITIZE_STRING);

	// Get secret from file for subscriber
	$csv_secret = $this->readSecretFromCSV($email);

	// Does the secret from the request match the one on file?
	if ($request_secret === $csv_secret) {
		$this->updateConfirmationInCSV($email);
		return true;
	}
	return false;
}
```

Depending on the return value of `validate()`, either a confirmation or error screen should be shown to the user.