---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails: Table Selection Form (Article 2 of 2)"
subtitle: "Article 2 of 2: Dynamic Table Selection in combination with Filtering and Sorting"
summary: "Build a powerful frontend table using Turbo 8 (with Morph Dom feature) to build an efficient table with sorting, filtering and selection"
authors: ['btihen']
tags: ['Rails', 'Tables', 'Turbo', 'Turbo 8', 'Morph', 'Morph Dom', 'Stimulus']
categories: ["Code", "Ruby Language", "Rails Framework", "Simulus JS"]
date: 2025-03-03T01:20:00+02:00
lastmod: 2025-03-03T01:20:00+02:00
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

This article uses: https://btihen.dev/posts/ruby/rails_8_0_rails_tables_filtering_sorting as a starting point.

Aticle 1 of 2: [Modern Rails: Table Filtering & Sorting](https://btihen.dev/posts/ruby/rails_8_0_rails_tables_filtering_sorting)
Article 2 of 2: [**Modern Rails: Table Selection Form**](https://btihen.dev/posts/ruby/rails_8_0_rails_tables_selection_form)

The code for this article can be found at: https://github.com/btihen-dev/rails_table_selection_form


## Basic Rails App Setup

Be sure you have a database (I assume postgresql), but feel free to reconfigure to your favorite database - this project is not database specific.

It assumes Rails 8.0.1 (but should work with 7.1+)
It also assumes Ruby 3.4.2 (but should work with 3.2+)

```bash
git clone https://github.com/btihen-dev/rails_table_filtering_sorting.git
cd rails_table_filtering_sorting
bundle install
yarn install
rails db:create
rails db:migrate
rails db:seed
```

See [Modern Rails: Table Filtering & Sorting](https://btihen.dev/posts/ruby/rails_8_0_rails_tables_filtering_sorting) article to understand the code up to this point.

## Row Selector

First we will add a checkbox to the column (for a select all):
```ruby
  <th scope="col">
  Select All
    <%= check_box_tag "select-all", nil, false %>
  </th>
```

and a checkbox in each row (to select the row)

```ruby
  <td scope="row">
    <%= check_box_tag "selected_rows[]",
        character.id,
        false, # not selected
        id: "selected_rows_#{character.id}"
    %>
  </td>
```
I this case we need to set the `id` manually to ensure it is unique - since the name `selected_rows[]` would usually be the id, but this will be repeated many times we help rails by adding a manual id using `id: "selected_rows_#{character.id}"`

So now the index page  will look like this:

```ruby
# app/views/characters/index.html.erb
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>

<h1>Characters</h1>

<div id="characters" class="text-center">

  <table class="table table-striped table-hover">
    <thead class="sticky-top">
      <tr class="table-primary">
        <th scope="col" class="align-top">
          Select<br>
          <div class="form-check form-switch">
          <%= check_box_tag "select-all",
              nil,
              false, # todo: are all displayed rows selected?
              class: "form-check-input"
          %>
          </div>
        </th>
        <th scope="col" class="align-top">
          ID <%= sort_link(column: "characters.id") %>
        </th>
        <th scope="col" class="align-top">
          Last Name <%= sort_link(column: "last_name") %><br>
          <%= render "form_match_filter", field_name: :last_name_filter, placeholder: "partial last name" %>
        </th>
        <th scope="col" class="align-top">
          First Name <%= sort_link(column: "first_name") %><br>
          <%= render "form_match_filter", field_name: :first_name_filter, placeholder: "partial first name" %>
        </th>
        <th scope="col" class="align-top">
          Gender <%= sort_link(column: "gender") %><br>
          <%= render "form_dropdown_filter", field_name: :gender_selection, options: Character.distinct.pluck(:gender).compact %>
        </th>
        <th scope="col" class="align-top">
          Species <%= sort_link(column: 'species.species_name') %><br>
          <%= render "form_dropdown_filter", field_name: :species_selection, options: Species.pluck(:species_name, :id) %>
        </th>
        <th scope="col" class="align-top">
          Company <%= sort_link(column: 'companies.company_name') %><br>
          <%= render "form_match_filter", field_name: :company_filter, placeholder: "partial company name" %>
        </th>
      </tr>
    </thead>

    <tbody class="scrollable-table">
      <% @characters.each do |character| %>
        <tr id="<%= dom_id(character) %>" class="align-middle">
          <td scope="row">
            <div class="form-check form-switch"">
              <%= check_box_tag "selected_rows[]",
                  character.id,
                  @selected_rows.include?(character.id), # is row selected?
                  id: "selected_row_#{character.id}", # unique id
                  class: "form-check-input"
              %>
            </div>
          </td>
          <th scope="row"><%= link_to character.id, edit_character_path(character) %></th>
          <td><%= character.last_name %></td>
          <td><%= character.first_name %></td>
          <td><%= character.gender %></td>
          <td><%= character.species.species_name %></td>
          <td class="text-start">
            <% character.character_jobs.each do |character_job| %>
              <div class="job-container mb-2">
                <div class="company-row">
                  <div class="h6"><b><%= character_job.job.company.company_name %></b></div>
                </div>
                <div class="job-details-row">
                  <span style="font-weight: 600;">- <%= character_job.job.role %></span>
                  <span>
                    <em>
                      (from: <%= character_job.start_date.strftime("%e %b '%y") %>
                      to: <%= character_job.end_date&.strftime("%e %b '%y") || 'present' %>)
                    </em>
                  </span>
                </div>
              </div>
            <% end %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>

</div>

<%= link_to "New character", new_character_path, class: "btn btn-primary" %>
```

now run `bin/dev` and open your browser and go to `http://localhost:3000` be sure you see a list of characters like:

![add_selection_switches](add_selection_switches.png)

Unfortunately, we can only select the switches, but nothing happens.

## Row Selection

To select a row we need to identify the the row and add it to the URL and of course we will need a form to submit the selected rows.

### Add Selected Route

We will need a new routee to handle the selected rows - using:
```ruby
post "/selected_characters", to: "selected_characters#index"
```

So now the router will look like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :jobs
  resources :companies
  resources :characters
  resources :species
  post "/selected_characters", to: "selected_characters#index"

  get "up" => "rails/health#show", as: :rails_health_check

  root "characters#index"
end
```

### Add Selected Controller

Let's build the controller:

```ruby
# app/controllers/selected_characters_controller.rb
class SelectedCharactersController < ApplicationController
  def index
    selected_rows = params[:selected_rows] || params['selected_rows']
    selected_ids =
      case selected_rows
      when Array
        selected_rows.map(&:to_i) || []
      when String
        selected_rows.to_s.split(',').map(&:to_i).reject(&:zero?)
      else
        []
      end

    # base query (remember not to create an N+1 query)
    @selected_characters =
      Character
        .includes(:species)
        .includes(character_jobs: { job: :company })
        .where(id: selected_ids)

    respond_to do |format|
      # format.html # Render the selected.html.erb view
      format.html { render :index }
      format.json { render json: @selected_characters }
    end
  end
end
```

# Selected Template

Now we need to create a template for the selected characters.

```ruby
# app/views/selected_characters/index.html.erb
<p style="color: green"><%= notice %></p>

<h1>Selected Characters</h1>

<div id="characters" class="container text-center">

<table class="table table-striped table-hover">
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">ID</th>
      <th scope="col">First Name</th>
      <th scope="col">Last Name</th>
      <th scope="col">Gender</th>
      <th scope="col">Species</th>
      <th scope="col">Company</th>
    </tr>
  </thead>

  <tbody class="scrollable-table">
    <% @selected_characters.each do |character| %>
      <tr id="<%= dom_id(character) %>">
        <th scope="row"><%= link_to character.id, edit_character_path(character) %></th>
        <td><%= character.first_name %></td>
        <td><%= character.last_name %></td>
        <td><%= character.gender %></td>
        <td><%= character.species.species_name %></td>
        <td class="text-start">
          <% character.character_jobs.each do |character_job| %>
            <div class="job-container mb-2">
              <div class="company-row">
                <div class="h6"><b><%= character_job.job.company.company_name %></b></div>
              </div>
              <div class="job-details-row">
                <span style="font-weight: 600;">- <%= character_job.job.role %></span>
                <span>
                  <em>
                    (from: <%= character_job.start_date.strftime("%e %b '%y") %>
                    to: <%= character_job.end_date&.strftime("%e %b '%y") || 'present' %>)
                  </em>
                </span>
              </div>
            </div>
          <% end %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>

</div>
```

### Add Selected Row Form

Now we need to add a form to submit the selected rows to the Characters template:

```ruby
# app/views/characters/index.html.erb
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>

<h1>Characters</h1>

<div id="characters" class="text-center">
  <%= form_with url: selected_characters_path, method: :post,
      data: { controller: "characters-table", turbo: false }, local: true do |form| %>

    <div class="d-flex justify-content-end mb-3">
      <%= form.submit "Submit Selected", class: "btn btn-primary" %>
    </div>

    <%= hidden_field_tag :selected_rows, params[:selected_rows] %>

    <table class="table table-striped table-hover">
      ...
    </table >

  <% end %>
</div>

<%= link_to "New character", new_character_path, class: "btn btn-primary" %>
```

And Stimulus to JS buttons:
```ruby
<!-- select all -->
<%= check_box_tag "select-all",
    nil,
    false,
    class: "form-check-input",
    data: {
      action: "characters-table#toggleSelectAll",
      "characters-table-target": "selectAll"
    }
%>

<!-- row selection -->
<%= check_box_tag "row-selector",
    character.id,
    @selected_rows.include?(character.id), # is row selected?
    id: "row-selector_#{character.id}", # unique id
    data: {
      action: "change->characters-table#selectRow",
      "characters-table-target": "rowSelector"
    },
    class: "form-check-input"
%>
```

Now it should look like:
```ruby
# app/views/characters/index.html.erb
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>

<h1>Characters</h1>

<div id="characters" class="text-center">
<%= form_with url: selected_characters_path, method: :post,
    data: { controller: "characters-table", turbo: false }, local: true do |form| %>

  <div class="d-flex justify-content-end mb-3">
    <%= form.submit "Submit Selected", class: "btn btn-primary" %>
  </div>

  <%= hidden_field_tag :selected_rows, params[:selected_rows] %>

  <table class="table table-striped table-hover">
    <thead class="sticky-top">
      <tr class="table-primary">
        <th class="align-top">
          Select<br>
          <div class="form-check form-switch">
          <%= check_box_tag "select-all",
              nil,
              false,
              class: "form-check-input",
              data: {
                action: "characters-table#toggleSelectAll",
                "characters-table-target": "selectAll"
              }
          %>
          </div>
        </th>
        <th class="align-top">
          ID <%= sort_link(column: "characters.id") %>
        </th>
        <th class="align-top">
          Last Name <%= sort_link(column: "last_name") %><br>
          <%= render "form_match_filter", field_name: :last_name_filter, placeholder: "partial last name" %>
        </th>
        <th class="align-top">
          First Name <%= sort_link(column: "first_name") %><br>
          <%= render "form_match_filter", field_name: :first_name_filter, placeholder: "partial first name" %>
        </th>
        <th class="align-top">
          Gender <%= sort_link(column: "gender") %><br>
          <%= render "form_dropdown_filter",
              field_name: :gender_selection, options: Character.distinct.pluck(:gender).compact %>
        </th>
        <th class="align-top">
          Species <%= sort_link(column: 'species.species_name') %><br>
          <%= render "form_dropdown_filter",
                     field_name: :species_selection, options: Species.pluck(:species_name, :id) %>
        </th>
        <th class="align-top">
          Company <%= sort_link(column: 'companies.company_name') %><br>
          <%= render "form_match_filter", field_name: :company_filter, placeholder: "partial company name" %>
        </th>
      </tr>
    </thead>

    <tbody class="scrollable-table">
      <% @characters.each do |character| %>
        <tr id="<%= dom_id(character) %>" class="align-middle">
          <td>
            <div class="form-check form-switch"">
              <%= check_box_tag "row-selector",
                  character.id,
                  @selected_rows.include?(character.id), # is row selected?
                  id: "row-selector_#{character.id}", # unique id
                  data: {
                    action: "change->characters-table#selectRow",
                    "characters-table-target": "rowSelector"
                  },
                  class: "form-check-input"
              %>
            </div>
          </td>
          <th scope="row"><%= link_to character.id, edit_character_path(character) %></th>
          <td><%= character.last_name %></td>
          <td><%= character.first_name %></td>
          <td><%= character.gender %></td>
          <td><%= character.species.species_name %></td>
          <td class="text-start">
            <% character.character_jobs.each do |character_job| %>
              <div class="job-container mb-2">
                <div class="company-row">
                  <div class="h6"><b><%= character_job.job.company.company_name %></b></div>
                </div>
                <div class="job-details-row">
                  <span style="font-weight: 600;">- <%= character_job.job.role %></span>
                  <span>
                    <em>
                      (from: <%= character_job.start_date.strftime("%e %b '%y") %>
                      to: <%= character_job.end_date&.strftime("%e %b '%y") || 'present' %>)
                    </em>
                  </span>
                </div>
              </div>
            <% end %>
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>

<% end %>
</div>

<%= link_to "New character", new_character_path, class: "btn btn-primary" %>
```

## Stimulus Controller

### Fix Filters

Let's test our existing filters before we add the row selection.

now we see any filter changes will submit the selected rows form - not just the filter form.

```js
// app/javascript/controllers/characters-table_controller.js
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["selectAll", "rowSelector"];

  filter(event) {
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      this.element.requestSubmit();
    }, 300);
  }
}
```

The problem is the `requestSubmit()` method will submit the selected_rows form too let's fix this using `Turbo.visit()` so this function should now look like:
```js
// app/javascript/controllers/characters-table_controller.js
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["selectAll", "rowSelector"];

  filter(event) {
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      const currentParams = new URLSearchParams(window.location.search);
      // Suppose the changed filter input has `name="last_name_filter"`
      // and its new value is in event.target.value
      currentParams.set(event.target.name, event.target.value);

      // Merge other existing params if needed (like selected_rows) -
      // this visit persists the selected rows - even those not visible
      Turbo.visit(`${window.location.pathname}?${currentParams.toString()}`, {
        action: "replace", // or 'advance'
      });
    }, 300);
  }
}
```

Now the filters and sorting should work as expected again.

### Add Row Selection Controll


Now we need to update our Stimuulus controller to handle the row selection.  This should do the trick.

```js
// app/javascript/controllers/characters-table_controller.js
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["selectAll", "rowSelector"];
  connect() {
    // console.log("Controller connected");
    this.initializeSelectedRows();
  }
  filter(event) {
    // console.log("filter change");
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      const currentParams = new URLSearchParams(window.location.search);
      // Suppose the changed filter input has `name="last_name_filter"`
      // and its new value is in event.target.value
      currentParams.set(event.target.name, event.target.value);

      // this now submits only to the filter form and not the selected_rows form
      Turbo.visit(`${window.location.pathname}?${currentParams.toString()}`, {
        action: "replace", // or 'advance'
      });
    }, 300);
  }
  selectRow(event) {
    // console.log("Row selected");
    if (!this.hasRowSelectorTarget) {
      console.error("No row selector targets found");
      return;
    }
    this.updateSelectedRows();
  }
  initializeSelectedRows() {
    // console.log("initializeSelectedRows");
    const urlParams = new URLSearchParams(window.location.search);
    const selectedRows = urlParams.get("selected_rows");
    // console.log("Initializing selected rows from URL:", selectedRows);
    if (selectedRows) {
      const selectedRowIds = selectedRows.split(",");
      this.rowSelectorTargets.forEach((checkbox) => {
        if (selectedRowIds.includes(checkbox.value)) {
          checkbox.checked = true;
        }
      });
    }
  }
  updateSelectedRows() {
    // console.log("Updating selected rows");
    const selectedRows = [];
    this.rowSelectorTargets.forEach((checkbox) => {
      if (checkbox.checked) {
        selectedRows.push(checkbox.value);
      }
    });
    const url = new URL(window.location);
    if (selectedRows.length > 0) {
      url.searchParams.set("selected_rows", selectedRows.join(","));
    } else {
      url.searchParams.delete("selected_rows");
    }
    console.log("Updating URL with selected rows:", selectedRows);
    // Update the URL without reloading the page
    window.history.replaceState({}, "", url);
  }
}
```

Test - be sure the selected checkboxes are shown in the URL when clicked, persist after a page reload and after a filter change!

![selected_rows_in_url](selected_rows_in_url.png)

### Select All

Now our previous function `filter` is to niave as it uses `requestSubmit` which will submit ANY form on the page - we only want it to submit the `filter` form.  So let's rewrite it to use `Turbo.visit` instead.

So now it should look like this:
```js
  toggleSelectAll() {
    // console.log("toggleSelectAll");
    const isChecked = this.selectAllTarget.checked;
    this.rowSelectorTargets.forEach((checkbox) => {
      checkbox.checked = isChecked;
    });
    this.updateSelectedRows();
  }
