---
title: "オブジェクト指向で書くテスタブルなソースコード"
emoji: "🖋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["オブジェクト指向", "テスト", "単体テスト", "typescript", "nodejs"]
published: true
---
## 概要

テストしにくい関数の組み合わせで作られているプログラムから、テストのしやすいオブジェクト指向で書かれたプログラムにリファクタリングを行う中で、クラスを用いてプログラムを書くとどのような嬉しいことがあるのかについて説明させていただければと思います

今回使用したソースコードなどは以下に置いてあります。ご自分で実行しながらやれれても面白いかもしれないです。
https://github.com/3l4l5/zenn-testable-source

### 想定読者

* オブジェクト指向について、聞いたことはあるけどイマイチピンとこない
* ソースコードに単体テストを入れようとしてるけど、具体的にどのように入れればいいのかわからない
* いまいちクラスを用いてプログラムを書くことの利点がわからない
* クラスがどのようなもの何かについては理解している

### 前提としている知識

* 単体テストの基本的な考え方
* TypeScript（ご自身の得意とされている言語にも同じような機能はあるはずですので適宜読み替えていただければと思います）

## 本題：とある学習塾の成績管理システムの例

※本記事で扱う対象やソースコードは全て架空のものです。

現在、とある塾の内部システムを作成しているとします。その中で、生徒のテストの点数の記録機能を作成することになりました。
実際の機能としては、単純に成績を登録するだけではなく、以下の四点の仕様があります。

* 既に登録されているテストが存在しなかった場合には、入力された試験の点数を登録する
* 生徒が前回受けた試験の点数よりも高かった場合のみ、情報が更新される
* 登録されている試験の日付が30日以上前の場合には、上の条件に当てはまらない場合でも情報を更新する
* 最終的にDBに保存されている生徒の成績を返す

上記の仕様を満たすようなソースコードを実装すると、たとえば次のようになります。（言語はTypeScriptです）

### まず普通に書いてみる


```TypeScript :updateUserOld.ts
type TestResult = {
    studentId: string,
    point: number,
    testDate: Date
}

const updateUserPoint = (studentData: TestResult): TestResult => {
    // DBのコネクションを生成
    const connection = getConnection()
    const oldStudentData = getStudentData(studentData.studentId, connection)
    if (!oldStudentData) {
        recordStudentData(studentData, connection)
        return studentData
    } else {
        const isPointUp = studentData.point >= oldStudentData.point
        const isMonthPassed =  Math.floor((studentData.testDate.getTime() - oldStudentData.testDate.getTime()) / 8640000) > 30;
        if (isMonthPassed) {
            updateStudentData(studentData, connection)
            return studentData
        } else if (isPointUp) {
            updateStudentData(studentData, connection)
            return studentData
        } else {
            // 登録しない
            return oldStudentData
        }
    }
}
```

DBのコネクションを取得したのちに、仕様に則ってテストの結果情報を登録したりしなかったりするスクリプトです。三つの条件分岐がありますが、特段難しいことはしていません。

```getStudentData```や```updateStudentData```、```recordStudentData```の詳細な実装は書いていませんが、idで指定された生徒情報を取得したり、生徒のStudent型のものを受け取り、更新したり、登録したりする関数だと思っていただければと思います。

このように担当する責任を関数で分けることによって、可読性やメンテナンス性を上げるプログラムの書き方を、構造化プログラミングと言います。やりたいことがコードを読むだけで大体理解できるので読みやすいですね！

しかし、おおむね問題ないコードのように思いますが、このコードは問題があります。

### 構造化プログラミングの問題点

その問題とはズバリ、「単体テストがしにくい」ということです。

このスクリプトをテストしようとするには以下の二点の方法が一般的に取られると思います

1. 実際にデータを準備して、テスト実行時にデータをDBに注入＋テスト終了後に後片付けを行うようにテストスクリプトを作成する
2. DBからデータを取得している部分をMockする

1の方法は特に難しいことはせず、テストが実行される際にあらかじめテストで使用するためのデータを入れておくようにするものです。これで解決しそうな気がしますが、テスト実行時に実際にDBへの接続が必要となるのは以下の三つの理由で得策とは言えません。

* テスト実行時に行わなければいけない動作が多い、また、テスト実行に時間がかかり、テストを行うハードルが上がる
* テスト実行時にDBに接続する必要があるため、DBに接続することができる環境からしかテストを実行することができない
* DBの状態によってテストの結果が変わってしまって、テストの結果が補償されない可能性がある

