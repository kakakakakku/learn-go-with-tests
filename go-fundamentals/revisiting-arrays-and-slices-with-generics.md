---
description: Revisiting arrays and slices with generics
---

# ジェネリクスを使用した配列とスライスの再検討

**[この章のすべてのコードはここにあります](https://github.com/andmorefine/learn-go-with-tests/tree/main/arrays)**

[配列とスライス](../go-fundamentals/arrays-and-slices.md)で書いた `SumAll` と `SumAllTails` の両方を見てください。持っていない場合は、[配列とスライス](../go-fundamentals/arrays-and-slices.md)の章のコードをテストと共にコピーしてください。

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

繰り返しのパターンはありますか？

- 何かしらの "初期化した" 結果値を作成します。
- コレクションを反復処理して、何かしらの操作（もしくは関数）を結果とスライス内の次の項目に適用して、結果に新しい値を設定します。
- 結果を返します。

このアイデアは、関数型プログラミングの界隈でよく話題になり、"reduce" もしくは [fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) と呼ばれます。

> 関数型プログラミングでは、fold（reduce、accumulate、aggregate、compress もしくは inject と呼ばれます）は、再帰的なデータ構造を分析し、特定の結合操作を使用して再帰的に処理した結果を再結合し、戻り値を構築する高階関数の種類を指します。通常、fold は、結合関数、データ構造の最上位ノード、および場合によっては特定の条件下で使用されるいくつかのデフォルト値とともに提示されます。次に、関数を体系的な方法で使用して、データ構造の階層の要素を結合します。

Go には常に高階関数があり、バージョン 1.18 では[ジェネリクス](../go-fundamentals/generics.md)も含まれているため、より広い分野で議論されているこれらの関数のいくつかを定義できるようになりました。現実逃避をしても意味がありません。これは Go エコシステムの外では非常に一般的な抽象化であり、理解することは重要です。

今、あなたたちの中で何人かはきっとこれにうんざりしていることでしょう。

> Go はシンプルであるべき

**簡単さとシンプルさを混同しないでください。** ループを実行したり、コードをコピーして貼り付けたりするのは簡単ですが、必ずしもシンプルではありません。簡単さとシンプルさの詳細は、[Rich Hickey の素晴らしいトーク - Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4) をご覧ください。

**慣れていないことと複雑なことを混同しないでください。** fold と reduce は、最初は恐ろしくコンピューター科学的に聞こえるかもしれませんが、実際には、非常に一般的な操作の抽象化です。コレクションを集めて、ひとつのアイテムにまとめます。一歩下がってみると、おそらくこれを _たくさん_ やっていることに気付くでしょう。

## 一般的なリファクタリング

ピカピカの新しい言語機能を使って人々が犯しがちな過ちは、具体的なユースケースを持たずに使い始めることです。彼らは推測や当てずっぽうに頼って努力を導きます。

ありがたいことに、"便利な" 関数を作成し、それらの周りにテストを用意したので、TDD のリファクタリング段階でアイデアを自由に試すことができ、試みているものは何でも、単体テストを介してその値を検証できることがわかります。

リファクタリングのステップでコードをシンプルにするためのツールとしてジェネリクスを使用すると、時期尚早な抽象化ではなく、有用な改善につながる可能性がはるかに高くなります。

試して、テストを再実行すれば安心です。変更が気に入ったら、コミットできます。気に入らない場合は、変更を元に戻しましょう。この実験の自由は、TDD の本当に大きな価値の1つです。

ジェネリクス構文は[前の章]([ジェネリクス](../go-fundamentals/generics.md))から精通している必要があります。独自の `Reduce` 関数を書いて、`Sum` と `SumAllTails` 内で使用してみてください。

### ヒント

最初に関数の引数について考えると、小さくて有効なソリューションの一覧が得られます。

- reduce したい配列
- ある種の結合関数、もしくは _アキュムレータ（状態を保持するための引数）_

"reduce" は非常によく文書化されたパターンであり、車輪を再発明する必要はありません。[wiki を読みましょう。](https://en.wikipedia.org/wiki/Fold_(higher-order_function))必要な他の引数を求める記述が見つかるはずです。

> 実際には、初期値を持つことは便利で自然なことです。

### `Reduce` の最初の一歩

```go
func Reduce[A any](collection []A, accumulator func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = accumulator(result, x)
	}
	return result
}
```

Reduce はパターンの _本質_ をキャプチャします。これは、コレクション、結合関数、初期値を取り、単一の値を返す関数です。型のまわりは特にごちゃごちゃしていません。

ジェネリクス構文を理解していれば、この関数の機能を理解するのに問題はないはずです。`Reduce` という認識された用語を使用することで、他の言語のプログラマーもその意図を理解できます。

### 使い方

```go
// Sum calculates the total from a slice of numbers.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails calculates the sums of all but the first number given a collection of slices.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

`Sum` と `SumAllTails` は、それぞれ最初の行で宣言された関数として計算の動作を記述するようになりました。コレクションで計算を実行する行為は、`Reduce` で抽象化されています。

## reduce のさらなる応用

テストを使用して、reduce 関数をいじって、再利用可能性を確認します。前の章から一般的なアサーション関数をコピーしました。

```go
func TestReduce(t *testing.T) {
	t.Run("multiplication of all elements", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concatenate strings", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### ゼロ値

乗算の例では、`Reduce` の引数としてデフォルト値を使用する理由を示しています。Go の `int` のデフォルト値 0 に依存していた場合、初期値に 0 を掛けてから次の値を掛けるので、0 しか得られません。1 に設定すると、スライスは同じままで、残りは次の要素で乗算されます。

オタクの友達に賢いと思わせたい場合は、これを[単位元](https://en.wikipedia.org/wiki/Identity_element)と呼びます。

> 数学では、集合に作用する二項演算の単位元もしくは中立元は、演算が適用されたときに集合のすべての要素が変更されずに残る集合の要素です。

さらに、単位元は 0 です。

`1 + 0 = 1`

掛け算で 1 です。

`1 * 1 = 1`

## `A` とは違う型で reduce をしたい場合は？

取引 `Transaction` のリストがあり、それらを取得する関数と、銀行の残高を把握するための名前が必要だとします。

TDD プロセスに従ってみましょう。

## 最初にテストを書く

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## テストを実行してみます

```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## テストを実行するための最小限のコードを記述し、失敗したテスト出力を確認します

型や関数はまだないので、それらを追加してテストを実行します。

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

テストを実行すると、次のように表示されます。

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## 成功させるのに十分なコードを書く

最初に `Reduce` 関数がないかのようにコードを書きましょう。

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## リファクタリング♪

この時点で、コード管理にある程度の規律を設けて、作業をコミットしてください。Monzo や Barclays（銀行名の例）などに挑戦する準備ができている動作中のソフトウェアがあります。

これで私たちの作業はコミットされ、自由にそれをいじって、リファクタリング段階でいくつかの異なるアイデアを試すことができます。公平を期すために、私たちのコードは完全に悪いわけではありませんが、この演習のために、`Reduce` を使用して同じコードを示したいと思います。

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

しかし、これはコンパイルされません。

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

その理由は、コレクションの型とは _異なる_ 型に reduce しようとしているからです。これは恐ろしく聞こえますが、実際には `Reduce` の型を調整するだけで機能します。関数本体を変更する必要はなく、既存の呼び出し元を変更する必要もありません。

```go
func Reduce[A, B any](collection []A, accumulator func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = accumulator(result, x)
	}
	return result
}
```

`Reduce` の制約を緩めることを可能にする2つ目の型制約を追加しました。これにより、人々は `A` のコレクションから `B` への `Reduce` を行うことができます。今回は `Transaction` から `float64` へ行います。

これにより、`Reduce` がより汎用的で再利用可能になり、型安全になります。テストを再度実行すると、コンパイルされて、成功するはずです。

## 銀行を拡張する

楽しむために、銀行のコードを改善したいと考えました。簡単にするために、TDD プロセスは省略しました。

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

以下は更新したコードです。

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

これはまさに `Reduce` のような概念を使用することの威力を示していると思います。`NewBalanceFor` は、_方法 (how)_ よりも _何が (what)_ 起こるかを記述することで、より _宣言的_ な感じがします。多くの場合、コードを読んでいるとき、私たちは多くのファイルを急いで調べており、_方法 (how)_ ではなく _何が (what)_ 起こるかを理解しようとしていますが、このスタイルのコードはそれを簡単にします。

詳細を掘り下げたい場合は、ループや状態の変更を気にせずに `applyTransaction`の _ビジネスロジック_ を確認できます。`Reduce` はそれを個別に処理します。

### fold と reduce はとても普遍的です

`Reduce`（もしくは `Fold`）の可能性は無限大です。これは、算術演算や文字列連結のためだけではなく、一般的なパターンです。他のいくつかのアプリケーションを試してください。

- `color.RGBA` を単一の色に混ぜてみませんか？
- アンケートの投票数、もしくはショッピングバスケットのアイテムを合計します。
- 多かれ少なかれ、リストの処理に関係する何か。

## 見つける

Go にはジェネリクスがあり、それらを高階関数と組み合わせたので、プロジェクト内の多くのボイラープレートコードを削減して、システムの理解と管理を簡単にすることができます。

検索したいコレクションの種類ごとに特定の `Find` 関数を書く必要はなくなりました。代わりに、`Find` 関数を再利用もしくは定義します。上記の `Reduce` 関数を理解していれば、`Find` 関数を書くのは簡単です。

以下がテストです。

```go
func TestFind(t *testing.T) {
	t.Run("find first even number", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

そして、以下が実装です。

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

繰り返しますが、ジェネリクス型を取るため、さまざまな方法で再利用できます。

```go
type Person struct {
	Name string
}

t.Run("Find the best programmer", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

ご覧のとおり、このコードは完璧です。

## まとめ

これらのような高階関数を適切に行うと、コードが読みやすく保守しやすくなりますが、次の経験則を覚えておいてください。

TDD プロセスを使用して、実際に必要な実際の特定の動作を押し出します。その後、リファクタリングの段階で、コードを整理するのに役立ついくつかの有用な抽象化を発見できるかもしれません。

TDD と適切なコード管理の習慣を組み合わせて練習します。リファクタリングを試みる _前に_、テストに合格したときに作業をコミットします。これにより、混乱した場合でも、作業状態に簡単に戻すことができます。

### 名前の問題

Go の外部で調査を行う努力をしてください。そうすれば、既に確立された名前で既に存在するパターンを再発明することはありません。

関数を書くと、`A` のコレクションを受け取り、それらを `B` に変換しますか？それを `Convert` と呼ばないでください。それは [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function)) です。これらのアイテムに "適切な" 名前を使用すると、他のユーザーの認知的負担が軽減され、検索エンジンでより詳しく知ることができます。

### これはイディオムではないですか？

心を開いてみてください。

Go のイディオムは、ジェネリクスがリリースされたために _根本的に_ 変更されることはありませんし、変更されるべきではありませんが、言語の変更により、イディオムは変更される _可能性_ はあります！これは論争の的となる点であってはなりません。

以下のように言えます。

> イディオムではありません。

これ以上の詳細がなければ、実用的でも有益でもありません。特に新しい言語機能について話し合うときには。

独断に基づくのではなくメリットに基づいて、コードのパターンとスタイルについて同僚と話し合ってください。適切に設計されたテストがある限り、自分とチームにとって何がうまく機能するかを理解していれば、いつでもリファクタリングと変更を行うことができます。

### リソース

fold は、コンピューターサイエンスの真の基礎です。さらに詳しく知りたい場合は、ここにいくつかの興味深いリソースがあります。

- [Wikipedia: Fold](https://en.wikipedia.org/wiki/Fold)
- [A tutorial on the universality and expressiveness of fold](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
