---
layout: post
title: ViewComponents, the Missing View Layer for Rails
categories:
  - articles
tags:
  - rails
  - view components
  - ui library
---

If you've worked with Rails for any measure of time, then you know that Rails' Views can quickly get out of hand. Between Helpers, instance variables, and inline logic, they quickly become bloated and tightly coupled to other view specific logic. Pushing these concerns into the Controller layer, or even a Presenter helps, but still lands the View in a place where it contains multiple responsibilities. If only there was a better way... Enter ViewComponents or as I like to call them, "The Missing View Layer for Rails".

<!--excerpt-->

## Our Example

First, let's take a look at an example of a User listing page with pagination and actions. I find this example to be a great illustration of the power of ViewComponents while staying grounded in a real-world feature.

{% include blockquote.html quote="You don't need to be familiar with the Pagy gem. Just know that it allows us to paginate a collection of objects." source_link="https://github.com/ddnexus/pagy" source_text="Pagy gem documentation" %}

```ruby
# users_controller.rb
class UsersController < ApplicationController
  USERS_PER_PAGE = 30

  def index
    @filters = {
      role: Role.all.select(:name)
    }

    @pagy, users = pagy(available_users, limit: USERS_PER_PAGE, page: params[:page])

    @users = users.map { |user| UserPresenter.new(user) }
  end

  private

  def filter_params
    params.permit(filters: :role)
  end

  def available_users
    if filter_params.present?
      User.where(role: filter_params)
    else
      User.all
    end
  end
end

# user_presenter.rb
class UserPresenter < SimpleDelegator
  def current_roles
    roles.map(&:name).to_sentence
  end

  def dropdown_actions
    [:edit, :destroy]
  end
end
```

```html
<!-- index.html.erb -->
<%= render partial: "filters", locals: { available_filters: @filters } %>

<table class="bg-gray-300 p-4">
  <thead>
    <tr>
      <th>Name</th>
      <th>Roles</th>
      <th>Actions</th>
    </tr>
  </thead>

  <tbody>
    <% @users.each do |user| %>
    <tr>
      <td><%= user.name %></td>
      <td><%= user.current_roles %></td>
      <td>
        <%= render partial: "dropdown", locals: { resource: user, actions:
        user.dropdown_actions } %>
      </td>
    </tr>
    <% end %>
  </tbody>

  <tfoot>
    <tr>
      <td colspan="3">
        <p><strong>Pagination</strong></p>
        <%= pagy_nav(@pagy) %>
      </td>
    </tr>
  </tfoot>
</table>
```

How many responsibilities can you find spread between the Controller, Presenter, and View for this single page? Here's the ones I quickly identified.

1. **(Controller)** Must understand how to set filters for the View
2. **(Controller)** Wraps every User within a UserPresenter for additional functionality outside standard Model.
3. **(Controller)** Must configure pagination values to prepare for View output
4. **(View)** Needs to understand the Filter partials interface to send the correct data
5. **(View)** Must know which table headers correspond to the User resource
6. **(View)** Must know how to render each individual User
7. **(View)** Must know which values to send to the actions' dropdown, so has to understand the interface
8. **(View)** Must utilize the proper DSL from the Pagy gem in the table footer
9. **(Presenter)** Must know which dropdown actions to show for each User
10. **(Presenter)** Must convert a User's roles into a human readable string

Now imagine a new feature requirement is requested to build a similar but slightly different page for a given User's blog posts. You'd end up copying and pasting much of the above or creating additional partials to help abstract common functionality. ViewComponents offer a cleaner, isolated way of containing related View concerns in a single location. This includes building UI libraries with consistent CSS styling, reusable components, better testability, and more.

## ViewComponents make the View layer awesome

ViewComponents have a number of advantages. From proper separation of concerns, to preview-ability and testing. They really feel like a total application changer once you start using them. Co-location is a great benefit of using them as it allows for a ViewComponent object to be related to specific component rendering view code. This helps to keep your UI components focusing on single concerns. Generally this would mean you'd have a `my_component.rb` and `my_component.html.erb` file, where the `my_component.rb` automatically renders the `my_component.html.erb` file.

{% include blockquote.html quote="A framework for creating reusable, testable & encapsulated view components, built to integrate seamlessly with Ruby on Rails." source_link="https://viewcomponent.org/" %}

Looking at the previous example let's inject some ViewComponent goodness.

