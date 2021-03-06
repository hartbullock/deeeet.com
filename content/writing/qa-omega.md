+++
date = "2015-09-17T18:47:11+09:00"
title = "Google Omegaとは何か? Kubernetesとの関連は? 論文著者とのQA（翻訳）"
+++

エンタープライズ向けのKubernetesサポートを行っている[kismatic Inc.]([https://kismatic.com/)による["Omega, and what it means for Kubernetes: a Q&A about cluster scheduling"](https://blog.kismatic.com/qa-with-malte-schwarzkopf-on-distributed-systems-orchestration-in-the-modern-data-center/)が非常に良いインタビュー記事だった．Google Omegaとは何か? 今までのスケジューリングと何が違うのか? 何を解決しようとしているのか? 今後クラスタのスケジューリングにはどうなっていくのか? をとてもクリアに理解することができた．

自分にとってスケジューリングは今後大事になる分野であるし，勉強していきたい分野であるのでKismaticの[@asynchio](https://twitter.com/asynchio)氏と論文の共著者である[Malte Schwarzkopf](http://www.cl.cam.ac.uk/~ms705/)氏に許可をもらい翻訳させてもらった．

## TL;DR

2013年に発表された[Omega論文](http://www.cl.cam.ac.uk/research/srg/netos/papers/2013-omega.pdf)の共著者である[Malte Schwarzkopf](http://www.cl.cam.ac.uk/~ms705/)がGoogle OmegaのShared-stateモデルの主な目的がScalabilityよりもむしろソフトウェア開発における柔軟性であったことを説明する．Shared-stateモデルによるスケジューリングは優先度をもったプリエンプションや競合を意識したスケジューリングを可能にする．

今までのTwo-levelモデルのスケジューラー（例えば[YARN](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)や[Mesos](http://mesos.apache.org/)）も同様の柔軟さを提供するが，Omega論文が発表された当時は，すべての状態がクラスタから観測できない問題（information hiding）や[Gang Scheduling](https://en.wikipedia.org/wiki/Gang_scheduling)におけるhoardingの問題に苦しんでいた．またなぜGoogleがShare-stateモデルを選択したのか，なぜGoogleは（Mesosのような）厳格な公平性をもったリソース配分を行わないのかについてもコメントする．

最後にMesosや[kubernetes](http://kubernetes.io/)のようにスケジューラーのモジュール化をサポートすることでいかにOSSのクラスター管理にOmegaの利点を持ち込むことができるのかを議論する．そしてより賢いスケジューリングを実現することでユーザはGoogleのインフラと同様の効率性を獲得できること提案する．

**2013年の論文の発表時と比べて何が変わったか? 何が変わらないか?**

クラスタのオーケストレーションは動きの早い分野である．しかしOmegaの中心となる原則はむしろよりタイムリーでとても重要であると思う．

2013年以来の大きな変化として，とりわけKubernetesやMesosのおかげで，Omegaのようなクラスタでインフラを運用するのが一般的になってきたことが挙げられる．2011年や2012年に立ち返ってみるとOmegaのShared-stateモデルによるスケジューリングの基礎となる仮説をGoogle以外の環境で実証するのはなかなかトリッキーなことだとみなされていた．しかし今日では，既存のモジュラーなスケジューラーをハックして新しい手法を試したり，[Google cluster trace](https://github.com/google/cluster-data)の公開されたTrace結果を使うことでより簡単にそれができる．

さらにOmegaで挑んだ，いかにクラスタ内で異なるタイプのタスクを効率良くスケジューリングするのか，いかに異なるチームがクラスタの全ての状態にアクセスし彼ら自身のスケジューラーを実装するのか，といった問題は業界で広く認識されてきた．コンテナにより多くの異なるタイプのアプリケーションを共有クラスタ内にデプロイすることが可能になり，開発者たちは全てのタスクを同じようにスケジューリングすることはできない/するべきではないことを認識し始めた．そしてオーケストレーション（例えばZookeeperやetcd）は共有された分散状態を管理する問題であるであるとみなされるようになった．

Omegaのコアにある考え方，つまりジョブのスケジューリングのモデリングとShared-stateモデルに基づくより概括的なクラスタのオペレーションは，まだ適切であると信じている．実際Kubernetesの中心にある考え方，ユーザが望むべき状態を指定しKubernetesがクラスタをそのゴールの状態に移行させること，はまさにOmegaにおいてスケジューラーが次の望むべき状態にクラスタの状態の変更を提案するときに起こっていることと全く同じである．

**なぜOmegaのShared-stateモデルは柔軟性があり様々な異なるタスクのリソース管理を効率的に行うことができるのか?**

数年前多くの組織が運用していたインフラ環境について考えてみる．例えば1つのHadoopクラスタで複数のユーザによるMapReduceジョブが走っていた．それは非常に簡単なスケジューリングである．全てのマシンにMapReduce worker用にn個のスロットがあり，スケジューラーはmapとreduceタスクを，その全てのスロットが埋まるまでもしくは全てのタスクがなくなるまで，ワーカーに割り当てる．

しかし，このスロットという単位は非常に荒いスケジューリングの単位である．全てのMapReduceジョブが同じ量のリソースが必要であるわけではなく，全てのジョブが与えられたリソースを使い切るわけではない．実際クラスタのいくつかのサーバーではMapReduce以外のプロセスが動いている場合もある．そのためスロットという単位で静的にマシンのリソースを区切るのではなく，タスクを[Bin-pack](https://en.wikipedia.org/wiki/Bin_packing_problem)してリソースを最適化させるのはより良い考え方である．現代のほとんどのスケジューラーはこれを実現している．

複数のリソース状況に基づく（複数次元の）Bin-packingは非常に難しい（NP完全問題である）．さらに異なるタイプのタスクはそれぞれ別のBin-packingの方法を好むため問題はより難しくなる．例えば，ネットワーク帯域が70%使われているマシンにMapReduceのジョブを割り当てるのは全く問題ないが，webサーバーのジョブをそのマシンに割り当てるのは好ましくない...

Omegaではそれぞれのスケジューラーは全ての利用可能なリソース，すでに動いているタスク，クラスタの負荷状況を見ることができ，それに基づきスケジューリングを行うことができる（Shared-stateモデル）．言い方を変えると，それぞれのスケジューラーは好きなBin-packingアルゴリズムを使うことができ，かつ全て同様の情報を共有している．

**BorgはScalabilityに制限があるのか? OmegaはBorgの置き換えなのか?**

違う．論文でそれをよりクリアにできたらと思う．多くのフォローアップで「中央集権型のスケジューラーは巨大なクラスタに対してスケールできないため分散スケジューラーに移行するべきである」と述べられてきた．しかしOmegaの開発の主な目的はScalabilityではなくより柔軟なエンジニアリングにある．Omegaのゴールは，様々なチームが独立してスケジューラーを実装できることにあり，これはScalabilityよりも重要な側面だった．スケジューラーの並列化がもたらすScalabilityは付属の利点にすぎない．実際のところちゃんと開発された中央集権型のスケジューラーは巨大なクラスタとタスクを扱えるまでにスケールできる．Borg論文は実際この点について書いている．

分散スケジューラーが必須になるニッチな状況もある．例えば，既存のワーカーに対して，レイテンシにセンシティブな短いリクエストを送るジョブを高速に配置する必要があるとき．これは[Sparrow scheduler](http://dl.acm.org/citation.cfm?id=2517349.2522716)のターゲットであり分散デザインが適切になる．しかしタスクがレイテンシにセンシティブ，もしくはタスクを秒間に数万回も配置する必要がなければ，中央集権型のスケジューラーであっても1万台のマシンを超えても問題ない．

**Omega論文内で指摘しているMesosのTwo-levelモデルのスケジューリングの欠点は何か?**

まずOmega論文におけるMesosの説明は2012年当時のMesosに基づいている．それから数年が経っており，論文でのいくつかの指摘はすでに取組まれており，同じように語ることはできない．

オファーベースのモデル，もしくは別のスケジューラーに対してクラスタ状態のサブセットのみを公開するモデルには大きく2つの欠点がある．これは例えばYARNのようなリクエストベースのデザインにも同様のことが言える．

まず1つ目はスケジューラーが割り当てらてた/提供されたリソースのみを見ることができることに関連する．Mesosのオファーシステムにおいて，リソースマネージャーはアプケーションスケジューラーに対して「これだけのリソースがあるよ．どれが使いたい?」と尋ね，アプリケーションスケジューラーはその中から選択を行う．しかしそのときアプリケーションスケジューラーはに別の関連する情報を知ることができない．例えば，自分には提供されなかったリソースは誰が使っているのか，より好ましいリソースがあるのか（そのために提供されたリソースを拒否してより良いリソースを待ったほうがよいのか）という情報を知ることができない．これが**information hiding**の問題である．information hidingによって問題になる他の例には優先度をもったプリエンプションがある．もし優先度の高いタスクが優先度の低いタスクを追い出すことができる必要があるとき，優先度の低いタスクが配置されるどの場所もまた効率的なリソースのオファーがある，がスケジューラーはそれをみることができない．

もちろんこれは解くことができる問題である．例えばMesosは優先度の低いタスクに利用されているリソースを優先度の高いタスクに提供することができる．もしくはスケジューラーのフィルターでプリエンプション可能なリソースを指定することもできる．しかしこれはリソースマネージャーのAPIとロジックが複雑になる．

2つ目は**hoarding**の問題．これは特に[Gang Sheduling](https://en.wikipedia.org/wiki/Gang_scheduling)によりスケジューリングされたジョブに影響を与える．このようなジョブは他のジョブが起動する前にリクエストした全てのリソースを獲得しておかなければならない．例えばこのようなジョブにはMPIがあるが，他にもstatefulなストリーム処理は起動するためにパイプライン全体が確保されている必要がある．MesosのようなTwo-levelモデルではこれらのジョブには問題が生じる．アプリケーションスケジューラーは要求したリソースが全て揃うまで待つ（揃わないかもしれない）こともできるし，十分なリソースを蓄積するため少量のオファーを順番に受け入れることもできる．もし後者なら，しばらくの間他のジョブ（例えば優先度の低いMapReduceのジョブなど）に効率良く利用できる可能性があるのにも関わらず，十分な量のリクエストが受け入れられるまでリソースは使われることなく蓄積（hoarding）される．

最近MesosphereでMesosの開発をしているBen Hindmanとこの問題ついて話したが，彼らはこれらを解決する並列のリソースオファー/予約が可能になるようにコアのオファーモデルを変更する計画があると話していた．例えば，Mesosは複数のスケジューラーに対して同じリソースを[Optimistic](https://issues.apache.org/jira/browse/MESOS-1607)に提供し，消失したスケジューラーのオファーを「無効にする」ことが可能になる．これはOmegaと同じ競合の解決が必要になる．その時点で2つのモデルは同じところに到達する．もしMesosがクラスタの全てのリソースをスケジューラーに提供するならそのスケジューラーはOmegaと同じ視点をもつことになる（詳しくは["proposal: Mesos is isomorphic to Omega if makes offers for everything available"](https://issues.apache.org/jira/secure/attachment/12656071/optimisitic-offers.pdf)）．しかしまだ実装は初期段階にある．

**MesosのTwo-levelモデルのスケジューリングはGoogleには適していないのか? クラスタの全てのリソースを全てのスケジューラーに共有するのはなぜ良いのか?**

上で挙げた優先度をもったプリエンプションを例に考えてみる．Googleではより重要なタスクのために低い優先度のタスクが停止されるのは普通である．実際，巨大なMapReduceジョブを起動するといくつかのworkerがプリエンプションで失敗する．これは新しいより重要なジョブがあったか，もしくはそのジョブで利用されていた特定のリソースを他のタスクが必要とし（例えばGmailのようなサービスでスパイクがあり）Borgスケジューラーがそのタスクをプリエンプションの対象として選んだためである．しかしタスクを立ち退かせるためにはスケジューラーはそれらの状態を見る必要がある．例えばTwitterのHeronインスタンスは異なるフレームワークであってもHadoopやSparkのジョブを立ち退かせるということをしたい．

Omega論文で述べた別の例を挙げると，静的にMapReduceジョブを分割するのではなくてクラスタのロードが低いときに動的に割り当てるリソースを増やしたい．なぜそのジョブはしっかり100のworkerを使う必要があるのか? それは単に人間が指定した数にすぎない．スケジューラーはある時間にどれだけの数のworkerを起動するべきがより良く理解できる．実際他のシステムも同様のアイディアを実現してきた．例えばMicrosoftの[Apolloスケジューラー](https://kismatic.com/company/qa-with-malte-schwarzkopf-on-distributed-systems-orchestration-in-the-modern-data-center/)はOpportunisticなタスクをサポートしており，[Jockey](http://dl.acm.org/citation.cfm?id=2168847)や[Quasar](http://dl.acm.org/citation.cfm?id=2541941)のようなresearch systemはAuto-sacalingは非常に大きな違いを生じさせることを示した．しかしそのような決定ができるためには，スケジューラーはクラスタ全体の状態がどうなっているかが見える必要がある．そのためオファーベースでTwo-levelモデルのデザインはGoogleでの要件に合わなかった．

**MesosのDominant Resource Fairness（DRF）アルゴリズムについてはどう思うか? なぜGoogleは異なるアプローチを選択したのか?**

(DRFアルゴリズムは非常に速くスケジューラーに対して1ms以下でリソースを提供することができMesosで使われている)

GoogleがDRFアルゴリズムを採用していないのはアルゴリズムの複雑さやパフォーマンスとは全く関係ない．単にしっかりとした公平なリソース配分がGoogleの環境には必要ないだけである．より多くのタスクを終わらせ全体の最適化を行うためにリソースの配分を一時的に不公平にできる柔軟性をGoogleは評価している．もちろん偶然（もしくは故意に）大量のリソースを使って他のタスクが不利になることがないようにはしている．

これは割り当てシステム（quota system）によって行われる．仮想的な通貨がリソースの購入と予算に使われる．もし高い優先度をもつ100のタスクをそれぞれ20GBのメモリを使って1時間稼働させたとすると，チームの口座から2000GB/hourが借りられることになる．もし貯金を使い果たすとチームは人に頼んで通貨を増やしてもらうか，タスクを終了させる方法を見つけなればならない．つまり1日で1月分の配分を使い果たすのは良い考えではなく，しんどくなる．Borg論文がこのquota systemについて詳しく解説している．

しかしこのやり方はリソース配分の公平性についてより根本的な疑問を呈した．厳格な公平性をもつが使われていないリソースを放置するのか，より良い最適化やタスクのよりスムーズな終了が持ち込まれたときに一時的な不公平を許容するのか．答えは組織やタスクの種類による．Googleでは効率が厳格な公平性に勝った．

**Omegaのような実装がOSSとして公開されることはあるのか?**

今すぐにはない．上で述べたようにOmegaはいくつかのOSSのプロジェクトに影響を与えている．第一にそしてより重要なのはもちろんKubernetesであり，BorgとOmega両方の影響を受けている．Kubernetesにおける重要なアイディアは，ユーザが自分のクラスタの望むべき状態を指定してクラスタマネージャーにそれを託すこと．これはOmegaのアプローチに非常に似ている．その意味でKubernetesはOmegaスタイルのスケジューラーを載せるのには理想的なプラットフォームであると言える．実際Kubernetesのスケジューラーはpluggableなコンポーネントになっているので実現性は高い．[ドキュメント](https://github.com/kubernetes/kubernetes/blob/master/docs/design/architecture.md)には開発者は「将来的に複数のクラスタスケジューラーとユーザによるスケジューラーをサポートすることを期待している」とある．Kubernetesは完全にコンポーネント化されているので，複数のマスターコントローラーが複数のスケジューラーとやりとりする，もしくはアプリケーションに特化したReplication Controllerがアプリケーションに特化したスケジューラーにリクエストを投げるといった方法が想像できる．

Omegaの影響を受けたスケジューラーは他にも存在する．例えばMicrosoft Researchは[Mercury](http://msr-waypoint.com/pubs/238833/mercury-tr.pdf)を作った．これは，もしかすると古いかもしれない情報に基づき分散で決定をするのか，正確な情報をもとに中央集権的に決定をするのかをタスクが選べるというハイブリッドなスケジューラーである．そのコードはYARNのupstreamに統合されている．

**OmegaのようなShared-stateモデルのスケジューラーはkubernetesクラスタにおいても柔軟性や効率を改善することができるのか?**

最近KubernetesとMesosをスケジューラーの観点から見てみたが，正直に言うと，GoogleがOmegaやBorgでやっているのと比較するととても単純である．しかしplugabbleなスケジューラーAPIにより最新かつより良いものに改良することができる．基礎はあるのでいろいろなことができる．

例えばCambridgeでは本格的なスケジューラーマネージャーである[Firmament](http://www.cl.cam.ac.uk/research/srg/netos/camsas/firmament/)を研究のために開発した．それによりとても良い結果を得ることができたが，Kubernetesに組み込むことでその研究の成果は他の人たちにも簡単に利用可能になる．

Share-stateなアプローチはKubernetesクラスタをさらに助けることになると思う．異なるタイプのアプリケーション異なるタイプのPodは異なった方法で配置する必要があり，それらを全てをkube-schdulerに統合するのは悪夢のようだ．そうではなく特別な目的をもった複数のスケジューラーを走らせることができればKubernetesにとって良いオプションになると思う．つまりより良い決定が可能になり，コードを綺麗にかつモジュール化することができる．

**クラスタのスケジューラーにおける他の課題は何か? 次に解くべき困難な問題は何か?**

コンテナもまた異なるタイプのクラスタを作ってきた．Kubernetesや他のコンテナオーケストレーションプラットフォーム（例えばMesosや[Tectonic](https://tectonic.com/)，[Docker Machine](https://github.com/docker/machine)）によって，MapReduceでmapタスクからネイティブプロセスを起動したり，特定のAPIのためにアプリケーションを再ビルドするといったことを複雑ことをせずに，異なるアプリケーションを同じクラスタ上で動かすことができるようになった．この結果として人々がより賢いスケジューリングに集中し始めることを期待している．今のところOSSではほとんどのプロジェクトがとにかく動かすということをやってきた．これはスケールするのは難しい．より良いスケジューリングの実現が次のハードルであると思う．

いくつかの困難な課題を挙げる，

- 負荷の高いマシンで**co-location interferenceを避ける**のはとても重要である．貧弱なスケジューリングで特定のマシンのリソース（例えばディスクやネットワーク，L3キャッシュなど）を圧迫するとバッチタスクのパフォーマンスは簡単に悪化する．user-facingのサービスはより被害を被る（例えばレイテンシが非常に大きくなる）．コンテナはCPUやメモリの隔離を可能にするが，基本的にはまだ多くのリソースをk共有している．そのため，どのタスクが同居してよいか，どのタスクか別々になるべきかをよりうまく予測できる必要がある．
- Googleが**"resource reclamation"**と呼んでいること，確保ししたが実際には使わなかった余剰のリソースを返還すること，をよりうまくできる必要がある．人間が慎重になりすぎること，アプリケーションに必要になるリソースを過度に見積もってしまうのはよく知られている．しかし良いスケジューラーであれば，あまり使われていないリソースを他の優先度の低いタスクに再配布することができる．
- **柔軟性を高める**ことができると良い．スケジューラーにコンテナを停止させる，移動させる，そして自動的にスケールさせる．例えば，CPUが必要な他のコンテナに場所を譲るためにコンテナをプリエンプションして破棄するのではなくて，効率よく停止させるために優先度の低いコンテナを抑制することもできるが，メモリが圧迫されてなければRAMに状態を保持することもできる．
- スクラッチで自分用のスケジューラーを実装するのではなくて**user-specifiedなスケジューリングポリシー**を簡単に持てると良い．現実的にはほとんどの組織，小さいもしくは伝統的な組織，にはpluggableのAPIが提供されていたとしてもスケジューラーを書くためのリソースはない．タスクやその組織のためのスケジューリングポリシーを考えることができるSREやDevOps的なエンジニアが必要である．そしてスケジューラーはリソース配分やContainer配置のAPIというよりも，そのポリシーを表現するためのプラットフォームにならなければならない．

上述した[Firmament](http://camsas.org/firmament)においてこれらの課題のいくつかに挑んだ．例えばそのpluggableなコストモデルにより直感的な方法でスケジューリングのポリシーを簡単に指定することができる．そしてco-location interferenceが発生する前にそれをを避けるコストモデルも開発した．さらに巨大なスケール（数万台のマシン）でも特定のスケジューリングポリシーの最適な決定を非常に速く行うことができる可能性も示した．これらは非常に良いな結果であり，近い将来これについては詳しく述べる．しかし「Google's infrastructure for everyone else」を実現するには多くのひとそして多くの素晴らしいアイディアが必要になる．
