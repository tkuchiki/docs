## Chef の書き方

Chef をいきなり書くのは慣れないと難しいです。そこで、手動で作業するときのコマンドを列挙して、Chef での書き方に翻訳していくのが良いでしょう。
例えば、HAProxy をインストールして動かしたいとします。
HAProxy を手動でインストールしようとすると、以下のようなコマンドになります。

```shell
yum install -y make gcc gcc-c++
curl http://www.haproxy.org/download/1.6/src/haproxy-1.6.9.tar.gz -o /tmp/haproxy-1.6.9.tar.gz
tar zxf haproxy-1.6.9.tar.gz
cd haproxy-1.6.9
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
rm -f haproxy-1.6.9.tar.gz
rm -rf haproxy-1.6.9
chkconfig haproxy on
service haproxy start
```

これを一つずつ Chef の書き方に翻訳していきます。
まず、cookbook の雛形を作成します。

```shell
bundle exec knife cookbook create haproxy
```

雛形ができたら recipe を書いていきましょう。

```shell
yum install -y make gcc gcc-c++
```

のように、package をインストールするのは以下のように書きます。

```shell
package "make"
package "gcc"
package "gcc-c++"
package "pcre-devel"
package "openssl-devel"
package "zlib-devel"
# package %W{make gcc gcc-c++ pcre-devel openssl-devel zlib-devel} のようにも書けます
```

次はファイルのダウンロードです。

```shell
curl http://www.haproxy.org/download/1.6/src/haproxy-1.6.9.tar.gz -o /tmp/haproxy-1.6.9.tar.gz
```

のように、remote サーバからファイルを取得するのには `remote_file` resource を使います。以下のように書きます。

```ruby
remote_file "/tmp/haproxy-1.6.9.tar.gz" do
  source "http://www.haproxy.org/download/1.6/src/haproxy-1.6.9.tar.gz"
end
```

次は tarball の解凍です。

```shell
tar zxf haproxy-1.6.9.tar.gz
```

Chef の標準 resource に tarball を解凍するものはありません。その場合は、`bash` や `execute` といった任意のコマンドを実行する resource を使います。以下のように書きます。

```ruby
bash "decompress haproxy-1.6.9.tar.gz" do
  cwd "/tmp"
  code "tar zxf haproxy-1.6.9.tar.gz"
end
```

解凍したら次はインストールです。

```shell
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
```
`make` や `install` コマンドを実行する標準 resource も存在しないので、こちらも `bash` resource を使います。直前で `bash` resource を使用しているので、分割したい理由がなければ混ぜてしまいましょう。`bash "decompress haproxy-1.6.9.tar.gz" do...` を以下のように書き換えます。

```ruby
bash "install haproxy-1.6.9" do
  cwd "/tmp"
  code <<EOC
tar zxf haproxy-1.6.9.tar.gz
cd haproxy-1.6.9
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
EOC
end
```

インストールが完了したら不要な tarball とディレクトリを削除します。

```shell
rm -f haproxy-1.6.9.tar.gz
rm -rf haproxy-1.6.9
```

Chef にはファイルとディレクトリを作成、削除する `file` と `directory` resource がありますので、それを使います。以下のように書きます。

```ruby
file "/tmp/haproxy-1.6.9.tar.gz" do
  action :delete
end

directory "/tmp/haproxy-1.6.9.tar.gz" do
  action    :delete
  recursive true
end
```

最後は、HAProxy の起動と自動起動の設定を行います。

```shell
chkconfig haproxy on
service haproxy start
```

Chef には `chkconfig on` と `service` コマンドに相当する `service` resource があります。

```ruby
service "haproxy" do
  # :enable => chkconfig haproxy on
  # :start  => service haproxy start
  action [ :enable, :start ]
end
```

以上で翻訳完了です。と言いたいところですが、`bash` resource は冪等性を保証していないため、そのためのコードが必要になります。それが、`only_if` と `not_if` です。
`only_if` は条件が true、`not_if` は条件が false のときに実行します。これらは、文字列を指定するとコマンドを実行して、exit code が 0 なら true、それ以外なら false になります。`{}` のように ruby block を指定すると ruby コードを実行して真偽値を見ます。

