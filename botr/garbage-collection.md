﻿ガベージコレクションの設計
=========================

(これは https://github.com/dotnet/coreclr/blob/master/Documentation/botr/garbage-collection.md の日本語訳です。対象rev.は 8d3936b ）

著者: Maoni Stephens ([@maoni0](https://github.com/maoni0)) - 2015

メモ：ガベージコレクションの話題について詳細に学習するには、このドキュメントの参考文献セクションにある『The Garbage Collection Handbook』を参照してください。

コンポーネント アーキテクチャ
======================

GCに属するコンポーネントは、アロケーター（allocator）とコレクター（collector）の2つです。アロケーターは追加のメモリの取得と、適切なタイミングでのコレクターの起動を担当します。コレクターはガベージ（ごみ）、言い換えると、プログラムによってもう利用されていないオブジェクトのメモリを回収します。

GC.Collectを手動で呼び出した場合や、メモリ不足の非同期通知をファイナライザースレッドが受け取った場合（コレクターが起動されます）など、コレクターが呼び出される場合は他にもあります。

アロケーターの設計
===================

アロケーターは実行エンジン（EE：Execution Engine）内のアロケーションヘルパーから、以下の情報を渡されつつ呼び出されます。

- 要求サイズ
- スレッドアロケーションコンテキスト
- ファイナライズ可能オブジェクトかどうか、といったようなことを示すフラグ群

GCはオブジェクトの種類に応じて特別な取扱いをするようなことはありません。GCはEEにオブジェクトのサイズを取得するよう依頼します。

サイズに基づき、GCはオブジェクトを2種類に分けます。すなわち、スモールオブジェクト（85,000バイト未満）とラージオブジェクト（85,000バイト以上）です。原則として、スモールオブジェクトとラージオブジェクトは同じ方法で処理されますが、ラージオブジェクトのコンパクションにはより多くのコストがかかるため、GCはこれらを区別します。

GCは、アロケーターにメモリを割り振るとき、アロケーションコンテキスト（allocation context）を考慮して動作します。アロケーションコンテキストのサイズはアロケーションクォンタム（allocation quantum）によって定義されます。

- **アロケーションコンテキスト** は、あるヒープセグメントよりも小さな領域で、それぞれ所定のスレッド専用のものとして使われます。シングルプロセッサ（つまり論理プロセッサ数が1）のマシンでは、単一のアロケーションコンテキストである、ジェネレーション0のアロケーションコンテキストが使用されます。
- **アロケーションクォンタム** は、アロケーションコンテキスト内でのオブジェクトの割り付け（アロケーション）を実行するために、アロケーターが追加のメモリを必要とするたびに割り付けるメモリのサイズです。割り付けは一般的には8k単位で、マネージドオブジェクトの平均サイズは約35バイトであるため、単一のアロケーションクォンタムを多数のオブジェクト割り付けに使用できるようになっています。

ラージオブジェクトはアロケーションコンテキストとアロケーションクォンタムを使用しません。単一のラージオブジェクトは、それ自身がこれらの比較的小さなメモリ領域よりも大きいかもしれません。さらに、これらの領域の（後述する）メリットは比較的小さなオブジェクトに固有のものです。ラージオブジェクトはヒープセグメントに直接割り付けられます。

アロケーターは以下を達成するように設計されています。

- **適切なタイミングでGCを起動する：** アロケーターは、アロケーションバジェット（コレクターによって設定されるしきい値）を超えた場合、またはアロケーターが所定のセグメント上でもう割り付けられなくなる場合にGCを起動します。アロケーションバジェットとマネージドセグメントについては後で詳しく説明します。
- **オブジェクトの局所性を保つ：** 同じヒープセグメントに一緒に割り付けられたオブジェクト群は、仮想アドレスが互いに近くなるように保存されます。
- **効率的なキャッシュの使用：** アロケーターは、オブジェクト単位ではなく、 _アロケーションクォンタム_ 単位でメモリを割り付けます。そのメモリにはすぐにオブジェクトが割り付けられるので、CPUキャッシュをウォームアップさせるため、アロケーターはそれらのメモリをゼロクリアします。アロケーションクォンタムは通常8kです。
- **効率的なロック処理：** アロケーションコンテキストとアロケーションクォンタムのスレッドアフィニティにより、あるアロケーションクォンタムに書き込むのは単一のスレッドのみであり続けることが保証されます。その結果、現在のアロケーションコンテキストが枯渇しない限り、オブジェクトの割り付けにロックは不要となります。
- **メモリ整合性：** GCはオブジェクト参照がランダムなメモリ位置を指し示さないように、新しく割り付けたオブジェクト用のメモリを常にゼロクリアします。
- **ヒープをクロール可能に保つ：** アロケーターは、フリーオブジェクトが各アロケーションクォンタム内の余剰メモリから作るようにします。たとえば、アロケーションクォンタムに30バイト残っていて、次に割り付けるオブジェクトが40バイトの場合、アロケーターはその30バイトをフリーオブジェクトにし、新しいアロケーションクォンタムを取得します。

割り付けAPI
---------------

     Object* GCHeap::Alloc(size_t size, DWORD flags);
     Object* GCHeap::Alloc(alloc_context* acontext, size_t size, DWORD flags);

上記の関数群はスモールオブジェクトとラージオブジェクトの両方の割り付けに使用可能です。LOHに直接割り付けるための関数もあります。

     Object* GCHeap::AllocLHeap(size_t size, DWORD flags);
 
コレクターの設計
=======================

GCの目標
---------------

GCはメモリをこの上なく効率的に管理しつつ、「マネージドコード」を記述する人々に必要とされる努力を極力小さくしようとします。効率的とは、以下を意味します。

- マネージドヒープが割り付け済みだが未使用のオブジェクト（つまりガベージ）を（比率または絶対量で）大量に含まないように、つまり余計なメモリを使わないようにするに足るくらいの頻度で発生すべきです。
- たとえ頻繁なGCによってメモリ使用量をより抑えられるとしても、他の有意義な作業に振り分けることのできたはずのCPU時間をできるだけ使わないよう、GCはめったに発生しないようにすべきです。
- GCは生産的であるべきです。GCがメモリを少ししか回収しなかったならば、GC（と関連するCPUサイクル）は無駄だったということになります。
- 各回のGCは高速であるべきです。多くのワークロードには低遅延であるという要件があります。
- マネージドコードの開発者は、（そのワークロードに対して相対的に）上手なメモリの使い方をするために、GCに詳しくなくてもかまわないようにすべきです。
- GCはさまざまなメモリの使い方のパターンにおいてうまく動作するよう自己チューニングすべきです。

マネージドヒープの論理表現
------------------------------------------

CLRのGCは世代別コレクター（generational collector）です。つまり、オブジェクトは複数の世代に論理的に分割されます。世代 _N_ が回収されるとき、生き残ったオブジェクトは世代 _N+1_ に属するようにマークされます。このプロセスを昇格（promotion）と言います。この動作の例外として、オブジェクトを降格（demote）または昇格させないことを決定する場合があります。

スモールオブジェクトの場合、マネージドヒープは、gen0、gen1、gen2の3世代に分割されます。ラージオブジェクトについては、gen3の1世代があります。gen0とgen1はエフェメラル（ephemeral、儚い）世代と言われます（オブジェクトは短い間だけ生存するためです）。

スモールオブジェクトヒープでは、世代番号は齢を表します。つまり、gen0は最も若い世代です。gen0にあるすべてのオブジェクトがgen1やgen2にある任意のオブジェクトより常に若いとは言っていません。後述する例外があります。ある世代をコレクションするということは、その世代にあるオブジェクトとそれよりも若い世代にあるすべてのオブジェクトのコレクションを行うという意味になります。

原則的には、ラージオブジェクトはスモールオブジェクトと同じように処理できますが、ラージオブジェクトのコンパクションには非常にコストがかかるので、別扱いされます。ラージオブジェクト用には世代が1つしかなく、パフォーマンス上の理由から常にgen2コレクションと一緒にコレクションが行われます。gen2とgen3は両方とも大きくなる可能性がありますし、エフェメラル世代（gen0とgen1）は有界のコストでGCが実行されなければなりません。

割り付けは最も若い世代で行われます。つまり、スモールオブジェクトでは、常にgen0で行われ、ラージオブジェクトでは、1つしか世代がないので、常にgen3で行われるということです。

マネージドヒープの物理表現
-------------------------------------------

マネージドヒープはマネージドヒープセグメントの集合です。ヒープセグメントは、GCによってOSから獲得されたメモリの連続したブロックです。ヒープセグメントはスモールオブジェクトセグメントとラージオブジェクトセグメントにパーティション化され、スモールオブジェクトとラージオブジェクトの区別が行われます。各ヒープにおいて、ヒープセグメントは互いに連鎖した構造になっています。CLRが読み込まれるときに、最低1つのスモールオブジェクトセグメントと、最低1つのラージオブジェクトセグメントが予約されます。

各スモールオブジェクトヒープにエフェメラルセグメントは常に1つしか存在せず、そこにgen0とgen1が存在します。エフェメラルセグメントはgen2オブジェクトを含むかもしれませんし、含まないかもしれません。エフェメラルセグメントに加えて、0以上の追加セグメントが存在します。これらはgen2オブジェクトのみを含むのでgen2セグメントとなります。

ラージオブジェクトヒープには1つ以上のセグメントがあります。

ヒープセグメントは低位のアドレスから高位のアドレスに向けて使用されます。つまり、セグメント上のより低位のアドレスのオブジェクトは、より高位のアドレスのオブジェクトよりも古いということです。繰り返しになりますが、後述する例外が存在します。

ヒープセグメントは必要に応じて獲得可能です。ライブオブジェクトを一切含まない場合、ヒープセグメントは削除されますが、マネージドヒープの初期セグメントは常に存在し続けます。各ヒープについて、スモールオブジェクトについてはそのGC中、ラージオブジェクトについてはその割り付け時に、一度に1つのセグメントが獲得されます。ラージオブジェクトはgen2コレクション（相対的にコストがかかります）によってのみ回収されるので、この設計によってパフォーマンスが向上しています。

ヒープセグメントは、それらが獲得された順番で、互いに連鎖しています。連鎖の中の最終セグメントは常にエフェメラルセグメントです。回収済みの（ライブオブジェクトのない）セグメントは削除されるのではなく再利用され、新しいエフェメラルセグメントになる可能性があります。セグメントの再利用はスモールオブジェクトヒープにのみ実装されています。ラージオブジェクトが割り付けられるたびに、ラージオブジェクトヒープ全体が考慮されます。スモールオブジェクトの割り付けではエフェメラルセグメントのみが考慮されます。
 
アロケーションバジェット
---------------------

アロケーションバジェットは各世代に関連付けられた論理的な概念です。アロケーションバジェットは、それを超えた時にその世代に対してGCが起動されるというサイズ制限です。

アロケーションバジェット（割り付け予算）は、概ねその世代の生存率に基づいた、その世代の資産セットです。生存率が高い場合、その世代に対するGCが次に行われたときのデッドオブジェクトとライブオブジェクトの比率の改善を期待して、アロケーションバジェットが拡大されます。

コレクションを実行する世代を決定する
---------------------------------------

GCが起動されたとき、GCは、まず、どの世代のコレクションを行うのかを決定しなければなりません。アロケーションバジェット以外にも、考慮しなければならない要因があります。

- ある世代の断片化状況。ある世代が大いに断片化しているならば、その世代のコレクションは生産的と考えられます。
- そのマシンにおけるメモリ負荷が高すぎる場合、空き領域ができる可能性があるならば GCがより積極的にコレクションを実施してもかまいません。これは、（マシン全体での）不要なページングを防ぐために重要です。
- エフェメラルセグメントの容量が足りない状態で動作している場合、新しいセグメントの獲得を避けるために、GCはエフェメラルコレクションをより積極的に行っても（つまり、より多くgen1コレクションを行っても）かまいません。

GC のフロー
----------------
 
マークフェーズ
----------

マーク（mark）フェーズの目標は、すべてのライブオブジェクトを見つけることです。

世代別コレクターのメリットは、常にすべてのオブジェクトを検索する必要なく、ヒープの一部だけをコレクションできることです。エフェメラル世代のコレクションを実行するとき、GCはそれらの世代でどのオブジェクトが生きているのかを見つけ出す必要があり、この情報はEEによって報告されます。EEによって生かされているオブジェクトに加えて、古い世代のオブジェクトも、若い世代のオブジェクトを参照することで、それらを生存させ続けることができます。

<!-- カードのsetは設定というには違和感があるので、ここではビットを立てるとしています。カードマーキングについては『ガベージコレクションのアルゴリズムと実装』p158などを参照。 -->
GCは古い世代のマーク処理にはカードを使用します。カードのビットは、割り当て（assignment）操作中にJITヘルパーによって立てられます。JITヘルパーはエフェメラルの範囲にあるオブジェクトを参照した場合、そのソース位置を表すカードを含むバイトのビットを立てます。エフェメラルコレクション中、GCはヒープの残りの領域についてはビットが立ったカードを調べ、それらのカードに対応するオブジェクトのみを調べるようにできます。

プランフェーズ
---------

<!-- スイープなどがカタカナ語なので、すべてカタカナに統一 -->
プラン（plan)フェーズは、効果的な結果を判定するためにコンパクションをシミュレートします。コンパクションが生産的ならば、GCは実際のコンパクションを開始します。そうでない場合、スイープを実行します。

リロケーションフェーズ
--------------

GCがコンパクションをすることにしたならば、オブジェクトの移動が起こるので、それらのオブジェクトに対する参照を更新しなければなりません。リロケーション（relocation）フェーズでは、コレクションが行われようとしている世代にあるオブジェクトを指し示す、すべての参照を検索する必要があります。一方、マークフェーズではライブオブジェクトのみを検査するので、弱参照を考慮する必要がありません。

コンパクションフェーズ
-------------

このフェーズは、プランフェーズでオブジェクトの移動先となるべき新しいアドレスが計算済みのため、非常に単純です。コンパクション（compact）フェーズは新しいアドレスにオブジェクトをコピーします。

スイープフェーズ
-----------

スイープ（sweep）フェーズはライブオブジェクトの間にあるデッドスペースを探します。そして、これらのデッドスペースの場所にフリーオブジェクトを作成します。隣接したデッドオブジェクト群は1つのフリーオブジェクトになります。これらのフリーオブジェクトはすべて _フリーリスト_ （ _freelist_ ）に入れられます。
 
コードのフロー
=========

用語：

- **WKS GC：** ワークステーション GC。
- **SRV GC：** サーバー GC。

機能的な振る舞い
-------------------

### WKS GC（同時実行GCが無効）

1. ユーザースレッドがアロケーションバジェット不足状態で動作し、GCを起動する。
1. GCは、マネージドスレッドをサスペンドするためにSuspendEEを呼び出す。
1. GCは対象とする世代を決定する。
1. マークフェーズを実行する。
1. プランフェーズを実行し、コンパクション付GCを実施すべきかどうかを決定する。
1. 実施すべきであれば、リロケーションフェーズとコンパクションフェーズを実行する。そうでないならば、スイープフェーズを実行する。
1. GCは、マネージドスレッドをリジュームするためにRestartEEを呼び出す。
1. ユーザースレッドが実行を再開する。

### WKS GC（同時実行GCが有効）

これは、バックグラウンドGCがどのように完了するのかを示しています。

1. ユーザースレッドがアロケーションバジェット不足状態で動作し、GCを起動する。
1. GCは、マネージドスレッドをサスペンドするためにSuspendEEを呼び出す。
1. GCはバックグラウンドGCが実行されるべきかどうかを決定する。
1. 実行されるべきであれば、バックグラウンドGCを行うために、バックグラウンドGCスレッドが起動（wake up）される。バックグラウンドGCスレッドは、マネージドスレッドをリジュームするためにRestartEEを呼び出す。
1. バックグラウンドGCの動作中、マネージドスレッドは割り付けをし続ける。
1. ユーザースレッドのアロケーションバジェットがなくなり、エフェメラルGC（我々がフォアグラウンドGCと呼んでいるもの）を起動する。これは、「WKS GC（同時実行GCが無効）」と同じように完了する。
1. バックグラウンドGCはマーク処理を完了するためにもう一度SuspendEEを呼び出し、その後、ユーザースレッドを実行しながらの同時実行スイープフェーズを開始するためにRestartEEを呼び出す。
1. バックグラウンドGCが完了する。

### SVR GC（同時実行GCが無効）

1. ユーザースレッドがアロケーションバジェット不足状態で動作し、GCを起動する。
1. 複数のサーバーGCスレッドが起動（wake up）され、マネージドスレッドをサスペンドするためにSuspendEEを呼び出す。
1. 複数のサーバーGCスレッドがGCの作業を実行する（同時実行GCが無効なワークステーションGCと同じフェーズ）。
1. 複数のサーバーGCスレッドは、マネージドスレッドをリジュームするためにRestartEEを呼び出す。
1. ユーザースレッドが実行を再開する。

### SVR GC（同時実行GCが有効）

このシナリオは、非バックグラウンドGCが複数のサーバーGCスレッドで行われることを除き、同時実行GCが有効なWKS GCと同じです。

物理的なアーキテクチャ
=====================

この節はコードフローを追うための補助のためのものです。

ユーザースレッドがクォンタムが不足した状態で動作し、try_allocate_more_space経由で新しいクォンタムを取得します。

try_allocate_more_spaceは、GCを起動する必要がある場合にGarbageCollectGenerationを呼び出します。

同時実行GCが無効なWKS GCの場合、GarbageCollectGenerationは、GCを起動したユーザースレッド上ですべて完了します。コードフローは次のとおりです。

     GarbageCollectGeneration()
     {
         SuspendEE();
         garbage_collect();
         RestartEE();
     }
     
     garbage_collect()
     {
         generation_to_condemn();
         gc1();
     }
     
     gc1()
     {
         mark_phase();
         plan_phase();
     }
     
     plan_phase()
     {
         // compactを行うかどうかを判定するための実際の計画フェーズの処理
         if (compact)
         {
             relocate_phase();
             compact_phase();
         }
         else
             make_free_lists();
     }

同時実行GCが有効なWKS GC（既定のケース）の場合、バックグラウンドGC用のコードフローは次のとおりです。

     GarbageCollectGeneration()
     {
         SuspendEE();
         garbage_collect();
         RestartEE();
     }
     
     garbage_collect()
     {
         generation_to_condemn();
         // バックグラウンドGCを行うかどうかを判定
         // バックグラウンドGCスレッドを起動するかどうかを判定
         do_background_gc();
     }
     
     do_background_gc()
     {
         init_background_gc();
         start_c_gc ();
     
         // BGCによって再開されるまで待機。
         wait_to_proceed();
     }
     
     bgc_thread_function()
     {
         while (1)
         {
             // イベントを待機
             // 起動
             gc1();
         }
     }
     
     gc1()
     {
         background_mark_phase();
         background_sweep();
     }

参考文献
=========

- [.NET CLRのGCの実装](https://raw.githubusercontent.com/dotnet/coreclr/master/src/gc/gc.cpp)
- [The Garbage Collection Handbook: The Art of Automatic Memory Management](http://www.amazon.com/Garbage-Collection-Handbook-Management-Algorithms/dp/1420082795)
- [Garbage collection (Wikipedia)](http://en.wikipedia.org/wiki/Garbage_collection_(computer_science))
