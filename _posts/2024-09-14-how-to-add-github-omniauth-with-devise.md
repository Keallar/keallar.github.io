---
layout: post
title: "Как добавить Github OmniAuth аутентификацию?"
date: 2024-09-14
tags: development tutorial rails authentication devise rspec
---
Если в Rails-приложении уже используется Devise, легко!

## Введение
При разработке Rails-приложений библиотека Devise зачастую становится основным инструментом для аутентификации пользователей.
Однако в современных веб-приложениях часто требуется предоставить возможность авторизации через сторонние сервисы, такие как GitHub.
Это можно легко реализовать с помощью библиотеки [omniauth-github](https://github.com/omniauth/omniauth-github), которая интегрируется с Devise и добавляет OmniAuth-аутентификацию.

## Решение

### Шаг 1: Регистрация OmniAuth приложения на GitHub
Первым шагом необходимо зарегистрировать OmniAuth-приложение на GitHub, сделав это по [ссылке](https://github.com/settings/applications/new).
![Registration GitHub Omniauth App](/assets/img/2024-09-8/register_oauth_app.png){:width="80%"}
Пример Homepage URL:
```
http://localhost:4000
```
Пример Authorization callback URL:
```
http://localhost:4000/users/auth/github/callback
```
После успешной регистрации OmniAuth приложения, вы получите `GITHUB_KEY` и `GITHUB_SECRET`, которые понадобятся для дальнейшей настройки.

### Шаг 2: Настройка Rails приложения
Теперь, когда ваше OmniAuth-приложение создано, выполните следующие шаги:
1. Включите необходимые библиотеки. Добавьте следуюшие гемы в `Gemfile`
```ruby
# Gemfile
gem 'omniauth-github'
gem 'omniauth-rails_csrf_protection'
```
2. Выполните команду для установки гемов: `bundle install`
3. Настройте OmniAuth для работы с GitHub в файле `config/initializers/devise.rb`. Добавьте следующую строку:
```ruby
# config/initializers/devise.rb
config.omniauth :github, ENV['GITHUB_KEY'], ENV['GITHUB_SECRET']
```
и создайте в ваш `.env` файл переменные `GITHUB_KEY` и `GITHUB_SECRET` со скопированными выше значениями из зарегистрированного GitHub приложения.
4. Добавьте необходимые поля для хранения данных OmniAuth в модель пользователей. Для этого создайте и примените миграцию:
```bash
rails g migration AddProviderAndUidToUsers provider:string uid:string
rails db:migrate
```
5. Обновите модель User, чтобы она поддерживала OmniAuth-аутентификацию:
```ruby
# app/models/user.rb
devise :database_authenticatable, :registerable, :recoverable, :rememberable,
          :validatable, :omniauthable, omniauth_providers: [:github]
```
6. Создайте контроллер для обработки OmniAuth-колбэков:
```ruby
# app/controllers/users/omniauth_callbacks_controller.rb
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
    skip_before_action :verify_authenticity_token, only: :github

    def github
      @user = User.from_omniauth(request.env['omniauth.auth'])

      if @user
        sign_in_and_redirect(@user, event: :authentication)
        set_flash_message(:notice, :success, kind: 'Github')
      else
        session['devise.github_data'] = request.env['omniauth.auth'].except(:extra)
        redirect_to new_user_registration_url
      end
    end

    def failure
      redirect_to root_path, alert: "Failure. Please try again, #{params[:message]}"
    end
end
```
7. Добавьте метод для обработки данных OmniAuth в модель ``User``:
```ruby
# app/models/user.rb
def self.from_omniauth(auth)
    find_or_create_by(provider: auth.provider, uid: auth.uid) do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
    end
end
```
8. Добавить кнопку аутенфикации через GitHub на форму регистрации
```erb
  <% provider = 'github' %>
  <%= button_to omniauth_authorize_path(resource_name, provider),
                      id: "#{provider}-button",
                      data: { turbo: false } %>
```

### Шаг 3: Тестирование
Для тестирования будут использоваться популярные библотеки `rspec` и `faker`.

1. Создайте `authorazation.rb` файл в папке `spec/support/shared`
2. В этом файле создайте `shared_examples` (подробнее о данной фиче [тут](https://rspec.info/features/3-12/rspec-core/example-groups/shared-examples/)) с контекстом
```ruby
# spec/support/shared/authorization.rb
shared_examples 'with GitHub authentication', :with_github_auth do
    context 'when authenticated with Github' do
      before do
        OmniAuth.config.test_mode = true
        OmniAuth.config.add_mock(:github, OmniAuth::AuthHash.new(Faker::Omniauth.github))
        Rails.application.env_config['devise.mapping'] = Devise.mappings[:user]
        Rails.application.env_config['omniauth.auth'] = OmniAuth.config.mock_auth[:github]
        post user_github_omniauth_callback_path
        follow_redirect!
      end

      after do
        OmniAuth.config.test_mode = false
        Rails.application.env_config['omniauth.auth'] = nil
      end

      it 'successfully authenticated' do
        expect(response).to have_http_status(:success)
      end
    end
end
```
3. В тестах, где требуется аутентификация пользователя через GitHub, добавьте вызов созданного shared_examples:
```ruby
it_behaves_like 'with GitHub authentication'
```
```ruby
# Пример
describe 'POST /competitions' do
    subject(:create_competition_response) do
      post '/competitions/', params: { name: 'New Competition' }
    end

    it_behaves_like 'with GitHub authentication'
end
```

## Заключение
Добавление GitHub OmniAuth через Devise и OmniAuth — это простой и быстрый способ улучшить пользовательский опыт в вашем Rails-приложении. 
Это не только избавляет пользователей от необходимости создавать новые аккаунты, но и делает процесс авторизации безопасным и удобным.