```

Now the full JS controller should look like:
```js
// app/javascript/controllers/characters-table_controller.js
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["selectAll", "rowSelector"];
  connect() {
    // console.log("Controller connected");
    this.initializeSelectedRows();
  }
  filter(event) {
    // console.log("filter change");
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      const currentParams = new URLSearchParams(window.location.search);
      // Suppose the changed filter input has `name="last_name_filter"`
      // and its new value is in event.target.value
      currentParams.set(event.target.name, event.target.value);

      // only submits to the filter form instead of all forms on the page
      Turbo.visit(`${window.location.pathname}?${currentParams.toString()}`, {
        action: "replace", // or 'advance'
      });
    }, 300);
  }
  selectRow(event) {
    // console.log("Row selected");
    if (!this.hasRowSelectorTarget) {
      console.error("No row selector targets found");
      return;
    }
    this.updateSelectedRows();
  }
  toggleSelectAll() {
    // console.log("toggleSelectAll");
    const isChecked = this.selectAllTarget.checked;
    this.rowSelectorTargets.forEach((checkbox) => {
      checkbox.checked = isChecked;
    });
    this.updateSelectedRows();
  }
  initializeSelectedRows() {
    // console.log("initializeSelectedRows");
    const urlParams = new URLSearchParams(window.location.search);
    const selectedRows = urlParams.get("selected_rows");
    // console.log("Initializing selected rows from URL:", selectedRows);
    if (selectedRows) {
      const selectedRowIds = selectedRows.split(",");
      this.rowSelectorTargets.forEach((checkbox) => {
        if (selectedRowIds.includes(checkbox.value)) {
          checkbox.checked = true;
        }
      });
    }
  }
  updateSelectedRows() {
    // console.log("Updating selected rows");
    const selectedRows = [];
    this.rowSelectorTargets.forEach((checkbox) => {
      if (checkbox.checked) {
        selectedRows.push(checkbox.value);
      }
    });
    const url = new URL(window.location);
    if (selectedRows.length > 0) {
      url.searchParams.set("selected_rows", selectedRows.join(","));
    } else {
      url.searchParams.delete("selected_rows");
    }
    console.log("Updating URL with selected rows:", selectedRows);
    // Update the URL without reloading the page
    window.history.replaceState({}, "", url);
  }
}
```

Test - select all (and deselect all) - should only affect the visible rows! should persist between page loads and filter changes.

Fix html (show as checked when all visible rows are selected)

```html
    <%= check_box_tag "select-all",
        nil,
        @all_visible_selected,
        class: "form-check-input",
        data: {
          action: "characters-table#toggleSelectAll",
          "characters-table-target": "selectAll"
        }
    %>,