### Filters Component

The least coupled concept are the Filters. For this example, we can assume that a filter allows the user to exclude records by enum columns to keep it simple. This means you might end up with a URL like: **my-site.com/users?filters[role]=admin** which would only return users with the **admin** role. Akin to executing the following SQL:

```sql
SELECT *
FROM users
INNER JOIN user_roles ON user_roles.user_id = users.id
INNER JOIN roles ON roles.id = user_roles.role_id
WHERE roles.name = 'admin'
```

We have a need of an Application-specific as well as General-purpose component as it needs to know which columns to show for the filters and filters could be reusable. ViewComponent's [best practices](https://viewcomponent.org/best_practices.html#two-types-of-viewcomponents) recommend seperating responsibilities between generic and specific use cases.

{% include blockquote.html quote="General-purpose ViewComponents implement common UI patterns, such as a button, form, or modal. Application-specific ViewComponents translate a domain object into one or more general-purpose components." source_link="https://viewcomponent.org/best_practices.html#two-types-of-viewcomponents" %}

For Filters, we can create a component for each of the use cases allowing us to resuse the common logic for future Filter related features.

### Application-specific

We'll start by creating a UsersFilterComponent to dictate how filters are built for this specific use case.

```ruby
# Application-specific
# users_filter_component.rb
class UsersFilterComponent < ViewComponent::Base

  # You could also use dependency injection for more complex use cases with available roles
  def filters
    {
      role: Role.all.select(:name)
    }
  end
end
```

Notice how the corresponding component view for UsersFilterComponent actually is using the General-purpose FilterComponent during its rendering.

```html
<!-- users_filter_component.html.erb -->
<%= render FilterComponent.new(available_filters: filters) %>
```

### General-purpose

Since our Application-specific component utilizes the General-purpose component, we can have the FilterComponent accept a generic `available_filters` argument. Some of this is psuedo code to illustrate the concept of taking a collection of filters from a Hash and rendering them iteratively in the View.

```ruby
# General-purpose
# filter_component.rb
class FilterComponent < ViewComponent::Base
  attr_reader :available_filters

  def initialize(available_filters:)
    @available_filters = available_filters
  end
end
```

```html
<!-- Psuedo code to build links like: ?filters[role]=admin -->
<!-- filter_component.html.erb -->
<div>
  <% available_filters.keys.each do |filter_key| %> 
    <% available_filters[filter_key].each do |filter| %> 
      <%= link_to filter.name, params.merge(filter_key: filter.name) %> 
    <% end %> 
  <% end %>
</div>
```

We now gain some nice cleanup in our Controller and View layers:

```ruby
# users_controller.rb
class UsersController < ApplicationController
  USERS_PER_PAGE = 30

  def index
    # - REMOVED -
    # @filters = {
    #  role: Role.all.select(:name)
    # }

    @pagy, users = pagy(available_users, limit: USERS_PER_PAGE, page: params[:page])

    @users = users.map { |user| UserPresenter.new(user) }
  end

 # ... Rest of Controller
```

```html
<!-- index.html.erb -->
<!-- Removed <%= render partial: "filters", locals: { available_filters: @filters } %> -->
<%= render UsersFilterComponent.new %>

<table class="bg-gray-300 p-4">
  <thead>
```

We now have a User specific filters component and a general-purpose filters component to utilize on other pages. Next let's dig into rendering the current User in the collection.

## UserComponent

The primary section we are abstracting is the render loop from within the table body. I've added comments to show the section we're looking at componentizing.

```html
  <tbody>
    <% @users.each do |user| %>
      <!-- UserComponent -->
      <tr>
        <td><%= user.name %></td>
        <td><%= user.current_roles %></td>
        <td><%= render partial: "dropdown", locals: { resource: user, actions: user.dropdown_actions } %></td>
      </tr>
      <!-- End UserComponent -->
    <% end %>
  </tbody>
```

We'll be pulling in our Presenter logic, as it now can completely live within our new UserComponent definition. With the upcoming change below, **UserPresenter** can be removed and our Controller slimmed down.

### Updated Controller
```ruby
# users_controller.rb
class UsersController < ApplicationController
  USERS_PER_PAGE = 30

  def index
    @pagy, @users = pagy(available_users, limit: USERS_PER_PAGE, page: params[:page])

    # - REMOVED -
    # @users = users.map { |user| UserPresenter.new(user) }
  end

# Also deleted the UserPresenter
```

