# Rails + Vue.js で ECサイト

## Rails + Vue.jsのプロジェクトを作成する

```
$ rails new kandume_app --webpack=vue
```

## Vue.jsの表示確認
---
基本的に、Railsで用意するビューファイルは１つのみで、そこを差し替えていく。


まずは以下のファイルを作成、編集する。

- app/controllers/home_controller.rb
- config/routes.rb
- app/views/home/index.html.erb


```ruby
# app/controllers/home_controller.rb
  class HomeController < ApplicationController
    def index
    end
  end
```

```ruby
# config/routes.rb
  Rails.application.routes.draw do
    root to: 'home#index'
  end
```

```ruby
# app/views/home/index.html.erb
  <%= javascript_pack_tag 'hello_vue' %>
```

javascript_pack_tagを使用することで、app/javascript/packs以下にあるJSファイルを探してくれる。
インストール時にhello_vue.jsというファイルが生成されているので、これをindexにて読み込ませる。
これでrails sして、「Hello Vue!」と表示されれば表示確認OK。

## devサーバーを設定する
---

Vue.js(と言うよりWebpack)関連のファイルは変更したらコンパイルする必要がある。

コンパイルは、``` bin/webpack ```でする事が出来る。

ただ毎回コンパイルするのは面倒、変更を検出して自動コンパイルするように設定する。

まずは foreman のGemをインストールする。

foremanはProcfileから複数のプロセスを管理することが出来る。

```ruby
#Gemfile
 + gem 'foreman'

```

これでbundle installする。
次に下記2点のファイルを作成。

| ファイル名 | 役割 |
|:-----------:|:------------:|
| bin/server | Procfile.devのコマンドを実行する |
| Procfile.dev | rails s と bin/webpack-dev-server を実行する |

```ruby
# bin/server

#!/bin/bash -i
bundle install
bundle exec foreman start -f Procfile.dev

```

```ruby
# Procfile.dev
web: bundle exec rails s
# watcher: ./bin/webpack-watcher
webpacker: ./bin/webpack-dev-server
```

bin/serverのパーミッションを変更しておく。

```
$ chmod 777 bin/server
```

これでbin/serverを実行してみると、http://localhost:5000で起動する。
（ポート番号が変わる。）

ここで、app.vueの下記の部分を変えて保存すると、コンパイル処理が走り、画面がリロードされる。
適当に変えて遊んでみて、問題なさそうなら大丈夫。
```js
// app/javascript/packs/app.vue
<script>
export default {
  data: function () {
    return {
      // この文字列が画面に表示されている
      message: "Hello Vue!"
    }
  }
}
</script>
```
## APIの準備
---
サーバサイドのAPI部分を実装していく。

## テーブルとモデルの生成
---


| カラム | 型 | |
|:-----------:|:------------:|:------------:|
| name | string | NULL:false|
| price | int |NULL: false|
| genre | string|NULL: false|
| maker | string|NULL: false|
| coment| string|NULL: false|
| stocks | int |NULL: false|
| created_at| DATETIME | NULL:false |
| updated_at | DATETIME | NULL:false |


```
$ rails g model Prodact name:string price:int genre:string maker:string coment:string stocks:int
```

マイグレーションファイルにnull: failseを追記する為、下記のように編集。

```ruby
class CreateProdacts < ActiveRecord::Migration[5.1]
  def change
    create_table :prodacts do |t|
      t.string :name, null: false
      t.int :price, null: false
      t.string :genre, null: false
      t.string :maker, null: false
      t.string :coment, null: false
      t.int :stocks, null: false

      t.timestamps
    end
  end
end
```

マイグレーションを実行する。

```
$ rails db:migrate
```

## コントローラーと返却するJSONファイルを生成
---

アクセスするURLは/api/prodactsのように名前空間を切りたい。