```ruby
# /path/to/file があったら実行
bash "mycommand" do
  code "..."
  only_if { File.exist?("/path/to/file") }
end

# /path/to/file がなかったら実行
bash "mycommand" do
  code "..."
  not_if { File.exist?("/path/to/file") }
end

# haproxy -v の実行結果に 1.6.9 が含まれていれば実行
bash "mycommand" do
  code "..."
  only_if "haproxy -v | grep -qs 1.6.9"
end

# haproxy -v の実行結果に 1.6.9 が含まれていなければ実行
bash "mycommand" do
  code "..."
  not_if "haproxy -v | grep -qs 1.6.9"
end
```

`only_if` と `not_if` をこれまでに書いたコードに適用すると以下のようになります。

```ruby
package "make"
package "gcc"
package "gcc-c++"
package "pcre-devel"
package "openssl-devel"
package "zlib-devel"

remote_file "/tmp/haproxy-1.6.9.tar.gz" do
  source "http://www.haproxy.org/download/1.6/src/haproxy-1.6.9.tar.gz"
  not_if "haproxy -v | grep -qs 1.6.9"
end

bash "install haproxy-1.6.9" do
  cwd "/tmp"
  code <<EOC
tar zxf haproxy-1.6.9.tar.gz
cd haproxy-1.6.9
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
EOC
  not_if "haproxy -v | grep -qs 1.6.9"
end

file "/tmp/haproxy-1.6.9.tar.gz" do
  action :delete
end

directory "/tmp/haproxy-1.6.9.tar.gz" do
  action    :delete
  recursive true
end

service "haproxy" do
  action [ :enable, :start ]
end
```

`not_if "haproxy -v | grep -qs 1.6.9"` は、HAProxy がインストールされていなければ 0 以外(= false)を返しますし、バージョンが 1.6.9 でなくても 0 以外を返します。もし、HAProxy 1.6.9 がインストールされている環境に 1.6.10 を入れ直したいときは、`grep -qs 1.6.9` を `grep -qs 1.6.10` に変更すればインストールし直すコードになります。
しかし、バージョンを書いている部分を一つずつ変更していくのは大変です。そこで、attributes を使います。アプリケーションコードを書くときでも、ハードコードしていると変更に弱いので変数を参照するようにすることはよくあることだと思いますが、Chef も同じです。使い回したい値は attributes に定義して、recipe や template から参照するようにします。attributes は cookbook の attributes ディレクトリ以下に、以下のように書きます。

```ruby
# haproxy/attributes/default.rb
default[:haproxy][:version] = {
  :version => "1.6.9",
}
```

attributes を定義すると、recipe や template から `node[:haproxy][:version]` のように参照することができます。attributes を使うように変更すると、以下のようになります。

```ruby
package "make"
package "gcc"
package "gcc-c++"
package "pcre-devel"
package "openssl-devel"
package "zlib-devel"

remote_file "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  source "http://www.haproxy.org/download/1.6/src/haproxy-#{node[:haproxy][:version]}.tar.gz"
  not_if "haproxy -v | grep -qs #{node[:haproxy][:version]}"
end

bash "install haproxy-#{node[:haproxy][:version]}" do
  cwd "/tmp"
  code <<EOC
tar zxf haproxy-#{node[:haproxy][:version]}.tar.gz
cd haproxy-#{node[:haproxy][:version]}
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
EOC
  not_if "haproxy -v | grep -qs #{node[:haproxy][:version]}"
end

file "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  action :delete
end

directory "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  action    :delete
  recursive true
end

service "haproxy" do
  action [ :enable, :start ]
end
```

設定ファイルを設置していなかったので、`template` resource を使ってファイルを用意します。template は erb で書くことができ、attributes を参照することができます。attributes と `templates/default/haproxy.cfg.erb` を以下のように書きます。

```ruby
# haproxy/attributes/default.rb
default[:haproxy][:version] = {
  :version    => "1.6.9",
  :stats_bind => "localhost:8088",
}
```

```
# templates/default/haproxy.cfg.erb
listen stats
    bind  <%= node[:haproxy][:stats_bind] %>
    mode  http
    stats enable
    stats hide-version
    stats uri /haproxy_stats
```

recipe は以下のように書きます。

