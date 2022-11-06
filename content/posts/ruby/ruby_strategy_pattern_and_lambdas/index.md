---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Strategy Pattern and Lambdas"
subtitle: "Simplify Complex Behavior Decisions"
summary: "With when Controllers need to trigger a chain of complex activities, one approach to keeping the code workable is to use Command Objects, the Command Pattern and even Lambdas"
authors: ["btihen"]
tags: ['Ruby', "strategy pattern", "lambda"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-08-14T01:11:22+02:00
lastmod: 2021-08-14T01:11:22+02:00
featured: true
draft: true

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

## Motivation

Often it is nice to track metadata associated with a model - that is expensive and or complex to lookup - so ideally, we generate this when we save and it is available with each document read.  With a lot of complex data this gets very tricky -- especially when this should be configurable (perhaps with a DSL).

I have found the strategy pattern is very helpful, but can have a lot of boiler-plate code.  So borrowing a little from Functional languages we can make the strategy pattern feel light-weight (Object purist may disagree with this).

To keep the example workable - I will use a very simple model, but I have run into several situations where records with a lot of relationships must be serialized or quickly accessed - including thousands of records which need to then do 7-10 queries and calculations.  This quickly becomes SLOW!

## Example Vet DB

Lets assume each `visit` we have the following types of data (much of it will be associations):

Doctor
Person / Owner (can have multiple pets)
Pet (Cats, Dogs, Lizzards, Fish, Birds, Hamsters, Horses)
Diagnostics (blood lab, saliva lab, phycial tests, etc)
Diagnosis (associated with each species)
Medicines (appropriate for each species)
Treatments (medicines & quantity, non-invasive therapies, surgeries)

At the end of the month we send the visits off to the billing agency - each species, test, etc has a billing code and expense that the billing agency will then send / collect for the vet.

In order to make this simple to serialize and send we will generate the summary info as a json.

## Basic setup

During a visit the Doctor records the visit and a charge will be sent to the billing agency with the following considerations:
* if owner has 2 or more pets treated this year apply `a` discount
* if owner has been a client for 5 years apply `b` discount
* pet type cat, dog, fish, etc -- apply rate
* Diagnostics used - apply charge for each
* Treatments applied - apply charge for each

## Setup

rails new vet_strategy
cd vet_strategy

rails g scaffold Person name
rails g scaffold Diagnostic name
rails g scaffold Treatment name

https://til.hashrocket.com/posts/kaawvv04xh-generate-a-migration-with-polymorphic-association
rails g scaffold Genu name # genuable:references{polymorphic}
class Genu < ApplicationRecord
  belongs_to :genuable, polymorphic: true
end

cat <<EOF>app/models/feline.rb
class Feline < Genu
  # belongs_to :specie, as: :genuable
end
EOF

cat <<EOF>app/models/canine.rb
class Canine < Genu
  # belongs_to :specie, as: :genuable
end
EOF

rails g model GenuDiagnostics genu:references diagnostics:references
rails g model GenuTreatments genu:references treatments:references

rails g scaffold Pet name birth_date:date owner:references genu:references





We had a straight-forward setup
```ruby
# lists other user's (challenger's) submissions
class VisitsController < ApplicationController
  def create
    visit = Visit.new(visits_params)
    # our commands return a truthy or falsy value
    if visit.save
      send_visit_payment_command
      redirect_to visits_path
    else
      flash.now[:error] = "Visit has errors"
    end
  end

  private

  def visits_params
    params.require(:visits) .permit(....)
  end
end
```

```ruby
class SendVisitPaymentCommand

  def new(visit)
    @visit = visit
  end

  def call(visit)
    new(visit).run
  end

  def run
    # if owner has 2 or more pets treated this year apply `a` discount
    # if owner has been a client for 5 years apply `b` discount
    # pet type cat, dog, fish, etc -- apply rate
    # Diagnostics used - apply charge for each
    # Treatments applied - apply charge for each
  end

  private
  attr_reader :invoices_form
end
```

## Getting Complex

Over time with legal changes and added features our logic was getting complex (even convoluted).


I find these are easiest to spot when there a lot of if statements - especially when they are controlling the behavior of another object.

```ruby
class InvoicesCommand

  def new(invoices_form)
    @invoices_form = invoices_form
  end

  def run
    customer_invoices = InvoiceBuilder.generate_invoices(invoices_form)
    send_invoices = if invoices_form[:print_all] && invoices_form[:customer_ids]
                      customer_invoices.select { |ci| ci[:customer].ebanking? && !ci[:invoice].monthly? }
                    elsif invoices_form[:customer_ids]
                      customer_invoices.reject { |ci| ci[:customer].ebanking? && ci[:invoice].monthly? }
                    # ... other user and system choices to filters
                    else
                      customer_invoices.reject { |ci| ci[:customer].ebanking? }  # customers setup knows
                    end
    job_id = SendInvoicesJob.call(send_invoices)
    {success: true, result: job_id}
  rescue StandardError => error
    {success: false, error: error}
  end

  private
  attr_reader :invoices_form

  def select_invoices_to_send(customer_invoices)
    return customer_invoices if invoices_form[:print_all]   # user wants a print copy for all
    return customer_invoices if invoices_form[:cutomer_ids] # user specifically chose these for
    # ... other user and system choices to filters

    # or just the default filter
    customer_invoices.filter { |ci| !ci[:customer].ebanking? }
  end
end
```

## First Attempt - (encapsulate logic in a method)

encapsulate in a method with guards and comments - now the main logic is clear
```ruby
class InvoicesCommand

  def new(invoices_form)
    @invoices_form = invoices_form
  end

  def run
    customer_invoices = InvoiceBuilder.generate_invoices(invoices_form)
    send_invoices = select_invoices_to_send(customer_invoices)
    job_id = SendInvoicesJob.call(send_invoices)
    {success: true, result: job_id}
  rescue StandardError => error
    {success: false, error: error}
  end

  private
  attr_reader :invoices_form

  def select_invoices_to_send(customer_invoices)
    # explain logic
    if invoices_form[:print_all] && invoices_form[:customer_ids]
      return customer_invoices
    end
    if invoices_form[:customer_ids] # user specifically chose these for
      return customer_invoices.reject { |ci| ci[:invoices].all?(&:monthly?) }
    end

    # ... other user and system choices to filters

    # or do the default filter - send to people without ebanking
    customer_invoices.filter { |ci| !ci[:customer].ebanking? }
  end
end
```

## Second Attempt - Strategy Pattern (Traditional)

But then it ocurred to us - why should this object have to sort through all the various inputs and deduce what the user or cron-job wanted to do with the filtering.  So we opted for the strategy pattern and each sender would send the filter pattern wanted.  Also this allows us to name each filter (and clarify intention).

```ruby
module InvoiceFilter
  # input: customer_invoices = [ {customer: customer, invoice: invoice}, ... ]
  class AllInvoices
    def call(customer_invoices)
      customer_invoices  # no filter - send on alle
    end
  end

  class AllWithoutEbilling
    def call(customer_invoices)
      customer_invoices.filter { |ci| !ci[:customer].ebanking? }
    end
  end

  class AllExceptionalInvoices
    def call(customer_invoices)
      customer_invoices.inject do |ci, result=[]|
        result << {
                    customer: customer,
                    invoices: invoices.map { |inv| !inv.monthly? }
                  }
        result
      end
    end
  end

end
```

Now we need to change the
```ruby
class InvoiceForm

  attr_reader :invoice_form

  # validations
  # validates ...

  def new(params)
    @params = params
  end

  def invoice_form
    invoice_form = {}
    # ... whatever is needed
    invoice_form[:filter] = filter_logic

    invoice_form
  end

  private
  def filter_logic
    return InvoiceFilter::AllInvoices.new            if params[:filter] == :all
    return InvoiceFilter::AllExceptionalInvoices.new if params[:filter] == :all_exceptional

    InvoiceFilter::AllWithoutEbilling.new
  end

end
```

```ruby
class InvoicesCommand

  def new(invoices_form)
    @invoices_form = invoices_form
  end

  def run
    customer_invoices = InvoiceBuilder.generate_invoices(invoices_form)
    invoices = filter_invoices(customer_invoices)
    job_id = SendInvoicesJob.call(invoices)
    {success: true, result: job_id}
  rescue StandardError => error
    {success: false, error: error}
  end

  private
  attr_reader :invoices_form

  def filter_invoices
    # in-case a strategy isn't chosen - we set the default strategy
    filter_strategy = invoices_form[:filter] || InvoiceFilter::SelectWithoutEbillingInvoices.new

    filter_strategy.call(customer_invoices)
  end
end
```

this design also allows us to send simple lambdas - the form could be rewritten with:
```ruby
class InvoiceForm
  attr_reader :invoice_form

  # validations
  # validates ...

  def new(params)
    @params = params
  end

  def invoice_form
    invoice_form = {}
    # ... whatever is needed
    invoice_form[:filter] = filter_logic

    invoice_form
  end

  private
  def filter_logic
    return InvoiceFilter::AllInvoices.new          if params[:filter] == :all
    return InvoiceFilter::AllExceptionalInvoices.new if params[:filter] == :all_exceptional

    InvoiceFilter::SelectWithoutEbillingInvoices.new
  end

end
```

## Third Attempt - Simplify Strategy with Lambdas

Some filters are very simple we also want to encourage extensions.

Its also good to note that lambdas are also invoked with .call(), so we transformed the simplest filers into lambdas.

Lambdas allow you to encapsulate code and assign it a variable name (& pass it around) and invoke it as  convenient.

```ruby
module InvoiceFilter
  # input: customer_invoices = [ {customer: customer, invoice: invoice}, ... ]

  ALL_INVOICES    = lambda { |customer_invoices| customer_invoices }
  ALL_NO_EBILLING = lambda do |customer_invoices|
                              customer_invoices.filter { |ci| !ci[:customer].ebanking? }
                           end
  class AllExceptionalInvoices
    def call(customer_invoices)
      customer_invoices.inject do |ci, result=[]|
        result << {
                    customer: customer,
                    invoices: invoices.map { |inv| !inv.monthly? }
                  }
        result
      end
    end
  end

end
```

Now we need to change the
```ruby
class InvoiceForm

  FILTER_ChOICES = {'all_customer_invoices'     => InvoiceFilter::ALL_INVOICES,
                    'all_customers_wo_ebilling' => InvoiceFilter::ALL_NO_EBILLING,
                    'all_exceptional_invoices'  => InvoiceFilter::AllExceptionalInvoices.new}
  FILTER_KEYS    = FILTER_ChOICES.keys

  attr_reader :invoice_form

  # validations
  # validates ...

  def new(params)
    @params = params
  end

  def invoice_form
    invoice_form = {}
    # ... whatever is needed
    invoice_form[:filter] = filter_logic

    invoice_form
  end

  private
  attr_reader :params

  def validate_filter
    # no choice is valid - we will use the default
    return if params[:filter].blank?
    # a filter (lambda?) sent in by a rake task
    return if params[:filter].responds_to?(:call)
    # a pre-defined filter chosen in the gui
    return if FILTER_KEYS.include?(params[:filter].to_sym)

    errors.add(:filter, 'not a valid filter')
  end

  def filter_logic
    # default filter if no filter is selected
    return InvoiceFilter::ALL_NO_EBILLING  if params[:filter].blank?

    # allow automated internal tasks with access to pass in their own filters
    return params[:filter]                 if params[:filter].responds_to?(:call)

    # did the user select a pre-defined filter available in the GUI
    if params[:filter].is_s? String && VALID_FILTER_KEYS.include?(filter_symbol)
      return FILTER_CHOICES[filter_symbol]
    end

    # this shouldn't happen if validated before running
    raise 'filter error'
  end

end
```

```ruby
class InvoicesCommand

  def new(invoices_form)
    @invoices_form = invoices_form
  end

  def run
    customer_invoices = InvoiceBuilder.generate_invoices(invoices_form)
    invoices = filter_invoices(customer_invoices)
    job_id = SendInvoicesJob.call(invoices)
    {success: true, result: job_id}
  rescue StandardError => error
    {success: false, error: error}
  end

  private
  attr_reader :invoices_form

  def filter_invoices
    # in-case a strategy isn't chosen - we set the default strategy
    filter_strategy = invoices_form[:filter] || InvoiceFilter::SelectWithoutEbillingInvoices.new

    filter_strategy.call(customer_invoices)
  end
end
```



## Resources

STRATEGY VS COMMAND PATTERN
* https://stackoverflow.com/questions/4834979/difference-between-strategy-pattern-and-command-pattern
* https://newbedev.com/using-a-strategy-pattern-and-a-command-pattern
* https://miafish.wordpress.com/2015/01/16/command-pattern-vs-strategy-pattern/
* https://coderanch.com/t/100214/engineering/Command-Strategy-Pattern

STRATEGY With Lambdas
* https://dockyard.com/blog/2013/07/25/design-patterns-strategy-pattern
* https://wickedlysmart.com/using-lambda-expressions-with-the-strategy-pattern/
* https://blog.sebastian-daschner.com/entries/strategy-pattern-cdi-lambdas

STRATEGY
* https://refactoring.guru/design-patterns/strategy/ruby/example
* http://rubyblog.pro/2016/10/ruby-strategy-pattern
* https://dev.to/andyobtiva/strategic-e9i

* https://github.com/AndyObtiva/strategic
* https://andymaleh.blogspot.com/2021/03/strategic-091-strategy.html


COMMAND
* https://www.sihui.io/design-pattern-command/
* https://rubypatterns.dev/general/command.html
* https://dockyard.com/blog/2013/11/05/design-patterns-command-pattern
* https://jobandtalent.engineering/command-pattern-how-and-why-we-use-it-fa8af952bca1
* https://blog.appsignal.com/2021/04/14/ruby-on-rails-controller-patterns-and-anti-patterns.html
* https://metalelf0.github.io/rails/2016/05/02/command-pattern.htm


LAMBDA
* https://scoutapm.com/blog/how-to-use-lambdas-in-ruby
* https://sodocumentation.net/ruby/topic/474/blocks-and-procs-and-lambdas
* https://blog.appsignal.com/2018/09/04/ruby-magic-closures-in-ruby-blocks-procs-and-lambdas.html
* https://www.freecodecamp.org/news/design-patterns-command-and-concierge-in-life-and-ruby-aab9815817ea/


Decorator
* https://www.rootstrap.com/blog/how-to-improve-maintainability-in-rails-applications-using-patterns/
* https://goiabada.blog/interactors-in-ruby-easy-as-cake-simple-as-pie-33f66de2eb78

OTHER LINKS
* https://refactoring.guru/design-patterns/command/ruby/example
* https://agprolink.asfmra.org/HigherLogic/System/DownloadDocumentFile.ashx?DocumentFileKey=71b7446e-6c64-3974-bb9e-0ff0a0cf8305
* https://makandracards.com/alexander-m/43748-command-pattern
* https://medium.com/@dljerome/design-patterns-in-ruby-command-802b785d1bbd
* https://medium.com/@ibell/the-ruby-command-pattern-pt-1-ad6711af0722
* https://stackoverflow.com/questions/43535421/command-pattern-in-ruby
* https://github.com/davidgf/design-patterns-in-ruby/blob/master/command.md


* https://stackoverflow.com/questions/43535421/command-pattern-in-ruby
* https://medium.com/@nakshtra17/ruby-design-pattern-command-method-3d1e3f41d39d
* http://journal.stuffwithstuff.com/2009/07/02/closures-and-the-command-pattern/
* http://radar.oreilly.com/2014/12/using-the-command-pattern-with-lambda-expressions.html
* https://github.com/piscolomo/ruby-patterns/blob/master/command.rb
* https://en.wikipedia.org/wiki/Command_pattern
* https://sodocumentation.net/ruby/topic/474/blocks-and-procs-and-lambdas



* https://www.freecodecamp.org/news/the-basic-design-patterns-all-developers-need-to-know/
* https://www.developer.com/java/understanding-lambda-enabled-design-patterns/
*