まずはルーティング
| 機能 | HTTPメソッド | URLパターン | URLパターン名 | ヘルパーメソッド | アクション|
|:-----------|:------------|:------------|:------------|:------------|:------------|
|一覧|GET|/prodacts|prodacts|prodacts_path|index|
|詳細|GET|/prodacts/:id|prodact|prodact_path|show|
|新登録画面|GET|/prodacts/new|new_prodact|new_prodact_path|new|
|登録|POST|/prodacts/|prodacts|prodacts_path|create|
|編集画面|GET|/prodacts/:id/edit|edit_prodact|edit_prodact_path|edit|
|更新|PUT|/prodacts/:id|prodact|prodact_path|update|
|削除|DELETE|/prodacts:id|task|prodact_path|destroy|


とりあえず一覧が表示できるところまで行うのでアクションはIndexのみ。

```ruby
#config/routes.rb
Rails.application.routes.draw do
  root to: 'home#index'

 namespace :api, format: 'json' do
   resources :prodacts, only: [:index]
 end
end
```

次はコントローラを作成。


コントローラーはapp/controllersの中にapiというディレクトリを作成し、そこに作る。

```ruby
class Api::ProdactsController < ApplicationController
  # GET /prodacts
  def index
    @prodacts = Prodact.all
    render json: @prodacts, status: :index
  end
end
```

ビューのJSONファイルも同様にviews/api/prodacts以下に作成する。

```ruby
json.set! :prodacts do
  json.array! @prodacts do |prodact|
    json.extract! prodact, :id, :name, :genre,:maker, :coment, :stocks, :created_at, :updated_at
  end
end
```

seeds.rbを作成して、curlコマンドで確認

```ruby
Prodact.create(name:"やきとりガーリックペッパー味",price:147,genre:"meat",maker:"ホテイ",stocks:100,coment:"国産鶏肉を炭火で焼き、にんにくの風味とペッパーの辛さを組み合わせた香ばしく食欲をそそる味に仕上げました。")
Prodact.create(name:"やきとり柚子こしょう味",price:137,genre:"meat",maker:"ホテイ",stocks:100,coment:"こちらの商品は国産鶏肉を炭火で焼き、塩味をベースに柚子こしょうで仕上げました")
Prodact.create(name:"やきとりたれ味",price:155,genre:"meat",maker:"ホテイ",stocks:100,coment:"国産鶏肉を炭火で香ばしく焼き上げ、フルーティーでまろやかな甘さの醤油だれで仕上げました。焦がし醤油を加えるとともに、固形量は変わらず、液量を抑えることで、煮鶏感を解消し、焙焼感をアップしました。又、炭火焼のもつ香ばしさを最大限に引き出しました。おつまみはもちろんのこと、お弁当のおかずやサラダのトッピング、料理の具材、炊き込みご飯等幅広くご利用いただけます。")
Prodact.create(name:"やきとり塩味",price:126,genre:"meat",maker:"ホテイ",stocks:100,coment:"国産鶏肉を炭火で焼き、旨味とコクを感じる塩味で仕上げた「炭火のやきとり」です。")
Prodact.create(name:"さばで健康 みそ煮 160g",price:427,genre:"fish",maker:"はごろも",stocks:100,coment:"さばの持っている栄養素をおいしく召し上がっていただくために、みそ味に仕上げました。もう1品おかずがほしいときや、ちょっとしたおつまみにも、手軽にお召し上がりいただけます。ご飯のお供にも最適です。そのままでもお召し上がり頂けますが、小鉢に移し、電子レンジで約1分程度温めてもおいしく召し上がれます。")
Prodact.create(name:"オリーブオイルサーディン(いわしの油漬) 100g",price:216,genre:"fish",maker:"宝幸",stocks:100,coment:"頭と尾、内臓を取り除いたいわしをBASSO社のイタリア産100%のピュアオリーブオイルと共にパックしたいわし油漬の缶詰です。惣菜としても料理の素材としてもお使いいただけます。")
```

DBに適用する

```
$ rails db:seed
```

なお、リセットする場合は、

```
$ rails db:setup
```

curl コマンドを使用してAPIの確認をする

```
$ curl localhost:5000/api/prodacuts
```

