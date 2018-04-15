# 課題：モナド変換子ライブラリ

## はじめに

ここでは[IOHKのHaskell講義](https://iohk.io/blog/iohk-haskell-and-cryptocurrency-course-in-barbados/)で課題として出されたUTXOを利用したトランザクション処理の問題を僕なりにアレンジしたものを紹介したいと思います。

なお他のプログラミング言語でも実装を試みるのも非常に興味深いです。もし実装できた方は教えて頂きたいです！

## 課題に取り組む前に
実はこの課題を真面目に取り組むと実務にかなり近いレベルの実装が可能です。

- 記述するコード、実装は可能な限りクオリティの高いものを目指してください。
- Haddockコメントも積極的に記述してください。([こちら](https://github.com/aisamanra/haddock-cheatsheet/blob/master/haddocks.pdf)からチートシートを参照できます）
- 記述スタイルに関しては[こちら](https://github.com/tibbe/haskell-style-guide/blob/master/haskell-style.md)を参考にしてください。

私(`@Hiroto`) はhaskell-jpのSlackチャンネルにいますので、もし質問がある場合には気軽に問い合わせてください。以下にてSlackの招待リンクを貼ります。<br>
https://join-haskell-jp-slack.herokuapp.com/

**May the compiler be with you (コンパイラーの力が共にあらんことを)**

それでは始めましょう。

## トランザクション処理

ほとんどの暗号通貨ではトランザクションは複数のインプットとアウトプットから構成されます。つまりそのデータ構造をおおまかに表現すると以下のようなものとなります:

```haskell
data Transactions = Transactions
     { tId     :: Id
     , tInput  :: [Input]
     , tOutput :: [Output]
     } deriving Show

type Id = Int
```
`Id`はトランザクションを一意に特定する値です。

インプットは**全て消費**され、それによって利用可能となった合計金額がアウトプットに分配されます。これを送金(トランザクション)と呼びます。

アウトプットは送金額、及び受取人のアドレスの情報を保持しています。ここではアドレスは`String`、金額は`Int`として表現します。

```haskell
data Output = Output
     { oValue   :: Int
     , oAddress :: Address
     } deriving Show

type Address = String
```

興味深いのはインプットです。インプットは過去のアウトプットを一意に特定するために、そのトランザクションID及び、そのトランザクションのアウトプットリストの有効なインデックスを必要とします。

```haskell
data Input = Input
     { iPrevious :: Id
     , iIndex    :: Index
     } deriving (Show, Eq, Ord)

type Index = Int
```

トランザクションを処理する際には**未使用のアウトプット**, (unspend transaction outputs, 以下`UTXOs`)を参照する必要があります。これはインプットとして利用可能なアウトプットを`Map`で管理しているものとなります。

```haskell
type UTXos = Map Input Output
```

この`Map`に含まれているものは全て未使用の（すなわち支払いに利用可能な）アウトプットです。この`Map`から一意のアウトプットを参照するには、トランザクションID及びそのアウトプットのインデックスが必要となります。よってインプットが`Key`となります。

### トランザクションの検証

トランザクションが有効であるためには以下の条件を満たさなければなりません:
1. 全てのインプットが`UTXOs`にて参照可能であること
2. インプットの合計金額がアウトプットよりも多いこと

### トランザクション処理

トランザクションが有効である場合には以下の処理が行われます:
1. トランザクションで使用されたアウトプットをUTXOから取り除く
2. トランザクションのアウトプットを`UTXOs`に書き込む

例えば以下の有効なトランザクションがあったとします。

```haskell
transaction :: Transaction
transaction = Transaction 1 [(有効なInput)] [(Output 100 "Andres"), (Ouput 200 "Lars")] 
```
このトランザクションが処理された場合、`UTXOs`は以下のように更新されます。

```haskell
[.., (Input 1 0, Output 100 "Andres"), (Input 1 1, Output 200 "Lars")]
```

なお、実際の暗号通貨ではインプットとアウトプットの差額は手数料となり、マイナーに割り当てるためのトランザクションを作成する必要があるのですが、ここではそれを無視し、差額は消滅するものとします。

## 問題

### 問1

以上の条件に従ってトランザクションを処理する関数、`processTranscation`を実装しなさい。またトランザクション処理後には`UTXOs`の状態を返しなさい。

```haskell
processTransaction :: Transaction -> ...
```

なお、エラー処理は可能な限り行ってください。

### 問2

次に複数のトランザクションを処理する関数, `processTransactions`を実装しなさい。

```haskell
processTransactions :: [Transaction] -> ...
```

### 問3

小規模のトランザクション及び`UTXOs`を定義し、実装した関数を用いてトランザクション処理を行い、実装が意図した通りに動作していることを確認してください。

### ボーナス問題

暗号通貨ではインプットを全て消費しなければならないため、**おつりアウトプット**なるものが必要となります。

これは例えばあなたが暗号通貨1000Qiitaを持っていたとして200だけを使用した場合、残りの800を自身のアドレスへ送金するためのアウトプットと考えるとわかりやすいでしょう。


おつりアウトプットを実装しなさい。

## ヒント

モナド変換子の基本に関しては以前私が書いた[記事](https://qiita.com/HirotoShioi/items/8a6107434337b30ce457)が参考になると思います。


#### 1.

`UTXOs`の状態を`State`モナド、エラー処理を`Either`モナドで取り扱うのが良さそうです。つまり以下のモナドスタックが考えられます。

```haskell
newtype App a = App (StateT UTXOs (ExcepT String Identity) a)
  deriving (Functor, 
          , Applicative,
          , Monad,
          , MonadState UTXOs
          , MonadError String)
```
つまり`processTransaction`, `processTransactions`関数の型シグネチャは以下のものとなります。

```haskell
processTransaction :: Transaction -> App ()

processTransactions :: [Transaction] -> App ()
```


#### 2.

`processTransactions`は一行で実装可能です。