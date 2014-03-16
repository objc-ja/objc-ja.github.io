---
layout: post
title: "軽量なView Controller"
category: issue-1
original:
  title: "Lighter View Controllers"
  url: http://www.objc.io/issue-1/lighter-view-controllers.html
  author: "Chris Eidhof"
translator:
  name: "@gonsee"
  url: http://twitter.com/gonsee
---
{% include JB/setup %}

view controllerはiOSプロジェクトの中で一番大きいファイルになりがちで、必要以上に多くのコードを含んでいることが多い。ほぼ決まってView Controllerはコードの中で最も再利用性の低い部分だ。View Controllerをスリムにし、再利用可能にして、より適切な場所にコードを移すテクニックを見ていこう。

この記事の[サンプルプロジェクト](https://github.com/objcio/issue-1-lighter-view-controllers)がGitHubにあるので参照されたい。

## データソースとその他のプロトコルを外に出す

View Controllerスリム化の最も強力なテクニックのひとつが、`UITableViewDataSource`の部分を独立したクラスに移すことだ。これを2回以上やってみるとパターンが見えてきて、このための再利用可能なクラスを作ることになるだろう。

例えば我々のプロジェクトでは`PhotosViewController`というクラスがあり、次のようなメソッドを持つ。

```objc
# pragma mark Pragma 

- (Photo*)photoAtIndexPath:(NSIndexPath*)indexPath {
    return photos[(NSUInteger)indexPath.row];
}

- (NSInteger)tableView:(UITableView*)tableView 
 numberOfRowsInSection:(NSInteger)section {
    return photos.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView 
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    PhotoCell* cell = [tableView dequeueReusableCellWithIdentifier:PhotoCellIdentifier 
                                                      forIndexPath:indexPath];
    Photo* photo = [self photoAtIndexPath:indexPath];
    cell.label.text = photo.name;
    return cell;
}
```

このコードの多くの部分が配列に関するもので、一部がview controllerの管理するphotosに固有のコードだ。そこで配列に関するコードを独自クラスに移してみよう。ここではcellの設定にblockを使うが、ユースケースや好みに応じてdelegateにしても構わない。

```objc
@implementation ArrayDataSource

- (id)itemAtIndexPath:(NSIndexPath*)indexPath {
    return items[(NSUInteger)indexPath.row];
}

- (NSInteger)tableView:(UITableView*)tableView 
 numberOfRowsInSection:(NSInteger)section {
    return items.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView 
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    id cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifier
                                              forIndexPath:indexPath];
    id item = [self itemAtIndexPath:indexPath];
    configureCellBlock(cell,item);
    return cell;
}

@end
```

view controllerにあった３つのメソッドをなくし、代わりにこのオブジェクトのインスタンスを作ってtable viewのdata sourceとしてセットできる。

```objc
void (^configureCell)(PhotoCell*, Photo*) = ^(PhotoCell* cell, Photo* photo) {
   cell.label.text = photo.name;
};
photosArrayDataSource = [[ArrayDataSource alloc] initWithItems:photos
                                                cellIdentifier:PhotoCellIdentifier
                                            configureCellBlock:configureCell];
self.tableView.dataSource = photosArrayDataSource;
```

これでindex pathを配列のインデックスにマッピングすることを考える必要がなくなり、table viewに配列を表示したい時はいつでもこのコードを再利用できる。さらに `tableView:commitEditingStyle:forRowAtIndexPath:`のようなメソッドを追加で実装して、このコードをすべてのtable view controllerで共有することも可能だ。

この方法の良いところは、このクラスを単体でテストでき、もう一度テストを書く必要がなくなることだ。もし配列ではないものを扱う場合であっても同じ原則が当てはまる。

今年我々が取り組んだアプリのひとつで、Core Dataをヘビーに使ったものがある。我々は同様のクラスを作ったが、配列を使うのではなく、fetched result controllerを使った。更新のアニメーションやsection header関連の処理、削除といったロジックをそこに実装した。このクラスのインスタンスを作ってfetch requestと、cellの設定のためのblockを与えれば、あとはこのクラスが面倒を見てくれる。

さらに、このアプローチは他のプロトコルにも応用できる。明らかな候補は`UICollectionViewDataSource`だ。これによって非常に高い柔軟性が得られる。例えば開発中のある時点で、`UITableView`の代わりに`UICollectionView`を使うことになった場合、view controllerのコードはほとんど何も修正する必要がないだろう。data sourceに両方のプロトコルをサポートすることさえ、やろうと思えば可能だ。

## ドメインロジックはモデルに

次の例は（別のプロジェクトの）view controllerのもので、ユーザーのactive priorityのリストを得るためのコードだ。

```objc
- (void)loadPriorities {
  NSDate* now = [NSDate date];
  NSString* formatString = @"startDate <= %@ AND endDate >= %@";
  NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
  NSSet* priorities = [self.user.priorities filteredSetUsingPredicate:predicate];
  self.priorities = [priorities allObjects];
}
```

これはしかし`User`クラスのカテゴリへ移した方がずっとすっきりする。そうすると`ViewController.m`はこうなる。

```objc
- (void)loadPriorities {
  self.priorities = [user currentPriorities];
}
```

そして`User+Extensions.m`は以下の通りだ。

```objc
- (NSArray*)currentPriorities {
  NSDate* now = [NSDate date];
  NSString* formatString = @"startDate <= %@ AND endDate >= %@";
  NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
  return [[self.priorities filteredSetUsingPredicate:predicate] allObjects];
}
```

コードによってはモデルオブジェクトに簡単に移動できないが、明らかにモデルのコードと密接に関連しているものがある。そんなときはストアクラスを利用することができる。

## ストアクラスを作る

我々のサンプルアプリケーションの最初のバージョンにはファイルからデータを読み込んでパースするコードがあった。このコードは次のようにview controllerの中にあった。

```objc
- (void)readArchive {
    NSBundle* bundle = [NSBundle bundleForClass:[self class]];
    NSURL *archiveURL = [bundle URLForResource:@"photodata"
                                 withExtension:@"bin"];
    NSAssert(archiveURL != nil, @"Unable to find archive in bundle.");
    NSData *data = [NSData dataWithContentsOfURL:archiveURL
                                         options:0
                                           error:NULL];
    NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
    _users = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"users"];
    _photos = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"photos"];
    [unarchiver finishDecoding];
}
```

view controllerはこれについて知る必要がない。我々はこの作業だけを行う *ストア* オブジェクトを作った。分離することによってそのコードが再利用可能になり、個別にテストできるようになり、view controllerを小さく保つことができる。ストアはデータの読み込み、キャッシュ、データベーススタックの設定などを担当することができる。ストアはしばしば *サービスレイヤー* または *レポジトリ* とも呼ばれる。

## ウェブサービスロジックはモデルレイヤーに

これは上記のトピックとよく似ている。ウェブサービスロジックをview controllerで処理してはならない。代わりにこれを別クラスにカプセル化しよう。view controllerはこのクラスのメソッドをコールバックハンドラ（例えばcompletionブロック）を使って呼ぶことができる。この方法の利点はキャッシングやエラーハンドリングもこのクラスの中でできることだ。

## ビューのコードはビューレイヤーに

複雑なview階層の構築をview controllerの中で行うべきではない。Interface Builderを使うか独自の`UIView`サブクラス内にカプセル化しよう。例えば独自のdate picker controlを作る場合、全てをview controllerの中で構築するより`DatePickerView`の中に入れる方がよい。これもまた再利用性とシンプルさを高めるテクニックだ。

もしInterface Builderを好むなら、この作業はInterface Builderを使っても可能だ。view controllerにしか使えないと思われがちだが、カスタムビューを個別のnibファイルから読み込むこともできる。サンプルアプリではphoto cellのレイアウトを含む[`PhotoCell.xib`](https://github.com/objcio/issue-1-lighter-view-controllers/blob/master/PhotoData/PhotoCell.xib)を作っている。

![PhotoCell.xibスクリーンショット](http://www.objc.io/images/issue-1/photocell.png)

上図のように、viewのプロパティを作って対応するsubviewにひもづけている（このxibではFile's Ownerは使用しない）。このテクニックは他のカスタムビューを作る際にも非常に便利だ。

## コミュニケーション

その他にview controllerの中でよく起こることのひとつは、他のview controllerやモデル、ビューとのコミュニケーションだ。これこそview controllerがやるべきことなのだが、可能な限り少ないコードで済ませたい部分でもある。

view controllerとモデルオブジェクト間のコミュニケーションのためのテクニックにはよく知られた方法（KVOやfetched result controllerなど）があるが、view controller間のコミュニケーションに関してはそれほど明確ではないことが多い。

view controllerのひとつがある状態を持っていて、他の複数のview controllerとの間でやりとりをするような場合によく問題になる。この状態を別のオブジェクトに移して、そのオブジェクトを複数のview controller間で受け渡すようにするとうまく行くことが多い。情報がただ一カ所にあるため、入れ子になったdelegateコールバックに悩まされることがないという利点がある。これについては複雑なテーマなので、今後連載の１回分を費やすかもしれない。

## まとめ

view controllerをより小さくするためのテクニックをいくつか見てきた。我々はこれらのテクニックを使えるところにはすべて使おうという努力はしない。我々のゴールはただひとつ、メンテ可能なコードを書くことだ。これまで見てきたパターンを知ることは、巨大で手に負えないview controllerをよりクリーンにする契機になるだろう。

## 参考リンク

* [View Controller Programming Guide for iOS](http://developer.apple.com/library/ios/#featuredarticles/ViewControllerPGforiPhoneOS/BasicViewControllers/BasicViewControllers.html)
* [Cocoa Core Competencies: Controller Object](http://developer.apple.com/library/mac/#documentation/General/Conceptual/DevPedia-CocoaCore/ControllerObject.html)
* [Writing high quality view controllers](http://subjective-objective-c.blogspot.de/2011/08/writing-high-quality-view-controller.html)
* [Stack Overflow: Model View Controller Store](http://programmers.stackexchange.com/questions/184396/mvcs-model-view-controller-store)
* [Unburdened View Controllers](https://speakerdeck.com/trianglecocoa/unburdened-viewcontrollers-by-jay-thrash)
* [Stack Overflow: How to avoid big and clumsy UITableViewControllers on iOS](http://programmers.stackexchange.com/questions/177668/how-to-avoid-big-and-clumsy-uitableviewcontroller-on-ios)