### Updated View index.html.erb
```html
<!-- index.html.erb -->
  <tbody>
    <% @users.each do |user| %>
      <%= render UserComponent.new(user:) %>
    <% end %>
  </tbody>
```

### New UserComponent
```ruby
# New component
# user_component.rb
class UserComponent < ViewComponent::Base
  attr_reader :user

  def initialize(user:)
    @user = user
  end

  # Pulled directly from Presenter
  def current_roles
    user.roles.map(&:name).to_sentence
  end

  # Pulled directly from Presenter
  def dropdown_actions
    [:edit, :destroy]
  end
end
```

### New UserComponent View
```html
<!-- user_component.html.erb -->
      <tr>
        <td><%= user.name %></td>
        <td><%= user.current_roles %></td>
        <td><%= render partial: "dropdown", locals: { resource: user, actions: user.dropdown_actions } %></td>
      </tr>
```

Now depending on how far you want to isolate the different View responsibilities there are a few additional improvements you could take:

1. Create General-purpose components for Table, TableHeader, TableBody, TableCell
2. Refactor the "dropdown" partial to become the DropdownComponent. Great example of a General-purpose abstraction as dropdowns are common.
3. Further abstract Application-specific parts of the table for Users listing.

I'm going to skip 2, to focus on 1 and 3.

## Slots and the UsersTableComponent

