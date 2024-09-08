---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x - Minimal Rails -- no Database"
subtitle: "Run a minimal Rails without a database on a SMALL machine"
summary: ""
authors: ['btihen']
tags: ['Rails', "Minimal", "No Database"]
categories: ["Code", "Minimal", "Ruby Language", "Rails Framework"]
date: 2024-05-06T01:20:00+02:00
lastmod: 2024-05-21T01:20:00+02:00
featured: true
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

## Overview

I was recently asked how to create a rails site without a database (in order to run on a very small footprint).

I thought that was an interesting question, so here is my exploration.  In oder to make the site dynamic, this article creates a landing page and an email form.

## Create Rails

```ruby
# active storage will install even without active record - so we MUST not allow that either

rails new nodb --skip-active-storage --skip-active-record   --css=bootstrap --javascript=esbuild

cd nodb

# be sure it starts without error (no migrations needed) and the default webpage shows up.
bin/dev
```

Assuming that works lets commit:
```bash
git add .
git commit -m 'initial rails no db'
```

## Landing page

Let's start by replacing the landing page with:
```bash
bin/rails g controller Landing index
```

update the routes with:
`get 'landing/index'`
and
`root "landing#index"`

So now the config/routes.rb would look like:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'landing/index'

  # Defines the root path route ("/")
  root "landing#index"
end
```

Now if you type `bin/rails routes | grep landing` you should see:
```bash
   Prefix      Verb      URI Pattern             Controller#Action
landing_index  GET  /landing/index(.:format)      landing#index
```

Let's test that our new landing page is the new 'ugly' page - beautification is left to the reader. Assuming this works lets commit.

```bash
git add .
git commit 'created custom landing page'
```

## Add an Email form

So to add a form we need a model and then generate a mailer. Without a database backend we need build a model that acts like and communicates like it has a database - we do this by including `ActiveModel` components:

### Create a contact model

To add a model we can still use the generators and then just make adjustments.  We include the desired attributes and types so that the view form will be created (without much effort on our part).

```bash
bin/rails g scaffold contact name email subject message:text
```

Note this will NOT actually create a model, every thing except the model. We will make the model by hand, however, we do need to update the controller - which assumes we will have a persisted state and adds methods we won't/can't use.

### Update the Controller

Within the controller be sure to remove the `set_contact`, `show`, `update`and `delete` methods.  All these methods assumes we have a stored contact.id (which we won't have without a DB)!

because the controller can't use show: we also need to adjust the `create` method to redirect to either `new`, `edit` or `/` depending on your preference and users.

So now the controller should look like:
```ruby
class ContactsController < ApplicationController
  def new
    @contact = Contact.new
  end

  def edit
  end

  def create
    @contact = Contact.new(contact_params)

    respond_to do |format|
      if @contact.valid?
        format.html { redirect_to new_contact_path, notice: "Message was successfully sent." }
        format.json { render :new, status: :created, location: @contact }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @contact.errors, status: :unprocessable_entity }
      end
    end
  end

  private

	# Only allow a list of trusted parameters through.
	def contact_params
		params.require(:contact).permit(:name, :email, :subject, :message)
	end
end
```

### Update the Routes

Now that we have made changes to the controller, let's update the routes to match the controller.

before trying our new form, by changing
`resources :contacts`
to
`resources :contacts, only: %i[new edit create]`

So now the routes should look like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'landing/index'
  resources :contacts, only: %i[new edit create]

  # Defines the root path route ("/")
  root "landing#index"
end
```

now we can run `bin/rails routes | grep contacts` - we should see the following:
```bash
Prefix       Verb URI Pattern                  Controller#Action
contacts     POST /contacts(.:format)          contacts#create
new_contact  GET  /contacts/new(.:format)      contacts#new
edit_contact GET  /contacts/:id/edit(.:format) contacts#edit
```

### Create the Model

Now let's make the model by hand.  Using 'virtual attributes' which are usually used in coordination with a normal model, where we want a form to contain additional information, but will not be stored in association with a model.

So we will add a file where models usually go:
```ruby
# app/models/contact.rb
class Contact
end
```

Now we need to include several ActiveModel components - to allow us to build our model.
* `ActiveModel::Model` - allows the model to work with views, etc.
* `ActiveModel::Attributes` - allows the views and the controller to see the attributes (& their types)
* `ActiveModel::Validations` - allows validation to be included within the model

So now our model will look like:
```ruby
# app/models/contact.rb
class Contact
	include ActiveModel
	include ActiveModel::Attributes
	include ActiveModel::Validations
end
```

Let's add the attributes and their types that we used in the generator (to create our form).  The module `ActiveModel::Attributes` provides an `attribute` method which looks like:
`attribute :attrib_name, :attrib_typ, default: some_value
The `default: value` part is optional and generally not used, but can be very important to ensure dangerous actions are manually turned on.

So now our code should look like:
```ruby