```

in the controller we now need to add the new instance variable that compares the visible `@characters` to the `@selected_rows` using:
```ruby
  def index
    ...
    # execute query
    @characters = query.all
    @all_visible_selected = @characters.all? { |c| @selected_rows.include?(c.id) }
  end
```

now the controller should look like:

```ruby
# app/controllers/characters_controller.rb
class CharactersController < ApplicationController
  before_action :set_character, only: %i[ show edit update destroy ]

  # GET /characters or /characters.json
  def index
    # query with sorting
    column = params[:column]
    direction = params[:direction]
    @selected_rows = params[:selected_rows]&.split(',')&.map(&:to_i) || []

    # base query
    query = Character
            .includes(:species)
            .includes(character_jobs: { job: :company })

    # add sort if direction is given
    query = if direction == 'none' || column.blank?
              query.order('characters.id')
            else
              query.order("#{column} #{direction}")
            end

    # partial match filters
    @first_name_filter = params[:first_name_filter]
    @last_name_filter = params[:last_name_filter]
    @company_filter = params[:company_filter]
    query = query.where('characters.first_name ilike ?', "%#{@first_name_filter}%") if @first_name_filter.present?
    query = query.where('characters.last_name ilike ?', "%#{@last_name_filter}%") if @last_name_filter.present?
    query = if @company_filter.present?
              query.joins(character_jobs: { job: :company }).where('companies.company_name ilike ?', "%#{@company_filter}%")
            else
              query
            end

    # Dropdown selections
    @gender_selection = params[:gender_selection]
    @species_selection = params[:species_selection]
    query = query.where(gender: @gender_selection) if @gender_selection.present?
    query = query.where(species_id: @species_selection) if @species_selection.present?

    # execute query
    @characters = query.all
    @all_visible_selected = @characters.all? { |c| @selected_rows.include?(c.id) }
  end
