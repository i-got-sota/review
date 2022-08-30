# コードレビュー

バックエンド 3 つの課題全てお疲れ様です!!
以下にコードレビューをしていきます。

## 全体

### コードについて

- 良い点

  - コメントがあり読みやすいコードとなっており素晴らしいです

  - 検索機能についても、2 種で検索できるようになっており良いですね

  - fstrings 等パイソニックな構文を使用しておりとても良いです。

    - 余裕があれば、リスト内包表記、型ヒント等より高度な構文や、pep8 に代表されるスタイルガイドを参照し、きれいなコードの記述を心がけましょう。

- 改善点

  - デバッグで使用した print 文は削除しておくようにしましょう

  - 変数名や関数名が camelCase で記述されています

    - python では変数名や関数名はスネークケースが推奨されています

  - requirements.txt の更新はしっかりするようにしましょう

    - faker や django-debug-toolbar が requirements に含まれておらず動作しませんでした

  - ユーザーがいない時用の画像が git に上がっておらずデザインが崩れてしまっています

  - 現状ログイン機能がうまく機能しておらず、ログインしていないまま friends その他の画面にアクセスするとエラー画面が表示されてしまいます。

    - ログインが必要なビュー関数には`@login_required`という wrap 関数を使用することでログインが必要なビューを作成することができます

    - class ベースビューでは LoginRequiredMixin を用いることでログインが必要なビューを実装できます。

  - 全体的に model や ORM をうまく活用しきれていない印象を受けます

    - ORM については公式ドキュメントを、余裕があればデータベースの設計の基礎について勉強してみるのも良いかもしれません

### デザイン

- 良い点

  - 独自のデザインが見られ、とても素晴らしいです

- 改善点

  - フォームのデザインにも気を使ってみましょう。

    - forms.py から`field.widget.attrs['class']`属性(解答例参照)や widget-tweaks を使用して、class を指定することで css を反映させることができます。

  - a リンクについてもデザイン変更してみましょう。

    - a リンクの色を変更する

      - color プロパティで通常の文字通り変更可能です

    - 訪れたことのあるリンクの色も変更する

      - :visited 擬似要素で訪れたことのあるリンクについてのスタイルが変更可能です

  - コントラスト比に気をつける(発展的)

    - 例えばログアウトボタンの赤色と文字の色のコントラスト比が不適切でとても見にくく user friendly ではないデザインになっています。

  - リンクが張ってあるボタンについて

    - リンクが文字部分にのみしか効いていないようです

      - UX の観点から、ボタンとして見えている部分をタップしても画面遷移できるようなスタイルが望ましいです

      - 改善方法は様々ありますが解答例では a リンクを block 要素に変更しています

### 各ページ

### ユーザー登録

- signup 時の画像登録(発展)

  - adapter を用いて、ユーザーを save する際に画像登録をフックすることで会員登録時にも画像を登録できます

    - 詳しくは解答例コードを参照してください。

### friend 一覧

- 検索機能については工夫されていてとても素晴らしいです。

- それぞれのユーザー表示についても独自のロジックが組まれており興味深いです

- ユーザー表示についてですが解答のように ORM を使用して処理をかけるとなお良いです

  - この点については解答例や「課題 3」の「正解例と補足」を参照してください。

- 最新のトーク内容、トーク時刻も工夫して表示できるとなお良いです。

- UX の観点からヘッダーについては`position:fixed`で固定できると良いです

- 画面をあふれたユーザーについて最後までスクロールしても最後のユーザーを見ることができないようです

  - フッターの分の padding または margin を下部に入れる等して工夫しましょう。

### トーク

- トーク内容が時間順に並んでいません

  - 現在の仕様ではトーク相手と自分を含めた全てのトーク内容についての sort ができていません

- トークモデルに関してもトークを受け取ったか送ったかをフラグで判定しなくとも一つの model で解決できます。

  ```
  class Talk(models.Model):
    # メッセージ
    talk = models.CharField(max_length=500)
    # 誰から
    talk_from = models.ForeignKey(
        User, on_delete=models.CASCADE, related_name="talk_from"
    )
    # 誰に
    talk_to = models.ForeignKey(User, on_delete=models.CASCADE, related_name="talk_to")
    # 時間は
    time = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return "{}>>{}".format(self.talk_from, self.talk_to)
  ```

  - 上記のような一つのモデルで解決できます。

  ```
  talk = Talk.objects.select_related(
        "talk_from", "talk_to"
    ).filter(
        Q(talk_from=user, talk_to=friend) | Q(talk_to=user, talk_from=friend)
    ).order_by("time")
  ```

  - talkroom におけるクエリについては Q オブジェクトの or 検索を用いて行い select_related を使用して「n+1 問題」が起こることを防ぎます。

### 設定

- 現状のコードでは以下のようになっています。

  ```
  def changeUserName(request):
      form = UserNameForm(request.POST)
      err = ""
      if request.method == "POST":
          if form.is_valid() or request.POST["username"] == request.user.username:
              form = UserNameForm(request.POST, instance=request.user)
              form.save()
              return redirect("setting")
          else:
              err = form.errors
      dic = {"title": "Change User Name", "form": form, "err": err}
      return render(request, "myapp/simpleform.html", dic)
  ```

  - このコードの問題点は

    - 2 行目の form 変数を使用してしまっており、form.is_valid に更新に使用する user も含めたバリデーションを行えていない

      - そのため、GET でアクセスした時のフォーム内容も不適切になってしまっている

      - `if form.is_valid() or request.POST["username"] == request.user.username:`も冗長です

  - 更新処理の際は、以下のように

  - GET の場合は、instance 引数において更新したいクエリを指定する

  - POST の場合、form を一旦変数に格納、その validation を行い、正しければ save するという形を一般的には取ります

    ```
    form = UserNameForm(instance=request.user)
        err = ""
        if request.method == "POST":
            form = UserNameForm(request.POST, instance=request.user)
            if form.is_valid():
                form.save()
                return redirect("setting")
    ```

  - また、このコードではエラーメッセージを変数に格納していますが、form オブジェクトからフロント側で取得できるのではないでしょうか。

- title について

  - `"title": "Change User Name"`等の title の文字色が黒で見にくい

    - せめて文字色だけでも見やすい色に変更しておきましょう。

- メールアドレスとパスワード変更の設定項目がない

  - メールアドレスに関しては allauth と django.dispatch.receiver を使用して実装できます。

  - パスワード変更に関して最も簡単な解決策は django 標準の password 変更ビューを使用することです。

## settings.py について

-

- パスワードの設定が以下のようになっており動作しませんでした。

```
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": """django.contrib.auth
        .password_validation.UserAttributeSimilarityValidator"""
    },
    {
        "" "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]
```

これは以下のように修正すると動作しました。
意図していない設定にせよ、git に上げる際は動作しないものをあげないように気をつけましょう。

```
AUTH_PASSWORD_VALIDATORS = [
    {
        "NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"
    },
    {
        "NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.CommonPasswordValidator",
    },
    {
        "NAME": "django.contrib.auth.password_validation.NumericPasswordValidator",
    },
]
```

- データベース設定や、メール設定は settings に生で書くのはセキュリティ的によろしくないので、環境変数として保存し、読み込むようにしましょう。

  - 環境変数や設定ファイルの分割については「開発入門」の「読む際の注意」の項目に記述があります。

- タイムゾーンや言語を変更して日本仕様の設定に変更しておきましょう

  ```
  LANGUAGE_CODE = "ja"

  TIME_ZONE = "Asia/Tokyo"
  ```