# app/models/contact.rb
class Contact
	include ActiveModel
	include ActiveModel::Attributes
	include ActiveModel::Validations

	# Our model attributes to be used by the
	# (note: do not add an `id` since we have no persisted state
	attribute :name, :string
	attribute :email, :string
	attribute :subject, :string
	attribute :message, :string
end
```

Now since we have included `ActiveModel::Validations` we can add some validations to ensure we get all the info we need.

We also need to override the `persisted?` method since this will otherwise try to access the `id` attribute and we know we cannot ever have a persisted record without a db.

So now, finally, our model should look like:

```ruby
# app/models/contact.rb
class Contact
	include ActiveModel
	include ActiveModel::Attributes
	include ActiveModel::Validations

	# Our model attributes to be used by the
	# (note: do not add an `id` since we have no persisted state
	attribute :name, :string
	attribute :email, :string
	attribute :subject, :string
	attribute :message, :string

	validates :name, :email, :subject, :message, presence: true

	# since we have no DB (id) we need to override the `persisted?` method - in this case it is always false!
	def persisted? = false
end
```

Now we can test the form by going to:
https://localhost:3000/contacts/new

Assuming this works, let's commit:
```bash
git add .
git commit -m 'added a contact model, controller and view (using virtual attributes)'
```

## Configure the Mail Settings

generally you should configure mail not send in dev and test mode and to contact a 'true' mail provide in production mode.  We will just use defaults and fake emails that can't be sent. There are several good tools to help with this,  `mailhog`, `letteropener`, etc.

In **Rails 8** all but production mode should be setup by default.

## Create a Mailer

Let's use the generator:
```bash
bin/rails generate mailer Contact
```

you can see this generated a few files:
```
create  app/mailers/contact_mailer.rb
create    app/views/contact_mailer
```

Let's update these to match our needs.

Let's start with the mailer:
```ruby
class ContactMailer < ApplicationMailer
end
```

Let's add some information and a method to invoke:
```ruby
class ContactMailer < ApplicationMailer
  default from: 'rails_nodb@example.com'

  def contact_email
    @contact = params[:contact]
    mail(to: 'contact_rep@example.com', subject: @contact.subject)
  end
end

```

Now that we have that, we can create a view template - given our method is called `contact_email` will need to name our template similarly: `contact_email.html.erb` and it might look something like:
```html
<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Contact Request Message from NoDB website<h1>
		<h2>Subject: <%= @contact.subject %></h2>
		<h2>From: <%= @contact.name %> (<%= @contact.email %>)</h2>
		<p>has sent the following message:
			<br>
      <%= @contact.message %>.
    </p>
  </body>
</html>
```

and for safety's sake (in case the user can't read the html) we will also make a template called: `contact_email.text.erb`

```txt
Contact Request Message from NoDB website
=========================================

SUBJECT: <%= @contact.subject %>
FROM: <%= @contact.name %> (<%= @contact.email %>)

has sent the following message:

<%= @contact.message %>.
```

To see a preview start rails and go to:
`http://localhost:3000/rails/mailers/contact_mailer/contact_email`

or to see all mail previews go to:
`http://localhost:3000/rails/mailers`


Now we can test this using:
```ruby
bin/rails c
contact = Contact.new(
	name: 'Nyima',
	email: 'nyima@example.com',
	subject: 'Hello',
	message: 'Time for dinner'
)
ContactMailer.with(contact: contact).contact_email.deliver_now
```
you should now see something like:
```bash
ContactMailer.with(contact: contact).contact_email.deliver_now
=> #<Mail::Message:17320, Multipart: true, Headers: <Date: Sun, 08 Sep 2024 11:38:17 +0200>, <From: rails_nodb@example.com>, <To: contact_rep@example.com>, <Message-ID: <66dd708962dab_17421f64-3f@GREM-VPJQF2D52M.mail>>, <Subject: Hello>, <Mime-Version: 1.0>, <Content-Type: multipart/alternative; charset=UTF-8; boundary="--==_mimepart_66dd70896202c_17421f64-4c3">, <Content-Transfer-Encoding: 7bit>>
```

assuming this works lets commit:
```bash
git add .
git commit -m "added a contact mailer"
```

If you are doing any of the mail trapping methods you should be able to read the message.

## Integrating the Mailer into the Controller

Normally, I use `.deliver_later` so that this runs as a background job, but sidekick and most jobs need access to a database, so we will just use: `deliver_now` (which stops everything within the thread to send the mail), but without database, this is a viable option.  So after a valid check add:
`ContactMailer.with(contact: @contact).contact_email.deliver_now`

now the contact controller `create` method should look like:

```ruby
# app/controllers/contacts_controller.rb
  # ...
  def create
    @contact = Contact.new(contact_params)

    respond_to do |format|
      if @contact.valid?
			  ContactMailer.with(contact: @contact).contact_email.deliver_now

        format.html { redirect_to new_contact_path, notice: "Message was successfully sent." }
        format.json { render :new, status: :created, location: @contact }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @contact.errors, status: :unprocessable_entity }
      end
    end
  end
	# ...
```

