---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080s
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# Passwordless Devise

Using Rails Secure ID Tokens

---

# Basic App Setup

```ruby
rails new passwordless_devise_code
cd passwordless_devise_code

bin/rails db:create
bin/rails g controller Landing index
bin/rails g scaffold Pet name species

## config/routes.rb
Rails.application.routes.draw do
  resources :pets
  get 'home',    as: 'home',    to: 'pets#index'
  get 'landing', as: 'landing', to: 'landing#index'
  root to: "landing#index"
end
```

now the landing page and pets are freely available.

---

# Add Devise to add Authentication

```ruby
bundle add devise
bundle install
bin/rails generate devise:install
bin/rails generate devise User
# update the migration to match any added features
bin/rails db:migrate
```

---

# Basic Devise Config

```ruby
# config/environments/development.rb
self.default_url_options = { host: 'http://localhost:3000' }
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
# Rails.application.routes.default_url_options = { host: 'http://localhost:3000' }

# app/views/layouts/application.html.erb
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

---

# Basic Authentication Restrictions

Prevents all pages except 'landing', from non-authenticated access

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :authenticate_user!
  def index
  end
end
```

---

# Email the User's Auth Token

Lets send the passwordless auth tokens via email

```ruby
bin/rails g mailer UserAuth send_link

# app/mailers/user_auth_mailer.rb
class UserAuthMailer < ApplicationMailer
  def send_link(user, url)
    @user = user
    @url  = url
    @host = Rails.application.config.hosts.first
    mail to: @user.email, subject: 'Sign in into #{@host}'
  end
end
```
---

# Email Contents

Lets include a greeting, the sending host and of course the auth_url (we will determine later)

```ruby
# app/views/user_auth_mailer/send_link.html.erb
<p>
  Hi <%= @user.email %>,
  For access to <%= @host %> <%= link_to "Click here", @url %>
</p>

# app/views/user_auth_mailer/send_link.text.erb
Hi <%= @user.email %>,
For access to <%= @host %> follow this link:
<%= @url %>
```

---

# Enable New Devise Auth Method

We need to extend Devise's sessions controller, based on our Secure Global ID knowledge

```ruby
rails g devise:controllers users -c=sessions

# app/controllers/users/sessions_controller.rb:
class Users::SessionsController < Devise::SessionsController
  def auth_token
    auth_token = params[:auth_token]
    user = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
    if user.present?
      sign_in(user)
      flash[:notice] = "Welcome back! #{user.email}"
      redirect_to home_path
    else
      flash[:alert] = 'OOPS - something went wrong.'
      redirect_to root_path
    end
  end
end
```

---

# Code Devise Session Authorization

Tell rails (& Devise) about our newly extended Controller

```ruby
#config/routes.rb
Rails.application.routes.draw do
  resources :pets
  devise_for :users, controllers: { sessions: 'users/sessions' }
  devise_scope :user do  # the path to the NEW AUTHORIZATION based on Tokens
    get 'users/auth/:token', as: :auth_user_session, to: 'users/sessions#auth_token'
  end
  get 'home',    to: 'pets#index'
  get 'landing', to: 'landing#index'
  root           to: "landing#index"
end
```

---

# View Our Routes

We should now have:

```ruby
new_user_session      GET     /users/sign_in(.:format)   users/sessions#new
user_session          POST    /users/sign_in(.:format)   users/sessions#create
destroy_user_session  DELETE  /users/sign_out(.:format)  users/sessions#destroy
auth_user_session     GET     /users/auth(.:format)      users/sessions#auth_token
```

---

# Test the Session Authorization

```ruby
bin/rails c

user = User.first
auth_sgid = user.to_sgid(expires_in: 1.hour, for: 'user_access')
auth_token = auth_sgid.to_s
auth_url = Rails.application.routes.url_helpers
                .auth_user_session_url(login_token: auth_token)
# in a rails app use '.deliver_later' (to make emailing a background job)
UserAuthMailer.send_link(user, auth_url).deliver_now
```
copy this URL into the browser and you should now be on the 'pets' page

---

# Extend Devise with Token Auth

```ruby
mkdir app/lib/devise
mkdir app/lib/devise/models
mkdir app/lib/devise/strategies
touch app/lib/devise/models/token_authenticatable.rb
touch app/lib/devise/strategies/token_authenticatable.rb

# app/lib/devise/models/passwordless_authenticatable.rb
require Rails.root.join('app/lib/devise/strategies/token_authenticatable')
module Devise
  module Models
    module TokenAuthenticatable
      extend ActiveSupport::Concern
    end
  end
end
```

---

# Add A Token Auth Strategy

NOTE: this does not actually sign-someone in, it checks that the account is found and not locked and sends an authorized token.

