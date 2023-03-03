---
description: Templating
---

# HTML Templates

**[この章のすべてのコードはここにあります](https://github.com/andmorefine/learn-go-with-tests/tree/main/blogrenderer)**

私たちは、誰しもがビザンチンビルドシステムで動作するギガバイトのトランスパイルされた JavaScript をベースに構築された最新のフロントエンドフレームワークを使用してウェブアプリケーションを構築したいと考える世界に住んでいます。[必ずしも必要ではないかもしれませんが。](https://quii.dev/The_Web_I_Want)

ほとんどの Go 開発者は、シンプルで安定した高速なツールチェーンを重視していると思いますが、フロントエンドの世界では、この点でしばしば失敗しています。

多くのウェブサイトは [SPA](https://en.wikipedia.org/wiki/Single-page_application) である必要はありません。**HTML と CSS は、コンテンツを配信する優れた方法で**、Go を使用して、HTML を配信するウェブサイトを作成できます。

それでも動的な要素が必要な場合は、クライアント側の JavaScript をいくつか追加するか、サーバー側のアプローチで動的なエクスペリエンスを提供できる [Hotwire](https://hotwired.dev) を試してみることをおすすめします。

Go では [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf) を精巧に使用して HTML を生成できますが、この章では、Go の標準ライブラリに HTML をより簡単で保守しやすい方法で生成するためのツールがいくつかあることを学びます。また、これまで遭遇したことのない種類のコードをテストするためのより効果的な方法についても学びます。

## 構築するもの

[ファイルの読み取り](../go-fundamentals/reading-files.md)の章では、[`fs.FS`](https://pkg.go.dev/io/fs)（ファイルシステム）を受け取り、存在するそれぞれの Markdown ファイルの `Post` のスライスを返します。

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

これが `Post` の定義です。

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

これが解析できる Markdown ファイルの一例です。

```markdown
Title: Welcome to my blog
Description: Introduction to my blog
Tags: cooking, family, live-laugh-love
---
# First recipe!
Welcome to my **amazing recipe blog**. I am going to write about my family recipes, and make sure I write a long, irrelevant and boring story about my family before you get to the actual instructions.
```

ブログソフトウェアを作成する旅を続けるとしたら、このデータを取得して HTML を生成し、ウェブサーバーが HTTP リクエストに応答して返すようにします。

このブログでは、以下の2種類のページを生成します。

1. **記事:** 特定の記事を表示します。`Post` の `Body` フィールドは Markdown を含む文字列なので、HTML に変換する必要があります。
2. **一覧:** 特定の記事を表示するためのハイパーリンクを含んだすべての記事を一覧表示します。

また、ウェブサイト全体で一貫した見た目が必要なので、各ページには、`<html>` のような通常の HTML タグや CSS スタイルシートのリンクを含む `<head>` など、必要なものをすべて配置します。

ブログソフトウェアを構築する場合、HTML を構築してユーザーのブラウザに送信する方法に関して、いくつかの選択肢があります。

私たちは `io.Writer` を受け入れるようにコードを設計します。これは、コードの呼び出し時に以下の柔軟性があることを意味します。

- それらを [os.File](https://pkg.go.dev/os#File) に書き込み、静的に配信できます
- HTML を [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter) に直接書き込めます
- もしくは、実際に何かを書き込むだけです！`io.Writer` を実装しているなら、ユーザーは `Post` から HTML を生成できます。

## 最初にテストを書く

いつものように、急ぎすぎる前に要件について考えることが重要です。この大規模な一連の要件を、私たちが取り組める達成可能な小さなステップに分割するにはどうすればよいでしょうか？

私の考えでは、一覧ページを表示するよりもコンテンツを表示することがより重要です。この機能をリリースして、素晴らしいコンテンツへのリンクを共有できます。コンテンツにリンクできない一覧ページは役に立ちません。

それでも、前述したように記事を表示するとなると、まだ大きく感じます。すべての HTML タグ、Body Markdown の HTML 変換、タグの一覧、など。

この段階では、特定のマークアップについてあまり気にしません。簡単な最初のステップは、記事のタイトルを `<h1>` として表示できるかどうかを確認することです。これは、少し前進できる最小で最初のステップのように**感じます**。

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
}
```

`io.Writer` を受け入れるという決定をしたので、テストも簡単になります。今回のケースでは、あとで内容を検査できる [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer) に書き込みます。

## テストを実行してみます

この本のこれまでの章を読んだことがあれば、これで十分に進められるはずです。パッケージが定義されていないか、`Render` 関数がないため、テストが実行できません。コンパイラのメッセージ通りに試してみて、テストを実行できる状態に到達し、明確なメッセージで失敗することを確認してください。

失敗したテストを実行することは非常に重要です。6ヶ月後に誤ってテストが失敗した場合、**今**、努力して明確なメッセージで失敗を確認したことに感謝するはずです。

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

これは、テストを実行するための最小限のコードです。

```go
package blogrenderer

// if you're continuing from the read files chapter, you shouldn't redefine this
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

テストは、空の文字列は期待値と一致しないことを訴えるはずです。

## 成功させるのに十分なコードを書く

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

ソフトウェア開発は主に学習活動であることを忘れないでください。仕事をしながら発見し、学ぶためには、質の高いフィードバックループが頻繁に得られるような方法で作業をする必要があります。これを行う最も簡単な方法は、小さなステップで作業をすることです。

そのため、現時点ではテンプレートライブラリの使用について気にしていません。"通常の" 文字列テンプレートだけで HTML を作成できます。テンプレート部分をスキップすることで、少し有用な動作を検証でき、パッケージの API の設計作業を少し行いました。

## リファクタリング♪

まだリファクタリングするところはあまりないので、次のイテレーションに進みましょう。

## 最初にテストを書く

これで、非常に基本的なバージョンが機能するようになりました。テストを反復して機能を拡張できます。今回のケースでは、`Post` からより多くの情報を表示します。

```go
	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
```

このように書くと、**嫌な感じがする**ことに注意してください。テストですべてのマークアップを見るのは気分が悪く、本文やすべての `<head>` コンテンツなど、実際の HTML に必要なコンテンツを含めていません。

とは言え、**今のところは**痛みを我慢しましょう。

## テストを実行してみます

説明とタグを表示していないため、期待値した文字列がないというエラーで失敗するはずです。

## 成功させるのに十分なコードを書く

コードをコピーするのではなく、自分で試してみてください。このテストを成功させるのは _少し面倒です_。私が試したとき、最初の試行でこのようなエラーが出ました。

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

改行！誰が気にするの？私たちのテストは正確な文字列一致なので、そうなります。そうするべきですか？今のところは、テストを成功させるために、改行を削除しました。

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1><p>%s</p>", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**うわぁ。** 私が書いたコードの中で最も優れたものではありません。また、マークアップの実装はまだまだ初期段階です。ページにはさらに多くのコンテンツなどが必要になるので、このアプローチが適切ではないことはすぐにわかります。

とても重要なのは、テストが成功していることです。動作するソフトウェアがあることです。

## リファクタリング♪

動作するコードでテストが成功するというセーフティネットがあるので、リファクタリング段階で実装アプローチを変更することを検討できるようになりました。

### テンプレートの紹介

Go には [text/template](https://pkg.go.dev/text/template) と [html/template](https://pkg.go.dev/html/template) という2つのテンプレートパッケージがあり、それらは同じインターフェースを持っています。それぞれが行うことは、テンプレートと一部のデータを組み合わせて文字列を生成できるようにすることです。

HTML バージョンとの違いは？

> パッケージ (html/template) は、コードインジェクションに対して安全な HTML 出力を生成するためのデータ駆動型テンプレートを実装しています。パッケージ (text/template) と同じインターフェイスを提供し、出力が HTML の場合は常に text/template の代わりに使用する必要があります。

テンプレート言語は [Mustache](https://mustache.github.io) と非常によく似ていて、懸念点を適切に分割して、非常にきれいな方法でコンテンツを動的に生成できます。Mustache が好んで言うように、よく使用される他のテンプレート言語と比較すると、それらは非常に制約があるか、"ロジックが少ない" と言えます。これは重要な**意図的な**設計上の決定です。

ここでは HTML の生成に焦点を当てていますが、プロジェクトで複雑な文字列の連結や呪文を実行している場合は、`text/template` に手を伸ばしてコードをクリーンアップすることをおすすめします。

### コードに戻る

以下がこのブログのテンプレートです。

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

この文字列をどこで定義しますか？いくつかの選択肢がありますが、手順を小さくするために、単純な文字列から始めましょう。

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

名前を付けて新しいテンプレートを作成し、テンプレート文字列を解析します。次に `Execute` 関数を使用して、データ（今回のケースでは `Post`）を渡します。

テンプレートは `{{.Description}}` のような記述を `p.Description` の内容に置き換えます。テンプレートは、値を繰り返す `range` や `if` などの基本的なプログラミング構文も提供します。詳細については、[text/template ドキュメント](https://pkg.go.dev/text/template) を参照してください。

**これは純粋なリファクタリングである必要があります。** テストを変更する必要はなく、引き続きテストを成功させる必要があります。重要なことは、私たちのコードは読みやすく、面倒なエラー処理がはるかに少ないことです。

Go のエラー処理の冗長性について不満を言う人がよくいますが、コードを書くためのより良い方法を見つけることができるので、今回のように、そもそもエラーが発生しにくくなります。

### さらにリファクタリング♪

`html/template` を使用することで確実に改善できましたが、コード内で文字列定数として使用するのは良くありません。

- まだかなり読みにくいです。
- IDE やエディタに適していません。シンタックスハイライト、フォーマット、リファクタリングなどの機能がありません。
- HTML のように見えますが、"通常の" HTML ファイルのように操作できません。

私たちがやりたいのは、テンプレートを別々のファイルに保存して、テンプレートをより適切に整理し、HTML ファイルであるかのように操作できるようにすることです。

"templates" というフォルダを作成し、その中に `blog.gohtml` というファイルを作成し、テンプレートをファイルに貼り付けます。

次に [Go 1.16 に含まれる埋め込み機能](https://pkg.go.dev/embed) を使用して、ファイルシステムを埋め込むようにコードを修正します。

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

コードに "ファイルシステム" を埋め込むことで、複数のテンプレートを読み込んで自由に組み合わせることができます。これは、HTML ページの上部のヘッダーやフッターなど、さまざまなテンプレート間で表示ロジックを共有する場合に役立ちます。

### 埋め込み？

埋め込みは[ファイルの読み取り](../go-fundamentals/reading-files.md)で軽く触れました。[標準ライブラリのドキュメントでも説明されています。](https://pkg.go.dev/embed)

> パッケージ埋め込みは、実行中の Go プログラムに埋め込まれたファイルへのアクセスを提供します。
>
> "embed" をインポートする Go ソースファイルは、//go:embed ディレクティブを使用して、コンパイル時にパッケージディレクトリまたはサブディレクトリから読み取ったファイルの内容を文字列、[]byte、または FS 型の変数として初期化できます。

なぜ使いたいのでしょうか？別の方法として、テンプレートを "通常の" ファイルシステムからロードすることも _できます_。ただし、それは、このソフトウェアを使用する場所でテンプレートが正しいファイルパスにあることを確認する必要があることを意味します。あなたは、開発、ステージング、ライブなど、さまざまな環境で仕事をするかもしれません。機能させるには、テンプレートが正しい場所にコピーされていることを確認する必要があります。

埋め込みを使用すると、ビルド時に Go プログラムにファイルが含まれます。これは、プログラムをビルドした後（1回だけ実行する必要があります）、いつでもファイルを使用できることを意味します。

便利なのは、個々のファイルだけでなく、ファイルシステムも埋め込むことができることです。ファイルシステムは [io/fs](https://pkg.go.dev/io/fs) を実装しています。これは、コードがどのような種類のファイルシステムで動作しているかを気にする必要がないことを意味します。

ただし、構成に応じて異なるテンプレートを使用したい場合は、より従来的な方法でディスクからテンプレートをロードすることを意識することをおすすめします。

## 次はテンプレートを "良いもの" にする

テンプレートを1行の文字列として定義したくありません。以下のように、読みやすく、操作しやすいように、スペースをあけることができるようにしたいと考えています。

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

しかし、こうするとテストは失敗します。これは、テストで具体的な文字列が返されることを想定しているためです。

しかし実際には、スペースは気にしません。マークアップに小さな変更を加えるたびにアサーション文字列を入念に更新し続けなければならない場合、このテストを維持することは悪夢です。テンプレートが大きくなるにつれて、この手の編集は管理が難しくなり、作業のコストが制御不能になります。

## 承認テストの紹介

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> ApprovalTests を使用すると、より大きなオブジェクト、文字列、およびファイルに保存できるその他のもの（画像、サウンド、CSV など）を簡単にテストできます。

この考え方は、"ゴールデン" ファイルまたはスナップショットテストに似ています。承認ツールは、テストファイル内の文字列をぎこちなく維持するのではなく、作成した "承認済み" ファイルと出力を比較できます。承認する場合は、新しいバージョンをコピーするだけです。テストを再実行すると、成功します。

`"github.com/approvals/go-approval-tests"` への依存関係をプロジェクトに追加し、テストを次のように編集します。

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

はじめて実行するときは、まだ何も承認されていないため失敗します。

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

以下のような2つのファイルが作成されます。

- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.received.txt`
- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.approved.txt`

受信したファイルには、承認されていない新しいバージョンの出力が含まれています。それを空の承認済みファイルにコピーして、テストを再実行します。

新しいバージョンをコピーすると、変更が "承認" され、テストが成功します。

ワークフローの動作を確認するには、テンプレートを編集して読みやすくします（意味的には同じです）。

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

テストを再実行します。コードの出力が承認されたバージョンと異なるため、新しい "受信済み" ファイルが生成されます。それを見て、変更に納得できる場合は、単に新しいバージョンをコピーしてテストを再実行してください。承認されたファイルは必ずソース管理にコミットしてください。

このアプローチにより、HTML のような大きく醜いものへの変更の管理がはるかに簡単になります。差分ツールを使用して差分を表示および管理でき、テストコードをよりきれいに保つことができます。

![差分ツールを使用して変更を管理する](https://i.imgur.com/0MoNdva.png)

これは、Approval Tests のかなりマイナーな使用方法ですが、テストの武器として非常に便利なツールです。[Emily Bache](https://twitter.com/emilybache) は動画 [interesting video where she uses approval tests to add an incredibly extensive set of tests to a complicated codebase that has zero tests](https://www.youtube.com/watch?v=zyM2Ep28ED8) を公開しています。"組み合わせテスト" は、間違いなく検討する価値のあるものです。

この変更を行った今でも、コードを十分にテストすることのメリットはありますが、マークアップをいじっているときにテストが邪魔になることはあまりありません。

### 私たちはまだ TDD をしていますか？

このアプローチの興味深い副作用は、TDD から遠ざかることです。もちろん、承認済みのファイルを希望する状態に手動で編集し、テストを実行してからテンプレートを修正して、定義した内容が出力されるようにすることもできます。

しかし、それは馬鹿げています！TDD は、作業、具体的には設計を行うための方法です。しかし、だからといって、**すべてに** 無理矢理使用する必要があるわけではありません。

重要なことは、私たちがパッケージの API を設計するための **設計ツール** として TDD を使用したということです。テンプレートの変更の場合、プロセスは次のようになります。

- テンプレートに小さな変更を加える
- 承認テストを実行する
- 出力を目で見て、正しいことを確認します
- 承認する
- 繰り返す

達成可能な小さなステップで作業することの価値を諦めるべきではありません。 変更を小さくする方法を見つけて、テストを再実行し続けて、行っていることに関する実際のフィードバックを取得してください。

テンプレートの周りのコードを変更するなどの作業を開始した場合、もちろん、TDD の作業方法に戻る必要があるかもしれません。

## マークアップを拡張する

ほとんどのウェブサイトは、現在よりもリッチな HTML を使用しています。 まず、`html` 要素と `head`、そしていくつかの `nav` からはじめましょう。一般的には、フッターのアイデアもあります。

サイトにさまざまなページが含まれる場合は、サイトの外観の一貫性を保つために、これらを1箇所で定義する必要があります。Go テンプレートは、他のテンプレートをインポートできる定義をサポートしています。

既存のテンプレートを編集して、top と bottom のテンプレートをインポートします。

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

次に、以下のように `top.gohtml` を作成します。

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My amazing blog!</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, like and subscribe, it really helps the channel guys" lang="en"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Budding Gopher's blog</h1>
        <ul>
            <li><a href="/">home</a></li>
            <li><a href="about">about</a></li>
            <li><a href="archive">archive</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

そして `bottom.gohtml` も作成します。

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

（もちろん、自由に好きなマークアップを定義してください！）

テストを再実行します。新しい "受信済み" ファイルを作成する必要があり、テストは失敗します。もう一度確認して、良ければ、古いバージョンにコピーして承認してください。テストを再実行すると、成功するはずです。

## ベンチマークをいじる言い訳

先に進む前に、コードが何をするかを考えてみましょう。

```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

- テンプレートを解析する
- テンプレートを使用して、記事を `io.Writer` に反映します。

ほとんどの場合、各記事のテンプレートを再解析することによるパフォーマンスへの影響は無視できますが、**無視しない** 努力もしてないので、コードを少し整理する必要があります。

この解析を何度も行わないことの影響を確認するために、ベンチマークツールを使用して関数の速度を確認できます。

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

私のコンピューターでの結果は以下です。

```
BenchmarkRender-8 22124 53812 ns/op
```

テンプレートを何度も再解析する必要をなくすために、解析されたテンプレートを保持する型を作成し、レンダリングを行う関数を作成します。

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {

	if err := r.templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

コードのインターフェイスを変更するため、テストを更新する必要があります。

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

ベンチマークは以下です。

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

テストは引き続き成功する必要があります。ベンチマークはどうですか？

`BenchmarkRender-8 362124 3131 ns/op` でした。前の NS per op は `53812 ns/op` だったので、改善できています！一覧ページなど、レンダリングする他の関数を追加すると、テンプレートの解析を複製する必要がないため、コードが簡素化されます。

## 本来の仕事に戻る

記事の表示に関して、残っている重要な部分は実際に `Body` を表示することです。思い出すと、それは作者が書いた Markdown のはずなので、HTML に変換する必要があります。

これは、読者の演習として残しておきます。そのための Go ライブラリを見つけることができるはずです。承認テストを使用して、検証します。

### サードパーティライブラリのテストについて

**ノート:** サードパーティのライブラリが単体テストでどのように動作するかを明示的にテストすることについて、あまり心配しないように注意してください。

自分が制御していないコードに対してテストを作成するのは無駄であり、メンテナンスのオーバーヘッドが増えてしまいます。[依存性注入](../go-fundamentals/dependency-injection.md)を使用して依存性を制御し、テストのためにその動作をモックしたい場合があります。

このケースでは、Markdown を HTML に変換するのはレンダリングの実装の詳細であると考えていて、承認テストで十分な自信が得られるはずです。

### 一覧を表示する

次に実装する機能は、一覧を表示するために、記事を HTML の順序付きリストとして並べることです。

私たちは API を拡張しているので、TDD の帽子をかぶり直します。

## 最初にテストを書く

一覧ページは最初単純に見えますが、テストを作成すると、設計の選択を迫られます。

```go
t.Run("it renders an index of posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
})
```

1. `Post` のタイトルフィールドを URL のパスの一部として使用していますが、実際には URL にスペースを入れたくないので、スペースをハイフンに置き換えています。
2. `PostRenderer` に `RenderIndex` 関数を追加しました。この関数も `io.Writer` と `Post` のスライスを受け取ります。

ここでテスト後の承認テストアプローチに固執していた場合、制御された環境でこれらの質問に答えることができなかったでしょう。**テストは私たちに考える余地を与えてくれます。**

## テストを実行してみます

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

以下のテスト失敗が得られるはずです。

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
```

## 成功させるのに十分なコードを書く

これは簡単なように _感じますが_、少し嫌な感じがします。私はそれを複数のステップで実装することにしました。

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

最初は個別のテンプレートファイルに煩わされたくありませんでした。ただそれを機能させたかっただけです。テンプレートの解析と分離は、後でできるリファクタリングと考えています。

これはまだ成功しませんが、近づいています。

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

テンプレートコードが `href` 属性のスペースをエスケープしていることがわかります。スペースをハイフンで文字列置換する方法が必要です。`[]Post` を繰り返し、メモリ内で置き換えることはできません。これは、アンカー内のスペースをユーザーに表示する必要があるためです。

いくつかの選択肢があります。最初に検討するのは、関数をテンプレートに渡すことです。

### 関数をテンプレートに渡す

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

_テンプレートを解析する前に_、テンプレートに `template.FuncMap` を追加します。これにより、テンプレート内で呼び出すことができる関数を定義できます。この場合、`{{sanitiseTitle .Title}}` を使用してテンプレート内で呼び出す `sanitiseTitle` 関数を作成しました。

これは強力な機能です。関数をテンプレートに組み込めるため、いくつかの非常に優れたことが可能になります。Mustache とロジックレステンプレートの原則に戻ると、なぜ彼らはロジックレスを提唱したのでしょうか？**テンプレートのロジックの何が問題ですか？**

これまで見てきたように、テンプレートをテストするには**まったく異なる種類のテストを導入する必要がありました。**

動作とエッジケースのいくつかの異なる順列を持つテンプレートに関数を導入すると想像してください。**どのようにテストしますか？** この現在の設計では、このロジックをテストする唯一の手段は、HTML を表示して文字列を比較することです。これは、ロジックをテストするための簡単で適切な方法ではありません。また、_重要な_ ビジネスロジックに求められるものではありません。

承認テストの手法により、これらのテストを維持するコストが削減されましたが、それでも、作成するほとんどの単体テストよりも維持に費用がかかります。管理が簡単になっただけで、マークアップのマイナーな変更には引き続き敏感です。テンプレートの周りに多くのテストを書く必要がないように、コードの設計に引き続き努力する必要があります。また、表示のコード内に存在する必要のないロジックが適切に分離されるように、懸案事項を分離するように努める必要があります。

Mustache の影響を受けたテンプレートエンジンが提供するものは、便利な制約です。あまり頻繁に回避しようとしないでください。**流れに逆らわないでください。** 代わりに、[ビューモデル](https://stackoverflow.com/a/11074506/3193)のアイデアを取り入れてください。ここでは、テンプレート言語にとって便利な方法で、表示する必要があるデータを含む特定の型を構築します。

このようにして、大量のデータを生成するために使用する重要なビジネスロジックが何であれ、HTML やテンプレートの煩雑な世界から離れて、個別に単体テストを行うことができます。

### 懸念事項の分割

では、代わりに何ができますか？

#### `Post` に関数を追加し、それをテンプレートで呼び出します

送信する型のテンプレートコードで関数を呼び出すことができるため、`SanitisedTitle` 関数を `Post` に追加できます。 これによりテンプレートが簡素化され、必要に応じてこのロジックを個別に簡単に単体テストできます。 これはおそらく最も簡単な解決策ですが、必ずしも最も単純であるとは限りません。

このアプローチの欠点は、これがまだ _ビュー_ ロジックであることです。システムの残りの部分にとっては興味深いものではありませんが、今ではコアドメインオブジェクトの API の一部になっています。時間の経過とともに、この手のアプローチは、[神のオブジェクト](https://en.wikipedia.org/wiki/God_object) の作成につながる可能性があります。

#### 本当に必要なデータを含む `PostViewModel` などの専用のビューモデル型を作成します

表示コードをドメインオブジェクト `Post` に結合するのではなく、代わりにビューモデルを使用します。

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

コードの呼び出し元は、`[]Post` から `[]PostView` にマップして、`SanitizedTitle` を生成する必要があります。これをきれいに保つ方法は、マッピングをカプセル化する `func NewPostView(p Post) PostView` を持つことです。

これにより、表示コードをロジックレスに保つことができ、懸念事項を最も厳密に分離することができますが、トレードオフは、記事を表示するためのやや複雑なプロセスです。

どちらの選択肢も問題ありません。この場合、最初の選択肢を使用したくなります。システムを進化させるにつれて、レンダリングの車輪を軽くするためだけにアドホックな関数を追加することに注意する必要があります。ドメインオブジェクトとビューの間の変換がより複雑になると、専用のビューモデルがより便利になります。

`Post` に関数を追加します。

```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

そして、表示コードでより単純な世界に戻れます。

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## リファクタリング♪

最後に、テストを成功させる必要があります。テンプレートをファイル (`templates/index.gohtml`) に移動し、Renderer を構築するときに一度ロードできるようになりました。

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

複数のテンプレートを解析して `templ` に持つことで、`ExecuteTemplate` を呼び出して、必要に応じて表示するテンプレートを指定する必要がありますが、到達したコードが見栄えが良いことに同意していただけることを願っています。

誰かがテンプレートファイルの1つの名前を変更すると、_わずかな_ リスクがあります。バグが発生する可能性がありますが、ユニットテストをすばやく実行すると、これをすばやく検出できます。

これで、パッケージの API 設計に満足し、TDD でいくつかの基本的な動作を得ました。承認を使用するようにテストを変更しましょう。

```go
	t.Run("it renders an index of posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

テストを実行して失敗することを確認してから、変更を承認してください。

最後に、ページごとのエントリーを一覧ページに追加します。

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

テストを再実行し、変更を承認すれば、一覧の作成は完了です！

## Markdown 本文の表示

自分で試してみることをおすすめしましたが、最終的に取ったアプローチは次のとおりです。

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ    *template.Template
	mdParser *parser.Parser
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	parser := parser.NewWithExtensions(extensions)

	return &PostRenderer{templ: templ, mdParser: parser}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p, r))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post, r *PostRenderer) postViewModel {
	vm := postViewModel{Post: p}
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), r.mdParser, nil))
	return vm
}
```

私は期待通りに機能する優れた [gomarkdown](https://github.com/gomarkdown/markdown) ライブラリを使用しました。

これを自分でやろうとすると、本文のレンダリングで HTML がエスケープされていることに気付くかもしれません。これは、Go の html/template パッケージのセキュリティ機能で、悪意のあるサードパーティの HTML が出力されるのを防ぎます。

これを回避するには、レンダリングに送信する型で、信頼できる HTML を [template.HTML](https://pkg.go.dev/html/template#HTML) でラップする必要があります。

> HTML は、既知の安全な HTML ドキュメントフラグメントをカプセル化します。サードパーティの HTML や、タグやコメントが閉じられていない HTML には使用しないでください。このパッケージによってエスケープされたサウンド HTML サニタイザーとテンプレートの出力は、HTML での使用に問題ありません。
>
> この型を使用すると、セキュリティリスクが生じます。カプセル化されたコンテンツは、テンプレート出力に逐語的に含まれるため、信頼できるソースから取得する必要があります。

そこで、**公開されていない**ビューモデル (`postViewModel`) を作成しました。これは、レンダリングに対する内部実装の詳細としてまだ見ているからです。これを個別にテストする必要はなく、API を汚染したくありません。

`Body` を `HTMLBody` に解析できるようにレンダリング時に作成し、テンプレートでそのフィールドを使用して HTML をレンダリングします。

## まとめ

[ファイルの読み取り](../go-fundamentals/reading-files.md)の章とこの章で学んだことを組み合わせると、十分にテストされたシンプルな静的サイトジェネレーターを簡単に作成し、独自のブログを立ち上げることができます。いくつかの CSS チュートリアルを見つけて、見栄えを良くすることもできます。

このアプローチは、ブログにとどまりません。 データベース、API、ファイルシステムなど、あらゆるソースからデータを取得し、それを HTML に変換してサーバーから返すことは、何十年にもわたる単純な手法です。人々は現代のウェブ開発の複雑さを嘆きたがりますが、複雑さを自分自身に負わせているのではないでしょうか？

Go はウェブ開発に最適です。特に、作成しているウェブサイトの実際の要件が何であるかを明確に考えている場合はなおさらです。サーバー上で HTML を生成する方が、React などのテクノロジーを使用して "ウェブアプリケーション" を作成するよりも、優れたシンプルで高性能なアプローチであることがよくあります。

### 何を学んだか

- HTML テンプレートを作成してレンダリングする方法。
- テンプレートをまとめて作成し、関連するマークアップを [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) にして、一貫した見た目を維持する方法。
- 関数をテンプレートに渡す方法と、それについてよく考える必要がある理由。
- "承認テスト" の書き方。これは、テンプレートレンダラーなどの大きな醜い出力をテストするのに役立ちます。

### ロジックのないテンプレートについて

いつものように、これはすべて**懸念の分離**に関するものです。システムのさまざまな部分の責任を考慮することが重要です。重要なビジネスロジックをテンプレートに漏らして、問題を混同し、システムの理解、保守、テストを困難にすることがよくあります。

### HTML だけじゃない

Go には、テンプレートから他の種類のデータを生成するための `text/template` があることを思い出してください。 データを何らかの構造化された出力に変換する必要がある場合は、この章で説明する手法が役立ちます。

### 参考文献とその他の資料

- [John Calhoun の 'Learn Web Development with Go'](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) にはテンプレートに関する優れた記事が多くあります。
- [Hotwire](https://hotwired.dev) - これらの手法を使用して、Hotwire ウェブアプリケーションを作成できます。主に Ruby on Rails を生み出した Basecamp によって構築されましたが、サーバーサイドであるため、Go で使用できます。