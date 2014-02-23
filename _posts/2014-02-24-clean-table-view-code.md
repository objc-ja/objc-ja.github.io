---
layout: post
title: "クリーンなTable Viewのコード"
description: ""
category: issue-1
tags: []
---
{% include JB/setup %}

原文：[Clean table view code](http://www.objc.io/issue-1/table-views.html) by Florian Kugler  
翻訳：佐藤新悟 ([@gonsee](http://twitter.com/gonsee))

table viewはiOSアプリにとって非常に用途の広い部品である。そのため多くのコードが直接的または間接的にtable viewに関連している。データの提供、table viewの更新、振る舞いの制御、選択時の反応などはその一部だ。本稿ではこのようなコードをクリーンでよく整理された形に保つためのテクニックを紹介する。

## UITableViewController vs. UIViewController

Appleはtable view専用のview controllerとして`UITableViewController`を提供している。table view controllerには、お決まりのコードを何度も書かずに済ませるための非常に便利な機能が多数実装されている。一方で、table view controllerは全画面表示のひとつのtable viewしか管理できない。しかしながら多くの場合これで事足りるし、もしそうじゃない場合も、以下で見ていくような対処法がある。

### Table View Controllerの機能

table view controllerはtable viewが初めて表示されるときにそのデータをロードする手助けをする。より具体的には、table viewのediting modeの切り替えや、キーボードからの通知に反応するといった仕事に加え、スクロールバーの点滅や選択の解除などいくつかの小さな仕事も担っている。これらの機能を使うためには、サブクラスでview event関連メソッド（`viewWillAppear:`や`viewDidAppear:`など）をオーバーライドする時、その中で必ずsuperを呼ぶ事が重要だ。

table view controllerは標準のview controllerに対してひとつユニークな利点がある。それはApple版pull to refreshのサポートだ。現時点で、`UIRefreshControl`の唯一ドキュメント化された使い方は、table view controllerの中で使う方法である。それ以外の方法もある事はあるが、そういった方法はiOSがアップデートされたときに動かなくなってしまう恐れがある。

これらの要素が集まって、Appleが定義した標準的table viewの振る舞いの多くを提供している。アプリがこれらの標準に従うのであれば、お決まりのコードを書くのを避けるためにできるだけtable view controllerを使うのが賢明だ。

### Table View Controllerの制限

table view controllerのviewプロパティは常にtable viewでなければならない。もし後からtable view以外のもの（例えば地図など）を同じ画面に表示したくなったら、不格好なハックに頼らざるを得ない。

インターフェースをコードまたは.xibファイルで定義しているなら、素のview controllerに移行するのはとても簡単だ。storyboardを使っている場合、移行には少しだけ手間がかかる。storyboardではtable view controllerを作り直すことなく素のview controllerに変更することができない。つまり中身をすべて新しいview controllerにコピーしてからすべてをつなぎ直す必要があるということだ。

さらに、移行によって失われたtable view controllerの機能を追加しなければならない。多くは`viewWillAppear`や`viewDidAppear`に１行追加する程度で済む。editing状態の切り替えにはtable viewのeditingプロパティを反転させるアクションメソッドを実装する必要がある。一番大きいのはキーボード関連の機能を再現するための作業だ。

こちらのルートを進む前に、ここで関心の分離（separation of concerns）も実現できる簡単な代替策を紹介する。

### 子View Controller

table view controllerを完全に捨ててしまうのではなく、table view controllerを別のview controllerの子view controllerとして追加するという方法もある（[view controller containmentについての記事](http://www.objc.io/issue-1/containment-view-controller.html
)を参照されたい）。こうすることで、table view controllerはこれまで通りtable viewだけを管理し、親view controllerがあらゆる追加要素の面倒を見るということが可能になる。

```objc
- (void)addPhotoDetailsTableView
{
    DetailsViewController *details = [[DetailsViewController alloc] init];
    details.photo = self.photo;
    details.delegate = self;
    [self addChildViewController:details];
    CGRect frame = self.view.bounds;
    frame.origin.y = 110;
    details.view.frame = frame;
    [self.view addSubview:details.view];    
    [details didMoveToParentViewController:self];
}
```

この解決策を採用する場合、子から親へのコミュニケーション経路を作る必要がある。例えば、ユーザーがtable viewのcellを選択したとき、親view controllerは別のview controllerをプッシュするためにこのことを知る必要がある。ユースケースによるが、多くの場合で一番きれいなやり方は、table view controllerのデリゲートプロトコルを定義し、親view controllerでそれを実装するという方法だ。

```objc
@protocol DetailsViewControllerDelegate
- (void)didSelectPhotoAttributeWithKey:(NSString *)key;
@end

@interface PhotoViewController () <DetailsViewControllerDelegate>
@end

@implementation PhotoViewController
// ...
- (void)didSelectPhotoAttributeWithKey:(NSString *)key
{
    DetailViewController *controller = [[DetailViewController alloc] init];
    controller.key = key;
    [self.navigationController pushViewController:controller animated:YES];
}
@end
```

ご覧のように、このやり方は関心の分離と再利用性という利点がある一方で、view controller間のコミュニケーションによるオーバーヘッドが発生する。個別のユースケースに応じて、よりシンプルにできるかもしれないし、必要以上に複雑になるかもしれない。これは開発者が考える必要がある。

## 関心の分離

table viewを扱う際には、モデル、ビュー、コントローラーの境界をまたいだ様々な仕事が存在する。view controllerがこれらすべてを担う場所になることを避けるために、できるだけ多くの仕事をより適切な場所へ分離することを試みる。これは可読性、メンテナンス性、テスト可能性を高めることにつながる。

ここで説明するテクニックは[Lighter View Controllers](http://www.objc.io/issue-1/lighter-view-controllers.html)の記事で紹介したコンセプトに則って発展させたものになる。どのようにdata sourceとモデルロジックを分離するかについてはこの記事を参照されたい。table viewの文脈においては、view controllerとviewの問題をいかに分離するかという点に注目していく。

### モデルオブジェクトとcell間の橋渡し

表示したいデータをビューレイヤーに渡さなければならないことがある。モデルとビューの明確な分離を維持したいので、これをtable viewのdata sourceで行うことがよくある。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView 
         cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView dequeueReusableCellWithIdentifier:@"PhotoCell"];
    Photo *photo = [self itemAtIndexPath:indexPath];
    cell.photoTitleLabel.text = photo.name;
    NSString* date = [self.dateFormatter stringFromDate:photo.creationDate];
    cell.photoDateLabel.text = date;
}
```

これだとdata sourceが特定のcellの設計に依存するコードでいっぱいになってしまう。こういったコードはcellクラスのカテゴリに出すのが良い。

```objc
@implementation PhotoCell (ConfigureForPhoto)

- (void)configureForPhoto:(Photo *)photo
{
    self.photoTitleLabel.text = photo.name;
    NSString* date = [self.dateFormatter stringFromDate:photo.creationDate];
    self.photoDateLabel.text = date;
}

@end
```

これによってdata sourceメソッドは非常にシンプルになる。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView
         cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView dequeueReusableCellWithIdentifier:PhotoCellIdentifier];
    [cell configureForPhoto:[self itemAtIndexPath:indexPath]];
    return cell;
}
```

サンプルコードでは、このtable viewのdata sourceは[独立したコントローラーオブジェクトに分離されていて](http://www.objc.io/issue-1/lighter-view-controllers.html#controllers)、cell設定用のブロックを渡して初期化するようになっている。この場合、ブロックは以下のようにシンプルになる。

```objc
TableViewCellConfigureBlock block = ^(PhotoCell *cell, Photo *photo) {
    [cell configureForPhoto:photo];
};
```

### Cellを再利用可能にする

同じタイプのcellが複数のモデルオブジェクトを表示できる場合、cellの再利用性を高めるためにもう一歩踏み込むことができる。まず、そのタイプのcellを使って表示するオブジェクトが従うべきプロトコルをcellに定義する。そしてcellのカテゴリにある設定メソッドを、このプロトコルに従うすべてのオブジェクトを受け入れるように書き換えればよい。この簡単な手順でcellを特定のモデルオブジェクトから切り離し、複数のデータタイプに適合させることができる。

### Cellの状態をCellの中で操作する

table viewの標準のハイライトや選択の挙動以上のことをやりたい場合、２つのデリゲートメソッドを実装して、タップされたcellを好きなように変更するという方法がある。例えば以下の通り。

```objc
- (void)tableView:(UITableView *)tableView
        didHighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.photoTitleLabel.shadowColor = [UIColor darkGrayColor];
    cell.photoTitleLabel.shadowOffset = CGSizeMake(3, 3);
}

- (void)tableView:(UITableView *)tableView
        didUnhighlightRowAtIndexPath:(NSIndexPath *)indexPath
{
    PhotoCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    cell.photoTitleLabel.shadowColor = nil;
}
```

しかし、これら２つのデリゲートメソッドもやはりcellがどのように実装されているかを知っている必要がある。もしcellを別のものと交換したり設計を変更したいときは、デリゲートメソッドの方も修正しなければならない。ビューの実装の詳細がデリゲートの実装に結びついてしまっている。そうなってしまわないように、このロジックをcell自身に移行するのが良いだろう。

```objc
@implementation PhotoCell
// ...
- (void)setHighlighted:(BOOL)highlighted animated:(BOOL)animated
{
    [super setHighlighted:highlighted animated:animated];
    if (highlighted) {
        self.photoTitleLabel.shadowColor = [UIColor darkGrayColor];
        self.photoTitleLabel.shadowOffset = CGSizeMake(3, 3);
    } else {
        self.photoTitleLabel.shadowColor = nil;
    }
}
@end
```

一般的に、我々はビューレイヤーの実装をコントローラーレイヤーの実装から分離するよう努力する。デリゲートはビューが異なる状態になりうることを知っている必要があるが、正しい状態にするためにビュー階層をどのように変更したらよいかや、サブビューのどの属性を変更したらよいかについては知るべきではない。そのようなロジックはすべてビューの中にカプセル化されているべきで、そうすることでシンプルなAPIを外部に提供できる。

### 複数のタイプのCellを扱う

ひとつのtable viewに複数の異なるタイプのcellがあると、data sourceメソッドはすぐに手に負えなくなる。サンプルアプリではphoto details tableに２種類のcellがある。一つはレーティングの星を表示するためのcellで、もう一つがキーと値のペアを表示するための汎用cellだ。これらの異なるタイプのcellを扱うコードを分離するために、deta sourceメソッドは単純に各cellに特化したメソッドにリクエストを投げている。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView  
         cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    NSString *key = self.keys[(NSUInteger) indexPath.row];
    id value = [self.photo valueForKey:key];
    UITableViewCell *cell;
    if ([key isEqual:PhotoRatingKey]) {
        cell = [self cellForRating:value indexPath:indexPath];
    } else {
        cell = [self detailCellForKey:key value:value];
    }
    return cell;
}

- (RatingCell *)cellForRating:(NSNumber *)rating
                    indexPath:(NSIndexPath *)indexPath
{
    // ...
}

- (UITableViewCell *)detailCellForKey:(NSString *)key
                                value:(id)value
{
    // ...
}
```

### Table Viewの編集

table viewは簡単に使える編集機能を備えており、cellの並び替えと削除ができる。これらのイベントが発生すると、table viewのdata sourceは[デリゲートメソッド](http://developer.apple.com/library/ios/#documentation/uikit/reference/UITableViewDataSource_Protocol/Reference/Reference.html#//apple_ref/occ/intfm/UITableViewDataSource/tableView:commitEditingStyle:forRowAtIndexPath:)によって通知を受ける。そのため、実際にデータの更新を実行するためのドメインロジックを、これらのデリゲートメソッドの中に書いていることが多い。

データの更新は明らかにモデルレイヤーの仕事だ。モデルは削除や並べ替えなどのAPIを公開すべきで、そうすればdata sourceメソッドから呼べるようになる。これによってコントローラーはビューとモデルの調整役に徹することができ、モデルレイヤーの実装の詳細を知らなくて済む。追加の効能として、モデルのロジックがビューコントローラーの他の仕事と分離できるため、テストもしやすくなる。

## まとめ

table view controllerは（もちろん他のコントローラーオブジェクトも！）主にモデルとビューの[調整と仲介の役割](http://developer.apple.com/library/mac/#documentation/General/Conceptual/DevPedia-CocoaCore/ControllerObject.html)を担うべきだ。明らかにビューやモデルのレイヤーに属する仕事をするべきではない。これを頭に入れておけば、デリゲートとdata sourceメソッドはずっと小さくなり、ほとんどが単純なお決まりのコードになるはずだ。

これはtable view controllerのサイズと複雑さを減らすだけでなく、ドメインロジックとビューロジックをより適切な場所に置くことにつながる。コントローラーレイヤーの上と下の実装の詳細がシンプルなAPIの内側にカプセル化され、最終的にはコードをずっと理解しやすく、また複数人で開発しやすくなる。

### 参考リンク
* [Blog: Skinnier Controllers using View Categories](http://www.sebastianrehnby.com/blog/2013/01/01/skinnier-controllers-using-view-categories/)
* [Table View Programming Guide](http://developer.apple.com/library/ios/#documentation/userexperience/conceptual/tableview_iphone/AboutTableViewsiPhone/AboutTableViewsiPhone.html)
* [Cocoa Core Competencies: Controller Object](http://developer.apple.com/library/mac/#documentation/General/Conceptual/DevPedia-CocoaCore/ControllerObject.html)