```

Test

![selected_characters_unsorted](selected_characters_unsorted.png)

## Extra - sort selected characters like in form

Let's allow our selection show in the same sort order as the form.

add
```ruby
    sort_column = params['column'] || :id
    sort_direction = params['direction'] || :asc

    # add sort if direction is given
    query = if direction == 'none' || column.blank?
              query.order('characters.id')
            else
              query.order("#{column} #{direction}")
            end
```

So now the controller would look like:
```ruby
# app/controllers/selected_characters_controller.rb
class SelectedCharactersController < ApplicationController
  def index
    selected_rows = params['selected_rows'] || params[:selected_rows]
    selected_ids =
      case selected_rows
      when Array
        selected_rows.map(&:to_i).reject(&:zero?)
      when String
        selected_rows.to_s.split(',').map(&:to_i).reject(&:zero?)
      else
        []
      end

    # base query
    query = Character
            .includes(:species)
            .includes(character_jobs: { job: :company })
            .where(id: selected_ids)

    # add sort if sort_direction is given
    sort_column = params['column'] || params[:column] || :id
    sort_direction = params['direction'] || params[:direction] || :asc
    query = if sort_direction == 'none' || sort_column.blank?
              query.order('characters.id')
            else
              query.order("#{sort_column} #{sort_direction}")
            end

    @selected_characters = query

    respond_to do |format|
      format.html { render :index }
      format.json { render json: @characters }
    end
  end