テストは手軽に結果を確認することができる手段です。そのテストにDBの接続などという技術的な事情が絡んでくるのはテストのハードルを上げ、最終的にはテストの信用を落とします。その先に待っているのはテストが全く書かれていない触ることのできない悲しきソースコードです。

余談ですが、ここに、アジャイルソフトウエア宣言で有名なMartin Fowler氏の言葉を書いておきます
> Whenever you are tempted to type something into a print statement or a debugger expression, write it as a test instead.
>何かを print 文やデバッガの式に書きたくなったときは、 代わりにその内容をテストに書くようにするんだ。
>
> -- Martin Fowler

さすがにテストコマンドを実行するごとにDBに接続していたら、プリントデバッグの感覚でテスト実行はできませんね。。

2については、実際にDBとやり取りしている関数をMockと呼ばれる仮の関数に置き換え実行させることで、その関数が呼ばれた時の動作を入れ替える方法です。これを用いることで、DBに接続しなくても、テストが行えるようになります。しかしながら、元のソースコードの関数を別のMock関数に置き換えることはできなくはないのですが、jestなどのテストツールを使う場合でも少々難易度が上がります（私もよく躓きます涙）

そこで、テストを簡単に実施するため（だけではないですが）、データベースに登録する部分をクラスを用いて実装してみましょう。実際にクラスを用いて書くことでどのようにテストが書きやすくなるのでしょうか。

### クラスでの実装例

```TypeScript :updateUser.ts
export interface IStudentDao {
  get: (id: string) => TestResult;
  update: (result: TestResult) => boolean;
  record: (result: TestResult) => boolean;
}

export type TestResult = {
  studentId: string;
  point: number;
  testDate: Date;
};

export const updateUserPoint = (
  testResult: TestResult,
  studentDao: IStudentDao
) => {
  const oldTestResult = studentDao.get(testResult.studentId);
  if (!oldTestResult) {
    studentDao.record(testResult);
    return testResult;
  } else {
    const isPointUp = testResult.point >= oldTestResult.point;
    const isMonthPassed =
      Math.floor(
        (testResult.testDate.getTime() - oldTestResult.testDate.getTime()) /
          8640000
      ) > 30;
    if (isMonthPassed) {
      studentDao.update(testResult);
      return testResult;
    } else if (isPointUp) {
      studentDao.update(testResult);
      return testResult;
    } else {
      // 登録しない
      return oldTestResult;
    }
  }
};

```

DB接続部分クラスで実装するとは言ったものの、実際はInterfaceしか実装してません。
Interfaceとは、クラスがどのようなメソッド（関数）や、フィールド（変数）を持つかを定義したものです。後続の処理では、このInterfaceを前提として処理を書くことができる他、実際に作成するクラスではこのInterfaceに会う形でクラスの中身を実装することで、実際のクラスとそのクラスを利用する処理の間で合意を取ることができます。
このInterfaceは、C#やPHPなどのさまざまな言語でサポートされています。また、最近のPython（2023年現在）でも抽象基底クラスをinterfaceのように使用することで似たような書き方をすることもできます。

updateUserPointは以前と変わり、関数の引数としてIStudentDaoというインターフェースを持つクラスを受け取るように変更されました。そして、処理の中では受け取ったクラスのメソッドを利用する形でDBへの問い合わせなどは実装されています。このようにInterfaceをあらかじめ定義しておくと、DBに対する処理の実装の詳細を全く気にすることなく、そのクラスの使い方のみを気にして処理を作ることができます。

さきほどの関数ベースの処理とは異なり、処理の実態を利用しているわけではないので、実際にDBと接続し、SELECTを実行する処理する部分などが存在しなくても、クラスを利用する処理を作成することができます。（さきほどの実装だと```getStudentData```などが存在しないとエディタに怒られますが、今回のケースだとInterfaceで補償しているので怒られません）

ではこの形にすることで、どうしてテストが書きやすくなるのでしょうか。
ここでキーポイントとなるのが、IStudentDaoのインターフェースです。updateUserPointが引数であるstudentDaoに要求していることは、「studentDaoはIStudentDaoのインターフェースに従ったオブジェクトであってほしい」ということのみです。極端に言えば、get, record, updateというメソッドを持っているかつ、それぞれの引数と返り値がinterfaceで定義した型に適合しているならどんなものでも入れていいということです。

