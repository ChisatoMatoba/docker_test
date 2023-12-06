## first commit
[Dockerを用いてRuby on Railsの環境構築をする方法( Docker初学者向け )→必要なファイルを用意](https://qiita.com/Yusuke_Hoirta/items/3a50d066af3bafbb8641#%E5%BF%85%E8%A6%81%E3%81%AA%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E7%94%A8%E6%84%8F)まで

## `docker-compose run web rails new . --force --database=mysql --skip-bundle`実行
- `docker-compose run web rails new . --force --database=mysql --skip-bundle`：成功！
  
  →`Dockerfile`が自動で追記・修正される

  → [自動生成されたDockerfile](https://raw.githubusercontent.com/ChisatoMatoba/docker_test/272568c2e27ec19e30df7fa1757339c854d468aa/Dockerfile)
  (これがあとでエラーのもとになります…)

  
- `docker-compose build --no-cache`：エラー
```sh
myapp % docker-compose build --no-cache
 => ERROR [web build 3/6] RUN bundle install &&     rm -rf ~/.bundle/ "/usr/local/bundle"/ruby/*/cache  1.0s
------
 > [web build 3/6] RUN bundle install &&     rm -rf ~/.bundle/ "/usr/local/bundle"/ruby/*/cache "/usr/local/bundle"/ruby/*/bundler/gems/*/.git &&     bundle exec bootsnap precompile --gemfile:
0.976 Your bundle only supports platforms [] but your local platform is x86_64-linux.
0.976 Add the current platform to the lockfile with
0.976 `bundle lock --add-platform x86_64-linux` and try again.
------
failed to solve: process "/bin/sh -c bundle install &&     rm -rf ~/.bundle/ \"${BUNDLE_PATH}\"/ruby/*/cache \"${BUNDLE_PATH}\"/ruby/*/bundler/gems/*/.git &&     bundle exec bootsnap precompile --gemfile" did not complete successfully: exit code: 16
```

## `bundle lock --add-platform x86_64-linux`を実行
これで`docker-compose build --no-cache`：成功！

## database.yml編集
```yml
default: &default
  adapter: mysql2
  encoding: utf8
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: password # docker-compose.yml の MYSQL_ROOT_PASSWORD:に書いたパスワード
  host: db # docker-compose.ymlで命名したMySQLのコンテナ名db
```

## `docker-compose up`失敗
- `docker-compose up`：エラーで失敗
- エラー内容：Missing `secret_key_base` for 'production' environment
```sh
myapp % docker-compose up
[+] Running 2/0
 ✔ Container myapp-db-1   Running                                                                       0.0s 
 ✔ Container myapp-web-1  Created                                                                       0.0s 
Attaching to db-1, web-1
web-1  | => Booting Puma
web-1  | => Rails 7.1.2 application starting in production 
web-1  | => Run `bin/rails server --help` for more startup options
web-1  | Exiting
web-1  | /usr/local/bundle/ruby/3.1.0/gems/railties-7.1.2/lib/rails/application.rb:655:in `validate_secret_key_base': Missing `secret_key_base` for 'production' environment, set this string with `bin/rails credentials:edit` (ArgumentError)
```
### 原因
`docker-compose run web rails new`で自動生成された`Dockerfile`で本番環境向けの記述になっていたため。

## `Dockerfile`をdevelopment用に変更
- product用の記述になっていた部分を変更
<img width="1023" alt="image" src="https://github.com/ChisatoMatoba/docker_test/assets/149556430/27568570-565f-4bd6-98a3-be73d8da8058">
<img width="1005" alt="image" src="https://github.com/ChisatoMatoba/docker_test/assets/149556430/c0f82a78-1496-4caf-b10e-caad84a87df9">

[コミット変更内容ページ](https://github.com/ChisatoMatoba/docker_test/commit/99866685c9b3753291a1ec1fe27952c125bc88c3)

- `docker-compose build --no-cache `：別のエラーで失敗
- エラー内容：`You have already activated error_highlight 0.3.0, but your Gemfile requires error_highlight 0.5.1.`
```sh
myapp % docker-compose build --no-cache  
[+] Building 189.7s (16/19)

=> ERROR [web build 6/6] RUN ./bin/rails assets:precompile RAILS_ENV=development                       0.7s
------
 > [web build 6/6] RUN ./bin/rails assets:precompile RAILS_ENV=development:
0.705 /usr/local/lib/ruby/3.1.0/bundler/runtime.rb:308:in `check_for_activated_spec!': You have already activated error_highlight 0.3.0, but your Gemfile requires error_highlight 0.5.1. Since error_highlight is a default gem, you can either remove your dependency on it or try updating to a newer version of bundler that supports error_highlight as a default gem. (Gem::LoadError)
```

- `Gemfile`では0.4.0になっていたのに…
```
  gem "error_highlight", ">= 0.4.0", platforms: [:ruby]
```
- "error_highlight"の設定を（bundle clearのあとに）0.3.0にしてみたり0.5.1にしてみたりしたが、このエラーが消えなかった

## gem "error_highlight"コメント化
- `Gemfile`の`gem "error_highlight"`行をコメント化
- `docker-compose build --no-cache `：成功！

## `docker-compose up -d`成功
- `docker-compose up -d`:成功
```sh
myapp % docker-compose ps   
NAME          IMAGE       COMMAND                   SERVICE   CREATED             STATUS          PORTS
myapp-db-1    mysql:8.1   "docker-entrypoint.s…"   db        About an hour ago   Up 52 seconds   0.0.0.0:3306->3306/tcp, 33060/tcp
myapp-web-1   myapp-web   "/rails/bin/docker-e…"   web       52 seconds ago      Up 52 seconds   0.0.0.0:3000->3000/tcp
```

## コンテナ内でのrailsコマンド実行について
- 参考サイトの通りではエラーになった
```sh
myapp % docker-compose exec web rails db:create
OCI runtime exec failed: exec failed: unable to start container process: exec: "rails": executable file not found in $PATH: unknown
```
- 解決策：`rails`の前に`bundle exec`を追加→成功！
```sh
myapp % docker-compose exec web bundle exec rails db:create
Created database 'myapp_development'
Created database 'myapp_test'
```

- `http://127.0.0.1:3000/`にアクセス

<img width="635" alt="image" src="https://github.com/ChisatoMatoba/docker_test/assets/149556430/54d3f4df-ff27-45be-ac56-ac838c3d2fa5">
