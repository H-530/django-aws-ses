# 🚀 django-aws-ses

Django email backend that uses [Boto 3](https://boto3.readthedocs.io/en/latest/) to seamlessly interact with [Aws Simple Email Service (SES)](https://aws.amazon.com/ses/).

---

## 📑 Table of Contents

- [📦 Installation](#-installation)
- [🔑 AWS Credential Setup](#-aws-credential-setup)
  - [👤 AWS Named Profile](#-aws-named-profile)
  - [☁️ AWS EC2 Instance Profile](#️-aws-ec2-instance-profile)
- [⚙️ Django Configuration](#️-django-configuration)
- [✉️ Usage](#️-usage)
- [📡 Signals](#-signals)
  - [⚡ pre_send](#-pre_send)
  - [✅ post_send](#-post_send)
- [🧪 Testing](#-testing)

---

## 📦 Installation

First, install the Django Aws SES email backend via pip:

```bash
$ pip install django-aws-ses
```

Next, ensure that your Amazon Web Services (AWS) API credentials are set up, or that your application is running on an Amazon EC2 instance equipped with an instance profile granting access to Amazon SES.

---

## 🔑 AWS Credential Setup

### 👤 AWS Named Profile

Create an AWS API credential profile named `test` using the [AWS CLI](https://aws.amazon.com/cli/):

```bash
$ aws --profile test configure
```

Ensure that the `AWS_PROFILE` environment variable is set so Boto 3 knows which credential profile to load:

```bash
$ AWS_PROFILE="test" gunicorn my:app
```

### ☁️ AWS EC2 Instance Profile

Create an [IAM instance profile](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-create-iam-instance-profile.html) with at least the `ses:SendRawEmail` action. Then, associate it with the EC2 instance(s) running your application. 

An example policy enabling access to `ses:SendRawEmail`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ses:SendRawEmail"],
      "Resource": "*"
    }
  ]
}
```

---

## ⚙️️ Django Configuration

Add or override the `EMAIL_BACKEND` setting inside your Django `settings.py` file:

```python
EMAIL_BACKEND = 'django_aws_ses.EmailBackend'
```

### Optional AWS Settings

If left unset, the backend will gracefully fall back to alternative Boto 3 credential providers.

```python
AWS_ACCESS_KEY_ID = 'my_access_key...'
AWS_SECRET_ACCESS_KEY = 'my_secret...'
```

Set your preferred AWS region (defaults to `'us-east-1'`):

```python
AWS_DEFAULT_REGION = 'eu-west-1'
```

#### Dedicated SES Credentials
Alternatively, you can provide separate credentials specifically for SES:

```python
AWS_SES_ACCESS_KEY_ID = 'my_access_key...'
AWS_SES_SECRET_ACCESS_KEY = 'my_secret...'
AWS_SES_REGION = 'us-west-2'
```

#### Configuration Sets
To enable detailed tracking (such as opens and clicks) via an SES configuration set, specify it below. Learn more in the [AWS Documentation](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/using-configuration-sets.html):

```python
AWS_SES_CONFIGURATION_SET_NAME = 'my_configuration_set'
```

---

## ✉️ Usage

Once configured, use standard Django mail utilities like `send_mail` to dispatch messages via Amazon SES:

```python
from django.core.mail import send_mail

send_mail(
    'Subject here',
    'Here is the message.',
    'from@example.com',
    ['to@example.com'],
    fail_silently=False,
)
```

---

## 📡 Signals

Two signals are provided: `pre_send` and `post_send`. Both receive the message object being processed. The `post_send` signal also receives the Amazon SES message ID.

### ⚡ pre_send

Modify or filter messages before they are dispatched. For example, to exclude blacklisted email addresses:

```python
from django.dispatch.dispatcher import receiver
from django_aws_ses import pre_send

@receiver(pre_send)
def remove_blacklisted_emails(sender, message=None, **kwargs):
    blacklisted_emails = Blacklisted.objects.values_list('email', flat=True)
    message.to = [email for email in message.to if email not in blacklisted_emails]
```

> **Note:** If `pre_send` filters out all recipients, the email is dropped entirely and `post_send` will not fire.

### ✅ post_send

Log successfully dispatched messages or track bounces and complaints:

```python
from django.dispatch.dispatcher import receiver
from django.utils import timezone
from django_aws_ses import post_send

@receiver(post_send)
def log_message(sender, message=None, message_id=None, **kwargs):
    SentMessage.objects.create(
        subject=message.subject,
        body=message.body,
        message_id=message_id,
        date_sent=timezone.now()
    )
```

---

## 🧪 Testing

The test suite is managed via `tox` and mocks all Boto 3 interactions with the Amazon API, meaning valid credentials are not required to run tests:

```bash
$ tox