---
description: Reading files
---

# ファイルの読み取り

- [**この章のすべてのコードはここにあります**](https://github.com/andmorefine/learn-go-with-tests/tree/main/reading-files)
- [これは私が問題に取り組み、Twitch ストリームで質問を受けているビデオです](https://www.youtube.com/watch?v=nXts4dEJnkU)

この章では、いくつかのファイルを読み取り、そこからいくつかのデータを取得し、何か役立つことを行う方法を学びます。

あなたは、友達と一緒にブログソフトウェアを作成しているとします。アイデアは、著者がファイルの先頭にいくつかのメタデータを付けて、Markdown で記事を投稿できるというものです。起動時に、ウェブサーバーはいくつかの `Post` を作成するためにフォルダを読み取り、他の `NewHandler` 関数がこれらの `Post` をブログのウェブサーバーのデータソースとして使用します。

私たちは、ブログ記事ファイルの置かれた特定のフォルダを `Post` のコレクションに変換するパッケージを作成するように依頼されました。

### サンプルデータ

hello world.md
```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

### 期待されるデータ

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## 反復的なテスト駆動開発

私たちは、ゴールに向かってシンプルかつ安全な手順を踏む反復的なアプローチを採用します。

そのためには作業を分割する必要がありますが、["ボトムアップ"](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design) アプローチの罠に陥らないように注意するべきです。

作業を始めるときに想像しすぎる必要はありません。たとえば `BlogPostFileParser` のように、あらゆることをまとめたような抽象化を行うことに誘惑されてしまう可能性があります。

これは _反復的ではなく_、TDD がもたらすはずのタイトなフィードバックを見逃してしまいます。

Kent Beck の言葉:

> 楽観主義はプログラミングをする上で危険です。フィードバックは治療です。

代わりに、私たちのアプローチはできるだけ早く _本当の_ 顧客価値を提供できるように努力するべきです（よく "ハッピーパス" と呼ばれます）。一度、少しの顧客価値をエンドツーエンドで提供したら、残りの要件を反復することは通常は簡単です。

## 見たいテストの種類を考える

始めるときのマインドセットとゴールを思い出しましょう。

- **見たいテストを書きます。** 顧客の観点で、これから作成するコードをどのように使用するかを考えます。
- _方法 (how)_ に気を取られず、_何を (what)_ と _なぜ (why)_ に集中します。

私たちのパッケージは、フォルダを指定できる関数を提供し、いくつかの記事を返す必要があります。

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

これに対してテストを作成するには、いくつかのサンプル記事を含むテストフォルダが必要です。_これは何も悪いことではありませんが_、いくつかのトレードオフがあります。

- 特定の動作をテストするために、テストごとに新しいファイルを作成する必要があるかもしれません
- ファイルのロードに失敗するなど、テストがしにくい動作もあります
- ファイルシステムにアクセスする必要があるため、テストの実行は少し遅くなります

また、ファイルシステムの特定の実装に不必要に結合しています。

### Go 1.16 で導入されたファイルシステムの抽象化

Go 1.16 では、[io/fs](https://golang.org/pkg/io/fs/) パッケージとして、ファイルシステムの抽象化が導入されました。

> fs パッケージは、ファイルシステムへの基本的なインターフェイスを定義します。ファイルシステムは、ホスト OS だけでなく、他のパッケージからも提供できます。

これにより、特定のファイルシステムへの結合度を緩めることができ、必要に応じてさまざまな実装を挿入できるようになります。

> [インターフェースのプロデューサー側では、zip.Reader と同じように、新しい embed.FS 型が fs.FS を実装します。新しい os.DirFS 関数は、OS ファイルのツリーを基にした fs.FS の実装を提供します。](https://golang.org/doc/go1.16#fs)

このインターフェースを使用する場合、パッケージのユーザーは標準ライブラリに組み込まれた多くのオプションを使用できます。Go の標準ライブラリ（たとえば `io.fs`、[`io.Reader`](https://golang.org/pkg/io/#Reader)、[`io.Writer`](https://golang.org/pkg/io/#Writer)）で定義されているインターフェースを活用する方法を学ぶことは、疎結合なパッケージを作成するために不可欠です。これらのパッケージは、ユーザーの手間を最小限に抑えて、想像とは異なるコンテキストで再利用できます。

今回のケースでは、ユーザーは記事を "実際の" ファイルシステムのファイルではなく、Go バイナリに埋め込むことを望んでいるでしょうか？どちらにしろ、_私たちのコードでは気にする必要はありません。_

私たちのテストでは、[testing/fstest](https://golang.org/pkg/testing/fstest/) パッケージが [net/http/httptest](https://golang.org/pkg/net/http/httptest/) で使い慣れたツールと同じように [io/FS](https://golang.org/pkg/io/fs/#FS) の実装を提供してくれます。

この情報を踏まえると、次の方法がより良いアプローチのように感じられます。

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```

## 最初にテストを書く

スコープはできる限り小さく有用なものに保つ必要があります。ディレクトリ内のすべてのファイルを読み取ることができていると確認できれば、それは良い出発点になります。それによって、開発中のソフトウェアに自信が持てるようになります。私たちは、返ってきた `[]Post` の数が、偽のファイルシステムのファイル数と同じであると確認できます。

この章を進めるための新しいプロジェクトを作成します。

- `mkdir blogposts`
- `cd blogposts`
- `go mod init github.com/{your-name}/blogposts`
- `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

テストのパッケージは `blogposts_test` であることに注意してください。TDD に慣れている場合、_コンシューマー駆動_ アプローチを採用します。_コンシューマー_ は内部の詳細まで気にしないため、私たちはテストしたくありません。パッケージ名に `_test` を追加することにより、パッケージの実際のユーザーのように、パッケージからエクスポートされたメンバーのみにアクセスできます。

私たちは [`testing/fstest`](https://golang.org/pkg/testing/fstest/) をインポートすることで、[`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS) 型にアクセスできます。偽のファイルシステムは、`fstest.MapFS` をパッケージに渡します。

> MapFS は、テストで使用するための単純なインメモリファイルシステムで、ファイルまたはディレクトリのパス名（引数を開く）をマップのように表します。

これは、テストファイルのフォルダを管理するよりも簡単で、より速く実行できます。

最後に、コンシューマーの観点から API の使用方法を定義し、正しい数の記事が作成されるかどうかを確認しました。

## テストを実行してみます

```
./blogpost_test.go:15:12: undefined: blogposts
```

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

パッケージは存在しません。新しいファイル `blogposts.go` を作成し、その中に `package blogposts` と書きます。その後、このパッケージをテストにインポートする必要があります。私だと、インポートは以下のようになります。

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

新しいパッケージには、記事のコレクションを返す `NewPostsFromFS` 関数がないため、現在テストはコンパイルできません。

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

よって、関数の雛形を作成してからテストを実行する必要があります。この時点でコードを考えすぎないようにしてください。実行中のテストが、期待どおりに失敗することを確認しようとしているだけです。このステップをスキップすると、予測をスキップして、役に立たないテストを作成してしまう可能性があります。

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

テストは期待どおりに正しく失敗するはずです。

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## 成功させるのに十分なコードを書く

私たちはテストを成功させるために ["スライム化"](https://deniseyu.github.io/leveling-up-tdd/) を使います。

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

Denise Yu は書きました。

>スライム化は、オブジェクトに "雛形" を与えるのに役立ちます。インターフェイスの設計とロジックの実行は2つの課題となるため、テストを戦略的にスライム化することで、一度に1つのことに集中できます。

私たちはすでに構造を持っています。では、代わりに何をしますか？

スコープを減らしたので、ディレクトリを読み取り、存在するファイルごとに記事を作成するだけです。ファイルを開いて解析することについては、まだ心配する必要はありません。

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

[`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) は、指定された `fs.FS` 内のディレクトリを読み取り、[`[]DirEntry`](https://golang.org/pkg/io/fs/#DirEntry) を返します。

エラーが発生する可能性があるため、私たちの理想の世界観はすでに失敗していますが、ここでの焦点は _テストに合格すること_ で、設計を変更することではないため、今のところエラーは無視します。

残りのコードは簡単です。記事を繰り返し処理して、それぞれに対して `Post` を作成し、スライスを返します。

## リファクタリング♪

テストが成功したとしても、新しいパッケージは具体的な実装 `fstest.MapFS` に結合されているため、このコンテキスト以外では使用できません。しかし、そうする必要はありません。`NewPostsFromFS` 関数の引数を変更して、標準ライブラリからのインターフェイスを受け入れます。

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

テストを再実行します。すべて機能するはずです。

### エラー処理

ハッピーパスを機能させることに焦点を当てたとき、そのときはエラー処理を無視していました。機能の反復を続ける前に、ファイルを操作するときにエラーが発生する可能性があることを認識する必要があります。ディレクトリを読み取るだけでなく、個々のファイルを開くときに問題が発生する可能性があります。`error` を返せるように API を変更しましょう（最初にテストを書いて）。

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

テストを実行してください。戻り値の数が間違っているというエラーが表示されるはずです。コードの修正は簡単です。

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

これでテストは成功します。TDD を実践しているあなたは、`fs.ReadDir` からエラーが伝播するコードを書く前に、失敗したテストを確認できなかったことに腹を立てるかもしれません。これを "適切に" 行うには、失敗した `fs.FS` テストダブルを挿入して `fs.ReadDir` が `error` を返すようにする新しいテストが必要です。

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh no, i always fail")
}
```

```go
// later
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

これにより、私たちのアプローチに自信が持てるはずです。使用しているインターフェイスには1つの関数があり、さまざまなシナリオを簡単にテストするためのテストダブルを作成できます。

場合によっては、エラー処理をテストすることは実用的ですが、今回のケースでは、エラーに対して _興味深いこと_ は何もしていません。ただエラーを伝播しているだけなので、新しいテストを書く手間をかける価値はありません。

次の反復では、`Post` 型を拡張して有用なデータを保持します。

## 最初にテストを書く

期待されたブログ記事スキーマの最初の行にあるタイトルフィールドから始めます。

指定された内容と一致するようにテストファイルの内容を変更する必要があります。そうすれば、正しく解析されていることを確認できます。

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// rest of test code cut for brevity
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## テストを実行してみます

```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

テストを実行できるように、新しいフィールドを `Post` 型に追加します。

```go
type Post struct {
	Title string
}
```

テストを再実行すると、明確な理由でテストが失敗します。

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## 成功させるのに十分なコードを書く

それぞれのファイルを開いてタイトルを抽出する必要があります。

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

この時点で焦点を当てているのは、洗練されたコードを書くことではなく、ソフトウェアが機能することです。

少し前進したように感じますが、多くのコードを記述し、エラー処理に関していくつかの仮説を立てる必要がありました。これは、同僚と話し合って最善のアプローチを決定するポイントです。

反復的なアプローチにより、要件の理解が不完全であるというフィードバックがすぐに得られました。

`fs.FS` は、`Open` 関数を使用して名前からファイルを開く方法を提供しています。そのファイルからデータを読み取ります。今のところ、洗練された解析は必要なく、文字列から `Title: ` 文字列を切り取るだけです。

## リファクタリング♪

"ファイルを開くコード" を "ファイルを解析するコード" から分離することで、コードが理解しやすく、操作しやすくなります。

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

新しい関数やメソッドをリファクタリングするときは、引数について注意深く考えてください。あなたが設計をしていて、テストに成功しているため、何が適切か自由に深く考えられます。結合度と凝縮度について考えてみましょう。今回のケースでは、以下のことを自問する必要があります。

> `newPost` は `fs.File` と結合する必要がありますか？この型のすべての関数とデータを使用しますか？私たちが _本当に_ 必要なものは何ですか？

今回のケースでは、`io.Reader` を必要とする `io.ReadAll` への引数としてのみ使用しています。したがって、関数内の結合を緩めて、`io.Reader`を要求する必要があります。

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

`getPost` 関数も同じように引数を作成できます。これは `fs.DirEntry` 引数を受け取りますが、`Name()` を呼び出してファイル名を取得しているだけです。そのすべてが必要なわけではありません。その型から切り離して、ファイル名を文字列として渡しましょう。完全にリファクタリングされたコードは次のとおりです。

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

ここからは、私たちの努力のほとんどは `newPost` 内にきちんと収めることができます。ファイルを開いたり、繰り返し処理したりする問題は解決したので、`Post` 型のデータの抽出に集中できます。技術的には必要ではありませんが、ファイルは関連するものを論理的にグループ化するための優れた方法であるため、`Post` 型と `newPost` を新しく `post.go` ファイルに移動しました。

### テストヘルパー

テストにも気を配る必要があります。`Posts` でアサーションをたくさん作成するので、それを支援するコードを書く必要があります。

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## 最初にテストを書く

テストをさらに拡張して、ファイルから次の行にある説明を抽出しましょう。テストが成功するまで、快適で親しみやすいはずです。

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// rest of test code cut for brevity
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## テストを実行してみます

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

新しいフィールドを `Post` に追加します。

```go
type Post struct {
	Title       string
	Description string
}
```

テストがコンパイルされて、失敗するはずです。

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## 成功させるのに十分なコードを書く

標準ライブラリには、データを1行ずつスキャンするのに役立つ便利なライブラリ [`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner) があります。

> Scanner は、改行で区切られたテキストファイルなどのデータを読み取るための便利なインターフェイスを提供します。

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

便利なことに、ファイルを読むために `io.Reader` を渡せます（疎結合に感謝します）。関数の引数を変更する必要はありません。

`Scan` を呼び出して1行を読み取り、`Text` を使用してデータを抽出します。

この関数は `error` を返すことはありません。今の時点で戻り値の型から削除するのも魅力的ですが、後で無効なファイル構造を処理する必要があることがわかっているため、そのままにしておくこともできます。

## リファクタリング♪

行をスキャンしてテキストを読むことを繰り返します。この操作を少なくとももう1回行うことがわかっています。DRY するための単純なリファクタリングなので、そこから始めましょう。

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

今回、コード行数はほとんど減りませんでしたが、コード行数がリファクタリングのポイントになることはめったにありません。ここで私がやろうとしているのは、行を読むときの _何を (what)_ を _方法 (how)_ から分離して、コードを読者にとってもう少し宣言的なものにすることです。

そして、7 と 13 のマジックナンバーでも機能しますが、あまり説明的ではありません。

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

創造的なリファクタリングの心でコードを眺めている今、readLine 関数でタグを削除できるようにしてみたいと思います。`strings.TrimPrefix` 関数を使用して、文字列からプレフィックスをトリミングする、より読みやすい方法があります。

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

あなたはこの考えが好きかもしれませんし、好きではないかもしれませんが、私は好きです。ポイントは、内部の詳細を自由に操作できるリファクタリング状態にあることで、テストを実行し続けて、まだ正しく動作することを確認できることです。満足できなければ、いつでも前の状態に戻すことができます。TDD アプローチは、アイデアを頻繁に実験する権利を与えてくれるので、優れたコードを書く機会が増えます。

次の要件は、記事のタグを抽出することです。もし理解できている場合は、読み進める前に自分で実装することをおすすめします。これで、適切な反復リズムが得られ、自信を持って次の行を抽出してデータを解析できるはずです。

簡単にするために、TDD の手順は省略しますが、タグを追加したテストを次に示します。

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// rest of test code cut for brevity
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

私が書いたものをコピーして貼り付けるだけでは、自分をだましているだけです。すべてが同じページにあることを確認するために、タグの抽出を含む私のコードを次に示します。

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

うまくいけば、ここで驚きはありません。私たちはタグの次の行を取得するために `readMetaLine` を再利用し、`strings.Split` を使ってそれらを分割できました。

ハッピーパスの最後の繰り返しは、本文の抽出です。

期待されたファイル形式を思い出してください。

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

最初の3行はすでに読み終わっています。次にもう1行読み取り、それを無視すると、ファイルの残りの部分に記事の本文が含まれています。

## 最初にテストを書く

区切り記号を持つようにテストデータを変更し、そして、すべてのコンテンツを取得できるかを確認するためにいくつかの改行を含む本文を含めます。

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

他の箇所と同じように、アサートにも追加します。

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## テストを実行してみます

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

期待どおりです。

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

`Post` に `Body` を追加すると、テストは失敗するはずです。

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## 成功させるのに十分なコードを書く

1. `---` セパレーターを無視するには、次の行をスキャンします
2. スキャンするものがなくなるまでスキャンを続けます

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // ignore a line

	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	body := strings.TrimSuffix(buf.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

- `scanner.Scan()` は、スキャンするデータがまだあるかどうかを表す `bool` を返すので、`for` を使って、データを最後まで読み続けられます
- すべての `Scan()` の後、`fmt.Fprintln` を使用してデータをバッファに書き込みます。Scanner が各行から改行を削除するため、改行を追加したバージョンを使用しますが、それらを管理する必要があります
- 上記の理由により、最後の改行を削除する必要があるため、末尾の改行はありません

## リファクタリング♪

残りのデータを関数に入れるというアイデアをカプセル化することで、将来の読者は、実装の詳細に関心を持たなくても、`newPost` で _何が_ 起こっているのかをすぐに理解できるようになります。

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // ignore a line
	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	return strings.TrimSuffix(buf.String(), "\n")
}
```

## さらに繰り返す

私たちは機能の "鋼の糸" を作成し、ハッピーパスに到達するための最短ルートをたどりましたが、プロダクションの準備が整うまでには、明らかに距離があります。

以下はまだ処理していません。

- ファイルの形式が正しくない場合
- ファイルが `.md` ではない場合
- メタデータフィールドの順序が異なる場合はどうなりますか？許容されますか？それを処理できるべきですか？

重要なのは、動作するソフトウェアがあり、インターフェイスを定義していることです。上記は単なる反復であり、動作を記述して駆動するための追加のテストです。上記のいずれかをサポートするために、_設計_ を変更する必要はなく、実装の詳細のみを変更する必要があります。

ゴールに集中し続けるということは、全体的な設計に影響を与えない問題に行き詰まるのではなく、重要な決定を下し、望ましい動作に対してそれらを検証することを意味します。

## まとめ

`fs.FS` や Go 1.16 のその他の変更により、ファイルシステムからデータを読み取り、簡単にテストする洗練された方法が得られるようになりました。

コードを "実際に" 試してみたい場合は以下です。

- プロジェクト内に `cmd` フォルダを作成し、`main.go` ファイルを追加します
- 以下のコードを追加します

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

- いくつかの Markdown ファイルを `posts` フォルダに追加して、プログラムを実行してください

プロダクションコード間の対称性に注意してください。

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

そして、テストコードは以下です。

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

これは、コンシューマー駆動でトップダウンな TDD が _正しい_ と感じる瞬間です。

私たちのパッケージのユーザーは、テストを見て、何をすべきか、どのように使用するかをすぐに理解することができます。 メンテナーとして、私たちのテストは _消費者の視点から_ 有用であると確信できます。実装の詳細やその他の付随的な詳細をテストしていないため、リファクタリング時にテストが妨げになるのではなく、役立つと確信できます。

[**依存性注入**](../go-fundamentals/dependency-injection.md)のような優れたソフトウェアエンジニアリングプラクティスに依存することで、コードのテストと再利用が簡単になります。

パッケージを作成するときは、それがプロジェクトの内部だけであっても、トップダウンのコンシューマー駆動のアプローチを好みます。これにより、設計を想像しすぎたり、必要のない抽象化を作成したりすることがなくなり、作成したテストが有用であると確認できます。

反復的なアプローチにより、すべてのステップが小さく保たれ、継続的なフィードバックにより、他のよりアドホックなアプローチよりも早く不明確な要件を明らかにすることができました。

### 書き込みは？

これらの新機能には、_ファイルの読み取り_ しかないことに注意してください。仕事でコードを書く必要がある場合は、他の場所を探す必要があります。標準ライブラリが現在提供しているものについて考え続けることを忘れないでください。データを書き込む場合は、コードを疎結合して再利用可能に保つために、`io.Writer` などの既存のインターフェースを活用することを検討する必要があります。

### 参考

- [Ben Congdon は素晴らしい記事を書いています。](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/)この記事では `io/fs` が紹介されており、この章を書くのに役立ちました。
- [ファイルシステムインタフェースに関するディスカッション](https://github.com/golang/go/issues/41190)