Following the abstraction to its next logical step, we can pull in several more View specific responsibilities into the ViewComponent layer. First we'll start by moving the entire **table** tag into our new component. From here we can abstract the table headers into a ViewComponent method. Lastly, since we iterate over a collection of Users we can lean on ViewComponent's [slots](https://viewcomponent.org/guide/slots.html#component-slots).

{% include blockquote.html quote="Think of slots as a way to render multiple blocks of content, including other components." source_link="https://viewcomponent.org/guide/slots.html" %}


### New UsersTableComponent

Since our UsersTableComponent will render an Application-specific component for User collections, we want each of the available users in the collection to be rendered by the singular UserComponent. We implement this by using the `render_many` slot to define a new collection called `:users` that renders with the UserComponent.

``` ruby
# users_table_component.rb
class UsersTableComponent < ViewComponent::Base
  renders_many :users, UserComponent

  def initialize(pagy:)
    @pagy = pagy
  end

  def headers
    ["name", "roles", "actions"]
  end
end
```

**renders_many :users, UserComponent** will give us a mechanism to define a collection of Users to render with the UserComponent.

### New UsersTableComponent View
```html
<!-- users_table_component.html.erb -->
<table class="bg-gray-300 p-4">
  <thead>
    <tr>
      <% headers.each do |header| %>
        <th><%= header %></th>
      <% end %>
    </tr>
  </thead>

  <tbody>
    <% users.each do |user| %>
      <%= user %> <!-- Will render with the UserComponent -->
    <% end %>
  </tbody>

  <tfoot>
    <tr>
      <td colspan="3">
        <p><strong>Pagination</strong></p>
        <%= pagy_nav(@pagy) %>
      </td>
    </tr>
  </tfoot>
</table>
```

### Updated View index.html.erb
We'll need to adjust our primary Controller view, to define the incoming collection of Users. Slots accomplish this with the DSL `with_slot_name` so for our case `with_users`. The `with_users` method call behaves exactly like the UserComponent class meaning that it can take the same arguments as the initializer. Think of `with_users` as equivalent to `UserComponent.new`.

```html
<!-- Updated index.html.erb -->
<%= render UsersFilterComponent.new %>
<%= render UsersTableComponent.new(pagy: @pagy) do |users_table_component| %>
  <% @users.each do |user| %>
    <% users_table_component.with_users(user:) %>
  <% end %>
<% end %>
```

Our cleanup is really coming along nicely now. One thing we haven't touched on is standardizing render styles. I added a Tailwind CSS class to the **<table>** tag which would be nice to have all our tables utilize. Obviously, this would be most benficial once we have more than a single use case but we'll abstract this to illustrate the point.

## TableComponent

Since we render `<table class="bg-gray-300 p-4">` along with a header, body, and footer portion of the table we can start to abstract these in order to ensure consistent output style. Slots are a natural way of separating a Table into discrete pieces. Note that slots automatically create predicate methods in order to check if the slot contains data. This can be seen below with the `if footer?` conditional check, since not all Tables will have footers.

{% include blockquote.html quote="To test whether a slot has been passed to the component, use the provided #{slot_name}? method." source_link="https://viewcomponent.org/guide/slots.html#predicate-methods" %}


### New TableComponent
```ruby
# table_component.rb
class TableComponent < ViewComponent::Base
  renders_one :header
  renders_one :body
  renders_one :footer
end
```

### New TableComponent View
```html
<!-- table_component.html.erb -->
<table class="bg-gray-300 p-4">
  <thead>
    <tr>
      <%= header %>
    </tr>
  </thead>
  <tbody><%= body %></tbody>

  <% if footer? %>
    <tfoot><%= footer %></tfoot>
  <% end %>
</table>
```

Notice how we split the generic sections of a standard table into the new TableComponent. The concerns are separated based on the thead, tbody, and optional tfoot sections. Implementors for this component can now supply what they want to render and have it appear in the appropriate section based on the slot.

We can now adjust our users_table_component to utilize the general-purpose TableComponent. This ensure that all of our table follow the same styling from our UI library.

### Updated UsersTableComponent
```html
<!-- users_table_component.html.erb -->
<%= render TableComponent.new do |table_component| %>
  <% table_component.with_header do %>
    <% headers.each do |header| %>
      <th><%= header %></th>
    <% end %>
  <% end %>

  <% table_component.with_body do %>
    <% users.each do |user| %>
      <%= user %> <!-- Will render with the UserComponent -->
    <% end %>
  <% end %>

  <% table_component.with_footer do %>
    <tr>
      <td colspan="3">
        <p><strong>Pagination</strong></p>
        <%= pagy_nav(@pagy) %>
      </td>
    </tr>
  <% end %>
<% end %>
```

## Customizing CSS Classes

One thing I like to add to components is a parameter to accept CSS classes for their top level element. This keeps the component opinionated but flexible enough to allow for customization when needed. 

```ruby
# table_component.rb
class TableComponent < ViewComponent::Base
  renders_one :header
  renders_one :body
  renders_one :footer

  attr_reader :classes

  def initialize(classes: "")
    @classes = classes
  end

  def component_classes
    "bg-gray-300 p-4 #{classes}"
  end
end
```

```html
<!-- table_component.html.erb -->
<table class="<%= component_classes %>">
  <thead>
    <tr>
      <%= header %>
    </tr>
  </thead>
  <tbody><%= body %></tbody>
  <tfoot><%= footer %></tfoot>
</table>

<!-- Could be used with -->
<%= render TableComponent.new(classes: "flex bg-red-400") %>

<!-- Which would render: -->
<table class="bg-gray-300 p-4 flex bg-red-400">
  <thead>
    <tr>
      <%= header %>
    </tr>
  </thead>
</table>
```

The [class_names TagHelper](https://api.rubyonrails.org/classes/ActionView/Helpers/TagHelper.html#method-i-class_names) comes in handy here to conditionally toggle classes based on boolean values. With it we can more easily toggle the duplicate bg Tailwind classes.

```ruby
# table_component.rb
class TableComponent < ViewComponent::Base
  renders_one :header
  renders_one :body
  renders_one :footer 

  attr_reader :classes

  def initialize(classes: "")
    @classes = classes
  end

  def component_classes
    class_names({
      "bg-gray-300" => classes.exclude?("bg-"), # Only use `bg-gray-300` if the `classes` argument does not include a Tailwind supported background color
      "p-4" => true,
      classes
    })
  end
end
```

There are additional concepts that can be added here such as variants which is a defined collection of styles based on a type. For example, a ButtonComponent could have a `:critical` variant to highlight the button in red. These are a great way to create different flavors of a component within a UI library ecosystem.

## ApplicationComponent

You can even go one step further and create an ApplicationComponent that defines that each inheriting ViewComponent can pass CSS classes. I generally try to avoid this as it means that every subclass must properly call `super` to pass the classes argument to the ApplicationComponent. That's additional information that could be surprising for implementors.

```ruby
# application_component.rb
class ApplicationComponent < ViewComponent::Base
  attr_reader :classes

  def intialize(**keyword_arguments)
    @classes = keyword_arguments[:classes] if keyword_arguments.key?(:classes)

    super # Pass to ViewComponent::Base
  end
end

# table_component.rb
class TableComponent < ApplicationComponent
  renders_one :header
  renders_one :body
  renders_one :footer

  def initialize(**keyword_arguments, another_setting: "test")
    @another_setting = another_setting

    super(**keyword_arguments) # Pass to ApplicationComponent
  end

  def component_classes
    "bg-gray-300 p-4 #{keyword_arguments[:classes]}"
  end
end
```

This is a basic enhancement but gives you an idea for creating standard ViewComponent functionality.

## Pagination

Lastly, we have several pagination related concerns that are currently burried within UsersTableComponent. Pagination is a common concern in regards to listing collections so abstracting this into a ViewComponent makes a lot of sense. This can give the added benefits of allowing more customized display logic around the previous / next buttons. Additionally, we will utilize [conditional rendering](https://viewcomponent.org/guide/conditional_rendering.html) with then `#render?` method to only show Pagination when the object responds to the appropriate underlying method.

{% include blockquote.html quote="Components can implement a #render? method to be called after initialization to determine if the component should render." source_link="https://viewcomponent.org/guide/conditional_rendering.html" %}

```ruby
class PaginationComponent < ViewComponent::Base
  def initialize(pagination:)
    @pagination = pagination
  end

  private

  # Only render if the following returns true
  def render?
    @pagination.respond_to?(:pages)
  end
end

# users_table_component.rb
class UsersTableComponent < ViewComponent::Base
  renders_many :users, UserComponent
  renders_one :pagination, PaginationComponent

  # ... rest of file
```

``` html
<!-- users_table_component.html.erb -->
<%= render TableComponent.new do |table_component| %>
  <% table_component.with_header do %>
    <% headers.each do |header| %>
      <th><%= header %></th>
    <% end %>
  <% end %>

  <% table_component.with_body do %>
    <% users.each do |user| %>
      <%= user %> <!-- Will render with the UserComponent -->
    <% end %>
  <% end %>

  <% table_component.with_footer do %>
    <tr>
      <td colspan="3">
        <% if pagination? %>
          <%= pagination %>
        <% end %>
      </td>
    </tr>
  <% end %>
<% end %>
```

### Updated index.html.erb
```html
<!-- Updated index.html.erb -->
<%= render UsersFilterComponent.new %>
<%= render UsersTableComponent.new(users: @users) do |users_table_component| %>
  <% @users.each do |user| %>
    <% users_table_component.with_users(user:) %>
  <% end %>

  <% users_table_component.with_pagination(pagination: @pagy) %>
<% end %>
```

You may have noticed above I allowed the Pagination component to accept a generic `pagination` argument. This was to allow for the potential for multiple pagination engines. On its own in an application this wouldn't be very useful since most applications utilize a single mechanism to generate pagination. BUT if this were instead within a UI library you could make the rendered pagination output the expected data to the library by checking this parameter.

```ruby
class PaginationComponent < ViewComponent::Base
  def initialize(pagination:)
    @pagination = pagination # Could be Pagy, Kaminari, something else...
```

Also much like rendering a top-level component, ViewComponent slots allow you to pass arguments onto their underlying component definition when using Component slots.

```ruby
# users_table_component.rb
class UsersTableComponent < ViewComponent::Base
  renders_many :users, UserComponent
  renders_one :pagination, PaginationComponent
```

```html
<!-- index.html.erb -->
<%= render UsersTableComponent.new(users: @users) do |users_table_component| %>
  <% @users.each do |user| %>
    <% users_table_component.with_users(user:) %>
  <% end %>

  <% users_table_component.with_pagination(pagination: @pagy) %>
<% end %>
```

So the above will render the UsersTableComponent, with the Pagination slot, where the Pagination slot is defined by the PaginationComponent, and the PaginationComponent accepts the `pagination` argument.

We've now built upon several of the previous concepts leveraging: Generic slots, Component slots, Slot predicate methods, collection rendering, and conditionally rendered components. Two things we've yet to cover are Previewing and Testing. We'll finish out the article with a brief summary of each.

## Previewing

Like Rails Mailer previews, ViewComponents have preview functionality that works in much the same way. By creating a Preview file and visiting the appropriate route, you'll have a playground to test out your component isolated from your application.

```ruby
# test/components/table_component_preview.rb
class TableComponentPreview < ViewComponent::Preview
  # Route: /rails/view_components/table_component/default
  def default
    render TableComponent.new do |table_component|
      table_component.with_header do
        tag.th do
          "My Header Content"
        end
      end

      table_component.with_body do
        tag.td do
          "My Body Content"
        end
      end
    end
  end

  # Route: /rails/view_components/table_component/with_footer
  def with_footer
    render TableComponent.new do |table_component|
      table_component.with_header do
        tag.th do
          "My Header Content"
        end
      end

      table_component.with_body do
        tag.td do
          "My Body Content"
        end
      end

      table_component.with_footer do
        tag.td do
          "My Footer Content"
        end
      end
    end
  end
end
```

```html
<!-- Default Preview -->
<table class="bg-gray-300 p-4">
  <thead>
    <tr>
      <th>My Header Content</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>My Body Content</td>
    </tr>
  </tbody>
</table>

<!-- With Footer Preview -->
<table class="bg-gray-300 p-4">
  <thead>
    <tr>
      <th>My Header Content</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>My Body Content</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <td>My Footer Content</td>
    </tr>
  </tfoot>
</table>
```

I highly recommend **Lookbook** as it provides additional functionality for changing the output of a component based on parameters.

{% include blockquote.html quote="Lookbook is a UI development environment for Ruby on Rails applications." source_link="https://lookbook.build/" %}

## Testing

[Testing ViewComponents](https://viewcomponent.org/guide/testing.html) is nearly as easy as testing a plain ole Ruby object. Generally there are two types of tests you'll want to write: Unit and System. Our UserComponent defines several methods to be utilized within its corresponding View file. We can test these quite easily with either RSpec or Minitest.

### UserComponent Unit Test

For our Unit test, we'll only focus on the defined methods that are part of the Component. Think of this as testing a Presenter's methods as it is a similiar context.

```ruby
# spec/components/user_component_spec.rb
describe UserComponent, type: :component do
  let(:user) { create(:user)}
  let(:component) { described_class.new(user:) }

  describe "#dropdown_actions" do
    it "returns the correct actions" do
      expect(component.dropdown_actions).to eq([:edit, :destroy])
    end
  end

  describe "#current_roles" do
    it "returns the correct roles" do
      create(:role, name: "Admin", user:)
      create(:role, name: "User", user:)

      expect(component.current_roles).to eq("Admin, User")
    end
  end
end
```

### UserComponent System Test

A system test, tests an end-to-end flow based on what the real world experience may look like. This is a great way to ensure that the component's content, styles, and sometimes elements are rendered as expected.

```ruby
# spec/components/user_component_spec.rb
describe UserComponent, type: :system do
  let(:user) { build_stubbed(:user, name: "Nic Cage") }

  it "renders the User within a Table row" do
    render_inline(UserComponent.new(user:))

    expect(page).to have_content(user.name)
    expect(page).to have_selector("tr")
  end
end
```

The above is a basic system test that uses Capybara under-the-hood to render and test the UserComponent's html output. Anything that can be done with Capybara is fair to utilize here.

## Conclusion

Now that we've abstracted several of the View concerns elsewhere, our top level interface for the Controller is much cleaner. It allows the Controller better focus on business logic giving us a natural place to introduce concepts like Service Objects, Query Objects, and Form Objects.

Here's the end-result for our Controller & View interface:
```ruby
# users_controller.rb
class UsersController < ApplicationController
  USERS_PER_PAGE = 30

  def index
    @pagy, @users = pagy(available_users, limit: USERS_PER_PAGE, page: params[:page])
  end

  private

  def filter_params
    params.permit(filters: :role)
  end

  def available_users
    if filter_params.present?
      User.where(role: filter_params)
    else
      User.all
    end
  end
end
```

```html
<!-- index.html.erb -->
<%= render UsersFilterComponent.new %>
<%= render UsersTableComponent.new(users: @users) do |users_table_component| %>
  <% @users.each do |user| %>
    <% users_table_component.with_users(user:) %>
  <% end %>

  <% users_table_component.with_pagination(pagination: @pagy) %>
<% end %>
```

The next engineer now doesn't need to worry about the various components of this View since they are isolated. This reduces the amount of mental overhead allowing them to focus on the feature requirements. We have dedicated locations for our Component logic, rendering, previewing, and testing.

I've only gone surface level with ViewComponents as there much more depth and nuance to them. If you've had success (or failures) with utilizing ViewComponents as your View layer, I'd love to hear about it in the comments below. Thanks for reading!