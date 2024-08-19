+++
title = 'Transform AWS Exam Generator Architecture to Open Source Part #5: Authentication and Emailing'
date = 2024-08-12T14:42:01+01:00
draft = false
[cover]
    image = 'img/exam-thumb.png'
tags = ['Knative','Authorization','Authentication','Oathkeeper','Kratos','Kubernetes']

+++


## Introduction

In this article, we’re going to replace the Cognito Service, I choose the Kratos and Oathkeeper from Ory as alternative.

The main functionalities of Congito here, is offering a way to sign in, sign up with or without SSO, email verification and use Sesssion Web Token for Frontend authorization.

Here is the full architecture of authentication and authorization

![ory_arch](/img/kratos-oathkeeper-arch.png)

Don't worry if didnt understand the architecture, we will dig deeper in the next headlines

## Kratos

Kratos is an identity and user management service, we will use it to authenticate user in, verify the email and generate Session Web token to pass it to authorization service later.

One feature that others will consider it a downside is **Kratos** doesn’t come with an UI, but Ory have created a separate [repo](https://github.com/ory/kratos-selfservice-ui-node). The repo has a dockerfile to build the image, it stay to us to create a helm chart.

The installation of kratos is handled by a helm chart, but we need to change few values first:

Kratos needs a database to store its state, so we supply a connection string of our private database, I created the user kratos earlier.

```yaml
config:
    ...
    dsn: postgresql://kratos:kratos@cluster-pg-rw.cnpg-system.svc.cluster.local:5432/kratos
```

For sending emails, a separate service is created alongside the kratos api, to get it working we add the **SMTP** credentials we have, I use mailgun as it provides an easy interface and integration mechanism with python, the **from_address** property here is the address that’s will appear in your inbox when you receive an email.

```yaml
config:
    courier:
      smtp:
        connection_uri: xxxxxxxx
        from_address: no-reply@enkinineveh.space

```

Flows property defines each stage like `login`, `registration`, `verification` and `errors`, because we’re using kratos UI all those pages will be under the root url like registration will be under: /registration.

```yaml
flows:
    error:
        ui_url: https://kratos.enkinineveh.space/error
    login:
        ui_url: https://kratos.enkinineveh.space/login
    verification:
        enabled: true
        ui_url: https://kratos.enkinineveh.space/verification
    registration:
        ui_url: https://kratos.enkinineveh.space/registration
    settings:
        ui_url: https://kratos.enkinineveh.space/settings
```

We said before we need to send a verification email, and prevent unverified users from login by using this hook, so we modify the login to adjust the use case

```yaml
flows:
    login:
        ui_url: https://kratos.enkinineveh.space/login
        after:
          hooks:
              - hook: require_verified_address

```

We will not enable ingress, because kratos-ui requires both services to be deployed under the same host, but unfortunately kratos-api helm chart doesn’t enable this, that’s why we will offload this to kratos-ui chart.

The domain we’re going to host the kratos-ui in is **kratos.enkinineveh.space** and the kratos api under **/app** path, then we pass environment variable to specify **kratos-app** url, **CSRF_COOKIE_NAME** and two random secret values

kratos-ui values.yaml

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.org/rewrites: "serviceName=kratos-public rewrite=/"
  hosts:
    - host: kratos.enkinineveh.space
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kratos-ui-charts
              port:
                number: 3000
        - path: /app
          pathType: Prefix
          backend:
            service:
              name: kratos-public
              port:
                number: 80
  tls:
    - secretName: enkinineveh.space-tls-prod
      hosts:
        - kratos.enkinineveh.space
...

secret:
  name: app-env-secret
env:
  secret:
    CSRF_COOKIE_NAME: "__HOST-kratos.enkinineveh.space"
    COOKIE_SECRET: "e482c93bc1e9eab3cd65105128b0615accecceeff100c541b06b1698bca6a497"
    CSRF_COOKIE_SECRET: "a22736d7cefe107b2bf0a156984d9af35e32ff642290c23e5c054ca10ccde879"
    DANGEROUSLY_DISABLE_SECURE_CSRF_COOKIES: "false"
    KRATOS_PUBLIC_URL: "https://kratos.enkinineveh.space/app"
    KRATOS_ADMIN_URL: "http://kratos-admin.auth"
```

To keep clean seperation, I created a seperate `auth` folder, Now let's deploy both services and test it out:

```yaml
releases:
  - name: kratos
    chart: ory/kratos
    namespace: exam
    values:
      - kratos/kratos-values.yaml
  - name: kratos-ui
    chart: ./kratos-ui-node/charts
    namespace: exam

```

```bash
helmfile apply
```

and here it goes, let’s head into registration to create a user, then if we try to login, it'll prevent us, so we must verify the user first using the verification form.

{{< video src="/videos/kratos-account-verify.mp4" width="640" height="360" type="video/mp4" >}}

Next thing is deploying oathkeeper and sharing this session to authenticate the user.

## Oathkeeper

Oathkeeper has two components API and Proxy, we will pick the proxy service to live in front of the exam-gen and exam-taking UI.

![oathkeeper-arch](https://gruchalski.com/assets/posts/2021-05-20/oathkeeper-proxy-mode.png)

The Oathkeeper Proxy service has an access list to decide which request should be allowed or deny it. For example we can enable a **cookie_session** on url **exam-generate-frontend** that reference exam-generate internal upstream service

oathkeeper values.yaml

```yaml

oathkeeper:
  # -- The ORY Oathkeeper configuration. For a full list of available settings, check:
  #   https://github.com/ory/oathkeeper/blob/master/docs/config.yaml
  accessRules: | 
    [
      {
        "id": "exam-generation-rule",
        "upstream": {
            "url": "http://exam-generation-frontend-charts:8501"
        },
        "match": {
          "url": "https://exam-generate-frontend.enkinineveh.space/<.*>",
          "methods": [
            "GET",
            "POST",
            "OPTIONS",
            "PUT",
            "PATCH"
          ]
        },
        "authenticators": [
          {
            "handler": "cookie_session"
          }
        ],
        "authorizer": {
          "handler": "allow"
        },
        "mutators": [{
          "handler": "header",
          "config": {
              "headers": {
                "X-USER-EMAIL": "{{ print .Extra.identity }}"
              }
          }
        }],
        "errors":[
          {
            "handler":"redirect"
          }
        ]
      },
```

also we saw in previous article that the front-take exam needs the authenticated user’s email.

From the oathkeeper documentation we can leverage that with the “header” mutator, but I tried it many times and couldn't get it to work.
If you have any idea how to make it work, I will be pleased to talk.

```yaml
{
    "id": "exam-taking-rule",
    "upstream": {
        "url": "http://exam-taking-frontend-charts:8501"
    },
    "match": {
        "url": "https://exam-taking-frontend.enkinineveh.space/<.*>",
        "methods": [
            "GET",
            "POST",
            "OPTIONS",
            "PUT",
            "PATCH",
            "HEAD"
        ]
    },
    "authenticators": [
        {
            "handler": "cookie_session"
        }
    ],
    "authorizer": {
        "handler": "allow"
    },
    "mutators": [{
        "handler": "header",
        "config": {
            "headers": {
                "X-USER-EMAIL": "{{ print .Extra.identity }}"
            }
        }
    }],
    "errors": [
        {
            "handler":"redirect"
        }
    ]
}

```

But you may ask yourself if oathkeeper and kratos are two separate services , how can oathkeeper verify the kratos cookie_session ?

Well kratos API provides the URL: `/sessions/whoami` to verify cookies, so when adding the cookie_session authenticator, we pass that url with few extra parameters

```yaml

authenticators:
    cookie_session:
        enabled: true
        config:
            check_session_url: http://kratos-public.exam/sessions/whoami # this reference kratos internal url
            forward_http_headers:
                - Cookie
                - X-USER-EMAIL
            preserve_path: true
            extra_from: "@this"
            # kratos will be configured to put the subject from the IdP here
            subject_from: "identity.traits.email"
            only:
                - ory_kratos_session
```

and one last thing in the configuration side is telling oathkeeper to redirect unauthorised requests to kratos login page and the `return_to_query_param` property is for redirecting user back to UI after successfully login

```yaml
errors:
    handlers:
        redirect:
            enabled: true
            config:
                to: https://kratos.enkinineveh.space/login
                return_to_query_param: "return_to"
                when:
                    - error:
                        - unauthorized
```

and we dont't forget to add the frontend domain into kratos `allowed_return_urls`, so requests cannot get blocked

```yaml
selfservice:
    allowed_return_urls:
        - https://kratos.enkinineveh.space/
        - https://exam-taking-frontend.enkinineveh.space/
        - https://exam-generate-frontend.enkinineveh.space/
```

Proxy service will be deployed in front of the UI apps, so we update the ingress of the both frontend apps to redirect traffic to oathkeeper-proxy dns and othkeeper proxy will decide based on the access rules if the request should be allowed or not.

`exam-taking-app values.yaml`

```yaml
hosts:
    - host: exam-taking-frontend.enkinineveh.space
      paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              name: oathkeeper-proxy
              port:
                number: 4455
        - path: /_stcore/stream
          pathType: ImplementationSpecific
          backend:
            service:
              name: exam-taking-frontend-charts
              port:
                number: 8501
```

`exam-generate-app values.yaml`

```yaml
  hosts:
    - host: exam-generate-frontend.enkinineveh.space
      paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
            service:
              name: oathkeeper-proxy
              port:
                number: 4455
        - path: /_stcore/stream
          pathType: ImplementationSpecific
          backend:
            service:
              name: exam-generation-frontend-charts
              port:
                number: 8501
```

here is a demo for authorization

{{< video src="/videos/oathkeeper-login.mp4" width="640" height="360" type="video/mp4" >}}

## After registration & Emailing

### PostSignUp Function

We nearly implement all the functionalities Cognito provides in this architecture but there is one missing feature, which is calling a `PostSignUp` function after registration to subscribe the educator to an Amazon Simple Notification Service , but in our case, we call `PostSignUp` to add a educator email in `subscribers` table so then the `email-fn` retrieve the email to send notifications to.

We start by creating the function, it should serve two main roles: intercepting the event and persisting the email inside a dynamodb table.
The handler function will get the email from `event['traits']['email']` as this the format kratos webhook use, then we created a mongo client and inserted the email into `MONGO_TABLE_NAME` .

So here the code of  `main.py`

```python
import os
from pymongo import MongoClient

def create_subscriber(email:str) -> list[str]:
    # Initialize DynamoDB table
    table_name = os.getenv("MONGO_TABLE_NAME")
    client = MongoClient(os.getenv("MONGO_URI"))
    db = client["exams"]
    collection = db[table_name]
    subscribers = collection.insert_one({"email":email})
    return subscribers

def handler(event,context):
    email = event['traits']['email']
    create_subscriber(email)

```

Wrap it inside FastAPI, build the image and push it into the registry.
Then we reference the image and add the environment variables to `values.yaml` as always:

```yaml

image:
  repository: gitea.enkinineveh.space/gitea_admin/post-signup-fn
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"

env:
  normal:
    MONGO_URI: "mongodb://databaseAdmin:sHWKYbXRalmNExTMiYr@my-cluster-name-rs0.mongo.svc.cluster.local/admin?replicaSet=rs0&ssl=false"
    MONGO_TABLE_NAME: "subscribers"
```

After deploying the chart, we retrive the service URL:

```bash
kubectl get kservice -n exam post-signup-fn-charts

NAME                    URL                                                   LATESTCREATED                 LATESTREADY                   READY   REASON
post-signup-fn-charts   http://post-signup-fn-charts.exam.svc.cluster.local   post-signup-fn-charts-00001   post-signup-fn-charts-00001   True

```

The other part of the puzzle is configuring kratos webhook to send **educator only** email to this functions, but how shall we identity educator emails from the student ones ?.

The answer to this question is adding another choice field contains user types, either: **educator** or **student**.
To achieve this, we will leverage the "custom identity schema" kratos provides, we already implemented the concept in previous step but we didn't talk about it.

Identity schema is the form you're seeing when you access the registration page, by default it comes with just email, password, fullanme , but we can customize it, to do that we change `identity.schemas` property. one side note is the identity schema use [json-schema](https://json-schema.org) to create and validate forms

```json
{
  "$id": "https://schemas.ory.sh/presets/kratos/identity.email.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Person",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "title": "E-Mail",
          "ory.sh/kratos": {
            "credentials": {
              "password": {
                "identifier": true
              }
            },
            "recovery": {
              "via": "email"
            },
            "verification": {
              "via": "email"
            }
          }
        },
        "fullname":{
          "type":"string",
          "title": "Full Name"
        },
        "phone_number":{
          "type":"number",
          "title":"Phone Number"
        },
        "user_type": { # we add this field
          "title":"User Type",
          "type": "string",
          "enum": ["student", "educator"],
          "default": "student"
        }
      },
      "required": [
        "email",
        "fullname",
        "phone_number",
        "user_type"
      ],
      "additionalProperties": false
    }
  }
}
```

Now we encode(base64) the following schema and pass it to `identity.schemas`:

```yaml
identity:
    default_schema_id: default
    schemas:
    - id: default
        url: base64://ew0KICAiJGlkIjogImh0dHBzOi8vc2NoZW1hcy5vcnkuc2gvcHJlc2V0cy9rcmF0b3MvaWRlbnRpdHkuZW1haWwuc2NoZW1hLmpzb24iLA0KICAiJHNjaGVtYSI6ICJodHRwOi8vanNvbi1zY2hlbWEub3JnL2RyYWZ0LTA3L3NjaGVtYSMiLA0KICAidGl0bGUiOiAiUGVyc29uIiwNCiAgInR5cGUiOiAib2JqZWN0IiwNCiAgInByb3BlcnRpZXMiOiB7DQogICAgInRyYWl0cyI6IHsNCiAgICAgICJ0eXBlIjogIm9iamVjdCIsDQogICAgICAicHJvcGVydGllcyI6IHsNCiAgICAgICAgImVtYWlsIjogew0KICAgICAgICAgICJ0eXBlIjogInN0cmluZyIsDQogICAgICAgICAgImZvcm1hdCI6ICJlbWFpbCIsDQogICAgICAgICAgInRpdGxlIjogIkUtTWFpbCIsDQogICAgICAgICAgIm9yeS5zaC9rcmF0b3MiOiB7DQogICAgICAgICAgICAiY3JlZGVudGlhbHMiOiB7DQogICAgICAgICAgICAgICJwYXNzd29yZCI6IHsNCiAgICAgICAgICAgICAgICAiaWRlbnRpZmllciI6IHRydWUNCiAgICAgICAgICAgICAgfQ0KICAgICAgICAgICAgfSwNCiAgICAgICAgICAgICJyZWNvdmVyeSI6IHsNCiAgICAgICAgICAgICAgInZpYSI6ICJlbWFpbCINCiAgICAgICAgICAgIH0sDQogICAgICAgICAgICAidmVyaWZpY2F0aW9uIjogew0KICAgICAgICAgICAgICAidmlhIjogImVtYWlsIg0KICAgICAgICAgICAgfQ0KICAgICAgICAgIH0NCiAgICAgICAgfSwNCiAgICAgICAgImZ1bGxuYW1lIjp7DQogICAgICAgICAgInR5cGUiOiJzdHJpbmciLA0KICAgICAgICAgICJ0aXRsZSI6ICJGdWxsIE5hbWUiDQogICAgICAgIH0sDQogICAgICAgICJwaG9uZV9udW1iZXIiOnsNCiAgICAgICAgICAidHlwZSI6Im51bWJlciIsDQogICAgICAgICAgInRpdGxlIjoiUGhvbmUgTnVtYmVyIg0KICAgICAgICB9LA0KICAgICAgICAidXNlcl90eXBlIjogew0KICAgICAgICAgICJ0aXRsZSI6IlVzZXIgVHlwZSIsDQogICAgICAgICAgInR5cGUiOiAic3RyaW5nIiwNCiAgICAgICAgICAiZW51bSI6IFsic3R1ZGVudCIsICJlZHVjYXRvciJdLCANCiAgICAgICAgICAiZGVmYXVsdCI6ICJzdHVkZW50Ig0KICAgICAgICB9DQogICAgICB9LA0KICAgICAgInJlcXVpcmVkIjogWw0KICAgICAgICAiZW1haWwiLA0KICAgICAgICAiZnVsbG5hbWUiLA0KICAgICAgICAicGhvbmVfbnVtYmVyIiwidXNlcl90eXBlIg0KICAgICAgXSwNCiAgICAgICJhZGRpdGlvbmFsUHJvcGVydGllcyI6IGZhbHNlDQogICAgfQ0KICB9DQp9