end
```

Now update the view with the hidden inputs to pass sort_info to the controller:
```ruby
  <%= hidden_field_tag :sort_column, params[:column] %>
  <%= hidden_field_tag :sort_direction, params[:direction] %>
```

So now the view should looks like:
```ruby
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>

<h1>Characters</h1>

<div id="characters" class="text-center", data-controller="characters-table">
<!-- <div id="characters" class="text-center"> -->
<%= form_with url: selected_characters_path, method: :post,
    data: { controller: "characters-table", turbo: false }, local: true do |form| %>

  <div class="d-flex justify-content-end mb-3">
    <%= form.submit "Submit Selected", class: "btn btn-primary" %>
  </div>

  <!-- send sort info to controller -->
  <%= hidden_field_tag :sort_column, params[:column] %>
  <%= hidden_field_tag :sort_direction, params[:direction] %>
  <!-- send selected rows to controller -->
  <%= hidden_field_tag :selected_rows, params[:selected_rows] %>

  <table class="table table-striped table-hover">
```

Test

![selected_characters_unsorted](selected_characters_unsorted.png)

## Resources

### Adding Bootstrap to an existing project

* https://www.youtube.com/watch?v=phOUsR0dm5s
* https://medium.com/@gjuliao32/installing-bootstrap-rails-7-a-step-by-step-guide-0fc4a843d94f

### Bootstrap Icons

* https://icons.getbootstrap.com/

### Rails Table Articles

* [Table Sorting Rails 7.1 - 21 Mar 2024](https://www.colby.so/posts/turbo-8-refresh-sorting)
* [Table Filtering Rails 7.0 - 15 Oct 2021](https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire)
* [Table Sorting Rails 7.0 - 19 Sep 2021](https://www.colby.so/posts/sortable-table-with-rails-and-turbo-frames)
* [Table Sorting with Stimulus](https://www.colby.so/posts/a-sortable-table-with-rails-and-stimulusreflex)

### Rails Hotwire

* [Turbo Rails Intro](https://www.colby.so/posts/turbo-rails-101-todo-list)
* [Hotwiring Rails Book](https://book.hotwiringrails.com/)
* [Hot Rails Tutorial - building Turbo Rails](https://www.hotrails.dev/turbo-rails)
* [Rebuilding Turbo Rails - new version](https://www.hotrails.dev/rebuilding-turbo-rails)
* [Rails Hotwire Modals](https://webcrunch.com/posts/hotwire-rails-turbo-modals)
* [Turbo Frame Pages in Ruby on Rails 7](https://www.youtube.com/watch?v=iwZDoz_Ya2k)
* [Digging into Turbo with Ruby on Rails 7](https://www.youtube.com/watch?v=0CSGsHnci2I)
* [Mastering Turbo Frames and Turbo Streams in Rails 7: Build a Journal Entry Tagging Feature](https://www.youtube.com/watch?v=lG5aRBJHDBQ)
* [Odin Probject - Turbo Tutorial](https://www.theodinproject.com/lessons/ruby-on-rails-turbo)
* [Hotwire Turbo Transitions](https://dev.to/nejremeslnici/how-to-use-view-transitions-in-hotwire-turbo-1kdi)

## APPENDIX

when using esbuild you should see the following two files and you should NOT see `import-map` files or config!

David Colby who wrote: https://www.colby.so/posts/turbo-8-refresh-sorting and https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire

writes:

For reference, `Procfile.dev` looks like this for me on a fresh esbuild + bootstrap install:
```bash
web: env RUBY_DEBUG_OPEN=true bin/rails server
js: yarn build --watch
css: yarn watch:css
```

And `package.json` looks like this:
```js
{
  "name": "app",
  "private": true,
  "dependencies": {
    "@hotwired/stimulus": "^3.2.2",
    "@hotwired/turbo-rails": "^8.0.4",
    "@popperjs/core": "^2.11.8",
    "autoprefixer": "^10.4.19",
    "bootstrap": "^5.3.3",
    "bootstrap-icons": "^1.11.3",
    "esbuild": "^0.20.2",
    "nodemon": "^3.1.0",
    "postcss": "^8.4.38",
    "postcss-cli": "^11.0.0",
    "sass": "^1.76.0"
  },
  "scripts": {
    "build": "esbuild app/javascript/*.* --bundle --sourcemap --format=esm --outdir=app/assets/builds --public-path=/assets",
    "build:css:compile": "sass ./app/assets/stylesheets/application.bootstrap.scss:./app/assets/builds/application.css --no-source-map --load-path=node_modules",
    "build:css:prefix": "postcss ./app/assets/builds/application.css --use=autoprefixer --output=./app/assets/builds/application.css",
    "build:css": "yarn build:css:compile && yarn build:css:prefix",
    "watch:css": "nodemon --watch ./app/assets/stylesheets/ --ext scss --exec \"yarn build:css\""
  },
  "browserslist": [
    "defaults"
  ]
}
```
