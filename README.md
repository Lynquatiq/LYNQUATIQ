```ruby name=routes.rb
Rails.application.routes.draw do
  post 'authenticate', to: 'authentication#authenticate'
end
```

```ruby name=authentication_controller.rb
class AuthenticationController < ApplicationController
  # POST /authenticate
  def authenticate
    user = User.find_by(lynquatiqmore@gmail.com: params[:email])

    if user&.authenticate(params[:password])
      token = JsonWebToken.encode(user_id: user.id)
      render json: { auth_token: token }, status: :ok
    else
      render json: { error: 'Invalid email or password' }, status: :unauthorized
    end
  end
end
```

```ruby name=user.rb
class User < ApplicationRecord
  has_secure_password
end
```

```ruby name=json_web_token.rb
class JsonWebToken
  SECRET_KEY = Rails.application.secrets.secret_key_base.to_s

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET_KEY)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET_KEY)[0]
    HashWithIndifferentAccess.new decoded
  rescue
    nil
  end
end
```

```ruby name=application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_request

  private

  def authenticate_request
    header = request.headers['Authorization']
    header = header.split(' ').last if header
    decoded = JsonWebToken.decode(header)
    @current_user = User.find(decoded[:user_id]) if decoded
  rescue ActiveRecord::RecordNotFound => e
    render json: { errors: e.message }, status: :unauthorized
  rescue JWT::DecodeError => e
    render json: { errors: e.message }, status: :unauthorized
  end
end
```

```yaml name=database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: <%= ENV['DATABASE_USERNAME'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
  host: <%= ENV['DATABASE_HOST'] %>

development:
  <<: *default
  database: my_app_development

test:
  <<: *default
  database: my_app_test

production:
  <<: *default
  database: my_app_production
  username: my_app
  password: <%= ENV['MY_APP_DATABASE_PASSWORD'] %>
```

This will create an endpoint `/authenticate` where you can send a POST request with `email` and `password` to get a JWT token. The `User` model should already have `email` and `password_digest` fields. The `has_secure_password` method in the `User` model will handle password encryption.