```ruby
# app/lib/devise/strategies/token_authenticatable.rb
require 'devise/strategies/authenticatable'
require_relative '../../../mailers/user_mailer'

module Devise::Strategies
  class TokenAuthenticatable < Authenticatable
    def authenticate!
      email = params.dig(:user, :email)
      user = User.find_by(email: email)
      if user.present? && !user.locked_at? # and other restrictions as needed
        # this for setting MUST MATCH what's in the Auth Session Controller!
        auth_sgid = user.to_sgid(expires_in: 1.hour, for: 'user_access')
        auth_token = auth_sgid.to_s
        auth_url = Rails.application.routes.url_helpers
                        .auth_user_session_url(login_token: auth_token)
        UserAuthMailer.send_url(user, auth_url).deliver_later
      end
      fail!("An email was sent to you with an authorization link.")s
    end
  end
end
Warden::Strategies.add(:token_authenticatable, Devise::Strategies::TokenAuthenticatable)
```

---

# Load the Devise Strategy

The Devise initializer MUST load the strategy (at the top of the file)

```ruby
# config/initializers/devise.rb
Devise.add_module(:token_authenticatable, {
  strategy:   true,
  route:      :session,
  controller: :sessions,
  model:      'app/lib/devise/models/token_authenticatable'
})
```

Start rails to see if all paths are correct.

---

# Define the Strategy Usage

We need to update the User Model.

```ruby
class User < ApplicationRecord
  before_validation :set_password, on: :create
  # add other necessary Devise features (use only ONE strategy!)
  devise :token_authenticatable, :validatable
  # this will allow the session validations to be happy
  def password_required?
    false # because we aren't using passwords
  end
  private
  # since we aren't using passwords
  def set_password
    tmp_passwd = SecureRandom.alphanumeric(20)
    self.password = tmp_passwd
    self.password_confirmation = tmp_passwd
  end
end
```
Devise has Passwords deeply embedded so its way easier to disable them, than remove them!

NOTE: its easiest to use ONE strategy per model!
---

# Test Token Strategy Integration

If all is setup correctly the devise routes will still have:

```ruby
new_user_session      GET     /users/sign_in(.:format)   users/sessions#new
user_session          POST    /users/sign_in(.:format)   users/sessions#create
destroy_user_session  DELETE  /users/sign_out(.:format)  users/sessions#destroy
auth_user_session     GET     /users/auth(.:format)      users/sessions#auth_token
```

---

# Configure & Generate the Devise Views

We need to update `app/views/users/sessions/new.html.erb` and other forms for features we enabled so that:
1. configure them to be scoped (in the devise initializer)
2. remove the references to passwords in the forms
3. update the session_path references in the form from `session_path(resource_name)` to `user_session_path`

```ruby
# config/initializers/devise.rb:
config.scoped_views = true

# generate the devise views (to override them)
bin/rails generate devise:views users
```

---

# Override the Default Devise Views

Remember to:
* remove passwords
* update the session_path

```ruby
# app/views/users/sessions/new.html.erb
<h2>Log in</h2>
<%= form_for(resource, as: resource_name, url: user_session_path) do |f| %>
  <div class="field">
    <%= f.label :email %><br />
    <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
  </div>
  <% if devise_mapping.rememberable? %>
    <div class="field">
      <%= f.check_box :remember_me %>
      <%= f.label :remember_me %>
    </div>
  <% end %>
  <div class="actions">
    <%= f.submit "Log in" %>
  </div>
<% end %>
<%= render "users/shared/links" %>
```

In our case, we only need to override `app/views/users/sessions/new.html.erb` since we haven't enabled anything but our strategy.

---

# test from console

Now the full workflow should work (assuming you are logged out):
1. go to: http://localhost:3000/pets (and you should be on the landing page)
2. go to: http://localhost:3000/users/sign_in
3. fill out and submit the form
4. click on the generated email (in Mailhog)?
5. now you should be on or able to access: http://localhost:3000/pets
6. now if you go to: http://localhost:3000/users/sign_out - you should no longer have access to: http://localhost:3000/pets

NOTE: I prefer should sgid key life-spans and longer session-lifespans (also configurable)

By default rails sessions have no expiration (until logout), I find this too long. To change this default behavior, you can set the session length with the setting:

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, expire_after: 14.days
```

---

# Resources

* https://github.com/heartcombo/devise
* https://chelseatroy.com/2019/04/08/modifying-authentication-behavior-in-devise/
* https://www.mintbit.com/blog/passwordless-authentication-in-ruby-on-rails-with-devise
* https://github.com/heartcombo/devise/issues/1984 (answer from: josevalim commented on Jul 19, 2012 fixed views)
* https://stackoverflow.com/questions/25374187/creating-a-custom-devise-strategy (answer from: Srikanth Jeeva on Feb 2 '18, fixed strategy loading)