```

here is the new form:

![kratos-new-registration-form](/img/kratos-registration-form.png)

Getting back to the webhook configuration, we will set an after registration webhook

```yaml
registration:
    ui_url: https://kratos.enkinineveh.space/registration
    after:
      hooks:
        - hook: web_hook
          config:
            url: http://post-signup-fn-charts.exam.svc.cluster.local
            body: base64://empty_for_now
            method: "POST"
            can_interrupt: false
            emit_analytics_event: false

```

you may notice we didn't supply the body, based on the documentation, kratos use jsonnet to define the logic. 

The code will intercept the event and pass only educator emails

```jsonnet
function(ctx)
if ctx.identity.traits.user_type == "student" then
  error "cancel"
else
  {
    "traits": {
      "email": ctx.identity.traits.email
    }
  }
```

Now we encode it as base64 and pass it to the body param

```yaml
config:
    url: http://post-signup-fn-charts.exam.svc.cluster.local
    body: base64://ZnVuY3Rpb24oY3R4KQ0KaWYgY3R4LmlkZW50aXR5LnRyYWl0cy51c2VyX3R5cGUgPT0gInN0dWRlbnQiIHRoZW4NCiAgZXJyb3IgImNhbmNlbCINCmVsc2UNCiAgew0KICAgICJ0cmFpdHMiOiB7DQogICAgICAiZW1haWwiOiBjdHguaWRlbnRpdHkudHJhaXRzLmVtYWlsDQogICAgfQ0KICB9