You can test this by starting rails filling out the form and checking the log files and you should see the email in the logs:
`http://localhost:3000/contacts/new`

and in the logs after submitting you should see:
```log
Started POST "/contacts" for ::1 at 2024-09-08 11:48:27 +0200
Processing by ContactsController#create as TURBO_STREAM
  Parameters: {"authenticity_token"=>"[FILTERED]", "contact"=>{"name"=>"Bill", "email"=>"btihen@gmail.com", "subject"=>"test", "message"=>"message"}, "commit"=>"Create Contact"}
  Rendering layout layouts/mailer.html.erb
  Rendering contact_mailer/contact_email.html.erb within layouts/mailer
  Rendered contact_mailer/contact_email.html.erb within layouts/mailer (Duration: 0.3ms | Allocations: 297)
  Rendered layout layouts/mailer.html.erb (Duration: 0.6ms | Allocations: 558)
  Rendering layout layouts/mailer.text.erb
  Rendering contact_mailer/contact_email.text.erb within layouts/mailer
  Rendered contact_mailer/contact_email.text.erb within layouts/mailer (Duration: 0.1ms | Allocations: 161)
  Rendered layout layouts/mailer.text.erb (Duration: 0.3ms | Allocations: 350)
ContactMailer#contact_email: processed outbound mail in 12.1ms
Delivered mail 66dd72eb8fd29_177c62ea4167ee@GREM-VPJQF2D52M.mail (3.7ms)
Date: Sun, 08 Sep 2024 11:48:27 +0200
From: rails_nodb@example.com
To: contact_rep@example.com
Message-ID: <66dd72eb8fd29_177c62ea4167ee@GREM-VPJQF2D52M.mail>
Subject: test
Mime-Version: 1.0
Content-Type: multipart/alternative;
 boundary="--==_mimepart_66dd72eb8f7c7_177c62ea416695";
 charset=UTF-8
Content-Transfer-Encoding: 7bit


----==_mimepart_66dd72eb8f7c7_177c62ea416695
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

Contact Request Message from NoDB website
=========================================

SUBJECT: test
FROM: Bill (btihen@gmail.com)

has sent the following message:

message.


----==_mimepart_66dd72eb8f7c7_177c62ea416695
Content-Type: text/html;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <style>
      /* Email styles need to be inline */
    </style>
  </head>

  <body>
    <!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Contact Request Message from NoDB website<h1>
                <h2>Subject: test</h2>
                <h2>From: Bill (btihen@gmail.com)</h2>
                <p>has sent the following message:
                        <br>
      message.
    </p>
  </body>
</html>

  </body>
</html>

----==_mimepart_66dd72eb8f7c7_177c62ea416695--

Redirected to http://localhost:3030/
Completed 302 Found in 87ms (Allocations: 58111)


Started GET "/" for ::1 at 2024-09-08 11:48:27 +0200
Processing by ContactsController#new as TURBO_STREAM
  Rendering layout layouts/application.html.erb
  Rendering contacts/new.html.erb within layouts/application
  Rendered contacts/_form.html.erb (Duration: 3.8ms | Allocations: 3976)
  Rendered contacts/new.html.erb within layouts/application (Duration: 4.3ms | Allocations: 4347)
  Rendered layout layouts/application.html.erb (Duration: 29.0ms | Allocations: 20498)
Completed 200 OK in 30ms (Views: 30.0ms | Allocations: 21242)
```

Assuming that worked lets commit:
```bash
git add .
git commit -m "integrated mailer into controller"
```

## Resources

### ActiveModel
* [Rails Guide: Active Model Basics](https://guides.rubyonrails.org/active_model_basics.html)
* [ActiveModel Documentation- Overview](https://api.rubyonrails.org/classes/ActiveModel.html)
* [ActiveModel::Attributes Documentation](https://api.rubyonrails.org/classes/ActiveModel/Attributes.html)
* [ActiveModel::Validations Documentation](https://api.rubyonrails.org/classes/ActiveModel/Validations.html)


### ActionMailer
* [Rails Guide: Action Mailer Basics](https://guides.rubyonrails.org/action_mailer_basics.html)
* [ActionMailer: A Tutorial](https://dev.to/bdavidxyz/action-mailer-a-tutorial-gfd)

### ActionMailer Config

* [Catch your emails in Rails development with Mailhog](https://www.dotruby.com/articles/catch-your-emails-in-rails-development-with-mailhog)
* [LetterOpener](https://github.com/ryanb/letter_opener)
* [MailHog](https://github.com/mailhog/MailHog)
* [Testing Emails in local using Ruby on Rails](https://medium.com/@dilip.bhaidiya/send-emails-from-local-using-ruby-on-rails-a2e87bf275ab)