## コンポーネントを使ってヘッダーを作成
---
ここからVue.jsを中心に画面を作っていく。


まずはヘッダーを作成する。


元となるビューファイルはRailsのindex.html.erbになるので、こちらにVue.jsを載せられるようにする。

```html
<div id="app">
  <navbar></navbar>
</div>
<%= javascript_pack_tag 'prodacts' %>
```

<navbar>というタグがありますが、Vue.js側でこのタグとコンポーネントを紐付け、表示します。

また、新しくtodo.jsというファイルをapp/javascript/packsに作成します。

```js
import Vue from 'vue/dist/vue.esm.js'

var app = new Vue({
  el: '#app',
});
```
ここで、import Vue from 'vue/dist/vue.esm.js'としています。
これは、後ほどコンポーネントを使用する際に完全ビルドする必要があるからだそうです。

これでindex.html.erb内の<div id="app">にマウントされます。
このまま実行しても特に何もありません。

それではコンポーネントを作成します。
packsの下にcomponentsディレクトリを作成して、そこにheader.vueを作成します。
コンポーネントは.vueで作成します。

```html
<template>
  <div>
    <nav>
      <div class="nav-wrapper">
        <a class="brand-logo" href="/">Kandume Store</a>
        <ul class="nav-list">
          <li>
            <a href="#">Cart</a>
          </li>
          <li>
            <a href="#">Contact</a>
          </li>
          <li>
            <a href="#">About</a>
          </li>
        </ul>
      </div>
    </nav>
  </div>
</template>

<script>
export default {};
</script>

<style scoped>
.nav-wrapper {
  width: margin 0 auto;
  height: 60px;
  background-color: aqua;
  color: antiquewhite;
}
.brand-logo {
  float: left;
  margin: 10px;
}
.nav-list li {
  float: right;
  margin: 10px;
  list-style: none;
}
</style>
```

これをprodact.jsに登録します。


```ruby
#app/javascript/packs/todo.js
import Vue from 'vue/dist/vue.esm.js'
import Header from './components/header.vue'

var app = new Vue({
  el: '#app',
  components: {
    'navbar': Header,
  }
});
```

navbarという名前でコンポーネントとして登録します。
（headerだと<header>タグがすでにHTML5に存在しているため。）
これで<navbar>タグが使用できるようになりました。

サーバーを再起動してアクセスするとヘッダーができているのではないでしょうか？

## Vue-Routerを使用してSPAっぽく
---
Vue-Routerを使用することで、登録されたパスとコンポーネントで画面内を差し替えることができます。

yarnを使ってvue-routerを追加します。

```
$ yarn add vue-router
```

今回は「商品一覧(メイン画面)」、「アバウト（おまけ）」、「コンタクト（おまけ）」
を用意する。

まずはコンポーネントを作成する。

```html
<!--app/javascript/packs/components/index.vue-->
<template>
  <div>
    <p>Index</p>
  </div>
</template>
```

```html
<!--app/javascript/packs/components/about.vue-->
<template>
  <div>
    <!-- 内容はお好みで -->
    <p>This is a sample of TODO application with Vue.js and Ruby on Rails.</p>
    <p>Sample code is <a href="https://github.com/naoki85/todo_app_with_vue_and_rails" target="_blank">here.</a></p>
  </div>
</template>
```

```html
<!--app/javascript/packs/components/contact.vue-->
<template>
  <div>
    <!-- 内容はお好みで -->
    <p>If you want to contact me, you send mail to below address.</p>
    <p>test@example.com</p>
  </div>
</template>
```

この、コンポーネントとパスを登録するrouter.jsを作成する。
こちらもrouterディレクトリを作成してそちらに作成する。

```js
//app/javascript/packs/router/router.js
import Vue from 'vue/dist/vue.esm.js'
import VueRouter from 'vue-router'
import Index from '../components/index.vue'
import About from '../components/about.vue'
import Contact from '../components/contact.vue'

Vue.use(VueRouter)

export default new VueRouter({
  mode: 'history',
  routes: [
    { path: '/', component: Index },
    { path: '/about', component: About },
    { path: '/contact', component: Contact },
  ],
})
```