```

We update the release and test the registration process for an educator account. if the registration ran successfully we will see a document has been created on subscribers table.

{{< video src="/videos/kratos_webhook.mp4" width="640" height="360" type="video/mp4" >}}

### Sending Email

Can you guess the last missing element of the puzzle ? It's the emailing part, Sending an email when the exam is generated or the student has answered the questions. the knative-sevice will get triggered whenever a message reaches the `email-topic` we created on the previous steps.

the `main.py` in the knative service will be composed of 2 functions:

- get_subscribers: to retrieve emails from the `subscribers` table.
- send_email: will decode the event message and send it to the list of emails we have

so here is the code

```python
from email.mime.text import MIMEText
from email.message import EmailMessage
import smtplib
from requests import HTTPError
import requests
from pymongo import MongoClient
import os


API_KEY = os.environ['MAILGUN_API_KEY']

def get_subscribers() -> list[str]:
    # Initialize DynamoDB table
    table_name = os.getenv("MONGO_TABLE_NAME")
    client = MongoClient(os.getenv("MONGO_URI"))
    db = client["exams"]
    collection = db[table_name]
    subscribers = collection.find()
    return list(e['email'] for e in subscribers )


def send_mail(subs,subject,body):
  requests.post(
        "https://api.mailgun.net/v3/xxxxxxx.mailgun.org/messages",
        auth=("api", API_KEY),
        data={
            "from": "Excited User <no-reply@enkinineveh.space>",
            "to": subs,
            "subject": subject,
            "text": body,
        },
    )

def main(event, context):
    body = event.decode()
    subs = get_subscribers()
    send_mail(subs,"HELLO", body)
```

finally build and push the image and the reference it in `values.yaml` in addtiontion to passing the enviornment variables.

```yaml
env:
  normal:
    BOOTSTRAP_SERVER: kafka-cluster-kafka-bootstrap.strimzi.svc:9092
    MAILGUN_API_KEY: xxxxx
    MONGO_TABLE_NAME: subscribers
    MONGO_URI: "mongodb://databaseAdmin:sHWKYbXRalmNExTMiYr@my-cluster-name-rs0.mongo.svc.cluster.local/admin?replicaSet=rs0&ssl=false"
    TOPIC_ARN: email-topic
```

We clearly done setting up everything, I will test the whole process from registring user and receiving verification code, generating an exam and receiving the email when it's ready, answering the questions and getting the scoreboard mail.

{{< video src="/videos/exam-demo-2.mp4" width="640" height="360" type="video/mp4" >}}


## Summary

and That's it, we transformed the architecture into a pure open source project.