次に実際のテストコードを書いてみます

### テストコード書くとこんな感じ

この、「引数として受け取るstudentDaoは、get, record, updateというメソッドを持っているオブジェクトならなんでも入れていい」という性質を利用すると、テストコードは以下のように書くことができます

```TypeScript :updateUser.test.ts
import { updateUserPoint, TestResult } from "../app/updateUser";

describe("試験登録のテスト", () => {
  // Mockオブジェクトを定義 interfaceを満たしていればいいので、以下のようにobjectで定義すれば良い
  const studentDaoMock = {
    get: jest.fn(),
    update: jest.fn(),
    record: jest.fn(),
  };
  test("30日以内の試験結果の登録テスト", () => {
    const user: TestResult = {
      studentId: "testId0001",
      point: 20,
      testDate: new Date("2023-09-01"),
    };
    // mockが返す情報を定義
    // 関数内でgetが呼ばれた時に返す値を設定してあげる
    const oldUser = {
      studentId: "testId0001",
      point: 30,
      testDate: new Date("2023-08-05"),
    };
    studentDaoMock.get.mockReturnValue(oldUser);

    // updateUserPointを呼ぶ
    const result = updateUserPoint(user, studentDaoMock);

    // テスト結果の確認
    // テストが行われたのが30日以内　かつ　テストの点数が下がっているので、更新されないことを確認
    // updateはされない
    expect(studentDaoMock.update).not.toHaveBeenCalled();
    // 帰ってくるUserは過去のuserが帰ってくる
    expect(result).toBe(oldUser);
  });
});
```

どうでしょうか。DBの処理部分をmockする（代わりのものを用意する）のがとても簡単だと思います。
Mockの関数を用意するためにやったこととしては

1. 同じInterfaceを持つオブジェクトを用意する
2. mock関数に具体的な返り値を設定
3. 実際の関数を呼んで
4. 帰ってきた値や、mock関数が呼ばれた回数などを確認

のみです。とてもシンプルかつ、テストしたいロジックに集中してテストを書くことができたと思います。
もし、最初のコードのように、updateUserPoint関数の中で利用されている関数を別のものに置き換えるなどをしようとした場合は、もう少し手順や設定が面倒なものになります。

では、実際にテストコードを実行してみましょう

```bash
$ npm run test updateUser.test.ts

>>>
 FAIL  src/test/updateUser.test.ts
  試験登録のテスト
    ✕ 30日以内の試験結果の登録テスト (3 ms)

  ● 試験登録のテスト › 30日以内の試験結果の登録テスト

    expect(jest.fn()).not.toHaveBeenCalled()

    Expected number of calls: 0
    Received number of calls: 1

    1: {"point": 20, "studentId": "testId0001", "testDate": 2023-09-01T00:00:00.000Z}

      29 |         // テストが行われたのが30日以内　かつ　テストの点数が下がっているので、更新されないことを確認
      30 |         // updateはされない
    > 31 |         expect(studentDaoMock.update).not.toHaveBeenCalled()
         |                                           ^
      32 |         // 帰ってくるUserは過去のuserが帰ってくる
      33 |         expect(result).toBe(oldUser)
      34 |     })

      at Object.<anonymous> (src/test/updateUser.test.ts:31:43)

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
Snapshots:   0 total
Time:        1.039 s, estimated 2 s

```

あれ、エラーになってしまいました。テストが通っていませんね。
どうやら、今回のテストケースの場合、30日以内で、かつ点数が低いので、updateは呼ばれないはずなのですが、呼ばれていますね。。。
実際にソースコードを見てみると、30日以内の計算式が間違っているみたいです！修正しましょう

```diff TypeScript :updateUser.ts
export interface IStudentDao {
  get: (id: string) => TestResult;
  update: (result: TestResult) => boolean;
  record: (result: TestResult) => boolean;
}

export type TestResult = {
  studentId: string;
  point: number;
  testDate: Date;
};

export const updateUserPoint = (
  testResult: TestResult,
  studentDao: IStudentDao
) => {
  const oldTestResult = studentDao.get(testResult.studentId);
  if (!oldTestResult) {
    studentDao.record(testResult);
    return testResult;
  } else {
    const isPointUp = testResult.point >= oldTestResult.point;
    const isMonthPassed =
      Math.floor(
        (testResult.testDate.getTime() - oldTestResult.testDate.getTime()) /
-         8640000
+         86400000
      ) > 30;
    if (isMonthPassed) {
      studentDao.update(testResult);
      return testResult;
    } else if (isPointUp) {
      studentDao.update(testResult);
      return testResult;
    } else {
      // 登録しない
      return oldTestResult;
    }
  }
};

```

