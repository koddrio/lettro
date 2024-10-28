# Lettro 

Lettro is an email marketing platform with full ownership and control. It is
primarily a CLI-based tool that integrates well with existing editorial,
collaboration and review processes. Lettro is split into several components:

* Core: configuration management, state management, editorial workflow
* Delivery: domains/dns management, sending of the email smtp/etc, block lists
* Audience: subscribers management, segmentation
* Media: uploads and delivery of static assets
* Analytics: tracking clicks and opens

Each component (except core) is served by a specific module. Lettro ships with
default modules for some components, but these can be replaced or extended
to suit your specific needs, for example:

* Change the Delivery module to use Amazon SES or Postmark
* Set the Audience module to use data from a MySQL database, or an external
  Customer Data Platform
* Configure the Media module to upload assets to Amaozn S3 or SFTP
* Use an Analytics module to track opens and clicks

Some modules may interact with each other, for example the Analytics module
will work with the Delivery module to understand bounces, spam complaints
and more. It will also work with the Audience module to provide unsubscribe
rates or engagement rates.

**Lettro is available in private beta.** [Join the waiting](https://forms.gle/BNnUySxEcCRxRaTV9)
list if you'd like to give it a spin.

# Setup

```bash
$ lettro init
```

Lettro initialized successfully, please configure delivery.

```yaml
delivery:
  module: smtp
  username: hi@koddr.io 
  password: hunter42
  host: smtp.google.com
  port: 578
  tls: true
  from_name: Karl Kubelet
  from_email: karl@koddr.io
  reply_to: karl@koddr.io
  bounce: bounce.gmail.com
  rate: 10
```

Okay, now Lettro can send emails, but we need to know who to send them to.
Let's create an audience.

```yaml
audience:
  module: sqlite
  filename: audience.sqlite 
```

Our subscribers.sqlite is empty right now. Let's import subs from elsewhere into
our Lettro audience and tag them too, so we can segment later.

```bash
$ lettro audience import path/to/existing/subs.csv --tag import
$ lettro audience import path/to/to/shoppers.csv --tag ecommerce-customers
```

The Lettro audience also includes information about bounced addresses, and spam
complainers. And can also optionally include open and click rates if an
analytics modules is enabled. This means that the audience module has to work
together with the delivery module, to understand when an email has hard-bounced
or a spam complaint has been received, so we stop sending messages to these
users.

We can manually interact with the Lettro audience too.

```bash
$ lettro audience add hi@mailbob.io
$ lettro audience bounce bye@mailbob.io
$ lettro audience spam nomailpls@mailbob.io
$ lettro audience query --email="*@gmail.com" --tag gmail-users
$ lettro audience get hi@mailbob.io
```

Some audience modules also support hosted subscription forms, REST APIs,
unsubscribe forms, confirmations, one-click unsubscribes and more. These
usually require some sort of hosting and domain (Docker container, Lambda,
etc.) Double-optin can also be part of this setup.

Confirmation emails, double-optin emails, subscription reminders and other
all integrate into Lettro's templating system. The event trail for every
subscriber is kept in the Lettro audience database, integrates with the
analytics module (to record when a user opened an issue or clicked a link),
with the delivery module to track when a user unsubscribed or bounced, etc.

# Creating a campaign

Now that we have our Lettro aduience ready and populated, and a delivery module
up and running, let's create our first Lettro campaign:

```bash
$ lettro create 001-happy-halloween.md
```

This file can be edited using your favorite Markdown or HTML editing software,
can be collaborated on using source code and revision management tools, such
as GitHub. Can be previewed in your browser locally, etc.

When working with Markdown, an HTML is necessary to produce the final output
for an email. By default, this will use Lettro's built-in default HTML
template. We can override this with a core configuration:

```yaml
core:
  template: path/to/template.html
```

You can use variables in your input Markdown content, and the output template
HTML, for example:

```markdown
---
subject: Happy Halloween!
logo_url: https://example.org
---

Your spooky content goes here.
```

These can now be used in the HTML template like so:

{% raw %}
```html
<h1>{{ subject }}</h1>

<img src="{{ logo_url }}">

{{ content }}
```
{% endraw %}

Some audience veriables are also available both in Markdown and HTML/MJML
templates, for example:

{% raw %}
```markdown
Hi, {{ audience:name }}!

This email has been sent to {{ audience:email }}.
To unsubscribe follow this link:

{{ audience:unsubscribe_url }}
```
{% endraw %}

Note that unsubscribe URLs are only available if your Audience module supports
hosted forms. Refer to the module documentation for more details on template
variables.

# Media

Lettro supports multiple formats for campagins: Markdown (with an HTML
template), HTML and MJML. You can reference image assets from the media folder,
for example:

```html
<img src="media/logo.png">
```

If a media module is configured, upon publishing a campagin, this image will
automatically be uploaded to the media asset manager (S3, etc.) and the correct
public link will be embedded in the outgoing or preview email.

Configuring a media module is quite simple, with support for S3, FTP and more:

```yaml
media:
  module: s3
  provider: s3.amazonaws.com
  access_key: something
  secret_key: something-secret
  region: us-east-1
  bucket: my-lettro-media
```

The media module can also be used to archive your campagins, and make them
available via a browser.

# Previewing & Sending

Once you've written your campagin, you can preview in different ways:

```bash
$ lettro preview 001-happy-halloween.md
$ lettro preview 001-happy-halloween.md --template=path/to/different.html
$ lettro preview 001-happy-halloween.md --audience=specific@email.org
```

These will preview the specific campagin locally in a browser window. Note that
browsers display emails differently from email software and services. To send
a preview email we can use:

```bash
$ lettro preview 001-happy-halloween.md --send-to=hi@mailbob.io
```

Once we're happy with the results, we can send the newsletter to our entire
audience, or a segment:

```bash
$ lettro send 001-happy-halloween.md
$ lettro send 001-happy-halloween.md --segment=spooky,creepy
```

Lettro keeps track of specific issues sent to specific recipients, and will
attempt to never send the same issue to the same recipient twice.

Depending on the delivery module configuration, the sending may be synchronous,
and could be interrupted by Ctrl+C, a crash, Internet problems and other
issues. You can check the status of an existing campagin, and resume if
necessary:

```bash
$ lettro send 001-happy-halloween.md --status
$ lettro send 001-happy-halloween.md
```

# Analytics

The Analytics module in Lettro can be configured to use a third-party (or your
own) redirect service, to track opens and clicks. The module may also integrate
with the Audience module to provide unsubscribe rates, and the Delivery module
to provide bounce rates and spam complaints. Finally, the analytics module can
also provide A/B testing for subject lines and content across your audience.

```yaml
analytics:
  module: lettro-metrics
  api_key: secret
  domain: click.example.org
```

Configuring this will automatically embed a 1x1 pixel image in all outgoing
emails, allowing Lettro to capture the open rate. All links will also be
replaces with tracked URLs for click tracking. After a campagin is sent, we
can check the performance with:

```bash
$ lettro analytics 001-happy-halloween.md
```

The module also provides per-user analytics, so we can check a subscriber's
full engagement history using:

```bash
$ lettro analytics --audience=hi@mailbob.io
```

The Analytics module extends the content variables to support A/B testing
for subject lines and content. For example:

{% raw %}
```markdown
---
variants:
- happy
- spooky
- dare

subject!happy: Happy Halloween!
subject!spooky: Spooky Boo!
subject!dare: I DARE you to open this!
---

{% if variant == "happy" %}
Hello and Happy Halloween!
{% elif variant == "spooky" %}
Spooky, no?
{% elif variant == "dare" %}
You opened it!
{% endif %}
```
{% endraw %}

Recipients will get randomly assigned a variant, and analytics can then
be viewed by variant:

```bash
$ lettro analytics 001-happy-halloween.md --by-variant
```

You will also be able to see which user was assigned which variant in the
user engagement history. These metrics are per-campagin.

