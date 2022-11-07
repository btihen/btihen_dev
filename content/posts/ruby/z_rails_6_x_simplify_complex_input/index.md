---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Simplify Complex Rails Input Forms"
subtitle: "Realworld Rails often needs to work with multiple models as input"
summary: "Form objects encapsulates the logic needed to validate and structure complex inputs (often requiring multiple models)"
authors: ["btihen"]
tags: ['Ruby', "command object", "command pattern", "lambda"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-08-11T01:11:22+02:00
lastmod: 2021-08-11T01:11:22+02:00
featured: false
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

Many Rails applications are working in complex domains and the inputs to get work done are equally complex (often working multiple models and other data not in models).

In order to keep our Controllers skinny we generally use `Form Objects`.

## Situation

* All customers are grouped by the service area they use.
* All customers have a monthly base fee according to their agreement.
* Some customers pay the standard monthly fee automatically and don't get send monthly bills.
* Some customers will pay extra if their usage exceeds their agreed limits.
* Some customers will pay extra if they incur an extra 3rd party invoice.
* Customers who pay the monthly fee automatically, will not get a monthly bill
* Customers who pay automatically will get sent bills for extra expenses
* Customers who do not pay the monthly fee automatically, will be billed monthly.
* All customers receive a bill when the monthly rate changes.

## Complex Input Form

Our controller needs to know if the incoming data is usable (needs to be validated), but we want to keep the controller skinny and focused on basic logic.  To do this we will use an input (form object).  We will also use a command object to execute complex business logic.

In this case our form needs to provide:
* invoice calculation date and a customer service area for monthly billing (all customers will be processed)
or
* selected customers (they can share a 3rd party bill) & a reference to invoice created by the contractor (to get the amount due and provide proof of service to include in the bill sent)

## Controller

The controller will manage these new ruby object and will look like:

```ruby
# app/controllers/invoices_controller.rb
class InvoicesController < ApplicationController

  def new
    invoices_form = InvoicesForm.new

    render :new, locals: {invoices_form: invoices_form}
  end

  def create
    # form objects can test for data validity
    # form objects use attributes since they are not tied directly to a model
    invoices_form = InvoicesForm.new(invoices_params)
    # our command objects always return a truthy or falsy value
    send_invoices = SendInvoicesCommand.new(invoices_form)

    if invoices_form.valid? && send_invoices.run
      redirect_to invoices_path, success: "Invoices Processing Successfully Submitted."
    else
      flash.now[:error] = "Your form had errors - please fix"
      render :new, locals: {invoices_form: invoices_form}
    end
  end

  private

  # Only allow a list of trusted parameters through.
  def invoices_params
    params.require(:invoices)
          .permit(:invoice_id, :group_id, customer_ids: [])
  end

end
```

## Form Object

```ruby
class InvoicesForm

  # these give your form the ability to do validations, have attributes, know if a model will be changed, etc.
  include ActiveModel::Model
  # include ActiveModel::Dirty         # not needed in this case, but its good know its there, when needed
  include ActiveModel::Attributes
  # include ActiveSupport::Callbacks   # I discourage callbacks (but they may be necessary)

  attr_reader :submission

  # delgate persistance to model
  delegate      :id, :persisted?, # :challenge,
                to: :submission, allow_nil: true
  attr_accessor :id  # MUST BE AFTER THE DELEGATE & NOT IN ATTRIBUTE TYPES

  # The Rails form builder methods (form_for and the rest) need model_name to be defined.
  def self.model_name
    ActiveModel::Name.new(self, nil, 'ChallengeSubmission')
  end

  attribute :order_key,           :string
  attribute :school_id,           :integer
  attribute :groupable_id,        :integer
  attribute :groupable_type,      :string
  attribute :authorable_id,       :integer
  attribute :authorable_type,     :string
  attribute :challenge_id,        :integer
  attribute :challenge_level_id,  :integer
  attribute :collaborator_ids,    :integer_array,   default: []
  attribute :submission_title,    :squished_string, default: nil
  attribute :submission_write_up, :trimmed_text,    default: nil

  validate  :validate_submission_title      # present if level 3 & 4
  validate  :validate_collaborators         # required if level 4
  validate  :validate_submission            # check if general basics are ok

  def self.new_from submission
    new(id:                   submission.id,
        school_id:            submission.school.id,
        order_key:            submission.order_key,
        authorable_id:        submission.authorable_id,   # submission.authorable.id
        authorable_type:      submission.authorable_type, # submission.authorable.class.name
        groupable_id:         submission.groupable_id,    # submission.groupable.id
        groupable_type:       submission.groupable_type,  # submission.groupable.class.name
        challenge_id:         submission.challenge.id,
        challenge_level_id:   submission.challenge_level.id,
        submission_title:     submission.submission_title,
        submission_write_up:  submission.submission_write_up,
        # collaborator_ids:     submission.collaborator_ids,
      )
  end

  def submission
    @submission ||= assign_submission_attribs
  end

  def challenge_collaborations
    challenge_collaborator_ids.map do |collaborator_id|
      db_collaboration = ChallengeCollaboration.find_by(order_key: order_key,
                                                        challenge_submission_id: id,
                                                        collaborable_id: collaborator_id,
                                                        collaborable_type: authorable_type) # collaborators must be the same type of account as author
      db_collaboration || ChallengeCollaboration.new( order_key: order_key,
                                                      challenge_submission_id: id,
                                                      collaborable_id: collaborator_id,
                                                      collaborable_type: authorable_type) # collaborators must be the same type of account as author
    end
  end

  # what info does the form need - should we hide more data?
  # protected

  def author
    # Student.find_by(id: student_id)
    (authorable_type.constantize).find_by(id: authorable_id)
  end

  def school
    School.find_by(id: school_id)
  end

  def group
    # StudentSquad.find_by(id: student_squad_id)
    (groupable_type.constantize).find_by(id: groupable_id)
  end

  def challenge
    Challenge.find_by(id: challenge_id)
  end

  def challenge_level
    challenge.challenge_level
  end

  def display_title_field?
    challenge_level.level_number >= 3
  end

  def collaborator_ids
    IntegerArray.new.cast(attributes["challenge_collaborator_ids"])
  end

  def collaborators
    # Student.where(id: challenge_collaborator_ids)
    (authorable_type.constantize).find_by(id: challenge_collaborator_ids)
  end

  private

  # consider moving to InitiativeServices::SaveInitiative
  def assign_submission_attribs
    # ensure students can only edit their own submissions
    @submission   = ChallengeSubmission.find_by(id:              id,
                                                order_key:       order_key,
                                                authorable_id:   authorable_id,
                                                authorable_type: authorable_type)
    @submission ||= ChallengeSubmission.new
    @submission.school               = school
    @submission.order_key            = order_key
    @submission.groupable            = group
    @submission.authorable           = author
    @submission.challenge            = challenge
    @submission.challenge_level      = challenge_level
    @submission.submission_title     = submission_title
    @submission.submission_write_up  = submission_write_up # attributes["submission_write_up"]
    # can't assign collaborations here! - rails autosaves - we want to save in the command
    # submission.challenge_collaborations = challenge_collaborations
    submission
  end

  def level_num
    challenge_level.level_number
  end

  # title required for levels 3 & 4
  def validate_submission_title
    level_num = challenge_level.level_number
    return if level_num < 3
    return if submission.submission_title.present?
    errors.add(:submission_title, "this challenge requires a title")
  end
  # collaborators required for level 4
  def validate_collaborators
    return if level_num <  4
    return if level_num >= 4 && collaborator_ids.present?

    collaborators = Student.where(id: collaborator_ids)
    # check that collabs are students in the same student_squad
    return if collaborators.all? { |c| c.student_squad.id == student.student_squad.id }

    errors.add(:collaborator_ids, "this challenge requires collaborator(s) in your squad")
  end

  def validate_submission
    return if submission.valid?

    submission.errors.each do |name, desc|
      # rename to match form name
      name = :collaborator_ids  if name.eql? :student_collaborators
      errors.add(name, desc)
    end
  end

end
```


## Example Command Object

This is very straight forward to implement.  Here is a quick example (extremely simplified):

```ruby
# lists other user's (challenger's) submissions
class InvoicesController < ApplicationController

  def new
    invoices_form = InvoicesForm.new(invoices_params)

    render :new, locals: {invoices_form: invoices_form}
  end

  def create
    invoices_form = InvoicesForm.new(invoices_params)
    send_invoices = SendInvoicesCommand.new(invoices_form)

    # our commands return a truthy or falsy value
    if invoices_form.valid? && send_invoices.run
      redirect_to background_jobs_path, success: "Invoices Successfully Submitted to be sent"
    else
      flash.now[:error] = "Your order form had errors - please fix"
      render :new, locals: {invoices_form: invoices_form}
    end
  end

  private

  # Only allow a list of trusted parameters through.
  def invoices_params
    params.require(:invoices)
          .permit(:invoice_id, :group_id, customer_ids: [])
  end

end
```