もう一度テストを実行してみます

```bash
$ npm run test updateUser.test.ts

>>>
 PASS  src/test/updateUser.test.ts
  試験登録のテスト
    ✓ 30日以内の試験結果の登録テスト (2 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        1.13 s, estimated 2 s
Ran all test suites.

```

いいですね！テストがあるおかげで、メインロジックとなる部分の間違いに気づくことができました！
このテストコード以外にも、この処理にはさまざまな仕様がありました。このコードが記事の最初に出てきた仕様をクリアしているか、テストを書いて確かめてみるといいと思います。

## 結局オブジェクト指向って？

今回は、関数で書かれた構造化プログラムをクラスやインターフェースを用いたオブジェクト指向で書き直すということをしてみました。ではオブジェクト指向とは結局なんだったのでしょうか。
私は、オブジェクト指向とは「インターフェースなどを用いて依存の方向をコントロールしながら、クラスなどを実際のモノのように考えてプログラムを組み立てるという考え方」だと今は思っております。

### 依存って？

依存とは、片方のモノ（関数やクラスなどのプログラム）が、もう片方のモノなしでは存在できない状態を言います
例えば以下のケースは関数Aは関数Bに依存しています

```typeScript
const A(){
    let value = 10;
    const result = B();
    return value * result
}
```

上のような状態だと、Bという関数が存在しないと、Aは実行できません。なので処理をクラス化し、Interfaceを用意した上で、InterfaceにAの処理を依存させることで、直接の依存を避けることができます。このように、オブジェクト指向でプログラムを作成すると、さまざまな観点で依存して欲しくない部分への依存を避けることができ、結果的にメンテナンス性の高いコードにしていくことができます。

### モノのように？

また、「モノのように」という部分については、実際の身の回りにあるもので考えてみるとわかりやすいかと思います。
例えば、iPhoneをケーブルを用いて充電したいと思った時に、iPhoneが気にしなければならないことは、Lightningポートの"interface"に適合しているケーブルが刺さり、供給される電荷を電池にためることです(実際にはもう少し複雑な仕様だと思いますが)。
そのため、極論Lightningのinterfaceに適合していれば、それがテスト用の電圧計や電流計がくっついているケーブルだったとしても、iPhone側は何も気にしなくてもいいはずです。
これが、今回の例で言うと、iPhoneが```updateUserPoint```で、Lightningというinterfaceが```IStudentDao```、電流計や電圧計がくっついてるLightningのinterfaceを持つテスト用ケーブルが```studentDaoMock```みたいなイメージになります。

## 最後に

本記事では、学習塾の試験結果登録アプリケーションの例を通して、オブジェクト指向で実装すると、どのようにテストが書きやすくなるのかについて説明させていただきました。
しかし、本記事では主張したいことを「テストの書きやすさ」としたかったせいで、「テストを書きやすくするためにオブジェクト指向でかくと良い」というような話の流れになってしまっていますが、「オブジェクト指向でコードを書くとテストが書きやすい」というのはオブジェクト指向の良さのほんの一部です。
より詳しく、オブジェクト指向の美味しいところを知りたい場合には、田中ひさてる様の著書である「ちょうぜつソフトウエア設計入門」という書籍を読んでいただければと思います。実例が交えられ、オブジェクト指向で書くとどのような利点があるのかなどについてとてもわかりやすく解説されております。（私はその本を読むまでオブジェクト指向というものがあんまりピンときていませんでした）

https://amzn.asia/d/gcVMCRk

また、オブジェクト指向について理解し、依存の方向を自由自在にコントロールできるようになれば、DDDやクリーンアーキテクチャなどのプラクティスについても理解がしやすくなると思います。

だいぶ長々と書いてしまいましたが、最後まで読んでいただきありがとうございます。この記事がどなたかの一助になれれば幸いです。そして、皆さんも単体テストのないソースコードにどしどし単体テストを入れていきましょう！！