パスとコンポーネントを結びつけます。
また、mode: 'history'とすることで、HTMLのhistory APIを使用して、一見同じビュー内ですがURLを書き換えることができます。
HTML5 Historyモード

また、VueRouterを使用すると、\<router-link>と\<router-view>というタグが使用できます。
\<router-link>は、\<a>タグとして変換されますが、画面遷移ではなくVueRouterに登録されたパスからコンポーネントを探します。
そして\<router-view>の部分に表示します。

ヘッダーの各リンクを修正します。

```js
- <li><a href="/">Top</a></li>
- <li><a href="/about">About</a></li>
- <li><a href="/contact">Contact</a></li>
+ <li><router-link to="/">Top</router-link></li>
+ <li><router-link to="/about">About</router-link></li>
+ <li><router-link to="/contact">Contact</router-link></li>
```

それぞのコンポーネントが表示される部分をinde.html.erbに作る

```html
<!--app/views/home/index.html.erb-->

<div id="app">
   <navbar></navbar>
+  <div class="container">
+    <router-view></router-view>
+  </div>
 </div>

 <%= javascript_pack_tag 'todo' %>
```

最後に、prodact.jsに追加する。
```js
//app/javascript/packs/todo.js
import Vue from 'vue/dist/vue.esm.js'
+ import Router from './router/router'
  import Header from './components/header.vue'

  var app = new Vue({
+   router: Router,
    el: '#app',
    components: {
      'navbar': Header,
    }
  });
```

これで、ヘッダーの各リンクを押すと本文が切り替わるのではないでしょうか？
URLもHistoryモードのおかげで書き換わっています。

ただ、例えばhttp://localhost:5000/aboutでリロードすると、Rails側でエラーになってしまいます。
たしかにroutes.rbで登録していません。
とりあえず、/aboutでも/contactでもHome#indexにとぶよう記述します。

```ruby
# config/routes.rb
Rails.application.routes.draw do
   root to: 'home#index'
+  get '/about',   to: 'home#index'
+  get '/contact', to: 'home#index'
```

これでhttp://localhost:5000/aboutにアクセスするとわかりますが、ちゃんとURLからAboutのコンポーネントを表示してくれます。

# Axiosを使ってAPI通信

axiosは、Ajax通信ライブラリです。
まずはこれをインストールします。

```
$ yarn add axios
```

Axiosを使用してindex.vueにてAPI通信して商品一覧を表示したい。

## axiosで試してみたがうまく行かない。使い方を再度勉強し直す必要がある。

そこで、使い慣れたfetchで実装する。

```js
// index.vue

created: function() {
    console.log("Vueインスタンス作成します！");
    const json = fetch("/api/prodacts");
    console.log(json);
    Promise.resolve(json)
      .then(result => {
        return result.json();
      })
      .then(response => {
        console.log(response);
        this.prodacts = response;
      })
      .catch(error => console.log(error));
    console.log("商品データ取得終了しました。");
```

フックするのがcreatedの時が良いのか、mountedの時が良いのか考える糸口が見つからなかったのでとりあえず、
createで実装。

表示のモックを作成。

```html
<template>
  <div class="prodact-box">
    <div v-for="prodact in prodacts">
      <!-- <img v-bind:src="imgsrc"> -->
      <!-- <img src="product-image/1.jpg" alt="商品画像"> -->
      <br>
      <img v-bind:src="'product-image/' + prodact.id +'.jpg'">
      <p>id: {{prodact.id}}</p>
      <p>title: {{prodact.name}}</p>
      <p>price: {{prodact.price}}</p>
    </div>
  </div>
</template>
```

ここでポイントは、
```html
<img v-bind:src="'product-image/' + prodact.id +'.jpg'">
```
である。

v-bindした属性の属性値の中にはjsのオブジェクトが直接書ける。
computedの中に書いていた時は上手くいかなかったがこれで上手くいった。