```
directory "/etc/haproxy" do
  owner "root"
  group "root"
  mode  0755
end

template "/etc/haproxy/haproxy.conf" do
  source   "haproxy.cfg.erb"
  owner    "root"
  mode     0644
  notifies :reload, "service[haproxy]"
end

service "haproxy" do
  supports :reload => true
  action [ :enable, :start ]
end
```

`service "haproxy" do...` に `supports :reload => true` が追加されています。`template` に `notifies :reload` を書くと、haproxy.cfg を修正したときだけ `service haproxy reload` を実行できるようになります。  
以上で HAProxy cookbook は完成です。しかし、もう少し Chef っぽく書くことができる箇所があります。上記のコードでは、`remote_file` と `bash` resource で同じ条件の `not_if` を使っています。ここを、`remote_file` resource が実行されたときだけ `bash` resource を実行する、というように書けたら処理の流れが良くなりそうです。Chef には `notifies` と `subscribes` という仕組みがあります。`notifies` は resource 実行時に 別の resource を実行し、`subscribers` は別の resource が変更されると実行するようにできます。`notifies` の例を HAProxy に当てはめると以下のようになります。

```ruby
remote_file "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  source   "http://www.haproxy.org/download/1.6/src/haproxy-#{node[:haproxy][:version]}.tar.gz"
  notifies :run, "bash[install haproxy-#{node[:haproxy][:version]}]", :immediately
  not_if   "haproxy -v | grep -qs #{node[:haproxy][:version]}"
end

bash "install haproxy-#{node[:haproxy][:version]}" do
  cwd "/tmp"
  code <<EOC
tar zxf haproxy-#{node[:haproxy][:version]}.tar.gz
cd haproxy-#{node[:haproxy][:version]}
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
EOC
  action :nothing
end
```

`notifies` の第3引数は、何も指定しなければ `:delay` が指定され、キューに貯められて最後にまとめて実行されます。今回のケースでは、`remote_file` resource の処理が終わったらすぐに実行してほしいので `:immediately` をつけます。`bash` resource の `action :nothing` は何もしないという意味で、通常は実行されなくなります。しかし、`remote_file` の notifies に `:run` が指定されているため、notifies から実行するときだけ、指定した action(ここでは `:run`)が実行されます。
以上で完成です。recipe 以下のようになります。

```ruby
package "make"
package "gcc"
package "gcc-c++"
package "pcre-devel"
package "openssl-devel"
package "zlib-devel"

remote_file "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  source   "http://www.haproxy.org/download/1.6/src/haproxy-#{node[:haproxy][:version]}.tar.gz"
  notifies :run, "bash[install haproxy-#{node[:haproxy][:version]}]", :immediately
  not_if   "haproxy -v | grep -qs #{node[:haproxy][:version]}"
end

bash "install haproxy-#{node[:haproxy][:version]}" do
  cwd "/tmp"
  code <<EOC
tar zxf haproxy-#{node[:haproxy][:version]}.tar.gz
cd haproxy-#{node[:haproxy][:version]}
make TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1
make install PREFIX=/usr
install -o 0755 examples/haproxy.init /etc/init.d/haproxy
chkconfig haproxy --add
EOC
  action :nothing
end

directory "/etc/haproxy" do
  owner "root"
  group "root"
  mode  0755
end

template "/etc/haproxy/haproxy.conf" do
  source   "haproxy.cfg.erb"
  owner    "root"
  mode     0644
  notifies :reload, "service[haproxy]"
end

file "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  action :delete
end

directory "/tmp/haproxy-#{node[:haproxy][:version]}.tar.gz" do
  action    :delete
  recursive true
end

service "haproxy" do
  supports :reload => true
  action [ :enable, :start ]
end
```

## よく使う resource まとめ

- `bash` or `execute`
    - resource が存在しない場合にコマンドを記述
- `cookbook_file`
    - ファイルを配置
    - ただし、`template` のように変数の埋め込みはできない
- `directory`
    - ディレクトリの作成、削除、パーミッション変更
- `file`
    - ファイルの作成、削除、パーミッション変更
        - `content` でファイルの内容を記述
- `git`
    - git の操作を行う
- `group`
    - グループを作成
- `link`
    - symlink, hardlink をはる
- `package`
    - パッケージのインストール、アップデート、削除
- `remote_file`
    - リモートサーバ上にあるファイルを取得
- `template`
    - erb のテンプレートからファイルを生成
- `user`
    - ユーザを作成
