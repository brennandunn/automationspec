# Manifesto (WIP)

## Why?

We'd like to help develop a standard for the operation of email service providers (ESPs), which have evolved over the years from "email blasts" to sophisticated frameworks for contact management and messaging.

We're not sure where this will go.

Initially, this is a bit of a rallying cry from technically-minded marketers who want the software we use to be better and more accessible. But – who knows – this might eventually develop into a proper framework that can be spun up and attached to any contact datastore, SMTP service, or SMS-sending platform.

Let's start with what you actually get with an ESP.

Most modern platforms effectively do the same thing:

- They store contacts, and these contacts have properties or have performed events
- You can email these contacts. This typically involves taking an email "body" and wrapping it with a template, processing it for each contact – allowing for mutating the final compiled email per properties on each contact
- Emails can be sent as either a broadcast (one off) or be dispatched within automations (as an action)
- Automations are flows that are triggered by some property change or event on a subscriber
- Automations have 3 core components: actions, decisions, and delays
- Automations can be temporal when they include delays and/or "wait until..." goals
- Actions are: updating properties on a subscriber, firing off a webhook, sending an email, dispatching a SMS, ...
- Decisions are: if/else OR switch statements around some data, usually associated with the current contact
- Delays are: steps that wait until a specific thing happens. This might be a time relative to the previous step (+3 hours later) _or_ an awaited mutation on a contact record
- A contact can't exist at two places in an automation

Ultimately, an ESP with marketing automation functionality is really a job queue with decision making capabilities that happens to sometime send emails.

Most email marketing platforms get in the way by layering a limited user interface on top of the retrieving or storing of the content and rules that define a complete system.

These limitations often manifest themselves as:
- Wanting to write an email in Markdown, but needing to put up with a WYSIWYG interface
- An inability to query particular properties or data on a contact due to a crippled templating system
- Coming up with a number of "hacks" – like storing throwaway computed data on a contact record – in order to make a decision node in an automation work the way you want it to

And while ESPs often expand their platform with additional contact intake(i.e. form / landing page builders), ecommerce, and CRM features, being able to deliver "the right message, to the right person, at the right time" is often difficult, or even impossible, to do given the limitations these UIs put on technically-minded marketers.

## Describing the ideal "System"

Let's look at what the ideal framework for setting up a complete email marketing system should look like. The interface, whether it be a code editor or something more involved, is immaterial at this point.

Here's what this framework should permit:

- A totally flexible way to define automations/flows that contacts move through
- The ability to write automated tests that ensure these automations continue to work as expected
- Agnosticism toward how a user-facing message (like an email) gets generated
- Global, contextual, and contact-specific state that can be used to personalize messages and route contacts through automations
- Agnosticism toward how an email message is sent (i.e. Sendgrid, Mailgun, Postmark)
- The ability to quickly define new actions that can be applied to a contact
- The ability to quickly define new properties


## A few (rough) examples

Let's look at a few examples. We'll start with a simple one...

```
./queue broadcasts/your_latest_newsletter.md
```

```markdown
---
subject: Our latest newsletter!
from: Brennan Dunn <me@brennandunn.com>
segment: active_subscribers
---

Hey {{ contact.first_name }},

This is our latest newsletter!
```

This is an example of sending a specific _email_ outside of a flow. This probably shouldn't be the norm.

Let's say you want to do some sort of transformation on every contact that receives this email? And also have it send at their local timezone?

```
./queue broadcasts/your_latest_newsletter.rb
```

```ruby
class YourLatestNewsletter < Flow

  trigger :now
  contacts :active_subscribers

  def perform
  
     ## this will wait until the next time it's 9am for the current contact
     delay '09:00', local: true
     send_email "This is a subject", "This is the body"
     contact.increment! :newsletters_sent
  
  end

end
```

The above is effectively a one-off automation (or "flow") that is triggered immediately (at the time of being run), goes out to all active subscribers, and then waits until 9am local, sends an email, and increments a counter.

Ideally, `trigger` and `contacts` can be set up however you'd like. `contacts` probably only makes sense for flows that are immediately triggered and one-off:

```ruby
class YourLatestNewsletter < Flow

  trigger { Time.parse('2020-12-25') }

```

```ruby
class YourLatestNewsletter < Flow

  trigger :event, { |event| event.type == "purchase" && event.amount >= 100 }

```

```ruby
class YourLatestNewsletter < Flow

  trigger :now
  contacts { |contact| contact.is_customer? }

```

Most flows will likely be evergreen and not something you start manually. To extend the above example, where we're incrementing a `newsletters_sent` property on a recipient contact, let's look at what that flow might look like:

```ruby
class AutoPitcher < Flow

  trigger :property, :newsletters_sent
  
  def perform
  
    ## every ten newsletters, we should pull someone out of our newsletter and send them an automated pitch sequence
    if contact.newsletters_sent % 10 == 0
      contact.should_get_newsletters = false
      delay 3.days
      contact.fire_event! :start_pitch, then: { |c| c.should_get_newsletters = true }
    end
  
  end

end
```

Everything should probably be evented. One flow shouldn't know about the existance of another flow. And this is a model that has worked really well in the software development world for quite a while – specifically, the publish/subscribe pattern.

While this decoupling is nice, we should also have a way for a caller to know when any and all flows that subscribe to an event/property update are complete. In the above case, we want to put someone back on the newsletter when any flow that handles the `start_pitch` wrap up.

Likewise, setting a property on a contact _immediately_ propagates that change everywhere and persists it in the database. There's no need to "save" it.

Setting up the pitch flow is pretty straight forward:

```ruby
class PitchMyCourse < Flow

  trigger :event, :start_pitch
  goal :event, { |event| event.type == "purchased" && event.product == "course" }
  local_time: true
  
  def perform(contact)
  
    if !contact.has_been_pitched_recently? && contact.events(:purchased).where(product: "course").blank?
      delay until: 'Sunday at 6:00pm'
      send_email 'course_pitch/prelaunch'
      delay until: '10:00am'
      contact.pitch_expires = time("Friday at 11:59pm")
      send_email 'course_pitch/sale_open'
      send_sms "Hey! My course is now available. Check your email" if contact.textable?
      if contact.hot_lead?
        delay until: '7:00pm'
        send_email 'course_pitch/flash_discount_for_awesome_contacts'
      end
      delay until '12:00pm'
      ## ...
    end
  
  end

end
```

A `goal` allows you to short-circuit a flow when something happens (like they buy the thing you're pitching.)

When we call `contact`, we're not just querying raw key/value properties or whatnot.

```ruby
class Contact < DatabaseContact

  ## special properties can be defined. In this case, customer_status MUST be one of the following
  property :customer_status, :list, ['trialing', 'active', 'canceled'], allow_blank: true
  
  def customer?
    events.where(:purchased).present?
  end
  
  def first_name
    (self[:first_name] || '').capitalize
  end

end
```


## Authors

- Brennan Dunn ([@brennandunn](https://twitter.com/brennandunn))
