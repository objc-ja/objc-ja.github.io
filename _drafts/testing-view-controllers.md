---
layout: post
title: "View Controllersのテスト"
category: issue-1
original:
  title: "Testing View Controllers"
  url: http://www.objc.io/issue-1/testing-view-controllers.html
  author: "Daniel Eggert"
translator:
  name: "@himara2"
  url: https://twitter.com/himara2
---
{% include JB/setup %}


テストを信仰するのはやめよう。テストは開発をスピーディに、そして楽しくしてくれるものだ。


## 物事をシンプルに保つ

簡単なものをテストすることは簡単で、複雑なものをテストすることは複雑だ。他の記事でも指摘したように、物事を小さくシンプルに保つことは一般に良いことだし、テストにとっても良いことであり、Win-Winといえる。TDD(test-driven development）を見てみると、それを好きな人・そうでない人の両方がいる。それについてここで深入りする気はないが、TDDは先にテストを書くものだとだけ言っておく。もし興味があれば、Wikipediaを見て欲しい。また、リファクタリングとテストを一緒に行うことはとても良いことだ、ということも言っておく。

UI部品のテストをすることは難しい。というのも、多くの動くパーツを内包していることが多いからだ。たいていの場合view ccontrollerはmodelやviewレイヤーの多くのクラスとインタラクティブである。テストをするためには独立して動くようにしておく必要がある。

view controllerを軽量に、テストをしやすくする方法について述べる。一般に、テストが難しいと思うとき、設計が壊れており、リファクタリングする必要があるということだ。[lighter view controller](http://www.objc.io/issue-1/lighter-view-controllers.html)にはヒントが多く含まれている。全体の設計が目指すのは関心の分離をなくすことだ。各クラスは1つのことだけを担当すべきで、そうすれば1つのことをテストすることができる。

テストを書けば書く程利益が減ることに注意しよう。何より、シンプルにテストを書くことだ。領域を小分けにして洗練していけば快適になる。


## モックを使う

小さなコンポーネントに分解できれば各クラスのテストを行えるようになる。テスト対象のクラスは他クラスとインタラクションがある場合が多い。こういった時、モックやスタブと呼ばれるものを使うと良い。モックはplaceholderだと考えれば良い。テスト対象のクラスは、実際のオブジェクトの代わりにplaceholderとインタラクティブに働く。こうすることで他の部分に左右されることなく、対象のテストに集中できる。

あるクラスの持つ配列をテストしたい場合を考える。あるタイミングでdata sourceはtableviewからcellをdequeする。テストの間tableviewを保持しておくことはできないが、モックのtableviewを使えばdata sourceをテストすることができる。最初は少し戸惑うかもしれないが、とても強力なので何度か使ううちに簡単に思うだろう。

Objective-Cの強力なモックツールに OCMock というものがある。OCMockはObjective-Cのランタイムを強力に、そして扱いやすくする人気のプロジェクトだ。モックを用いてテストを行うための様々な仕掛けがある。
The power tool for mocking in Objective-C is called OCMock. It’s a very mature project that leverages the power and flexibility of the Objective-C runtime. It pulls some cool tricks to make testing with mock objects fun.

data sourceのテストについては以下で説明する。


## SenTestKit

SenTestingKitというテストフレームワークを使っていく。SenTestingKitは1997からObjective-C開発者に知られており（1997とはiPhoneがリリースされる10年前）、現在ではXcodeに組み込まれている。

SenTestingKitはテストを実行するものだ。SenTestingKitを用いることでクラスをテストすることができる。テスト対象のクラス毎にテストを作成する。このクラスは名前の最後がTestsとなっていて、名前はテスト対象のクラスのものになる。

テストクラスの内部のメソッドが実際のテストを行う。メソッド名はtestで始まる必要がある。特別に -setUp と -tearDown というメソッドがある。テストクラスは単なるクラスであるので、テストを実装する際にはプロパティの追加やヘルパーメソッドの追加は自由に行える。

テストの良い方法は、テスト用にカスタマイズしたbaseクラスを作ることだ。そこに便利なロジックを作り、テストをより簡単に行えるようにする。便利なサンプル集の入ったプロジェクトを使うようにすると便利だ。テストのためにXcodeのテンプレートは使わない。単に.mファイルを追加するだけという、もっとシンプルで効果的な方法にする。規約にしたがって、テストクラスの語尾はTestsで終わらせる。名前はテスト対象のものを用いる。

## Xcodeによるインテグレーション

テストは自分が選択して追加したリソースに対して行われる。一部のファイルをテストしたい場合、それらをテストターゲットに追加すると、Xcodeはtest bundleに取り込む。NSBundleを使って配置することもできる。簡単にアクセスできるように `-URLForResource:withExtension:` を実装しておくと良いだろう。

Xcodeの各schemeには対応するtestがあるかどうかを定義する。Cmd+Rでアプリを起動し、Cmd+Uでテストを実行する。

テストを実行するとアプリが実際に起動し、test bundleが読み込まれる。そうして欲しくない場合もあるかもしれない。その場合、app delegateに以下のようなcodeを書こう：

```
static BOOL isRunningTests(void) __attribute__((const));

- (BOOL)application:(UIApplication *)application   
        didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    if (isRunningTests()) {
        return YES;
    }
    
    //
    // Normal logic goes here
    //
    
    return YES;
}

static BOOL isRunningTests(void)
{
    NSDictionary* environment = [[NSProcessInfo processInfo] environment];
    NSString* injectBundle = environment[@"XCInjectBundle"];
    return [[injectBundle pathExtension] isEqualToString:@"octest"];
}
```

XcodeのSchemeを変更しておけば大きなメリットを享受できるだろう。テストの前後にrun scriptを走らせることができるし、複数のtest bundleを保有することもできる。これは大きなプロジェクトで有効だ。最も重要なのは、複数のテストのON/OFFができること。これはテストをデバッグする際に便利である（その場合はすべてONにする）

また、コードとテストケースにブレイクポイントをセットし、テスト実行時にデバッガーをその箇所で止めることができる。


## DataSourceをテストする

view controllerを分割することでテストを簡単にしましょう。これからArrayDataSourceをテストします。まず、基本となるsetupをつくります。interface と implementation を同じファイルに追加します。（理由：@interfaceを他でincludeする必要がないのでとても良い）


```
#import "PhotoDataTestCase.h"

@interface ArrayDataSourceTest : PhotoDataTestCase
@end

@implementation ArrayDataSourceTest
- (void)testNothing;
{
    STAssertTrue(YES, @"");
}
@end
```

これは正式なものではなく、基本のテストのsetupだ。テストを走らせると-testNothingメソッドが実行され、STAssertマクロが作動する。ちなみにSTというのはSenTestingKitから来ている。このマクロがIssues navigatorに失敗を通知してくれる。


## 最初のテスト

サンプルで実装したtestNothingメソッドの代わりに実際のテストを書いていこう：

```
- (void)testInitializing;
{
    STAssertNil([[ArrayDataSource alloc] init], @"Should not be allowed.");
    TableViewCellConfigureBlock block = ^(UITableViewCell *a, id b){};
    id obj1 = [[ArrayDataSource alloc] initWithItems:@[]
                                      cellIdentifier:@"foo"
                                  configureCellBlock:block];
    STAssertNotNil(obj1, @"");
}
```

## Putting Mocking into Practice

次に、ArrayDataSourceにあるメソッドのtestを行う。

```
- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath;
```

これをtestするために次のテストメソッドを作成した。

```
- (void)testCellConfiguration;
```

最初にデータソースを作成する。

```
__block UITableViewCell *configuredCell = nil;
__block id configuredObject = nil;
TableViewCellConfigureBlock block = ^(UITableViewCell *a, id b){
    configuredCell = a;
    configuredObject = b;
};
ArrayDataSource *dataSource = [[ArrayDataSource alloc] initWithItems:@[@"a", @"b"] 
                                                      cellIdentifier:@"foo"
                                                  configureCellBlock:block];
```

configureCellBlockは渡されたオブジェクトを保持する以外には何もしない。こうしておくとテストが楽になる。

次に、tableviewのモックをつくる：

```
id mockTableView = [OCMockObject mockForClass:[UITableView class]];
```

data sourceは `-dequeueReusableCellWithIdentifier:forIndexPath:` をcallする。このメッセージを受け取った際にmockにふるまいをさせたい。最初にcellを作り、mockを準備するには、

```
UITableViewCell *cell = [[UITableViewCell alloc] init];
NSIndexPath* indexPath = [NSIndexPath indexPathForRow:0 inSection:0];
[[[mockTableView expect] andReturn:cell]
        dequeueReusableCellWithIdentifier:@"foo"
                             forIndexPath:indexPath];
```

こうしておく。
最初は少し戸惑うかもしれない。ここでは、mockは一部のコールのみを登録している。なので厳密にはmockはtableviewではない。-expect という特別なメソッドはmockに「何が発生した時に何を行えば良いか」を設定するものだ。

さらに、-expect はmockにコールが発生したことも伝えている。後でmockの-verifyメソッドを呼んだ時、メソッドがcallされてない状態ならテスト失敗となる。-stubも同じようにmockを生成できるが、こちらはメソッドがcallされたかどうかをケアしない。

次に、実行のためのトリガーとなるcodeを書く。テストをしたいメソッドを呼んでみよう：

```
NSIndexPath* indexPath = [NSIndexPath indexPathForRow:0 inSection:0];
id result = [dataSource tableView:mockTableView
            cellForRowAtIndexPath:indexPath];
```

そして、それがうまくいってることをテストしよう：

```
STAssertEquals(result, cell, @"Should return the dummy cell.");
STAssertEquals(configuredCell, cell, @"This should have been passed to the block.");
STAssertEqualObjects(configuredObject, @"a", @"This should have been passed to the block.");
[mockTableView verify];
```

STAssert マクロは値が等しいかどうかをテストする。1つ目と2つ目のテストで、-isEqualは使わない。なぜなら、result, cell, configuredCellのすべてが同じオブジェクトであることをテストしたいからだ。3つ目のテストでは-isEqualを使っている、そして最後に-verifyでmockをテストする。


```
id mockTableView = [self autoVerifiedMockForClass:[UITableView class]];
```

上記はコンビニエンスラッパーで、テストの最後に自動で-verifyをcallしてくれるものだ。


## UITableViewControllerのテスト

次に、PhotosViewControllerに注目してみよう。これはUITableViewControllerのサブクラスで、ここまでテストしていたdata sourceを使っている。view controllerに残っているコードは非常にシンプルである。

cellをタップするとPhotoViewControllerのインスタンスがpushされ、詳細のviewに遷移する、というのをテストしたい。他の部品との依存をよりシンプルにするため、ここでもmockを使おう。

はじめにUINavigationControllerのmockを作る

```
id mockNavController = [OCMockObject mockForClass:[UINavigationController class]];
```

次に、partial mockingを作る。PhotosViewControllerがそのnavigationControllerとしてmockNavControllerを返すようにしたい。直接navigation controllerをセットすることはできないので、スタプにmockNavControllerを返すだけのシンプルなメソッドを用意し、それをPhotosViewControllerのインスタンスにセットする：

```
 PhotosViewController *photosViewController = [[PhotosViewController alloc] init];
 id photosViewControllerMock = [OCMockObject partialMockForObject:photosViewController];
 [[[photosViewControllerMock stub] andReturn:mockNavController] navigationController];
```

photosViewControllerで-navigationControllerが呼ばれた際、mockNavControllerを返してくれる。これもOCMockによるとても強力な仕組みだ。

navigation controlerのmockには、例えばdetail view controllerにnilでない値がセットされているかを確認させる：
We now tell the navigation controller mock what we expect to be called, i.e. a detail view controller with photo set to a non-nil value:

```
UIViewController* viewController = [OCMArg checkWithBlock:^BOOL(id obj) {
    PhotoViewController *vc = obj;
    return ([vc isKindOfClass:[PhotoViewController class]] &&
            (vc.photo != nil));
}];
[[mockNavController expect] pushViewController:viewController animated:YES];
```

viewがロードされること、rowのタップをシミュレーションする：
Now we trigger the view to be loaded and simulate the row to be tapped:

```
UIView *view = photosViewController.view;
STAssertNotNil(view, @"");
NSIndexPath* indexPath = [NSIndexPath indexPathForRow:0 inSection:0];
[photosViewController tableView:photosViewController.tableView 
        didSelectRowAtIndexPath:indexPath];
```

最後にmockで期待していたメソッドが呼ばれたかを確認しよう:

```
[mockNavController verify];
[photosViewControllerMock verify];
```

ここまでで、navigation controllerとのインタラクション、正しいview controllerの作成のテストを実装した。

ここで再び自作のコンビニエンスメソッドを使って、

```
- (id)autoVerifiedMockForClass:(Class)aClass;
- (id)autoVerifiedPartialMockForObject:(id)object;
```

これを用いれば-verify忘れを防げる。


## さらなる可能性

ここまで見てきたように、partial mockingはきわめて強力だ。 `-[PhotosViewController setupTableView]` メソッドのソースをみたら、app delegateからどうやってmodel objectを取得しているのかが分かる：

```
 NSArray *photos = [AppDelegate sharedDelegate].store.sortedPhotos;
```

ここまでのテストはこの記述に依存している。この依存を解消する一つの方法は、app delegateにあらかじめ定義しておいた以下のようなデータを返すpartial mockingを使えば良い：

```
id storeMock; // assume we've set this up
id appDelegate = [AppDelegate sharedDelegate]
id appDelegateMock = [OCMockObject partialMockForObject:appDelegate];
[[[appDelegateMock stub] andReturn:storeMock] store];
```

`[AppDelegate sharedDelegate].store` がcallされたらstoreMockが返される。これはすごいことだ。テストをできる限りシンプルに、そして必要に応じて複雑にできる。


## 注意すべきこと

partial mockはそれが存在する限り対象のオブジェクトをmockに代替する。それを止めるためには、`[aMock stopMocking]`というメソッドを呼べば良い。大抵の場合、partial mockはテストの期間ずっとactiveにしたいはずだ。Make sure that happens by putting a `[aMock verify]` at the end of the test method. Otherwise ARC might dealloc the mock early. And you probably want that -verify anyway.
 

# NIBのロードをテストする

PhotoCellはNIBでセットアップされている。outletsが正しく読み込まれていることを確認するテストを書いてみよう。まずPhotoCellクラスを見ると、

```
 @interface PhotoCell : UITableViewCell
 
 + (UINib *)nib;
 
 @property (weak, nonatomic) IBOutlet UILabel* photoTitleLabel;
 @property (weak, nonatomic) IBOutlet UILabel* photoDateLabel;
 
 @end
```

テストの実装は以下のようになる。

```
@implementation PhotoCellTests

- (void)testNibLoading;
{
    UINib *nib = [PhotoCell nib];
    STAssertNotNil(nib, @"");
    
    NSArray *a = [nib instantiateWithOwner:nil options:@{}];
    STAssertEquals([a count], (NSUInteger) 1, @"");
    PhotoCell *cell = a[0];
    STAssertTrue([cell isMemberOfClass:[PhotoCell class]], @"");
    
    // Check that outlets are set up correctly:
    STAssertNotNil(cell.photoTitleLabel, @"");
    STAssertNotNil(cell.photoDateLabel, @"");
}

@end
```

とても簡単だが、これでOKだ。

変更が加わった時、テストとクラス/nibの両方を更新しなければならない、と思う人がいるかもしれない。その通りだ。We need to weigh this against the likelihood of breaking the outlets. xibファイルで作業を行えば、これはよく起こると気づくだろう。


## Side Note About Classes and Injection

XcodeによるIntegrationのセクションで、test bundleはappに追加されるといった。具体的にどのように働いているのかの詳細は語らなかった（それが巨大すぎるため）。test bundleからObjective-Cクラスを実行中のアプリに追加する。こうしてtestを走らせられるようになる。

よく迷うのは、appとtestのbundle両方にクラスを追加するかどうか、ということだ。我々は上記の例のように、PhotoCellをtestとappのbundle両方に追加した。そして、`[PhotoCell class]` のコールはtest bundleからの場合は異なるポインタを返す。そしてそれに従ってテストを行う

```
STAssertTrue([cell isMemberOfClass:[PhotoCell class]], @"");
```

これは失敗する。Injectionはとても複雑で、次のことに気をつけないといけない。アプリの.mファイルをtest targetに追加してはいけない。予期せぬふるまいに直面することになる。


## もう少し考えておくべきこと

もし継続的インテグレーションを行うなら、テストを書いてそれを動かすのは素晴らしいアイデアだ。詳細はこの章の範囲とは少々外れてしまうが。scriptはRunUnitTests scriptによって実行され、TEST_AFTER_BUILD 環境の変数となる。

もう一つ興味深いのは、自動化されたパフォーマンステストを独立にtest bundleに作れることだ。test methodの中には好きにテストを追加することができる。特定のコールのタイミングや、一定の範囲内であることをチェックするSTAssertを使う際にオプションをつける。


## Further Reading

Test-driven development  
OCMock  
Xcode Unit Testing Guide  
Book: Test Driven Development: By Example  
Blog: Quality Coding  
Blog: iOS Unit Testing  
Blog: Secure Mac Programing  