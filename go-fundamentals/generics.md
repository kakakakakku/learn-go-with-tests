---
description: Generics
---

# ジェネリクス

**[この章のすべてのコードはここにあります](https://github.com/andmorefine/learn-go-with-tests/tree/main/generics)**

この章では、ジェネリクスの概要を説明し、ジェネリクスについての不安を取り除き、将来のコードをシンプルにする方法について説明します。これを読めば、次の書き方がわかります。

- ジェネリクス引数を取る関数
- 一般的なデータ構造

## 独自のテストヘルパー (`AssertEqual`, `AssertNotEqual`)

ジェネリクスを調べるために、いくつかのテストヘルパーを作成します。

### 整数に対するアサート

基本的なことからはじめて、ゴールに向かって繰り返しましょう。

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %d", got)
	}
}
```

### 文字列に対するアサート

整数の等価性についてアサートできるのは素晴らしいですが、`string` についてアサートしたい場合はどうすればよいでしょうか？

```go
t.Run("asserting on strings", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

エラーが発生します。

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

時間をかけてエラーを読むと、`integer` を期待する関数に `string` を渡そうとしているので、コンパイラが不満を言っていることがわかります。

#### 型安全を振り返る

この本の前の章を読んだことがあれば、もしくは静的型付言語の経験があれば、これは驚くことではありません。Go コンパイラは、操作したい型を定義することで、関数や構造体などを実装することを期待しています。

`integer` を期待する関数に `string` を渡すことはできません。

これは儀式のように感じるかもしれませんが、非常に役立ちます。これらの制約を定義することにより、

- 関数の実装がよりシンプルになります。処理する型をコンパイラに定義することで、**可能性のある実装の数を制限します**。`Person` と `BankAccount` を "追加" することはできません。`integer` を大文字にすることはできません。ソフトウェアでは、制約が非常に役立つことがよくあります。
- 意図しない関数に誤ってデータを渡すことを防げます。

Go では、[インターフェース](../go-fundamentals/structs-methods-and-interfaces.md)を使用して型をより抽象化する方法が提供されているので、具体的な型ではなく、代わりに必要な動作を提供する型を取る関数を設計できるようになります。これにより、型の安全性を維持しながら、ある程度の柔軟性が得られます。

### 文字列もしくは整数を取る関数？（もしくは他のもの）

Go が関数をより柔軟にするためのもう1つの選択肢は、引数の型を "任意" にできる `interface{}` として宣言することです。

代わりにこの型を使用するように引数を変更してみてください。

```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})
```

テストはコンパイルされて、成功するはずです。それらを失敗させようとすると、メッセージを出力するために整数 `%d` フォーマット文字列を使用しているため、出力が少し不安定になることがわかります。そこで、あらゆる種類の値をより適切に出力するために、一般的な `%+v` フォーマットに変更してください。

### `interface{}` の問題

私たちの `AssertX` 関数は非常にシンプルですが、やってることは、[人気のあるライブラリが提供している機能](https://github.com/matryer/is/blob/master/is.go#L150)とあまり変わりません。

```go
func (is *I) Equal(a, b interface{})
```

問題はなんですか？

`interface{}` を使用すると、関数に渡されるものの型について有用な情報をコンパイラに伝えられないため、コードを記述するときにコンパイラが役に立ちません。2つの異なる型を比較してみてください。

```go
AssertNotEqual(1, "1")
```

私たちはこういった場面を回避しようとします。テストはコンパイルされ、期待どおりに失敗しますが、エラーメッセージの `got 1, want 1` は意味不明です。そもそも、文字列を整数と比較したいですか？`Person` と `Airport` を比較しますか？

`interface{}` を受け取る関数を書くことは非常に難しく、バグが発生しやすくなります。これは、制約が _失われて_、コンパイル時にどのような種類のデータを扱っているかという情報がないためです。

これは、**コンパイラが役に立たない**ことを意味していて、代わりに**ランタイム エラー**が発生する可能性が高くなり、ユーザーに影響を与えたり、機能停止を引き起こしたり、さらに悪いことになります。

多くの場合、開発者はこういった汎用関数を実装するためにリフレクションを使用する必要がありますが、これだと読み書きが複雑になり、プログラムのパフォーマンスを低下させる可能性があります。

## ジェネリクスを使用した独自のテストヘルパー

理想的には、扱うすべての型に対して特定の `AssertX` 関数を作成する必要はありません。_どの型_ でも機能するが、[リンゴとオレンジ](https://en.wikipedia.org/wiki/Apples_and_oranges)は比較できない _唯一の_ `AssertEqual` 関数を使用できるようにしたいと考えています。

ジェネリクスは、**制約を定義できるようにする**ことで、抽象化（インターフェイスなど）を作成する方法を提供します。これらを使用すると、`interface{}` が提供するのと同様のレベルの柔軟性を持ちながら、型安全を保持し、呼び出し元により良い開発者体験を提供する関数を作成できます。

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("asserting on strings", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // uncomment to see the error
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %v", got)
	}
}
```

Go でジェネリクス関数を実装するには、"ジェネリクス型を定義してラベルを付ける" という手の込んだ方法である "型パラメーター" を提供する必要があります。

この場合、型パラメーターの型は `comparable` であり、`T` というラベルを付けています。このラベルにより、関数への引数の型を定義できます (`got, want T`)。

比較をする関数内で型 `T` の値で `==` 演算子と `!=` 演算子を使用したいことをコンパイラに定義するために `comparable` を使っています。以下のように型を `any` に変えてみます。

```go
func AssertNotEqual[T any](got, want T)
```

すると、以下のエラーが表示されます。

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

すべての（もしくは `any`）型でこれらの演算子を使用することはできないため、これは非常に理にかなっています。

### [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) を持つジェネリクス関数は `interface{}` と同じですか？

以下の2つの関数を考えてみましょう。

```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

ジェネリクスのポイントは何ですか？`any` は何も説明していませんか？

制約に関して `any` は "任意" を意味し、`interface{}` も同じ意味です。実際に `any` は 1.18 で追加されたもので、_`interface{}` の単なるエイリアス_ です。

ジェネリクスとの違いは、_あなたがまだ特定の型を記述していること_ で、それが意味することは、この関数が _1つの型_ のみで動作するようにまだ制約していることです.

これが意味することは、任意の型の組み合わせ（例えば `InterfaceyFoo(apple, orange)`）で `InterfaceyFoo` を呼び出すことができるということです。しかし、`GenericFoo` は _1つの型_ でのみ動作するため、まだいくつかの制約があります。

可能:

- `GenericFoo(apple1, apple2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("one", "two")`

不可能（コンパイルに失敗する）

- `GenericFoo(apple1, orange1)`
- `GenericFoo("1", 1)`

もし関数がジェネリクス型を返す場合、呼び出し元は、型アサーションを作成するのではなく、型をそのまま使用することもできます。関数が `interface{}` を返すときは、コンパイラはその型について何の保証もできません。

## 次は汎用データ型

[スタック](https://en.wikipedia.org/wiki/Stack_(abstract_data_type))データ型を作成します。スタックは、要件の観点から理解しやすいです。アイテムを "上に" `Push` し、アイテムを "上から" `Pop` して、再びアイテムを取得できるデータ構造です（LIFO - 後入れ先出し）。

簡単にするために、以下の `int` のスタックと `string` のスタックのコードに到達した TDD プロセスは省略しました。

```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

いくつかのアサーション関数を作成しました。

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("got %v, want true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("got %v, want false", got)
	}
}
```

以下がテストです。

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("string stack", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// check stack is empty
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// add another thing, pop it back again
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### 課題

- `StackOfStrings` と `StackOfInts` のコードはほとんど同じです。重複は必ずしも世界の終わりではありませんが、コードの読み取り、書き込み、および保守が必要になります。
- 2つの型でロジックをコピーしているため、テストもコピーする必要がありました。

スタックの _アイデア_ を1つの型に取り込み、それらに対する1つのテストセットを用意したいと考えています。つまり、同じ動作を維持したいので、テストを変更するべきではありません。

ジェネリクスがなければ、以下が _できること_ です。

```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- 前の `StackOfInts` と `StackOfStrings` の実装を、新しく統合した型 `Stack` のエイリアスにしています。
- `values` が `interface{}` の[スライス](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md)になるようにするために、`Stack` から型安全を削除しました。

このコードを試すには、アサート関数から型制約を削除する必要があります。

```go
func AssertEqual(t *testing.T, got, want interface{})
```

こうすると、テストは成功します。誰がジェネリクスを必要としているのでしょうか？

### 型安全性を捨てることの課題

最初の問題は、`AssertEquals` で見たものと同じです。型安全が失われています。オレンジのスタックにリンゴを `Push` できてしまいます。

それはしないというルールがあったとしても、**`interface{}` を返すメソッドを扱うのは不安なので、** コードをうまく扱えるかわかりません。

以下のテストを追加します。

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(firstNum+secondNum, 3)
})
```

型安全を失うことの弱さを示すコンパイラエラーが表示されます。

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

`Pop` が `interface{}` を返す場合、コンパイラはデータが何であるかについての情報を持っていないことを意味し、したがって実行できることを厳しく制限します。整数であることを認識できないため、`+` 演算子を使用できません。

これを回避するには、呼び出し元はそれぞれの値に対して[型アサーション](https://golang.org/ref/spec#Type_assertions)を行う必要があります。

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// get our ints from out interface{}
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // need to check we definitely got an int out of the interface{}

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // and again!

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

このテストから感じる不快感は、`Stack` の実装のすべての潜在的なユーザーに対して繰り返されます。

### 解決するための汎用データ構造

関数にジェネリクス引数を定義できるように、ジェネリクスデータ構造を定義できます。

これが、汎用データ型を特徴とする新しい `Stack` 実装です。

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values)==0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) -1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

以下がテストです。完全な型安全性を備えて、私たちが望むように動作することを示しています。

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// can get the numbers we put in as numbers, not untyped interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

一般的なデータ構造を定義するための構文は、関数への一般的な引数を定義することと同じだと気付くでしょう。

```go
type Stack[T any] struct {
	values []T
}
```

以前と _ほぼ_ 同じですが、**スタックの型によって、操作できる値の型が制限される**ということだけです。

`Stack[Orange]` もしくは `Stack[Apple]` を作成すると、スタックで定義された関数は、作業中のスタックの特定の型のみを返します。

```go
func (s *Stack[T]) Pop() (T, bool)
```

作成するスタックの型に応じて、何かしらの方法で生成される実装の型をイメージできます。

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

リファクタリングが完了したので、同じロジックを何度も確認する必要がないため、文字列スタックのテストを安全に削除できます。

一般的なデータ型を使用すると、以下のようになります。

- 重要なロジックの重複を減らせました。
- `Pop` が `T` を返すようにしたので、`Stack[int]` を作成すると、実際には `Pop` から `int` が返されます。型アサーションを必要とせずに `+` を使用できるようになりました。
- コンパイル時のミスを防止します。リンゴのスタックにオレンジを `Push` することはできません。

## まとめ

この章では、ジェネリクス構文の概要と、ジェネリクスが役立つ理由についてのいくつかのアイデアを紹介しました。ジェネリクスに関する他のアイデアを試すために安全に再利用できる独自の `Assert` 関数を作成し、必要な型のデータを型安全な方法で格納するためのシンプルなデータ構造を実装しました。

### ほとんどの場合、ジェネリクスは `interface{}` を使用するよりも簡単です

静的に型付けされた言語に慣れていない場合、ジェネリクスのポイントがすぐにはわからないかもしれませんが、この章の例で、Go 言語が私たちが望むほどの表現力がないことを示せたと思います。特に `interface{}` を使用すると、コードは以下のようになります。

- 安全性が低くなり（リンゴとオレンジが混ざる）、より多くのエラー処理が必要です。
- あまり表現力がなく `interface{}` はデータについて何も教えてくれません。
- [リフレクション](https://github.com/quii/learn-go-with-tests/blob/main/reflection.md)に頼る可能性が高く、型アサーションなどにより、コンパイル時から実行時に確認が押し出されるため、コードの操作が難しくなり、エラーが発生しやすくなります。

静的に型付けされた言語を使用することは、制約を記述する行為です。うまく使えば、安全で使いやすいだけでなく、実行できる範囲を減らせるため、書きやすいコードを作成できます。

ジェネリクスは、コード内で制約を表現する新しい方法を提供します。これにより、実証されているように、Go 1.18 までは不可能だったコードを統合してシンプルにできます。

### ジェネリクスは Go を Java に変えますか？

- いいえ

Go コミュニティでは、悪夢のような抽象化と不可解なコードベースにつながるジェネリクスについて、たくさんの [FUD（不安、不確実、不信）](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt)があります。通常は "慎重に使用する必要がある" と警告されています。

これは事実ですが、どの言語機能にも当てはまるため、特に役立つアドバイスではありません。

ジェネリクスと同様に、コード内の制約を記述する方法であるインターフェースを定義する能力について不満を言う人はあまりいません。インターフェースを説明するとき、_お粗末な可能性がある_ 設計上の選択を行っています。ジェネリクスは、コードを使用するのを混乱させ、煩わしくする能力において独特ではありません。

### あなたはすでにジェネリクスを使っている

配列、スライス、もしくはマップを使用したことがあれば、 あなたは _すでに一般的なコードの利用者_ です。

```
var myApples []Apples
// You cant do this!
append(myApples, Orange{})
```

### 抽象化は汚い言葉ではありません

[AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html) に染まるのは簡単ですが、コードベースのふりをしないでください。抽象化がまったくないことも悪くはありません。明快さの欠如した異種の関数と型のコレクションではなく、必要に応じて関連する概念を _収集_ するのはあなたの仕事です。そうすれば、あなたのシステムは理解しやすく、変更しやすくなります。

### [機能させ、正しくし、速くする](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

適切な設計上の決定を下すのに十分な情報がない状態で速すぎる抽象化を行うと、人々はジェネリクスの問題に遭遇します。

レッド、グリーン、リファクタリングの TDD サイクルは、**事前に抽象化を想像する**よりも、動作を実現するために _実際に必要な_ コードについて、より多くのガイダンスがあることを意味します。しかし、あなたはまだ注意する必要があります。

ここには厳格なルールはありませんが、有用な共通化ができることがわかるまで、物事を共通化することは避けてください。さまざまな `Stack` 実装を作成したとき、重要なことは、テストに裏打ちされた `StackOfStrings` や `StackOfInts` などの具体的な動作からはじめたことです。実際のコードから実際のパターンを確認し始めることができ、テストに裏打ちされて、より汎用的なソリューションに向けてリファクタリングを検討することができました。

同じコードを3回見たときにのみ共通化するようにアドバイスされることがよくあります。

私が他のプログラミング言語でたどった一般的な道は次のとおりです。

- 何かしらの動作を駆動するための 1 つの TDD サイクル
- 他のいくつかの関連するシナリオを実行するための別の TDD サイクル

> うーん、これらは似ているように見えますが、下手に抽象化をするよりも多少重複していた方がましです。

- 少し寝かせてみる
- 別の TDD サイクルを回してみる

> OK、これを共通化できるか試してみたいと思います。私は TDD を使用しているため、いつでもリファクタリングできます。このプロセスは、設計しすぎる前に実際に必要な動作を理解するのに役立ちました。

- この抽象化はいいですね！テストはまだ成功していて、コードはよりシンプルです。
- 多くのテストを削除できるようになりました。動作の _本質_ を捉え、不要な詳細を削除しました。